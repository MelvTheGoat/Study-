# Fantasy-Premier-League: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

For each skill: what it is in simple words, where it shows up in this project, and the related things I should also be able to explain so I'm not caught out.

---

## 1. Integer linear programming (optimisation)

**Simple explanation:** you describe a choice as yes/no variables, write the rules as linear inequalities, and tell a solver what to maximise. The solver finds the best answer that breaks no rule.

**In this project:** `optimise/squad.py`. Binary `squad`, `start` and `captain` variables per player, FPL rules as constraints, points as the objective, solved with PuLP + CBC.

**Also be ready to explain:**
- **LP vs ILP:** LP allows fractions and is fast. ILP forces whole numbers and is harder (NP-hard in general), solved by branch and bound.
- **Branch and bound:** solve the relaxed LP, split on a fractional variable, and prune branches that can't beat the best answer so far.
- **Why greedy fails** when constraints interact (the knapsack problem is the classic example).
- **Feasibility vs optimality:** what "Infeasible" means (`InfeasibleSquad`) and why a time limit can return a non-optimal answer.
- **Linking constraints:** `start ≤ squad`, `captain ≤ start`.
- **Other solvers:** CBC (free), HiGHS (free, fast), Gurobi/CPLEX (commercial). OR-Tools as an alternative.

---

## 2. Bayesian-style shrinkage (small-sample estimation)

**Simple explanation:** when you have little data, pull your estimate toward a sensible average. The more data you get, the less you pull.

**In this project:** `player_rates._shrink`: weight = minutes / (minutes + 450), blended with a price-scaled positional prior. Team strength does the same toward 1.45 goals per match.

**Also be ready to explain:**
- **Empirical Bayes** and the **beta-binomial** idea: prior "pseudo-counts" plus observed counts.
- **Regression to the mean:** why hot starts cool off.
- **Bias-variance trade-off:** shrinkage adds a little bias to cut a lot of variance.
- **Choosing the prior strength** (450 minutes here): ideally tuned by backtest, not guessed.
- **Why price makes a good prior:** it's FPL's own expert guess at output.

---

## 3. Probability models for sport (Poisson)

**Simple explanation:** goals are rare, roughly independent events, so the number scored in a match is well described by a Poisson distribution with a given average.

**In this project:** clean-sheet probability = P(0 goals) = e^(−λ). Goals-conceded deductions are summed over P(k goals) for k = 0..8, because the deduction steps every two goals.

