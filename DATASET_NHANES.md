# Il dataset: NHANES 2013–2014

## Cos'è NHANES

**National Health and Nutrition Examination Survey** — indagine statunitense condotta a cicli biennali dal CDC (Centers for Disease Control and Prevention). Ogni ciclo raccoglie dati su un campione rappresentativo della popolazione civile adulta non istituzionalizzata degli Stati Uniti.

Non è solo un'intervista: si articola in **tre livelli di raccolta**:

| Livello | Dove | Cosa raccoglie |
|---------|------|----------------|
| Questionario | A domicilio | Storia clinica auto-riferita, stile di vita, alimentazione, condizioni mediche dichiarate |
| Visita fisica | Centro mobile (MEC) | Misure fisiche: pressione arteriosa, BMI, forza della presa, circonferenze, diametri |
| Laboratorio | Centro mobile (MEC) | Prelievi ematici e urinari: HbA1c, colesterolo, emocromo, biomarcatori |

---

## Il dataset che usiamo

| Parametro | Valore |
|-----------|--------|
| Ondata | 2013–2014 |
| Totale persone | 10.175 |
| Adulti ≥18 anni (quelli usati) | **6.113** |
| Split di genere | 48% uomini — 52% donne |
| Variabili (colonne/feature) | **~1.800** |

Le ~1.800 variabili sono l'insieme di tutte le colonne del dataset — domande del questionario, misure fisiche e valori di laboratorio. In termini di machine learning: sono le **feature**, gli ingredienti che il modello usa per imparare.

---

## I 6 domini clinici

### Demografico e socio-economico
Sesso, età, etnia, reddito familiare, livello di istruzione.

### Misure fisiche
- **BMI** (Body Mass Index / Indice di Massa Corporea): peso(kg) ÷ altezza(m)². Soglie standard: <18.5 sottopeso · 18.5–24.9 normopeso · 25–29.9 sovrappeso · ≥30 obesità. Non misura la composizione corporea ma è la misura di adiposità più usata in clinica e nelle survey epidemiologiche.
- Pressione arteriosa sistolica e diastolica
- Circonferenza vita e diametri addominali sagittali (proxy di adiposità viscerale)
- Forza della presa al dinamometro (grip strength, in kg) — indicatore di forza muscolare e funzionalità fisica

### Esami di laboratorio
- **HbA1c** (emoglobina glicata, %): riflette il controllo glicemico medio degli ultimi 2–3 mesi. È il predittore più forte del diabete — nelle nostre feature è il primo per correlazione.
- Colesterolo totale, LDL, HDL
- Emocromo: emoglobina (g/dL), ematocrito (%) — con range fisiologici normali diversi tra uomini e donne
- Biomarcatori renali ed epatici

### Condizioni mediche auto-riferite
Diagnosi riferite dal paziente, tipo "Le ha mai detto un medico che ha...". Esempi:
- `bp_ever_high_bp` — ipertensione mai diagnosticata
- `bp_chol_ever_high` — colesterolo mai alto
- `mc_arthritis_ever` — artrite mai diagnosticata ← **target Caso 1B**

### Stile di vita
Alimentazione (compresa spesa in fast food e ristoranti), attività fisica, fumo, alcol, sonno, olfatto, udito.

### Salute riproduttiva
Gravidanza, allattamento, salute prostatica. **Rimosse dal modello** perché presenti strutturalmente solo per un sesso: se lasciate, il modello impara il sesso dai valori mancanti sistematici, prima ancora di vedere il dato clinico.

---

## Come scegliamo il target di classificazione

NHANES **non ha una label predefinita** — è una survey epidemiologica generale, non un dataset costruito per un task specifico. Siamo noi a scegliere quale variabile diventa il target di classificazione binaria.

Nei due casi del workshop:

| Caso | Target scelto | Variabile | Prevalenza |
|------|--------------|-----------|------------|
| Caso 1 (LogReg) | Diabete | `diab_doctor_told_diabetes` | ~12% |
| Caso 1B (XGBoost) | Artrite | `mc_arthritis_ever` | ~26% |
| Caso 2 (Two Clinics) | Diabete | `diab_doctor_told_diabetes` | ~12% |

Entrambe le variabili sono risposte al questionario ("Le ha mai detto un medico che ha...?") che il codice converte in binario con il recode NHANES: **1 → 1 (sì), 2 → 0 (no)**, eliminando i codici 7/9/NaN (non sa / non risponde).

