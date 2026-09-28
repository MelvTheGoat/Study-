# Nigerian-Fintech-Compliance-RAG: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

1. **A RAG assistant for Nigerian fintech rules (CBN + NDPA/NDPC) that answers only from the documents, cites every claim, and shows the passage.**
   *Why it matters:* trust through visible sources.

2. **The corpus is 6 sample summaries (100 chunks), not the official texts. Some figures are stale.**
   *Why it matters:* say this before anyone asks.

3. **Hybrid retrieval: BM25 + MiniLM embeddings (384-dim, via ONNX), merged by RRF with k=60.**
   *Why it matters:* exact terms plus meaning, fused fairly.

4. **No PyTorch and no vector DB. Peak memory ~225 MB on a 2 GB box (367 MB with the re-ranker).**
   *Why it matters:* engineering for real constraints.

5. **The prompt contract: answer only from S1–S6, cite inline, never invent markers, emit a refusal marker if unsupported.**
   *Why it matters:* makes grounding and refusal measurable.

6. **Guardrails: length cap, injection heuristics, and a PII scrub (BVN, NIN, NUBAN, +234, cards, emails) in and out. Invented citations stripped and flagged.**
   *Why it matters:* privacy and honesty.

7. **Retrieval eval (50 labelled questions): recall@5 0.960, hit@10 1.000, MRR 0.842. Reproducible without a key.**
   *Why it matters:* the strongest measured number.

8. **Generation eval uses a stub + lexical judge: groundedness 1.0 (tautological), key facts 0.726, refusals 0.400. Real-LLM quality is not measured.**
   *Why it matters:* know what's proven and what isn't.

9. **Negative result: no retrieval-score cutoff separates off-topic questions. 90% caught = 26% of fair questions rejected.**
   *Why it matters:* a great interview story about measuring before optimising.

10. **Ablation: a hashing embedder gets MRR 0.820 vs 0.842, so keyword search does most of the work. 110 tests pass.**
    *Why it matters:* you understand *why* the system works.
