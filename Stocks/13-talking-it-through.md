# Stock Question Agent: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to where it is today. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: this project is early. The foundation is built and tested. The AI helper itself is the next step. So this story ends partway, and I'll tell you exactly where.*

---

## Okay, so what are we building?

Alright. You know how AI chatbots can sound really sure of themselves and still be wrong? Ask one for a company's share price, and it might just make one up. That's called a hallucination.

So here's our project. We want an AI helper that answers factual questions about stocks, both US stocks and Nigerian stocks on the NGX. But the most important part isn't the helper. It's **measuring how often it's right**.

And there's a clever twist. AI models have read loads about Apple, and almost nothing about Nigerian Breweries. So if we test both, we can see how much of the model's skill is just memory, and how much is real reasoning.

## So what do we need?

Let's list it out:

1. **A way to describe a test run**, exactly, so results can be repeated.
2. **A way to talk to the AI model**, that works the same everywhere.
3. **Saving answers**, because free GPU time is limited, about 30 hours a week.
4. **Retrying failures**, but only the ones worth retrying.
5. **A record of every call**: how long it took, how big it was.
6. **Pretend models for testing**, so tests don't need a GPU.
7. **Data**: stock prices with known correct answers.
8. **Test questions and graders**, to mark answers right or wrong.
9. **The AI helper itself.**
10. **Automatic checks** on every change.

Now here's the big decision. We build the **measuring stick first**. The order is: settings, then the plumbing for talking to the model, then tests, then data, then questions, and only then the helper. Today, the first part is done.

## Step zero: set up the workshop

First the basics. A `README.md` as the front page. It's honest: where results would go, it says "not run yet" instead of made-up numbers.

A `.gitignore`, so git doesn't save things like the saved answers folder.

Then the install lists. `requirements.txt` has just three tools: Pydantic, PyYAML and httpx. `requirements-dev.txt` adds the testing tools on top: pytest, pytest-cov and ruff. And `pyproject.toml` describes the project as a Python package, with its settings for the tests and the linter.

The code lives in `src/stockagent/`. Its `__init__.py` has a one-line description: "Question answering over US and Nigerian stock data, plus its eval harness."

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `requirements.txt`: the three tools the code needs.
> - `requirements-dev.txt`: extra tools for testing and tidying.
> - `pyproject.toml`: the project's package details and tool settings.
> - `src/stockagent/__init__.py`: marks the main code folder as a package.

## Step one: one file describes one test run

The first real piece is the settings. In `src/stockagent/config.py`, we say: one test run is described by one settings file, and nothing else.

Which model, which settings for the model, which test questions, which kind of helper, which random seed. Everything that could change a result lives in that one file. So any result can be traced back and repeated.

And it's strict. Here's the real line that makes it so:

```python
_STRICT = ConfigDict(extra="forbid")
```

That means any setting the code doesn't recognise causes an error.

Why does that matter? Say you misspell "temperature" as "temparature". Most programs would quietly ignore it, and every result would be labelled wrong. This one refuses to run.

Each settings file also gets a short fingerprint, a 12-character code. It's written next to every result. If two results disagree, the first thing to check is whether their fingerprints match.

Our first real settings file is `configs/smoke.yaml`. It's a tiny run: 5 questions, model only, no tools. Its own description says it's "not a measurement of anything". It's just there to prove the whole chain works end to end.

It names the planned model too: Qwen2.5-7B, squeezed down to about 5 GB so it fits on a free 16 GB GPU. And it sets temperature to 0, the most predictable mode, so results don't wobble randomly.

Tested in `tests/test_config.py`: settings load, check themselves, and refuse to hide mistakes.

> **📁 Files we just created**
> - `src/stockagent/config.py`: strict settings for one test run, with a fingerprint.
> - `configs/smoke.yaml`: a tiny first run that just proves the chain works.
> - `tests/test_config.py`: checks settings load and reject mistakes.

## Step two: one shape for every model call

Next, we decide what "talking to a model" looks like. That's `src/stockagent/llm/base.py`.

Every call to any model goes through one shape: send a request, get a response. Tests get a pretend model, the GPU computer gets a real one, and the code in between can't tell the difference.

It also defines two kinds of failure. **Temporary** ones, like "server busy", are worth retrying. **Permanent** ones, like a badly formed request, are not.

And it builds the cache key, a fingerprint of the *entire* request. Here's the start of that function:

```python
def cache_key(self) -> str:
    """Stable hash of the entire request.
```

The clever bit is that it's built from every field automatically. So if someone adds a new setting later, it's included in the key without anyone remembering to add it. Otherwise, the cache could serve an old answer made with different settings.

> **📁 Files we just created**
> - `src/stockagent/llm/base.py`: one shape for every model call, the two failure types, and the cache key.

## Step three: the plumbing

Now we build the pieces that sit around every model call. Think of them like layers of an onion.

**The real connection.** `src/stockagent/llm/openai_client.py` talks to the model over the web. It uses the OpenAI message format, which vLLM, the planned model server, also speaks. So the same code works on Kaggle or anywhere else. It sorts failures: timeouts, "too many requests" and server errors are temporary, and everything else is permanent.

**Retries.** `src/stockagent/llm/retry.py` retries temporary failures only, up to 4 times. Here's how it works out each wait:

```python
ceiling = min(self.base_delay_s * (2**attempt), self.max_delay_s)
```

So it waits 1 second, then 2, then 4, then 8, never more than 30. And each wait gets a random trim, so lots of retries don't all hit at once.

