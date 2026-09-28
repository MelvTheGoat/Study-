# Stack ("Reckon"): What This Proves I Know

Repo: https://github.com/MelvTheGoat/Stack

---

## 1. Payment reconciliation (domain)

**Simple explanation:** matching money received against money owed, and explaining any difference.

**In this project:** four channels, part/over/split payments, duplicates, strangers' payments, fees and T+1 settlement, a daily report.

**Also be ready to explain:** bank statement vs ledger reconciliation, settlement vs authorisation, chargebacks, suspense accounts, and why cash can only be attested.

---

## 2. Entity resolution and fuzzy name matching

**Simple explanation:** deciding whether two differently written names are the same person.

**In this project:** normalisation, noise-word removal, reordering, spelling variants, short forms, `difflib`, and guards against near-identical different people.

**Also be ready to explain:** Levenshtein / Jaro-Winkler, token-set similarity, blocking, false merges vs false splits, and phonetic encodings (Soundex, Metaphone).

---

## 3. Cost-sensitive decision thresholds

**Simple explanation:** choose the cut-off that minimises expected cost, not the one that maximises accuracy.

**In this project:** ₦60 vs ₦5,000 → 83:1 → threshold 0.85 via cross-validation.

**Also be ready to explain:** expected cost = p(FP)·C_FP + p(FN)·C_FN, why the optimal threshold is `C_FP/(C_FP+C_FN)` for calibrated probabilities (≈0.988 here in theory vs 0.85 chosen empirically), and precision-recall trade-offs.

---

## 4. Logistic regression and calibration

**Simple explanation:** a linear model that outputs a probability. Calibration makes that probability honest.

**In this project:** hand-written gradient descent with L2 and class balancing. Platt vs isotonic compared on Brier. ECE and reliability bands.

**Also be ready to explain:** log loss, Brier score, reliability diagrams, when isotonic overfits (small data), and class imbalance handling.

---

## 5. Webhooks and payment-provider integration

**Simple explanation:** the provider calls your URL when something happens. You must verify it, handle retries, and not trust order.

**In this project:** HMAC SHA-512 signatures, idempotency, 200-then-process, the verify-call authority, and a forward-only state machine.

**Also be ready to explain:** at-least-once delivery, idempotency keys, replay attacks, and event sourcing vs state.

---

## 6. Money handling in software

**Simple explanation:** never store money as floating point.

**In this project:** integer kobo, parsing that rejects sub-kobo, explicit rounding modes, basis-point percentages.

**Also be ready to explain:** banker's rounding, currency minor units, fee calculation, and allocation of remainders.

---

## 7. Human-in-the-loop ML

**Simple explanation:** the machine does the confident part, and people do the rest, in the best order.

**In this project:** a queue by money at risk, suggestions with reasons, approve/reject, and labels from rejections with no auto-retraining.

**Also be ready to explain:** active learning, feedback loops and drift, label bias (only rejected cases labelled), and precision@k for queues.

---

## 8. Evaluation with synthetic data

**Simple explanation:** when real data can't be shared, generate realistic data with a known answer key, and be clear about its limits.

**In this project:** a seeded corpus, explicit `SCENARIOS`, `make eval` regenerating every number, adversarial cases.

**Also be ready to explain:** simulation bias, the sim-to-real gap, and why tuning a parser on examples you wrote overstates accuracy.

---

## 9. Backend engineering (FastAPI, SQLAlchemy)

**In this project:** FastAPI routes, background tasks, session lifecycle, SQLAlchemy 2, Jinja pages, a JSON API, mypy strict, and an append-only audit.

**Also be ready to explain:** dependency injection and teardown order, transactions and isolation, Postgres vs SQLite, and audit-log design.

---

## 10. Using LLMs as a constrained last layer

**Simple explanation:** only ask an LLM what rules can't answer, force its answer into a schema, and accept "I don't know".

**In this project:** a schema generated from the Pydantic model, allowed to refuse, never re-asked, off by default.

**Also be ready to explain:** structured outputs / JSON schema, hallucination risk, cost control, and evaluating "extra reads gained" vs "payments invented".
