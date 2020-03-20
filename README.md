# Data and scripts for coronavirus project. 

# Please update this README when adding any new files. 

## data/
- `AID1706_binarized_sars.csv` is our main training data, with 1 for the 400 positives and 0 for the 300K negatives. 
- the test sets we want to predict on for now are:
  - `broad_repurposing_library.csv`, `external_library.csv` (the 866 molecules that Robert Malone sent us)
  - `expanded_external_library.csv` (the ~2500 molecules that Robert Malone sent us afterward, which don't seem to be a superset of the previous).
- `corona_literature.csv` and `corona_literature_idex.csv` contain FDA-approved drugs that have been studied in the (generic) coronavirus literature. These are not guaranteed to be effective against any targets; they simply appear in the literature. 
  - `corona_literature.csv` was generated through direct name matches at https://www.cureffi.org/wp-content/uploads/2013/10/drugs.txt
  - `corona_literature_idex.csv` was generated through the PubChem idex service and may contain multiple SMILES for generic drug names. 
  - `evaluation_set_v2.csv` is the combination of the broad repurposing library with Robert Malone's 60 reference actives, with those present in the training data removed. Reference actives are labeled as 1 while the broad library is labeled as 0. 
  - `AID1706_binarized_sars.csv` is the training data augmented with the evaluation set's positives, for use when making final predictions for lab testing. 
  - `mpro_xchem_all.csv` is ~1200 molecules with 78 hits on SARS-CoV-2 Mpro/3CLpro from https://www.diamond.ac.uk/covid-19/for-scientists/Main-protease-structure-and-XChem/Downloads.html
  - `PLpro.csv` contains training data for the PLpro target combined from the AID485353 and AID652038 datasets. 

# 316_AID1706_preds/ 
The most recent model predictions on each of the 3 test sets using the model trained on only the AID1706 dataset. Predictions using 5-model chemprop ensemble with no hyperparameter opt at the moment. For reference, benchmarking non-ensembled versions of this model on scaffold split (5 folds) gets 0.8383 +/- 0.0283 AUC. This model got .963 AUC on evaluation_set_v2.csv. 

# conversions/
Files for converting between smiles/cid/name. Obtained from https://pubchem.ncbi.nlm.nih.gov/idexchange/idexchange.cgi

# similarity_computations/ 
The nearest neighbor computations from each test set to the training set. 

# scripts/ 
Various processing scripts for reuse/reproducibility.

# plots/ and statistics/ 
tsne plots of data distributions from different sets and some numerical stats about the data. 

# interpretation/

Rationale extraction (https://github.com/chemprop/chemprop/blob/master/interpret.py) from the positives of the combined AID datasets and from the Broad Repurposing Hub using a model trained on the combined AID datasets. The commands used to generate the rationales are in `commands.txt` while the `.csv` files contain the rationale SMILES and the `.png` files contain t-SNE visualizations of the original SMILES and rationales.

# AID1706_splits.zip
Cross-validation splits for the combined_binarized_sars.csv file. These are generated by scripts/create_crossval_splits.py in chemprop: https://github.com/chemprop/chemprop/blob/master/scripts/create_crossval_splits.py. There are 5 splits for each of random and scaffold, generated by making 5 folds and then picking 1 test fold and 1 val fold for each split. These are splits for model benchmarking purposes. When training a model for prediction of compounds to test in the lab, you should make sure to use the full data and not withhold extra data for testing. 

# raw_data/
Original raw data files and format conversions. 

# old/
Older versions of files from when we combined AID1706 data with other data that was unhelpful. 

# Experiment Commands

These commands are for running experiments using [chemprop](https://github.com/chemprop/chemprop) and should be run from the main directory in the chemprop repo. You made need to modify some paths (such as `save_dir`) depending on your directory structure.

## Generating RDKit features

To speed up experiments, you can pre-generate RDKit features using the script `save_features.py` in `chemprop/scripts`. You should run these two commands:

```
python save_features.py
    --data_path ../../coronavirus_data/data/AID1706_binarized_sars.csv \
    --save_path ../../coronavirus_data/features/AID1706_binarized_sars.csv \
    --features_generator rdkit_2d_normalized
```

```
python save_features.py
    --data_path ../../coronavirus_data/data/evaluation_set_v2.csv \
    --save_path ../../coronavirus_data/features/evaluation_set_v2.csv \
    --features_generator rdkit_2d_normalized
```

By default this will run feature generation using parallel processing. On occasion the parallel processing gets stuck near the end of feature generation, so if this happens, just kill the process and restart with the `--sequential` flag. This will pick up where the parallel version stopped and will finish correctly.


## Training and testing on AID1706

Train and test on 5-fold CV, scaffold-based splits, of AID1706 (you need to unzip AID1706_splits.zip).

```
python train.py \
    --data_path ../coronavirus_data/data/AID1706_binarized_sars.csv \
    --dataset_type classification \
    --save_dir ../coronavirus_data/ckpt/AID1706 \
    --features_path ../coronavirus_data/features/AID1706_binarized_sars.npz \
    --no_features_scaling \
    --split_type predetermined \
    --folds_file ../coronavirus_data/splits/scaffold/split_0.pkl \
    --val_fold_index 1 \
    --test_fold_index 2 \
    --quiet
```

and repeat for each of the 5 splits (numbered 0 to 4). 

## Training on AID1706 and testing on Broad/external validation set

Train on 90%/10%/0% random split of AID1706 and test on combined Broad/external validation set where the external validation molecules are treated as positives and the Broad molecules are treated as negatives.

```
python train.py \
    --data_path ../coronavirus_data/data/AID1706_binarized_sars.csv \
    --dataset_type classification \
    --save_dir ../coronavirus_data/ckpt/AID1706 \
    --features_path ../coronavirus_data/features/AID1706_binarized_sars.npz \
    --no_features_scaling \
    --split_sizes 0.9 0.1 0.0 \
    --num_folds 5 \
    --quiet
```

```
python train.py \
    --data_path ../coronavirus_data/data/evaluation_set_v2.csv \
    --dataset_type classification \
    --features_path ../coronavirus_data/features/evaluation_set_v2.npz \
    --no_features_scaling \
    --quiet \
    --test
```

## Class balance

To run experiments with class balance, switch to the `class_weights` branch of chemprop (`git checkout class_weights`) and add the `--class_balance` flag during training.
