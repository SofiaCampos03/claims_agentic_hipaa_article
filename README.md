# Agentic Claims-Denial Framework

A Jupyter notebook that answers questions about healthcare claim denials with a team of LLM agents. The analysis the
agents rely on (model training, explanations, graph neural networks, recourse) is all computed in the notebook from
one CSV, and every report passes a HIPAA/GDPR review before it is shown.

```
user question
      │
      ▼
┌──────────────────────────────┐  search_policy_docs  ┌───────────────────────────┐
│ COORDINATOR (Groq gpt-oss)   │◄────────────────────►│ Hybrid Contextual RAG     │
│ plans · routes · synthesises │                      │ BM25 + dense (RRF) over   │
└──┬────────┬────────┬───────┬─┘                      │ LLM-contextualised chunks │
   ▼        ▼        ▼       ▼                        └───────────────────────────┘
 PREDICT  STATS +  COMBINA-  CAUSALITY
 (XGBoost EXPLAIN  TIONS     (counterfactual
  calib.) (SHAP)   (GNN)      recourse)
                     │
                     ▼
        HIPAA / GDPR post-review  →  released report
        (PHI redaction, payer names → PAYER-xxxxxx, LLM verdict)
```

| Agent | What it computes |
|---|---|
| **Prediction** | Optuna-tuned, Platt-calibrated XGBoost denial probability |
| **Statistics + explainability** | TreeSHAP drivers (log-odds) + population and payer statistics |
| **Combinations** | Code co-occurrence graph + message-passing GNN (code propensity), pair/triple denial lift, claim↔code GNN — every code shown with its label, plus an interactive d3 code-denial network (global and per claim) |
| **Causality** | Counterfactual recourse: the claim re-scored under actionable changes (pre-authorization, processing time) |
| **HIPAA / GDPR** | Runs after the coordinator: deterministic PHI redaction and payer pseudonymisation, plus an LLM review grounded in the HIPAA summary |
| **Coordinator** | Routes to the specialists, adds policy evidence from the RAG, writes the final report |

All models share **one stratified train/test split**. The GNN statistics use training claims only, and demo claims
come from the test set, so they are out-of-sample for every model.

## Repository layout

```
claims_agentic_framework.ipynb   the notebook (run top to bottom)
requirements.txt
.env.example                     template for .env (Groq API key + optional overrides)
config/feature_spec_xgboost.yaml feature definitions for the XGBoost model
config/code_labels.csv           code → description dictionary (ICD-10 diagnoses, procedures, medications)
data/README.md                   data dictionary — put claim_metrics.csv here
docs/README.md                   policy documents for the RAG — put them here
```

Generated at run time and git-ignored: `artifacts/` (model, cached analytics and the interactive graphs in
`artifacts/graphs/*.html`) and `.rag_cache/` (chunk contexts and embeddings).

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env           # then set GROQ_API_KEY in .env (https://console.groq.com)
jupyter lab                    # or open the notebook in VS Code
```

`.env` holds your secrets and is git-ignored, so it is never committed. The notebook loads it at the start of
Section 1, and its values take precedence over variables set in the shell.

1. Put the claims extract at `data/claim_metrics.csv`. Its columns are described in [`data/README.md`](data/README.md).
2. Put the policy documents (PDF or Markdown) in `docs/`. See [`docs/README.md`](docs/README.md).
3. Run the notebook top to bottom.

**Runtime** (about 100k claims, Apple Silicon): the first run takes about 6 minutes. That covers 30 Optuna trials
(~2 min), the claim GNN (~1.5 min), and contextualising and embedding the documents once. Results are cached in
`artifacts/` and `.rag_cache/`, so later runs take about 1.5 minutes. The model retrains automatically when the CSV,
the sample size, the split or the feature spec change.

## Configuration

Every setting can be set in `.env` (all of them are listed in [`.env.example`](.env.example)), as an environment
variable, or edited in the settings cells.

| Variable | Default | Meaning |
|---|---|---|
| `GROQ_API_KEY` | — | Groq API key (required) |
| `GROQ_MODEL` | `openai/gpt-oss-120b` | LLM used by every agent |
| `TRAINING_CSV` | `data/claim_metrics.csv` | Claims extract |
| `DOCS_DIR` | `docs/` | RAG documents |
| `HIPAA_DOCS` | `summary_HIPAA.md` | Files indexed for the compliance agent (the rest go to the policy index) |
| `EMBED_MODEL` | `BAAI/bge-small-en-v1.5` | fastembed model for dense retrieval |
| `TRAIN_SAMPLE` | `0` (all) | Use a random subset of claims |
| `TUNE_N_TRIALS` / `TUNE_SCORING` / `CV_FOLDS` | `30` / `f1` / `5` | Optuna tuning |
| `RUN_TRAINING` | `0` | `1` forces a retrain |
| `FORCE_RECOMPUTE` | `0` | `1` rebuilds the cached analytics |
| `GNN_*` | see notebook | GNN size, epochs, device (`auto` → CUDA / MPS / CPU) |
| `TORCH_NUM_THREADS` | `1` | Keep at 1 on macOS: XGBoost and PyTorch ship different OpenMP runtimes and can deadlock |
| `PSEUDONYMISE_CLAIM_IDS` | `1` | Replace claim ids with `CLAIM-xxxxxx` in reports (`0` keeps them) |
| `PSEUDONYM_SALT` | random per run | Secret salt for the pseudonyms. Set it to keep them stable between runs |

## Privacy and limitations

- **Data minimisation:** the tools give the LLM only the fields the analysis needs, and dates are reduced to the year.
  Every report is redacted (SSN, phone, e-mail, full dates, record numbers, …), and payer names are replaced with
  stable pseudonyms.
- **Cloud LLM:** Groq is a cloud API. The minimised claim features are sent to it. Production use with real
  PHI/personal data needs a BAA (HIPAA) and a GDPR Art. 28 processor agreement with the provider.
- **GDPR checks:** no GDPR text is indexed, so the GDPR review applies a checklist (Art. 5, 9, 4(5), 22) written in the
  agent's prompt. HIPAA findings cite the retrieved HIPAA passages.
- **Recourse is not causation:** it is a model what-if, the calibrated XGBoost re-scored under the change.
- **The LLM can still be wrong:** the coordinator is instructed to use only tool numbers and never to suggest
  unbundling or other improper billing, but reports should be reviewed by a person. The model output is decision
  support, not an automated denial decision.
