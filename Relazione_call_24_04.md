# Workshop Gender Bias — Scheda Operativa dei Tre Casi

**Dataset di lavoro**: NHANES 2013-2014, master deduplicato (700 colonne, soglia |r| >= 0.999), restretto agli adulti >= 18 (N=6.113; 2.916 M / 3.197 F).
**File gia' pronti**: `master_dedup_numeric.csv`, `corr_dedup_pearson.{csv,xlsx}`, `corr_with_gender_full.csv`, `workshop_validation/`.

## 0. Pre-flight comune (una sola volta)

| Step | Operazione | Razionale |
|---|---|---|
| 0.1 | Restringi adulti `age_years >= 18` | Allinea il dominio (no pediatrico) ai target clinici |
| 0.2 | Codifica binari NHANES: `1->1`, `2->0`, `3/7/9->NaN` (per `diab_*`, `mc_*`, `bp_told_*`) | Senza questo, la prevalenza diventa >100% e i coefficienti hanno segno invertito |
| 0.3 | Drop variabili **structural-missing-by-sex** (`rh_*`, `imm_hep_b_first_dose_given_site`, ...) | Altrimenti il modello "vede" il sesso dai NaN e tutti i risultati sono inquinati |
| 0.4 | Imputazione **dentro** ogni training fold (median) | Evita leakage |
| 0.5 | **Test set fisso e bilanciato** (1.500: 750 M / 750 F, distribuzioni di eta' e BMI realistiche, mai toccato) | Confronti coerenti tra i tre casi |
| 0.6 | Seed = 42, `n_bootstrap = 100` per CI, log salvato | Riproducibilita' |

Output cartelle attese:
```
archive/case1/  archive/case2/  archive/case3/
```
Ogni cartella conterra': i CSV con i numeri, i PNG per slide, e un `summary.json`.

---

# CASE 1 - Bias di numerosita' del genere

**Domanda di ricerca.** *A parita' di tutto il resto, il solo cambio del rapporto M/F nel training sposta in modo misurabile i parametri e le predizioni del modello?*

**Ipotesi.** Il sesso porta con se' una distribuzione condizionata diversa delle feature; quando il modello stima un solo set di parametri, i pesi vengono "tirati" verso il sesso maggioritario, peggiorando le predizioni sul minoritario.

**Razionale didattico.** E' la dimostrazione piu' elementare del bias: nessuna manipolazione fine, solo conteggio. Lo si capisce in trenta secondi e prepara il pubblico ai casi 2 e 3.

## Setup comune del Caso 1

- **Pool**: pool adulto post-pre-flight.
- **Manipolazione**: 4 scenari, **N totale fisso = 3.000**, rapporto F:M variabile:
  - **S0**: 50/50 (1500 F, 1500 M) - *baseline*
  - **S1**: 30/70 (900 F, 2100 M)
  - **S2**: 15/85 (450 F, 2550 M)
  - **S3**:  5/95 (150 F, 2850 M)
- **Test set**: il test fisso 50/50 dello step 0.5.
- **Bootstrap**: 100 ripetizioni su seed diversi -> IC al 95% sui delta.

## Opzioni di esecuzione

### Opzione 1A - *Linear regression* "killer slide" (raccomandata come apertura)

| Voce | Specifica |
|---|---|
| Target | `bp_systolic_1` (continuo, mmHg) |
| Predittori | `bmi`, `age_years` |
| Modello | OLS bivariata; in slide tieni solo `bmi` per il grafico 2D |
| Metrica | MAE su F, MAE su M, MAE globale, slope e intercept |
| Test | t-test pareato sui residui F vs M tra S0 e S3; CI bootstrap su slope/intercept |

**Output per slide**
- Scatter `BMI -> SBP` con **4 rette** sovrapposte (una per scenario), colorate dal blu al rosso -> mostra la **rotazione progressiva** della retta verso il pattern maschile.
- Tabella: |slope|, intercept, MAE F, MAE M nei 4 scenari.
- Una frase: *"abbiamo cambiato solo il numero di donne. Nient'altro. Il modello non e' piu' lo stesso."*

**Pro**: visivamente immediato, niente ML esoterico.
**Contro**: target continuo poco "clinico" per un pubblico che pensa al rischio di morte.

---

### Opzione 1B - *Logistic regression* su outcome binario

| Voce | Specifica |
|---|---|
| Target | `diab_doctor_told_diabetes` (binario, prev ~12% adulti) |
| Predittori | top-30 feature per |corr| col target, escluse quelle proxy del sesso (`hh_ref_gender`, ecc.) |
| Modello | Logistic L2 (`C=1`), standardizzazione |
| Metriche | AUROC globale, **AUROC stratificato per sesso**, TPR/FPR/PPV per sesso, Brier per sesso, calibration intercept |
| Test | DeLong su AUROC F (S0 vs S3); McNemar sulle predizioni di test stratificate per sesso |

**Output per slide**
- Curve ROC sovrapposte per sesso, una per scenario -> la curva delle donne si deforma e cala.
- Barre AUROC F e AUROC M lungo i 4 scenari -> divergenza progressiva (gap M - F che cresce).
- Tabella numerica.

**Pro**: target clinico pulito, AUROC e' metrica universale.
**Contro**: l'effetto su AUROC globale e' spesso modesto - il "trucco" e' guardare la stratificazione per sesso.

---

### Opzione 1C - *Gradient Boosting* (XGBoost o LightGBM)

| Voce | Specifica |
|---|---|
| Target | `mc_arthritis_ever` (binario, prev ~26% - lo stesso del Caso 3, per *narrative consistency* tra slide) |
| Predittori | stesso pool dell'opzione 1B |
| Modello | XGBoost, `early_stopping=20`, `max_depth=4`, `lr=0.05` |
| Metriche | come 1B + SHAP per sesso |
| Test | DeLong, McNemar |

**Output per slide**
- Stesso plot della 1B (ROC stratificate, AUROC vs scenario).
- **SHAP summary** del modello S3 stratificato per sesso -> la feature piu' importante per le donne e' penalizzata; segnala che il bias persiste **anche con modelli non lineari**.

**Pro**: blinda il messaggio ("non e' colpa del modello lineare").
**Contro**: aggiunge complessita' computazionale.

---

### Opzione 1D - *Curva di rottura* (bonus, una sola slide riassuntiva)

| Voce | Specifica |
|---|---|
| Stesso modello (consigliato 1B) | Logistic L2 |
| Manipolazione | griglia fine di % F nel train: 5, 10, 15, 20, 30, 40, 50 -> 14 punti |
| Output | curva `AUROC_F` e `AUROC_M` vs `%F nel train`, con CI bootstrap |
| Lettura | identifica empiricamente il **punto di rottura** (es. "sotto il 20% di donne, AUROC_F crolla del 5%") |

**Per la slide**: una linea che cala bruscamente e' didatticamente potente.

## Pitfall del Caso 1

- Tenere fisso **N totale**, non solo le proporzioni, altrimenti confondi *sample size* e *imbalance*.
- Imputazione e standardizzazione **per fold**, non sul pool: senza questo gli scenari S0 e S3 condividono informazione e gli intervalli di confidenza sono troppo stretti.
- Riportare sempre la metrica stratificata, non solo quella globale: il bias del Caso 1 e' prevalentemente di **sottogruppo**.

---

# CASE 2 - Bias di rappresentativita' intra-genere ("Two Clinics")

**Domanda di ricerca.** *Tenendo M=F nei numeri ma squilibrando in modo asimmetrico la distribuzione di una feature interna, posso produrre un classifier sistematicamente sbilanciato per sesso?*

**Ipotesi.** Se nel training le donne e gli uomini occupano regioni quasi disgiunte di una feature **predittiva ma di per se' sex-neutra**, il modello associa quella feature al sesso e produce predizioni distorte sulla popolazione realistica.

**Razionale didattico.** E' il caso piu' sottile: non c'e' squilibrio numerico (50/50), non ci sono missing strani, le statistiche descrittive base passano un audit superficiale. Eppure il modello fallisce. E' il messaggio politico forte del workshop: *"il bias non e' colpa di una variabile, e' colpa di come l'hai campionata."*

## Setup comune del Caso 2

- **N**: 4.000 (2.000 M, 2.000 F).
- **Test set**: il test fisso bilanciato dello step 0.5, distribuzione realistica della feature di perturbazione.
- **Modelli**: Logistic L2 + Gradient Boosting.
- **Bootstrap 100x**.

## Opzioni di esecuzione (4 case study, in ordine di forza didattica)

### Opzione 2A - Asimmetria di **BMI** ("Two Clinics")

**Validazione empirica preliminare** (gia' calcolata):
- corr(BMI, gender) = **+0.077** sul dato originale -> BMI e' quasi indipendente dal sesso.
- corr(BMI, diabete) = -0.115 in M, -0.107 in F -> relazione **identica** nei due sessi.
- corr(BMI, chol_meds) = +0.121 in M, +0.111 in F -> identica.
- -> **BMI e' feature neutra: il bias che produrremo e' interamente nel sampling.**

| Voce | Specifica |
|---|---|
| Target | `diab_doctor_told_diabetes` (binario, prev ~12%) |
| Asse di perturbazione | `bmi` |
| Storia | "Le donne sono state reclutate in un centro di prevenzione cardiovascolare (BMI <= 28); gli uomini in una clinica bariatrica (BMI >= 28). N e proporzioni invariati." |
| Manipolazione | F: BMI <= 28 (sotto-mediana F); M: BMI >= 28 (sopra-mediana M); resampling per arrivare a N=2.000 in ciascuno |
| Test | McNemar baseline-vs-perturbato stratificato per sesso; gap di calibration intercept; demographic parity difference |
| Atteso | P(diabete | F) crolla, P(diabete | M) sale; McNemar significativo soprattutto sulle donne; SHAP del modello perturbato mostra che il sesso e' "ricostruito" via BMI |

**Pro**: asse validato come "neutro", storia plausibile, plot di density BMI x sex pre/post devastante.
**Contro**: nessuno tecnico - e' il case study piu' solido.

---

### Opzione 2B - Asimmetria di **fumo** ("Tobacco Asymmetry")

**Validazione preliminare consigliata** (rapida, prima del workshop): verifica che `smoke_cigarette_use` o `smoke_cigarette_pack_years` correli col target scelto in modo simile per M e F.

| Voce | Specifica |
|---|---|
| Target | `mc_chronic_bronchitis_ever` (prev 5.5%; il ranking ci dice che lo smoking e' top-gap gia' di base - *cum grano salis*) o `mc_heart_attack_ever` |
| Asse | `smoke_cigarette_use` o `smoke_pack_years` |
| Storia | "Il braccio femminile e' stato reclutato in un programma di screening per non-fumatrici; quello maschile in una coorte industriale (alta esposizione)." |
| Manipolazione | F: solo never-smoker; M: solo current/heavy smoker |
| Test | McNemar, gap AUROC per sesso, equal opportunity difference |
| Atteso | Il modello impara "M = polmone compromesso, F = sano" -> falsi negativi devastanti su F fumatrici nel test set |

**Pro**: storia molto realistica (selection bias di reclutamento industria vs prevenzione).
**Contro**: il fumo **non e'** sex-neutro nel dato grezzo (gli uomini fumano piu' delle donne in NHANES), quindi la perturbazione amplifica un asse gia' asimmetrico - il messaggio "il bias nasce dal sampling" e' meno cristallino.

---

### Opzione 2C - Asimmetria di **stato socioeconomico** ("SES Mismatch")

| Voce | Specifica |
|---|---|
| Target | `bp_told_to_take_chol_meds` o `diab_doctor_told_diabetes` |
| Asse | `family_income_to_poverty_ratio` o `education_20_plus` |
| Storia | "Le donne reclutate da centri urbani convenzionati con assicurazioni private; gli uomini da cliniche pubbliche di periferia." |
| Manipolazione | F: poverty ratio > 3 (alto reddito); M: poverty ratio < 1 (sotto soglia poverta') |
| Test | McNemar, calibration per sesso, fairness metrics |
| Atteso | Inversione apparente del rischio cardiovascolare per sesso, perche' SES e' confounder pesante |

**Pro**: storia sociologicamente potente, apre discussione su determinanti sociali della salute.
**Contro**: SES e' correlato con quasi tutto -> l'effetto e' grande ma il *meccanismo* e' meno chiaro per il pubblico tecnico (potresti spiegarlo come un effetto puramente di confounding piuttosto che di sampling).

---

### Opzione 2D - Asimmetria di **eta' all'interno del sesso** (l'esempio originale dell'utente)

Mantieni questa come **case study di backup** o come prova preliminare didattica. E' l'esempio piu' scontato; lo lasci per chi vuole vederlo, ma le opzioni 2A-2C sono superiori.

## Pitfall del Caso 2

- Verificare *prima* che l'asse di perturbazione sia **realisticamente neutro** sul dato grezzo (corr(asse, gender) bassa) - altrimenti il messaggio e' che hai amplificato un bias preesistente, non creato un bias da campionamento.
- Mai presentare solo metriche globali: AUROC e accuracy globali possono restare quasi invariati. Il bias vive nelle metriche stratificate.
- Riportare sempre la **distribuzione dell'asse per sesso prima/dopo**: senza quel plot la slide e' poco convincente.

---

# CASE 3 - Modelli sex-specific guidati dalla matrice di correlazione

**Domanda di ricerca.** *Esistono target dove la struttura predittiva e' realmente diversa tra M e F? Se si', due modelli sex-specific battono un modello unisex sulla rispettiva popolazione, in modo statisticamente significativo?*

**Ipotesi.** La matrice di correlazione, calcolata separatamente per i due sessi, identifica target con **inversioni di segno** o **ampi gap di intensita'** nelle relazioni feature->outcome. Su questi target, il modello unisex media i pesi e fa torto a entrambi i sessi.

**Razionale didattico.** E' il caso "costruttivo" del workshop: dopo aver mostrato come il bias rovina i modelli (Casi 1 e 2), si mostra come la matrice di correlazione e la PCA *guidano* la scelta di un design sex-aware.

## Setup comune del Caso 3

- **Tre training set**:
  - **A. Unisex 50/50** - 4.000 (2.000 M + 2.000 F)
  - **B. M-only** - 2.000 M
  - **C. F-only** - 2.000 F
- **Tre test set di valutazione**:
  - A valutato sulle popolazioni M e F del test fisso, *separatamente*
  - B valutato solo sui maschi del test
  - C valutato solo sulle femmine del test
- **Feature**: top-30 predittori del target scelto, selezionati con il criterio `max(|r_M|, |r_F|)` calcolato sulla matrice di correlazione sex-splittata (file `case3_topvars_<target>.csv`).
- **Modello principale**: Logistic L2 (interpretabile, confrontabile via DeLong/McNemar).
- **Modello secondario**: Gradient Boosting per blindare il messaggio.

## Opzioni di target (tutte gia' validate sul ranking empirico)

### Opzione 3A - `mc_arthritis_ever` (artrite, raccomandata)

| Indicatore | Valore |
|---|---|
| Prevalenza adulti | 26% |
| Score divergenza top-50 | 0.114 |
| Top var divergente | `hemoglobin_gdl` (r_M=-0.179, r_F=+0.053) |
| Plausibilita' clinica | Artrite reumatoide ~3x piu' frequente in F; nei maschi associata ad anemia di malattia, in F l'Hb e' gia' "bassa" per il ciclo mestruale -> il segnale si confonde |

**Pro**: prevalenza alta, inversione di segno netta, letteratura molto solida.
**Contro**: target auto-riportato, eterogeneo (RA + OA insieme).

---

### Opzione 3B - `bp_told_to_take_chol_meds` (uso di statine)

| Indicatore | Valore |
|---|---|
| Prevalenza adulti | 32% |
| Score divergenza top-50 | **0.115** (top assoluto) |
| Top var divergente | `cholesterol_total_mgdl` (r_M=-0.125, r_F=+0.064) |
| Plausibilita' clinica | Pre-menopausa l'estrogeno protegge il profilo lipidico delle donne; la prescrizione di statine non segue solo il valore di colesterolo come negli uomini -> mappa feature->Y davvero diversa |

**Pro**: massimo score di divergenza; inversione di segno potente.
**Contro**: il target e' **prescrittivo, non patologico** - e' proxy comportamentale del medico. Spiegabile ma richiede una slide narrativa in piu'.

---

### Opzione 3C - `diab_doctor_told_diabetes` (diabete)

| Indicatore | Valore |
|---|---|
| Prevalenza adulti | 12% |
| Score divergenza top-50 | 0.098 |
| Top var divergente | `diab_taking_insulin` (r_M=+0.471, r_F=+0.266) |
| Plausibilita' clinica | Differenze note nella terapia insulinica tra sessi; profilo metabolico (massa grassa viscerale) diverso |

**Pro**: target universalmente riconosciuto dal pubblico, didatticamente comodo.
**Contro**: la divergenza e' di **intensita'**, non di **segno** - meno spettacolare nelle slide.

---

### Opzione 3D - `mc_chronic_bronchitis_ever` (bronchite cronica)

| Indicatore | Valore |
|---|---|
| Prevalenza adulti | 5.5% |
| Score divergenza top-50 | 0.080 |
| Top var divergente | `smoke_cigar_past_5d` (r_M=-0.018, r_F=-0.176) |
| Plausibilita' clinica | Le donne fumatrici svilupperebbero bronchite cronica con esposizione minore (vulnerabilita' polmonare); negli uomini il segnale del fumo e' diluito dalla concorrenza di altre esposizioni |

**Pro**: storia clinica forte (vulnerabilita' polmonare F).
**Contro**: prevalenza bassa -> il classifier richiede classi sbilanciate, soglia di decisione piu' delicata.

## Cosa misuri nel Caso 3 (uguale per qualsiasi opzione di target)

1. **Coefficienti dei tre modelli affiancati** (Logistic): per ognuna delle 30 feature, tre barre - Unisex / M-only / F-only. Atteso: per la top var divergente (es. `hemoglobin_gdl`), barra M-only negativa, F-only positiva, Unisex ~ 0. **Slide chiave**.
2. **AUROC** per modello e popolazione, con CI bootstrap: B > A su M; C > A su F.
3. **DeLong** sulle AUROC: A-M vs B sui maschi, A-F vs C sulle femmine - due p-value.
4. **McNemar** sulle predizioni binarie (stessa coppia di confronti) -> conferma che le decisioni cambiano materialmente.
5. **Calibration plot** per sesso: il modello unisex tende a sovrastimare in un sesso e sottostimare nell'altro.
6. **PCA validation**: PC1 dello spazio delle 30 feature stratificata per sesso -> mostra che i due "sotto-manifold" non si sovrappongono completamente (la PCA *aveva ragione* a indicare la sex-specificity).

## Pitfall del Caso 3

- Standardizzare **separatamente** sui tre training set; poi applicare ciascuno scaler al proprio sotto-test.
- Usare la stessa lista di 30 feature per i tre modelli (non rifare feature selection separata, altrimenti confronti non comparabili).
- Sui maschi del test, NON valutare il modello F-only e viceversa - la slide deve dichiarare con chiarezza "ogni modello sulla sua popolazione".
- Riportare sempre la **prevalenza nel test set di destinazione**: AUROC e' insensibile, ma calibration e PPV no.

---

# Mappa di consegna (deliverable)

| Caso | Slide attese | File di codice | File di output |
|---|---|---|---|
| 1 | 4 (apertura, A/B/C, riepilogo curva) | `_case1.py` | `archive/case1/{coeffs.csv, roc_*.png, auroc_curve.png, mcnemar.csv}` |
| 2 | 4 (storia, density BMI, P shift, McNemar) | `_case2.py` | `archive/case2/{density_pre_post.png, pred_shift.png, fairness_metrics.csv}` |
| 3 | 4 (matrice corr scatter, coeffs affiancati, ROC per sesso, DeLong/McNemar) | `_case3.py` | `archive/case3/{coeffs_3models.png, roc_per_sex.png, delong_mcnemar.csv}` |
| Tutti | 1 finale di sintesi | `_workshop_summary.py` | `archive/workshop_results.xlsx` (tabella riassuntiva) |

## Sequenza di esecuzione consigliata

1. Pre-flight (scrivere `_preflight.py`, salvare `adult_clean.parquet` e il test fisso).
2. Caso 1 - opzione 1A in apertura, poi 1B/1C, poi 1D come bonus.
3. Caso 2 - opzione 2A "Two Clinics" come case study principale; lasciare 2B/2C come slide di approfondimento opzionali.
4. Caso 3 - opzione 3A `mc_arthritis_ever` come default; 3B come variante "max divergenza".
5. Slide finale: tabella unica con tutti i p-value.

## Decisioni che servono prima di scrivere il codice

| Decisione | Default proposto | Alternative |
|---|---|---|
| Caso 1, opzioni da implementare | 1A + 1B + 1D | aggiungere 1C |
| Caso 2, opzione | 2A (BMI) | 2B (fumo) - richiede check di neutralita' |
| Caso 3, target | 3A (`mc_arthritis_ever`) | 3B (statine) |
| Modello secondario | Gradient Boosting | Random Forest |
| Test di equita' formali | Equal Opportunity Difference + Demographic Parity | aggiungere Predictive Parity |
