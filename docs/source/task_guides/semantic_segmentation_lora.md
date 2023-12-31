<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->

# Semantic segmentation using LoRA

This guide demonstrates how to use LoRA, a low-rank approximation technique, to finetune a SegFormer model variant for semantic segmentation.
By using LoRA from 🤗 PEFT, we can reduce the number of trainable parameters in the SegFormer model to only 14% of the original trainable parameters.

LoRA achieves this reduction by adding low-rank "update matrices" to specific blocks of the model, such as the attention
blocks. During fine-tuning, only these matrices are trained, while the original model parameters are left unchanged.
At inference time, the update matrices are merged with the original model parameters to produce the final classification result.

For more information on LoRA, please refer to the [original LoRA paper](https://arxiv.org/abs/2106.09685).

## Install dependencies

Install the libraries required for model training:

```bash
!pip install transformers accelerate evaluate datasets peft -q
```

## Authenticate to share your model

To share the finetuned model with the community at the end of the training, authenticate using your 🤗 token.
You can obtain your token from your [account settings](https://huggingface.co/settings/token).

```python
from huggingface_hub import notebook_login

notebook_login()
```

## Load a dataset

To ensure that this example runs within a reasonable time frame, here we are limiting the number of instances from the training 
set of the [SceneParse150 dataset](https://huggingface.co/datasets/scene_parse_150) to 150. 

```python
from datasets import load_dataset

ds = load_dataset("scene_parse_150", split="train[:150]")
```

Next, split the dataset into train and test sets. 

```python
ds = ds.train_test_split(test_size=0.1)
train_ds = ds["train"]
test_ds = ds["test"]
```

## Prepare label maps

Create a dictionary that maps a label id to a label class, which will be useful when setting up the model later: 
* `label2id`: maps the semantic classes of the dataset to integer ids.
* `id2label`: maps integer ids back to the semantic classes.

```python
import json
from huggingface_hub import cached_download, hf_hub_url

repo_id = "huggingface/label-files"
filename = "ade20k-id2label.json"
id2label = json.load(open(cached_download(hf_hub_url(repo_id, filename, repo_type="dataset")), "r"))
id2label = {int(k): v for k, v in id2label.items()}
label2id = {v: k for k, v in id2label.items()}
num_labels = len(id2label)
```

## Prepare datasets for training and evaluation

Next, load the SegFormer image processor to prepare the images and annotations for the model. This dataset uses the 
zero-index as the background class, so make sure to set `do_reduce_labels=True` to subtract one from all labels since the
background class is not among the 150 classes. 

```python
from transformers import AutoImageProcessor

checkpoint = "nvidia/mit-b0"
image_processor = AutoImageProcessor.from_pretrained(checkpoint, do_reduce_labels=True)
```

Add a function to apply data augmentation to the images, so that the model is more robust against overfitting. Here we use the 
[ColorJitter](https://pytorch.org/vision/stable/generated/torchvision.transforms.ColorJitter.html) function from 
[torchvision](https://pytorch.org/vision/stable/index.html) to randomly change the color properties of an image.

```python
from torchvision.transforms import ColorJitter

jitter = ColorJitter(brightness=0.25, contrast=0.25, saturation=0.25, hue=0.1)
```

Add a function to handle grayscale images and ensure that each input image has three color channels, regardless of 
whether it was originally grayscale or RGB. The function converts RGB images to array as is, and for grayscale images 
that have only one color channel, the function replicates the same channel three times using `np.tile()` before converting 
the image into an array.

```python
import numpy as np


def handle_grayscale_image(image):
    np_image = np.array(image)
    if np_image.ndim == 2:
        tiled_image = np.tile(np.expand_dims(np_image, -1), 3)
        return Image.fromarray(tiled_image)
    else:
        return Image.fromarray(np_image)
```

Finally, combine everything in two functions that you'll use to transform training and validation data. The two functions 
are similar except data augmentation is applied only to the training data.  

```python
from PIL import Image


def train_transforms(example_batch):
    images = [jitter(handle_grayscale_image(x)) for x in example_batch["image"]]
    labels = [x for x in example_batch["annotation"]]
    inputs = image_processor(images, labels)
    return inputs


def val_transforms(example_batch):
    images = [handle_grayscale_image(x) for x in example_batch["image"]]
    labels = [x for x in example_batch["annotation"]]
    inputs = image_processor(images, labels)
    return inputs
```

To apply the preprocessing functions over the entire dataset, use the 🤗 Datasets `set_transform` function:

```python 
train_ds.set_transform(train_transforms)
test_ds.set_transform(val_transforms)
```

## Create evaluation function

Including a metric during training is helpful for evaluating your model's performance. You can load an evaluation 
method with the [🤗 Evaluate](https://huggingface.co/docs/evaluate/index) library. For this task, use 
the [mean Intersection over Union (IoU)](https://huggingface.co/spaces/evaluate-metric/accuracy) metric (see the 🤗 Evaluate 
[quick tour](https://huggingface.co/docs/evaluate/a_quick_tour) to learn more about how to load and compute a metric):

```python
import torch
from torch import nn
import evaluate

metric = evaluate.load("mean_iou")


def compute_metrics(eval_pred):
    with torch.no_grad():
        logits, labels = eval_pred
        logits_tensor = torch.from_numpy(logits)
        logits_tensor = nn.functional.interpolate(
            logits_tensor,
            size=labels.shape[-2:],
            mode="bilinear",
            align_corners=False,
        ).argmax(dim=1)

        pred_labels = logits_tensor.detach().cpu().numpy()
        # currently using _compute instead of compute
        # see this issue for more info: https://github.com/huggingface/evaluate/pull/328#issuecomment-1286866576
        metrics = metric._compute(
            predictions=pred_labels,
            references=labels,
            num_labels=len(id2label),
            ignore_index=0,
            reduce_labels=image_processor.do_reduce_labels,
        )

        per_category_accuracy = metrics.pop("per_category_accuracy").tolist()
        per_category_iou = metrics.pop("per_category_iou").tolist()

        metrics.update({f"accuracy_{id2label[i]}": v for i, v in enumerate(per_category_accuracy)})
        metrics.update({f"iou_{id2label[i]}": v for i, v in enumerate(per_category_iou)})

        return metrics
```

## Load a base model 

Before loading a base model, let's define a helper function to check the total number of parameters a model has, as well
as how many of them are trainable.

```python
def print_trainable_parameters(model):
    """
    Prints the number of trainable parameters in the model.
    """
    trainable_params = 0
    all_param = 0
    for _, param in model.named_parameters():
        all_param += param.numel()
        if param.requires_grad:
            trainable_params += param.numel()
    print(
        f"trainable params: {trainable_params} || all params: {all_param} || trainable%: {100 * trainable_params / all_param:.2f}"
    )
```

Choose a base model checkpoint. For this example, we use the [SegFormer B0 variant](https://huggingface.co/nvidia/mit-b0). 
In addition to the checkpoint, pass the `label2id` and `id2label` dictionaries to let the `AutoModelForSemanticSegmentation` class know that we're 
interested in a custom base model where the decoder head should be randomly initialized using the classes from the custom dataset.

```python
from transformers import AutoModelForSemanticSegmentation, TrainingArguments, Trainer

model = AutoModelForSemanticSegmentation.from_pretrained(
    checkpoint, id2label=id2label, label2id=label2id, ignore_mismatched_sizes=True
)
print_trainable_parameters(model)
```

At this point you can check with the `print_trainable_parameters` helper function that all 100% parameters in the base 
model (aka `model`) are trainable.

## Wrap the base model as a PeftModel for LoRA training

To leverage the LoRa method, you need to wrap the base model as a `PeftModel`.  This involves two steps:

1. Defining LoRa configuration with `LoraConfig`
2. Wrapping the original `model` with `get_peft_model()` using the config defined in the step above.

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=32,
    lora_alpha=32,
    target_modules=["query", "value"],
    lora_dropout=0.1,
    bias="lora_only",
    modules_to_save=["decode_head"],
)
lora_model = get_peft_model(model, config)
print_trainable_parameters(lora_model)
```

Let's review the `LoraConfig`. To enable LoRA technique, we must define the target modules within `LoraConfig` so that 
`PeftModel` can update the necessary matrices. Specifically, we want to target the `query` and `value` matrices in the 
attention blocks of the base model. These matrices are identified by their respective names, "query" and "value". 
Therefore, we should specify these names in the `target_modules` argument of `LoraConfig`.

After we wrap our base model `model` with `PeftModel` along with the config, we get 
a new model where only the LoRA parameters are trainable (so-called "update matrices") while the pre-trained parameters 
are kept frozen. These include the parameters of the randomly initialized classifier parameters too. This is NOT we want 
when fine-tuning the base model on our custom dataset. To ensure that the classifier parameters are also trained, we 
specify `modules_to_save`. This also ensures that these modules are serialized alongside the LoRA trainable parameters 
when using utilities like `save_pretrained()` and `push_to_hub()`.

In addition to specifying the `target_modules` within `LoraConfig`, we also need to specify the `modules_to_save`. When 
we wrap our base model with `PeftModel` and pass the configuration, we obtain a new model in which only the LoRA parameters 
are trainable, while the pre-trained parameters and the randomly initialized classifier parameters are kept frozen. 
However, we do want to train the classifier parameters. By specifying the `modules_to_save` argument, we ensure that the 
classifier parameters are also trainable, and they will be serialized alongside the LoRA trainable parameters when we 
use utility functions like `save_pretrained()` and `push_to_hub()`.

Let's review the rest of the parameters:

- `r`: The dimension used by the LoRA update matrices.
- `alpha`: Scaling factor.
- `bias`: Specifies if the `bias` parameters should be trained. `None` denotes none of the `bias` parameters will be trained.

When all is configured, and the base model is wrapped, the `print_trainable_parameters` helper function lets us explore 
the number of trainable parameters. Since we're interested in performing **parameter-efficient fine-tuning**, 
we should expect to see a lower number of trainable parameters from the `lora_model` in comparison to the original `model` 
which is indeed the case here.

You can also manually verify what modules are trainable in the `lora_model`.

```python
for name, param in lora_model.named_parameters():
    if param.requires_grad:
        print(name, param.shape)
```

This confirms that only the LoRA parameters appended to the attention blocks and the `decode_head` parameters are trainable.

## Train the model

Start by defining your training hyperparameters in `TrainingArguments`. You can change the values of most parameters however 
you prefer. Make sure to set `remove_unused_columns=False`, otherwise the image column will be dropped, and it's required here.
The only other required parameter is `output_dir` which specifies where to save your model. 
At the end of each epoch, the `Trainer` will evaluate the IoU metric and save the training checkpoint.

Note that this example is meant to walk you through the workflow when using PEFT for semantic segmentation. We didn't 
perform extensive hyperparameter tuning to achieve optimal results.

```python
model_name = checkpoint.split("/")[-1]

training_args = TrainingArguments(
    output_dir=f"{model_name}-scene-parse-150-lora",
    learning_rate=5e-4,
    num_train_epochs=50,
    per_device_train_batch_size=4,
    per_device_eval_batch_size=2,
    save_total_limit=3,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    logging_steps=5,
    remove_unused_columns=False,
    push_to_hub=True,
    label_names=["labels"],
)
```

Pass the training arguments to `Trainer` along with the model, dataset, and `compute_metrics` function.
Call `train()` to finetune your model.

```python
trainer = Trainer(
    model=lora_model,
    args=training_args,
    train_dataset=train_ds,
    eval_dataset=test_ds,
    compute_metrics=compute_metrics,
)

trainer.train()
```

## Save the model and run inference

Use the `save_pretrained()` method of the `lora_model` to save the *LoRA-only parameters* locally. 
Alternatively,  use the `push_to_hub()` method to upload these parameters directly to the Hugging Face Hub 
(as shown in the [Image classification using LoRA](image_classification_lora) task guide).

```python
model_id = "segformer-scene-parse-150-lora"
lora_model.save_pretrained(model_id)
```

We can see that the LoRA-only parameters are just **2.2 MB in size**! This greatly improves the portability when using very large models.

```bash
!ls -lh {model_id}
total 2.2M
-rw-r--r-- 1 root root  369 Feb  8 03:09 adapter_config.json
-rw-r--r-- 1 root root 2.2M Feb  8 03:09 adapter_model.bin
```

Let's now prepare an `inference_model` and run inference.

```python
from peft import PeftConfig

config = PeftConfig.from_pretrained(model_id)
model = AutoModelForSemanticSegmentation.from_pretrained(
    checkpoint, id2label=id2label, label2id=label2id, ignore_mismatched_sizes=True
)

inference_model = PeftModel.from_pretrained(model, model_id)
```

Get an image:

```python
import requests

url = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/semantic-seg-image.png"
image = Image.open(requests.get(url, stream=True).raw)
image
```

<div class="flex justify-center">
    <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/semantic-seg-image.png" alt="photo of a room"/>
</div>

Preprocess the image to prepare for inference.

```python
encoding = image_processor(image.convert("RGB"), return_tensors="pt")
```

Run inference with the encoded image.

```python
with torch.no_grad():
    outputs = inference_model(pixel_values=encoding.pixel_values)
    logits = outputs.logits

upsampled_logits = nn.functional.interpolate(
    logits,
    size=image.size[::-1],
    mode="bilinear",
    align_corners=False,
)

pred_seg = upsampled_logits.argmax(dim=1)[0]
```

Next, visualize the results.  We need a color palette for this. Here, we use ade_palette(). As it is a long array, so
we don't include it in this guide, please copy it from [the TensorFlow Model Garden repository](https://github.com/tensorflow/models/blob/3f1ca33afe3c1631b733ea7e40c294273b9e406d/research/deeplab/utils/get_dataset_colormap.py#L51).

```python
import matplotlib.pyplot as plt

color_seg = np.zeros((pred_seg.shape[0], pred_seg.shape[1], 3), dtype=np.uint8)
palette = np.array(ade_palette())

for label, color in enumerate(palette):
    color_seg[pred_seg == label, :] = color
color_seg = color_seg[..., ::-1]  # convert to BGR

img = np.array(image) * 0.5 + color_seg * 0.5  # plot the image with the segmentation map
img = img.astype(np.uint8)

plt.figure(figsize=(15, 10))
plt.imshow(img)
plt.show()
```

As you can see, the results are far from perfect, however, this example is designed to illustrate the end-to-end workflow of 
fine-tuning a semantic segmentation model with LoRa technique, and is not aiming to achieve state-of-the-art 
results. The results you see here are the same as you would get if you performed full fine-tuning on the same setup (same 
model variant, same dataset, same training schedule, etc.), except LoRA allows to achieve them with a fraction of total 
trainable parameters and in less time.

If you wish to use this example and improve the results, here are some things that you can try:

* Increase the number of training samples.
* Try a larger SegFormer model variant (explore available model variants on the [Hugging Face Hub](https://huggingface.co/models?search=segformer)).
* Try different values for the arguments available in `LoraConfig`.
* Tune the learning rate and batch size.


