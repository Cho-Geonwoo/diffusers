<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
-->


# Text-to-image

<Tip warning={true}>

The text-to-image fine-tuning script is experimental. It's easy to overfit and run into issues like catastrophic forgetting. We recommend you explore different hyperparameters to get the best results on your dataset.

</Tip>

Text-to-image models like Stable Diffusion generate an image from a text prompt. This guide will show you how to finetune the [`CompVis/stable-diffusion-v1-4`](https://huggingface.co/CompVis/stable-diffusion-v1-4) model on your own dataset with PyTorch and Flax. All the training scripts for text-to-image finetuning used in this guide can be found in this [repository](https://github.com/huggingface/diffusers/tree/main/examples/text_to_image) if you're interested in taking a closer look.

Before running the scripts, make sure to install the library's training dependencies:

```bash
pip install git+https://github.com/huggingface/diffusers.git
pip install -U -r requirements.txt
```

And initialize an [🤗 Accelerate](https://github.com/huggingface/accelerate/) environment with:

```bash
accelerate config
```

If you have already cloned the repo, then you won't need to go through these steps. Instead, you can pass the path to your local checkout to the training script and it will be loaded from there.

## Hardware requirements

Using `gradient_checkpointing` and `mixed_precision`, it should be possible to finetune the model on a single 24GB GPU. For higher `batch_size`'s and faster training, it's better to use GPUs with more than 30GB of GPU memory. You can also use JAX/Flax for fine-tuning on TPUs or GPUs, which will be covered [below](#flax-jax-finetuning).

You can reduce your memory footprint even more by enabling memory efficient attention with xFormers. Make sure you have [xFormers installed](./optimization/xformers) and pass the `--enable_xformers_memory_efficient_attention` flag to the training script.

xFormers is not available for Flax.

## Upload model to Hub

Store your model on the Hub by adding the following argument to the training script:

```bash
  --push_to_hub
```

## Save and load checkpoints

It is a good idea to regularly save checkpoints in case anything happens during training. To save a checkpoint, pass the following argument to the training script:

```bash
  --checkpointing_steps=500
```

Every 500 steps, the full training state is saved in a subfolder in the `output_dir`. The checkpoint has the format `checkpoint-` followed by the number of steps trained so far. For example, `checkpoint-1500` is a checkpoint saved after 1500 training steps.

To load a checkpoint to resume training, pass the argument `--resume_from_checkpoint` to the training script and specify the checkpoint you want to resume from. For example, the following argument resumes training from the checkpoint saved after 1500 training steps:

```bash
  --resume_from_checkpoint="checkpoint-1500"
```

## Fine-tuning

<frameworkcontent>
<pt>
Launch the [PyTorch training script](https://github.com/huggingface/diffusers/blob/main/examples/text_to_image/train_text_to_image.py) for a fine-tuning run on the [Pokémon BLIP captions](https://huggingface.co/datasets/lambdalabs/pokemon-blip-captions) dataset like this.

Specify the `MODEL_NAME` environment variable (either a Hub model repository id or a path to the directory containing the model weights) and pass it to the [`pretrained_model_name_or_path`](https://huggingface.co/docs/diffusers/en/api/diffusion_pipeline#diffusers.DiffusionPipeline.from_pretrained.pretrained_model_name_or_path) argument.

```bash
export MODEL_NAME="CompVis/stable-diffusion-v1-4"
export dataset_name="lambdalabs/pokemon-blip-captions"

accelerate launch --mixed_precision="fp16"  train_text_to_image.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --dataset_name=$dataset_name \
  --use_ema \
  --resolution=512 --center_crop --random_flip \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --gradient_checkpointing \
  --max_train_steps=15000 \
  --learning_rate=1e-05 \
  --max_grad_norm=1 \
  --lr_scheduler="constant" --lr_warmup_steps=0 \
  --output_dir="sd-pokemon-model" \
  --push_to_hub
```

To finetune on your own dataset, prepare the dataset according to the format required by 🤗 [Datasets](https://huggingface.co/docs/datasets/index). You can [upload your dataset to the Hub](https://huggingface.co/docs/datasets/image_dataset#upload-dataset-to-the-hub), or you can [prepare a local folder with your files](https://huggingface.co/docs/datasets/image_dataset#imagefolder).

Modify the script if you want to use custom loading logic. We left pointers in the code in the appropriate places to help you. 🤗 The example script below shows how to finetune on a local dataset in `TRAIN_DIR` and where to save the model to in `OUTPUT_DIR`:

```bash
export MODEL_NAME="CompVis/stable-diffusion-v1-4"
export TRAIN_DIR="path_to_your_dataset"
export OUTPUT_DIR="path_to_save_model"

accelerate launch train_text_to_image.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --train_data_dir=$TRAIN_DIR \
  --use_ema \
  --resolution=512 --center_crop --random_flip \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --gradient_checkpointing \
  --mixed_precision="fp16" \
  --max_train_steps=15000 \
  --learning_rate=1e-05 \
  --max_grad_norm=1 \
  --lr_scheduler="constant" 
  --lr_warmup_steps=0 \
  --output_dir=${OUTPUT_DIR} \
  --push_to_hub
```

#### Training with multiple GPUs

`accelerate` allows for seamless multi-GPU training. Follow the instructions [here](https://huggingface.co/docs/accelerate/basic_tutorials/launch)
for running distributed training with `accelerate`. Here is an example command:

```bash
export MODEL_NAME="CompVis/stable-diffusion-v1-4"
export dataset_name="lambdalabs/pokemon-blip-captions"

accelerate launch --mixed_precision="fp16" --multi_gpu  train_text_to_image.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --dataset_name=$dataset_name \
  --use_ema \
  --resolution=512 --center_crop --random_flip \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --gradient_checkpointing \
  --max_train_steps=15000 \ 
  --learning_rate=1e-05 \
  --max_grad_norm=1 \
  --lr_scheduler="constant" \
  --lr_warmup_steps=0 \
  --output_dir="sd-pokemon-model" \
  --push_to_hub
```

</pt>
<jax>
With Flax, it's possible to train a Stable Diffusion model faster on TPUs and GPUs thanks to [@duongna211](https://github.com/duongna21). This is very efficient on TPU hardware but works great on GPUs too. The Flax training script doesn't support features like gradient checkpointing or gradient accumulation yet, so you'll need a GPU with at least 30GB of memory or a TPU v3.

Before running the script, make sure you have the requirements installed:

```bash
pip install -U -r requirements_flax.txt
```

Specify the `MODEL_NAME` environment variable (either a Hub model repository id or a path to the directory containing the model weights) and pass it to the [`pretrained_model_name_or_path`](https://huggingface.co/docs/diffusers/en/api/diffusion_pipeline#diffusers.DiffusionPipeline.from_pretrained.pretrained_model_name_or_path) argument.

Now you can launch the [Flax training script](https://github.com/huggingface/diffusers/blob/main/examples/text_to_image/train_text_to_image_flax.py) like this:

```bash
export MODEL_NAME="runwayml/stable-diffusion-v1-5"
export dataset_name="lambdalabs/pokemon-blip-captions"

python train_text_to_image_flax.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --dataset_name=$dataset_name \
  --resolution=512 --center_crop --random_flip \
  --train_batch_size=1 \
  --max_train_steps=15000 \
  --learning_rate=1e-05 \
  --max_grad_norm=1 \
  --output_dir="sd-pokemon-model" \
  --push_to_hub
```

To finetune on your own dataset, prepare the dataset according to the format required by 🤗 [Datasets](https://huggingface.co/docs/datasets/index). You can [upload your dataset to the Hub](https://huggingface.co/docs/datasets/image_dataset#upload-dataset-to-the-hub), or you can [prepare a local folder with your files](https://huggingface.co/docs/datasets/image_dataset#imagefolder).

Modify the script if you want to use custom loading logic. We left pointers in the code in the appropriate places to help you. 🤗 The example script below shows how to finetune on a local dataset in `TRAIN_DIR`:

```bash
export MODEL_NAME="duongna/stable-diffusion-v1-4-flax"
export TRAIN_DIR="path_to_your_dataset"

python train_text_to_image_flax.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --train_data_dir=$TRAIN_DIR \
  --resolution=512 --center_crop --random_flip \
  --train_batch_size=1 \
  --mixed_precision="fp16" \
  --max_train_steps=15000 \
  --learning_rate=1e-05 \
  --max_grad_norm=1 \
  --output_dir="sd-pokemon-model" \
  --push_to_hub
```
</jax>
</frameworkcontent>

## Training with Min-SNR weighting

We support training with the Min-SNR weighting strategy proposed in [Efficient Diffusion Training via Min-SNR Weighting Strategy](https://arxiv.org/abs/2303.09556) which helps to achieve faster convergence
by rebalancing the loss. In order to use it, one needs to set the `--snr_gamma` argument. The recommended
value when using it is 5.0. 

You can find [this project on Weights and Biases](https://wandb.ai/sayakpaul/text2image-finetune-minsnr) that compares the loss surfaces of the following setups:

* Training without the Min-SNR weighting strategy
* Training with the Min-SNR weighting strategy (`snr_gamma` set to 5.0)
* Training with the Min-SNR weighting strategy (`snr_gamma` set to 1.0)

For our small Pokemons dataset, the effects of Min-SNR weighting strategy might not appear to be pronounced, but for larger datasets, we believe the effects will be more pronounced.

Also, note that in this example, we either predict `epsilon` (i.e., the noise) or the `v_prediction`. For both of these cases, the formulation of the Min-SNR weighting strategy that we have used holds. 

<Tip warning={true}>

Training with Min-SNR weighting strategy is only supported in PyTorch.

</Tip>

## LoRA

You can also use Low-Rank Adaptation of Large Language Models (LoRA), a fine-tuning technique for accelerating training large models, for fine-tuning text-to-image models. For more details, take a look at the [LoRA training](lora#text-to-image) guide.

## Inference

Now you can load the fine-tuned model for inference by passing the model path or model name on the Hub to the [`StableDiffusionPipeline`]:

<frameworkcontent>
<pt>
```python
from diffusers import StableDiffusionPipeline

model_path = "path_to_saved_model"
pipe = StableDiffusionPipeline.from_pretrained(model_path, torch_dtype=torch.float16, use_safetensors=True)
pipe.to("cuda")

image = pipe(prompt="yoda").images[0]
image.save("yoda-pokemon.png")
```
</pt>
<jax>
```python
import jax
import numpy as np
from flax.jax_utils import replicate
from flax.training.common_utils import shard
from diffusers import FlaxStableDiffusionPipeline

model_path = "path_to_saved_model"
pipe, params = FlaxStableDiffusionPipeline.from_pretrained(model_path, dtype=jax.numpy.bfloat16)

prompt = "yoda pokemon"
prng_seed = jax.random.PRNGKey(0)
num_inference_steps = 50

num_samples = jax.device_count()
prompt = num_samples * [prompt]
prompt_ids = pipeline.prepare_inputs(prompt)

# shard inputs and rng
params = replicate(params)
prng_seed = jax.random.split(prng_seed, jax.device_count())
prompt_ids = shard(prompt_ids)

images = pipeline(prompt_ids, params, prng_seed, num_inference_steps, jit=True).images
images = pipeline.numpy_to_pil(np.asarray(images.reshape((num_samples,) + images.shape[-3:])))
image.save("yoda-pokemon.png")
```
</jax>
</frameworkcontent>
