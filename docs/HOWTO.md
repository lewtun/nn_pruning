# Using nn_pruning on your own dataset

## Using with a Hugging Face Trainer
Examples are provided for [question answering](https://github.com/huggingface/nn_pruning/tree/main/nn_pruning/examples/question_answering)
and [text classification](https://github.com/huggingface/nn_pruning/tree/main/nn_pruning/examples/text_classification).

You can check how they are structured.

They basically use two classes to implement task specific code: 
- [sparse_xp.SparseXP](https://github.com/huggingface/nn_pruning/blob/main/nn_pruning/sparse_xp.py)
- [sparse_trainer.SparseTrainer](https://github.com/huggingface/nn_pruning/blob/main/nn_pruning/sparse_trainer.py)
 
The SparseXP basically contains a `nn_pruning.patch_coordinator.ModelPatchingCoordinator` instance, that will manage all the details of pruning.
The SparseTrainer class contains some functions that must be called during training.

It's not a subclass a trainer, as it's more intended to be added as a "Mixin" of your
 own subclass of Trainer, like in QATrainer:

`class QASparseTrainer(SparseTrainer, QATrainer):`

where QATrainer *is* a subclass of Trainer.

So if you don't have you own Trainer class, you can probably use simply a dummy class:

```
class MyTrainer(SparseTrainer, transformers.Trainer):
   pass
```


## Using without a Trainer

The previous code was itself based on a single class, `ModelPatchingCoordinator`.
You can then use it directly:

**Initialization:**
- Create your BERT transformers model (the lib just support BERT for the moment, but other models are very easy to add, we just need to configure a few regexps on the layer names)
- Create a `SparseTrainingArguments` object
- Create a `ModelPatchingCoordinator` (see the constructor for more information)
- Call the `patch_model` method on your ] (trial parameter can be ignored, it's for future use), you can ignore the returned value
- Call create_optimizer_groups

**Then, at each training step,**
Before calling model forward:
 - call `schedule_threshold` with the current time stept
 - call the `log` function to add pruning specific information
 
**Before calling backward:** 
- call `regularization_loss`, the first returned value is the regularization loss, other are used for logging
- if you are using distillation (you should as it improves results a lot), `call distil_loss_combine`,
with the loss returned by the model (ce_loss, so not containing the regularization loss), and the model inputs/outputs.
It will returns a total loss, typically 0.1 * ce_loss + 0.9 * distil_loss, you can then use for the next step.
- Then add the regularization loss with the total loss, and you have the loss to be used for backward.


**After training: compiling the model**

If you save your model as is, it will not be usable: the weights must be masked in a permanent way.
Just call the `compile` function on the `ModelPatchingCoordinator`, it will transform the model in-place, and make it transformers compatible.
Just save your model as usual, and you are good to go.



## Using for inference
You can use the [example notebook](https://github.com/huggingface/nn_pruning/blob/main/documentation/notebooks/Using%20pruned%20transformers%20for%20inference.ipynb)
to check how you can optimize your model for runtime.
This is needed for the moment, as the saved model contains a lot of zeroes, we just need to remove them before inference
but it's only a line of code and a very light operation: just removing heads/rows/columns from the linear layers. 






