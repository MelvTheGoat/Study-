# Nigerian-Fintech-Compliance-RAG

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

## In short

A RAG assistant for Nigerian fintech regulation (CBN, NDPA 2023, NDPC). It answers in plain English only from indexed documents, cites every claim, prints the source passage, scrubs personal data, and refuses when the documents don't cover the question. It uses hybrid BM25 + embedding search merged with reciprocal rank fusion, and fits in ~225 MB of RAM.

## Key facts

| | |
|---|---|
| Language | Python |
| Core tech | Streamlit, onnxruntime (MiniLM), BM25, RRF, numpy, Groq/Gemini/OpenAI or a stub |
| Retrieval (reproduced) | recall@5 0.960, MRR 0.842 on 50 labelled questions |
| Generation | Only measured with a stub. Real-LLM quality not measured yet. |
| Corpus | 6 sample summaries, not the official texts |
| Tests | 110 passing |

## Files

| File | What's in it |
|---|---|
| [01-system-design.md](01-system-design.md) | Parts, diagram, stack, data flow, trade-offs, 10x |
| [02-how-to-write-the-system-design.md](02-how-to-write-the-system-design.md) | Whiteboard steps |
| [03-linkedin-post.md](03-linkedin-post.md) | LinkedIn post |
| [04-blog-post.md](04-blog-post.md) | Blog post |
| [05-explain-to-non-technical.md](05-explain-to-non-technical.md) | Plain-language explanation |
| [06-explain-to-technical.md](06-explain-to-technical.md) | Engineer-level explanation |
| [07-defend-in-interview.md](07-defend-in-interview.md) | Pitch, questions, weak spots |
| [08-ten-points-to-know.md](08-ten-points-to-know.md) | 10 key facts |
| [09-what-this-proves-i-know.md](09-what-this-proves-i-know.md) | Skills and related topics |
