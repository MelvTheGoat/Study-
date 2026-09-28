# Nigerian-Fintech-Compliance-RAG: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

---

## 60-second pitch

> "Nigerian fintech rules are scattered across CBN and NDPC documents, and some questions have two right answers, like breach reporting deadlines. I built a RAG assistant that answers only from the indexed regulations, cites every claim, and prints the source passage underneath. Retrieval is hybrid: BM25 for exact terms and ONNX-run MiniLM embeddings for meaning, merged with reciprocal rank fusion. The model must cite passages S1 to S6 or emit a refusal marker, and an output guard strips any citation to a passage that wasn't retrieved. Personal data like BVN and NIN is scrubbed before anything leaves the machine.
>
> On 60 hand-labelled questions, the right evidence is in the top 5 96% of the time, with MRR 0.842, and it all fits in about 225 MB by avoiding PyTorch. Two honest caveats: the repo ships sample summaries rather than official texts, and answer quality with a real LLM isn't measured yet. CI uses a stub model."

---

## Questions and honest answers

### 1. "Why hybrid search?"
Regulations are full of exact terms, like "Tier 1", "72 hours" and "N5,000,000", which keyword search nails. Meaning-based search catches paraphrases. My ablation showed keyword search does most of the work here. Swapping the real embedder for a hashing stand-in only dropped MRR from 0.842 to 0.820.

### 2. "Why RRF instead of adding scores?"
BM25 scores are unbounded and cosine scores sit in a narrow band, so adding them lets BM25 dominate. RRF uses only ranks, `1/(k+rank)` with k=60, so a passage both methods like rises, and one method's confident mistake can't take over.

### 3. "Why no vector database?"
100 chunks. Exact cosine is one matrix multiply in milliseconds, and would be at 100× the size too. A vector DB adds a dependency and approximation error for no gain.

### 4. "Why not PyTorch?"
Memory. PyTorch costs 350–450 MB before doing anything. Running the same MiniLM through onnxruntime keeps the whole app at ~225 MB peak on a 2 GB free tier.

### 5. "How do you stop hallucinated citations?"
Passages are labelled S1–S6. The prompt forbids inventing markers, and the output guard strips any citation to a marker that wasn't retrieved and labels that claim unverified. It's visible, not silent.

### 6. "How do you handle off-topic questions?"
The model decides, using a refusal marker. I tested a cheap retrieval-score cutoff and it failed: off-topic and on-topic scores overlap. "Data protection rules in Kenya" scored higher than 40 of 50 fair questions. A cutoff catching 90% of off-topic questions rejected 26% of fair ones.

### 7. "How do you know it works?"
Retrieval is measured properly: 50 labelled questions, recall@5 0.960, hit@10 1.000, MRR 0.842, reproducible without a key. Generation is only measured with a quoting stub in CI, so those scores are floors or near-tautologies. The real-LLM run is a one-liner with a key, and I'd report it before claiming answer quality.

### 8. "Your 0.400 refusal score is bad."
It's the stub's score. A stub that quotes can't refuse. With a real model, refusal depends on the model reading the question. I'd measure it with the real provider and report it next to false refusals.

### 9. "How do you protect personal data?"
The input is scrubbed for BVN, NIN, NUBAN, +234 numbers, cards and emails before any third-party call, and the output is scrubbed again. It's pattern-based, so it reduces exposure but doesn't guarantee it. I'd tell users not to paste customer records.

### 10. "What about prompt injection?"
Low severity here: no tools, no secrets, and the documents are my own. The input filter mainly stops quota abuse. It's a heuristic, so a block gives a message, not a ban.

### 11. "The documents aren't real. Doesn't that undermine everything?"
It limits real-world use, and the app warns in three places. But the pipeline and the retrieval metrics are real measurements on a real index. Swapping in the official PDFs is two commands. The fetch script already lists the exact URLs.

### 12. "How would you scale to all CBN circulars?"
Fetch and diff on a schedule, version the index by date, add an abbreviation glossary, move to an ANN index only past ~100× the size, share rate limits via Redis, and build a cross-reference graph so linked clauses come back together.

### 13. "Why section-based chunking?"
Legal text is organised by numbered clauses. A chunk that is one clause gives clean citations and avoids mixing unrelated rules.

### 14. "What's the weakest part?"
The corpus is non-authoritative, and generation quality with a real model is unmeasured. After that, abbreviations: "CISO" isn't matched to "Chief Information Security Officer" by the fallback embedder.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Sample documents | "It's not real law." | "Correct, and flagged everywhere. The system and retrieval metrics are real. Official PDFs are a fetch and a rebuild away." |
| Stub generation metrics | "Perfect groundedness is meaningless." | "Agreed, it proves the plumbing. A real-LLM eval is the next measurement." |
| Small golden set | "60 questions written by you." | "Yes. It catches regressions, and I'd add questions from real compliance staff." |
| Lexical judge | "It punishes paraphrase." | "It's a conservative floor. A semantic judge or human review would be better." |
| Re-ranker unmeasured | "Why include it?" | "Memory is measured (367 MB peak). Quality isn't, so it's off by default." |
