# Dataset

This folder contains the data files used for the workshop on clinical dataset bias, representativeness, and subgroup performance analysis.

The dataset supports the full workshop pipeline, including:
- clinical case introduction
- exploratory data analysis (EDA)
- representativeness audit
- model training and evaluation
- subgroup performance gap analysis
- experimental demonstration that bias can originate from dataset composition

---

## Clinical task

The dataset is used for a **binary classification** task in a clinical setting:

**Target variable:** `hospital_death`

Meaning:
- `0` = the patient did **not** die in hospital
- `1` = the patient **died** in hospital

This task is used throughout the workshop to show how:
- aggregate metrics can hide subgroup disparities
- apparently balanced datasets may still be non-representative
- changing the composition of the training data can change subgroup performance and observed bias

---

## Folder contents

This folder currently includes:

- `full_data.csv`
- `train_features.csv`
- `train_labels.csv`
- `test_features.csv`
- `test_labels.csv`

---

## File overview

### `full_data.csv`
Complete dataset in a format closest to the original project-level data.

**Recommended use**
- clinical case introduction
- dataset description
- exploratory data analysis
- representativeness audit
- subgroup definition
- inspection of sensitive or stratification variables

**Notes**
- includes both predictors and target-related context
- best file for understanding the dataset structure
- more suitable than preprocessed feature matrices for teaching and interpretation

---

### `train_features.csv`
Preprocessed training features used as model input.

**Recommended use**
- model training
- baseline comparison across algorithms
- subgroup performance analysis
- experimental scenarios with modified training distributions

**Notes**
- does **not** contain the target variable
- larger than `full_data.csv` because it is already model-ready
- categorical variables may have been expanded through encoding
- intended for the modeling phase, not for intuitive dataset description

---

### `train_labels.csv`
Target values corresponding to `train_features.csv`.

**Recommended use**
- supervised model training
- performance evaluation on the training split

**Notes**
- contains the target column `hospital_death`
- row order must match `train_features.csv`

---

### `test_features.csv`
Preprocessed test features used for final model evaluation.

**Recommended use**
- final model testing
- subgroup comparison
- performance gap analysis
- fixed test set for bias-origin experiments

**Notes**
- does **not** contain the target variable
- should always be used together with `test_labels.csv`

---

### `test_labels.csv`
Target values corresponding to `test_features.csv`.

**Recommended use**
- final evaluation of model predictions
- comparison of overall vs subgroup metrics

**Notes**
- contains the target column `hospital_death`
- row order must match `test_features.csv`

---

## Features vs labels

The project separates the data into:

- **features**: input variables used by the model to make predictions
- **labels**: correct target values the model must learn to predict

In this workshop:
- `train_features.csv` / `test_features.csv` = model inputs
- `train_labels.csv` / `test_labels.csv` = model targets

This is standard practice in machine learning workflows.

---

## Recommended file usage by workshop phase

### Phase 1 — Clinical case and dataset introduction
Use:
- `full_data.csv`

Why:
- easier to explain
- better for describing the dataset to non-expert participants
- more suitable for identifying relevant subgroups and contextual variables

---

### Phase 2 — Representativeness audit
Use:
- `full_data.csv`

Why:
- preserves interpretable dataset structure
- allows direct inspection of subgroup composition
- supports paper-based audit activities
- useful for computing intersections such as:
  - sex × subgroup
  - age × sex
  - ethnicity × sex
  - site/device × sex, if applicable

---

### Phase 3 — Performance gap analysis
Use:
- `train_features.csv`
- `train_labels.csv`
- `test_features.csv`
- `test_labels.csv`

Why:
- these files are already prepared for model fitting and evaluation
- ideal for comparing:
  - overall metrics
  - majority-group metrics
  - minority-group metrics
  - subgroup-intersection metrics

---

### Phase 4 — Demonstration that bias originates in the dataset
Use:
- `train_features.csv`
- `train_labels.csv`
- `test_features.csv`
- `test_labels.csv`

Why:
- training data can be modified across scenarios
- the test set can remain fixed
- this supports controlled experiments such as:

  - **Scenario A**: nearly balanced by sex, imbalanced by severity  
  - **Scenario B**: same sex balance, improved severity balance  
  - **Scenario C**: same sex and severity balance, imbalanced by site/device  
  - **Scenario D**: improved balance across subgroup intersections  

This phase is essential to show that, even with the **same algorithm**, changing the training data changes subgroup behavior and observed bias.

---

## Why `train_features.csv` is larger than `full_data.csv`

This is expected.

`train_features.csv` is a **preprocessed, model-ready feature matrix**, while `full_data.csv` is closer to the original dataset format.

Possible reasons include:
- one-hot encoding of categorical variables
- expansion into multiple numeric columns
- standardized numeric values
- wider tabular structure after preprocessing

So:
- `full_data.csv` is better for interpretation
- `train_features.csv` is better for modeling

---

## Data workflow recommendation

For the workshop, keep a clear separation between:

### Interpretable data
Used for:
- teaching
- inspection
- representativeness
- subgroup reasoning

Primary file:
- `full_data.csv`

### Model-ready data
Used for:
- training
- evaluation
- subgroup performance analysis
- scenario-based experiments

Primary files:
- `train_features.csv`
- `train_labels.csv`
- `test_features.csv`
- `test_labels.csv`

---

## Version control recommendation

Large preprocessed files such as `train_features.csv` and `test_features.csv` may be difficult to store directly in Git.

Recommended strategy:
- use **Git** for:
  - code
  - notebooks
  - documentation
  - configuration
  - README files
- use **Google Drive** or shared storage for:
  - large CSV files
  - processed datasets
  - backup copies

If needed, feature matrices should be regenerable from the raw/full dataset through documented preprocessing steps.

---

## Team note

All team members should be familiar with:
- the clinical task
- the meaning of `hospital_death`
- the difference between `full_data.csv` and preprocessed feature files
- which file belongs to which workshop phase
- the rationale for choosing this dataset for the workshop

Even team members focused on slides or facilitation should understand the dataset structure well enough to explain the logic behind the exercises.

---

## Quick reference

| File | Type | Main purpose |
|---|---|---|
| `full_data.csv` | complete dataset | case description, EDA, representativeness |
| `train_features.csv` | preprocessed inputs | model training |
| `train_labels.csv` | training targets | supervised learning |
| `test_features.csv` | preprocessed inputs | model evaluation |
| `test_labels.csv` | test targets | final evaluation |

---

## Current status

This dataset is currently the main workshop dataset because it:
- is already available
- is compatible with a Google Colab workflow
- supports subgroup analysis
- supports representativeness reasoning
- supports performance gap analysis
- supports controlled experiments on dataset-induced bias