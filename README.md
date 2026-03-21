# Clinical AI Bias Workshop  
[![Google Colab](https://img.shields.io/badge/Platform-Google%20Colab-F9AB00?logo=googlecolab&logoColor=white)](#getting-started)
[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](#getting-started)
[![Status](https://img.shields.io/badge/Status-In%20Preparation-6f42c1)](#status)
[![Audience](https://img.shields.io/badge/Audience-Beginners%20to%20Intermediate-0a7ea4)](#workshop-goal)

This repository contains the material for a hands-on workshop on **dataset bias, representativeness, and subgroup performance gaps in clinical AI**.

The workshop is designed for participants with limited coding experience and is intended to run primarily in **Google Colab**, with a strong focus on **interpretation, critical thinking, and fairness-aware model evaluation** rather than advanced implementation details.

---

## Workshop goal

The purpose of the workshop is to help participants understand that:

- bias can emerge from the **dataset**, not only from the algorithm
- strong aggregate metrics can hide important **subgroup disparities**
- a dataset can appear balanced and still be **non-representative**
- changing the composition of the training data can change subgroup performance, even when the model remains the same

The workshop combines a clinical use case, exploratory analysis, basic modeling, and controlled experiments to show how bias can be detected and discussed in a practical way.

---

## Workshop structure

The workshop is organized into four phases.

### 1. Clinical case and dataset introduction
Participants are introduced to the clinical prediction task and to the dataset used throughout the workshop.

### 2. Representativeness
Participants explore how bias can arise from:
- sample composition
- inclusion criteria
- clinical setting
- site or device
- collected variables

This phase includes a paper-based audit focused on subgroup and sex distributions.

### 3. Performance gap
Participants observe how model performance changes across:
- the full dataset
- majority sex
- minority sex
- selected subgroup intersections, where feasible

Metrics may include:
- AUC
- sensitivity
- specificity

Both a simple baseline model and a lightweight ML/deep learning model may be used.

### 4. Demonstration that bias originates in the dataset
Participants see experimentally that, with the **same algorithm**, changing the composition of the training data can change subgroup performance and observed bias.

This phase is based on multiple training scenarios with a **fixed test set**.

---

## Dataset

The workshop uses the dataset already included in this project.

**Clinical task**
- binary classification
- prediction of `hospital_death`

**Main files**
- `dataset/full_data.csv`
- `dataset/train_features.csv`
- `dataset/train_labels.csv`
- `dataset/test_features.csv`
- `dataset/test_labels.csv`

**Recommended usage**
- use `full_data.csv` for dataset description, EDA, and representativeness audit
- use train/test feature-label files for modeling and evaluation

For a more detailed description of the data files, see:
- `dataset/README.md`

---

## Repository structure

```text
.
├── dataset/
│   ├── README.md
│   ├── full_data.csv
│   ├── train_features.csv
│   ├── train_labels.csv
│   ├── test_features.csv
│   └── test_labels.csv
├── notebooks/
│   ├── 01_dataset_and_context.ipynb
│   ├── 02_representativeness_audit.ipynb
│   ├── 03_performance_gap.ipynb
│   └── 04_bias_origin_experiment.ipynb
├── slides/
├── docs/
├── outputs/
└── README.md
