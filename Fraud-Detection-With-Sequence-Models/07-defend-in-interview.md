# Fraud-Detection-With-Sequence-Models: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

---

## 60-second pitch

> "I measured whether sequence models, a GRU, TCN and Transformer over a customer's recent transactions, actually beat a strong LightGBM baseline for real-time card fraud. I built a synthetic world with three labelled fraud mechanisms, realistic confounders, and an adversary that adapts near the end, and enforced no-leakage with behavioural tests and one shared feature implementation for training and serving.
>
> Result, over three seeds: before the drift they're tied, and LightGBM is marginally ahead. After the drift, the GRU holds PR-AUC 0.765 vs LightGBM's 0.615, because the trees leaned on the exact features the attacker neutralised. But the cost saving at the deployed threshold isn't statistically significant. The smallest model, the GRU, won. It's served via ONNX at 7 ms p99, and an alert-rate monitor caught the drift that PSI missed."

---

## Questions and honest answers

### 1. "Why not just use LightGBM?"
On a stationary problem you should, and my results show it: pre-drift it matched or beat every network, trained in 20 seconds, and had a tenth of the seed variance. Sequence models earn their keep when the attack is a pattern across transactions and you expect the adversary to adapt.

### 2. "How do you know there's no leakage?"
Behavioural tests: scramble and then delete all transactions after a cut point, and assert that nothing before it changes. Flip every label, and assert the features are identical. Target encoding is out-of-time. And a deliberately leaky feature proves the tests can fail. The suite caught a real leak, a global smoothing prior, on its first run.

### 3. "Why bound history at 31 transactions?"
Online/offline parity. An authorisation request can't carry a lifetime of history, and a separate feature store would drift from the training code. The cost is losing long-horizon signal. At scale, the answer is a feature store that shares the same code path.

### 4. "Why did the GRU beat the Transformer?"
About 1,100 fraud examples in training. The Transformer has more capacity but not enough signal to use it. The GRU is smaller (78k params), faster (~2 min) and more stable. I'd only reach for attention with evidence the GRU is the bottleneck.

### 5. "Is the improvement real?"
In ranking, yes: 0.870 ± 0.014 vs 0.826 ± 0.001 is about 3σ. In cost at the operating point, no: 0.0799 ± 0.0122 vs 0.0860 ± 0.0035 is inside one standard deviation. I'd need about 10 seeds to settle it.

### 6. "Tell me about a mistake."
I first reported a 27% cost saving. I'd calibrated and thresholded the average of three seeds' scores, which is an ensemble, and ensembling helps noisy networks far more than stable trees. We deploy one checkpoint, so now everything runs per seed. The saving disappeared.

### 7. "Why doesn't PSI work here?"
Over 99% of transactions are legitimate and don't change, so the score distribution barely moves (peak PSI 0.0011). The attacker changes the fraud band only. The alert rate at a fixed threshold looks at that band and dropped to 57% of normal in the first drifted window.

### 8. "How do you handle class imbalance?"
I compared plain cross-entropy, focal loss and weighted sampling. Focal made the model under-confident in the alert region (alpha 0.25 shrinks positive probabilities). Weighted sampling hurt ranking more than calibration. Isotonic calibration fixes probabilities without changing ranking, so that's what's deployed.

### 9. "How fast is it?"
Reported p99 end to end is 7.2 ms against a 50 ms budget, on a shared 4-core CPU. The ONNX forward pass is 0.28 ms. Building features takes 1.3 ms, so optimise the features before the model.

### 10. "How would you scale it?"
A feature store with the same feature code for long history, streaming ingestion with per-customer state, batched ONNX inference, graph features for merchant compromise, shadow/champion-challenger deployment, and a delayed-label pipeline for chargebacks.

### 11. "What's your explanation method?"
Attention weights (where the model read from), gradient attributions (how much the score moves), and TreeSHAP for LightGBM. I'm explicit that attention isn't a causal explanation. It's useful for pointing analysts to related prior transactions.

### 12. "How realistic is your drift?"
It's one drift designed by me, the same person who built the models. That's the biggest caveat. A drift that attacks sequence structure itself would likely hurt the sequence models more.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Synthetic data | "Would this hold on real fraud?" | "It's evidence, not proof. The real-data loaders exist, but IEEE-CIS lacks true customer keys and ULB can't form sequences." |
| 3 seeds | "Too few." | "Agreed for the cost claim. It's enough to see the ranking gap, since LightGBM's spread is 0.001." |
| Per-transaction recall | "Card-testing bursts are easy after the first one." | "Yes, the numbers are optimistic. Episode-level recall is the first thing I'd add." |
| Results not committed | "Can I see the outputs?" | "`make experiment` regenerates them. I'd commit a summary JSON next time." |
| Single commit | "When did you build this?" | "It was pushed as one commit. The history isn't in the repo." |
