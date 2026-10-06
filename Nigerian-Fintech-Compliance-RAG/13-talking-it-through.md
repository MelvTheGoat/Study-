# Nigerian Fintech Rules Q&A: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: the app currently holds six written summaries of the rules, not the official documents. So it's a working demo, not a legal tool yet.*

---

## Okay, so what are we building?

Alright. If you run a Nigerian fintech, you have to follow a lot of rules. They come from the Central Bank of Nigeria (CBN), the Nigeria Data Protection Act 2023, and the data protection commission (NDPC).

And they're scattered across PDFs on different websites, referring to each other, changing quietly. Even a simple question like "how fast must we report a data breach?" can have two right answers, one for each regulator.

So here's our project: an app that answers questions about these rules in plain English. But it only answers from its stored documents. Every claim gets a label like [S1], showing exactly which passage it came from, with the passage shown underneath. And if the documents don't cover the question, it refuses.

This approach is called RAG, retrieval-augmented generation. In plain words: first *find* the right passages, then let an AI *write* the answer using only those. Like an open-book exam.

## So what do we need?

1. **The documents**, and a way to fetch them.
2. **A way to read them** while keeping their section numbers.
3. **Chunking**: cutting them into small pieces, one section each.
4. **Two kinds of search**: by keywords and by meaning.
5. **A way to merge** the two searches.
6. **A ready-made search index**, so the app starts fast.
7. **Strict instructions for the AI**, so it only uses the passages and cites them.
8. **Safety checks** before and after the AI, including removing personal details.
9. **Usage limits**, to protect the free AI quota.
10. **The app page.**
11. **An honest evaluation**, that can run with no internet and no key.
12. **A small memory footprint**, because it must fit in 2 GB on a free server.

## Step zero: set up the workshop

First, a `README.md`, the front page, and a `.gitignore`. Then `pyproject.toml`, the project's details and tool settings.

The install lists are split in two, and that's important here. `requirements.txt` is what the live app needs, and its own note says every package costs memory on a 2 GB box. So there's deliberately no PyTorch. `requirements-dev.txt` adds the testing tools, and its note says they're "never installed" on the live server.

`.env.example` is a template for settings, like which AI service to use and its key. And `src/config.py` reads every setting from the environment, so the live app can be reconfigured without changing code.

The code lives in `src/`, with `src/__init__.py` describing it: "Retrieval-augmented assistant for Nigerian fintech regulation."

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: project details and tool settings.
> - `requirements.txt`: what the live app installs, kept small for memory.
> - `requirements-dev.txt`: testing tools, never installed live.
> - `.env.example`: a template for settings and keys.
> - `src/__init__.py`: marks the code folder as a package.
> - `src/config.py`: every setting, read from the environment.

## Step one: get the documents

We start with the documents, because everything depends on them.

`scripts/fetch_corpus.py` tries to download the official CBN and NDPC PDFs. But Nigerian regulator websites often block automated programs, show error pages, or move files around. And that's exactly what happened: it couldn't get any of the 7.

So instead of failing quietly, it writes `MANUAL_SOURCES.md`. That file lists every document, where to download it in a normal browser, and the exact file name to save it as. It fails loudly and helpfully.

In the meantime, six written summaries stand in, in `data/corpus/`:

- `cbn_aml_cft_regulations.md`: anti-money-laundering rules.
- `cbn_cybersecurity_framework_psp.md`: cybersecurity rules for payment companies.
- `cbn_psp_licensing.md`: licensing for payment service providers.
- `cbn_tiered_kyc.md`: customer checks, by account tier.
- `ndpa_2023.md`: the Nigeria Data Protection Act 2023.
- `ndpc_gaid_guidance.md`: the NDPC's guidance on applying the Act.

Each has a header saying what it is, who issued it, and that it's a "sample summary". The app uses that to show a warning.

> **📁 Files we just created**
> - `scripts/fetch_corpus.py`: tries to download the official documents.
> - `MANUAL_SOURCES.md`: what to download by hand, and where to save it.
> - `data/corpus/cbn_aml_cft_regulations.md`, `cbn_cybersecurity_framework_psp.md`, `cbn_psp_licensing.md`, `cbn_tiered_kyc.md`, `ndpa_2023.md`, `ndpc_gaid_guidance.md`: the six sample summaries.

