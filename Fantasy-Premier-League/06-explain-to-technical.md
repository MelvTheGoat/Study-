# Fantasy-Premier-League: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

*For another engineer. Simple English, but no skipping the details.*

---

## One-paragraph summary

Two automated FPL teams play the 2026/27 season from one shared projection. **Best XI** solves a fresh squad each gameweek. **The Manager** carries one squad under real transfer, hit, budget and chip rules. The backend is Python 3.11 with four runtime dependencies (FastAPI, Uvicorn, httpx, PuLP). State is one SQLite file. A stateless scheduler, run by GitHub Actions, decides what's due. The site is a React app served as static JSON on GitHub Pages. 610 tests, no network needed.

## Architecture

```
fplai/
  rules/      pure logic: squad, formations, autosubs, captaincy, pricing,
              transfers, chips, scoring, scoring table from the API
  data/       client (cache + rate limit), ingest (JSON -> rows),
              repository (rows -> rule types), schema.sql
  model/      player_rates, team_strength, xpts, projections
  optimise/   squad.py: ILP for squad, XI, captain
  strategy/   best_xi.py, manager.py, explain.py
  jobs/       seed, refresh, results, backfill, pick, live, score,
              export, prune, scheduler
  api/        read-only FastAPI app + view builders
  cli.py      every job as a command
```

**The main boundary:** `rules/` imports nothing from `data/`, and `data/` has no rule logic. So every rule is testable with plain Python objects.

**Deployment:** `.github/workflows/season.yml` runs at `7,37 * * * *`. It restores `fplai.sqlite3.gz` from the `season-data` branch, runs `fplai schedule --watch`, then `fplai prune`, then force-pushes the DB back as a single commit, uploads it as a 30-day artifact, runs `fplai export`, builds the frontend and deploys to Pages. `concurrency: season` stops two runs writing at once. Docker and a Render blueprint are also there, for an always-on host with a `/data` volume.

## Key decisions

| Decision | Why | Cost |
|---|---|---|
| Component model, not trained ML | Too little early-season data. Parts can be explained on the site. | Weights are reasoned, not fitted. No backtest. |
| ILP (PuLP/CBC), not greedy | Budget, positions, club cap and formation interact | Solver dependency. 30 s time limit. |
| SQLite | One writer, megabytes of data, one file to store and back up | Single instance only |
| Stateless scheduler | Restarts and missed runs resolve themselves | Every tick re-reads the DB (cheap here) |
| Static export via the API itself | Pages hosting. The static copy can't drift from the live API. | Re-exported every run |
| DB on a force-pushed branch | No repo bloat from a binary changing every 30 minutes | Only 30 days of artifact backups |
| Scoring table from API | 2026/27 changed values (GK goal = 10) | Depends on API fields staying put |

## The model

### Minutes (`player_rates._playing_time`)
- Recent evidence: starts and appearances over the club's last `RECENT_MATCHES = 6` fixtures.
- Prior-season evidence: minutes share of available minutes ÷ `NAILED_STARTER_SHARE = 0.80`, worth at most `PRIOR_START_MATCHES_CAP = 10` matches, fading as `1 / (1 + own_matches / 3)`.
- Pooled, then pulled toward a squad-player baseline (start 0.35, appear 0.55) with `START_PRIOR_MATCHES = 1`.
- Multiplied by availability: API `chance_of_playing / 100`, otherwise status (`s`, `u`, `n`, `i` → 0, `d` → 0.5).
- Expected minutes = P(start) × 78 + P(cameo) × 22.
- P(60+ min) = P(start) × 0.86 + P(cameo) × 0.08.

### Per-90 rates (`player_rates._shrink`)
- Pools this season with prior seasons at `PRIOR_SEASON_WEIGHT = 0.45`.
- Shrinks toward a positional prior × price factor: `(price / reference_price) ** elasticity`, clamped to [0.35, 3.0]. Elasticities: goals 1.6, assists 1.1, BPS 0.7, saves 0.3, defensive contribution 0.
- Shrinkage weight = minutes / (minutes + 450).
- Uses xG/xA for this season when the API gives them.

### Team strength (`team_strength.py`)
- Prior from fixture difficulty (e.g. difficulty 1 → 2.10 scored / 0.85 conceded, difficulty 5 → 0.90 / 2.20).
- Results blended in by matches played (`BLEND_HALF_LIFE_MATCHES = 8`), shrunk to `LEAGUE_MEAN_GOALS = 1.45` with `SHRINKAGE_MATCHES = 6`.
- Home ×1.10, away ×0.92.

### Points per fixture (`xpts.project_fixture`)
- Goals/assists = rate × minutes share × (team expected goals / 1.45). Penalty takers get an extra 0.13 × 0.79 per match.
- Clean sheet = Poisson P(0 conceded) × P(60+).
- Goals-conceded deduction is summed over the Poisson distribution (0–8 goals), because the deduction steps every 2 goals.
- Bonus = P(60+) × curve(expected BPS). The curve is applied to the per-90 BPS, not a minutes-diluted one, because E[f(x)] ≠ f(E[x]) for a non-linear curve.
- Double gameweeks add fixtures. Blank gameweeks project zero.

