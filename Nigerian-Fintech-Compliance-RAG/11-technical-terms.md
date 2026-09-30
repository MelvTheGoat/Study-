# Nigerian Fintech Rules Q&A: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow a question's journey: the rules themselves, preparing the documents, searching, writing the answer, keeping it safe, and measuring it.

---

## 1. The Rules Being Searched

### CBN, NDPA and NDPC
**What it means:** the CBN is the Central Bank of Nigeria. The NDPA is the Nigeria Data Protection Act 2023. The NDPC is the Nigeria Data Protection Commission, which enforces it.

**Why it's needed here:** they're the sources of the rules the app answers questions about. Some questions have two correct answers, one from each regulator.

### Circular
**What it means:** an official notice from a regulator, like the CBN, setting or changing a rule.

**Why it's needed here:** much of Nigerian fintech regulation lives in circulars, spread across many PDFs.

### KYC (know your customer)
**What it means:** checks a financial company must do to confirm who its customers are. In Nigeria, accounts come in tiers with different limits.

**Why it's needed here:** it's a common compliance question. The sample documents use older (2013) tier limits, which is one reason they aren't for real use.

### Authoritative source
**What it means:** the official, original text of a rule, as opposed to a summary.

**Why it's needed here:** each document is marked authoritative or not. The current six are summaries, so the app shows a warning.

---

## 2. Preparing the Documents

### Corpus
**What it means:** the full collection of documents the app searches.

**Why it's needed here:** answers can only come from it. Right now it's six summary documents.

### Text extraction
**What it means:** pulling plain text out of files like PDFs.

**Why it's needed here:** PDFs aren't plain text. Extraction also keeps section numbers and titles, which citations need.

### Chunk and chunking
**What it means:** a chunk is a small piece of a document. Chunking is how documents are split.

**Why it's needed here:** the LLM can only read a limited amount, and precise citations need small pieces. Here, chunks follow sections, up to 1,200 characters, giving 100 chunks.

### Overlap
**What it means:** repeating a little text from the end of one chunk at the start of the next.

**Why it's needed here:** when a long section is split, 180 characters of overlap stop a sentence from being cut in half and lost.

### Index
**What it means:** a prepared structure that makes searching fast, like the index at the back of a book.

**Why it's needed here:** the index is built once, ahead of time, and saved with the project. So the app starts fast and never rebuilds it when deployed.

### Manifest
**What it means:** a small file describing how something was built: when, and with which settings.

**Why it's needed here:** it records how the index was made, so results can be reproduced.

---

## 3. Searching

### Retrieval
**What it means:** finding the passages most likely to answer a question.

**Why it's needed here:** it's the "R" in RAG. If retrieval misses the right passage, the answer can't be right.

### Keyword search (BM25)
**What it means:** ranking passages by shared words. BM25 is a standard formula that also gives rare words more weight.

**Why it's needed here:** rules use exact terms like "Tier 1", "72 hours" and "N5,000,000". Keyword search catches these very well. On this corpus, it does most of the work.

### Embedding
**What it means:** a list of numbers that captures what a piece of text means.

**Why it's needed here:** it lets the app find passages with the same meaning, even when the words differ.

### Embedding model (MiniLM)
**What it means:** a program that turns text into embeddings. MiniLM is a small, free one.

**Why it's needed here:** it's light enough to run on a small server. Each embedding is a list of 384 numbers.

### ONNX
**What it means:** a common file format for AI models, which a light tool (ONNX Runtime) can run.

**Why it's needed here:** running MiniLM the usual way needs a big toolkit using hundreds of megabytes of memory. ONNX gives the same results using far less, so the app fits in 2 GB.

### Vector search (cosine similarity)
**What it means:** comparing embeddings to find the closest meanings. Cosine similarity measures how closely two embeddings point in the same direction.

**Why it's needed here:** it's the "meaning search". With only 100 chunks, it's done in a single quick calculation, so no special database is needed.

### Hybrid search
**What it means:** using keyword search and meaning search together.

**Why it's needed here:** each catches things the other misses. Keywords catch exact terms, and meaning catches rephrased questions.

### Reciprocal Rank Fusion (RRF)
**What it means:** a way to merge rankings by position rather than score. A passage near the top of both lists wins.

**Why it's needed here:** keyword and meaning scores are on different scales, so they can't be added directly. Positions can be combined fairly.

