# Reckon, Payment Matching: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. Picture a small Nigerian business. Customers pay by card, by bank transfer, into a special account, or in cash. And most payments don't say which invoice they're for.

So every evening, someone sits down and matches payments to invoices by hand. And it goes wrong in the same ways every time. Names misspelled, part payments, overpayments, one transfer for three invoices, duplicates, money from strangers.

So here's our project, called Reckon. It matches each payment to the right invoice, but **only when it's sure**. Anything unclear goes to a person, in a list sorted by money at risk, biggest first. And every decision is written in a permanent record.

The project has a house rule, and you'll hear it again and again: **certain first, scored second, AI last.** Exact rules go first. A careful scoring model goes second. An AI reader is the very last resort.

## So what do we need?

1. **Exact money**, with no rounding errors ever.
2. **Settings**, with a safety check on the payment key.
3. **A database.**
4. **Safe intake of payment alerts** from Paystack, a Nigerian payments company.
5. **Realistic data with an answer key**, because real business data can't be published.
6. **Certain rules** that never guess.
7. **Name matching** that copes with Nigerian bank notes.
8. **A scoring model**, with honest chances and a cut-off line based on cost.
9. **A reader for payment reports** written in everyday words.
10. **A review list** for people, and a **daily report**.
11. **A permanent record** of every decision.
12. **Proof**: every number in the README must be rebuildable.

## Step zero: set up the workshop

First, a `README.md`, the front page. A `.gitignore`. And `pyproject.toml`, which describes the project and what it needs. There's a main install, and two optional extras: one for development tools, and one for the optional AI reader.

`.env.example` is a template for settings, like the Paystack test key. You copy it to `.env`, and the private copy is never saved.

Then the tools for keeping quality high. `Makefile` has short commands for common jobs: install, lint, format, check types, test, build the data, train, evaluate, serve, deploy and clean. `scripts/check.sh` runs every check exactly the way the automatic checks do, so "green here" means "green there". `.pre-commit-config.yaml` runs tidying checks every time you save a change.

And `scripts/hooks/commit-msg` is a small one. It strips extra footer lines out of commit messages, like co-author tags and tool links, to keep the history clean.

The code lives in `src/recon/`. Its `__init__.py` marks it as a package.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: the project's details, needs and optional extras.
> - `.env.example`: a template for settings and keys.
> - `Makefile`: short commands for common jobs.
> - `scripts/check.sh`: runs every check, the same way the automatic checks do.
> - `.pre-commit-config.yaml`: checks that run on every saved change.
> - `scripts/hooks/commit-msg`: keeps commit messages clean.
> - `src/recon/__init__.py`: marks the code folder as a package.

## Step one: money, done right

Before anything else, money. Because if money is wrong, everything is wrong.

`src/recon/money.py` has one rule: every amount is a whole number of kobo, and ₦1 is 100 kobo. There are no fractions of a kobo anywhere: not in a variable, not in the database, not halfway through a sum. Here's the real check:

```python
if isinstance(self.kobo, bool) or not isinstance(self.kobo, int):
    raise MoneyError(
        f"kobo must be an int, got {type(self.kobo).__name__}: {self.kobo!r}. "
        "A float here means someone divided money somewhere."
    )
```

If anything that isn't a whole number sneaks in, it stops with that message. Computers store decimals with tiny errors, and those errors add up. Whole kobo means sums are always exact.

Next, `src/recon/fees.py` knows what Paystack keeps from each payment, and when the rest lands in the bank: a day later. That's why sales and the bank credit never match exactly.

Tested in `tests/test_money.py`, which promises: "if these pass, no amount in the system can drift". And `tests/test_fees.py`.

> **📁 Files we just created**
> - `src/recon/money.py`: money as whole kobo, with no fractions allowed.
> - `src/recon/fees.py`: Paystack's fees and its next-day payout.
> - `tests/test_money.py` and `tests/test_fees.py`: money and fee tests.

## Step two: settings, a database and fixed words

`src/recon/config.py` reads the settings. And it has one strong opinion: it **refuses to start with a live Paystack key**.

```python
if value.startswith("sk_live_"):
    raise ConfigError(
        "PAYSTACK_SECRET_KEY is a live key. This service is test mode only. "
```

Reckon only ever reads, but a live key sitting in a program is still a risk. So it simply won't allow one.

`src/recon/db.py` opens the database. By default it's SQLite, a single file, so you can run it with no database server. Change one setting and it uses PostgreSQL instead.

`src/recon/models.py` designs the tables. Every money column is named something like `amount_kobo`, and it's a plain whole number. There's no decimal column anywhere.