## Step two: read them, keeping the structure

Now we read the documents. And here's the key thing: regulations are organised in parts, sections and subsections, and that's what people cite. So we must keep it.

`src/ingest/schema.py` defines the shapes the rest of the app uses, especially a "chunk". And its notes make a strong point: citations are the product. If a chunk's document name or section number is wrong, the citation is wrong.

`src/ingest/extract.py` pulls text out of PDF, Markdown and text files, keeping the section structure, plus details like who issued it, the date, and whether it's official.

`src/ingest/chunking.py` cuts the text into chunks. The default follows section boundaries, because a clause split in half loses its meaning. Only sections that are too long get split, at paragraphs or sentences, up to 1,200 characters, with 180 characters of overlap. The six documents become 100 chunks.

Tested in `tests/test_ingest.py`.

> **📁 Files we just created**
> - `src/ingest/__init__.py`: marks the reading folder as a package.
> - `src/ingest/schema.py`: the shapes, especially a chunk with its citation details.
> - `src/ingest/extract.py`: reads files while keeping sections.
> - `src/ingest/chunking.py`: cuts text at section boundaries.
> - `tests/test_ingest.py`: reading and chunking tests.

## Step three: two kinds of search

Now search. We use two kinds, because each catches things the other misses.

**Keyword search.** `src/retrieval/bm25.py` is BM25, a standard formula that ranks passages by shared words, giving rare words more weight. It's written by hand rather than installed, so the index can be saved as plain text and committed. Regulations are full of exact terms like "Tier 1" and "72 hours", so keyword search really matters here.

**Meaning search.** `src/retrieval/embedder.py` turns text into embeddings, lists of numbers that capture meaning. It uses a small free model called MiniLM, run through ONNX Runtime instead of the usual heavy toolkit. Same model, same results, but about 150 MB of memory instead of about 600 MB.

`src/retrieval/store.py` holds the saved index: the chunk details, the embeddings and the keyword index. With only 100 chunks, the meaning search is one quick calculation, so no special database is needed.

## Merging the two

The two searches score in different ways, so you can't just add their scores. `src/retrieval/fusion.py` merges them by *position* instead, using Reciprocal Rank Fusion:

```python
def reciprocal_rank_fusion(
    ranked_lists: Sequence[Sequence[int]],
    k: int = 60,
```

A chunk ranked high in both lists wins. It's like two judges each ranking contestants, then adding up the places.

There's also `src/retrieval/rerank.py`, an optional second pass with a slower, more careful model. It's off by default. `scripts/export_reranker.py` prepares that model, if you want to switch it on.

And `src/retrieval/pipeline.py` runs it all: both searches, merge, optional re-rank, then the top chunks with their citation details.

Tested in `tests/test_retrieval.py`.

> **📁 Files we just created**
> - `src/retrieval/__init__.py`: marks the search folder as a package.
> - `src/retrieval/bm25.py`: keyword search, written by hand.
> - `src/retrieval/embedder.py`: meaning search, with a light model runner.
> - `src/retrieval/store.py`: the saved search index.
> - `src/retrieval/fusion.py`: merges the two searches by position.
> - `src/retrieval/rerank.py`: an optional careful re-ranking, off by default.
> - `scripts/export_reranker.py`: prepares the re-ranking model.
> - `src/retrieval/pipeline.py`: runs the whole search.
> - `tests/test_retrieval.py`: search tests.

## Step four: build the index once

Making embeddings takes minutes. We don't want the live app doing that every time it starts. So `scripts/build_index.py` builds the index once, on your own computer, and saves it into an `index` folder that's committed with the code.

- `index/chunks.jsonl`: all 100 chunks, with their citation details.
- `index/vectors.npy`: the embeddings.
- `index/bm25.json`: the keyword index.
- `index/manifest.json`: when and how it was built, so it can be reproduced.

