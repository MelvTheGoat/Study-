# Fraud-Detection-With-Sequence-Models: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

---

## Summary

A controlled comparison of GRU/TCN/Transformer sequence models against a strong LightGBM baseline for real-time card fraud, on a synthetic world with three labelled mechanisms, realistic confounders and adversarial drift. It has behavioural leakage tests, online/offline feature parity (bounded 31-transaction history), per-seed calibration, cost policy, ONNX serving (p99 7.2 ms reported), and alert-rate drift monitoring. ~9,200 lines. **One commit (7 Aug 2026). Results come from the README, and the `results/` files aren't committed.**

## Architecture

```
src/simulate.py        world generator (mechanisms, confounders, drift in final 15%)
src/data/features.py   bounded causal features, ONE implementation for batch + serving
src/data/sequences.py  left-padded windows (32), masks, weighted sampling
src/data/splits.py     temporal splits + 1-day embargo
src/baselines.py       LightGBM (built to win) + 4-rule floor
src/models/            base encoder (embeddings), rnn (GRU), tcn, transformer
src/train.py           bce | focal(alpha=0.25) | weighted; early stop on val PR-AUC; seeds
src/evaluation.py      PR-AUC, precision/recall@capacity, per-mechanism recall
src/calibration.py     Brier, quantile ECE, alert-region calibration, temperature, isotonic
src/policy.py          expected cost/txn, validation threshold, FP-cost sweep
src/explain.py         attention, gradient attribution, TreeSHAP
src/monitoring.py      PSI, alert-rate ratio, rolling precision, per-mechanism recall
src/api/               ONNX export, FastAPI service, audit, latency benchmark
```

## Key decisions (from DECISIONS.md)

| Decision | Why |
|---|---|
| Synthetic first, real optional | Mechanism labels and controllable drift |
| Confounders as important as the fraud | Stops a trivially separable problem (PR-AUC 1.0) |
| Behavioural leakage tests + a detector with teeth | Structural review misses subtle leaks |
| Expanding smoothing prior | Found a real leak on the first run |
| Out-of-time target encoding | Shuffled K-fold leaks the future |
| Bounded 31-transaction history | Online/offline parity without a feature store |
| Service asks for 62 prior transactions | Oldest window position gets full history, as in training |
| Embedding width `1.6·card^0.56`, capped at 16 | Avoids a 400k-param merchant table on ~1k positives |
| Rare merchants (<3) share a row | Avoids memorising noise |
| Scored transaction at the last position | Same causal contract for all architectures |
| Early stop on validation PR-AUC | Loss isn't the objective |
| 3 imbalance arms × 3 seeds, mean ± sd | Honest variance |
| Threshold on validation, applied to test, **per seed** | Matches the single deployed checkpoint |
| Missed fraud cost = £15 + amount | Loss scales with the transaction |
| FP cost swept £2–£30 | The ordering is checked for stability |

## Models

- **Shared encoder:** 6 categorical embeddings (47,006 params in total, mostly merchant) plus numeric features.
- **GRU:** 1 layer, hidden 64. Input rolled so real transactions start at index 0 and padding trails, and the state is read at `length−1`. 78,031 params.
- **TCN:** 4 dilated causal blocks (1/2/4/8), kernel 3, 48 channels, receptive field 61. Causal by left padding. 112,591 params.
- **Transformer:** 2 pre-norm blocks, d=64, 4 heads, learned positions, causal + padding mask. Fused attention for training and an explicit version for weights (tested equal). 105,679 params.
- **Losses:** BCE, focal (α=0.25), weighted sampling (~40× positives).

## Evaluation (reported)

| | GRU (best) | LightGBM |
|---|---|---|
| PR-AUC | 0.870 ± 0.014 | 0.826 ± 0.001 |
| Pre-drift PR-AUC | 0.950 | 0.956 |
| Post-drift PR-AUC | 0.765 | 0.615 |
| Precision @0.5% | 0.816–0.824 | 0.765 |
| Card-testing recall | 0.92 | 0.84 |
| Cost / txn | 0.0799 ± 0.0122 | 0.0860 ± 0.0035 |
| Train time (4 CPU) | ~2–3 min | ~20 s |

- **Calibration:** alert-region ratio (predicted/actual) is 0.69× for focal (under-confident), ~1.0× for BCE, and 1.04–1.23× for weighted. Isotonic brings ECE to ~0.0001–0.0005 with no ranking loss.
- **Latency:** ONNX forward pass p50 0.284 ms vs PyTorch 0.790 ms. Features 1.30 ms. End-to-end p99 7.2 ms.
- **Monitoring:** score PSI peak 0.0011 (never fires). The alert-rate ratio dropped to 0.57 in the first drifted window.
- **Tests:** leakage, features and sequences, models, calibration and policy, evaluation, monitoring, serving parity, a metric regression gate, training, simulation. **My run: 144 tests pass (~71 s on CPU).**

## Known weaknesses

- One synthetic world and one author-designed drift.
- 3 seeds can't resolve the cost gap.
- ~1,137 training frauds (the Transformer is under-fed).
- Per-transaction, not per-episode, recall (optimistic for card-testing bursts).
- A 31-transaction cap loses long-horizon signal.
- Assumed costs. No fairness evaluation (no demographics).
- IEEE-CIS and ULB adaptations are lossy (no true customer or merchant keys, and ULB can't form sequences).
- Results aren't committed, so they can't be verified without re-running.
