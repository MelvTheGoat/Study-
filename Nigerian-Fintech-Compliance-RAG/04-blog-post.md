# A regulation assistant that shows its working: RAG for Nigerian fintech compliance

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

## Why I built it

Nigerian fintech compliance rules are scattered. Some live in CBN circulars, some in the Nigeria Data Protection Act 2023, some in NDPC directives. They're PDFs on several websites, they refer to each other constantly, and they get revised quietly.

So when someone asks a simple question, you can either read a few hundred pages or guess. And guessing is expensive. Take "how long do we have to report a data breach?". It has **two** answers. The NDPA gives one deadline for telling the Commission. The CBN cybersecurity framework gives a different one for telling the Payments System Management Department. A licensed PSP that just had a breach has both clocks running.

I wanted a tool that answers in plain English, but only from the documents, with every claim cited and the exact passage printed underneath, so a compliance person can check it in about five seconds. And one that says "I don't know" when the documents don't cover it.

One honest note first: **the repo ships six written summaries, not the real regulations.** The official PDFs were blocked from my build environment, so the fetch script wrote a list of exact URLs instead. The summaries follow the real section numbering but are paraphrased, and some figures are out of date. Everything below is about the system, which works the same once the real PDFs are dropped in.

## How it works

### Built ahead of time

Documents are extracted (PDF, Markdown or text), split **by section**, because legal text is organised in clauses and a chunk should be one clause, and indexed twice:

- a **keyword index** (BM25), and
- a **meaning index**: 384-dimensional embeddings from `all-MiniLM-L6-v2`.

The index is committed to the repo, so deploying never rebuilds it. A manifest records the build date and settings.

### At question time

1. **Check the question.** Length cap, obvious prompt-injection attempts, and personal data such as BVN, NIN, NUBAN account numbers, `+234` phone numbers, cards and emails are **scrubbed before anything leaves the machine**.
2. **Search twice.** Keyword search is great at exact terms like "Tier 1" or "72 hours". Meaning-based search catches paraphrases.
3. **Merge by rank.**
4. **Ask the model**, with the top 6 passages labelled [S1] to [S6], to answer only from them, cite each claim, and output a specific refusal marker if they don't cover the question.
5. **Check the answer.** Any citation pointing at a passage that wasn't retrieved is stripped, and the claim is labelled unverified.

## The choices I'd defend

### No PyTorch

The usual way to run MiniLM pulls in PyTorch, which costs 350–450 MB of memory before computing anything. I run the identical model through `onnxruntime` and `tokenizers` instead. On a 2 GB free-tier box, that one decision is why everything else fits. Measured peak memory for the whole deployed setup is about **225 MB**.

### No vector database

With 100 passages, the vectors are a plain numpy array, and search is one matrix multiplication: milliseconds now, and still milliseconds at 100× this size. A vector database would add a dependency and a build step to solve a problem I don't have.

### Merge by rank, not score

The two searches produce scores that mean different things. BM25 scores are unbounded, while cosine scores sit in a narrow band. Add them, and keyword search drowns out the other. So I throw the scores away and combine positions, with Reciprocal Rank Fusion:

```python
for ranked, weight in zip(ranked_lists, weights, strict=True):
    for rank, doc_id in enumerate(ranked, start=1):
        scores[doc_id] = scores.get(doc_id, 0.0) + weight / (k + rank)
```

With `k = 60` (from the original RRF paper), a passage both searches like rises to the top, and one search's confident-but-wrong top hit can't take over.

### Refusal is an output you can measure

The model emits a specific marker when the passages don't support an answer. That turns refusal from a hope into something the evaluation can count.

## Does it work?

I wrote 60 test questions: **50** fair ones with the correct source sections labelled by hand, and **10** it *should* refuse. Retrieval and answering are scored separately, because they break for different reasons. Anyone can re-run it with no API key (`python eval/run_eval.py`), and when I did, the numbers matched.

**Finding the right passage:**

| Metric | Score |
|---|---|
| Labelled evidence in the top 5 | **0.960** |
| Questions with a right passage in the top 10 | **1.000** |
| Mean reciprocal rank | **0.842** |

The app sends the top 6 passages to the model, so 0.960 in the top 5 is the number that matters. The evidence is almost always in front of the model.

**Writing the answer** comes with a big caveat. CI runs without an API key, so it uses a *stub* model that quotes passages, and a judge that compares words, not meaning. The perfect groundedness and citation scores are nearly guaranteed by a quoting stub. Key-fact recall (0.726) is a floor. And the 0.400 refusal rate is the stub's, not a real model's. **Answer quality with a real LLM isn't measured in the repo yet.**

## Two things that surprised me

### You can't cheaply spot an off-topic question

The obvious optimisation: if search results look weak, refuse and skip the model call. I tried it, and it doesn't work.

| | Lowest | Typical | Highest |
|---|---|---|---|
| Fair questions (50) | 0.349 | 0.601 | 0.785 |
| Off-topic questions (10) | 0.423 | 0.538 | 0.670 |

They overlap almost completely. "What are the data protection rules in Kenya?" scored 0.670, higher than 40 of the 50 fair questions. It *is* a question about data protection rules, and the documents are full of those. The one word that makes it unanswerable barely registers.

A cutoff that catches 90% of off-topic questions rejects **26%** of fair ones. For a tool whose job is answering questions, that's a terrible trade. So there's no cutoff. Refusal is left to the model, which can tell that Kenya isn't Nigeria.

### The clever half of search does less than I thought

I swapped the real embedding model for a dumb hashing stand-in with no understanding of language:

| Meaning-based search | MRR | Top-5 recall |
|---|---|---|
| Real MiniLM model | 0.842 | 0.960 |
| Hashing stand-in | 0.820 | 0.900 |

The real model adds about 2 points of rank quality and 6 of recall. That's real, but keyword search does most of the lifting. In hindsight it makes sense: regulation is full of distinctive exact terms. On a corpus with more paraphrasing, the gap should widen.

## Keeping it safe and cheap

- **In:** length cap (600 characters), injection heuristics, PII scrubbing before any third-party call.
- **Out:** invented citations stripped and flagged, and PII scrubbed again.
- **Abuse:** 8 questions per visitor per 5 minutes and 300 per day. The daily counter is saved to disk so a restart doesn't reset it.
- **Cold start:** on a free tier the first question after idle takes 30–45 seconds, and the app says so instead of showing a spinner that looks like a crash.

This app has no tools and no secrets to steal, so prompt injection is low-severity here. The input filter mostly stops someone turning it into a free general chatbot on my API quota.

## What I learned

- **Measure retrieval and generation separately.** They fail for different reasons.
- **Hybrid search needs rank fusion,** not score arithmetic.
- **Memory is a design constraint.** Dropping PyTorch was the biggest win in the project.
- **Negative results are results.** The failed off-topic cutoff and the small semantic gain were the most useful things I learned.
- **Be loud about sample data.** The warnings appear in the file, the source panel and the page.

## What's next

- Fetch the official PDFs and rebuild the index.
- Run the evaluation with a real LLM and a meaning-based judge.
- Measure whether the optional re-ranker improves answers (its memory cost is measured at 367 MB peak. Its quality isn't).
- Add a glossary for abbreviations like "CISO", which currently defeat both searches with the fallback embedder.
