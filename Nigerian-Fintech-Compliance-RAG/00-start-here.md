# Nigerian Fintech Rules Q&A: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: the app currently holds **six written summaries** of the rules, not the official documents. Some details in them are out of date. So it's a working demo, not a legal tool yet.

## 1. The Problem

Nigerian fintech companies must follow many rules. These come from the Central Bank of Nigeria (CBN), the Nigeria Data Protection Act 2023, and the data protection commission (NDPC).

The rules are spread across many PDF files on different websites. They refer to each other and change quietly. Even a simple question like "How fast must we report a data breach?" can have two correct answers, one for each regulator.

## 2. The Big Idea

This app answers questions about these rules in plain English. But it only answers using its stored documents. It never makes up rules.

Every claim in an answer has a label, like [S1], showing exactly which passage it came from. The full passage is shown underneath. If the documents don't cover a question, the app refuses to answer.

This approach is called RAG, short for retrieval-augmented generation. In plain words: first *find* the right passages, then let an AI *write* the answer using only those passages. It's like an open-book exam.

## 3. How It Works, Step by Step

**Before any questions (done once):**

**Step 1: Read the documents.** Text is pulled out of the files, keeping the section numbers and titles.

**Step 2: Cut them into chunks.** Each chunk is usually one section of a rule, up to 1,200 characters. Long sections are split into pieces that overlap a little, so no sentence is lost at the edge. The six documents become 100 chunks.

**Step 3: Build two search indexes.** One is for keyword search. The other is for meaning search, using embeddings (lists of numbers that capture what text means). Both are saved with the project, so the app starts fast.

**When someone asks a question:**

**Step 4: Safety check on the way in.** The app checks the question's length and looks for tricks, like "ignore your rules". It removes personal details, like BVNs, NINs, account numbers, phone numbers and emails, before anything is sent to an outside AI service. It also checks usage limits.

**Step 5: Search two ways.** Keyword search finds chunks with the same words. Meaning search finds chunks with a similar meaning, even with different words.

**Step 6: Merge the results.** The two searches score differently, so their scores can't be added. Instead their *positions* are combined: a chunk ranked high in both wins. The top 6 chunks are labelled S1 to S6.

**Step 7: The AI writes the answer.** It's told: only use these 6 passages, cite each claim, and use a special "refuse" signal if they don't cover the question.

**Step 8: Safety check on the way out.** If the AI cites a passage it wasn't given, like [S9], that citation is removed and the claim is marked "unverified". Personal details are removed again.

**Step 9: Show the answer.** The user sees the answer, then every cited passage in full, with a warning if the document is only a summary.

## 4. The Clever Parts

**Two kinds of search together.** Rules use exact terms like "Tier 1", "72 hours" and "N5,000,000", so keyword search is great at those. Meaning search catches questions asked in different words. Together they're stronger than either alone.

**Tiny memory use.** The app must fit in 2 GB of memory on a small free server. The meaning model is run with a light tool (ONNX Runtime) instead of the usual heavy one. It uses about 225 MB at its peak.

**No fancy database needed.** With only 100 chunks, the meaning search is one quick calculation. A special search database isn't needed.

**Invented citations are caught.** AI can make up sources. Here, any fake citation is removed and the claim is clearly marked.

**Free to run and test.** The app can use a free AI service, like Groq. For testing, it uses a "stub" that just quotes passages, so no key or money is needed.

## 5. The Important Words

- **LLM (large language model)**: an AI that reads and writes text.
- **RAG**: find the right passages first, then write the answer using only them.
- **Chunk**: a small piece of a document, usually one section.
- **Embedding**: a list of numbers that captures what text means.
- **Keyword search (BM25)**: finding passages that share the question's words.
- **Rank fusion**: merging two search results by their positions.
- **Citation**: a label linking a claim to its source passage.
- **Refusal**: the app saying "my documents don't cover this" instead of guessing.
- **PII**: personal details that identify someone, like a phone number or BVN.
- **Guardrail**: a safety check before or after the AI.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **Streamlit**: builds the simple web page.
- **pypdf**: reads text from PDF files.
- **ONNX Runtime and MiniLM**: run the meaning search with very little memory.
- **NumPy**: does the meaning-search maths.
- **Home-made BM25**: does the keyword search.
- **Groq, Gemini or OpenAI**: the AI service that writes the answer.
- **pytest, ruff and mypy**: keep the code correct.
- **Docker and Google Cloud Run**: package it and run it online.

## 7. How Good Is It?

The project tests itself with 60 questions: 50 it should answer, and 10 it should refuse.

**Search works well.** The right passage was in the top 5 results 96% of the time. No question missed the right passage in the top 10.

**Answer quality with a real AI hasn't been measured yet.** The saved results use the quoting stub, which makes some scores look perfect automatically. The stub only refused 4 of the 10 off-topic questions, but a real AI may do better.

An interesting finding: the meaning search added less than expected. Keyword search did most of the work on these documents.

## 8. What's Weak or Missing

- The documents are summaries, not the official rules, and some are out of date.
- Answer quality with a real AI hasn't been measured.
- Off-topic questions can't be filtered by search scores, because fair and off-topic scores overlap too much. So refusing is left to the AI.
- Short forms like "CISO" may not match "Chief Information Security Officer" in the text.
- Personal details are found with text patterns, which can miss unusual formats.

## 9. What This Project Shows You Can Do

- Build a RAG app that cites every claim.
- Combine keyword and meaning search.
- Protect personal data before it reaches outside services.
- Fit an AI app into a small, free server.
- Test search and answers separately, and report honestly.

## 10. Ten Things to Remember

1. It answers questions about Nigerian fintech rules in plain English.
2. It only answers from its documents, and refuses otherwise.
3. Every claim is cited, and the full passage is shown.
4. The documents are six summaries for now, not the official rules.
5. Documents are cut into 100 chunks, mostly one section each.
6. It searches by keyword and by meaning, then merges the results.
7. Personal details are removed before anything leaves the app.
8. Fake citations are removed and marked "unverified".
9. It fits easily in 2 GB of memory.
10. Search finds the right passage in the top 5 results 96% of the time.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
