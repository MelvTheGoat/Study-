# Stocks: LinkedIn Post

*About 150 words. Copy from the line below.*

---

A language model can sound sure of itself and still invent a share price.

So I'm building a stock question-answering agent the other way round: the measuring stick first, the agent second.

The project will answer factual questions about US stocks and Nigerian stocks on the NGX. Models know a lot about Apple and very little about Nigerian Breweries. That gap shows how much of a model's "skill" is memory.

What's built so far is the plumbing, and it's tested:
- One YAML file defines a run, and unknown keys are rejected so a typo can't mislabel results
- Every model call is cached by a hash of the full request, retried only when retrying makes sense, and logged
- CI runs the tests a second time with the network switched off

No results yet, and the README says so.

The hardest part so far is data. The NGX website blocks automated access, and I won't work around it.

https://github.com/MelvTheGoat/Stocks

#LLM #Evaluation #Python #NigerianStocks