> **Nota tecnica**: senza questo recode la prevalenza del diabete apparirebbe al ~40% invece del 12%, perché il codice 2 verrebbe letto come valore numerico positivo anziché come "no".

---

## Le feature nel codice — cosa significano

### Caso 1A — diabete, Logistic Regression (top-30 per correlazione)

Le feature sono selezionate automaticamente per correlazione assoluta di Pearson con il target, escluso il sesso (`gender`) per non privilegiare nessun gruppo.

| Feature | Significato |
|---------|-------------|
| `hba1c_pct` | Emoglobina glicata (%) — il predittore dominante, quasi una misura diretta del diabete |
| `bp_chol_ever_high` | Colesterolo mai risultato alto — comorbidità tipica del diabete T2 |
| `bp_told_to_take_chol_meds` | Prescritta terapia ipolipemizzante |
| `gen_health_cond` | Salute generale auto-dichiarata su scala 1–5 |
| `cv_chest_pain_ever` | Dolore toracico mai avuto — segnale di rischio cardiovascolare |
| `cv_shortness_of_breath` | Dispnea da sforzo |
| `cb_money_spent_food_fast_food` | Spesa mensile in fast food ($) — variabile comportamentale/socio-economica |
| `cb_money_spent_food_restaurants` | Spesa mensile al ristorante ($) |
| `blood_donation_past_12mo` | Donazione sangue ultimi 12 mesi — proxy indiretta di salute percepita |
| `audio_hearing_aid_used` | Usa apparecchio acustico — complicanza neurosensoriale correlata al diabete |

### Caso 1B — artrite, XGBoost (top-30 per correlazione)

| Feature | Significato |
|---------|-------------|
| `age_years` | Età — predittore dominante, l'artrite è malattia dell'invecchiamento |
| `waist_circ_cm` | Circonferenza vita (cm) — proxy di infiammazione sistemica legata all'adiposità |
| `sagittal_abd_diameter_1/2/avg` | Diametro addominale sagittale (cm) — altra misura di adiposità viscerale |
| `grip_hand1_trial1_kg` (e varianti) | Forza della presa al dinamometro (kg) — ridotta nell'artrite reumatoide |
| `grip_difficulty_grasp_objects` | Difficoltà ad afferrare oggetti — conseguenza diretta dell'artrite alle mani |
| `grip_difficulty_reach_above_head` | Difficoltà ad alzare le braccia |
| `pf_difficulty_walking_quarter_mile` | Difficoltà a camminare 400 metri — funzionalità fisica |
| `sleep_told_sleep_disorder` | Disturbo del sonno diagnosticato — comorbidità del dolore cronico |
| `bp_ever_high_bp` | Ipertensione — cluster infiammatorio metabolico |
| `occ_work_past_week_status` | Stato lavorativo nell'ultima settimana — proxy funzionale |

**Asimmetria SHAP artrite**: `sleep_told_sleep_disorder` ha importanza 0.29 per le donne vs 0.18 per gli uomini. L'artrite reumatoide ha prevalenza F/M circa 3:1 — il modello cattura un'asimmetria biologica reale, non solo statistica.

### Caso 2 — Two Clinics, BMI come asse di perturbazione

Il BMI è **forzato dentro** le feature del Caso 2 (anche se non è nelle top-30 per correlazione col diabete) perché è l'asse di campionamento asimmetrico: F con BMI ≤28, M con BMI ≥28. Nel modello perturbato, il BMI sale al 5°–6° posto nell'importanza SHAP — prova che il modello ha usato BMI come proxy indiretto del sesso.

---

## Tre cose da sapere per rispondere alle domande

1. **Il dataset è rappresentativo per design**: NHANES usa un campionamento stratificato su etnia, età e reddito per rispecchiare la popolazione USA adulta. Non è un dataset ospedaliero — include persone sane e malate.

2. **Nessuna label predefinita**: la scelta del target è nostra. Diabete e artrite non sono "speciali" — sono state scelte perché binarie, prevalenza ragionevole, e ben documentate in letteratura per il bias di genere.

3. **Il sesso è escluso dalle feature del modello**: il codice esclude `gender` e qualsiasi variabile che contiene `sex_` nel nome. Il punto del workshop è che il bias emerge lo stesso — il modello lo ricostruisce indirettamente dalle feature cliniche correlate al sesso (BMI, emoglobina, grip strength, ecc.).
