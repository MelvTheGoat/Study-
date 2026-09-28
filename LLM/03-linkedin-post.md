# LLM (gptlab): LinkedIn Post

*About 160 words. Copy from the line below.*

---

I use language models every day. So I wanted to know if I could train one from scratch.

gptlab trains small GPT-style models (1M to 100M parameters) in plain PyTorch on free Kaggle GPUs. No training framework: the model, training loop, tokenizer pipeline and evaluation are all written by me.

The interesting part isn't the model. It's surviving free hardware. Kaggle sessions end after about 11 hours, so:
- training stops itself before the limit and saves a checkpoint atomically
- it saves the weights, optimiser, loss scaler, data position and every random-number state
- a test checks that "train, stop, resume" gives bit-identical losses to training straight through

The data position is just the step number, so resuming needs nothing else.

No GPU runs yet. Scaling laws, ablations (RoPE, RMSNorm, SwiGLU) and HellaSwag scores come next, and every number will come from a logged run.

https://github.com/MelvTheGoat/LLM

#PyTorch #LLM #DeepLearning #MachineLearning
