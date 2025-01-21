# DK-BEHRT

BERT to explore how representation of diseases are learned inside and outside of EHR datasets.

## Getting started

### Requirements
Python 3.10 is required due to the `icdmappings` package. This project relies on huggingface models and trainers. Please ensure that an appropriate torch backend and the `transformers`/`evaluate` packages are installed.

```bash
pandas
numpy
icdmappings
pickle
tqdm
scikit-learn
torch
transformers
evaluate
```

### Project setup

The scripts need to be told the project root and the dataset root directories in the `paths.py` folder such as:

```python
mimic_root = '/home/ulzee/gpt/data/mimic4/mimiciv/2.2'
project_root = '/home/ulzee/gpt/icdbert'
```



## 0. Splits and data parsing

The patient ids of the train, val, and test splits need to be provided in `files/`. The project will provide appropriate data splits which should be identical to ones found in [https://github.com/ulzee/ehr-outcomes](https://github.com/ulzee/ehr-outcomes).

The following scripts must be run to convert ICD9 to ICD10 codes in MIMIC and save a parsed version of the dataset.

```bash
python ./scripts/convert_icd10.py 5
python ./scripts/parse_diagnoses.py 5
```

The option `5` specifies how many digits of ICD10 will be kept (once chosen this should be kept consistent with later steps).


## 1. Create a tokenizer capable of parsing ICD codes

The following script will create a tokenizer that can interpret codes that are found in the parsed dataset from the previous step.

```bash
python ./scripts/create_bert_tokenizer.py 5
```

## 2. (optional) Obtain external embeddings

To experiment with the embedding-attention architecture, an embeddig lookup table for tokens should be created. The lookup is expected to be a python dictionary of token to embedding vector and should be saved in pickle format.
The pikcle file can be anywhere and can be loaded by the scripts later on.

## 3. Pretraining

### Base BERT
```bash
python train_bert.py --mode base --lr 1e-4 --layers 4
```

### Embedding Attention BERT
```bash
python train_bert.py --mode attn --lr 1e-4 --layers 4 --use_embedding ./icd10/embeddings/hug/microsoft--biogpt_hidden_collated.pk
```

If you're using hoffman, the scripts in `./scripts/hoffman` may be useful:
```bash
./scripts/hoffman/gpu.sh train_bert --mode attn --lr 5e-4 --layers 4 --disable_visible_devices --use_embedding /u/home/u/ulzee/gpt/data/icd10/embeddings/hug/microsoft--biogpt_hidden_collated.pk
```

## 4. Finetune

Since the data may be formatted slightly differently for downstream tasks, the [https://github.com/ulzee/ehr-outcomes](https://github.com/ulzee/ehr-outcomes) project needs to be setup in a folder adjacent to the current project to explore finetuning.

### Finetune pretrained model (default approach)

Once a model is trained, it can be finetuned to evaluate downstream tasks as prepared in `ehr-outcomes`. For example, the following command will finetune a snapshot of the embedding-attention BERT for the mortality task. The data in this case will be loaded from `ehr-outcomes` although the snapshot will be in the current project folder.

```bash
./scripts/hoffman/gpu.sh finetune --load ./runs/bert-attn-cr5-lr0.0005-e192-layers4-h1_Ot53O/checkpoint-50000 --task mortality --lr 1e-5
```

### Finetune ignoring pretraining

The same model architecture can also be fit from scratch by ignoring the pretrained weights with the `--scratch` command. In this case the snapshot will still be loaded and the model arch will be inferred from the snapshot.

```bash
./scripts/hoffman/gpu.sh finetune --load ./runs/bert-attn-cr5-lr0.0005-e192-layers4-h1_Ot53O/checkpoint-50000 --task mortality --lr 1e-5 --scratch
```

### Finetune last layer only

In some cases one may wish to only fit the classification layer of BERT. This can be down with the `--freeze` command.

```bash
./scripts/hoffman/gpu.sh finetune --load ./runs/bert-attn-cr5-lr0.0005-e192-layers4-h1_Ot53O/checkpoint-50000 --task mortality --lr 1e-5 --freeze
```

## 5. Predict and score

Given any of the finetuning commands above, pass the `--predict` option once finetuning finishes (or is terminated) to score the model's predictions on the test set:

```bash
./scripts/hoffman/gpu.sh finetune --load ./runs/bert-attn-cr5-lr0.0005-e192-layers4-h1_Ot53O/checkpoint-50000 --task mortality --lr 1e-5 --predict
```
