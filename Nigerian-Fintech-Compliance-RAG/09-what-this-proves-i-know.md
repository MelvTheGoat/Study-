# Nigerian-Fintech-Compliance-RAG: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

---

## 1. Retrieval-Augmented Generation (RAG)

**Simple explanation:** find relevant text first, then ask an LLM to answer using only that text.

**In this project:** section chunking, hybrid search, top-6 passages, a strict citation prompt, citation verification.

**Also be ready to explain:** why RAG reduces hallucination (but doesn't remove it), context window limits, chunk size trade-offs, query rewriting, and RAG vs fine-tuning.

---

## 2. Information retrieval (BM25, embeddings, fusion)

**Simple explanation:** BM25 scores documents by matching words (rare words count more). Embeddings compare meaning. Fusion combines them.

**In this project:** custom BM25, MiniLM via ONNX, RRF (k=60).

**Also be ready to explain:** TF-IDF vs BM25 (k1, b parameters), dense vs sparse retrieval, cosine similarity, cross-encoder re-ranking vs bi-encoder retrieval, and ANN indexes (HNSW, IVF) and when you need them.

---

## 3. Retrieval evaluation metrics

**Simple explanation:** did the right passage show up, and how high?

**In this project:** recall@k, hit@k, MRR against hand-labelled sections.

**Also be ready to explain:** precision@k, NDCG, why strict recall differs from hit rate with multiple relevant passages, and building a golden set.

---

## 4. LLM evaluation and guardrails

**Simple explanation:** check that answers are grounded, correctly cited, and refuse when they should.

**In this project:** groundedness, citation validity, key-fact recall, correct and false refusals, the stub vs real provider, a lexical judge.

**Also be ready to explain:** LLM-as-judge and its biases, faithfulness vs answer relevance, why stubs make some metrics tautological, and refusal calibration.

---

## 5. Efficient model serving on small machines

**Simple explanation:** run ML models cheaply by choosing lighter runtimes.

**In this project:** ONNX runtime instead of PyTorch, ~225 MB peak, measured with `check_memory.py`.

**Also be ready to explain:** ONNX export, quantisation, cold starts on serverless/free tiers, and memory vs latency.

---

## 6. Privacy and security for LLM apps

**Simple explanation:** don't send personal data to third parties, and don't let users hijack the model.

**In this project:** PII regexes (BVN, NIN, NUBAN, +234, cards, emails) in and out, injection heuristics, rate limits.

**Also be ready to explain:** the OWASP LLM Top 10 (prompt injection, data leakage), NDPA 2023 basics, and pattern vs model-based PII detection.

---

## 7. Nigerian fintech regulation (domain)

**Simple explanation:** the rules Nigerian payment companies follow.

**In this project:** the NDPA 2023, NDPC GAID, CBN tiered KYC, AML/CFT/CPF, the cybersecurity framework for PSPs, and licence categories (PSB, MMO, etc.).

**Also be ready to explain:** tiered KYC (Tier 1–3 limits and ID requirements), BVN/NIN, the breach notification duty (to the regulator vs to data subjects), and licence types and capital. Use the official texts, not the sample summaries.

---

## 8. Experiment-driven engineering (negative results)

**Simple explanation:** test an idea before shipping it, and keep the result even when it's "no".

**In this project:** the failed refusal cutoff, and the embedder ablation showing keyword search does most of the work.

**Also be ready to explain:** ablation studies, why reporting negative results builds trust, and threshold trade-off tables.

---

## 9. Building and deploying a small ML app

**In this project:** Streamlit, a committed index, Docker, Cloud Run notes, CI on 3.10–3.12 with a retrieval-regression gate, and mypy/ruff.

**Also be ready to explain:** reproducible builds, CI quality gates, and free-tier constraints (sleep, ephemeral disk).
