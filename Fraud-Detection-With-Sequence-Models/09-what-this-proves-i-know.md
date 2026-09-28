# Fraud-Detection-With-Sequence-Models: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

---

## 1. Sequence modelling (RNN, TCN, Transformer)

**Simple explanation:** models that read an ordered list of events and use the order.

**In this project:** GRU, dilated causal TCN, a causal Transformer, all with the prediction at the last position.

**Also be ready to explain:** GRU vs LSTM gates, vanishing gradients, dilated convolutions and receptive field, self-attention and causal masks, positional encodings, padding and masking.

---

## 2. Embeddings for high-cardinality categoricals

**Simple explanation:** learn a small vector for each merchant so similar merchants end up close together.

**In this project:** width `1.6·card^0.56` capped at 16, and a shared "rare" row.

**Also be ready to explain:** one-hot vs target encoding vs embeddings, the hashing trick, cold start for new merchants, and overfitting in the long tail.

---

## 3. Fraud detection domain

**Simple explanation:** find the few bad transactions among many good ones, fast.

**In this project:** card testing, account takeover, merchant compromise, chargeback delay, alert capacity, cost of false declines.

**Also be ready to explain:** authorisation vs settlement, 3-D Secure, velocity rules, device fingerprinting, label delay, friendly fraud, and review queues.

---

## 4. Leakage and temporal validation

**Simple explanation:** never let the model see the future during training.

**In this project:** behavioural leakage tests, out-of-time target encoding, an expanding prior, temporal splits with an embargo.

**Also be ready to explain:** point-in-time features, purged/embargoed CV, target leakage vs temporal leakage, and train/serve skew.

---

## 5. Imbalanced learning

**Simple explanation:** handle problems where positives are rare.

**In this project:** BCE vs focal vs weighted sampling, PR-AUC, precision@capacity, and accuracy shown next to "never fraud".

**Also be ready to explain:** focal loss (α, γ), why PR-AUC over ROC-AUC under imbalance, and resampling effects on calibration and ranking.

---

## 6. Calibration and cost-sensitive thresholds

**Simple explanation:** make scores honest probabilities, then pick the cutoff by money.

**In this project:** temperature and isotonic, alert-region calibration, expected cost/txn, a validation-chosen threshold per seed, an FP-cost sweep.

**Also be ready to explain:** quantile ECE, why aggregate ECE hides alert-band errors, and expected-cost thresholds.

---

## 7. Experimental rigour

**Simple explanation:** make sure a difference isn't luck or a quirk of the setup.

**In this project:** 3 seeds with mean ± sd, a strong baseline, pre/post-drift splits, per-mechanism breakdown, the ensemble mistake corrected.

**Also be ready to explain:** statistical significance with few seeds, effect size vs variance, and "evaluate what you deploy".

---

## 8. Real-time ML serving

**Simple explanation:** answer within a strict time budget.

**In this project:** ONNX export, FastAPI, latency percentiles, features vs model timing, an audit log, a health check requiring a loaded model.

**Also be ready to explain:** p50 vs p99, batching vs latency, ONNX Runtime, and cold starts.

---

## 9. Drift monitoring under adversarial conditions

**Simple explanation:** detect when attackers change tactics.

**In this project:** PSI failed, and the alert-rate ratio at a fixed threshold worked. Per-mechanism recall per window.

**Also be ready to explain:** covariate vs concept drift, targeted drift, and label-free monitoring signals.

---

## 10. Explainability limits

**Simple explanation:** attention shows where the model looked, not why it decided.

**In this project:** attention, gradient attributions and TreeSHAP, with caveats in the API response.

**Also be ready to explain:** "attention is not explanation", integrated gradients, and SHAP additivity.