`src/recon/enums.py` holds small, fixed sets of words the system can use, like payment statuses. It also gives each status a rank, which we'll need in a moment.

Tested in `tests/test_config.py` and `tests/test_models.py`.

> **📁 Files we just created**
> - `src/recon/config.py`: settings, refusing live payment keys.
> - `src/recon/db.py`: opens the database, SQLite or PostgreSQL.
> - `src/recon/models.py`: the tables, with money as whole kobo.
> - `src/recon/enums.py`: fixed words, like statuses, with their ranks.
> - `tests/test_config.py` and `tests/test_models.py`: settings and table tests.

## Step three: taking in payment alerts safely

When a payment happens, Paystack sends an automatic message, a webhook. But we can't just trust it.

First, `src/recon/paystack/signature.py` proves the message really came from Paystack. Paystack stamps each message with a secret code made from the message itself. We check that stamp. A fake message won't have the right one.

Then `src/recon/paystack/events.py` turns Paystack's six kinds of "money moved" messages into one simple shape that the rest of the system uses.

Now `src/recon/ingest.py` does the work in two separate steps, on purpose. Step one: check the stamp, save the raw message, and reply "got it" straight away. That has to be fast, because Paystack resends messages if the reply is slow. Step two happens afterwards, in the background, using its own database connection.

And in step two, we don't trust the message's details either. `src/recon/paystack/client.py` asks Paystack directly: "Is this payment real, and how much was it?" Paystack's answer always wins.

And remember those status ranks? A payment's status can only move *forward*.

That matters because messages arrive out of order. A refund can turn up before its payment. With ranks, an old message can never undo a newer state.

`src/recon/app.py` is the web front door: the webhook address and a health check. The review pages get added to it later.

Tests: `tests/test_signature.py`, `tests/test_events.py`, `tests/test_ingest.py` and `tests/test_app.py`. And `tests/factories.py` builds pretend webhook messages, shaped exactly the way Paystack shapes them.

> **📁 Files we just created**
> - `src/recon/paystack/__init__.py`: marks the Paystack folder as a package.
> - `src/recon/paystack/signature.py`: proves a message came from Paystack.
> - `src/recon/paystack/events.py`: turns six message types into one shape.
> - `src/recon/paystack/client.py`: asks Paystack directly to confirm a payment.
> - `src/recon/ingest.py`: receive fast, then process in the background.
> - `src/recon/app.py`: the web front door.
> - `tests/test_signature.py`, `tests/test_events.py`, `tests/test_ingest.py`, `tests/test_app.py`: intake tests.
> - `tests/factories.py`: realistic pretend Paystack messages.

## Step four: the most valuable file

Now we need data to test on. But real business payments can't be published. So we make our own, honestly.

`src/recon/corpus/generate.py` builds a realistic month of payments, *and the answer key*. Its own notes call it "the most valuable file in the repo". Anyone can write a matcher. The hard part is having something honest to check it against.

It writes out every tricky situation on purpose: payments with an invoice reference, payments into a customer's own account, cash, duplicates, twin invoices that fit equally, and money from strangers.

Two helpers make it realistic. `src/recon/corpus/names.py` has Nigerian names, and all the ways they get written down wrong. `src/recon/corpus/narrations.py` has the notes Nigerian banks put on transfers, like `NIP/GTB/OKONKWO ADA/PAYMENT`. Often that's all you get: no customer number, no order number.

The generated data is saved in a `fixtures` folder:

- `fixtures/customers.json`: 57 customers.
- `fixtures/orders.json`: 520 invoices.
- `fixtures/transactions.json`: 439 payments.
- `fixtures/dedicated_accounts.json`: 22 customers' own payment accounts.
- `fixtures/settlements.json`: 23 Paystack payouts to the bank.
- `fixtures/ground_truth.json`: the answer key for all 439 payments.
- `fixtures/intake_labelled.json`: 40 written payment reports, with the right answers.
- `fixtures/manifest.json`: a summary of what was made, and how many of each situation.

Tested in `tests/test_corpus.py`. It checks the data like evidence: the money is exact, and the answer key adds up.

> **📁 Files we just created**
> - `src/recon/corpus/__init__.py`: marks the corpus folder as a package.
> - `src/recon/corpus/generate.py`: builds a month of payments and the answer key.
> - `src/recon/corpus/names.py`: Nigerian names and their common misspellings.
> - `src/recon/corpus/narrations.py`: realistic bank transfer notes.
> - `fixtures/customers.json`, `orders.json`, `transactions.json`, `dedicated_accounts.json`, `settlements.json`: the generated month.
> - `fixtures/ground_truth.json`: the answer key.
> - `fixtures/intake_labelled.json`: written payment reports with answers.
> - `fixtures/manifest.json`: a summary of what was generated.
> - `tests/test_corpus.py`: checks the generated data.

