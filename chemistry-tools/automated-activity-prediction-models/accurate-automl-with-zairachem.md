---
description: We present ZairaChem, Ersilia's modeling pipeline for chemistry data
---

# Accurate AutoML with ZairaChem

[ZairaChem](https://github.com/ersilia-os/zaira-chem) is [Ersilia](https://ersilia.io)'s AutoML tool for supervised learning of small molecule property data. In the field of computer-aided drug discovery, this task is known as Quantitiative Structure Activity/Property Relationship Modeling (QSAR/QSPR).

ZairaChem offers a relatively complex modeling pipeline, offering robust performance over a wide set of tasks. If, instead, you want to build quick baseline models, we recommend to check [LazyQSAR](https://github.com/ersilia-os/lazy-qsar), the light-weight modeling tool of Ersilia.

At the moment, ZairaChem works in these two simple types of tasks:

* Single binary classification
* Single regression

## Installation

ZairaChem can be installed as follows:

```bash
git clone https://github.com/ersilia-os/zaira-chem.git
cd zaira-chem
bash install_linux.sh
```

A Conda environment called `zairachem` will be created. Start by activating this environment:

```bash
conda activate zairachem
```

Check that ZairaChem has been installed properly. The following will display the command-line interface (CLI) options.

```bash
zairachem --help
```

## Quick start

To get started, let's use a classification task from [Therapeutic Data Commons](https://tdcommons.ai/).

```
zairachem example --classification --file_name input.csv
```

This file can be split into train and test sets.

```
zairachem split -i input.csv
```

The command above will generate two files in the current folder, named `train.csv` and `test.csv`. By default, the train:test ratio is 80:20.

### Fit

You can train a model as follows:

```
zairachem fit -i train.csv -m model
```

This command will run the full ZairaChem pipeline and produce a `model` folder with processed data, model checkpoints, and reports.

### Predict

You can then run predictions on the test set:

```
zairachem predict -i test.csv -m model -o test
```

ZairaChem will run predictions using the checkpoints stored in `model` and store results in the `test` directory. Several performance plots will be generated alongside prediction outputs.

## Pipeline steps

The ZairaChem pipeline consists of the following steps:

1. Session
2. Setup
3. Describe
4. Estimate
5. Pool
6. Report
7. Distill (mainly based on the [Olinda](https://github.com/ersilia-os/olinda) package; integration in progress)
8. Finish

### Session

You can start a ZairaChem training session as follows:

```
zairachem session --fit -i train.csv -m model
```

Likewise, you can start a prediction session:

```
zairachem session --predict -i test.csv -m model -o test
```

The `session` command will simply create the necessary folders and a session log.

### Setup

In this step, data preparation is done, including:

* Identification of relevant columns (compound identifier, SMILES, and value) in the input file.
* Chemical structure standardization.
* Deduplication.
* Data balancing and augmentation using a reference set of molecules (e.g. ChEMBL).
* Binarization when a cutoff is specified.
* Transformation (Guassianization) of continuous data.
* Folds and clusters assignments.

Give an initialized session (fit or predict), data preparation will be done accordingly. To perform this step, simply run:

```
zairachem setup
```

Most data generated in the `setup` step will be stored in `model/data` (fit) or `test/data` (predict). The most important file in this folder is `data.csv`, containg the result of the data preparation step. Other files are generated, like `mapping.csv`, which match `data.csv` to the row indices of the input file.

### Describe

In the `describe` step, small molecule descriptors are calculated. ZairaChem provides a set of default descriptors, including the [Chemical Checker signaturizer](https://bioactivitysignatures.org), Grover embeddings and Morgan fingerprints and Mordred descriptors.

Other descriptors can be easily incorporated thanks to the [Ersilia Model Hub](https://ersilia.io/model-hub). They can be specified in a `parameters.json` file.

Several operations are performed for each of the descriptors, including:

* Calculation of descriptors for each molecule using the Ersilia Model Hub.
* Removal of constant-value columns and columns with a high degree of missing values.
* Imputation of the rest missing values.
* Robust scaling of contiuous descriptors.

In addition, a reference descriptor is calculated (Grover). To this reference descriptors, the following dimentionality reduction techniques are applied:

* UMAP
* PCA

Optionally, supervised versions of thesealgorithms are applied:

* Supervised UMAP
* LolP

All of the above can be performed by running the following command:

```
zairachem describe
```

Please note that calculating some descriptors (for example, Grover) may be a slow procedure. However, the Ersilia backend is linked to an in-house caching library called [Isaura](https://github.com/ersilia-os/isaura) that is able to access pre-calculated data. At the moment, Isaura works on local caching. However, we are currently setting up a cloud-based database in order to facilite access to pre-calculations stored online.

### Estimate

This step is aimed at training AutoML models based on the descriptors calculated above.

The following supervised models are applied:

* Baseline LazyQSAR models (based on Morgan fingerprints and classic descriptors).
* FLAML models on each of the pre-calculated descriptors.
* AutoGluon model based on the manifolds of the reference embedding.
* Keras Tuner fully-connected network based on the reference embedding.
* MolMap convolutional neural network.

All of these steps can be performed with the following command.

```
zairachem estimate
```

### Pool

In the pooling step, results from the estimators above are aggregated. A weighted average is applied, based on the expected performance of each of the individual estimators.

Pooling can be performed with the following command:

```
zairachem pool
```

### Report

ZairaChem provides automated performance reports as well as a output table.

* Output table
* Performance table
* Plots

```
zairachem report
```

### Distill

ZairaChem models are computationally demanding. At the end of the procedure, our goal is to provide a distilled model. This distilledm model is stored in an interoperable format (ONNX) and can be deployed as an AWS lambda. The Ersilia package for creating distilled models is called [Olinda](https://github.com/ersilia-os/olinda).

### Finish

The finish command simply offers options for cleaning

```
ersilia finish
```

## How to run a step of interest

It is possible to run a specific step from a previous session. In this case, simply initialize the session pointing to the relevant folders:

```
zairachem session --path model
```

ZairaChem will automatically identify the session as training (fit) task or as a prediction task.

Once the session has been set, you can run the command of choice. For example:

```
zairachem describe
```

### The session file

In the session file, multiple steps are specified. Each step in ZairaChem has an associated name. You can restart the pipeline at any given step.