The live app just loads these at start-up. Fast, and the same every time.

Let's pause. We have documents, read with their sections.

We have chunks, two kinds of search, a way to merge them, and a ready-made index. That's the "find" half of RAG done. Now the "write" half.

> **📁 Files we just created**
> - `scripts/build_index.py`: builds the search index once, offline.
> - `index/chunks.jsonl`, `index/vectors.npy`, `index/bm25.json`, `index/manifest.json`: the saved index.

## Step five: strict instructions for the AI

Now we write the answer. And the goal, as the project's notes say, is to make a wrong-but-confident answer *hard* to produce. In a regulatory setting, "I couldn't find this" is far better than a made-up rule.

`src/constants.py` holds a few special markers that both the instructions and the safety checks need, like the "refuse" signal. They live in their own file so the two sides always agree.

`src/generation/prompts.py` builds the instructions. The top 6 passages are labelled S1 to S6. The AI is told: answer only from these, cite every claim, never invent a label, and use the refuse signal if they don't cover the question.

`src/generation/providers.py` sends it to an AI service. There's no AI model on the server itself, because there's no room for one. So it's a call to a free-tier service: Groq, Gemini or OpenAI. Or a "stub" that just quotes passages, which is used for testing with no key.

`src/generation/answer.py` runs the whole thing: search, build the instructions, get the answer, check it. It can stream the answer word by word for the app, or return it all at once for testing.

`src/textutils.py` has small shared text helpers, so the stub and the evaluation judge agree on what counts as a "content word".

Tested in `tests/test_generation.py`.

> **📁 Files we just created**
> - `src/constants.py`: shared markers, like the refuse signal.
> - `src/generation/__init__.py`: marks the answer folder as a package.
> - `src/generation/prompts.py`: strict instructions, with passages labelled S1 to S6.
> - `src/generation/providers.py`: calls the AI service, or the free stub.
> - `src/generation/answer.py`: search, instruct, answer, check.
> - `src/textutils.py`: shared text helpers.
> - `tests/test_generation.py`: answer tests.

## Step six: safety checks

Now the safety checks, on both sides of the AI.

`src/guardrails/pii.py` finds and removes personal details. And its notes make a sharp point: people using a customer-checks tool are exactly the people who'll paste customer records into the box. So it looks for BVNs, NINs, bank account numbers, Nigerian phone numbers, card numbers and emails.

`src/guardrails/input_guard.py` checks the question on the way in. It checks the length, looks for tricks like "ignore your rules", and removes personal details *before* anything goes to an outside AI service.

`src/guardrails/output_guard.py` checks the answer on the way out. The most important check is simple: the answer may only cite labels that were actually given. If the AI cites [S9], that citation is removed and the claim is marked "unverified". It also removes personal details again.

Tested in `tests/test_guardrails.py`.

> **📁 Files we just created**
> - `src/guardrails/__init__.py`: marks the safety folder as a package.
> - `src/guardrails/pii.py`: finds and removes personal details.
> - `src/guardrails/input_guard.py`: checks questions on the way in.
> - `src/guardrails/output_guard.py`: checks answers on the way out, catching fake citations.
> - `tests/test_guardrails.py`: safety tests.

## Step seven: limits and feedback

A public AI app with no limits is, as the notes say, "a bill waiting to happen". On a free tier, one busy afternoon could use up the whole day's quota.

So `src/limits.py` caps each visitor at 8 questions per 5 minutes, and the whole app at 300 a day. The daily count is saved to disk, so it survives a restart.

And `src/feedback.py` records thumbs up and thumbs down. It only adds entries, never edits them. It's the cheapest signal of quality, and the only one that comes from real questions.

Tested in `tests/test_limits.py`.

> **📁 Files we just created**
> - `src/limits.py`: per-visitor and daily limits.
> - `src/feedback.py`: an add-only record of thumbs up and down.
> - `tests/test_limits.py`: limit and feedback tests.

## Step eight: the app

`app.py` is the page itself, built with Streamlit, a tool for making simple web pages in Python. It shows the question box, the answer, every cited passage in full, and warnings on sample documents.

