# Credit-Risk-Decisioning: System Design

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

## The problem, in 3 lines

A lender doesn't just need "will this person default?". It needs a calibrated probability, an approve/decline decision that accounts for what a bad loan and a lost customer each cost, legally usable reasons for every decline, and proof that the decisions are fair.
This system does all four, on a synthetic loan book with known ground truth (so reject inference and fairness can actually be checked), and serves decisions through one engine used by both an API and batch scoring.
The headline result is a **cost**, not an AUC.

> The headline results come from a **synthetic** book (30,000 applications, 36 monthly vintages). The real-data path (UCI credit card default) is implemented but was never run on the actual file, because the download was blocked in the build environment.

## Diagram

```mermaid
flowchart TB
    subgraph DATA["Data"]
        SIM[simulate.py<br/>30k apps, 36 vintages, drift from v24,<br/>injected income bias for group B,<br/>legacy approval policy, truth for everyone]
        UCI[ingest.py<br/>UCI credit default<br/>checksum + schema, not run on real file]
    end

    subgraph TRAIN["Training programme (scripts/run_experiments.py)"]
        SPLIT[Out-of-time split<br/>fit v0-19, calibrate v20-23, test v24-35]
        MODELS[Model zoo<br/>WOE scorecard, L2 logistic, LightGBM]
        CAL[Calibration<br/>raw / Platt / isotonic,<br/>chosen by slope then ECE]
        POL[Cost policy<br/>p* = margin / (margin + LGD)]
        RI[Reject inference study<br/>fuzzy, parcelling, Heckman, oracle]
        FAIR[Fairness audit<br/>DPD, AIR, EOD, group ECE, trade-off curve]
        MON[Monitoring<br/>PSI, vintage curves, triggers]
        ART[Versioned artifact bundle<br/>model + calibrator share training_run_id + hashes]
    end

    subgraph SERVE["Serving"]
        ENG[DecisionEngine.decide<br/>the ONE decision path]
        API[FastAPI /decision,<br/>/health, /model-info]
        BATCH[score_batch.py]
        AUD[(Append-only audit log<br/>fsync'd JSON lines)]
        UI[Streamlit form<br/>port 8080, calls API on 8000]
    end

    SIM --> SPLIT
    UCI -.-> SPLIT
    SPLIT --> MODELS --> CAL --> POL --> ART
    MODELS --> RI
    CAL --> FAIR
    CAL --> MON
    ART --> ENG
    ENG --> API --> AUD
    ENG --> BATCH
    UI --> API
    CI[GitHub Actions<br/>lint, types, tests, calibration gate,<br/>docker smoke, push to AWS ECR] -.-> ART
```

## Each part, and why it's there

| Part | Code | What it does | Why it's there |
|---|---|---|---|
| Simulator | `src/simulate.py` | Generates a book with a known default process: interactions (utilisation × delinquency), a DTI cliff, a thin-file step, drift from vintage 24, group B's recorded income understated by 15%, a legacy approval policy, and a hidden loan-officer signal (the MNAR case). Ground truth is kept for declined applicants too. | Reject inference and fairness can't be proven on real data, because you never see what a declined person would have done. |
| Ingest | `src/ingest.py` | Idempotent UCI download, checksum, schema validation, pseudo-out-of-time design. | A real-data path. It was blocked from the build environment, so it's tested on a fixture only. |
| Scorecard | `src/scorecard.py` | WOE binning (monotonic) + logistic regression → a points scorecard. Points-shortfall reason codes. | The industry-standard, spreadsheet-auditable baseline. |
| Model zoo | `src/models.py` | LightGBM, L2 logistic, scorecard behind one interface. | Fair comparison. |
| Calibration | `src/calibration.py` | Raw / Platt / isotonic. Brier, ECE, MCE, slope, intercept. Picks the calibrator with slope within 0.15 of 1, then lowest ECE. | Expected loss = PD × LGD × EAD needs a real probability. |
| Evaluation | `src/evaluation.py` | ROC-AUC, Gini, PR-AUC, KS, precision at review capacity (1/5/10%), resampling comparison. | Accuracy is meaningless at a 17% base rate. |
| Policy | `src/policy.py` | Cost model (LGD 0.75, margin 9%, EAD = loan amount), closed-form and empirical cutoff, approval/loss frontier, sensitivity grid. | The threshold matters more than the model. |
| Reject inference | `src/reject_inference.py` | Fuzzy augmentation, parcelling, Heckman two-step, oracle, under MAR and MNAR. | To test whether correcting selection bias actually helps. |
| Fairness | `src/fairness.py` | Demographic parity difference, adverse impact ratio, equal opportunity difference, within-group calibration, λ trade-off curve, "model excess" vs true risk gap. | Protected attributes have proxies. You have to measure. |
| Explain | `src/explain.py` | SHAP contributions and adverse action reason codes. Protected-basis reasons (e.g. `age`) are flagged, not silently printed. | Legal notices must be right and reproducible. |
| Monitoring | `src/monitoring.py` | Score PSI, feature PSI, vintage default curves, a combined-watch trigger. | Showed that PSI alone would have missed the deterioration. |
| Artifacts | `src/artifacts.py` | Versioned bundle. The model and calibrator share a `training_run_id` and content hashes. **The service refuses to start on a mismatch.** | A mismatched pair keeps the AUC but silently breaks the probabilities. |
| Engine | `src/engine.py` | `DecisionEngine.decide`, the only scoring path. | API and batch can't drift. A test requires byte-identical results. |
| API | `src/api/main.py`, `schemas.py`, `audit.py` | FastAPI with input validation. Writes an append-only, fsync'd audit record before responding. | Every decision must be retrievable later. |
| Batch | `scripts/score_batch.py` | Scores a portfolio through the same engine. | Same logic, different entry point. |
| UI | `app.py`, `entrypoint.sh` | A Streamlit form on 8080 posting to the API on 8000. Shows the decision, PD, expected loss and reason codes. | A demo front end. Added 11 Aug. |
| CI/CD | `.github/workflows/ci.yml`, `Dockerfile` | Lint, mypy, tests, a calibration regression gate, a Docker smoke test, then build and push to **AWS ECR** on main/master. The Docker build trains the model inside the image (`--quick`). | The model and calibrator are baked in from one run. |

