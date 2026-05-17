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
5. [Conclusioni e tre lezioni per la pratica clinica](#conclusioni-e-tre-lezioni-per-la-pratica-clinica)

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
**Cos'è**: misura quanto bene il modello sa **ordinare** i pazienti dal più al meno a rischio. Va da 0.5 (modello che tira a indovinare) a 1.0 (perfetto).

**Lettura intuitiva**: presi a caso un malato e un sano, qual è la probabilità che il modello dia un punteggio più alto al malato? Un AUROC di 0.85 significa "85 volte su 100".

**Quando è insensibile al bias**: l'AUROC dipende solo dall'**ordine** delle predizioni, non dal **valore** delle probabilità. Se il modello assegna probabilità sbagliate ma nel giusto ordine relativo, AUROC resta alta. Per questo nei Casi 1 e 2 osserviamo AUROC stabile mentre il bias è evidente in altre metriche.

### MAE (Mean Absolute Error)
**Cos'è**: errore medio assoluto fra valore predetto e valore vero, in unità del target. Se il target è la pressione sistolica in mmHg e MAE = 12, il modello sbaglia in media di 12 mmHg.

**Quando è informativo**: misura l'errore in valore assoluto, sesso per sesso. Se MAE delle donne cresce mentre MAE degli uomini resta stabile, il bias è visibile direttamente.

### Calibration intercept
**Cos'è**: misura sintetica di quanto le probabilità del modello corrispondono alla realtà.

**Lettura intuitiva**: quando il modello dice *"questo paziente ha 20% di probabilità di diabete"*, ci aspettiamo che su 100 pazienti con quella stima circa 20 lo abbiano davvero. Un calibration intercept di **0** = modello perfettamente calibrato; più ci si allontana da 0, peggio è.

**Tecnicamente**: si fitta una regressione logistica `y_reale ~ α + β·logit(p_predetta)` e si prende α. Idealmente α=0, β=1.

**Perché conta in clinica**: i clinici prendono decisioni su soglie ("se rischio > 25%, invio allo specialista"). Una soglia funziona solo se le probabilità sono **veritiere**. Il bias di calibrazione fa sì che la stessa soglia produca decisioni diverse fra sessi — invisibile guardando solo AUROC.

### Equal Opportunity Difference (EOD)
**Cos'è**: `TPR_M − TPR_F`, dove TPR = *true positive rate* = **sensibilità** = fra tutti i malati reali, quanti ne identifica il modello?

**Lettura intuitiva**: se il modello identifica il 90% dei diabetici maschi ma solo l'81% delle diabetiche femmine, EOD = +0.09. Significa che il modello "perde" più donne malate di uomini malati. Ideale: **zero** — stessa sensibilità in entrambi i sessi.

**Quando il segno si inverte**: EOD negativo significa che il modello identifica meglio i malati tra le donne. Non è necessariamente un miglioramento — nel Caso 2 è il segno di un bias che ha cambiato direzione.

### DeLong test
**Cos'è**: confronta due **AUROC** sullo stesso test set. Restituisce un p-value per *"le due AUROC sono significativamente diverse?"*.

**Limite**: insensibile alla calibrazione. Due modelli con identica AUROC ma calibrazione molto diversa sono "uguali" per DeLong.

### McNemar test
**Cos'è**: confronta due **classificatori binari** sullo stesso test set, contando le discordanze. Restituisce un p-value per *"i due modelli classificano diversamente i pazienti?"*.

**Quando si applica**: per misurare se due modelli prendono decisioni cliniche diverse a soglia 0.5. Può essere non significativo anche quando il bias è presente (come in 1B e Caso 2).

### t-test pareato
**Cos'è**: confronta due misurazioni fatte **sugli stessi soggetti** in due condizioni diverse.

**Quando si applica**: nel Caso 1A confrontiamo l'errore assoluto di ogni paziente tra il modello S0 e S3, separatamente per sesso.

---

## Caso 1 — Bias di numerosità del genere

### Setup

Quattro versioni del training set, **stessa quantità totale (3.000 persone)**, N_BOOTSTRAP = 50. Rapporti F:M variabili:

| Scenario | Donne | Uomini | Quota donne |
|----------|-------|--------|-------------|
| **S0** | 1500 | 1500 | 50% (baseline) |
| **S1** | 900 | 2100 | 30% |
| **S2** | 450 | 2550 | 15% |
| **S3** | 150 | 2850 | 5% |

Test set fisso: 750 M + 750 F. Stesso pool di pazienti, sempre.

---

### 1A — Regressione lineare sulla pressione sistolica

**Target**: `bp_systolic_1` (mmHg). **Predittori**: BMI, età. **Modello**: OLS.

#### Risultati

**MAE per scenario × sesso (mmHg, più basso = meglio)**

| Scenario | MAE Donne | MAE Uomini | Δ F−M |
|----------|-----------|------------|-------|
| S0 50/50 | 11.72 | 11.78 | −0.06 |
| S1 30/70 | 11.98 | 11.62 | +0.36 |
| S2 15/85 | 12.24 | 11.58 | +0.66 |
| S3 5/95  | **12.44** | 11.58 | **+0.86** |

**T-test pareato (S0 vs S3, stratificato per sesso)**

| Sesso | MAE S0 | MAE S3 | Δ | p |
|-------|--------|--------|---|---|
| **Donne** | 11.72 | 12.47 | **+0.74 mmHg** | **1.6 × 10⁻¹⁴** |
| Uomini | 11.78 | 11.59 | −0.19 mmHg | 0.062 (n.s.) |

**Coefficienti**: l'intercetta cresce da 94.9 a 101.8 mmHg (S0→S3) — la retta trasla verso il pattern maschile.

#### Lettura

Il peggioramento sulle donne è altamente significativo, su quello sugli uomini non lo è. Questo schema asimmetrico è la firma del bias di numerosità: il modello ottimizza per la popolazione dominante nel training.

---

### 1B — Regressione logistica sul diabete

**Target**: `diab_doctor_told_diabetes` (binario, prevalenza ~12%). **Feature**: top-30 per |corr| col target. **Modello**: LogReg L2 (C=1).

#### Risultati

**AUROC per scenario × sesso**

| Scenario | AUROC Donne | AUROC Uomini | Gap M−F |
|----------|-------------|--------------|---------|
| S0 50/50 | 0.901 | 0.918 | +0.017 |
| S1 30/70 | 0.901 | 0.919 | +0.018 |
| S2 15/85 | 0.899 | 0.919 | +0.020 |
| S3 5/95  | **0.898** | 0.916 | +0.018 |

**DeLong test (AUROC F: S0 vs S3)**: Δ = −0.002, p = **0.74** → non significativo

**McNemar (S0 vs S3, donne, soglia 0.5)**: 8 vs 9 discordanze, p = **1.00** → non significativo

**Calibration intercept (ideale = 0)**

| Scenario | Donne | Uomini | Gap |
|----------|-------|--------|-----|
| S0 50/50 | −0.51 | −0.34 | 0.17 |
| S1 30/70 | −0.53 | −0.40 | 0.13 |
| S2 15/85 | −0.49 | −0.30 | 0.19 |
| S3 5/95  | −0.50 | −0.35 | 0.15 |

**EOD (TPR_M − TPR_F)**: stabile intorno a +0.017/+0.020 in tutti gli scenari — gap piccolo ma persistente.

#### Lettura

**Risultato chiave**: AUROC, DeLong e McNemar dicono "nessuna differenza". La calibrazione rivela il bias strutturale: le donne hanno calibration intercept peggiore degli uomini in **tutti** gli scenari, anche al 50/50. Il gap medio è ~0.17 punti.

Il bias di calibrazione non peggiora linearmente con meno donne — era già strutturale al 50/50. Questo suggerisce che il problema non si risolve solo con il riequilibrio numerico.

---

### 1C — XGBoost sull'artrite

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

**McNemar (donne, S0 vs S3)**: 64 vs 42 discordanze, p = **0.041** → **significativo** (S0 batte S3)

**Calibration intercept**

| Scenario | Donne | Uomini | Gap |
|----------|-------|--------|-----|
| S0 50/50 | +0.10 | −0.28 | 0.38 |
| S1 30/70 | +0.15 | −0.19 | 0.34 |
| S2 15/85 | +0.23 | −0.21 | 0.44 |
| S3 5/95  | **+0.28** | −0.20 | 0.48 |

→ Calibration intercept delle donne peggiora monotonicamente da +0.10 a +0.28 (3× peggio)

**SHAP (modello S3)**: le feature più importanti per donne e uomini sono diverse. In particolare `hemoglobin_gdl`/`hematocrit_pct` — con range fisiologici normali diversi tra sessi — hanno importanza asimmetrica: il modello allenato su uomini porta la "logica maschile" di questa feature anche sulle donne.

#### Lettura

Con XGBoost il bias si vede su **3 metriche su 4**: AUROC cala, McNemar è significativo, calibration peggiora. Il modello più sofisticato non risolve il bias — lo rende più articolato e più difficile da diagnosticare con le metriche standard.

---

### Riepilogo Caso 1

| Esperimento | Modello | Test | Vede il bias? | Dove |
|-------------|---------|------|---------------|------|
| **1A** OLS — pressione | t-test pareato | ✅ Sì | MAE donne +0.74 mmHg, p < 10⁻¹⁴ |
| **1B** LogReg — diabete | DeLong | ❌ No | p = 0.74 |
| **1B** LogReg — diabete | McNemar | ❌ No | p = 1.00 |
| **1B** LogReg — diabete | Cal. intercept | ✅ Sì | Gap M/F strutturale in tutti gli scenari |
| **1C** XGBoost — artrite | McNemar | ✅ Sì | p = 0.041 |
| **1C** XGBoost — artrite | Cal. intercept | ✅ Sì | +0.10 → +0.28 (3× peggio) |

**Messaggio**: AUROC e McNemar (in 1B) non vedono niente. La calibrazione stratificata per sesso vede sempre il bias.

---

## Caso 2 — Two Clinics (asimmetria intra-genere su BMI)

### Setup

**N totale = 4.000 (2.000 M + 2.000 F, perfettamente bilanciati)**. N_BOOTSTRAP = 50. Cambia solo la distribuzione di BMI:

- **Baseline**: campionamento random. Le distribuzioni di BMI si sovrappongono.
- **Two Clinics**: F con BMI ≤ 28 (centro cardiovascolare), M con BMI ≥ 28 (clinica bariatrica).

**Validazione preliminare** (BMI è sex-neutro sui dati grezzi):
- corr(BMI, gender) ≈ +0.077 → quasi zero
- corr(BMI, diabete) = −0.115 in M, −0.107 in F → relazione identica nei due sessi

Tutto il bias osservato è interamente dovuto al campionamento, non a proprietà biologiche del BMI.

**Modelli**: LogReg L2 (principale) + XGBoost (conferma). **Target**: diabete (stesso di 1B).

### Risultati

**Metriche chiave (LogReg L2)**

| Variante | AUROC F | AUROC M | Cal.Int. F | Cal.Int. M | EOD |
|----------|---------|---------|------------|------------|-----|
| Baseline    | 0.902 | 0.916 | −0.46 | −0.37 | **+0.09** |
| Two Clinics | 0.888 | 0.915 | **−0.74** | −0.29 | **−0.045** |
| Δ | −0.014 | −0.001 | **−0.28 (−60%)** | +0.08 | **−0.135** |

**McNemar (baseline vs Two Clinics, soglia 0.5)**:
- Donne: p = **0.65** → non significativo
- Uomini: p = **0.15** → non significativo

**Probabilità media predetta vs prevalenza reale (~12.8%)**

| Variante | P(diabete\|F) | P(diabete\|M) | Errore su F |
|----------|---------------|---------------|-------------|
| Baseline    | 12.9% | 12.1% | +0.1 pp (perfetto) |
| Two Clinics | **14.0%** | 11.5% | **+1.2 pp (sovrastima)** |

**XGBoost**: stesso pattern — AUROC F scende di 0.011, AUROC M di 0.002. Conferma che non è colpa del modello lineare.

**SHAP (modello perturbato XGBoost)**: il BMI sale al 5°–6° posto nell'importanza. Il modello ha ricostruito il sesso dal BMI. Le feature con delta maggiore tra baseline e perturbato sono `bp_chol_ever_high` e `cb_money_spent_food_fast_food` — cluster cardio-metabolico e comportamentale correlato al BMI.

### Lettura

**McNemar non significativo**: le decisioni binarie a soglia 0.5 sembrano uguali. Se ci si ferma qui, non si vede niente.

**Calibration intercept**: crolla del 60% sulle donne (−0.46 → −0.74). Stabile sugli uomini.

**EOD inverte di segno**: il modello passa da identificare meglio i diabetici tra gli uomini (+0.09) a identificarli meglio tra le donne (−0.045). Non è un miglioramento — è una inversione causata dal bias di campionamento.

Il modello non ha imparato una regola sbagliata. Ha generalizzato male perché ha visto una distribuzione non rappresentativa: donne sempre magre, uomini sempre grossi. Quando incontra una donna grassa o un uomo magro nel test (fuori distribuzione), sbaglia in modo asimmetrico.

---

## Conclusioni e tre lezioni per la pratica clinica

### Lezione 1 — Il bias non vive nella metrica più popolare

In entrambi i casi, AUROC e McNemar a soglia 0.5 sono quasi sempre non significativi. Il bias c'è, ma si nasconde:

- **1B**: cal. intercept donne = −0.50 vs uomini = −0.34 (gap 47% peggiore) — in tutti gli scenari
- **1C**: cal. intercept donne da +0.10 a +0.28 (3× peggio) — monotonico
- **Caso 2**: cal. intercept donne −0.46 → −0.74 (−60%); EOD inverte di segno (Δ = −0.135)

→ **AUROC misura il ranking, non la veridicità delle probabilità**. Per usare un modello in clinica serve la **calibrazione stratificata per sesso**, non solo la discriminazione.

### Lezione 2 — Meccanismi diversi, stessa invisibilità alle metriche standard

| Caso | Meccanismo | Dove si vede |
|------|------------|--------------|
| **1** Numerosità | Sotto-rappresentazione quantitativa | MAE (1A), calibration intercept (1B/1C), McNemar (1C) |
| **2** Two Clinics | Asimmetria di campionamento su una feature | Calibration intercept, EOD, probabilità media predetta |

Meccanismi diversi → famiglie di metriche diverse. Non esiste una "metrica universale del bias": serve un audit multi-metrica stratificato per sesso.

### Lezione 3 — Le soluzioni "ovvie" non bastano

**"Basta avere 50/50 di donne e uomini"** — il Caso 2 smentisce. Con numeri perfetti ma campionamento asimmetrico il modello ricostruisce il sesso indirettamente e produce predizioni distorte.

**"Uso un modello più sofisticato"** — il Caso 1C smentisce. XGBoost mostra il bias ancora più chiaramente della logistica, su più metriche.

→ Il bias di genere è un **problema strutturale del dato**, non risolvibile con una singola correzione. Servono:
1. Audit multi-metrica stratificato (sempre, prima del deploy)
2. Trasparenza sulle scelte di campionamento del training
3. Validazione su popolazioni demograficamente diverse

### La frase da portare via

> *Il sesso entra dalla finestra anche quando lo si esclude dalla porta. Per accorgersene bisogna guardare la calibrazione stratificata, non solo l'AUROC. Per evitarlo bisogna progettare il dato, non il modello.*

---

## Appendice — File generati dal notebook

```
archive/
├── case1/
│   ├── 1A_bootstrap_records.csv      # MAE per sesso e scenario (50×4 righe)
│   ├── 1A_summary.csv                # Media + CI95% MAE per scenario
│   ├── 1B_bootstrap_records.csv      # AUROC + cal_int + EOD (50×4 righe)
│   ├── 1B_summary_full_metrics.csv   # Tutte le metriche 1B
│   ├── 1B_delong.csv / 1B_mcnemar.csv
│   ├── 1C_bootstrap_records.csv      # Come 1B ma XGBoost
│   ├── 1C_summary_full_metrics.csv
│   └── 1C_delong.csv / 1C_mcnemar.csv / 1C_calibration_intercept.csv
└── case2/
    ├── 2_logreg_bootstrap_records.csv   # AUROC + cal_int + EOD (50×2 varianti)
    ├── 2_logreg_full_metrics.csv
    ├── 2_mcnemar_baseline_vs_perturbed.csv
    ├── 2_equal_opportunity_difference.csv
    ├── 2_calibration_intercept.csv
    └── 2_xgboost_bootstrap_records.csv
```

Il notebook rigenera tutti i file da zero con `Run All` (~15–20 minuti live, ~8–10 minuti in locale con N_BOOTSTRAP=50).