One nice touch: a free server sleeps when nobody's using it. So when it wakes up, the app shows a "starting up" notice instead of looking broken.

Tested in `tests/test_app.py`. It's the one file a deployment actually runs, so it gets its own smoke tests. Shared test helpers live in `tests/conftest.py`, with a tiny made-up set of documents, so tests need no internet, model download or key.

> **📁 Files we just created**
> - `app.py`: the web page.
> - `tests/test_app.py`: smoke tests for the page.
> - `tests/conftest.py`: shared test helpers with tiny made-up documents.

## Step nine: an honest evaluation

Now, how good is it? `eval/golden_set.yaml` holds 60 test questions: 50 that should be answered, each with the right sections labelled, and 10 that should be refused.

`eval/metrics.py` measures search and answers **separately**, on purpose. They fail for different reasons and get fixed in different ways. `eval/judge.py` judges whether an answer sticks to its passages. By default it works offline, with no key, so the automatic checks can run it every time and get the same result.

`eval/run_eval.py` runs it all. It writes `eval/results.json` with the raw numbers and `eval/RESULTS.md` with a readable report. `eval/__init__.py` marks the folder as a package, and `tests/test_eval.py` tests the evaluation itself.

> **📁 Files we just created**
> - `eval/__init__.py`: marks the evaluation folder as a package.
> - `eval/golden_set.yaml`: 60 test questions.
> - `eval/metrics.py`: search and answer scores, measured separately.
> - `eval/judge.py`: an offline judge of whether answers stick to the passages.
> - `eval/run_eval.py`: runs the evaluation.
> - `eval/results.json` and `eval/RESULTS.md`: the results, raw and readable.
> - `tests/test_eval.py`: tests the evaluation.

## Step ten: proving it fits, and shipping it

The app must fit in 2 GB of memory. So `scripts/check_memory.py` measures the real peak memory. It's about 225 MB, or about 367 MB with the re-ranker on.

`Dockerfile` packs the app into a box, called a container, that runs the same anywhere. `DEPLOY.md` gives the exact steps to run it on Google Cloud Run, which scales down to nothing when nobody's using it. That means it costs close to nothing at low traffic.

And `.github/workflows/ci.yml` checks every change automatically: lint, types and the 110 tests, all without a key or the internet.

> **📁 Files we just created**
> - `scripts/check_memory.py`: measures real peak memory.
> - `Dockerfile`: packs the app into a box.
> - `DEPLOY.md`: steps to run it on Google Cloud Run.
> - `.github/workflows/ci.yml`: automatic checks on every change.

## So, how's it doing?

Search works well. The right passage was in the top 5 results 96% of the time, and no question missed it in the top 10.

But answer quality with a real AI **hasn't been measured**. The saved results use the quoting stub, which makes some scores look perfect automatically. The stub only refused 4 of the 10 off-topic questions.

## What's still missing?

- **The documents are summaries**, not the official rules, and some are out of date.
- **Real AI answers haven't been measured.**
- **Off-topic questions can't be caught by search scores**, because fair and off-topic scores overlap too much. So refusing is left to the AI.
- **Short forms** like "CISO" may not match the full title in the text.
- **Personal details are found by patterns**, which can miss unusual formats.

## Let's put it all together

So let's look at it in one breath.

We **tried to fetch** the official documents and, when that failed, wrote clear instructions for doing it by hand. We **read the documents** keeping their sections, and **chunked them by section**, giving 100 chunks. We built **two kinds of search**, merged them **by position**, and saved a **ready-made index**.

Then we wrote **strict instructions** for the AI, with labelled passages and a refuse signal. We added **safety checks** on both sides: removing personal details, and catching fake citations. We added **limits**, a **feedback record**, and the **app page**. Finally, an **honest offline evaluation**, a **memory check**, and **automatic checks**.

Notice how it links. The section structure kept in step two is what makes citations precise. The light model runner in step three is what lets the app fit in 2 GB. And the shared markers in `constants.py` are what let the instructions and the output check agree on what a refusal looks like.

That's the project. Find first, write only from what you found, and show your sources.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