## Tech stack

| Tool | What it's used for | Why this one |
|---|---|---|
| Python 3.11 | Everything | Typed (mypy) |
| numpy, pandas | Data | Standard |
| scikit-learn | Logistic, calibration | Well known |
| LightGBM | Best model | Handles interactions and thresholds |
| SHAP | Model-agnostic reasons | Works for any model |
| FastAPI + Uvicorn | Decision API | Validation with Pydantic |
| Streamlit | Demo UI | Quick form |
| Docker (multi-stage) | Build + runtime | Trains in the builder stage. Non-root runtime. |
| GitHub Actions + AWS ECR | CI and image registry | Push on main |
| pytest, ruff, mypy | Quality | README says 223 tests. See 06 for my run. |

## Data flow, step by step

**Training (`python -m scripts.run_experiments`):**
1. Simulate 30,000 applications with truth for everyone.
2. Split out-of-time: fit on vintages 0–19 (12,095), calibrate on 20–23 (2,389), test on 24–35 (6,516), which contain the drift.
3. Fit the scorecard, logistic and LightGBM, on approved applicants only (as in real life).
4. Calibrate. On this book the raw LightGBM was selected.
5. Choose the cutoff: `p* = 0.09 / (0.09 + 0.75) ≈ 0.107`.
6. Run the reject inference, fairness, resampling and monitoring studies, and write `reports/tables` and figures.
7. Save the versioned artifact bundle.

**Serving (`POST /decision`):**
1. Validate the input (Pydantic ranges).
2. The engine loads the bundle (refusing on a mismatch), predicts PD, applies the calibrator, and computes EL = PD × LGD × EAD and expected value.
3. Decide approve/decline against the cutoff, and give the scorecard score and band.
4. Produce reason codes (points shortfall), flagging any protected basis.
5. Append the audit record (timestamp, input hash, score, band, decision, reasons, model version, cutoff), then return.

## Trade-offs and limits

**Key results (out-of-time, synthetic, from the repo's `reports/`):**
- LightGBM net cost **−2,506,563** (profit 384.68 per application) vs scorecard −2,168,367. The scorecard gives up 13.5% of profit.
- **The threshold matters more than the model:** F1-optimal cutoff costs 729,735 more than cost-optimal, and a naive 0.5 cutoff **loses money** (+1,227,921).
- AUC: LightGBM 0.8379, logistic 0.8321, scorecard 0.8148. Monotonicity costs 0.0004 AUC. Coarse binning costs 1.73 points.
- Resampling (class weights, SMOTE, undersampling) destroyed calibration (ECE up to 0.28) for no ranking gain.
- Reject inference **hurt** under MAR and **helped** under MNAR, and you can't tell which you're in from a real book.
- Fairness: group AIR 0.755 (below 4/5), EOD 0.136, and a 2.0-point "manufactured" PD excess for group B from the income bias. Full parity costs 1.64% of profit.
- Monitoring: PSI (0.103) **didn't fire** while the 12-month default rate rose 75%. Only the combined-watch rule fired.

**Limits:**
- **Synthetic data.** No number transfers to a real portfolio.
- **UCI path not run** on the real file.
- **LGD, EAD and margin are assumed**, and the cutoff moves from 0.053 to 0.211 across plausible values.
- **`age` is both a feature and a protected basis.**
- **The audit log is a local file**, not WORM storage.
- **No intersectional fairness analysis.**
- **The Streamlit UI's defaults are outside the training range:** income ₦5,000,000 (training 9,000–900,000), DTI 2.5 (training mean ~0.33, max 2.5), utilisation 1.2, loan ₦2,500,000 (training max 200,000). The labels also say ₦ although the synthetic book has no currency. The API accepts these values (its limits are wider), so the demo scores out-of-distribution applicants by default.
- **Two processes in one container** (uvicorn in the background plus Streamlit) with no supervisor. The Docker health check only checks Streamlit.

## What I'd change at 10x scale

- **Feature store + real bureau data** with point-in-time joins, and real LGD/EAD models.
- **Real-time decisioning service** separate from the UI, behind a gateway, with autoscaling. One process per container.
- **WORM audit storage** (e.g. S3 Object Lock) with retention and replication.
- **Scheduled recalibration** on recent matured outcomes (the runbook's advice), with champion/challenger.
- **A monitoring pipeline** with vintage dashboards and combined triggers, plus alerting.
- **Intersectional fairness audits**, and a legal review of `age`.
- **Input validation aligned to the training distribution**, with out-of-range warnings.
