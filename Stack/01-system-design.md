# Stack ("Reckon"): System Design

Repo: https://github.com/MelvTheGoat/Stack

## The problem, in 3 lines

A Nigerian business gets paid by card, dedicated virtual account, bank transfer and cash, and most payments don't say which invoice they're for.
Every evening someone matches payments to invoices by hand, and it goes wrong the same ways: misspelled names, part payments, overpayments, one transfer for three invoices, duplicates, money from strangers.
Reckon matches each payment to its invoice, closes only the ones it can prove or is nearly certain about, and puts the rest in a review queue ranked by money at stake, with a full audit trail.

## Diagram

```mermaid
flowchart TB
    PS[Paystack webhook<br/>6 event types] -->|HMAC SHA-512 check| WH[POST /webhooks/paystack<br/>store raw, answer 200]
    WH -->|background task,<br/>new session| VER[Verify call<br/>GET /transaction/verify]
    VER --> ST[Forward-only status<br/>pending→failed→success→<br/>reversed→refunded]
    PROSE[Payment reports in prose<br/>WhatsApp, voice notes] --> PAT[Pattern parser<br/>amounts, names, dates,<br/>channels, invoice refs]
    PAT -->|unreadable| LLM[Optional LLM reader<br/>schema-constrained,<br/>may refuse, asked once]
    PAT --> REC
    LLM --> REC
    ST --> REC[Plain payment record<br/>reference, amount (kobo),<br/>time, channel, text]

    subgraph MATCH["Matching: certain first, scored second, model last"]
        DUP[Duplicate guard]
        D1[Exact reference]
        D2[Dedicated account +<br/>one invoice of that amount]
        D3[One invoice of exact amount<br/>in the time window]
        PROB[Logistic model on 18 features<br/>Platt-calibrated<br/>close if p ≥ 0.85]
    end

    REC --> DUP --> D1 --> D2 --> D3 --> PROB
    PROB -->|p < 0.85 or ties| Q[Review queue<br/>ranked by money at risk]
    D1 & D2 & D3 & PROB -->|closed| AUD[(Append-only audit log<br/>layer, score, evidence, who)]
    Q -->|approve / reject| AUD
    Q -->|rejections| LAB[/api/labels<br/>for manual retraining/]
    AUD --> REP[Daily settlement report<br/>gross, fees, timing]
```

## Each part, and why it's there

| Part | Code | What it does | Why it's there |
|---|---|---|---|
| Money | `money.py` | Every amount is a whole number of kobo in a `Money` class. Raises on fractions of a kobo. | Floats drift, and `Decimal` allows impossible amounts. |
| Webhook endpoint | `app.py`, `ingest.py` | Checks the HMAC SHA-512 signature, stores the raw event, answers 200 at once, and processes in a background task with a fresh DB session. Idempotent. | Paystack retries slow endpoints. Starlette runs background tasks before a shared session commits (measured), so a separate session is needed. |
| Verify call | `paystack/client.py` | Asks Paystack `GET /transaction/verify/{ref}` before a payment counts. Paystack's answer wins. "No such reference" ≠ "couldn't ask". | A webhook body is a snapshot, and the part an attacker can aim at. |
| Event folding | `paystack/events.py`, `state.py` | Six events normalised. Status only moves forward by rank. | Webhooks arrive out of order (a refund can land before its charge). |
| Fees | `fees.py` | Paystack Nigerian fee schedule and T+1 settlement. | The bank credit is a day late and smaller than sales. |
| Corpus | `corpus/generate.py` | Generates a month of payments **and the answer key**, with scenarios written out in `SCENARIOS`. | Real data isn't publishable. Every number can be regenerated. |
| Deterministic layer | `match/deterministic.py` | Duplicate guard, then exact reference (checked against the real ledger), dedicated account with exactly one fitting invoice, and exact amount in the time window. **Never fires on a tie.** | Certainty first. A tie isn't certainty. |
| Name similarity | `match/similarity.py` | Normalises, removes narration noise words, handles reordering, spelling variants and short forms, and uses `difflib`. Must *not* merge two near-identical customers. | Bank narrations like `NIP/GTB/OKONKWO ADA/PAYMENT`. |
| Features | `match/features.py` | 18 named features per (payment, candidate invoice), e.g. name similarity, surname exact, landed in their account, amount exact/closeness, under/overpaid, recency, channel flags, only open invoice, seen this payer before, round number. | Each is something a bookkeeper would check, so the queue can explain it in words. |
| Model | `match/model.py`, `training.py` | Logistic regression written out by hand (no scikit-learn). | 18 features and a few thousand rows. Readable and reproducible. |
| Calibration | `match/calibration.py` | Platt scaling vs isotonic, compared. Platt chosen. | The threshold only works if "90%" means 90%. |
| Threshold | `match/threshold.py` | From costs: a review costs ₦60 (3 minutes at ₦1,200/hour), a wrong auto-clear ₦5,000 (2 hours + ₦2,600 goodwill), about 83:1. Cross-validated on the residual. Result: 0.85. | A business decision, written down in naira. |
| Pipeline | `match/pipeline.py` | Runs the layers in order and records which layer decided. | "What happened today" is readable, not assumed. |
| Intake | `intake/parse.py`, `amounts.py`, `schema.py` | Reads prose payment reports into a typed record with patterns, or refuses. | Informal channels are common. |
| LLM reader | `intake/` (optional extra `.[llm]`) | A hosted LLM on what the patterns can't read, constrained to a JSON schema generated from the same Pydantic model. May answer "can't read". Never asked twice. Off by default. | Last layer only. A tool that silently calls an API is a surprise bill. |
| Review | `review/queue.py`, `web.py` | A queue ordered by money at risk, with suggestions and reasons. Approve/reject. Rejections become labels. | A person's evening is limited. The biggest money goes first. |
| Report | `report/settlement.py` | Daily report: gross, fees and timing shown separately. | No single unexplained difference. |
| Audit | `audit.py` | Append-only log: input, layer, score, evidence, when, who. | Every decision can be traced. |
| Evaluation | `evaluation/*` | `make eval` regenerates every number in the README. | Numbers can't be typed by hand. |
| Deploy | `Dockerfile`, `cloudbuild.yaml`, `scripts/deploy.sh`, `smoke.py` | Cloud Run. The deploy refuses to ship an image that fails the smoke test. | Don't ship what you haven't opened. |

