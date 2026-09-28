# LLM RAG System: The Models Behind the Search

RAG (Retrieval-Augmented Generation) means an AI chatbot first looks up helpful text, then writes its answer using that text. A company help bot that answers from its own manuals is a common example. This file is about the two models that make the "look up" part smart, and how to make them better over time.

*How to split documents, store them and show sources is covered in [Artificial Intelligence/01-rag-system.md](../Artificial%20Intelligence/01-rag-system.md). Here we focus only on the models.*

## Key Terms

- **LLM (large language model)**: an AI that reads text and writes text, like ChatGPT.
- **Model**: a program that learned patterns from examples.
- **Passage**: a short piece of text from a document, a few paragraphs long.
- **Embedding model**: a model that turns text into a list of numbers, so that texts with similar meaning get similar numbers.
- **Vector search**: finding the passages whose numbers are closest to the question's numbers.
- **Re-ranker**: a second model that carefully re-scores a shortlist of passages.
- **Recall@50**: how often the right passage is somewhere in the top 50 results.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The LLM can only answer well if it's given the right passages. So these models have one job: put the most helpful passages in front of the LLM.

**Step 2: Figure out the data.** To measure and improve the models, we need examples of questions paired with the passage that truly answers them. These can come from experts, from support logs, or from users' thumbs-up and thumbs-down.

**Step 3: Sketch the two-model setup.** A fast embedding model finds a shortlist, then a slower, smarter re-ranker picks the best few. This "fast then careful" pattern appears all over machine learning.

**Step 4: Follow one question.** Trace a question to the final 5 passages, noting how long each model takes.

**Step 5: Measure each model separately.** Check the embedding model and the re-ranker on their own. If answers are bad, you then know which model to fix.

**Step 6: Improve over time.** Collect the cases where search failed and use them to train better models. Plan for what it costs to swap in a new model.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design the search models for a company help bot.

- **1 million** passages from company documents.
- **10** questions per second.
- Give the LLM the best **5** passages within **0.5 seconds**.

### The Big Picture (Step 3)

```
   [User question]
          |
          v
   [Embedding Model]     -- question -> list of numbers
          |
          v
   [Vector Search]       -- 1,000,000 passages -> top 50
          |
          v
   [Re-ranker Model]     -- 50 passages -> best 5
          |
          v
   [LLM writes the answer]
          |
          v
   [Feedback Logs] -----> [Training Data] -> improve both models
```

Step by step:

1. A user asks, "How do I reset my password?"
2. The Embedding Model turns the question into a list of numbers.
3. Vector Search compares those numbers with the numbers for all 1 million passages, which were prepared earlier, and returns the closest 50.
4. The Re-ranker reads the question together with each of the 50 passages and keeps the best 5.
5. The LLM writes an answer from those 5 passages.
6. Feedback (like thumbs-down) is saved and later turned into training data to improve both models.

### The Main Parts (Step 3)

**The Embedding Model.** It turns text into numbers so that meaning becomes distance. Imagine a giant map where every passage is a pin: passages about passwords cluster in one corner, passages about refunds in another. A question gets its own pin, and we grab the nearest ones.

The magic is that "reset my password" can land near "change your login details", even with no shared words.

**The Re-ranker.** The embedding model reads the question and each passage *separately*, which is fast but a bit shallow. The re-ranker reads them *together*, word by word, which is slower but much sharper.

Think of it like hiring. The embedding model is a quick CV scan that cuts 1,000 applicants to 50. The re-ranker is the proper interview for those 50. You'd never interview all 1,000.

### How It Answers a Request (Step 4)

The 0.5-second budget, step by step:

1. Embed the question: about **20 ms** (milliseconds, thousandths of a second).
2. Vector search over 1 million passages: about **30 ms**.
3. Re-rank 50 passages. If each took 10 ms one at a time, that's 50 × 10 = **500 ms**, far too slow. By scoring all 50 together in one batch (all at once), it takes about **100 ms**.
4. Total: 20 + 30 + 100 = **150 ms**, well inside 500 ms.

The lesson: the re-ranker is the slow part, so we only give it a shortlist.

### How the Models Improve (Step 6)

**Start with a ready-made model.** Many good models are free to download. Start there and measure before changing anything.

**Collect training pairs.** Every time a user marks an answer as helpful, you get a (question, good passage) pair. Every thumbs-down hints at a bad passage. Experts can also write pairs for common questions.

**Teach with "tricky wrong answers".** Training works best when you show the model near-misses: a passage about "reset *email* settings" for a *password* question. It's like a teacher who tests you on the confusing cases, not the easy ones.

**Fine-tune.** Fine-tuning means giving an existing model extra training on your own examples, so it learns your company's words and product names. It's like an experienced chef learning your restaurant's menu: they already know how to cook, and just need your recipes.

Training the re-ranker to copy a much larger model's judgments is another option (advanced - skip for now).

### How We Know It's Working (Step 5)

Use a test set of, say, **200** questions with known correct passages.

- **Recall@50 (embedding model)**: if the right passage is in the top 50 for 180 of 200 questions, recall@50 = 180 ÷ 200 = **90%**. If it's missing here, the re-ranker never even sees it.
- **Hit rate@5 (re-ranker)**: how often the right passage survives into the final 5. For example, 160 ÷ 200 = **80%**.
- **Speed**: time taken by each model. The re-ranker usually dominates.

### What Can Go Wrong (Step 6)

- **The model doesn't know your jargon.** A general model may not know that "SKU" means "product code". Fine-tune it on your own question and passage pairs.
- **Changing the embedding model means redoing everything.** New numbers don't match old numbers, so all 1 million passages must be converted again. Plan and test this before switching.
- **The re-ranker is too slow.** Give it fewer passages (30 instead of 50), or run it in batches.
- **Good test scores, bad real answers.** Your test questions may not match what users really ask, so refresh them with real questions regularly.

## Quick Recap

- RAG search usually uses two models: a fast embedding model (shortlist), then a careful re-ranker (best few).
- Embeddings turn meaning into distance, so similar ideas sit close together.
- Measure each model on its own: recall@50 for the embedding model, hit rate@5 for the re-ranker.
- Improve with real question-passage pairs and tricky near-miss examples.
- Swapping embedding models means re-converting every passage, so plan for it.