### Re-ranker (cross-encoder)
**What it means:** a slower, more careful model that re-orders a short list of results by reading the question and each passage together.

**Why it's needed here:** it might improve the order. It's switched off by default, and whether it helps hasn't been measured.

### Fallback embedder
**What it means:** a simple backup way of making embeddings, based on word fingerprints rather than meaning.

**Why it's needed here:** tests can run without the real model. It's also used to measure how much the real model adds, which turned out to be a little.

---

## 4. Writing the Answer

### LLM (large language model)
**What it means:** an AI that reads and writes text.

**Why it's needed here:** it turns the found passages into a clear, plain-English answer.

### RAG (retrieval-augmented generation)
**What it means:** find the right passages first, then have the LLM write an answer using only them.

**Why it's needed here:** an LLM alone might make up rules. Giving it the passages keeps it grounded, like an open-book exam.

### Prompt
**What it means:** the instructions and text sent to the LLM.

**Why it's needed here:** the prompt gives the 6 passages labelled S1 to S6, and strict rules: answer only from them, cite each claim, and never invent labels.

### Citation
**What it means:** a label linking a claim to its source passage, like [S2].

**Why it's needed here:** users can check every claim against the exact passage shown underneath.

### Refusal marker
**What it means:** a special signal the LLM must use when the passages don't answer the question.

**Why it's needed here:** refusing is safer than guessing about legal rules. A fixed signal makes refusals easy to detect and count.

### Provider
**What it means:** the company whose AI service writes the answer.

**Why it's needed here:** the app can use Groq (free tier), Gemini or OpenAI. You pick one with a setting.

### Stub
**What it means:** a pretend LLM that just quotes the passages, with no real AI involved.

**Why it's needed here:** tests and automatic checks can run for free, with no key. The downside is that answer-quality numbers from the stub don't say much about a real LLM.

---

## 5. Keeping It Safe

### Guardrail
**What it means:** a safety check placed before or after the AI.

**Why it's needed here:** there are two. The input guard checks questions, and the output guard checks answers.

### Prompt injection
**What it means:** a trick where a user hides instructions in their question, like "ignore your rules and…".

**Why it's needed here:** the input guard looks for common tricks like this before anything reaches the LLM.

### PII scrubbing
**What it means:** removing personal details from text. PII means personally identifiable information.

**Why it's needed here:** users might paste a BVN, NIN, bank account number (NUBAN), phone number, card number or email. These are removed before being sent to any outside service, and again from the answer.

### BVN, NIN and NUBAN
**What it means:** the Bank Verification Number, the National Identification Number, and the standard 10-digit Nigerian bank account number.

**Why it's needed here:** they're the Nigerian ID numbers the scrubber looks for.

### Citation verification
**What it means:** checking that every citation points to a passage that was actually given to the LLM.

**Why it's needed here:** an LLM might invent "[S9]". The output guard removes it and labels the claim "unverified", so it's visible instead of hidden.

### Rate limiting
**What it means:** capping how many requests someone can make.

**Why it's needed here:** it protects the free AI quota. Each visitor gets 8 questions per 5 minutes, and there are 300 a day overall.

---

## 6. Measuring It

### Golden set
**What it means:** a fixed list of test questions with known correct answers.

**Why it's needed here:** there are 60: 50 answerable (with the right sections labelled) and 10 that should be refused.

### Recall@k
**What it means:** how often the right passage appears in the top k results.

**Why it's needed here:** it shows whether search is working. Recall@5 is 96%.

### MRR (mean reciprocal rank)
**What it means:** a score for how high the first right passage appears. 1 means always first, and 0.5 means usually second.

**Why it's needed here:** it's 0.84 here, meaning the right passage is usually at or near the top.

### Groundedness
**What it means:** whether the answer only says things found in the passages.

**Why it's needed here:** it's the key quality check for RAG. With the quoting stub it's automatically perfect, so a real LLM still needs to be measured.

### False refusal
**What it means:** refusing a question that could have been answered.

**Why it's needed here:** refusing too much makes the app useless. It's 6% here.

### Ablation
**What it means:** removing or swapping one part to see how much it helps.

**Why it's needed here:** swapping MiniLM for the simple fallback only dropped MRR from 0.84 to 0.82. That shows keyword search does most of the work on this corpus.