## Tech stack

| Tool | What it's used for | Why this one |
|---|---|---|
| Python 3.11 | Everything | Typed (mypy strict) |
| FastAPI + Uvicorn | Webhook, review pages, JSON API | Async, background tasks |
| SQLAlchemy 2 | DB (SQLite by default, Postgres via env) | Swap stores without code changes |
| Pydantic / pydantic-settings | Schemas and config | Also generates the LLM's JSON schema |
| httpx | Paystack API | Async and mockable |
| Jinja2 | Review and report pages | Simple server-rendered HTML |
| Standard library (`difflib`, `math`) | Name similarity, logistic regression | No compiler, fully readable |
| Optional LLM SDK | Last extraction layer | Off by default |
| pytest, ruff, mypy, pre-commit | Quality | 494 tests pass with the LLM extra installed |
| Docker + Cloud Run | Deploy | Container with a smoke-tested build |

## Data flow, step by step

1. Paystack sends a webhook. Check the signature, store the raw body, reply 200.
2. A background task opens a new session and calls **verify**. Paystack's amount, fees and status win. Status only moves forward.
3. The payment becomes a plain record: reference, amount in kobo, time, channel, text.
4. **Duplicate guard:** same money, same payer, same day, and the invoice is already settled → a person decides.
5. **Exact reference** in structured data or narration, checked against the ledger → closed.
6. **Dedicated account** with exactly one invoice of that amount → closed.
7. **Exactly one open invoice** for that exact amount in the window → closed.
8. **Otherwise score** every plausible invoice with the 18-feature logistic model, calibrate, and close if p ≥ 0.85 with a clear winner.
9. **Everything else** goes to the review queue, sorted by money at risk, with a suggested match and reasons in words.
10. Every closure or human decision is written to the append-only audit log. The daily report shows gross, fees and timing.

## Trade-offs and limits

**Measured on the generated corpus (I re-ran `make eval` and got the same numbers as the README):**

| | Value |
|---|---|
| Payments | 439, worth ₦37,181,750.67 |
| Closed without a person | 76.1% (rules 69.9%, model 6.2%) |
| Precision of auto-closes | 100% (0 wrong of 334) |
| Recall | 80.1% |
| Sent to review | 105 (23.9%), ₦9,774,840.25 |
| First 40 reviews cover | 91% of the money at risk |
| Calibration | Brier 0.0219, ECE 0.024 (Platt) |
| Saving vs doing all by hand | 16.7 hours, ₦20,040 per month (pessimistic) |

**Honest limits (most from the repo's own list):**
- **The data is simulated.** Every percentage depends on the scenario guesses in `SCENARIOS`.
- **Threshold chosen on 87 residual payments.** Too few to trust two decimal places.
- **100% precision won't survive real data.**
- **Cash can't be verified.** It's "attested", not "verified".
- **Nothing retrains itself.** Rejections become labels for a person to use.
- **No age escalation in the queue.** Small old payments wait forever.
- **Only Paystack is wired up.** The "provider-neutral" seam hasn't been tested by a second provider.
- **LLM reader not scored yet.** It needs an API key, and the README numbers exclude it.
- **Tests depend on the optional extra.** With only the dev install (what CI installs), 8 LLM-reader tests fail with a missing module. With `.[llm]`, all 494 pass. `pyproject.toml` says the suite passes with no model provider, but that's no longer true.
- **Not deployed yet.** The container image hadn't been built when the README was written.
- **One process, in memory.** Postgres via `RECON_DATABASE_URL` for anything bigger.

## What I'd change at 10x scale

For 10x payments or many businesses:
- **Postgres + a job queue** (e.g. Celery or Cloud Tasks) instead of in-process background tasks.
- **Multi-tenant data** with per-business ledgers and thresholds.
- **Candidate indexing** (by amount bucket and customer) so scoring doesn't compare against every open invoice.
- **Pull settlement batches from Paystack** on a schedule (not built yet).
- **A second provider adapter** (e.g. Flutterwave or Moniepoint) to prove the provider-neutral seam.
- **Monitored, human-approved retraining** from review-queue labels, with drift alerts.
- **Age escalation** in the queue alongside money at risk.
- **Real data** to re-fit the model and re-draw the threshold with hundreds of residual cases.
