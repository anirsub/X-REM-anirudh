# X-REM

## Preparation

Make sure to download the github repositories below: 

* [ifcc](https://github.com/ysmiura/ifcc)
    * download model_medrad_19k.tar.gz by running `resources/download.sh`
* [CheXbert](https://github.com/stanfordmlgroup/CheXbert)
    * download the [pretrained weights](https://stanfordmedicine.box.com/s/c3stck6w6dol3h36grdc97xoydzxd7w9) 
* [CXR-RePaiR](https://github.com/rajpurkarlab/CXR-RePaiR)
* [CXR-Report-Metric](https://github.com/rajpurkarlab/CXR-Report-Metric)

Once you clone the repositories, add the following files to the respective folders:

```
mv M2TransNLI.py example_m2trans_nli.csv m2trans_nli_filter.py <path to ifcc folder>
mv compute_avg_score.py prepare_df.py <path to CXR-Report-Metric folder>
```

As we made multiple edits to the ALBEF directory, please refer to the ALBEF directory uploaded here instead of cloning a new one. 

Download our [pretrained checkpoints](https://drive.google.com/file/d/11UorBbh5cOcDfIzy_lCgMdn0zThvpDbp/view?usp=sharing) here!
   

## Data

Download the train/test reports and images from [MIMIC-CXR](https://physionet.org/content/mimic-cxr/2.0.0/). You may have to request for an approval to access the files. We also provide the ids used in our study in `mimic_pretrain_study_id.csv`, `mimic_pretrain_study_id.csv`, and `mimic_challenge_dicom_id.csv`. 

## Environment

Create a conda environment for X-REM:

```
conda env create -f environment.yml -n X-REM-env
```

Activate the environment:

```
conda activate X-REM-env
```

## Preprocessing
Refer to the data preprocessing step in [CXR-RePaiR](https://github.com/rajpurkarlab/CXR-RePaiR) to acquire `mimic_train_impressions.csv`, `mimic_test_impressions.csv`, and `cxr.h5`.  

Preprocess data to be used for pre-training ALBEF:

```
python3 -m preprocess_mimic.py --data_dir <path to MIMIC>  --impressions_train_path <path to mimic_train_impressions.csv> --impressions_test_path <path to mimic_test_impressions.csv> --out_dir <path to store the processed data>
```

## Training

Pretrain ALBEF:
```
cd ALBEF 
python3 -m torch.distributed.launch --nproc_per_node=4 --use_env Pretrain.py --config configs/Pretrain.yaml --output_dir <output path>  --checkpoint <path to pretrained ALBEF checkpoint>  --resume true
```

Generate the train file for finetuning ALBEF on image-text matching task (visual entailment task with a binary (entailment/non-entailment) classification):
```
python generate_ve_train.py
```
Finetune the ALBEF model on image-text matching task:
```
cd ALBEF 
python3 -m torch.distributed.launch --nproc_per_node=4 --use_env VE.py --config ./configs/VE.yaml --output_dir <output path> --checkpoint <path to checkpoint>
```

## Inference

```
cd ALBEF
python3 XREM-pipeline.py --save_path before_nli.csv
cd ../ifcc
conda activate m2trans
python3 m2trans_nli_filter.py --input_path before_nli.csv --save_path after_nli.csv
conda deactivate
```

This generates a dataframe containing two columns: "Report Impression" column holding reports before applying the nli filter, and 
"filtered" column containing the filtered reports. 
    
## Evaluation


```
cd CXR-Report-Metric
conda activate <environment name for CXR-Report-Metric>
python prepare_df.py --fpath <input path> --opath <output path>
python test_metric.py
python3 compute_avg_score.py --fpath <input path>
```
Refer to [CXR-Report-Metric](https://github.com/rajpurkarlab/CXR-Report-Metric) for a detailed explanation on the metric.

## Supplementary Experiments

Generate reports without using the image-text matching scores: 
```
cd ALBEF
python3 XREM-pipeline.py --albef_retrieval_delimiter ' ' --save_path no_itm.csv --albef_retrieval_top_k 2 --albef_itm_top_k 0
```

Generate reports without the nli filter:
```
cd ALBEF
python3 XREM-pipeline.py --albef_ve_delimiter ' ' --save_path no_nli.csv --albef_itm_top_k 1
```

Replace the nli filter with bertscore as the metric for measuring redundancy:
```
cd ALBEF
python3 XREM-pipeline.py --save_path before_bertscore.csv
python3 bertscore_filter.py --input_path before_bertscore.csv --save_path after_bertscore.csv
```
