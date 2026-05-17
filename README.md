dataset: https://drive.google.com/drive/folders/15uHafn7dPMz6nAEAR2M27z-oYZyOJIwv?hl=it

Cartella con Slides e Note: [https://docs.google.com/document/d/16mO6olskOOdfW9gMZpkZguCOlMTwVjSn_3RR67_I9EA/edit?tab=t.0](https://drive.google.com/drive/folders/1FfVu3Cg9fPm5WI9rwil3kHKtSXjrl6Fx)

Report da tenere aggiornato (per pubblicazione): https://docs.google.com/document/d/16mO6olskOOdfW9gMZpkZguCOlMTwVjSn_3RR67_I9EA/edit?tab=t.0

---

# Clinical AI Bias Workshop

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/oliviariccomi/gender-bias-analysis/blob/main/Fase_3_4_Modelli_Performance_Gap.ipynb)
[![Google Colab](https://img.shields.io/badge/Platform-Google%20Colab-F9AB00?logo=googlecolab&logoColor=white)](#come-aprire-il-notebook)
[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](#requisiti)
[![Status](https://img.shields.io/badge/Status-Ready-2ea44f)](#)
[![Audience](https://img.shields.io/badge/Audience-Clinici%20%2B%20Ricercatori-0a7ea4)](#workshop-goal)

Workshop hands-on sul **bias di genere nei modelli predittivi clinici** — SaniDays Roma, 23 maggio 2026.

---

## Come aprire il notebook

**Clicca qui → apre il notebook nella tua sessione personale Google Colab:**

**[▶ Apri in Google Colab](https://colab.research.google.com/github/oliviariccomi/gender-bias-analysis/blob/main/Fase_3_4_Modelli_Performance_Gap.ipynb)**

Oppure copia questo link nel browser:
```
https://colab.research.google.com/github/oliviariccomi/gender-bias-analysis/blob/main/Fase_3_4_Modelli_Performance_Gap.ipynb
```

> Ogni persona che clicca ottiene la propria sessione indipendente. Non è necessario caricare nessun file — il dataset NHANES viene scaricato automaticamente dalla prima cella.

**Procedura:**
1. Clicca il link sopra
2. Accedi con il tuo account Google (se richiesto)
3. `Runtime → Run all` (oppure `Ctrl+F9`)
4. La prima esecuzione installa le dipendenze e scarica i dati (~2 minuti)

---

## Workshop goal

Due domande, due esperimenti su dati reali (NHANES 2013-2014, 6.113 adulti):

1. **Caso 1 — Bias di numerosità**: se riduco la quota di donne nel training, il modello peggiora sulle donne?
2. **Caso 2 — Two Clinics**: se le donne e gli uomini sono reclutati da contesti diversi (pur essendo 50/50 in numero), il modello sbaglia comunque?

**Messaggio chiave**: il bias di genere nei modelli clinici si nasconde nelle metriche di calibrazione, non nell'AUROC. Per vederlo serve stratificare per sesso.

---

## Struttura del notebook

| Sezione | Contenuto | Durata stimata |
|---------|-----------|----------------|
| **Pre-flight** | Pulizia dataset NHANES, test set fisso, spiegazione bootstrap | ~5 min |
| **Caso 1A** | Regressione lineare sulla pressione sistolica | ~2 min |
| **Caso 1B** | Regressione logistica sul diabete | ~4 min |
| **Caso 1C** | XGBoost sull'artrite + SHAP | ~5 min |
| **Caso 2** | Two Clinics (asimmetria BMI) + LogReg + XGBoost | ~5 min |

**Totale: ~20 minuti live**, tutto eseguito in tempo reale.

---

## Dataset

**NHANES 2013-2014** — National Health and Nutrition Examination Survey (USA).

- 6.113 adulti (≥18 anni): 2.916 uomini, 3.197 donne
- Variabili: esami del sangue, misure fisiche, questionari, diagnosi cliniche
- File: `data/raw_dataset/NHANES_2013_2014_master.csv` (scaricato automaticamente su Colab)

---

## Requisiti

Nessuna installazione locale necessaria — tutto gira su Google Colab.

Se volete eseguire in locale:
```bash
pip install xgboost shap statsmodels scikit-learn pandas numpy matplotlib seaborn
```
Su macOS, XGBoost richiede `libomp`:
```bash
brew install libomp
```

---

## Repository

```
.
├── Fase_3_4_Modelli_Performance_Gap.ipynb   # Notebook principale del workshop
├── Fase_1_2_EDA_Representativeness.ipynb    # EDA e analisi di rappresentatività
├── data/
│   └── raw_dataset/
│       └── NHANES_2013_2014_master.csv      # Dataset principale
├── RECAP_RISULTATI.md                        # Risultati numerici di riferimento
├── script_presentazione.md                  # Script parlato del workshop
└── NHANES_2013_2014_CSV.zip                 # Archivio originale NHANES
```