## The optimiser (`optimise/squad.py`)

```
maximise   Σ pts[i]·start[i]
         + Σ pts[i]·0.12·(squad[i] − start[i])
         + Σ (pts[i] / price_m[i])·0.6·(squad[i] − start[i])
         + Σ pts[i]·(mult − 1)·captain[i]
subject to Σ squad = 15, Σ start = 11, Σ captain = 1
           start[i] ≤ squad[i], captain[i] ≤ start[i]
           per position: squad count = 2/5/5/3, start count within formation min/max
           per club: Σ squad ≤ 3
           Σ price[i]·squad[i] ≤ budget
           (Manager) Σ squad[i] for i not owned ≤ k
```

The captain ends up as the highest-projected starter. `optimise_lineup` re-solves the XI from a fixed 15. It's used for Bench Boost (every player counts) and Triple Captain (multiplier 3).

## The Manager (`strategy/manager.py`)

- Horizon points = Σ 0.82^offset × xPts over 5 gameweeks.
- For k in 0..5 (`MAX_TRANSFERS_CONSIDERED`): solve with at most k incoming players, priced at **selling price** for owned players (FPL keeps half of any rise, rounded down, and takes falls in full).
- Reject a hit unless `objective_k − objective_0 ≥ hit_cost + FPLAI_HIT_MARGIN (2.0)`.
- Chips: value each available chip this week, rank by margin over its own threshold (BB 22, TC 12, FH 18, WC 30), play the best if it clears. Wildcard is blocked before GW5 unless forced. Near expiry of set 1, the best chip is played anyway.
- Free Hit: the squad carried forward is the pre-chip squad.

## No-leakage guarantees

1. `player_prices` snapshots cost per gameweek. `load_roster_at_gameweek` is the only way models read prices.
2. `projections` primary key includes `made_for_gameweek` and `target_gameweek`.
3. `save_locked_picks` allows rewriting before the deadline and raises `LockedPicksExist` after it, or when no deadline is recorded.
4. Pick window: opens `PICK_LEAD = 8h` before the deadline, closes at `PICK_CUTOFF = 2 min`.

**Known leak:** backfilled gameweeks read current `status` / `chance_of_playing`, because FPL has no history for them.

## Scheduler (`jobs/scheduler.py`)

- `due_work(db, now)` is a pure function of DB + clock. Order: seed (if not ready) → refresh if older than 3h → finalise → catch-up / missing results → stale scores → pick → live.
- `tick` re-derives due work up to `MAX_PASSES = 3`, because a refresh can create a finalisation that wasn't due a moment ago.
- `watch` keeps a run alive (15-minute ticks, 4-hour budget) only while picks are due, so the next publish isn't held back.
- Clock and sleep are injected, so time rules are tested without real waiting.

## How it's tested and evaluated

- **610 tests pass** (`pytest`, ~43 s locally, no network). Biggest areas: model, transfers, squad rules, ingest, chips, strategy, scoring, scheduler, API, optimiser.
- Ingest tests run on **real recorded API responses**, trimmed to 43 players at evenly spaced prices so the £100m budget actually binds.
- Blank and double gameweeks use **synthetic fixtures**, because none existed when the code was written.
- Some tests fail **on purpose** if the world changes, e.g. `test_granular_attack_and_defence_ratings_are_no_longer_published`.
- `test_pruning_changes_nothing_the_site_shows` exports the site before and after pruning and compares every file.
- **Evaluation:** real FPL points vs the official `average_entry_score`, per gameweek. After GW5 (season DB, 28 Sep 2026): Manager 268, Best XI 322. Beat the average: Manager 1/5, Best XI 2/5.
- **Not measured in the repo yet:** projection accuracy (e.g. error of xPts vs actual points), any backtest on a past season, calibration of clean-sheet or start probabilities.

## Known weaknesses

- **No backtest.** Model weights and priors are reasoned. `POSITION_PRIORS` say "derived from typical Premier League seasons, not fitted".
- **Hit test uses the optimiser objective**, which includes bench weight and the bench value term, not pure projected points. So a hit could be justified partly by a better-value bench.
- **Mixed signals in shrinkage:** this season uses xG/xA, but prior-season totals are actual goals/assists.
- **Defensive contribution** probability is `min(rate × minutes share, 1)`, a rough approximation.
- **Chip thresholds** are corrections, not validated.
- **Team-news leak** in backfilled gameweeks.
- **Model chosen after seeing GW1–4.** The live DB's GW1–4 picks were locked on 17 Sep, minutes after the minutes-model fix (`manager_state.locked_at`). The data is time-boxed, but the model version isn't. Only GW5+ is out-of-sample.
- **Docs drift:** README says ~490 tests, `how-it-works.md` says 574 (actual 610). The docs list chip bars 16/10/12/20 and "0–3 transfers", but the code uses 22/12/18/30 and 0–5. The docs also say "two hours before a deadline", but the code uses 8 hours.
- **Dead constant:** `BENCH_SLOT_WEIGHTS` is defined and exported but not used.
- **Single writer:** SQLite plus `concurrency: season`. It can't scale out as-is.
- **Depends on an unofficial, public API** that already changed fields this season.
