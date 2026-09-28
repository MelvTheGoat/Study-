# RAG System: Building the App

RAG (Retrieval-Augmented Generation) is a way to make an AI chatbot answer from *your* documents instead of from memory. Think of a company help bot that answers "How many holiday days do I get?" by reading the staff handbook and pointing to the exact page. It matters because it cuts down on made-up answers and lets people check where each answer came from.

*This file is about building the app. The search models (programs that learned patterns from examples) and how to improve them are in [Machine Learning/05-llm-rag-system.md](../Machine%20Learning/05-llm-rag-system.md).*

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Chunk**: a small piece of a document, a few paragraphs long.
- **Chunking**: splitting documents into chunks.
- **Embedding**: a list of numbers that captures a chunk's meaning, so similar chunks get similar numbers.
- **Vector database**: a database that stores embeddings and quickly finds the ones closest to a question.
- **Prompt**: the full message we send to the LLM, including instructions, the question and the chunks.
- **Hallucination**: when an AI confidently makes something up.
- **Citation**: a link showing which document a sentence came from.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide who will ask questions and what documents should answer them. Also decide what should happen when the documents *don't* have the answer. "I don't know" is often the right reply.

**Step 2: Figure out the data.** Collect the documents: PDFs, web pages, wiki pages. Check how often they change and who is allowed to see them.

**Step 3: Sketch the two halves.** A RAG app has a *preparation* half (get documents ready ahead of time) and an *answering* half (respond to a question). Drawing them separately makes the design much clearer.

**Step 4: Walk through one question.** Follow a question from the user to the answer with sources. Check what the LLM sees at each moment.

**Step 5: Decide how to know it works.** Check whether answers are correct and backed by the documents. A small set of test questions with known answers goes a long way.

**Step 6: Plan to keep it fresh and safe.** Documents change, and not everyone may see every document. Plan updates and permissions early.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's build a help bot for a company's internal documents.

- **10,000** documents, about **10 pages** each.
- **100** questions per minute at busy times.
- An answer, with sources, in about **5 seconds**.

How many chunks will we store? Step by step:

1. 10,000 documents × 10 pages = 100,000 pages.
2. Split each page into about 3 chunks.
3. 100,000 × 3 = **300,000 chunks**.

### The Big Picture (Step 3)

```
  PREPARATION (done ahead of time)
  [Documents] -> [Split into chunks] -> [Embed + store in Vector Database]

  ANSWERING (every question)
  [User question]
        |
        v
  [Find top 5 chunks] <---- (Vector Database)
        |
        v
  [Build prompt: rules + question + chunks]
        |
        v
  [LLM writes answer with citations]
```

Step by step:

1. **Ahead of time**, we split every document into chunks, turn each chunk into an embedding, and store it in the Vector Database.
2. A user asks, "How many holiday days do new staff get?"
3. The app finds the 5 chunks most related to the question.
4. It builds a prompt: instructions ("only use these chunks, cite them"), the question, and the 5 chunks.
5. The LLM writes the answer and adds a citation to each fact, like "[HR Handbook, page 4]".

### The Main Parts (Step 3)

**Loading documents.** First we pull text out of PDFs, web pages and so on. We keep useful labels with each document, like its title, date and who can see it. These labels are called metadata (extra facts about the data).

**Chunking.** Documents are too long to send to the LLM whole, so we cut them into chunks. It's like turning a textbook into a stack of index cards, each card about one idea.

Two simple tips:

- **Cut at natural breaks**, like headings or paragraphs, not in the middle of a sentence.
- **Overlap a little.** Let each chunk repeat the last sentence or two of the one before. That way an answer that sits on a boundary isn't cut in half.

**Storing.** Each chunk is turned into an embedding and saved in the Vector Database, together with its text and metadata. Think of it as a library where books with similar topics sit on the same shelf. pgvector (an add-on that lets PostgreSQL, a popular free database, store and search embeddings) is a common, simple choice.

**Finding the right chunks.** When a question arrives, it's turned into an embedding too, and the database returns the closest chunks. It's like a librarian quickly pulling the 5 most relevant index cards from the box.

Many apps also add plain keyword search for exact terms like product codes (advanced - skip for now).

**Building the prompt.** We give the LLM clear instructions, such as:

- "Answer only from the chunks below."
- "Add a citation after each fact."
- "If the chunks don't contain the answer, say you don't know."

It's like handing a student an open-book exam with the right pages bookmarked, plus the rule "show your sources".

**Answering with sources.** The LLM writes the answer, and the app turns citations into clickable links. The user can check the source in seconds, which builds trust.

### How We Know It's Working (Step 5)

Write **50 test questions** with known correct answers and the document that holds each one.

- **Correct answers**: if 45 of 50 answers are right, that's 45 ÷ 50 = **90%**.
- **Right document found**: how often the correct document is among the 5 chunks we fetched. If it's missing, the LLM can't possibly answer well.
- **Grounded answers**: how often every fact in the answer is supported by a cited chunk. This catches hallucinations.
- **User feedback**: a simple thumbs-up or thumbs-down under each answer.

### What Can Go Wrong (Step 6)

- **Hallucination.** The LLM adds facts not in the chunks. Use strict instructions, require citations, and allow "I don't know".
- **Out-of-date answers.** An old policy document still gets found. Re-process documents when they change, and prefer the newest version.
- **Wrong people seeing documents.** A junior employee shouldn't get answers from board papers. Store who can see each chunk, and only search chunks the user is allowed to see.
- **Answers cut in half.** Bad chunking splits a table or a list. Cut at headings and use a small overlap.

## Quick Recap

- RAG = find relevant chunks of your documents, then let the LLM answer from them.
- Prepare ahead of time: split documents into chunks, embed them, and store them with metadata.
- Tell the LLM to use only the given chunks, cite them, and say "I don't know" when needed.
- Test with real questions, and check both "did we find the right document?" and "is the answer correct?"
- Keep documents fresh, and respect who is allowed to see what.
