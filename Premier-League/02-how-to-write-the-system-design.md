# Premier-League: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Premier-League

---

## Step 1: Requirements (2 min)

One line:
> "Predict every Premier League match each gameweek, retrain after every gameweek, and publish an honest public record."

**Functional**
1. Ingest results and fixtures (PL plus lower divisions and cups).
2. Build features for context: form, table pressure, congestion, manager, availability, squad quality, head-to-head, venue.
3. Predict home/draw/away probabilities and a scoreline.
4. Store every prediction, and never overwrite.
5. Serve a website with gameweek, season and model pages.
6. Update automatically.

**Non-functional**
- **No leakage:** features only from before the gameweek's first kick-off.
- **No hand-made adjustments:** every signal is a feature, and the model weighs it.
- **Calibrated probabilities:** the site shows probabilities, not just picks.
- **Cheap hosting:** read-only serverless site.
- **Self-checking:** detect a stale site.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Matches per gameweek | 10 |
| Matches in DB | ~6,460 (all competitions, 2010-11 on) |
| Features per match | ~200+ (README 204, build doc 213) |
| Training matches (GW6 run) | 6,130 |
| Working DB | ~61 MB |
| Serving DB | ~1.4 MB |
| Backtest | 1,050 matches |
| Daily job | 06:00 UTC, ~1 min if nothing new |

**Say:** "Small data. The challenge is time-correct features and a trustworthy publishing loop."

---

## Step 3: High-level boxes (2 min)

```
[openfootball] [match stats] [squad ratings] [manual CSVs]
          \          |              |             /
                 [Ingest + name normalisation]
                              |
               [Working SQLite: matches, context]
                              |
      [Chronological feature pass: Elo + forward-only club state]
                              |
         [Outcome blend: LightGBM + logistic]   [Dixon-Coles scoreline]
                              |
               [model_runs + predictions (append-only)]
                              |
      [Export read-only web DB] -> [Flask on Vercel] <- [live-site check]
```

---

## Step 4: Deep dive (10 min)

### 4a. Leakage-proof features
- Club state only moves forward in one chronological pass.
- Every match in gameweek N uses state from **before the first kick-off of N**, even a Monday game uses the pre-Saturday state. Train and serve match.

### 4b. The promoted-club problem
- Cross-division Elo (PL, Championship, League One, cups), K=20, home +60, regress 25% between seasons, division offsets.
- The scoreline model fits the Championship too, with a level offset.

### 4c. Outcome model
- LightGBM on all features, early stopping on a chronological tail.
- Multinomial logistic regression on a compact core.
- Serve `0.6·boosted + 0.4·linear`, renormalised. Optimise log loss.
- Draws come out as probability, rarely as the argmax.

### 4d. Scoreline model
- Dixon-Coles: `λ_home = attack_home × defence_away × home_adv`, `λ_away = attack_away × defence_home`.
- Rho correction for 0-0, 1-0, 0-1, 1-1. Time decay `exp(−0.0011·age_days)`. Ridge.
- Show the most likely score **consistent with the predicted outcome**.

### 4e. Lagging sources
- Drop any column filled for <50% of this gameweek's fixtures, and record the drop in `model_runs`.
- Squad rating carried forward, with its **age** as a feature.

### 4f. Publishing loop
- Daily job: ingest → compare with the serving DB → maybe retrain → export → render check → commit.
- Fast-forward other branches (no force).
- `check_publication.py` + `check_live_site.py` fail the job if the site is stale.

---

## Step 5: Bottlenecks (2 min)

1. **Source lag** (stats, ratings): handled by column dropping and rating age.
2. **Missing injuries:** the biggest accuracy gap.
3. **Deploy drift:** the branch rename caused 11 green runs on a stale site. Now checked live.
4. **Full feature rebuild each run:** fine now (under a minute), but won't scale to many leagues.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Features, not rules | Hand-tuned adjustments | Let the data decide | Needs data for weak signals |
| Blend GBM + linear | GBM alone | Calibration early in the season | More moving parts |
| Log loss | Accuracy | Probabilities are the product | Draws never picked |
| Dixon-Coles | Direct score classifier | Goals are counts, a well-known model | Independence assumptions |
| Committed serving DB | Hosted DB | Free, read-only, versioned | Repo holds a binary |
| Daily check | Weekly cron | Irregular fixtures | Tiny daily CI cost |
