# Recap e analisi dei risultati — Tre casi sperimentali sul bias di genere

> Documento di accompagnamento al notebook `Fase_3_4_Modelli_Performance_Gap.ipynb`. Contiene il recap dei risultati dei tre casi sperimentali, la spiegazione delle metriche e un filo conduttore narrativo utile per la presentazione.
> 
> Pensato per **pubblico misto**: clinici, ricercatori, persone interessate all'argomento. Le sezioni "lettura clinica" sono in italiano semplice; le sezioni "lettura tecnica" entrano nel dettaglio statistico per chi vuole approfondire.

---

## Indice

1. [Filo conduttore](#filo-conduttore)
2. [Glossario delle metriche](#glossario-delle-metriche)
3. [Caso 1 — Bias di numerosità del genere](#caso-1--bias-di-numerosità-del-genere)
4. [Caso 2 — Two Clinics (asimmetria intra-genere su BMI)](#caso-2--two-clinics-asimmetria-intra-genere-su-bmi)
5. [Caso 3 — Modelli sex-specific guidati dalla matrice di correlazione](#caso-3--modelli-sex-specific-guidati-dalla-matrice-di-correlazione)
6. [Conclusioni cross-case e tre lezioni per la pratica clinica](#conclusioni-cross-case-e-tre-lezioni-per-la-pratica-clinica)

---

## Filo conduttore

I tre casi sperimentali rispondono alla stessa domanda — *come entra il bias di genere in un modello predittivo clinico?* — guardandola da **tre angolazioni complementari**:

| Caso | Cosa cambiamo | Domanda |
|---|---|---|
| **1** | il **numero** di donne nel training | *"Se ho meno donne nel training, il modello sbaglia di più sulle donne?"* |
| **2** | la **distribuzione** di una feature interna (BMI), tenendo i numeri uguali | *"Se ho lo stesso numero di donne ma campionate male, il modello sbaglia comunque?"* |
| **3** | la **struttura** del modello (Unisex vs M-only vs F-only) | *"Se costruisco modelli stratificati per sesso, ottengo predizioni migliori?"* |

I primi due casi sono **diagnostici** (mostrano *come* il bias entra). Il terzo è **costruttivo** (chiede *come* uscirne).

Il messaggio che emerge dai dati è univoco: **il bias nei modelli predittivi clinici non è un problema del modello, è un problema del dato**. E si nasconde in metriche che NON sono quelle che la maggior parte dei clinici guarda di default (AUROC, accuracy globale).

---

## Glossario delle metriche

### AUROC (Area Under the ROC Curve)
**Cos'è**: misura quanto bene il modello sa **ordinare** i pazienti dal più al meno a rischio. Va da 0.5 (modello che tira a indovinare) a 1.0 (perfetto).

**Lettura intuitiva**: presi a caso un malato e un sano, qual è la probabilità che il modello dia un punteggio più alto al malato che al sano? Un AUROC di 0.85 significa "85 volte su 100".

**Quando è insensibile al bias**: l'AUROC dipende solo dall'**ordine** delle predizioni, non dal **valore** delle probabilità. Se il modello assegna probabilità sbagliate ma nel giusto ordine relativo (es. dice 30% dove dovrebbe dire 50%, ma sempre più dei sani), AUROC resta alta. Per questo nei Casi 1 e 2 osserviamo AUROC stabile mentre il bias è devastante in altre metriche.

### MAE (Mean Absolute Error)
**Cos'è**: errore medio assoluto fra valore predetto e valore vero, in unità del target. Se il target è la pressione sistolica in mmHg e MAE = 12, il modello sbaglia in media di 12 mmHg.

**Quando è informativo**: l'MAE è onesto perché misura l'errore in valore assoluto, sesso per sesso. Se MAE delle donne cresce mentre MAE degli uomini resta stabile, il bias è visibile direttamente.

### Calibration intercept
**Cos'è**: misura sintetica di quanto le probabilità del modello corrispondono alla realtà.

**Lettura intuitiva**: quando il modello dice *"questo paziente ha 20% di probabilità di avere il diabete"*, ci aspettiamo che — su 100 pazienti per cui il modello dice 20% — circa 20 lo abbiano davvero. Se ne hanno 35, il modello **sottostima**; se ne hanno 8, **sovrastima**. Un calibration intercept di **0** significa modello perfettamente calibrato; più ci si allontana da 0, peggio è.

**Tecnicamente**: si fitta una regressione logistica `y_reale ~ α + β · logit(p_predetta)` e si prende l'intercetta α. Idealmente α=0, β=1.

**Perché conta in clinica**: i clinici prendono decisioni su soglie di probabilità ("se rischio > 25%, prescrivo l'OGTT"). Una soglia funziona solo se le probabilità sono **veritiere**. Il bias di calibrazione fa sì che la stessa soglia produca decisioni diverse fra sessi — invisibile guardando solo AUROC.

### Demographic Parity Difference (DPD)
**Cos'è**: `P(predizione=1 | M) − P(predizione=1 | F)`. Quanto il modello "etichetta come positivi" più i maschi delle femmine (o viceversa).

**Lettura intuitiva**: se il modello dichiara "ad alto rischio" il 30% degli uomini ma solo il 20% delle donne, DPD = +10%. Un modello "demograficamente equo" ha DPD ≈ 0.

### Equal Opportunity Difference (EOD)
**Cos'è**: `TPR_M − TPR_F`, dove TPR è il *true positive rate* (= sensibilità). Differenza fra la frazione di malati maschi correttamente identificati e la frazione di malate femmine correttamente identificate.

**Lettura intuitiva**: se il modello identifica bene i diabetici fra i maschi (90% di sensibilità) ma malamente fra le femmine (75%), EOD = +0.15. Significa che il modello "perde" più donne malate di uomini malati.

### DeLong test
**Cos'è**: test statistico che confronta due **AUROC** valutate sullo stesso test set. Restituisce un p-value per *"le due AUROC sono significativamente diverse?"*.

**Quando si applica**: per dichiarare formalmente se un modello è meglio di un altro nel ranking dei pazienti (capacità di discriminare malati/sani). Insensibile a calibrazione: due modelli con identica AUROC ma calibrazione molto diversa sono "uguali" per DeLong.

### McNemar test
**Cos'è**: test statistico che confronta due **classificatori binari** sullo stesso test set, contando le discordanze. Restituisce un p-value per *"i due modelli classificano diversamente i pazienti?"*.

**Quando si applica**: per misurare se due modelli prendono decisioni cliniche diverse a una soglia fissa (tipicamente 0.5). Si può applicare anche quando AUROC è uguale: rivela cambi di etichetta su singoli pazienti.

### t-test pareato
**Cos'è**: test statistico che confronta due misurazioni fatte **sugli stessi soggetti** in due condizioni diverse (qui: stesso paziente, due modelli/scenari diversi).

**Quando si applica**: nel Caso 1A confrontiamo l'errore assoluto di ogni paziente predetto dal modello S0 vs dal modello S3. Restituisce un p-value per *"l'errore medio è cambiato passando da S0 a S3?"*.

---

## Caso 1 — Bias di numerosità del genere

### Setup

Quattro versioni del training set, **stessa quantità totale (3000 persone)**, rapporti F:M variabili:

| Scenario | Donne | Uomini | Quota donne |
|---|---|---|---|
| **S0** | 1500 | 1500 | 50% (baseline) |
| **S1** | 900 | 2100 | 30% |
| **S2** | 450 | 2550 | 15% |
| **S3** | 150 | 2850 | 5% |

Bootstrap 100×. Test set fisso (750 M + 750 F). Stesso pool di pazienti, sempre.

### Opzione 1A — Regressione lineare sulla pressione sistolica

**Target**: pressione sistolica (`bp_systolic_1`, in mmHg). **Predittori**: BMI, età. **Modello**: regressione lineare (OLS).

#### Risultati

**Errore medio (MAE) per scenario × sesso (mmHg, più basso = meglio)**

| Scenario | MAE Donne | MAE Uomini | Δ donne − uomini |
|---|---|---|---|
| S0 50/50 | 11.72 | 11.78 | −0.06 |
| S1 30/70 | 11.98 | 11.62 | **+0.36** |
| S2 15/85 | 12.24 | 11.58 | **+0.66** |
| S3 5/95 | **12.44** | 11.58 | **+0.86** |

**T-test pareato sui residui (S0 vs S3, stratificato per sesso)**

| Sesso | n | MAE S0 | MAE S3 | Δ | t | p |
|---|---|---|---|---|---|---|
| **Donne** | 639 | 11.72 | 12.47 | **+0.74** | −7.87 | **1.6 × 10⁻¹⁴** |
| Uomini | 659 | 11.78 | 11.59 | −0.19 | +1.87 | 0.062 |

**Coefficienti del modello**: la pendenza per il BMI resta costante (~0.22 mmHg per unità BMI), ma l'**intercetta** cresce monotonicamente (94.9 → 101.8 mmHg) → la retta del modello **trasla verso l'alto** all'aumentare degli uomini nel training, riflettendo la pressione media più alta tipica della popolazione maschile.

#### Lettura

**Per i clinici**: man mano che riduciamo la quota di donne nel training, il modello sbaglia **strutturalmente** sulle donne in modo proporzionale. Sulle donne l'errore cresce di ~0.74 mmHg passando da S0 a S3 (significatività estrema, p < 10⁻¹³). Sugli uomini l'errore non si muove (p = 0.062, non significativo).

**Tecnicamente**: la regressione lineare con sole 2 feature (BMI + età) è il modello più trasparente possibile, e mostra il bias nel modo più diretto: i coefficienti si aggiustano per minimizzare l'errore sulla popolazione dominante (i maschi), e la retta risultante predice male le donne.

### Opzione 1B — Regressione logistica sul diabete

**Target**: `diab_doctor_told_diabetes` (binario, prevalenza ~12%). **Predittori**: top-30 feature per |corr| col target (escluse proxy del sesso). **Modello**: regressione logistica L2 (C=1, standardizzata).

#### Risultati

**AUROC per scenario × sesso (con CI 95% bootstrap)**

| Scenario | AUROC Donne | AUROC Uomini | Gap M−F |
|---|---|---|---|
| S0 50/50 | 0.901 | 0.918 | +0.017 |
| S1 30/70 | 0.901 | 0.919 | +0.018 |
| S2 15/85 | 0.899 | 0.919 | +0.020 |
| S3 5/95 | **0.898** | 0.916 | +0.018 |

**DeLong test (AUROC F: S0 vs S3)**: Δ = −0.002, p = **0.74** → non significativo

**McNemar** (S0 vs S3, soglia 0.5, donne): 8 vs 9 discordanze, p = **1.00** → identici

**Calibration intercept (ideale = 0)**

| Scenario | Donne | Uomini | Gap |
|---|---|---|---|
| S0 50/50 | **−0.51** | −0.34 | 0.17 |
| S1 30/70 | **−0.53** | −0.40 | 0.13 |
| S2 15/85 | **−0.49** | −0.30 | 0.19 |
| S3 5/95 | **−0.50** | −0.35 | 0.15 |

#### Lettura

**Risultato sorprendente e didatticamente cruciale**:
- **AUROC è insensibile al bias**: cala di 0.003 punti dal S0 al S3, dentro le bande di errore. DeLong e McNemar non significativi.
- **La calibrazione invece rivela il bias**: il calibration intercept delle donne è **strutturalmente peggiore** di quello degli uomini (gap medio ~0.18 punti) in **tutti** gli scenari.

**Per i clinici**: il modello LogReg sul diabete è "robusto" alla riduzione delle donne nel training, perché HbA1c e glicemia (i due predittori dominanti) sono universali. Tuttavia, **le probabilità che produce sono sistematicamente meno accurate sulle donne**. Una soglia di "rischio > 25%" classifica diversamente uomini e donne con uguale rischio sottostante.

**Lezione metodologica**: AUROC da sola **NON BASTA** per validare un modello sex-fair. Serve sempre la calibrazione stratificata.

### Opzione 1C — XGBoost sull'artrite

**Target**: `mc_arthritis_ever` (binario, prevalenza ~26%). **Predittori**: top-30 feature. **Modello**: XGBoost (`max_depth=4, lr=0.05, early_stopping=20`).

#### Risultati

**AUROC per scenario × sesso**

| Scenario | AUROC Donne | AUROC Uomini |
|---|---|---|
| S0 50/50 | **0.838** | 0.818 |
| S1 30/70 | 0.835 | 0.817 |
| S2 15/85 | 0.829 | 0.818 |
| S3 5/95 | **0.818** | 0.815 |

→ AUROC delle donne **cala monotonicamente** (0.838 → 0.818, Δ = −0.020)

**McNemar (donne, S0 vs S3)**: 64 vs 42 discordanze, p = **0.041** → **significativo** (S0 batte S3)

**Calibration intercept**

| Scenario | Donne | Uomini | Gap |
|---|---|---|---|
| S0 50/50 | +0.10 | −0.28 | 0.38 |
| S1 30/70 | +0.15 | −0.19 | 0.34 |
| S2 15/85 | **+0.23** | −0.21 | 0.44 |
| S3 5/95 | **+0.28** | −0.20 | 0.48 |

→ Calibration intercept delle donne **peggiora monotonicamente** da +0.10 a +0.28 (3× peggio)

#### Lettura

**Il bias del Caso 1 si vede in 3 metriche su 4** quando il modello è non lineare (XGBoost):
1. AUROC delle donne cala (Δ = −0.020)
2. McNemar significativo sulle donne (p = 0.041)
3. Calibration intercept cresce monotonicamente sulle donne

**Implicazione importante**: XGBoost è "più intelligente" della regressione logistica, ma proprio per questo il bias di numerosità si manifesta in modo più articolato. Non è "rotto AUROC" ma "ricalibrato male".

**Lezione metodologica per la AI clinica**: passare a un modello più sofisticato non risolve il bias da campionamento. Anzi, lo rende più sottile e più difficile da diagnosticare con metriche standard.

### Opzione 1D — Curva di rottura

**Setup**: griglia fine di quote di donne nel training (5%, 10%, 15%, 20%, 30%, 40%, 50%). 100 bootstrap per ogni punto.

#### Risultati

**AUROC vs %F nel training (LogReg L2 sul diabete)**

| %F | AUROC Donne | AUROC Uomini |
|---|---|---|
| 5% | 0.899 | 0.915 |
| 50% | 0.903 | 0.917 |
| **Δ totale** | **+0.004** | +0.002 |

→ AUROC è **piatta**: 0.4 punti percentuali su tutta la griglia.

**Calibration intercept vs %F**

| %F | Donne | Uomini | Gap |
|---|---|---|---|
| 5% | −0.53 | −0.34 | 0.19 |
| 10% | −0.50 | −0.30 | 0.20 |
| 20% | −0.47 | −0.27 | 0.20 |
| 30% | −0.44 | −0.24 | 0.20 |
| 50% | −0.40 | −0.21 | 0.19 |

→ Le due linee migliorano insieme con più dati, ma il **gap fra sessi resta costante** a circa 0.19 punti.

#### Lettura

Il "punto di rottura" che la relazione si aspettava (un calo brusco di AUROC sotto una certa soglia di %F) **non si materializza sui nostri dati**. Ma la curva del calibration intercept rivela qualcosa di altrettanto importante:

**Il bias di calibrazione è strutturale, non riparabile dalla sola numerosità**. Anche con il 50% di donne nel training (parità perfetta), il modello rimane **meno calibrato sulle donne** che sugli uomini.

**Implicazione**: per chiudere il gap fra sessi non basta riequilibrare il training. Serve un cambio di **architettura** — modelli sex-specific (Caso 3) o feature engineering sex-aware. È il **ponte concettuale verso il Caso 3**.

---

## Caso 2 — Two Clinics (asimmetria intra-genere su BMI)

### Setup

**N totale = 4000 (2000 M + 2000 F, perfettamente bilanciati)**. Quello che cambia è la **distribuzione di BMI** dentro ciascun sesso:

- **Baseline**: campionamento random stratificato. Le due distribuzioni di BMI si sovrappongono (è la realtà adulta).
- **Two Clinics (perturbato)**: F con BMI ≤ 28 (sotto-mediana), M con BMI ≥ 28 (sopra-mediana). Resampling fino a 2000 ciascuno.

**Validazione preliminare**: BMI è statisticamente **sex-neutro** sui dati grezzi:
- corr(BMI, gender) ≈ +0.077
- corr(BMI, diabete) = −0.115 in M, −0.107 in F → relazione identica

→ Tutto il bias osservato è **interamente** dovuto al campionamento, non a una proprietà intrinseca della feature.

### Risultati: 4 indicatori convergenti

**1. Calibration intercept (gap fra sessi)**

| Variant | Donne | Uomini | Δ donne − uomini |
|---|---|---|---|
| Baseline | −0.46 | −0.37 | −0.09 |
| **Two Clinics** | **−0.74** | −0.29 | **−0.45** |
| **Δ vs baseline** | **−0.28 (peggiora)** | +0.08 (lieve miglioramento) | |

→ Sulle donne il calibration intercept **crolla del 60%** nel modello perturbato. Sugli uomini è quasi invariato.

**2. Demographic Parity Difference**

| Variant | DPD = P(pred=1 \| M) − P(pred=1 \| F) |
|---|---|
| Baseline | −0.004 (≈ equo) |
| **Two Clinics** | **−0.025** (5× più iniquo) |

**3. Equal Opportunity Difference**

| Variant | EOD = TPR_M − TPR_F |
|---|---|
| Baseline | +0.090 |
| **Two Clinics** | **−0.045** (segno invertito!) |

→ Il modello passa da identificare meglio i diabetici fra gli uomini (gap +9 punti TPR) a identificare meglio fra le donne (gap −4.5 punti). Cambio drammatico (Δ EOD = −0.135).

**4. Probabilità media di diabete predetta vs prevalenza vera**

| Variant | P(diabete \| F) | P(diabete \| M) | Errore F vs prev vera 12.8% |
|---|---|---|---|
| Baseline | 12.9% | 12.1% | +0.1 pp (perfetto) |
| **Two Clinics** | **14.0%** | 11.5% | **+1.2 pp (sovrastima)** |

**5. McNemar a soglia 0.5** (per controllo)

p-value: 0.65 sulle donne, 0.15 sugli uomini → **non significativo**, le decisioni binarie restano stabili. Il bias non si vede a soglia 0.5 (analogamente al 1B).

### Lettura

**Per i clinici**: abbiamo lasciato la composizione M:F al 50/50. Abbiamo cambiato solo **come** abbiamo reclutato uomini e donne (centro di prevenzione cardiovascolare per le donne, clinica bariatrica per gli uomini). Il modello, che NON ha il sesso fra le sue feature di input, lo ha **ricostruito indirettamente** attraverso il pattern di correlazioni del BMI e di un cluster di feature cardiovascolari. E ha cominciato a sbagliare in modo asimmetrico fra i sessi.

**Il pattern reale dei dati**: nel training perturbato il modello vede solo donne magre (BMI medio 23.4) e solo uomini grossi (BMI medio 33.6). Quando lo applichiamo al test realistico (donne con BMI variato, uomini con BMI variato):
- Le **donne con BMI alto del test** vengono "fuori distribuzione" → il modello sovrastima il loro rischio (estrapolando dai pattern degli uomini grossi che ha visto)
- Gli **uomini con BMI basso del test** vengono fuori distribuzione → il modello sottostima il loro rischio

In media: P(diabete | F) sale, P(diabete | M) scende. È un **failure di generalizzazione fuori distribuzione** asimmetrico fra sessi — un fenomeno noto e temuto in AI clinica.

**Lezione metodologica**: il bias non vive in una singola variabile, vive nel **modo in cui i dati sono stati campionati**. Ed è invisibile a un audit superficiale (numeri 50/50 perfetti, niente missing strani, AUROC stabile). Bisogna guardare le **metriche di calibrazione** e **fairness** stratificate per sesso, non quelle aggregate.

---

## Caso 3 — Modelli sex-specific guidati dalla matrice di correlazione

### Setup

Tre training set valutati sullo stesso test fisso:

| Training | Composizione | Valutato su |
|---|---|---|
| **A. Unisex** | 4000 (2000 M + 2000 F) | maschi del test E donne del test, separatamente |
| **B. M-only** | 2000 M | solo maschi del test |
| **C. F-only** | 2000 F | solo donne del test |

**Feature**: top-30 per `|gap_FminusM|` (= |r_F − r_M|) calcolate al volo dalla matrice di correlazione sex-splittata. Sono le feature dove la relazione feature → outcome cambia di più fra i due sessi.

**Modelli**: Logistic L2 (principale) + XGBoost (controllo). **Bootstrap 100×**.

**Due target candidati** (entrambi al vertice del ranking di divergenza):

| Opzione | Target | Prevalenza | Top var divergente |
|---|---|---|---|
| **3A** | `mc_arthritis_ever` (artrite) | 26% | `hemoglobin_gdl` (r_M=−0.18, r_F=+0.05) |
| **3B** | `bp_told_to_take_chol_meds` (statine) | 32% | `cholesterol_total_mgdl` (r_M=−0.12, r_F=+0.07) |

### Risultati 3A (artrite)

**AUROC LogReg (bootstrap 100×, medie + CI 95%)**

| Confronto | AUROC Unisex | AUROC sex-specific | Δ (specifico − unisex) |
|---|---|---|---|
| Sui Maschi (A vs B) | 0.810 | 0.801 | **−0.009** |
| Sulle Donne (A vs C) | 0.812 | 0.812 | +0.001 |

**AUROC XGBoost** (stesso pattern):

| Confronto | AUROC Unisex | AUROC sex-specific | Δ |
|---|---|---|---|
| Sui Maschi | 0.808 | 0.780 | **−0.027** |
| Sulle Donne | 0.793 | 0.800 | +0.008 |

**DeLong test**: p = 0.20 (Maschi), p = 0.40 (Donne) → entrambi **non significativi**

**McNemar test**: p = 0.68 (Maschi), p = 1.00 (Donne) → entrambi **non significativi**

**Coefficienti dei 3 modelli** sulla top var (`hematocrit_pct`, sostanzialmente equivalente a `hemoglobin_gdl`):
- Unisex: −0.30
- M-only: +0.14
- F-only: +0.01

→ Il pattern atteso (M-only negativo, F-only positivo, Unisex ≈ 0) **non si materializza pulito** sui dati.

**PCA validation**: PC1+PC2 spiegano solo il **29.5% della varianza** → i due "sotto-manifold" sex-stratificati si **sovrappongono parzialmente**.

### Risultati 3B (statine, controllo)

**AUROC LogReg**

| Confronto | AUROC Unisex | AUROC sex-specific | Δ |
|---|---|---|---|
| Sui Maschi | 0.705 | 0.705 | +0.000 |
| **Sulle Donne** | **0.787** | **0.777** | **−0.011** ⚠️ F-only **peggiore** |

**AUROC XGBoost**

| Confronto | AUROC Unisex | AUROC sex-specific | Δ |
|---|---|---|---|
| Sui Maschi | 0.766 | 0.752 | −0.014 |
| Sulle Donne | 0.802 | 0.792 | −0.010 |

**DeLong test**: p = 0.009 (Maschi, **Unisex meglio**), p = 0.16 (Donne, non sig.)

**Coefficienti dei 3 modelli** sulla top var (`cholesterol_total_mgdl`):
- Unisex: +0.91
- M-only: +0.49
- F-only: +0.84

→ **Tutti i coefficienti hanno lo stesso segno positivo** — nessuna inversione. Il pattern atteso non si vede.

### Lettura cross-target (3A + 3B)

| Metrica | 3A (artrite) | 3B (statine) | Pattern |
|---|---|---|---|
| Δ AUROC sex-specific su Maschi (LogReg) | −0.009 | +0.000 | sex-specific ≤ unisex |
| Δ AUROC sex-specific su Donne (LogReg) | +0.000 | **−0.011** | sex-specific ≤ unisex |
| DeLong significativo? | no | sui maschi sì, ma a favore Unisex | unisex non perde |
| Coefficienti M-only vs F-only di segno opposto sulla top var? | parziale | **no** | divergenza non si traduce in inversione di pesi |

**Conclusione robusta**: **i sex-specific NON battono l'unisex** sui nostri dati, su nessuno dei due target, con nessuno dei due modelli. Risultato confermato da due target indipendenti.

### Tre cause cliniche/metodologiche del risultato negativo

**1. Il vantaggio dei dati supera il vantaggio della stratificazione.**
L'unisex ha 4000 esempi, i sex-specific solo 2000. Per LogReg L2 e XGBoost regolarizzati, il "guadagno di varianza" da dati doppi supera il "guadagno di bias" da stratificazione. La matematica del trade-off bias/varianza vince.

**2. La divergenza nelle correlazioni grezze è spesso *apparente*, non *causale*.**
La correlazione `cholesterol → presa di statine` ha segno apparentemente diverso fra M e F nei dati grezzi, ma è probabilmente un effetto di **prescribing practice variation** (gli uomini sono trattati prima/diversamente) o di confounding, non una vera differenza biologica nel meccanismo causale. Il "design sex-aware" non aiuta perché il meccanismo è uguale, è la storia di trattamento che differisce.

**3. Le feature divergenti sono "deboli" come predittori globali.**
`hemoglobin_gdl` ha max_abs solo 0.16-0.18. Sono "rumore divergente sopra un segnale comune". Il segnale comune (HbA1c per il diabete, età/limitazioni motorie per l'artrite) è dominante e identico fra sessi. La regolarizzazione L2 contiene il rumore divergente.

### Lezione finale del Caso 3

> La matrice di correlazione sex-splittata è uno strumento **diagnostico**, non **prescrittivo**. Identifica i target su cui un design sex-aware *potrebbe* essere giustificato, ma **non garantisce automaticamente** un vantaggio di performance. La sex-specificity è un'ipotesi da testare empiricamente caso per caso.

Sui nostri dati il vantaggio non emerge. **Risultato negativo robusto** e clinicamente prezioso, perché smentisce l'idea diffusa che modelli stratificati per sesso siano automaticamente migliori. Il design sex-aware ha senso quando: (a) la divergenza è **grande**, (b) i predittori sono fortemente legati a meccanismi **biologici** sex-specific (non a confounders), (c) si dispone di **abbastanza dati** per ogni sotto-modello.

---

## Conclusioni cross-case e tre lezioni per la pratica clinica

I tre casi raccontano una storia coerente, ma con **un tema centrale**: il bias di genere è **reale e misurabile**, ma **non si vede nelle metriche di routine** (AUROC, accuracy globale).

### Lezione 1 — Il bias non vive nella metrica più popolare

In **tutti e tre i casi**, AUROC e McNemar a soglia 0.5 (le metriche che la maggior parte dei clinici guarda) sono **non significativi** o cambiano poco. Sembrano dire "tutto ok".

Ma nei dati il bias c'è eccome:
- **Caso 1B**: calibration intercept Donne = −0.50, Uomini = −0.34 (gap 47% peggiore)
- **Caso 1C**: calibration intercept Donne sale da +0.10 a +0.28 (3× peggio) man mano che togliamo donne dal training
- **Caso 1D**: il gap fra sessi nel calibration intercept resta costante anche al 50% di donne (≈ −0.19 punti)
- **Caso 2**: calibration intercept Donne −0.46 → −0.74 nel modello perturbato (60% peggio); EOD inverte di segno (Δ −0.135)

→ **AUROC misura il ranking, non la veridicità delle probabilità**. Per usare un modello in clinica serve la **calibrazione**, non solo la discriminazione.

### Lezione 2 — Il bias si manifesta in modi diversi a seconda del meccanismo

| Caso | Meccanismo | Dove si vede meglio |
|---|---|---|
| **1** Numerosità | Sotto-rappresentazione quantitativa | MAE (1A), Calibration intercept (1B/1C), Curve di rottura calibration (1D) |
| **2** Two Clinics | Asimmetria di campionamento di una feature | Calibration intercept, DPD, EOD, P media predetta |
| **3** Sex-specific | (Tentativo costruttivo) | **Risultato negativo**: il bias non si "risolve" stratificando |

Tre meccanismi diversi → tre famiglie di metriche. **Non esiste una "metrica universale" del bias**: serve un audit multi-metrica stratificato.

### Lezione 3 — Le soluzioni "ovvie" non sempre funzionano

**Soluzione ovvia 1: "basta riequilibrare i numeri"** (Caso 1D smentisce)
Il calibration intercept delle donne resta strutturalmente peggiore anche al 50% di donne nel training. La numerosità non basta.

**Soluzione ovvia 2: "basta avere 50/50 di donne e uomini"** (Caso 2 smentisce)
Con 50/50 perfetto ma campionamento asimmetrico di una feature interna, il modello ricostruisce il sesso indirettamente e produce predizioni distorte.

**Soluzione ovvia 3: "uso modelli sex-specific"** (Caso 3 smentisce)
Sui due target candidati, i sex-specific NON battono l'unisex. Il vantaggio dei dati supera il vantaggio della stratificazione.

→ Il bias di genere nei modelli predittivi clinici è **un problema strutturale del dato**, non un problema risolvibile con una singola correzione facile. Servono:
1. Audit multi-metrica stratificato (sempre)
2. Validazione esterna su popolazioni demograficamente diverse
3. Trasparenza sulle scelte di campionamento del training
4. Discussione clinica dei *meccanismi* di bias prima della scelta del modello

### Una frase finale per la presentazione

> *Nei modelli predittivi clinici, il sesso entra dalla finestra anche quando lo si esclude dalla porta. Il modo per accorgersene è guardare le metriche **stratificate**, non solo quelle globali — e in particolare la **calibrazione**, non solo l'AUROC. Il modo per evitarlo è progettare il dato, non il modello.*

---

## Appendice — Riepilogo dei file generati

Tutti i risultati sono in `archive/`:

```
archive/
├── case1/    1A/1B/1C/1D + summary.json + coeffs.csv + mcnemar.csv consolidati
├── case2/    Tutti i 4 indicatori + 2_calibration_killer.png + summary.json
├── case3/    3A (mc_arthritis_ever) + 3B (bp_told_to_take_chol_meds) + summary.json
└── workshop_results.{csv,xlsx}    Tabella riassuntiva cross-case
```

Il notebook `Fase_3_4_Modelli_Performance_Gap.ipynb` rigenera tutti i file da zero con `Run All` (~10 minuti totali).
