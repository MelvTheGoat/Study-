# Nigerian-Fintech-Compliance-RAG: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

---

## Step 1: Requirements (2 min)

One line:
> "Answer questions about Nigerian fintech regulation using only the official texts, cite every claim, and refuse when the texts don't cover it."

**Functional**
1. Ingest regulation documents (PDF/Markdown/text).
2. Retrieve the relevant clauses for a question.
3. Generate an answer only from those clauses, with inline citations.
4. Show the full source passage for each citation.
5. Refuse out-of-scope questions.

**Non-functional**
- **Trustworthy:** no invented citations.
- **Private:** personal data scrubbed before any third-party call.
- **Cheap:** a free tier, 2 GB RAM, a free LLM key.
- **Measurable:** retrieval and generation scored separately.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Documents | 6 (sample summaries) |
| Chunks | 100 (section-based, ≤1,200 chars, 180 overlap) |
| Embedding dim | 384 (MiniLM) |
| Passages sent to the LLM | 6 |
| RAM budget | 2 GB. Peak ~225 MB. |
| Rate limits | 8 / 5 min / visitor, 300 / day |
| Eval | 60 questions (50 answerable + 10 refuse) |

**Say:** "The index is tiny. The real constraints are memory, trust and cost."

---

## Step 3: High-level boxes (2 min)

```
OFFLINE: [Docs] -> [Extract] -> [Chunk by section] -> [Embed (ONNX)] + [BM25] -> [index/ committed]

ONLINE:  [Question] -> [Input guard + PII scrub + rate limit]
                     -> [BM25 search] + [Vector search]
                     -> [RRF fusion] -> (optional re-rank)
                     -> [Prompt with S1..S6 + rules] -> [LLM]
                     -> [Output guard: citation check + PII] -> [Answer + sources]
```

---

## Step 4: Deep dive (10 min)

### 4a. Chunking
- By section, because legal text is organised in numbered clauses. Keep section IDs for citations.

### 4b. Hybrid retrieval
- **BM25** for exact terms ("Tier 1", "72 hours").
- **Vectors** for paraphrase. MiniLM through ONNX, cosine via one matmul.
- **RRF:** `score(d) = Σ 1/(k + rank_i(d))` with k=60. Uses ranks, because the two score scales aren't comparable.

### 4c. Generation
- Passages labelled [S1]..[S6]. Rules: answer only from them, cite inline, never invent a marker, emit a refusal marker if unsupported.
- Providers: groq, gemini, openai, stub.

### 4d. Guardrails
- **In:** length ≤600, injection heuristics, PII (BVN, NIN, NUBAN, +234 phones, cards, emails) scrubbed.
- **Out:** citations to unretrieved sources stripped and the claim labelled unverified. PII scrubbed again.

### 4e. Evaluation
- Retrieval: recall@k, hit@k, MRR against hand-labelled sections.
- Generation: groundedness, citation validity, key-fact recall, correct/false refusals.
- A stub model and lexical judge in CI, and a real provider on demand.

### 4f. Memory
- No PyTorch (it saves ~400 MB). ONNX runtime. The index is ~6 MB per 4,000 chunks.

---

## Step 5: Bottlenecks (2 min)

1. **Cold start on a free tier:** 30–45 s. Communicated in the UI.
2. **LLM rate limits:** handled by per-visitor and daily caps.
3. **Off-topic detection:** no retrieval-score cutoff works (the distributions overlap), so it depends on the LLM.
4. **Stale regulation:** the index needs rebuilding when rules change.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| ONNX MiniLM | PyTorch | 2 GB RAM | Export step |
| numpy array | Vector DB | Tiny corpus | Won't scale forever |
| RRF | Score sum | Scores not comparable | Loses score magnitude |
| No score cutoff | Cheap refusal | Cutoff rejects 26% of fair questions | Relies on the LLM |
| Stub in CI | Real LLM in CI | No key, free, deterministic | Generation metrics are a floor |
| Committed index | Build on deploy | Fast, reproducible starts | Must rebuild after doc changes |
