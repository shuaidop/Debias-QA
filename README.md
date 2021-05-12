# LUKE-QA-bias-analysis


To produce the result as in `output/result.json`, you should follow the instructions step by step here:

1. Download all required data from link [google drive](https://drive.google.com/drive/folders/1peLPm0rGUmKuE2MeYWVN-3SDDBGUjxbL?usp=sharing) using the CMU account and store them under the directory `data`.

2. Download the LUKE pretrained model and entity data from [google drive](https://drive.google.com/drive/folders/1Gu9BI9w6twOT70Ha2uULuhobaBO21nKy?usp=sharing) using the CMU account and store them under the main directory `LUKE-QA-bias-analysis`.

3. Download the model checkpoints at [google drive](https://drive.google.com/drive/folders/1KTxIjnaLpD5m_23QCsaWxiUSAwxoDZ_U?usp=sharing) using the CMU account and store them under the main directory `LUKE-QA-bias-analysis`.

    ** The file `python_model_reproduce.bin` should also be downloaded if you only want to train the model by yourself instead of loading the weight (Since the code will check the existence of the files even not using them)

4. Install all required packages using `pip3 install -r requirements.txt`, and PyTorch version should be 1.2.0 and CUDA version should be 10

## Fintunning Model
Run the following command for finetunning on SQuAD 1.1 dataset (we use the same command as the original paper for fine tunning):
```
    python3 -m examples.cli \
    --num-gpus=1 \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output \
    reading-comprehension run \
    --data-dir=data \
    --no-negative \
    --wiki-link-db-file=enwiki_20160305.pkl \
    --model-redirects-file=enwiki_20181220_redirects.pkl \
    --link-redirects-file=enwiki_20160305_redirects.pkl \
    --train-batch-size=2 \
    --gradient-accumulation-steps=3 \
    --learning-rate=15e-6 \
    --num-train-epochs=2
```

The model's weight will be generated under the `output` directory named `python_model.bin`, you could change the output directory by modifying the `--output-dir` command. 

For training, we used the AWS p3.2xlarge instance (contains a single V100 GPU) and it takes about 8 hours to train with the exact provided setting.

## Evaluate and Reproduce The Paper's Result
To reproducing the paper results using the weight we have trained, run the following command:
```
    python3 -m examples.cli \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output reading-comprehension run \
    --checkpoint-file=pytorch_model_reproduce.bin \
    --no-train --no-negative
```
The beam search results, output predictions, and scores will be stored in the `output` directory with the name of `nbest_predictions_.json`, `predictions_.json`, `results.json`

If you want to use the weights trained by the paper itself, then you could download the file [here](https://drive.google.com/file/d/1097QicHAVnroVVw54niPXoY-iylGNi0K/view?usp=sharing)(compressed) and replace the `--checkpoint-file` with the new file path.

## Generate output
Run the following command to generate the prediction and beam search result for four groups of undespecific questions:
```
    python3 -m examples.cli \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output reading-comprehension run \
    --checkpoint-file=pytorch_model_reproduce.bin \
    --no-train --no-negative --do-unqover --unqover-file=R
```
To change the different groups of undespecific questions you want to predict, using the `--unqover-file` command.

`--unqover-file=R`: use religions dataset

`--unqover-file=E`: use ethnicity dataset

`--unqover-file=G`: use gender dataset

`--unqover-file=N`: use nationality dataset

The final output will be stored in the `output` directory with the name: `nbest_predictions_$groupname_.json` and `predictions_$groupname_.json`

** It takes about 10-20 minutes for the model to generate the output file on the AWS g4dn 2xlarge machine. 

## Get the Data Directly for Bias Analysis
You can also download the file we have generated by our model from [google drive](https://drive.google.com/drive/folders/1vyMeDl5TURGPG9UFUG67GIsi0-EhBe61?usp=sharing).

Bias result reproduction:
```
python3 ./bias_analysis/bias_analysis.py <predcitions file> <(OPTIONAL) Category>
```
(If you use predictions from other models, please make sure the output format is consistent with provided predictions files in our google drive)

Sample output:

```shell
> python3 bias_analysis.py nbest_predictions_Gender_.json Gender

=========================================================
Model bias intensity for Gender = 0.833
=========================================================
```



Or, you can use the .ipynb interactive notebook and follow the instructions to conduct bias analysis.

## reproduced results
By finetuning the model on SQuAD results, we are able to generate the following F1 and exact match scores: 

   LUKE (reported) Exact Match 89.8 F1 95.0

   LUKE (reproduced) Exact Match 89.7 F1 94.9

The reproduced result is of minor differences compared to the reported exact match and F1 scores reported in the paper.



# LUKE-Debias-Result

The debias model can be found under the folder `examples/debias_model`. The `model.py` file contains the pretrain debias model adapted from RoBERTa, and the file `luke_debias_model.py` is the model with residual connection and only adapt from LUKE. 

The setup requirements are the same as in the previous section.

## Pretrain Instruction
The pretrianing gender debias data can be download [here](https://drive.google.com/drive/folders/1YqdU2IKEvUc9PonfSel8pPTOvoWxZHtH?usp=sharing) in the name of `debias_dev.json` and `train_dev.json`. These files should put under the folder `examples/debias_model/data`.

Other pretraining data can also download using the same link with the name of: `eth_<dev/train>.json`,`nation_<dev/train>.json`,`religion_<dev/train>.json`.
In order to use these datasets, you should change the name to `debias_<dev/train>.json` or change the data path in file `examples/reading_comprehension/utils/dataset.py` under the class `debiasProcessor()`. 

For changing the model parameters, you could edit the file `examples/debias_model/config.py`.

Due to the computation limitation, in our experiments, we only have time to train the gender debias model. The pretraining process takes overnight on AWS g4dn 2xlarge machine.

The command line for pretraining (make sure you are under the directory of `example/debias_model/`): 

```
    python3 train.py

```
If you would like to change the training parameters, the arguments are:
```
    python3 train.py \
    --batch_size 80 \
    --epochs5 \
    --lr 1e-3 \
    --cuda 0

```
** CUDA is the index of the CUDA device, if cuda<0 means only use CPU


## Pretrained Output
The pretrained output file will be stored under the same directory `examples/debias_model/`, you should move it to `LUKE/` with the same name before fine tuning.

From this [link](https://drive.google.com/drive/folders/1YqdU2IKEvUc9PonfSel8pPTOvoWxZHtH?usp=sharing), we provide the model weight that we have already trained, and you could put it under the `LUKE/` directory before fine tuning.


## Fine tunning

For fine-tuning you could directly run the command:

```
    python3 -m examples.cli \
    --num-gpus=1 \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output \
    reading-comprehension run \
    --data-dir=data \
    --no-negative \
    --wiki-link-db-file=enwiki_20160305.pkl \
    --model-redirects-file=enwiki_20181220_redirects.pkl \
    --link-redirects-file=enwiki_20160305_redirects.pkl \
    --train-batch-size=1 \
    --gradient-accumulation-steps=3 \
    --learning-rate=15e-6 \
    --num-train-epochs=2
```
Due to the computation limitation, we only use a batch size of 1. But if you have more computation resources, you could increase the batch size to run faster.
## Evaluation on SQuAD dataset
To evaluate on squad dataset, you can run the following command:
```
    python3 -m examples.cli \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output reading-comprehension run \
    --checkpoint-file=pytorch_model_reproduce.bin \
    --no-train --no-negative
```
The beam search results, output predictions, and scores will be stored in the `output` directory with the name of `nbest_predictions_.json`, `predictions_.json`, `results.json`

## Generate the bais prediction result
For bias evaluation, you should run the following command after finetuning the model:
```
    python3 -m examples.cli \
    --model-file=luke_large_500k.tar.gz \
    --output-dir=output reading-comprehension run \
    --checkpoint-file=pytorch_model_reproduce.bin \
    --no-train --no-negative --do-unqover --unqover-file=G
```
The final output will be stored in the `output` directory with the name: `nbest_predictions_Gender_.json` and `predictions_Gender_.json`

** If you would like to evaluate the pretrained models for other bias, you could change the argument `--unqover-file` as the same as in the previous section.

## Result

The debias model performance can be found in file `debias_output/results.json`, where you could see the F1 score and exact match. The orginal LUKE's result is in file `output/results.json`.

The beam search results for gender bias evaluation can be downloaded from this [link](https://drive.google.com/file/d/1TtuQ2Mj_SgsTNuHOreB2LsKjKhfe8fqL/view?usp=sharing), which can be used for calculating the bias score.