Let's pause. Money is exact. Settings are safe.

Alerts come in safely and get confirmed. And we have an honest month of data with an answer key. Now we can start matching.

## Step five: layer one, the certain rules

First, `src/recon/match/ledger.py` loads the book of who owes what, and indexes it for quick lookups. Is this text a real invoice number? Whose special account is this? Which invoices are open for this amount?

Then `src/recon/match/result.py` decides what every matching layer hands back: the match, which layer decided, and why.

Now `src/recon/match/deterministic.py`, layer one. Everything here is exact. A duplicate check goes first.

Then an exact invoice number, checked against the real book. Then money landing in a customer's own account, with exactly one invoice of that amount. Then exactly one open invoice for that exact amount, in the right time window.

And here's its golden rule: **it never fires on a tie**. If two invoices fit equally, it steps back. A tie isn't certainty.

Tests: `tests/test_deterministic.py`, whose note says it all: "layer one must be certain or silent". And `tests/test_deterministic_on_corpus.py` pins layer one's numbers on the generated month. If a change moves them, this test fails, and the README must be updated in the same change.

> **📁 Files we just created**
> - `src/recon/match/__init__.py`: marks the matching folder as a package.
> - `src/recon/match/ledger.py`: the book of who owes what, ready for lookups.
> - `src/recon/match/result.py`: what every matching layer hands back.
> - `src/recon/match/deterministic.py`: layer one, exact rules that refuse ties.
> - `tests/test_deterministic.py` and `tests/test_deterministic_on_corpus.py`: layer one tests.

## Step six: matching messy names

Before layer two, we need to compare names. And that's hard here. The name on the transfer and the name on the invoice are the same person written two different ways.

`src/recon/match/similarity.py` handles it. It tidies case and punctuation, removes bank noise words like "NIP" and "PAYMENT", handles spelling variants and short forms, and doesn't care about word order. But its most important job is the opposite: telling two *different* customers apart, even when their names look alike.

Tested in `tests/test_similarity.py`. Its note says: "name similarity has one job that matters: tell two customers apart."

> **📁 Files we just created**
> - `src/recon/match/similarity.py`: compares names carefully, without merging different people.
> - `tests/test_similarity.py`: tests name matching.

## Step seven: layer two, the scoring model

Now the payments layer one couldn't settle. First, `src/recon/match/candidates.py` builds a short list of invoices each payment could possibly be for. Comparing every payment against every open invoice would be silly, so this narrows it down.

Then `src/recon/match/features.py` turns each payment and invoice pair into 18 facts. Each is something a bookkeeper would check, like "does the surname match exactly?" or "have we seen this payer before?". And each is named after its question, so the reasons can be explained in words.

`src/recon/match/model.py` is a logistic regression, a simple model that weighs those facts and turns them into a chance. It's written out by hand, with no machine learning library. Why? With 18 facts and a few thousand rows, a hand-written model is small, readable and repeatable.

## Making the chance honest, and drawing the line

`src/recon/match/calibration.py` makes the chance mean what it says. If the model says 85% across many cases, about 85% of them should be right. It compares two methods, and Platt scaling was chosen.

Then `src/recon/match/threshold.py` draws the line between "close it" and "ask a person". And as its notes say, that's a business decision, not a maths one. Here are the real costs:

```python
COST_OF_A_REVIEW = _cost_of(REVIEW_MINUTES)
COST_OF_A_WRONG_AUTO_CLEAR = _cost_of(UNPICKING_MINUTES) + GOODWILL_COST
```

A review takes 3 minutes of a bookkeeper's time, about ₦60. A wrong close takes 2 hours to unpick plus ₦2,600 of goodwill, about ₦5,000 in all. So a mistake costs about 83 reviews. With those costs, the line comes out at 85%.

`src/recon/match/training.py` fits the model on exactly the payments layer one couldn't settle. And it picks the line carefully, testing on each part of that data in turn. `src/recon/match/probabilistic.py` is layer two itself: for each leftover payment, it scores the short list and closes a match only at 85% or more, with a clear winner.

The trained results are saved in a `models` folder: `models/model.json` (the weights), `models/calibrator.json` (the honesty adjustment) and `models/threshold.json` (the line, and how it was chosen).

And `src/recon/match/pipeline.py` runs the layers in order. Its notes say it's "the whole architecture in one file". Nothing reaches a later layer that an earlier one could have settled.

