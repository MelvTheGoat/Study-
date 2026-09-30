# Stock Question Agent: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

Remember, this project is at an early stage. Terms marked *(Planned)* describe parts that are designed but not built yet.

---

## 1. The AI Side

### LLM (large language model)
**What it means:** an AI that reads and writes text, like ChatGPT.

**Why it's needed here:** it's the thing being tested. The project wants to know how often an LLM gives correct facts about stocks.

### Hallucination
**What it means:** when an AI confidently makes something up, like a share price that was never real.

**Why it's needed here:** it's the exact problem the project exists to measure. An AI can sound sure and still be wrong.

### Agent *(Planned)*
**What it means:** an LLM that can use tools, like a price lookup or a calculator, step by step to finish a task.

**Why it's needed here:** looking up a price is more reliable than remembering it. The plan compares three kinds: model only ("closed book"), model plus given notes ("retrieval"), and model plus tools ("agent").

### Closed book
**What it means:** the model answers from memory alone, with no notes or tools.

**Why it's needed here:** it's a baseline, a simple starting point to compare against. If tools don't beat it, they aren't helping.

### Self-check *(Planned)*
**What it means:** an optional extra step where the model reviews its own answer before giving it.

**Why it's needed here:** it's a setting in the agent config, so the project can test whether checking helps.

### Memory vs reasoning
**What it means:** memory is facts the model saw during training. Reasoning is working things out from information it's given.

**Why it's needed here:** models have seen far more about US stocks than Nigerian ones. Comparing the two shows how much a correct answer came from memory.

---

## 2. Measuring the AI

### Evaluation (eval) *(Planned)*
**What it means:** testing the AI with questions whose correct answers you already know.

**Why it's needed here:** it's the core of the project. The plan is to build the eval first, so every later change can be measured.

### Grader *(Planned)*
**What it means:** a piece of code that marks one answer as right or wrong.

**Why it's needed here:** three are planned: **numeric** (is the number close enough?), **exact** (does it match exactly?) and **source** (did it cite the right document?).

### Split *(Planned)*
**What it means:** dividing the questions into separate groups, like "dev" for practice and "test" for the final score.

**Why it's needed here:** if you keep tweaking the AI against the test questions, the score stops being honest. Keeping a separate test set prevents that.

### As-of date
**What it means:** a fixed date that all the answers are correct for.

**Why it's needed here:** share prices change daily. Freezing the date means "the right answer" doesn't change under your feet.

### Baseline
**What it means:** a simple version used as a comparison point.

**Why it's needed here:** a result only means something next to a baseline. Here, "closed book" is the baseline for the agent.

---

## 3. Talking to the Model

### API
**What it means:** a way for one program to ask another for something, by sending a message to a web address.

**Why it's needed here:** the model runs on a server, and the project talks to it through an API.

### OpenAI-compatible API
**What it means:** an API that uses the same message format as OpenAI's, which many tools copy.

**Why it's needed here:** the planned model server (vLLM) speaks this format. So the same client code works wherever the model runs.

### HTTP
**What it means:** the basic language computers use to send requests and answers over the web.

**Why it's needed here:** every call to the model is an HTTP request. The project's HTTP client handles sending, timeouts and reading the reply.

### Timeout
**What it means:** the longest you'll wait for an answer before giving up.

**Why it's needed here:** without one, a stuck server could freeze a whole run and waste GPU time.

### Temperature and greedy decoding
**What it means:** temperature controls how random the model's answers are. At 0 ("greedy"), it always picks its most likely next word.

**Why it's needed here:** the default is 0. If answers changed randomly between runs, you couldn't tell whether an improvement was real.

### Seed
**What it means:** a starting number for anything random, so the same seed gives the same results.

**Why it's needed here:** it helps make runs repeatable. The code notes it also depends on the server and hardware.

### Token
**What it means:** a small piece of text, roughly three quarters of a word. Models read and write in tokens.

**Why it's needed here:** tokens measure how much work each call is. The logger records them for every call.

### Latency
**What it means:** how long you wait for an answer.

**Why it's needed here:** speed is a result too. Cached answers are marked, so they don't make the model look faster than it is.

---

## 4. Handling Failures

### Transient error
**What it means:** a temporary failure, like "server busy" or "too many requests". Trying again later may work.

**Why it's needed here:** these are the only errors that get retried.

### Permanent error
**What it means:** a failure that won't fix itself, like a badly formed request.

**Why it's needed here:** retrying these would only waste scarce GPU time, so they fail straight away.

### Exponential backoff
**What it means:** waiting longer after each failure: 1 second, then 2, then 4, and so on.

**Why it's needed here:** a busy server needs time to recover. Here it waits 1, 2, 4 then 8 seconds, with a 30-second cap.

### Jitter
**What it means:** adding a small random change to each wait.

**Why it's needed here:** if many requests retry at exactly the same moment, they overload the server again. Randomness spreads them out.

---

## 5. Saving and Recording

### Cache
**What it means:** a saved copy of an answer, reused when the same question comes up again.

**Why it's needed here:** GPU time is limited to about 30 hours a week. With the cache, re-grading old answers costs no model calls at all.

### Cache key (hash)
**What it means:** a hash is a short fingerprint made from some data. The same data always gives the same fingerprint. The cache key is the fingerprint of a request.

**Why it's needed here:** the key uses *every* setting of the request. So an answer made with different settings can never be served by mistake, even if new settings are added later.

### Sharding
**What it means:** splitting many files across lots of folders instead of one.

**Why it's needed here:** one folder with huge numbers of files gets slow. The cache spreads files across 256 folders.

### Atomic write
**What it means:** saving a file in a way that it's either fully written or not there at all, never half-written.

**Why it's needed here:** Kaggle can kill a session at any moment. The cache writes to a temporary file first, then renames it in one step.

### Call log (JSONL)
**What it means:** a file with one line of data per model call. JSONL means each line is a separate piece of JSON (simple labelled data).

**Why it's needed here:** it records tokens, time taken, errors and whether each call was cached. It sits above the cache, so even cached answers are counted.

### Run config and fingerprint
**What it means:** the config is a settings file for one run. Its fingerprint is a 12-character code made from the settings.

**Why it's needed here:** results are labelled with the fingerprint, so you can tell if two results came from the same setup.

### Strict validation
**What it means:** checking a settings file and rejecting anything unexpected.

**Why it's needed here:** a typo in a setting would otherwise be ignored, and every result would be mislabelled. Here, unknown settings cause an error.

---

## 6. Running and Testing

### GPU
**What it means:** a special computer chip that does lots of maths at once. AI models need one to run fast.

**Why it's needed here:** the plan is to use a free GPU on Kaggle, which shapes every design choice.

### Quantisation *(Planned)*
**What it means:** storing a model's numbers with less detail so it takes less memory.

**Why it's needed here:** the planned model is squeezed to 4-bit, about 5 GB, so it fits on the free 16 GB GPU with room to spare.

### Fake client
**What it means:** a pretend model used in tests, which gives scripted replies.

**Why it's needed here:** tests can run instantly with no GPU. One special fake fails the test if it's called at all, which proves a path never reaches the model.

### CI (continuous integration)
**What it means:** a robot that automatically checks the code every time it's changed.

**Why it's needed here:** it runs the linter (a tool that spots messy code), the tests, and the tests again with the internet blocked. A second check fails if any file or commit mentions an AI tool, to keep authorship clean.

### Bot filter (WAF)
**What it means:** a security guard for a website that blocks automated visitors. WAF stands for web application firewall.

**Why it's needed here:** the official Nigerian exchange site has one. The project refuses to sneak around it, so it has to find another allowed data source.
