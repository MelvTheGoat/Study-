# Nigerian-Fintech-Compliance-RAG: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

---

## Summary

A hybrid-retrieval RAG app (Streamlit, ~3,800 lines) over Nigerian fintech regulation: section-based chunking, BM25 plus ONNX MiniLM embeddings fused with RRF (k=60), an optional cross-encoder re-ranker, a strict citation prompt with a refusal marker, input/output guardrails (injection heuristics, PII scrubbing, citation verification), rate limits, and an eval harness that scores retrieval and generation separately. **110 tests pass (~1 s). I re-ran the eval and the retrieval numbers match.** The corpus is 6 sample summaries (100 chunks), not authoritative texts. The main build landed in one commit on 8 Aug 2026, and later commits are docs, UI and smoke tests.

## Architecture

```
app.py                      Streamlit UI
src/config.py               env settings
src/ingest/                 extract (pypdf), chunking (section, 1200/180), schema
src/retrieval/embedder.py   ONNX MiniLM (+ hashing fallback)
src/retrieval/store.py      numpy vectors, cosine via matmul
src/retrieval/bm25.py       BM25
src/retrieval/fusion.py     reciprocal_rank_fusion(k=60, weights)
src/retrieval/rerank.py     optional ONNX cross-encoder
src/retrieval/pipeline.py   orchestrates search
src/generation/prompts.py   S1..Sn markers, rules, refusal marker
src/generation/providers.py groq | gemini | openai | stub
src/generation/answer.py    generate + post-process
src/guardrails/             input_guard, output_guard, pii
src/limits.py               per-session + daily caps (persisted)
src/feedback.py             thumbs up/down
eval/                       golden_set.yaml (60), metrics, judge, run_eval
scripts/                    build_index, fetch_corpus, export_reranker, check_memory
index/                      committed vectors.npy, bm25.json, chunks.jsonl, manifest.json
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| ONNX, not PyTorch | MiniLM via onnxruntime + tokenizers | ~152 MB vs ~600 MB. Fits 2 GB. |
| numpy "vector store" | Exact cosine by matmul | Tiny corpus. Exact beats approximate. |
| Section chunking | ≤1,200 chars, 180 overlap | Clause-level citations |
| RRF | `Σ w/(k+rank)`, k=60, deterministic tie-break | BM25 and cosine scales aren't comparable |
| Refusal marker | Specific token in the prompt contract | Detectable, measurable refusal |
| Citation verification | Drop citations to unretrieved markers, label unverified | No silent fabrication |
| PII scrub in and out | BVN, NIN, NUBAN, +234, cards, emails | Nothing sensitive reaches a third party |
| No retrieval-score refusal gate | Measured overlap | A gate rejects too many fair questions |
| Committed index + manifest | Build date, settings | Reproducible, fast cold starts |
| Stub provider + lexical judge in CI | No key needed | Deterministic, free CI |

## Evaluation (reproduced)

**Retrieval** (50 labelled questions): recall@1 0.710, recall@3 0.850, recall@5 0.960, recall@10 0.980, hit@1 0.760, hit@5 0.980, hit@10 1.000, MRR 0.842. No question missed a relevant passage in the top 10.

**Ablation:** swapping MiniLM for a hashing embedder gives MRR 0.820 and recall@5 0.900 (from the README).

**Generation** (stub + lexical judge): groundedness 1.000, citation validity 1.000 (tautological for a quoting stub), key-fact recall 0.726, correct refusals 0.400, false refusals 0.060. The failures listed by the harness include missing key facts (e.g. KYC tier limits), a false refusal on the "CISO" question, and 6 off-topic questions answered anyway by the stub.

**Refusal gate study:** on-topic max-score range 0.349–0.785 vs off-topic 0.423–0.670. Cutoff 0.50 catches 40% of off-topic and rejects 6% of fair questions. 0.55 catches 70% and rejects 20%. 0.65 catches 90% and rejects 26%.

**Memory:** peak ~225 MB, or ~367 MB with the re-ranker (`check_memory.py`).

**Not measured:** answer quality with a real LLM, the re-ranker's effect on quality, and a semantic (non-lexical) judge.

## Known weaknesses

- Non-authoritative, partly stale sample corpus.
- Generation metrics are floors or tautologies under the stub.
- Refusal depends entirely on the LLM's behaviour.
- The lexical groundedness judge penalises faithful paraphrase.
- Abbreviation mismatch ("CISO" vs "Chief Information Security Officer").
- Regex-based PII detection.
- Per-container rate limits, and ephemeral counters on a free tier.
- A small golden set (60), written by the author.
