# Fraud-Detection-With-Sequence-Models: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

1. **The question: do GRU/TCN/Transformer over a customer's recent transactions beat a strong LightGBM for real-time fraud?**
   *Why it matters:* the project is a measurement, not a sales pitch.

2. **Synthetic world: ~297k transactions, 0.63% fraud, 3 labelled mechanisms (card testing, account takeover, merchant compromise), realistic confounders, adversarial drift in the final 15%.**
   *Why it matters:* it lets you ask "which attack is each model blind to?"

3. **No leakage: features only from earlier transactions, labels never used as features, out-of-time target encoding, and behavioural tests including one proving the tests can fail.**
   *Why it matters:* the most dangerous silent bug in fraud modelling.

4. **One feature implementation for batch and serving, with history bounded to 31 prior transactions. The service asks for 62.**
   *Why it matters:* online/offline parity.

5. **Ranking: GRU PR-AUC 0.870 ± 0.014 vs LightGBM 0.826 ± 0.001 (~3σ).**
   *Why it matters:* the headline ranking result.

6. **Pre-drift tied (0.950 vs 0.956). Post-drift 0.765 vs 0.615. Card-testing recall 0.92 vs 0.84.**
   *Why it matters:* the advantage is holding up when attackers adapt.

7. **Cost per transaction 0.0799 ± 0.0122 vs 0.0860 ± 0.0035. Not significant. The earlier "27% saving" came from an accidental ensemble and was corrected.**
   *Why it matters:* honesty about the deployment metric.

8. **The smallest model (GRU, 78k params, ~2 min) beat the Transformer (106k, ~5 min).**
   *Why it matters:* capacity needs data.

9. **Serving: ONNX + FastAPI, p99 7.2 ms vs a 50 ms budget. Features (1.3 ms) cost more than the model (0.28 ms). Audit before the response.**
   *Why it matters:* real-time constraints.

10. **Monitoring: score PSI never fired (peak 0.0011). The alert rate at a fixed threshold caught the drift in the first window. 144 tests pass.**
    *Why it matters:* targeted drift needs targeted alarms.