**Saving answers.** `src/stockagent/llm/cache.py` saves every answer as a small file, spread across 256 folders so no folder gets too full. It writes to a temporary file first, then renames it in one step. So if Kaggle kills the session halfway, you never get a broken half-file.

**The call log.** `src/stockagent/llm/calllog.py` writes one line for every call: how many tokens, how long it took, whether it came from the cache, any errors. Cost and speed are results too, not just accuracy.

**Putting the layers together.** `src/stockagent/llm/__init__.py` has `build_client`, which stacks them in a set order: log, then cache, then retry, then the real connection. The order matters. A call that fails twice then succeeds is saved once and logged once.

**Pretend models.** `src/stockagent/llm/fake.py` has two. One gives scripted answers, for driving tests. The other, `NeverCalledClient`, fails the test if it's ever called at all. That proves certain paths never reach the model.

Tests: `tests/test_llm_base.py` checks the request shape, the cache key and the fakes. `tests/test_llm_plumbing.py` checks the cache, retries and logging. `tests/test_openai_client.py` checks the real connection, using a pretend network so nothing leaves the computer.

Let's pause there. We now have strict settings, one shape for every call, safe saving, smart retries and a full record. That plumbing is done and tested. That's exactly where the project is today.

> **📁 Files we just created**
> - `src/stockagent/llm/openai_client.py`: the real connection to the model.
> - `src/stockagent/llm/retry.py`: retries temporary failures, waiting longer each time.
> - `src/stockagent/llm/cache.py`: saves every answer safely.
> - `src/stockagent/llm/calllog.py`: one line of records per call.
> - `src/stockagent/llm/__init__.py`: stacks the layers in the right order.
> - `src/stockagent/llm/fake.py`: pretend models for tests.
> - `tests/test_llm_base.py`, `tests/test_llm_plumbing.py`, `tests/test_openai_client.py`: the plumbing tests.

## Step four: automatic checks

Every change gets checked automatically, by GitHub Actions, a free service that runs code when it changes.

`.github/workflows/tests.yml` runs three things. First ruff, a linter that spots messy code. Then the tests.

Then the tests *again*, with the internet blocked. If any test is secretly using the network, it fails right there, not on some random day when a service is down.

`.github/workflows/authorship.yml` is a different kind of check. It fails if any file or commit mentions an AI tool, to keep the project's authorship clean. It's a rule, not a habit, so it can't quietly lapse.

And `tests/test_package.py` is the simplest test of all. It checks the package can be imported and the test runner is wired up correctly. All 64 tests pass in about 1.5 seconds.

> **📁 Files we just created**
> - `.github/workflows/tests.yml`: lint, tests, and tests with the internet blocked.
> - `.github/workflows/authorship.yml`: keeps the project's authorship clean.
> - `tests/test_package.py`: checks the package and test setup work.

## Step five: the data, and why it's hard

Now the data. And this is where the project hit a real wall.

Before collecting anything, every source gets checked and written down in `DATA_SOURCES.md`. Its rule is simple: "Nothing gets collected until it has an entry here." Every check is dated.

The official Nigerian exchange website, ngxgroup.com, sits behind a bot filter. Every automated visit gets a JavaScript challenge, and a normal program can never get past it. And the site's terms of use couldn't be checked either.

The project refuses to sneak around the filter. So that source is marked "blocked, not in use". Another site has the data, but its terms page doesn't exist, so it's not used yet either. US sources haven't been investigated yet.

There's an empty folder waiting for the Nigerian data collector: `src/stockagent/collectors/`, with just its `__init__.py`.

> **📁 Files we just created**
> - `DATA_SOURCES.md`: every data source checked, with dates and terms.
> - `src/stockagent/collectors/__init__.py`: an empty folder waiting for the data collector.

## What comes next (planned, not built)

So here's where the story pauses. What's planned, in order:

1. **A data collector**, once an allowed source is found.
2. **Test questions** with known answers, each fixed to a date, since prices change daily. They'll be split into practice and final sets, so the final score stays honest.
3. **Graders**: one for numbers within a small margin, one for exact answers, and one for checking the right source was used.
4. **The AI helper**, in three kinds: model only, model with notes, and model with tools like a price lookup and a calculator.
5. **Running it on Kaggle**, with vLLM serving the model on a free GPU.

Notice how the plumbing we built is waiting for all of this. The settings file already has fields for the helper type and the test split. The pretend models are ready to drive the graders. The cache means re-grading will cost nothing.

## How's it doing?

There are no results yet. How often the AI is right, how fast it is and what it costs haven't been measured, because the questions don't exist yet.

What *is* proven is the foundation. All 64 tests pass, including with the internet blocked.

## What's still missing?

- **Most of the system**: the data, the questions, the graders and the helper.
- **An allowed data source** for Nigerian stocks.
- **US data sources** haven't been looked at yet.
- **The client sends one request at a time.** vLLM is much faster with many at once, so that will matter later.
- **The README mentions an `ARCHITECTURE.md`** file that doesn't exist yet.

## Let's put it all together

So let's look at it in one breath.

We decided to **build the measuring stick first**. We made **strict settings**, so one file describes one run and typos can't hide. We gave every model call **one shape**, with two kinds of failure and a cache key built from every field.

Then we built the **plumbing**: a real connection, smart retries, safe saving, a full call log, all stacked in the right order. We made **pretend models**, including one that proves the model is never called. We added **automatic checks**, including running the tests with the internet blocked.

And we did the data homework honestly, refusing to sneak past a bot filter.

Notice how it links. The fingerprint from the settings, the cache key and the call log all serve one goal: every result can be traced and trusted. That's exactly what a measuring stick needs.

That's the project so far. The foundation is solid. The helper is next.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
