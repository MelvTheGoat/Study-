# Stack ("Reckon"): Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Stack

---

## Summary

A payment-to-invoice reconciliation service (FastAPI, SQLAlchemy, mypy strict, ~7,900 lines including tests) with Paystack ingestion (HMAC SHA-512, idempotent webhooks, verify-call authority, forward-only status), integer-kobo money, a four-rule deterministic layer that refuses ties, an 18-feature hand-written logistic regression with Platt calibration and a cost-derived threshold (0.85), a money-ranked review queue, an append-only audit log, a daily settlement report and prose-report intake (patterns, then an optional schema-bound LLM). `make eval` regenerates every README number from a seeded synthetic corpus, and I reproduced them exactly. **494 tests pass with the `.[llm]` extra, and 8 fail without it.**

## Architecture

```
recon/
  money.py                 Money(kobo:int), parsing, rounding modes
  fees.py                  Paystack NG fees, T+1 settlement
  paystack/signature.py    HMAC SHA-512
  paystack/events.py       6 events -> normalised
  paystack/client.py       verify call
  ingest.py, app.py        webhook: verify sig, store raw, 200, background task
  state.py, enums.py       status ranks, layers
  audit.py                 append-only log
  corpus/                  seeded generator + answer key + names/narrations
  match/deterministic.py   duplicate guard + 3 exact rules
  match/similarity.py      name normalisation, variants, short forms, difflib
  match/candidates.py      plausible invoices per payment
  match/features.py        18 features
  match/model.py           logistic regression (stdlib)
  match/calibration.py     Platt vs isotonic
  match/threshold.py       cost matrix -> threshold
  match/training.py        fit + CV threshold
  match/pipeline.py        layers in order
  intake/                  prose -> typed record (patterns, optional LLM)
  review/                  queue, approve/reject, labels, pages
  report/settlement.py     daily report
  evaluation/              harness, truth, scoring, extraction
```

## Key decisions (from `docs/DECISIONS.md` and the code)

| # | Decision | Why | Cost |
|---|---|---|---|
| 1 | Integer kobo | Floats drift, Decimal allows sub-kobo | Boundary conversions |
| 2 | Verify call is the authority | A webhook body is a stale, attackable snapshot | Extra call, new failure mode |
| 3 | Separate receive and process sessions | Starlette runs background tasks before dependency teardown | Two connections per webhook |
| 4 | Forward-only status | Out-of-order events (refund before charge) | Can't express reversal after refund |
| 5 | Generated corpus + answer key | Real data unpublishable, and every number reproducible | Assumption-dependent |
| 6 | Threshold by cross-validation | Too few residual cases for a held-out slice | Still only 87 payments |
| 7 | Calibration picks the smoother method on a near tie | Stability | — |
| 8 | Model closes only when nearly certain | 83:1 cost ratio | Low model coverage (6%) |
| 9 | Unreadable reports go to a person, once | Re-asking buys confidence, not information | — |
| 10 | Extraction scored on a set partly written afterwards | Avoids tuning to the test | Small set (40) |
| 11 | Queue ordered by money | Protects the books | Old small items starve |
| 12 | Rejections stored as labels | Negative examples are informative | Approvals mostly not stored |

## Algorithms

- **Name similarity:** normalise (case, punctuation), drop narration noise words (`nip`, `trf`, `payment`...), map spelling variants and short forms, compare token sets order-free, use `difflib.SequenceMatcher` for near spellings, and guard against near-identical *different* customers.
- **Logistic regression:** `sigmoid(w·x + b)` fitted by plain gradient descent in pure Python (300 epochs, learning rate 0.3, L2 0.001, class balancing), with an overflow-safe sigmoid. Features scaled to roughly 0–1 so weights are comparable.
- **Calibration:** Platt (a logistic on the score) vs isotonic, compared on development data by Brier score: raw 0.0934, Platt 0.0516, isotonic 0.0526. Platt picked. On 390 held-out pairs: Brier 0.0219, ECE 0.024. **But 347 of the 390 pairs sit in the 0–0.1 band.** The bands at or above the 0.85 line hold only 8 pairs (3 in 0.8–0.9, 5 in 0.9–1.0), so calibration where it matters most rests on very few cases.
- **Threshold:** minimise expected cost `FP × ₦5,000 + reviews × ₦60` over cross-validated folds of the residual. The result is 0.85.
- **Twin invoices** (two identical fits): the deterministic layer declines them. The scoring layer closes them, and the answer key accepts either invoice.

## How it's evaluated (reproduced by me)

| Metric | Value |
|---|---|
| Auto-clear rate | 76.1% (rules 69.9%, model 6.2%) |
| Precision | 100% (0 wrong of 334) |
| Recall | 80.1% |
| Review queue | 105 cases, ₦9.77M. The first 40 cover 91% (precision@40 = 0.875). |
| Adversarial cases closed wrongly | 0 (duplicates, twins, no-matching-order) |
| Calibration | Brier 0.0219, ECE 0.024 |
| Prose extraction (patterns) | 94.4% fully right on readable, 100% correct refusals, 0 invented (40 reports) |
| LLM extraction | Not measured yet (needs a key) |

**Tests:** 494 pass when the optional LLM extra is installed. With only `.[dev]` (what CI installs), 8 tests in `test_intake.py` for the LLM reader fail with `ModuleNotFoundError`. That contradicts the `pyproject.toml` comment saying the suite passes with no model provider.

## Known weaknesses

- Synthetic data, so the percentages are conditional on the scenario assumptions.
- 100% precision is a corpus artefact.
- Threshold tuned on 87 residual payments (4-fold CV). At 0.85, 21 were auto-cleared and 66 reviewed.
- Calibration at the high end rests on only 8 held-out pairs above 0.8.
- The LLM-reader tests are coupled to an optional dependency (see above). CI likely fails on them.
- Single provider. Settlement batches come from the corpus, not the Paystack API.
- No age escalation. No automated retraining (by design).
- One process. SQLite by default.
- Container not yet built or deployed (per the README).
