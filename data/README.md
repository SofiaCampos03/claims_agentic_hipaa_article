# `claim_metrics.csv` — data dictionary

Place the file at `data/claim_metrics.csv` (or point `TRAINING_CSV` to it). The notebook loads it with
`pandas.read_csv`, keeps claims whose `claim_status` is final (`paid`, `partially_paid`, `rejected`, plus `accepted` /
`approved` if present) and `claim_total > 10`, and builds everything else from it.

- **Grain:** one row per adjudicated claim, about 100,000 rows and 81 columns.
- **Target:** `claim_status ∈ {rejected, partially_paid}` → *denied* (1); `paid` → 0. This is **not** the `is_denied` boolean.
- **Formats:**
  - Timestamps are ISO strings with a UTC offset (`YYYY-MM-DD HH:MM:SS.fff +HHMM`).
  - Durations are PostgreSQL interval strings (`20 days 11:24:00`).
  - Code lists are PostgreSQL array literals (`{E11.9,I10}`). An empty or missing list means no codes of that type.

**Used** marks how the notebook uses each column:
- **F** — model feature, through `config/feature_spec_xgboost.yaml`.
- **T** — target.
- **G** — code graph / GNN.
- **R** — recourse what-if.
- **S** — statistics shown to the agents.
- **K** — key.
- **—** — not used.

> ⚠️ **Leakage:** some columns are only known *after* adjudication (remittance, rejected lines, processing time,
> `is_denied`). They must never be used as features. The feature spec lists them under `numerical.inactive` or
> leaves them out.

## Identity and outcome

| Column | Type | Description | Used |
|---|---|---|---|
| `claim_id` | int | Unique claim identifier | K |
| `business_identifier` | text | External / business claim reference | — |
| `claim_status` | text | Final status: `paid`, `partially_paid`, `rejected` | T |
| `claim_status_description` | text | Human-readable status | — |
| `is_denied` | bool | Payer's denial flag (narrower than the target; ⚠️ outcome) | — |

## Payer

| Column | Type | Description | Used |
|---|---|---|---|
| `payer_id` | int | Payer identifier | — |
| `payer_part_of` | int | Parent payer / group identifier | — |
| `insurer_name` | text | Payer name (11 payers incl. `Self-pay`). Pseudonymised as `PAYER-xxxxxx` in every report | F, S |
| `insurer_avg_claim_total` | float | Payer's average claim amount | F |
| `insurer_claim_volume` | int | Payer's claim volume | — |
| `insurer_rejection_rate`, `insurer_rejection_rate_{30,60,90,120}d` | float [0,1] | Payer denial rate, overall and over trailing windows | — (inactive: dominance / leakage risk) |
| `insurer_rejection_rate_global` | float [0,1] | Payer denial rate over all history | — |
| `insurer_complaint_rejection_rate_{30,60,90,120}d` | float [0,1] | Payer denial rate for the claim's chief complaint | — |
| `insurer_gender_complaint_rejection_rate_{global,30d,60d,90d,120d}` | float [0,1] | … by complaint and gender | — |
| `insurer_age_complaint_rejection_rate_{global,30d,60d,90d,120d}` | float [0,1] | … by complaint and age range | — |

## Patient

| Column | Type | Description | Used |
|---|---|---|---|
| `age_range` | text | Age band (`0-5` … `65+`, 8 bands) | F |
| `gender` | text | `female` / `male` | F |
| `patient_claim_count` | int | Number of claims of the patient | F |
| `patient_avg_claim_total` | float | Patient's average claim amount | F |
| `patient_rejection_ratio`, `patient_rejection_ratio_{30,60,90}d` | float [0,1] | Patient's historical denial ratio | — (inactive) |
| `patient_rejection_ratio_120d` | float [0,1] | Patient's denial ratio over the last 120 days | F (binned + `≥ 0.5` flag) |
| `days_since_last_claim` | interval | Time since the patient's previous claim | — |

## Claim

| Column | Type | Description | Used |
|---|---|---|---|
| `claim_type` | text | `institutional`, `professional`, `oral`, `vision`, `pharmacy` | F, R (structural what-if, never recommended) |
| `claim_subtype` | text | `inpatient` / `outpatient` | F, S |
| `claim_total` | float | Billed amount | F, S |
| `claim_total_per_diagnosis` | float | `claim_total / diagnosis count` | F (binned) |
| `claim_total_per_vital`, `claim_total_per_medication` | float | Amount ratios | — |
| `claim_remittance` | float | Amount paid ⚠️ outcome | — |
| `claim_remittance_ratio` | float [0,1] | Paid / billed ⚠️ outcome | — |
| `number_of_rejected_lines` | int | Rejected claim lines ⚠️ outcome | — |
| `claim_month` | int 1–12 | Month of the claim | F |
| `claim_day_of_week` | int 0–6 | Day of week (0 = Sunday) | F |
| `vetted_claims` | int | Number of vetted claims linked to the record | — |
| `number_of_similar_{rejected,approved,partially_paid,paid}` | int | Similar-claim outcome counts (all zero in this extract) | — |

## Dates and process

| Column | Type | Description | Used |
|---|---|---|---|
| `draft_date`, `created_date`, `submitted_date` | timestamp | Claim lifecycle dates | — |
| `status_date` | timestamp | Date of the final status | S (year only) |
| `encounter_start_date`, `encounter_end_date` | timestamp | Clinical encounter dates | — |
| `encounter_account` | int | Encounter account identifier | — |
| `bureaucracy_time` | interval | Administrative time before adjudication | F (in days), R |
| `processing_time` | interval | Adjudication time ⚠️ post-decision | — |

## Pre-authorization

| Column | Type | Description | Used |
|---|---|---|---|
| `has_pre_auth` | bool | A pre-authorization was requested | F, R |
| `was_pre_auth_approved` | bool | The pre-authorization was approved | F, R |
| `pre_auth_status` | text | `approved` / `rejected` (only when requested, ~11% of rows) | — |
| `pre_auth_time` | interval | Time to the pre-authorization decision | — |

## Clinical codes

| Column | Type | Description | Used |
|---|---|---|---|
| `diagnosis_codes` | array | ICD-10 diagnosis codes, e.g. `{E11.9,I10}` | F (multi-hot), G |
| `procedure_codes` | array | Procedure / service codes (~2% empty) | F (multi-hot), G |
| `medication_codes` | array | Medication codes (~42% empty) | F (multi-hot), G |
| `number_of_procedures` | int | Procedure count | G, S |
| `number_of_medication` | int | Medication count | G, S |
| `unmatched_medication_diagnosis_count` | int | Medications without a matching diagnosis | — |
| `chief_complaint`, `chief_complaint_code` | text | Chief complaint (20 values) | — |
| `admission_diagnosis` | text | Admission diagnosis | — |
| `vitals_count` | int | Number of vitals recorded | — |
| `diagnosis_vitals_ratio`, `medication_diagnosis_ratio` | float | Count ratios | — |

Code descriptions (e.g. `rx:FI0077` → *Chemotherapy drug*) come from `config/code_labels.csv`, not from the CSV.

Derived inside the notebook (not in the CSV):
- `target`
- `bureaucracy_time_days`
- `diagnosis_count` (from `diagnosis_codes`)
- `no_pre_auth_flag`, `pre_auth_denied_flag`, `patient_high_denial_120d`
- the prefixed code list `dx:… px:… rx:…`