Tests: `tests/test_model.py` checks the model, calibration and line, each on its own. `tests/test_probabilistic.py` checks layer two and the order of layers. And `tests/test_pipeline_on_corpus.py` checks both layers on the generated month, which are the README's headline numbers.

> **📁 Files we just created**
> - `src/recon/match/candidates.py`: a short list of possible invoices.
> - `src/recon/match/features.py`: 18 bookkeeper-style facts per pair.
> - `src/recon/match/model.py`: the hand-written logistic regression.
> - `src/recon/match/calibration.py`: makes the chances honest.
> - `src/recon/match/threshold.py`: the 85% line, from real costs.
> - `src/recon/match/training.py`: fits the model and picks the line.
> - `src/recon/match/probabilistic.py`: layer two, scoring the leftovers.
> - `src/recon/match/pipeline.py`: runs the layers in order.
> - `models/model.json`, `models/calibrator.json`, `models/threshold.json`: the saved trained results.
> - `tests/test_model.py`, `tests/test_probabilistic.py`, `tests/test_pipeline_on_corpus.py`: layer two tests.

Let's circle back. We have certain rules, careful name matching, and a scoring model with honest chances and a cost-based line. The matching brain is done. Now, what about payments reported in plain words?

## Step eight: reading written payment reports

Lots of businesses hear about payments in a WhatsApp message, like "Ada paid 45k for invoice 42 this morning". So we need to read those.

`src/recon/intake/schema.py` says exactly what a payment report must look like before it's allowed in. `src/recon/intake/amounts.py` reads amounts out of sentences, because Nigerians write money in lots of ways, like "45k" or "45,000". And `src/recon/intake/parse.py` uses patterns to turn the whole sentence into a proper record, or refuses if it can't.

Then the last resort, following the house rule: AI last. There's an optional AI reader file in the intake folder. It only runs on reports the patterns gave up on.

It must answer in the exact shape the schema defines, it may say "I can't read this", and it's only asked once. It's switched off by default, so the tool never quietly runs up a bill.

Tested in `tests/test_intake.py`. Its rule: "when in doubt, hand it to a person". A reader that invents a payer might score well, but it puts made-up names in the books.

> **📁 Files we just created**
> - `src/recon/intake/__init__.py`: marks the intake folder as a package.
> - `src/recon/intake/schema.py`: what a payment report must look like.
> - `src/recon/intake/amounts.py`: reads amounts like "45k".
> - `src/recon/intake/parse.py`: turns a sentence into a record, or refuses.
> - The optional AI reader file in the same folder: the last resort, off by default.
> - `tests/test_intake.py`: tests the reader.

## Step nine: the review list, the report, and the record

Now the person's side. `src/recon/review/queue.py` makes the review list, sorted by money at risk, biggest first. Why? A bookkeeper has about two hours and roughly forty decisions in them, so the biggest money should come first.

`src/recon/review/web.py` builds the pages: the review list, and the daily report. They're plain web pages with no JavaScript, because they'll be used on a phone, in a shop. The page designs are in `src/recon/review/templates/`: `base.html` is the shared frame, `queue.html` is the review list, and `report.html` is the daily report.

When a person rejects a suggestion, it's saved as a label, for a person to use later to improve the model. Nothing retrains by itself.

`src/recon/report/settlement.py` is the daily report: what came in, what Paystack kept, and what reached the bank, all shown separately. Its job is to explain the difference, not hide it.

And `src/recon/audit.py` is the permanent record. It has one function, `record`. It adds a row, and it never edits or deletes one. As its notes say, if you want to correct an audit row, you write a new one.

`src/recon/state.py` holds everything the pages read from: the loaded data, the trained matcher and the matches, kept in memory. Its notes say plainly that it's a single-program demo.

Tests: `tests/test_review.py` for the list and pages, and `tests/test_settlement.py` for the daily report.

> **📁 Files we just created**
> - `src/recon/review/__init__.py` and `src/recon/report/__init__.py`: mark those folders as packages.
> - `src/recon/review/queue.py`: the review list, biggest money first.
> - `src/recon/review/web.py`: the review and report pages.
> - `src/recon/review/templates/base.html`, `queue.html`, `report.html`: the page designs.
> - `src/recon/report/settlement.py`: the daily report.
> - `src/recon/audit.py`: the permanent, add-only record.
> - `src/recon/state.py`: what the pages read from, held in memory.
> - `tests/test_review.py` and `tests/test_settlement.py`: review and report tests.

## Step ten: proving every number

Now a rule I really like: **nothing goes in the README that can't be rebuilt**.

