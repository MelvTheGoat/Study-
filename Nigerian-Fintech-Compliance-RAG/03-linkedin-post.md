# Nigerian-Fintech-Compliance-RAG: LinkedIn Post

*About 160 words. Copy from the line below.*

---

"How fast do we have to report a data breach?" In Nigerian fintech, that has two right answers: one deadline for the NDPC, another for the CBN.

I built a small assistant that answers questions like that from the regulations themselves, cites every claim, and prints the exact passage underneath so you can check it in seconds.

It uses hybrid search: keyword search for exact terms like "Tier 1" or "72 hours", plus meaning-based search, merged by rank rather than score. Invented citations get stripped and flagged.

Two things surprised me:
1. You can't cheaply spot off-topic questions. "Data protection rules in Kenya?" scored higher than 40 of 50 real questions. A cutoff catching 90% of them rejected 1 in 4 good ones.
2. The semantic model added less than expected. Keyword search did most of the work.

It fits in ~225 MB of RAM by running the embedding model on ONNX instead of PyTorch.

Note: it currently ships sample summaries, not the official texts.

https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

#RAG #LLM #Fintech #Nigeria
