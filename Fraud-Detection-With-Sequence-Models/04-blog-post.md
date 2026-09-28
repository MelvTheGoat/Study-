# Do sequence models beat gradient boosting at fraud? Only after the fraudsters adapt

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

## Why I built it

"Deep learning beats trees" is a common claim, and on flat tabular data it's usually wrong. Gradient-boosted trees handle mixed types, missing values and interactions with very little tuning. Reaching for a neural network *because* it's a neural network is how projects waste six months.

But transaction fraud has two properties where a sequence model has a real structural advantage:

1. **Fraud lives in patterns across a customer's history.** A £3 streaming payment is normal. Nine of them in four minutes from a new device is card testing. The signal is in the order, the spacing, and how the current transaction differs from *this* customer's own recent behaviour. A tree only sees the patterns an engineer thought to turn into features.
2. **Merchants, devices and merchant categories have huge vocabularies.** A learned embedding can place merchants that attract similar fraud near each other and share strength across the long tail.

So the thesis isn't "deep learning is better". It's: here's a problem where sequence models *plausibly* win, and here's an honest measurement of whether they did.

## The setup

### Data with known fraud mechanisms

I generated about 300,000 transactions for 2,500 customers over 180 days, at a fraud rate near 0.65%, with three injected mechanisms and **known labels**:

| Mechanism | Where the signal is |
|---|---|
| Card testing | Bursts of small authorisations |
| Account takeover | A sharp break from the customer's own baseline |
| Merchant compromise | Cards breached at one merchant, used elsewhere days later |

No public dataset tells you which attack caused which fraud, so no public dataset can answer "which mechanisms is this model blind to?"

The first version was too easy: the baseline scored a perfect PR-AUC of 1.0, so there was nothing to compare. The fix wasn't to weaken the models but to make the world realistic. Customers have several devices, some legitimate customers buy in bursts (exactly what card testing hides behind), people travel, attackers reuse a victim's own device 15–25% of the time, and 4% of fraud is never disputed.

In the final 15% of the timeline, every attack **adapts against its own defence**: card testing raises amounts into the normal range, takeover mimics the victim's spending, merchant compromise spreads across long-tail merchants, and the device fingerprint stops being distinctive.

### A baseline built to win

LightGBM gets velocity counts over 1 hour, 24 hours and 7 days, amount z-scores against the customer's own history, time since the last transaction and the last use of each merchant or device, novelty flags, out-of-time target encodings, and native categoricals. It sees the same information as the sequence models, flattened into one row. So the question is clean: **is there signal in the ordering that a well-built aggregate misses?**

### Three sequence models

A GRU, a TCN (causal by construction) and a Transformer (with a causal mask) share one input encoder and read their prediction from the last window position, which is always the transaction being scored. Each was trained three ways (plain cross-entropy, focal loss, weighted sampling) with three seeds.

## The hard parts

### Not reading the future

The most dangerous bug in fraud modelling is a feature that reads the future. Training converges, validation looks great, and only the live system disagrees. Three rules are enforced by **behavioural tests**:

1. A feature at time *t* uses only earlier transactions. The test scrambles, and then deletes, everything after a cut point and checks nothing before it changed.
2. Labels are never features, because chargebacks arrive weeks later. Flipping every label must leave every feature identical.
3. Anything that uses labels is fitted out-of-*time*, not just out-of-fold.

And a test checks that the tests can fail:

```python
def test_leakage_detector_has_teeth(transactions: pd.DataFrame) -> None:
    """The future-perturbation check must reject a known-leaky feature.

    Without this, a refactor that made the checks vacuous would still pass.
    """
```

The suite found a real leak on its first run: the smoothing prior for target encoding used the fraud rate over the whole training window. It was one number and a tiny effect, but it still depended on the future. It's now an expanding prior.

### Online must equal offline

The second most dangerous bug is an online feature pipeline that quietly disagrees with the offline one. So the per-customer feature code is written **once** and called by both the batch pipeline and the scoring service, and a test checks they match.