`src/recon/evaluation/harness.py` is behind `make eval`. It rebuilds the data, retrains the model, and recomputes every number in the README from scratch.

`src/recon/evaluation/truth.py` reads the answer key and decides if a match is right. That's harder than it sounds with twin invoices, where either answer is correct. `src/recon/evaluation/score.py` turns matches into the final numbers. And `src/recon/evaluation/extraction.py` scores the written-report reader separately, because reading and matching fail for different reasons.

Tests: `tests/test_harness.py` checks the harness. And `tests/test_readme.py` checks the README is quoting the real numbers. Without a test, a rule like that wouldn't last.

> **📁 Files we just created**
> - `src/recon/evaluation/__init__.py`: marks the evaluation folder as a package.
> - `src/recon/evaluation/harness.py`: rebuilds every README number from scratch.
> - `src/recon/evaluation/truth.py`: reads the answer key and judges matches.
> - `src/recon/evaluation/score.py`: turns matches into numbers.
> - `src/recon/evaluation/extraction.py`: scores the report reader on its own.
> - `tests/test_harness.py` and `tests/test_readme.py`: keep the README honest.

## Step eleven: shipping it

`Dockerfile` packs Reckon into a box, called a container, that runs the same anywhere. `.dockerignore` says what to leave out of it.

`cloudbuild.yaml` builds the box on Google Cloud. And here's the key rule: the smoke test is a build step. As its notes say, a box can start cleanly and still break on the review pages. So nothing gets deployed until the real pages have been opened.

`scripts/smoke.py` is that smoke test. It checks a running copy really works, from the outside. `scripts/deploy.sh` deploys to Google Cloud Run, then runs the smoke test again on the live copy. `docs/DEPLOY.md` explains how to run it on your own computer and in the cloud.

`.github/workflows/ci.yml` runs the checks on every change. And `tests/test_packaging.py` catches "things that only break after you deploy". Shared test helpers live in `tests/conftest.py` and `tests/__init__.py`.

Finally, `docs/DECISIONS.md` records every big decision: what was picked, what wasn't, and what it costs.

> **📁 Files we just created**
> - `Dockerfile` and `.dockerignore`: pack the app into a box.
> - `cloudbuild.yaml`: builds the box, with the smoke test as a required step.
> - `scripts/smoke.py`: checks a running copy really works.
> - `scripts/deploy.sh`: deploys, then smoke-tests the live copy.
> - `docs/DEPLOY.md`: how to run it locally and in the cloud.
> - `.github/workflows/ci.yml`: checks every change.
> - `tests/test_packaging.py`: catches things that break only after deploying.
> - `tests/conftest.py` and `tests/__init__.py`: shared test helpers.
> - `docs/DECISIONS.md`: every big decision and its cost.

## So, how's it doing?

On the generated month of 439 payments, worth about ₦37 million:

- 76% closed without a person: 70% by the certain rules, and 6% by the model.
- **0 wrong closes** out of 334.
- 105 went to review, and the first 40 covered 91% of the money at risk.
- About 17 hours of work saved a month.

`make eval` rebuilds all of these, and they come out the same every time. But the data is made up. On real data, there will be some mistakes.

## What's still missing?

- **All results come from made-up data**, so they depend on the guesses used to make it.
- **The 85% line** was set using only 87 payments.
- **Cash can't be confirmed** with Paystack.
- **Nothing retrains by itself.**
- **Small old payments can wait forever**, because the list sorts by money only.
- **Only Paystack is connected.**
- **Without the optional AI extra installed, 8 tests fail**, though the project's notes say they shouldn't.
- **It hadn't been deployed yet**, according to the README.

## Let's put it all together

So let's look at it in one breath.

We set up the workshop with strict quality checks. We made **money exact**, in whole kobo. We made **settings safe**, refusing live keys. We took in **payment alerts** safely: check the stamp, reply fast, then confirm with Paystack, with status that only moves forward.

We built an **honest month of data** with an answer key. Then **layer one**, exact rules that refuse ties, **careful name matching**, and **layer two**, a hand-written model with **honest chances** and an **85% line from real costs**. We added a **reader for written reports**, with AI only as a switched-off last resort.

Then the **review list**, biggest money first, the **daily report**, and a **permanent record**. And we made sure **every README number can be rebuilt**, with a test to enforce it.

Notice how it links. The house rule, certain first, scored second, AI last, shows up in matching *and* in reading reports. The answer key from step four is what lets us prove the numbers in step ten. And whole kobo from step one runs through every single file.

That's the project. Match what you can prove, and let people decide the rest.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
