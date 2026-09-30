# Card Fraud with Sequence Models: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like sequence models or calibration, are explained in [11-technical-terms.md](11-technical-terms.md).

Everything runs on a normal computer's processor (CPU). No GPU is needed.

---

## The Language

### Python
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** almost every data and machine learning tool works with Python.

---

## Working with Data

### NumPy, pandas and pyarrow
**What it is:** NumPy does fast maths on long lists of numbers. pandas works with tables of data. pyarrow reads and writes large data files quickly.

**Why it's used here:** they generate and store about 300,000 simulated payments, and build the features from ordered lists of each customer's payments.

---

## Building Models

### PyTorch
**What it is:** a popular Python tool for building and training AI models.

**Why it's used here:** it builds and trains the three sequence models: the GRU, the TCN and the Transformer. They're small enough to train on a normal CPU in a few minutes.

### LightGBM
**What it is:** a fast tool that builds many small yes/no flowcharts, each fixing the last one's mistakes.

**Why it's used here:** it's the strong baseline, the standard choice for fraud. It trains in about 20 seconds.

### scikit-learn
**What it is:** Python's most common toolbox for everyday machine learning.

**Why it's used here:** it provides isotonic calibration (adjusting scores into honest chances) and many of the measures, like PR-AUC.

### SHAP
**What it is:** a tool that explains a single prediction by showing how much each feature pushed it up or down.

**Why it's used here:** it explains LightGBM's decisions. For the sequence models, the project uses other methods to show which earlier payments mattered most.

---

## Serving

### ONNX and ONNX Runtime
**What it is:** ONNX is a standard file format for AI models. ONNX Runtime is a fast tool for running them.

**Why it's used here:** the winning model is exported to ONNX for the live service. The project reports it runs about 2.8 times faster than in PyTorch.

### FastAPI and Uvicorn
**What it is:** FastAPI is a Python tool for building APIs (ways for programs to ask for things). Uvicorn is the program that runs it.

**Why it's used here:** they serve the fraud-scoring service. It has addresses for scoring a payment, a health check, model details and live measurements.

### Docker Compose
**What it is:** Docker packs an app into a box, called a container, that runs the same anywhere. Compose starts one or more boxes with a single settings file.

**Why it's used here:** it runs the scoring service. The trained model files are attached "read-only", so the service can't accidentally change them.

---

## Quality Checks

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 144 tests pass in about 71 seconds. The most important ones check for data leakage, and that live and training features match exactly.

### ruff and mypy
**What it is:** ruff is a "linter" that flags mistakes and messy style. mypy checks every value is used as the right kind.

**Why it's used here:** they catch small mistakes early, which matters when bugs can be silent.

### make
**What it is:** a classic tool that runs a named list of commands, like `make experiment`.

**Why it's used here:** one command runs the whole experiment: 11 model setups, each trained 3 times. The results files aren't saved in the project, so this is how to reproduce them.

---

## Optional Real Data

### IEEE-CIS and ULB datasets
**What it is:** two public card fraud datasets used in research.

**Why they're included:** the project has loaders for them, but you must download them yourself. They don't have the customer and shop IDs that sequence models need, and the loaders say so.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python | The language everything is written in |
| NumPy, pandas, pyarrow | Make and handle the payment data |
| PyTorch | Builds the sequence models |
| LightGBM | The strong baseline |
| scikit-learn | Calibration and measures |
| SHAP | Explains LightGBM's decisions |
| ONNX Runtime | Runs the model fast in the live service |
| FastAPI + Uvicorn | The scoring service |
| Docker Compose | Runs the service in a box |
| pytest, ruff, mypy | Keep the code correct |
| make | Runs the full experiment with one command |
| IEEE-CIS, ULB | Optional real datasets |
