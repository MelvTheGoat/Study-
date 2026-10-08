# Training a Small LLM from Scratch: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software or service) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like tokenizers or checkpoints, are explained in [11-technical-terms.md](11-technical-terms.md).

One choice stands out: the training code is written directly in PyTorch, without a ready-made training framework. That's harder, but it means every step is visible and understood.

---

## The Language

### Python
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** almost all AI tools are built for Python, so it's the standard choice for training models.

---

## Building and Training the Model

### PyTorch
**What it is:** a popular Python tool for building and training AI models. It does the heavy maths on GPUs (special chips that do lots of maths at once).

**Why it's used here:** the whole model and training loop are built on it. It also provides fast attention, training across 2 GPUs, the short fp16 number format, and an optional speed-up step called "compile". The project needs version 2.3 or newer.

---

## Preparing the Data

### tokenizers (Hugging Face)
**What it is:** a very fast tool for building and running tokenizers (the tools that split text into tokens). It's written in Rust, a fast programming language.

**Why it's used here:** learning a tokenizer from about a billion characters would take far too long in plain Python.

### pyarrow
**What it is:** a tool for reading and writing large data files quickly, including parquet files (a compact table format).

**Why it's used here:** the FineWeb-Edu text comes as parquet files, so pyarrow reads them.

### NumPy
**What it is:** a Python tool for fast maths on long lists of numbers.

**Why it's used here:** it reads the token files using memory-mapping, which means only the parts being used are loaded. The files are gigabytes big, so this saves a lot of memory.

---

## Storing Things

### Hugging Face Hub (huggingface_hub)
**What it is:** Hugging Face is a website for sharing AI models and data. `huggingface_hub` is the Python tool for uploading and downloading from it.

**Why it's used here:** it stores the prepared token files and the model checkpoints for free. Each new session downloads what it needs from there. To stay inside free storage, only the latest and final checkpoints are kept.

### git (results branch)
**What it is:** a tool that keeps track of versions of files. A branch is a separate line of saved files.

**Why it's used here:** a special `results` branch stores each job's status and logs. It's free and easy to read. If two sessions save at the same moment, one simply tries again.

### PyYAML
**What it is:** a Python tool for reading YAML files, a simple, human-friendly settings format.

**Why it's used here:** each experiment is described in its own YAML settings file, and the job to-do list is a YAML file too.

---

## Running the Work

### Kaggle
**What it is:** a data science website that gives out free computer time, including sessions with 2 T4 GPUs (16 GB of memory each).

**Why it's used here:** it's free. The downsides shape the whole design: sessions end after about 11 hours, and T4s only support the fp16 number format, not the safer bf16.

### Jupyter notebook
**What it is:** a document that mixes code and notes, and runs the code in steps. Kaggle runs work inside notebooks.

**Why it's used here:** the project's notebook is pasted into Kaggle once. After that it simply runs the job runner, which works through the to-do list.

---

## Testing and Reporting

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code still works.

**Why it's used here:** 193 tests pass on a normal computer, with no GPU needed. They include a check that stopping and restarting training gives exactly the same model.

### matplotlib *(Planned)*
**What it is:** a Python tool for drawing charts.

**Why it's planned:** for plotting results like loss curves once real runs happen.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python | The language everything is written in |
| PyTorch | Builds and trains the model |
| tokenizers | Builds the tokenizer fast |
| pyarrow | Reads the web text files |
| NumPy | Reads the huge token files a bit at a time |
| Hugging Face Hub | Stores data and checkpoints for free |
| git results branch | Stores job statuses and logs |
| PyYAML | Reads the settings and the to-do list |
| Kaggle + notebook | Free GPUs to run the jobs |
| pytest | Checks the code, with no GPU needed |
| matplotlib | Charts (planned) |