**Also be ready to explain:**
- **Why E[f(X)] ≠ f(E[X])** for non-linear f (Jensen's inequality). That's why the bonus curve and the concession deductions aren't applied to averages.
- **Home advantage** as a multiplier on expected goals.
- **Dixon-Coles**, a known improvement on independent Poissons for low scores.
- **Expected goals (xG):** what it measures and why it predicts better than actual goals.

---

## 4. Expected value and decision-making under uncertainty

**Simple explanation:** choose the option with the best average outcome, and add a safety margin when your estimates are shaky.

**In this project:** transfers are compared on expected points over a 5-week horizon with 0.82 decay. A hit must beat 4 + 2 margin. Chips are ranked by how far they clear their own threshold.

**Also be ready to explain:**
- **Discounting future value:** why later gameweeks count less.
- **Option value:** why saving a Wildcard, or rolling a free transfer, can be right.
- **Risk margins:** protecting against model error.
- **Explore vs exploit** (and why FPL is mostly exploit).

---

## 5. Data leakage and time-aware evaluation

**Simple explanation:** a model tested with information it wouldn't have had at the time looks better than it is. You must only use the past to predict the future.

**In this project:** per-gameweek price snapshots, projections keyed by `made_for_gameweek`, and `LockedPicksExist` after the deadline. The documented leak is historical injury flags.

**Also be ready to explain:**
- **Point-in-time data** and **as-of joins**.
- **Walk-forward / rolling-origin backtesting** vs random train/test splits.
- **Target leakage** vs **temporal leakage**.
- **Survivorship bias:** e.g. using today's player list to replay August.

---

## 6. Backend API design (FastAPI)

**Simple explanation:** a small web service that returns JSON at well-named URLs.

**In this project:** read-only endpoints: `/api/gameweeks`, `/api/{model}/gameweek/{gw}`, `/api/{model}/season`, `/api/{model}/gameweek/{gw}/player/{id}`, `/api/summary`, plus `/api/models`, `/api/setup` (what a fresh deployment is doing) and `/api/health` (for the container health check). Unknown `/api/…` paths return a 404, not the site's HTML.

**Also be ready to explain:**
- **Separating reads from writes:** jobs decide, the API only serves.
- **Status codes** (200, 404, 422) and why a silent 200 on an unknown path is a bug.
- **Serving a single-page app** from the same origin (no CORS).
- **Static export** of an API, and when it's enough.

---

## 7. Database design and integrity

**Simple explanation:** shape the tables so wrong data can't be written, rather than trusting code to behave.

**In this project:** SQLite with 17 tables. Composite primary keys (`player_id, made_for_gameweek, target_gameweek`). Locked picks guarded in the repository layer. The schema file explains its reasons in comments.

**Also be ready to explain:**
- **SQLite limits:** one writer at a time, file locking, WAL mode.
- **When to move to Postgres.**
- **Snapshot tables** (prices per gameweek) vs overwriting.
- **Idempotent jobs:** re-running changes nothing.

---

## 8. Scheduling, jobs and reliability

**Simple explanation:** make background work safe to repeat and safe to miss.

**In this project:** stateless `due_work(db, now)`, re-derived up to 3 passes per tick, an 8-hour pick window, a watch loop with a 4-hour budget, and stale-score detection.

**Also be ready to explain:**
- **Idempotency** and **at-least-once** execution.
- **Level-triggered vs edge-triggered** designs ("what is owed now?" vs "react to an event").
- **Injecting clocks** for testable time logic.
- **Cron limits** on shared runners, and alternatives (queues, Temporal, Airflow).

---

## 9. CI/CD and low-cost deployment

**Simple explanation:** let automated pipelines build, run and publish the project.

**In this project:** GitHub Actions cron, a `concurrency` group, DB stored gzipped on a force-pushed branch, a 30-day artifact backup, a static site on Pages, plus a Docker multi-stage build and a Render blueprint.

**Also be ready to explain:**
- **Multi-stage Docker builds:** Node builds the frontend, the slim Python image runs it.
- **Health checks** and persistent volumes.
- **Why force-pushing a binary** keeps a repo small, and what you lose.
- **Secrets and permissions** in workflows (`contents: write`, `pages: write`).

---

## 10. Testing strategy

**Simple explanation:** test the most where a silent mistake costs the most.

**In this project:** 610 tests. Real recorded API fixtures (43 players at spread-out prices so the budget binds). Synthetic blank and double gameweeks. Tests that fail on purpose if the API changes. A before/after publish test for pruning. An end-to-end seed test.

**Also be ready to explain:**
- **Unit vs integration vs end-to-end tests.**
- **Recorded fixtures vs mocks.**
- **Why a test that "looked obviously true" can be wrong** (the forward-formation example in the docs).
- **What coverage doesn't tell you:** the pieces were tested, but the published result wasn't, until the seed test.

---

## 11. Frontend basics (React + Vite)

**Simple explanation:** a component-based web page that fetches JSON and draws it.

**In this project:** React 18 components for the pitch, players, score card, transfers, season chart and player sheet. Vite for dev (with an `/api` proxy) and builds. `VITE_BASE` and `VITE_API_SUFFIX` let the same code run on Pages.

**Also be ready to explain:**
- **State and props**, fetching with `useEffect`.
- **Same-origin vs CORS.**
- **Static hosting** and base paths.

---

## 12. Domain modelling (turning rules into code)

**Simple explanation:** write real-world rules as precise, named, tested code.

**In this project:** every FPL rule is a named constant with its source (`rules/constants.py`, `docs/rules-sources.md`). Selling price keeps half of any rise, rounded down. Auto-subs walk the bench in order and keep a legal formation. The one uncertain rule is isolated in `CHIP_GAMEWEEK_ACCRUES_FREE_TRANSFER`.

**Also be ready to explain:**
- **Integer money** (prices in tenths of a million) to avoid floating-point errors.
- **Keeping pure logic apart from I/O.**
- **Verifying assumptions against the source** (the API's own settings), not blog posts.
