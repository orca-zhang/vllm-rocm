(adding-a-new-model)=

# Adding a New Model

This document provides a high-level guide on integrating a [HuggingFace Transformers](https://github.com/huggingface/transformers) model into vLLM.

```{note}
The complexity of adding a new model depends heavily on the model's architecture.
The process is considerably straightforward if the model shares a similar architecture with an existing model in vLLM.
However, for models that include new operators (e.g., a new attention mechanism), the process can be a bit more complex.
```

```{note}
By default, vLLM models do not support multi-modal inputs. To enable multi-modal support,
please follow [this guide](#enabling-multimodal-inputs) after implementing the model here.
```

```{tip}
If you are encountering issues while integrating your model into vLLM, feel free to open an issue on our [GitHub](https://github.com/vllm-project/vllm/issues) repository.
We will be happy to help you out!
```

## 0. Fork the vLLM repository

Start by forking our [GitHub] repository and then [build it from source](#build-from-source).
This gives you the ability to modify the codebase and test your model.

```{tip}
If you don't want to fork the repository and modify vLLM's codebase, please refer to the "Out-of-Tree Model Integration" section below.
```

## 1. Bring your model code

Clone the PyTorch model code from the HuggingFace Transformers repository and put it into the [vllm/model_executor/models](https://github.com/vllm-project/vllm/tree/main/vllm/model_executor/models) directory.
For instance, vLLM's [OPT model](https://github.com/vllm-project/vllm/blob/main/vllm/model_executor/models/opt.py) was adapted from the HuggingFace's [modeling_opt.py](https://github.com/huggingface/transformers/blob/main/src/transformers/models/opt/modeling_opt.py) file.

```{warning}
When copying the model code, make sure to review and adhere to the code's copyright and licensing terms.
```

## 2. Make your code compatible with vLLM

To ensure compatibility with vLLM, your model must meet the following requirements:

### Initialization Code

All vLLM modules within the model must include a `prefix` argument in their constructor. This `prefix` is typically the full name of the module in the model's state dictionary and is crucial for:

- Runtime support: vLLM's attention operators are registered in a model's state by their full names. Each attention operator must have a unique prefix as its layer name to avoid conflicts.
- Non-uniform quantization support: A quantized checkpoint can selectively quantize certain layers while keeping others in full precision. By providing the `prefix` during initialization, vLLM can match the current layer's `prefix` with the quantization configuration to determine if the layer should be initialized in quantized mode.

The initialization code should look like this:

```python
from torch import nn
from vllm.config import VllmConfig
from vllm.attention import Attention

class MyAttention(nn.Module):
    def __init__(self, vllm_config: VllmConfig, prefix: str):
        super().__init__()
        self.attn = Attention(prefix=f"{prefix}.attn")

class MyDecoderLayer(nn.Module):
    def __init__(self, vllm_config: VllmConfig, prefix: str):
        super().__init__()
        self.self_attn = MyAttention(prefix=f"{prefix}.self_attn")

class MyModel(nn.Module):
    def __init__(self, vllm_config: VllmConfig, prefix: str):
        super().__init__()
        self.layers = nn.ModuleList(
            [MyDecoderLayer(vllm_config, prefix=f"{prefix}.layers.{i}") for i in range(vllm_config.model_config.hf_config.num_hidden_layers)]
        )

class MyModelForCausalLM(nn.Module):
    def __init__(self, vllm_config: VllmConfig, prefix: str = ""):
        super().__init__()
        self.model = MyModel(vllm_config, prefix=f"{prefix}.model")
```

### Computation Code

Rewrite the {meth}`~torch.nn.Module.forward` method of your model to remove any unnecessary code, such as training-specific code. Modify the input parameters to treat `input_ids` and `positions` as flattened tensors with a single batch size dimension, without a max-sequence length dimension.

```python
def forward(
    self,
    input_ids: torch.Tensor,
    positions: torch.Tensor,
    kv_caches: List[torch.Tensor],
    attn_metadata: AttentionMetadata,
) -> torch.Tensor:
    ...
```

```{note}
Currently, vLLM supports the basic multi-head attention mechanism and its variant with rotary positional embeddings.
If your model employs a different attention mechanism, you will need to implement a new attention layer in vLLM.
```

For reference, check out the [LLAMA model](https://github.com/vllm-project/vllm/blob/main/vllm/model_executor/models/llama.py). vLLM already supports a large number of models. It is recommended to find a model similar to yours and adapt it to your model's architecture. Check out the [vLLM models](https://github.com/vllm-project/vllm/tree/main/vllm/model_executor/models) directory for more examples.

## 3. (Optional) Implement tensor parallelism and quantization support

If your model is too large to fit into a single GPU, you can use tensor parallelism to manage it.
To do this, substitute your model's linear and embedding layers with their tensor-parallel versions.
For the embedding layer, you can simply replace {class}`torch.nn.Embedding` with {code}`VocabParallelEmbedding`. For the output LM head, you can use {code}`ParallelLMHead`.
When it comes to the linear layers, we provide the following options to parallelize them:

- {code}`ReplicatedLinear`: Replicates the inputs and weights across multiple GPUs. No memory saving.
- {code}`RowParallelLinear`: The input tensor is partitioned along the hidden dimension. The weight matrix is partitioned along the rows (input dimension). An *all-reduce* operation is performed after the matrix multiplication to reduce the results. Typically used for the second FFN layer and the output linear transformation of the attention layer.
- {code}`ColumnParallelLinear`: The input tensor is replicated. The weight matrix is partitioned along the columns (output dimension). The result is partitioned along the column dimension. Typically used for the first FFN layer and the separated QKV transformation of the attention layer in the original Transformer.
- {code}`MergedColumnParallelLinear`: Column-parallel linear that merges multiple {code}`ColumnParallelLinear` operators. Typically used for the first FFN layer with weighted activation functions (e.g., SiLU). This class handles the sharded weight loading logic of multiple weight matrices.
- {code}`QKVParallelLinear`: Parallel linear layer for the query, key, and value projections of the multi-head and grouped-query attention mechanisms. When number of key/value heads are less than the world size, this class replicates the key/value heads properly. This class handles the weight loading and replication of the weight matrices.

Note that all the linear layers above take {code}`linear_method` as an input. vLLM will set this parameter according to different quantization schemes to support weight quantization.

## 4. Implement the weight loading logic

You now need to implement the {code}`load_weights` method in your {code}`*ForCausalLM` class.
This method should load the weights from the HuggingFace's checkpoint file and assign them to the corresponding layers in your model. Specifically, for {code}`MergedColumnParallelLinear` and {code}`QKVParallelLinear` layers, if the original model has separated weight matrices, you need to load the different parts separately.

## 5. Register your model

Finally, register your {code}`*ForCausalLM` class to the {code}`_VLLM_MODELS` in [vllm/model_executor/models/registry.py](https://github.com/vllm-project/vllm/blob/main/vllm/model_executor/models/registry.py).

## 6. Out-of-Tree Model Integration

You can integrate a model without modifying the vLLM codebase. Steps 2, 3, and 4 are still required, but you can skip steps 1 and 5. Instead, write a plugin to register your model. For general introduction of the plugin system, see [plugin-system](#plugin-system).

To register the model, use the following code:

```python
from vllm import ModelRegistry
from your_code import YourModelForCausalLM
ModelRegistry.register_model("YourModelForCausalLM", YourModelForCausalLM)
```

If your model imports modules that initialize CUDA, consider lazy-importing it to avoid errors like {code}`RuntimeError: Cannot re-initialize CUDA in forked subprocess`:

```python
from vllm import ModelRegistry

ModelRegistry.register_model("YourModelForCausalLM", "your_code:YourModelForCausalLM")
```

```{important}
If your model is a multimodal model, ensure the model class implements the {class}`~vllm.model_executor.models.interfaces.SupportsMultiModal` interface.
Read more about that [here](#enabling-multimodal-inputs).
```

```{note}
Although you can directly put these code snippets in your script using `vllm.LLM`, the recommended way is to place these snippets in a vLLM plugin. This ensures compatibility with various vLLM features like distributed inference and the API server.
```
