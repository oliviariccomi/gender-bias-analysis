# Dataset Description

This folder contains ICU patient data used for **hospital mortality prediction**, sourced from the [WiDS Datathon 2020](https://www.kaggle.com/c/widsdatathon2020) (Women in Data Science). The dataset is designed to study gender bias in ICU mortality prediction models.

## Files Overview

| File | Rows | Columns | Size |
|---|---|---|---|
| `full_data.csv` | 91,713 | 99 | ~37 MB |
| `train_features.csv` | 73,370 | 119 | ~117 MB |
| `train_labels.csv` | 73,370 | 1 | ~147 KB |
| `test_features.csv` | 18,343 | 119 | ~29 MB |
| `test_labels.csv` | 18,343 | 1 | ~37 KB |

## File Descriptions

### `full_data.csv`
The raw, unprocessed dataset with **99 columns** containing patient demographics, clinical measurements, and outcomes. Key columns include:
- **Demographics**: `age`, `gender`, `ethnicity`, `height`, `weight`, `bmi`
- **Target variable**: `hospital_death` (binary: 0 = survived, 1 = died)
- **APACHE scores**: severity-of-illness metrics (`gcs_eyes_apache`, `gcs_motor_apache`, `gcs_verbal_apache`, etc.)
- **Lab values (day 1)**: `d1_*` prefixed columns (e.g., `d1_glucose_max`, `d1_creatinine_min`)
- **Vitals**: heart rate, blood pressure, temperature, respiratory rate, SpO2
- **Comorbidities**: `diabetes_mellitus`, `cirrhosis`, `hepatic_failure`, `leukemia`, `lymphoma`, `aids`, `immunosuppression`, `solid_tumor_with_metastasis`
- **ICU metadata**: `hospital_id`, `patient_id`, `pre_icu_los_days`, `elective_surgery`
- **APACHE IV predictions**: `apache_4a_hospital_death_prob`, `apache_4a_icu_death_prob`

### `train_features.csv` / `test_features.csv`
Pre-processed feature matrices (80/20 split) with **119 columns**. Transformations applied:
- **Numerical features are standardized** (zero mean, unit variance)
- **Categorical features are one-hot encoded**: `elective_surgery_0/1`, `ethnicity_*`, `gender_F/M`, `intubated_apache_*`, `ventilated_apache_*`, comorbidity flags (`aids_*`, `cirrhosis_*`, etc.)
- Missing values appear to be imputed with near-zero values

### `train_labels.csv` / `test_labels.csv`
Single column `hospital_death` (binary classification target):
- `0` = patient survived
- `1` = patient died during hospital stay

## Train/Test Split

| Set | Samples | Proportion |
|---|---|---|
| Train | 73,370 | ~80% |
| Test | 18,343 | ~20% |

## Gender Bias Analysis Context

The `gender` column (raw data) and `gender_F` / `gender_M` columns (processed data) are the **sensitive attributes** for fairness analysis. The goal is to evaluate whether mortality prediction models exhibit differential performance or bias across gender groups.