That test forced the biggest design decision. The first version used lifetime statistics (every merchant ever seen), which is easy offline and impossible online, because an authorisation request can't carry unlimited history. So every history feature is now **bounded at 31 prior transactions**. The service asks for 62, so the oldest position in the window has the same history behind it as during training.

### Not fooling myself about cost

An earlier version reported that the sequence models cut expected cost by **27%**. Nothing leaked and the comparison was even fair, but it calibrated and thresholded the **average of three seeds' scores**. That's an ensemble, and ensembling helps noisy neural networks far more than stable trees. The deployed artefact is **one checkpoint**, so the pipeline now does calibration, thresholding and costing **per seed**. The honest number is much smaller.

## Results (3 seeds, synthetic)

**Ranking:** the best GRU reaches PR-AUC **0.870 ± 0.014** vs LightGBM **0.826 ± 0.001**, about three standard deviations apart.

**The real finding is when that advantage appears:**

| | Pre-drift PR-AUC | Post-drift PR-AUC |
|---|---|---|
| GRU (bce) | 0.950 | **0.765** |
| LightGBM | **0.956** | 0.615 |

Before the drift, every model sits between 0.943 and 0.956, and LightGBM is at the top. After the drift, LightGBM loses the most. It leaned hardest on the device fingerprint and merchant risk rates, which are exactly what the attacker neutralised. The sequence models still see that three transactions came from the same unfamiliar device inside four minutes, a *relational* fact that survives the attacker changing the values.

Had my test period ended before the drift, I'd have concluded sequence models aren't worth it, and I'd have been right on that evidence. Fraud is adversarial, so a stationary holdout measures the one regime guaranteed not to last.

**By mechanism:** card testing recall 0.92 (GRU) vs 0.84 (LightGBM), the biggest win, where the ordering *is* the evidence. On merchant compromise, LightGBM is slightly ahead.

**Cost per transaction:** 0.0799 ± 0.0122 vs 0.0860 ± 0.0035. That's a 7% gap, **smaller than one standard deviation**. On three seeds, I can't claim a saving. A better ranker isn't automatically cheaper, because PR-AUC covers every threshold while cost is judged at one threshold picked from a few hundred validation frauds.

**Smaller won:** the GRU (78k parameters, ~2 minutes to train) beat the Transformer (106k, ~5 minutes). With about 1,100 fraud examples, attention has capacity but not enough signal to spend it on.

**Calibration surprised me twice.** The neural networks were *under*-confident, not over-confident, and only with focal loss (alpha 0.25 shrinks positive probabilities). And weighted sampling didn't wreck calibration. It hurt *ranking* instead. Isotonic regression fixed calibration without costing any ranking.

## Serving and monitoring

The GRU is exported to ONNX and served by FastAPI. Measured p99 end-to-end latency is **7.2 ms against a 50 ms budget**. ONNX Runtime was 2.8× faster than eager PyTorch on the forward pass, and **building the features took longer than running the model**. Every decision is written to an audit log before the response.

For monitoring, the standard drift measure (score PSI) **never fired**. It peaked at 0.0011, because over 99% of traffic is legitimate and didn't change. The **alert rate at the fixed threshold** caught it in the first window with drifted traffic, dropping to 57% of its usual level. Fraud drift is targeted, abrupt, and labelled too late, so the alarm has to come from what's visible at authorisation time.

## What I learned

- **Build the baseline to win.** Losing to a tree is a fine result if the tree was strong.
- **Test what happens when the world changes.** The whole finding lives after the drift.
- **Evaluate what you deploy.** The 27% "saving" was an ensemble nobody deployed.
- **Behavioural leakage tests beat code review**, and they need a test that proves they can fail.
- **Measure the whole latency path.** The model is rarely the slow part.

## What would change the conclusion

More seeds (10 would settle the cost question), more fraud examples, a drift that attacks *sequence* structure, longer history via a feature store, graph models for merchant compromise, and evaluating per attack episode rather than per transaction.

*Note: these numbers come from the project's README. The results files aren't committed, so re-running `make experiment` is the way to check them.*
