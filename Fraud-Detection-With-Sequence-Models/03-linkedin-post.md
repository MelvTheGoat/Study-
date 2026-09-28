# Fraud-Detection-With-Sequence-Models: LinkedIn Post

*About 170 words. Copy from the line below.*

---

Do deep learning models beat gradient boosting at fraud detection? I measured it, and the answer is "only after the fraudsters adapt".

I built a fraud system comparing GRU, TCN and Transformer models over each customer's recent transactions against a LightGBM baseline built to win, on synthetic data with three known attack types and an adversary that changes tactics near the end.

What I found (3 seeds each):
- Before the drift: tied. LightGBM was marginally ahead.
- After the drift: GRU PR-AUC 0.765 vs LightGBM 0.615. The trees leaned on exactly the features the attacker neutralised.
- Card testing, a pattern across consecutive transactions, is where ordering helped most: 0.92 vs 0.84 recall.
- The cost saving at the deployed threshold was not significant. A better ranker isn't automatically cheaper.

Also: the smallest model (GRU) won, standard PSI drift monitoring never fired, and the served model scores in 7 ms at p99.

An earlier version reported a 27% saving. It came from accidentally averaging three models into an ensemble. I corrected it.

https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

#FraudDetection #DeepLearning #MachineLearning #MLOps
