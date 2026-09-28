# Fraud-Detection-With-Sequence-Models: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

---

## Step 1: Requirements (2 min)

One line:
> "Score each card authorisation for fraud in real time, using the customer's recent transactions, and decide approve/review/decline at the lowest cost."

**Functional**
1. Score a transaction given its recent history.
2. Decide using a cost-based threshold.
3. Explain which prior transactions mattered.
4. Log every decision.
5. Monitor for adversarial drift.

**Non-functional**
- **p99 ≤ 50 ms** server-side (the authorisation budget is a few hundred ms end to end).
- **No leakage:** features only from strictly earlier transactions, and never labels.
- **Online = offline:** identical feature code.
- **Honest evaluation:** multiple seeds, temporal splits, cost at the operating point.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Transactions (synthetic) | ~297k, 0.63% fraud |
| Customers | 2,500 |
| Training frauds | 1,137 |
| History per request | 62 prior transactions (window 32, features bounded at 31) |
| Merchant vocabulary | ~2.9k (embedding 16) |
| Latency p99 (README) | 7.2 ms end to end |

**Say:** "Fraud is rare, 0.6%, so accuracy is useless. The key metrics are PR-AUC, precision at the alert budget, and cost."

---

## Step 3: High-level boxes (2 min)

```
[Auth request + last 62 txns] -> [Shared causal feature code] -> [Window of 32]
      -> [ONNX model (GRU)] -> [Isotonic calibration] -> [Cost threshold]
      -> [Decision + top prior txns] -> [Audit log] -> [Response]
                                    \-> [Monitoring: alert rate, PSI, precision]
Offline: [Simulator] -> [Temporal split + embargo] -> [LightGBM vs GRU/TCN/Transformer x 3 seeds]
```

---

## Step 4: Deep dive (10 min)

### 4a. Leakage control
- Features use only earlier transactions for that customer. Tested by scrambling or deleting the future.
- Labels are never features (chargebacks arrive weeks later). Tested by flipping labels.
- Target encoding is **out-of-time**, not shuffled K-fold. The smoothing prior is expanding.
- A deliberately leaky feature proves the tests can fail.

### 4b. Online/offline parity
- One feature function over ordered arrays, called by batch and serving.
- Bounded at 31 prior transactions, because a request can't carry unlimited history.

### 4c. Models
- A shared encoder with embeddings (`1.6·card^0.56`, capped at 16). Rare merchants share a row.
- GRU (hidden 64), TCN (dilations 1/2/4/8, causal by padding), Transformer (2 blocks, 4 heads, causal mask).
- The scored transaction sits at the last position.

### 4d. Training and evaluation
- Early stopping on validation PR-AUC. bce vs focal vs weighted. 3 seeds.
- PR-AUC, precision@0.1/0.5/1%, per-mechanism recall, pre- vs post-drift.

### 4e. Decision policy
- Costs: FP £8, missed fraud £15 + amount, review £2.
- Threshold on validation, applied to test **per seed**.

### 4f. Serving
- ONNX Runtime (0.28 ms forward pass), FastAPI, audit before the response, `/metrics` for latency.

### 4g. Monitoring
- Score PSI (failed here: >99% of traffic is legitimate and unchanged).
- The **alert rate at a fixed threshold** caught the drift immediately.

---

## Step 5: Bottlenecks (2 min)

1. **Feature computation** (1.3 ms) costs more than the model (0.28 ms).
2. **History retrieval** for 62 transactions per request. A feature store at scale.
3. **Label delay:** chargebacks come weeks later.
4. **Adversarial adaptation:** the attacker targets your strongest features.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Bounded 31-txn history | Feature store | Parity without a second system | Long-horizon signal lost |
| GRU deployed | Transformer | Smaller, faster, better here | Less capacity if data grows |
| ONNX | Eager PyTorch | 2.8× faster forward pass | Export step |
| Isotonic calibration | Raw | Fixes the alert region, keeps ranking | Needs validation data |
| Per-seed costing | Seed-averaged ensemble | Matches what's deployed (one checkpoint) | Honest but noisier |
| Alert-rate monitoring | PSI only | PSI misses targeted drift | Needs a fixed threshold |
