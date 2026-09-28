# Search (Job Hunt): LinkedIn Post

*About 160 words. Copy from the line below. Note: the repo is private, so the post doesn't link it.*

---

Job hunting from Nigeria for ML roles has a hidden filter: half the "great matches" can't hire you.

So I built a small tool that does the first pass for me.

It reads jobs every day from 405 company career boards and a set of free job boards, removes duplicates, and keeps only data, ML and AI roles.

Then it answers the question most job sites skip: can I actually get this one?
Each job gets a label: remote and open to Africa, in Nigeria, elsewhere in Africa, or abroad with a visa sponsor. To judge sponsors it checks the UK and Netherlands sponsor registers and US H-1B filing history, and it saves the exact sentence behind each label.

Finally it scores each job 0–100 against my CV with a small local embedding model. No paid APIs.

It never applies for me. I read every job and apply myself.

#JobSearch #Python #MachineLearning #NLP
