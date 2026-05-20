# Recap e analisi dei risultati — Due casi sperimentali sul bias di genere

> Documento di accompagnamento al notebook `Fase_3_4_Modelli_Performance_Gap.ipynb`. Contiene il recap dei risultati dei due casi sperimentali, la spiegazione delle metriche e un filo conduttore narrativo utile per la presentazione.
>
> Pensato per **pubblico misto**: clinici, ricercatori, persone interessate all'argomento.

---

## Indice

1. [Filo conduttore](#filo-conduttore)
2. [Glossario delle metriche](#glossario-delle-metriche)
3. [Caso 1 — Bias di numerosità del genere](#caso-1--bias-di-numerosità-del-genere)
4. [Caso 2 — Two Clinics (asimmetria intra-genere su BMI)](#caso-2--two-clinics-asimmetria-intra-genere-su-bmi)
5. [Conclusioni e lezioni per la pratica clinica](#conclusioni-e-lezioni-per-la-pratica-clinica)

---

## Filo conduttore

Due casi sperimentali, stessa domanda — *come entra il bias di genere in un modello predittivo clinico?* — vista da due angolazioni:

| Caso | Cosa cambiamo | Domanda |
|------|---------------|---------|
| **1** | il **numero** di donne nel training | *"Se ho meno donne nel training, il modello sbaglia di più sulle donne?"* |
| **2** | la **distribuzione** di una feature interna (BMI), tenendo i numeri uguali | *"Se ho lo stesso numero di donne ma campionate male, il modello sbaglia comunque?"* |

Entrambi i casi sono **diagnostici**: mostrano *come* il bias entra, non come uscirne.

Il messaggio che emerge è univoco: **il bias nei modelli predittivi clinici non è un problema del modello, è un problema del dato**. E si nasconde in metriche che NON sono quelle che la maggior parte dei clinici guarda di default (AUROC, accuracy globale).

---

## Glossario delle metriche

### AUROC (Area Under the ROC Curve)
**Cos'è**: misura quanto bene il modello sa **ordinare** i pazienti dal più al meno a rischio. Va da 0.5 (casuale) a 1.0 (perfetto).

**Lettura intuitiva**: presi a caso un malato e un sano, qual è la probabilità che il modello dia un punteggio più alto al malato? Un AUROC di 0.85 significa "85 volte su 100".

**Formula**: $\text{AUROC} = P(\hat{p}_{\text{malato}} > \hat{p}_{\text{sano}})$

**Quando è insensibile al bias**: l'AUROC dipende solo dall'**ordine** delle predizioni, non dal **valore** delle probabilità. Se il modello assegna probabilità sbagliate ma nel giusto ordine relativo, AUROC resta alta. Per questo nei Casi 1 e 2 osserviamo AUROC stabile mentre il bias è evidente nella calibrazione.

### Calibration intercept
**Cos'è**: misura sintetica di quanto le probabilità del modello corrispondono alla realtà.

**Lettura intuitiva**: quando il modello dice *"questo paziente ha 30% di probabilità di diabete"*, ci aspettiamo che su 100 pazienti con quella stima circa 30 lo abbiano davvero. Se ce ne sono solo 20, il modello **sovrastima** il rischio.

**Formula**: si fitta la regressione logistica $\text{logit}(P(Y=1)) = \alpha + \beta \cdot \text{logit}(\hat{p})$ e si prende $\alpha$. Idealmente $\alpha = 0$, $\beta = 1$.

| $\alpha$ | Significato |
|----------|-------------|
| $= 0$ | calibrazione perfetta |
| $< 0$ | il modello **sovrastima** il rischio |
| $> 0$ | il modello **sottostima** il rischio |

**Perché conta in clinica**: i clinici prendono decisioni su soglie ("se rischio > 25%, invio allo specialista"). Una soglia funziona solo se le probabilità sono **veritiere**. Il bias di calibrazione fa sì che la stessa soglia produca decisioni sistematicamente diverse fra sessi — invisibile guardando solo AUROC.

---

## Caso 1 — Bias di numerosità del genere

### Setup

Quattro versioni del training set, **stessa quantità totale (3.000 persone)**, N_BOOTSTRAP = 30. Rapporti F:M variabili:

| Scenario | Donne | Uomini | Quota donne |
|----------|-------|--------|-------------|
| **S0** | 1500 | 1500 | 50% (baseline) |
| **S1** | 900 | 2100 | 30% |
| **S2** | 450 | 2550 | 15% |
| **S3** | 150 | 2850 | 5% |

Test set fisso: 750 M + 750 F. Stesso pool di pazienti, sempre.

---

### 1A — Logistic Regression sul diabete

**Target**: `diab_doctor_told_diabetes` (binario, prevalenza ~12%). **Feature**: top-30 per |corr| col target. **Modello**: LogReg L2 (C=1).

#### Risultati

**AUROC per scenario × sesso**

| Scenario | AUROC Donne | AUROC Uomini | Gap M−F |
|----------|-------------|--------------|---------|
| S0 50/50 | 0.901 | 0.918 | +0.017 |
| S1 30/70 | 0.901 | 0.919 | +0.018 |
| S2 15/85 | 0.899 | 0.919 | +0.020 |
| S3 5/95  | **0.898** | 0.916 | +0.018 |

→ AUROC praticamente invariata: il modello sembra stabile.

**Calibration intercept (ideale = 0)**

| Scenario | Donne | Uomini | Gap |
|----------|-------|--------|-----|
| S0 50/50 | −0.51 | −0.34 | 0.17 |
| S1 30/70 | −0.53 | −0.40 | 0.13 |
| S2 15/85 | −0.49 | −0.30 | 0.19 |
| S3 5/95  | −0.50 | −0.35 | 0.15 |

#### Lettura

**Risultato chiave**: l'AUROC non si muove. La calibrazione rivela il bias strutturale: le donne hanno calibration intercept peggiore degli uomini in **tutti** gli scenari, anche al 50/50. Il gap medio è ~0.17 punti.

Il bias di calibrazione non peggiora con meno donne — era già strutturale al 50/50. Il problema non si risolve solo con il riequilibrio numerico.

---

### 1B — XGBoost sull'artrite

**Target**: `mc_arthritis_ever` (binario, prevalenza ~26%). **Feature**: top-30. **Modello**: XGBoost (max_depth=4, lr=0.05, early_stopping=20).

#### Risultati

**AUROC per scenario × sesso**

| Scenario | AUROC Donne | AUROC Uomini |
|----------|-------------|--------------|
| S0 50/50 | **0.838** | 0.818 |
| S1 30/70 | 0.835 | 0.817 |
| S2 15/85 | 0.829 | 0.818 |
| S3 5/95  | **0.818** | 0.815 |

→ AUROC delle donne cala monotonicamente (0.838 → 0.818, Δ = −0.020)

**Calibration intercept**

| Scenario | Donne | Uomini | Gap |
|----------|-------|--------|-----|
| S0 50/50 | +0.10 | −0.28 | 0.38 |
| S1 30/70 | +0.15 | −0.19 | 0.34 |
| S2 15/85 | +0.23 | −0.21 | 0.44 |
| S3 5/95  | **+0.28** | −0.20 | 0.48 |

→ Calibration intercept delle donne peggiora monotonicamente da +0.10 a +0.28 (3× peggio)

**SHAP (modello S3)**: le feature più importanti per donne e uomini sono diverse. In particolare `hemoglobin_gdl`/`hematocrit_pct` — con range fisiologici normali diversi tra sessi — hanno importanza asimmetrica: il modello allenato su uomini porta la "logica maschile" di queste feature anche sulle donne.

#### Lettura

Con XGBoost il bias si vede su più metriche: AUROC cala monotonicamente, calibration peggiora sistematicamente. Il modello più sofisticato non risolve il bias — lo rende più articolato e più difficile da diagnosticare con le metriche standard.

---

### Riepilogo Caso 1

| Esperimento | Dove si vede il bias |
|-------------|----------------------|
| **1A** LogReg — diabete | Cal. intercept: gap M/F strutturale (~0.17) in tutti gli scenari |
| **1B** XGBoost — artrite | AUROC cala (0.838→0.818), cal. intercept 3× peggio (0.10→0.28), SHAP asimmetrico |

**Messaggio**: l'AUROC non vede niente. La calibrazione stratificata per sesso vede sempre il bias.

---

## Caso 2 — Two Clinics (asimmetria intra-genere su BMI)

### Setup

**N totale = 4.000 (2.000 M + 2.000 F, perfettamente bilanciati)**. N_BOOTSTRAP = 30. Cambia solo la distribuzione di BMI:

- **Baseline**: campionamento random. Le distribuzioni di BMI si sovrappongono.
- **Two Clinics**: F con BMI ≤ 28 (centro cardiovascolare), M con BMI ≥ 28 (clinica bariatrica).

**Validazione preliminare** (BMI è sex-neutro sui dati grezzi):
- corr(BMI, gender) ≈ +0.077 → quasi zero
- corr(BMI, diabete) = −0.115 in M, −0.107 in F → relazione identica nei due sessi

Tutto il bias osservato è interamente dovuto al campionamento, non a proprietà biologiche del BMI.

**Modelli**: LogReg L2 (principale) + XGBoost (conferma). **Target**: diabete (stesso di 1A).

### Risultati

**Metriche chiave (LogReg L2)**

| Variante | AUROC F | AUROC M | Cal.Int. F | Cal.Int. M |
|----------|---------|---------|------------|------------|
| Baseline    | 0.902 | 0.916 | −0.46 | −0.37 |
| Two Clinics | 0.888 | 0.915 | **−0.74** | −0.29 |
| Δ | −0.014 | −0.001 | **−0.28 (−60%)** | +0.08 |

**XGBoost**: stesso pattern — AUROC F scende di 0.011, AUROC M di 0.002. Conferma che non è colpa del modello lineare.

**SHAP (modello perturbato XGBoost)**: il BMI sale al 5°–6° posto nell'importanza. Il modello ha ricostruito il sesso dal BMI. Le feature con delta maggiore tra baseline e perturbato sono `bp_chol_ever_high` e `cb_money_spent_food_fast_food` — cluster cardio-metabolico e comportamentale correlato al BMI.

### Lettura

**AUROC**: quasi identico nelle due varianti — guardare solo l'AUROC avrebbe nascosto tutto.

**Calibration intercept**: crolla del 60% sulle donne (−0.46 → −0.74). Stabile sugli uomini.

Il modello non ha imparato una regola sbagliata. Ha generalizzato male perché ha visto una distribuzione non rappresentativa: donne sempre magre, uomini sempre grossi. Quando incontra una donna con BMI alto o un uomo con BMI basso nel test, sbaglia in modo asimmetrico.

---

## Conclusioni e lezioni per la pratica clinica

### Lezione 1 — Il bias non vive nella metrica più popolare

In entrambi i casi, l'AUROC è quasi invariata. Il bias c'è, ma si nasconde:

- **1A**: cal. intercept donne ~−0.50 vs uomini ~−0.34 (gap strutturale, presente in tutti gli scenari)
- **1B**: cal. intercept donne da +0.10 a +0.28 (3× peggio, monotonicamente)
- **Caso 2**: cal. intercept donne −0.46 → −0.74 (−60%)

→ **AUROC misura il ranking, non la veridicità delle probabilità**. Per usare un modello in clinica serve la **calibrazione stratificata per sesso**, non solo la discriminazione.

### Lezione 2 — Meccanismi diversi, stessa invisibilità

| Caso | Meccanismo | Dove si vede |
|------|------------|--------------|
| **1** Numerosità | Sotto-rappresentazione quantitativa | Calibration intercept (1A/1B) |
| **2** Two Clinics | Asimmetria di campionamento su una feature | Calibration intercept, probabilità media predetta |

Non esiste una "metrica universale del bias": serve un audit multi-metrica stratificato per sesso.

### Lezione 3 — Le soluzioni "ovvie" non bastano

**"Basta avere 50/50"** — il Caso 2 smentisce. Con numeri perfetti ma campionamento asimmetrico il modello ricostruisce il sesso indirettamente e produce predizioni distorte.

**"Uso un modello più sofisticato"** — il Caso 1B smentisce. XGBoost mostra il bias ancora più chiaramente della logistica, su più metriche.

→ Il bias di genere è un **problema strutturale del dato**, non risolvibile con una singola correzione.

### La frase da portare via

> *Il sesso entra dalla finestra anche quando lo si esclude dalla porta. Per accorgersene bisogna guardare la calibrazione stratificata, non solo l'AUROC. Per evitarlo bisogna progettare il dato, non il modello.*

---

## Appendice — File generati dal notebook

```
archive/
├── case1/
│   ├── 1B_bootstrap_records.csv        # AUROC + cal_int per sesso e scenario (30×4 righe)
│   ├── 1B_summary_full_metrics.csv     # Medie 1A LogReg
│   ├── 1C_bootstrap_records.csv        # Come sopra ma XGBoost
│   ├── 1C_summary_full_metrics.csv     # Medie 1B XGBoost
│   ├── 1B_shap_compare_by_sex.csv      # Ranking SHAP F vs M (1B XGBoost)
│   └── summary.json                    # Riepilogo numerico dei due esperimenti
└── case2/
    ├── 2_logreg_bootstrap_records.csv  # AUROC + cal_int (30×2 varianti)
    ├── 2_logreg_full_metrics.csv
    ├── 2_xgboost_bootstrap_records.csv
    └── 2_shap_compare_by_sex.csv       # Ranking SHAP F vs M modello perturbato
```

Il notebook rigenera tutti i file da zero con `Run All` (~9–12 minuti live con N_BOOTSTRAP=30).
