<!--⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.
-->

# int8 training for automatic speech recognition

Quantization reduces the precision of floating point data types, decreasing the memory required to store model weights. However, quantization degrades inference performance because you lose information when you reduce the precision. 8-bit or `int8` quantization uses only a quarter precision, but it does not degrade performance because it doesn't just drop the bits or data. Instead, `int8` quantization *rounds* from one data type to another.

<Tip>

💡 Read the [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale](https://arxiv.org/abs/2208.07339) paper to learn more, or you can take a look at the corresponding [blog post](https://huggingface.co/blog/hf-bitsandbytes-integration) for a gentler introduction.

</Tip>

This guide will show you how to train a [`openai/whisper-large-v2`](https://huggingface.co/openai/whisper-large-v2) model for multilingual automatic speech recognition (ASR) using a combination of `int8` quantization and LoRA. You'll train Whisper for multilingual ASR on Marathi from the [Common Voice 11.0](https://huggingface.co/datasets/mozilla-foundation/common_voice_11_0) dataset.

Before you start, make sure you have all the necessary libraries installed:

```bash
!pip install -q peft transformers datasets accelerate evaluate jiwer bitsandbytes
```

## Setup

Let's take care of some of the setup first so you can start training faster later. Set the `CUDA_VISIBLE_DEVICES` to `0` to use the first GPU on your machine. Then you can specify the model name (either a Hub model repository id or a path to a directory containing the model), language and language abbreviation to train on, the task type, and the dataset name:

```py
import os

os.environ["CUDA_VISIBLE_DEVICES"] = "0"
model_name_or_path = "openai/whisper-large-v2"
language = "Marathi"
language_abbr = "mr"
task = "transcribe"
dataset_name = "mozilla-foundation/common_voice_11_0"
```

You can also log in to your Hugging Face account to save and share your trained model on the Hub if you'd like:

```py
from huggingface_hub import notebook_login

notebook_login()
```

## Load dataset and metric

The [Common Voice 11.0](https://huggingface.co/datasets/mozilla-foundation/common_voice_11_0) dataset contains many hours of recorded speech in many different languages. This guide uses the [Marathi](https://huggingface.co/datasets/mozilla-foundation/common_voice_11_0/viewer/mr/train) language as an example, but feel free to use any other language you're interested in. 

Initialize a [`~datasets.DatasetDict`] structure, and load the [`train`] (load both the `train+validation` split into `train`) and [`test`] splits from the dataset into it:

```py
from datasets import load_dataset
from datasets import load_dataset, DatasetDict

common_voice = DatasetDict()

common_voice["train"] = load_dataset(dataset_name, language_abbr, split="train+validation", use_auth_token=True)
common_voice["test"] = load_dataset(dataset_name, language_abbr, split="test", use_auth_token=True)
common_voice["train"][0]
```

## Preprocess dataset

Let's prepare the dataset for training. Load a feature extractor, tokenizer, and processor. You should also pass the language and task to the tokenizer and processor so they know how to process the inputs:

```py
from transformers import AutoFeatureExtractor, AutoTokenizer, AutoProcessor

feature_extractor = AutoFeatureExtractor.from_pretrained(model_name_or_path)
tokenizer = AutoTokenizer.from_pretrained(model_name_or_path, language=language, task=task)
processor = AutoProcessor.from_pretrained(model_name_or_path, language=language, task=task)
```

You'll only be training on the `sentence` and `audio` columns, so you can remove the rest of the metadata with [`~datasets.Dataset.remove_columns`]:

```py
common_voice = common_voice.remove_columns(
    ["accent", "age", "client_id", "down_votes", "gender", "locale", "path", "segment", "up_votes"]
)
common_voice["train"][0]
{
    "audio": {
        "path": "/root/.cache/huggingface/datasets/downloads/extracted/f7e1ef6a2d14f20194999aad5040c5d4bb3ead1377de3e1bbc6e9dba34d18a8a/common_voice_mr_30585613.mp3",
        "array": array(
            [1.13686838e-13, -1.42108547e-13, -1.98951966e-13, ..., 4.83472422e-06, 3.54798703e-06, 1.63231743e-06]
        ),
        "sampling_rate": 48000,
    },
    "sentence": "आईचे आजारपण वाढत चालले, तसतशी मथीही नीट खातपीतनाशी झाली.",
}
```

If you look at the `sampling_rate`, you'll see the audio was sampled at 48kHz. The Whisper model was pretrained on audio inputs at 16kHZ which means you'll need to downsample the audio inputs to match what the model was pretrained on. Downsample the audio by using the [`~datasets.Dataset.cast_column`] method on the `audio` column, and set the `sampling_rate` to 16kHz. The audio input is resampled on the fly the next time you call it:

```py
from datasets import Audio

common_voice = common_voice.cast_column("audio", Audio(sampling_rate=16000))
common_voice["train"][0]
{
    "audio": {
        "path": "/root/.cache/huggingface/datasets/downloads/extracted/f7e1ef6a2d14f20194999aad5040c5d4bb3ead1377de3e1bbc6e9dba34d18a8a/common_voice_mr_30585613.mp3",
        "array": array(
            [-3.06954462e-12, -3.63797881e-12, -4.54747351e-12, ..., -7.74800901e-06, -1.74738125e-06, 4.36312439e-06]
        ),
        "sampling_rate": 16000,
    },
    "sentence": "आईचे आजारपण वाढत चालले, तसतशी मथीही नीट खातपीतनाशी झाली.",
}
```

Once you've cleaned up the dataset, you can write a function to generate the correct model inputs. The function should:

1. Resample the audio inputs to 16kHZ by loading the `audio` column.
2. Compute the input features from the audio `array` using the feature extractor.
3. Tokenize the `sentence` column to the input labels.

```py
def prepare_dataset(batch):
    audio = batch["audio"]
    batch["input_features"] = feature_extractor(audio["array"], sampling_rate=audio["sampling_rate"]).input_features[0]
    batch["labels"] = tokenizer(batch["sentence"]).input_ids
    return batch
```

Apply the `prepare_dataset` function to the dataset with the [`~datasets.Dataset.map`] function, and set the `num_proc` argument to `2` to enable multiprocessing (if `map` hangs, then set `num_proc=1`):

```py
common_voice = common_voice.map(prepare_dataset, remove_columns=common_voice.column_names["train"], num_proc=2)
```

Finally, create a `DataCollator` class to pad the labels in each batch to the maximum length, and replace padding with `-100` so they're ignored by the loss function. Then initialize an instance of the data collator:

```py
import torch

from dataclasses import dataclass
from typing import Any, Dict, List, Union


@dataclass
class DataCollatorSpeechSeq2SeqWithPadding:
    processor: Any

    def __call__(self, features: List[Dict[str, Union[List[int], torch.Tensor]]]) -> Dict[str, torch.Tensor]:
        input_features = [{"input_features": feature["input_features"]} for feature in features]
        batch = self.processor.feature_extractor.pad(input_features, return_tensors="pt")

        label_features = [{"input_ids": feature["labels"]} for feature in features]
        labels_batch = self.processor.tokenizer.pad(label_features, return_tensors="pt")

        labels = labels_batch["input_ids"].masked_fill(labels_batch.attention_mask.ne(1), -100)

        if (labels[:, 0] == self.processor.tokenizer.bos_token_id).all().cpu().item():
            labels = labels[:, 1:]

        batch["labels"] = labels

        return batch


data_collator = DataCollatorSpeechSeq2SeqWithPadding(processor=processor)
```

## Train

Now that the dataset is ready, you can turn your attention to the model. Start by loading the pretrained [`openai/whisper-large-v2`]() model from [`~transformers.AutoModelForSpeechSeq2Seq`], and make sure to set the [`~transformers.BitsAndBytesConfig.load_in_8bit`] argument to `True` to enable `int8` quantization. The `device_map=auto` argument automatically determines how to load and store the model weights:

```py
from transformers import AutoModelForSpeechSeq2Seq

model = AutoModelForSpeechSeq2Seq.from_pretrained(model_name_or_path, load_in_8bit=True, device_map="auto")
```

You should configure `forced_decoder_ids=None` because no tokens are used before sampling, and you won't need to suppress any tokens during generation either:

```py
model.config.forced_decoder_ids = None
model.config.suppress_tokens = []
```

To get the model ready for `int8` quantization, use the utility function [`prepare_model_for_int8_training`](https://github.com/huggingface/peft/blob/34027fe813756897767b9a6f19ae7f1c4c7b418c/src/peft/utils/other.py#L35) to handle the following:

- casts all the non `int8` modules to full precision (`fp32`) for stability
- adds a forward hook to the input embedding layer to calculate the gradients of the input hidden states
- enables gradient checkpointing for more memory-efficient training

```py
from peft import prepare_model_for_int8_training

model = prepare_model_for_int8_training(model)
```

Let's also apply LoRA to the training to make it even more efficient. Load a [`~peft.LoraConfig`] and configure the following parameters:

- `r`, the dimension of the low-rank matrices
- `lora_alpha`, scaling factor for the weight matrices
- `target_modules`, the name of the attention matrices to apply LoRA to (`q_proj` and `v_proj`, or query and value in this case)
- `lora_dropout`, dropout probability of the LoRA layers
- `bias`, set to `none`

<Tip>

💡 The weight matrix is scaled by `lora_alpha/r`, and a higher `lora_alpha` value assigns more weight to the LoRA activations. For performance, we recommend setting bias to `None` first, and then `lora_only`, before trying `all`.

</Tip>

```py
from peft import LoraConfig, PeftModel, LoraModel, LoraConfig, get_peft_model

config = LoraConfig(r=32, lora_alpha=64, target_modules=["q_proj", "v_proj"], lora_dropout=0.05, bias="none")
```

After you set up the [`~peft.LoraConfig`], wrap it and the base model with the [`get_peft_model`] function to create a [`PeftModel`]. Print out the number of trainable parameters to see how much more efficient LoRA is compared to fully training the model!

```py
model = get_peft_model(model, config)
model.print_trainable_parameters()
"trainable params: 15728640 || all params: 1559033600 || trainable%: 1.0088711365810203"
```

Now you're ready to define some training hyperparameters in the [`~transformers.Seq2SeqTrainingArguments`] class, such as where to save the model to, batch size, learning rate, and number of epochs to train for. The [`PeftModel`] doesn't have the same signature as the base model, so you'll need to explicitly set `remove_unused_columns=False` and `label_names=["labels"]`.

```py
from transformers import Seq2SeqTrainingArguments

training_args = Seq2SeqTrainingArguments(
    output_dir="your-name/int8-whisper-large-v2-asr",
    per_device_train_batch_size=8,
    gradient_accumulation_steps=1,
    learning_rate=1e-3,
    warmup_steps=50,
    num_train_epochs=3,
    evaluation_strategy="epoch",
    fp16=True,
    per_device_eval_batch_size=8,
    generation_max_length=128,
    logging_steps=25,
    remove_unused_columns=False,
    label_names=["labels"],
)
```

It is also a good idea to write a custom [`~transformers.TrainerCallback`] to save model checkpoints during training:

```py
from transformers.trainer_utils import PREFIX_CHECKPOINT_DIR


class SavePeftModelCallback(TrainerCallback):
    def on_save(
        self,
        args: TrainingArguments,
        state: TrainerState,
        control: TrainerControl,
        **kwargs,
    ):
        checkpoint_folder = os.path.join(args.output_dir, f"{PREFIX_CHECKPOINT_DIR}-{state.global_step}")

        peft_model_path = os.path.join(checkpoint_folder, "adapter_model")
        kwargs["model"].save_pretrained(peft_model_path)

        pytorch_model_path = os.path.join(checkpoint_folder, "pytorch_model.bin")
        if os.path.exists(pytorch_model_path):
            os.remove(pytorch_model_path)
        return control
```

Pass the `Seq2SeqTrainingArguments`, model, datasets, data collator, tokenizer, and callback to the [`~transformers.Seq2SeqTrainer`]. You can optionally set `model.config.use_cache = False` to silence any warnings. Once everything is ready, call [`~transformers.Trainer.train`] to start training!

```py
from transformers import Seq2SeqTrainer, TrainerCallback, Seq2SeqTrainingArguments, TrainerState, TrainerControl

trainer = Seq2SeqTrainer(
    args=training_args,
    model=model,
    train_dataset=common_voice["train"],
    eval_dataset=common_voice["test"],
    data_collator=data_collator,
    tokenizer=processor.feature_extractor,
    callbacks=[SavePeftModelCallback],
)
model.config.use_cache = False
trainer.train()
```

## Evaluate

[Word error rate](https://huggingface.co/spaces/evaluate-metric/wer) (WER) is a common metric for evaluating ASR models. Load the WER metric from 🤗 Evaluate:

```py
import evaluate

metric = evaluate.load("wer")
```

Write a loop to evaluate the model performance. Set the model to evaluation mode first, and write the loop with [`torch.cuda.amp.autocast()`](https://pytorch.org/docs/stable/amp.html) because `int8` training requires autocasting. Then, pass a batch of examples to the model to evaluate. Get the decoded predictions and labels, and add them as a batch to the WER metric before calling `compute` to get the final WER score:

```py
from torch.utils.data import DataLoader
from tqdm import tqdm
import numpy as np
import gc

eval_dataloader = DataLoader(common_voice["test"], batch_size=8, collate_fn=data_collator)

model.eval()
for step, batch in enumerate(tqdm(eval_dataloader)):
    with torch.cuda.amp.autocast():
        with torch.no_grad():
            generated_tokens = (
                model.generate(
                    input_features=batch["input_features"].to("cuda"),
                    decoder_input_ids=batch["labels"][:, :4].to("cuda"),
                    max_new_tokens=255,
                )
                .cpu()
                .numpy()
            )
            labels = batch["labels"].cpu().numpy()
            labels = np.where(labels != -100, labels, tokenizer.pad_token_id)
            decoded_preds = tokenizer.batch_decode(generated_tokens, skip_special_tokens=True)
            decoded_labels = tokenizer.batch_decode(labels, skip_special_tokens=True)
            metric.add_batch(
                predictions=decoded_preds,
                references=decoded_labels,
            )
    del generated_tokens, labels, batch
    gc.collect()
wer = 100 * metric.compute()
print(f"{wer=}")
```

## Share model

Once you're happy with your results, you can upload your model to the Hub with the [`~transformers.PreTrainedModel.push_to_hub`] method:

```py
model.push_to_hub("your-name/int8-whisper-large-v2-asr")
```

## Inference

Let's test the model out now!

Instantiate the model configuration from [`PeftConfig`], and from here, you can use the configuration to load the base and [`PeftModel`], tokenizer, processor, and feature extractor. Remember to define the `language` and `task` in the tokenizer, processor, and `forced_decoder_ids`:

```py
from peft import PeftModel, PeftConfig

peft_model_id = "smangrul/openai-whisper-large-v2-LORA-colab"
language = "Marathi"
task = "transcribe"
peft_config = PeftConfig.from_pretrained(peft_model_id)
model = WhisperForConditionalGeneration.from_pretrained(
    peft_config.base_model_name_or_path, load_in_8bit=True, device_map="auto"
)
model = PeftModel.from_pretrained(model, peft_model_id)
tokenizer = WhisperTokenizer.from_pretrained(peft_config.base_model_name_or_path, language=language, task=task)
processor = WhisperProcessor.from_pretrained(peft_config.base_model_name_or_path, language=language, task=task)
feature_extractor = processor.feature_extractor
forced_decoder_ids = processor.get_decoder_prompt_ids(language=language, task=task)
```

Load an audio sample (you can listen to it in the [Dataset Preview](https://huggingface.co/datasets/stevhliu/dummy)) to transcribe, and the [`~transformers.AutomaticSpeechRecognitionPipeline`]:

```py
from transformers import AutomaticSpeechRecognitionPipeline

audio = "https://huggingface.co/datasets/stevhliu/dummy/resolve/main/mrt_01523_00028548203.wav"
pipeline = AutomaticSpeechRecognitionPipeline(model=model, tokenizer=tokenizer, feature_extractor=feature_extractor)
```

Then use the pipeline with autocast as a context manager on the audio sample:

```py
with torch.cuda.amp.autocast():
    text = pipe(audio, generate_kwargs={"forced_decoder_ids": forced_decoder_ids}, max_new_tokens=255)["text"]
text
"मी तुमच्यासाठी काही करू शकतो का?"
```
