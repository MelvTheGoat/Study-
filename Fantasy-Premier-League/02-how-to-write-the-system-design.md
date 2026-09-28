# Fantasy-Premier-League: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

Follow these steps on a whiteboard or in an interview. Each step has the words to say and what to draw, using this project's real details. Aim for about 20–30 minutes in total.

---

## Step 1: Requirements (2–3 min)

Say the problem in one sentence first:
> "Two bots play Fantasy Premier League from the same predictions. One has real limits, one rebuilds every week. The score gap measures what continuity costs."

**Functional requirements**: write these as a list.
1. Pull players, prices, fixtures and live points from the FPL API.
2. Predict expected points for every player for the next 5 gameweeks.
3. Before each deadline, pick a legal squad for both models:
   - **Best XI:** best 15 from scratch, £100m, no transfers or chips.
   - **Manager:** one squad all season, 1 free transfer a week, −4 per extra, two chip sets.
4. After matches, score both with real FPL points (auto-subs, captain, hits) against the official gameweek average.
5. Show squads, reasons, transfers and season totals on a website.

**Non-functional requirements**: these drive the design.
- **No leakage:** a past gameweek may only use data that existed before its deadline.
- **Never miss a deadline.** A missed deadline is a gameweek the Manager sits out.
- **Be polite to the API.** It's public with no login.
- **Cheap.** Ideally free.
- **Read-only site.** Nobody types into it.

**Out of scope:** user accounts, playing real FPL teams, effective ownership.

---

## Step 2: Numbers and scale (2 min)

Write these numbers on the board. They show the system is small, which justifies simple choices.

| Thing | Number | Where it comes from |
|---|---|---|
| Players | ~667 | `players` table in the season DB |
| Clubs | 20 | Premier League |
| Gameweeks | 38 | Season length |
| Planning horizon | 5 gameweeks | `FPLAI_PLANNING_HORIZON` default |
| Squad size | 15 (11 start) | FPL rules |
| API pace | ≥ 1 second between requests | `FPLAI_MIN_REQUEST_INTERVAL` |
| History fetch | 1 request per player, 1–10 minutes | README |
| Optimiser solves per Manager pick | up to 6 (0–5 transfers) + chip checks | `MAX_TRANSFERS_CONSIDERED = 5` |
| Database size | megabytes | README |
| Writers | 1 | one job run at a time |

**Conclusion to say out loud:** "This is tiny. One process, one SQLite file, no server at runtime. The hard parts are correctness and timing, not scale."

---

## Step 3: High-level boxes (3–4 min)

Draw five boxes left to right, then the scheduler above them:

```
                [ Scheduler: what is due right now? ]
                     |        |         |        |
[FPL API] -> [Client + Ingest] -> [SQLite] -> [Model + Optimiser] -> [Locked picks]
                                     |
                                     v
                         [Read-only API] -> [Static export] -> [React site on Pages]
```

Say one line per box:
- **Client + Ingest:** cached, rate-limited, turns JSON into rows.
- **SQLite:** the whole season in one file.
- **Model:** expected points from parts (minutes, goals, assists, clean sheets and so on).
- **Optimiser:** ILP picks the best legal 15 / 11 / captain.
- **Strategies:** Best XI calls it once. Manager compares 0–5 transfers and chips.
- **Scheduler:** stateless. Reads the DB against the clock.
- **API → export → site:** read-only. Exported to static JSON so Pages can host it.

---

## Step 4: Deep dive on each part (10–12 min)

Go in the order the interviewer cares about. If they don't say, use this order.

### 4a. The rules engine (say this first: it's the foundation)
- Pure logic in `rules/`, no database, no network.
- Covers squad shape 2/5/5/3, max 3 per club, formation limits, auto-subs, captain/vice, selling price, free transfers and hits, two chip sets, scoring.
- Scoring values are **read from the API** (`scoring_table.py`), because rules changed in 2026/27 (e.g. goalkeeper goal = 10 points).
- One rule still unconfirmed is kept as one named constant: `CHIP_GAMEWEEK_ACCRUES_FREE_TRANSFER`.

