# Premier-League: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Premier-League

1. **Predicts every PL match each gameweek (home/draw/away probabilities + scoreline), retrains after each gameweek, and publishes automatically.**
   *Why it matters:* it's a full loop, not a notebook.

2. **No hand-made rules: every contextual signal is a feature (~200+ per match, as home, away and difference).**
   *Why it matters:* the core design principle.

3. **Leakage-proof: one forward-only chronological pass. All matches in GW N use the state before N's first kick-off.**
   *Why it matters:* train and serve see the same information.

4. **Cross-division Elo (PL, Championship, League One, cups): K=20, home +60, 25% regression, division offsets.**
   *Why it matters:* solves the promoted-club cold start.

5. **Outcome = 0.6 LightGBM + 0.4 multinomial logistic regression, optimised for log loss, with early-stopped rounds.**
   *Why it matters:* interactions plus calibration.

6. **Scoreline = Dixon-Coles Poisson with rho, time decay (0.0011/day), ridge, and Championship results fitted too.**
   *Why it matters:* goals are counts, and promoted clubs get sane ratings.

7. **Lagging sources: drop columns <50% populated for the target gameweek, and give squad ratings an age feature.**
   *Why it matters:* the model never leans on evidence it won't have.

8. **Append-only `model_runs` and predictions, history restored on rebuild, late predictions flagged.**
   *Why it matters:* the public record can't be rewritten.

9. **Backtest (README, 1,050 matches): 52.1% vs 43.2% always-home, log loss 0.9948 vs 1.0061 Elo, well calibrated. Live GW1–5: 21/50, mostly backfilled.**
   *Why it matters:* know both numbers and their caveats.

10. **Daily 06:00 UTC Action: skip if unchanged, else retrain, export a 1.4 MB read-only DB, render check, commit, fast-forward deploy branches, and verify the live site. It also syncs managers from Wikidata and records FPL availability daily. 98 tests.**
    *Why it matters:* "don't trust green CI" is a strong story.
