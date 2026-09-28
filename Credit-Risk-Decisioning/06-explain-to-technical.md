# Credit-Risk-Decisioning: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

---

## Summary

A consumer-credit decisioning system: a synthetic loan book with known ground truth, out-of-time evaluation, a model zoo (WOE scorecard, L2 logistic, LightGBM), calibration selection, a cost-optimal policy, reject inference under MAR and MNAR, a fairness audit with the true-vs-model gap split, PSI and vintage monitoring, versioned artifact bundles with integrity checks, one decision engine for API and batch, an append-only audit log, a Streamlit demo, and CI that builds and pushes to AWS ECR. The core was built 4–6 Aug 2026, the UI and ports on 11 Aug, and the ECR deploy on 28 Aug.

## Architecture

```
src/simulate.py          DGP: interactions, DTI cliff, thin-file step, drift v24+,
                         group-B income bias (x0.85 recorded), legacy policy,
                         latent officer signal (MNAR), branch_capacity_index (exclusion)
src/ingest.py            UCI: download, checksum (TOFU), schema, pseudo-OOT
src/scorecard.py         monotonic WOE binning -> logistic -> points
src/models.py            LightGBM | L2 logistic | scorecard, one interface
src/calibration.py       raw | Platt | isotonic; Brier, ECE, MCE, slope, intercept
src/evaluation.py        ROC/PR-AUC, KS, Gini, precision@capacity, resampling study
src/policy.py            cost model, sweep, closed form, frontier, sensitivity grid
src/reject_inference.py  fuzzy augmentation, parcelling, Heckman 2-step, oracle
src/fairness.py          DPD, AIR, EOD, per-group ECE, lambda curve, model excess
src/explain.py           SHAP + points-shortfall reason codes, protected flags
src/monitoring.py        PSI, vintage curves, triggers (incl. combined watch)
src/artifacts.py         bundle: manifest, hashes, shared training_run_id
src/engine.py            DecisionEngine.decide (the only path)
src/api/                 FastAPI (/decision, /health, /model-info), schemas, audit
app.py + entrypoint.sh   Streamlit :8080 -> API :8000 (both in one container)
```

## Key decisions

| Decision | Why |
|---|---|
| Synthetic book with truth for rejects | Only way to validate reject inference and fairness |
| Out-of-time split by vintage, drift in test | Tests robustness, not interpolation |
| Headline = net cost | The business objective |
| `p* = margin/(margin+LGD)` | Cost-optimal for calibrated PD. Model-independent. |
| Calibrator selection: slope within 0.15 of 1, then ECE | Avoids isotonic's overfit (in-period ECE 0, OOT slope 0.726) |
| No resampling | Destroys calibration (ECE 0.026 → 0.19–0.28) |
| Points-shortfall reasons deployed | Reproducible from the points table forever. SHAP needs a versioned background set. |
| Flag protected-basis reasons | `age` reasons need legal review |
| λ=0 recommended | Group-specific cutoffs are likely unlawful and don't fix the probabilities |
| Refuse to start on an artifact mismatch | Silent miscalibration is invisible to AUC |
| One engine for API and batch | A byte-identical test |
| Audit before the response | Every decision is traceable |

## Results (out-of-time, synthetic; from `reports/`)

| Model | AUC | PR-AUC | KS | Brier | ECE | Net cost |
|---|---|---|---|---|---|---|
| LightGBM | 0.8379 | 0.5736 | 0.5238 | 0.1042 | 0.0255 | −2,506,563 |
| Logistic L2 | 0.8321 | 0.5646 | 0.5039 | 0.1059 | 0.0289 | −2,465,079 |
| Scorecard | 0.8148 | 0.5156 | 0.4847 | 0.1112 | 0.0303 | −2,168,367 |

- Cutoffs: empirical 0.120 (−2,584,041), closed form 0.107, F1 0.211 (+729,735 cost), 0.5 (+1,227,921, a loss).
- Precision@1/5/10%: 0.908 / 0.764 / 0.656.
- Reject inference (bias on rejects): MAR baseline −0.051 → fuzzy +0.102 (worse). MNAR baseline −0.154 → fuzzy −0.007. Heckman best 11%.
- Fairness: group AIR 0.755, EOD 0.136, model excess +0.0202. Age-band AIR 0.451. Parity costs 41,028 (1.64%).
- Monitoring: score PSI 0.103, worst feature PSI 0.212, vintage default +75%. Only the combined watch fires.

## How it's tested

- The README says **223 tests (~40 s)**, including a **calibration regression gate** (`tests/test_calibration_gate.py` against `tests/calibration_baseline.json`) and the API/batch byte-identity test.
- CI: lint, mypy, tests, the calibration gate, a Docker build and smoke test (`/health`, `/model-info`), then an ECR push on main/master.
- **My own run:** with the latest dependencies, **4 tests fail** (all Heckman / reject-inference tests). `src/reject_inference.py` calls `selection_model.predict(..., linear=True)`, and statsmodels 0.15 removed that argument. `pyproject.toml` only says `statsmodels>=0.14`, so a fresh install picks up 0.15. With `statsmodels<0.15`, **all 227 tests pass** (~46 s). The README says 223, so tests were added since. The fix is `which="linear"` or an upper bound on statsmodels.

## Known weaknesses

- **Unpinned dependency breaks Heckman on fresh installs** (statsmodels 0.15, see above).
- **Synthetic numbers don't transfer.** The UCI path hasn't been run on the real file (download blocked). Checksum is trust-on-first-use.
- **Assumed LGD, EAD and margin.** The cutoff swings 0.053–0.211.
- **The exclusion restriction is gifted** to Heckman (an upper bound).
- **All accounts are seasoned.** There's no immature tail.
- **`age` is used and is protected.**
- **Local-file audit log.** No WORM storage.
- **No intersectional fairness.**
- **Demo UI issues:** defaults outside the training range (income ₦5M vs 9k–900k, loan ₦2.5M vs max 200k, DTI 2.5 at the training max, utilisation 1.2), labelled in ₦ though the book has no currency, the loan field has no max while the API caps it at 5M (the API returns 422), and two unsupervised processes in one container with a health check only on Streamlit.
- **Deploy secrets** (AWS keys) are needed in repo secrets. The ECR job does its own separate `docker build`, which retrains the model rather than reusing the image that passed the smoke test.