### 4b. The expected-points model
Draw it as a sum:
```
xPts = appearance + goals + assists + clean sheet + goals conceded
     + saves + defensive contribution + bonus + cards
```
- **Minutes:** chance of starting from the last 6 club matches, pooled with prior seasons (capped at 10 matches of weight and fading as this season grows), then multiplied by the API's `chance_of_playing`.
- **Per-90 rates:** shrunk toward a prior scaled by price (`RATE_SHRINKAGE_MINUTES = 450`, so about 5 full games before a player's own rate counts as much as the prior).
- **Team strength:** fixture difficulty blended with real results (half-life 8 matches), shrunk to the league mean of 1.45 goals, home ×1.10 / away ×0.92.
- **Clean sheets:** Poisson on expected goals conceded.
- **Bonus:** a curve from expected bonus point system (BPS) score to expected bonus.

### 4c. The optimiser
Write the ILP on the board:
- Variables per player: `squad[i]`, `start[i]`, `captain[i]` (all 0 or 1).
- Constraints: 15 in squad, 11 start, 1 captain, `start ≤ squad`, `captain ≤ start`, 2/5/5/3, formation min/max, ≤ 3 per club, total price ≤ budget.
- Objective: starters' points + 0.12 × bench points + a small value-per-£m bonus for bench + captain's points again.

### 4d. The Manager's decision
- For k = 0..5: best squad reachable with at most k new players.
- Cost = hits from `resolve_transfers`. A hit only counts if the gain beats **4 + 2** (the hit plus `FPLAI_HIT_MARGIN`).
- Horizon points decay 0.82 per week ahead.
- Chips: each has its own bar (Bench Boost 22, Triple Captain 12, Free Hit 18, Wildcard 30). No voluntary Wildcard before GW5. A chip about to expire is played anyway.

### 4e. No leakage (interviewers love this)
Three guards, all in the schema or repository:
1. `player_prices` keeps a price per gameweek.
2. `projections` is keyed by `(player, made_for_gameweek, target_gameweek)`.
3. `save_locked_picks` refuses to rewrite picks once the deadline has passed (`LockedPicksExist`).

### 4f. Scheduling and deploy
- Stateless `due_work(db, now)` returns a list of jobs. That makes it testable with a fake clock.
- GitHub Actions cron at :07 and :37. The pick window opens 8 hours before the deadline and closes 2 minutes before. A run inside the window stays alive up to 4 hours.
- DB gzipped and force-pushed to `season-data`, with a 30-day artifact backup.

---

## Step 5: Bottlenecks and what breaks first (3 min)

List these in order of risk:
1. **GitHub cron running late.** Observed gaps of 2–7 hours. Fix: 8-hour window plus the watch loop. Backup plan: the backfill repairs a missed gameweek from pre-deadline data.
2. **FPL API changes or blocks.** Fields have already changed this season. Fix: cache, rate limit, tests that fail on purpose if the API changes (e.g. `test_granular_attack_and_defence_ratings_are_no_longer_published`).
3. **Silent wrong numbers.** Four bugs reached the live site, and all gave believable output instead of an error (e.g. a season of zeros). Fix: stale-score detection, setup stages that check results exist, an end-to-end seed test.
4. **SQLite single writer.** Fine now. It breaks as soon as two jobs write at once.

---

## Step 6: Trade-offs (3 min)

Say each as "I chose X over Y because Z, and it costs W":

| Chose | Over | Because | Cost |
|---|---|---|---|
| Hand-built model | Trained ML model | Too little data early in the season, and explainable parts | Not tuned. No backtest yet. |
| ILP | Greedy picking | Constraints interact, so ILP finds the true best | Needs a solver (CBC) |
| SQLite | Postgres | One writer, megabytes, one file to back up | No concurrency |
| GitHub Actions + Pages | Always-on server | Free, and the site is read-only | Best-effort timing |
| Force-pushed DB branch | Full history | Repo would grow by gigabytes | Only the 30-day artifact as backup |
| Read team news live | Store it historically | API doesn't publish history | Backfilled weeks know about later injuries |

Close with the 10x answer: shared projection, Postgres, a job queue, parallel optimiser solves, a real backtest (see `01-system-design.md`).
