# mT5: Multilingual T5

多语言T5(mT5)是一种大规模的预训练文本到文本transformer模型，
遵循与以下类似的方法进行训练：[T5](https://github.com/google-research/text-to-text-transfer-transformer).
此repo是源自论文 [mT5 paper][paper].

## Table of Contents

* [Languages covered](#languages-covered)
* [Results](#results)
* [Usage](#usage)
  * [Training](#training)
  * [Fine-Tuning](#fine-tuning)
* [Released Model Checkpoints](#released-model-checkpoints)
* [How to Cite](#how-to-cite)

## Languages covered

mT5 在 [mC4](https://www.tensorflow.org/datasets/catalog/c4#c4multilingual_nights_stay) 语料上预训练, 包含 101 种语言:

Afrikaans, Albanian, Amharic, Arabic, Armenian, Azerbaijani, Basque,
Belarusian, Bengali, Bulgarian, Burmese, Catalan, Cebuano, Chichewa, Chinese中文,
Corsican, Czech, Danish, Dutch, English, Esperanto, Estonian, Filipino,
Finnish, French, Galician, Georgian, German, Greek, Gujarati, Haitian Creole,
Hausa, Hawaiian, Hebrew, Hindi, Hmong, Hungarian, Icelandic, Igbo, Indonesian,
Irish, Italian, Japanese, Javanese, Kannada, Kazakh, Khmer, Korean, Kurdish,
Kyrgyz, Lao, Latin, Latvian, Lithuanian, Luxembourgish, Macedonian, Malagasy,
Malay, Malayalam, Maltese, Maori, Marathi, Mongolian, Nepali, Norwegian,
Pashto, Persian, Polish, Portuguese, Punjabi, Romanian, Russian, Samoan,
Scottish Gaelic, Serbian, Shona, Sindhi, Sinhala, Slovak, Slovenian, Somali,
Sotho, Spanish, Sundanese, Swahili, Swedish, Tajik, Tamil, Telugu, Thai,
Turkish, Ukrainian, Urdu, Uzbek, Vietnamese, Welsh, West Frisian, Xhosa,
Yiddish, Yoruba, Zulu.

## Results
截至2020年11月，mT5在许多跨语言的NLP任务上均实现了最先进的性能。例如，
[XTREME](https://github.com/google-research/xtreme) zero-shot 分类,
结构化的预测和问答任务(显示F1 Score) 

| Model | XNLI | PAWS-X | WikiAnn-NER | XQuAD | MLQA | TyDiQA-GoldP |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| mBERT | 65.4 | 81.9 | 62.2 | 64.5 | 61.4 | 59.7 |
| XLM | 69.1 | 80.9 | 61.2 | 59.8 | 48.5 | 43.6 |
| InfoXLM | 81.4 | - | - | - | 73.6 | - |
| X-STILTs | 80.4 | 87.7 | 64.7 | 77.2 | 72.3 | 76.0 |
| XLM-R | 79.2 | 86.4 | 65.4 | 76.6 | 71.6 | 65.1 |
| VECO | 79.9 | 88.7 | 65.7 | 77.3 | 71.7 | 67.6 |
| RemBERT | 80.8 | 87.5 | **70.1** | 79.6 | 73.1 | 77.0 |
| mT5-Small | 67.5 | 82.4 | 50.5 | 58.1 | 54.6 | 35.2 |
| mT5-Base | 75.4 | 86.4 | 55.7 | 67.0 | 64.6 | 57.2 |
| mT5-Large | 81.1 | 88.9 | 58.5 | 77.8 | 71.2 | 69.9 |
| mT5-XL | 82.9 | 89.6 | 65.5 | 79.5 | 73.5 | 75.9 |
| mT5-XXL | **85.0** | **90.0** | 69.2 | **82.5** | **76.0** | **80.8** |

## Usage

### Training

要运行此代码，您需要安装  [t5 library](https://pypi.org/project/t5/). 训练的一般说明 
微调，评估和导出模型以进行推理，可以在一下repo中找出
[t5 repo](https://github.com/google-research/text-to-text-transfer-transformer). 
提供了`t5_mesh_transformer` 命令，为了使用该库中其他mT5任务，请从该目录运行并添加命令 
`--module_import="multilingual_t5.tasks"`. 也支持  [mT5 in
HuggingFace](https://huggingface.co/transformers/model_doc/mt5.html); 参见T5 repository中的说明 
[here](https://github.com/google-research/text-to-text-transfer-transformer#t5models).

在平台上训练`mT5-Large`模型 
[mc4](https://www.tensorflow.org/datasets/catalog/c4#c4multilingual_nights_stay)
如本文所述从头开始执行任务： 

```
export PROJECT=yourproject
export ZONE=yourzone
export BUCKET=yourbucket
export TPU=yourtpu

ctpu up --name=$TPU --project=$PROJECT --zone=$ZONE --tpu-size=v3-256 --tpu-only --noconf

TASK=mc4
MODEL_DIR="${BUCKET}${TASK}"

python -m t5.models.mesh_transformer_main \
  --tpu="${TPU}" \
  --gcp_project="${PROJECT}" \
  --tpu_zone="${ZONE}" \
  --model_dir="${MODEL_DIR}" \
  --gin_file="models/t5.1.1.large.gin" \
  --gin_param="MIXTURE_NAME = '${TASK}'" \
  --gin_param="utils.run.sequence_length = {'inputs': 1024, 'targets': 256}" \
  --gin_param="utils.run.batch_size = ('tokens_per_batch', 1048576)" \
  --gin_param="utils.run.learning_rate_schedule=@learning_rate_schedules.rsqrt_no_ramp_down" \
  --gin_param="run.train_steps = 1000000" \
  --gin_param="utils.tpu_mesh_shape.model_parallelism = 1" \
  --gin_param="utils.tpu_mesh_shape.tpu_topology = 'v3-256'" \
  --eval_mode="perplexity_eval" \
  --eval_gin_param="mesh_eval_dataset_fn.num_eval_examples = 10000" \
  --t5_tfds_data_dir="${BUCKET}/t5-tfds" \
  --module_import="multilingual_t5.tasks"
```

### Fine-Tuning

例如，要在XNLI_zeroshot任务上微调mT5-Large模型： 

```
export PROJECT=yourproject
export ZONE=yourzone
export BUCKET=yourbucket
export TPU=yourtpu

ctpu up --name=$TPU --project=$PROJECT --zone=$ZONE --tpu-size=v3-256 --tpu-only --noconf

TASK=xnli_zeroshot
SEQUENCE_LENGTH_GIN=xnli
PRETRAINED_DIR=gs://t5-data/pretrained_models/mt5/large
PRETRAINED_STEPS=1000000
FINETUNE_STEPS=20000
MODEL_DIR="${BUCKET}${TASK}"

# Run fine-tuning
python -m t5.models.mesh_transformer_main \
  --tpu="${TPU}" \
  --gcp_project="${PROJECT}" \
  --tpu_zone="${ZONE}" \
  --model_dir="${MODEL_DIR}" \
  --gin_file="${PRETRAINED_DIR}/operative_config.gin" \
  --gin_file="sequence_lengths/${SEQUENCE_LENGTH_GIN}.gin" \
  --gin_param="utils.tpu_mesh_shape.tpu_topology = 'v3-256'" \
  --gin_param="MIXTURE_NAME = '${TASK}'" \
  --gin_param="utils.run.train_steps=$((PRETRAINED_STEPS+FINETUNE_STEPS))" \
  --gin_param="utils.run.init_checkpoint='${PRETRAINED_DIR}/model.ckpt-${PRETRAINED_STEPS}'" \
  --t5_tfds_data_dir="${BUCKET}/t5-tfds" \
  --module_import="multilingual_t5.tasks" \
  --gin_location_prefix="multilingual_t5/gin/"
```

其余实验显示在[tasks.py](multilingual_t5 / tasks.py)文件中。

## Released Model Checkpoints

我们针对[paper][paper]中描述的预训练模型发布了以下checkpoints： 
We have released the following checkpoints for pre-trained models described in our [paper][paper]:

* **mT5-Small** (300 million parameters): [gs://t5-data/pretrained_models/mt5/small](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/small/)
* **mT5-Base** (600 million parameters): [gs://t5-data/pretrained_models/mt5/base](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/base/)
* **mT5-Large** (1.2 billion parameters): [gs://t5-data/pretrained_models/mt5/large](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/large/)
* **mT5-XL** (4 billion parameters): [gs://t5-data/pretrained_models/mt5/xl](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/xl/)
* **mT5-XXL** (13 billion parameters): [gs://t5-data/pretrained_models/mt5/xxl](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/xxl/)

# How to Cite

If you extend or use this work, please cite the [paper][paper] where it was
introduced:

```
@misc{xue2020mt5,
    title = {{mT5}: A massively multilingual pre-trained text-to-text transformer},
    author = {Linting Xue and Noah Constant and Adam Roberts and Mihir Kale and Rami Al-Rfou and Aditya Siddhant and Aditya Barua and Colin Raffel},
    year = {2020},
    eprint = {2010.11934},
    archivePrefix = {arXiv},
    primaryClass = {cs.CL}
}
```

[paper]: https://arxiv.org/abs/2010.11934
