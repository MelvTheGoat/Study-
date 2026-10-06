# FPL AI Manager: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. So you know Fantasy Premier League. You pick 15 real players, they score points in real matches, and every week you can make one free transfer. Any extra transfer costs you 4 points.

Every FPL manager is stuck with last week's decisions. You can't just pick the perfect team every week. That costs you points, but nobody knows how many, because you only ever play one team.

So here's our project. We build two robot teams that play the whole 2026/27 season using exactly the same predictions.

**The Manager** plays by the real rules. One free transfer, −4 for each extra, chips that expire. **Best XI** cheats on purpose. It picks a brand-new perfect squad every single week.

Because both use the same predictions, the only difference is the rules. So the gap between their scores tells us what being stuck really costs. That's the whole idea. Keep it in your head, because everything we build serves it.

## So what do we need?

Let's list what a project like this needs before we touch anything.

1. **The rules of the game**, written as exact code. If one rule is wrong, every number after it is wrong.
2. **Data** from FPL itself: players, prices, fixtures, live points.
3. **A place to keep it**, a database.
4. **A way to predict points** for every player.
5. **A way to pick the best legal squad.** That's harder than it sounds.
6. **The two teams**, each with its own way of deciding.
7. **Jobs** that do each piece of work at the right moment: refresh data, pick before the deadline, score after.
8. **Something that decides what's due**, and runs by itself.
9. **A website** to show it all.
10. **A cheap way to host it**, ideally free.

And one big rule over all of it: **no hindsight**. A decision for gameweek 3 can only use what was known before gameweek 3's deadline. Hold onto that, it comes up again and again.

## Step zero: set up the workshop

First we set up the project. A `README.md` at the top, the front page, saying what this is and how to run it. A `.gitignore` telling git which files not to save.

The code lives in a `backend` folder, inside a package called `fplai`. Each folder gets a small `__init__.py` file, which tells Python "this folder is a package".

Then `backend/pyproject.toml`. That's where we say what the project needs to install.

And here's a choice worth noticing: the back end only needs **four** outside tools to run. FastAPI, Uvicorn, httpx and PuLP. No pandas, no NumPy, no machine learning libraries.

Why so few? Because the points model, as you'll see, is simple arithmetic. It doesn't need heavy tools. Fewer tools means less to install and less to break.

And `backend/fplai/config.py` holds the settings, like where the data lives and the hit margin. Every setting can be changed with an environment variable, without touching the code.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `backend/pyproject.toml`: the project's details and its four install needs.
> - `backend/fplai/__init__.py`, plus an `__init__.py` in each sub-folder (`api`, `data`, `jobs`, `model`, `optimise`, `rules`, `strategy`): mark folders as Python packages.
> - `backend/fplai/config.py`: the settings, all changeable from outside.

## Step one: write the rules first

Now, here's an interesting choice. We don't start with data. We start with the **rules**.

Why? Because rules are facts. A squad must have 2 goalkeepers, 5 defenders, 5 midfielders and 3 forwards. No more than 3 from one club.

Get any rule wrong, and every score after it is wrong, with no error message to warn you. So the rules come first, and they get the most tests.

And they live in a folder called `rules`, with a strict design: **pure logic, no data access**. The rules never touch the database or the internet. That means we can test every rule with tiny made-up squads, instantly.

Here's how it breaks down. `backend/fplai/rules/types.py` defines the basic building blocks, like a player, a squad and a lineup. They're deliberately plain. `backend/fplai/rules/constants.py` holds the game's fixed numbers for 2026/27, like the £100m budget and the 2/5/5/3 shape.

Then one file per rule:

- `rules/squad.py`: is this a legal squad of 15, and is this a legal starting 11?
- `rules/pricing.py`: selling prices. You keep half of any price rise, rounded down, but you take a fall in full.
- `rules/transfers.py`: one free transfer each week, saved up to five, and 4 points for each extra.
- `rules/chips.py`: the chips. This season has two full sets, and the first set expires at the gameweek 19 deadline.
- `rules/captaincy.py`: your captain scores double, or triple with Triple Captain. If the captain doesn't play, the vice-captain takes over.
- `rules/autosubs.py`: automatic substitutions. If a starter plays zero minutes, the first legal bench player comes on. A goalkeeper can only be replaced by the other goalkeeper.

And two files for scoring. `rules/scoring_table.py` reads the points for each action straight from FPL, instead of typing them in. That turned out to matter: this season, a goalkeeper's goal went up to 10 points. `rules/scoring.py` takes the real points from FPL and applies the manager-level rules on top, like captains, subs and hits.

Where did we check each rule? In `docs/rules-sources.md`. It lists every rule and where it was confirmed, mostly straight from FPL's own settings.

And the tests, one file per rule. They all share little helpers in `backend/tests/conftest.py`, which build tiny made-up squads. Then there's `test_squad_rules.py`, `test_pricing.py`, `test_transfers.py`, `test_chips.py`, `test_captaincy.py`, `test_autosubs.py` and `test_scoring.py`. Plus `test_season_integration.py`, which plays several gameweeks in a row, to catch rules that only go wrong in combination.

> **📁 Files we just created**
> - `backend/fplai/rules/types.py`: the basic building blocks.
> - `backend/fplai/rules/constants.py`: the game's fixed numbers.
> - `backend/fplai/rules/squad.py`: legal squads and formations.
> - `backend/fplai/rules/pricing.py`: buying and selling prices.
> - `backend/fplai/rules/transfers.py`: free transfers and hits.
> - `backend/fplai/rules/chips.py`: the two chip sets.
> - `backend/fplai/rules/captaincy.py`: captain and vice-captain.
> - `backend/fplai/rules/autosubs.py`: automatic substitutions.
> - `backend/fplai/rules/scoring_table.py`: points per action, read from FPL.
> - `backend/fplai/rules/scoring.py`: a team's final score for a gameweek.
> - `docs/rules-sources.md`: where every rule was checked.
> - `backend/tests/__init__.py` and `backend/tests/conftest.py`: test setup and helpers.
> - `backend/tests/test_squad_rules.py`, `test_pricing.py`, `test_transfers.py`, `test_chips.py`, `test_captaincy.py`, `test_autosubs.py`, `test_scoring.py`: one test file per rule.
> - `backend/tests/test_season_integration.py`: rules working together over several weeks.

## Step two: get the data, politely

Now we get the data. It all comes from FPL's public API, which is a way for programs to ask FPL for data. It's free and needs no login.

But free and open means we have to be polite. So we write `backend/fplai/data/client.py`. It waits at least 1 second between requests, saves every answer to disk, and retries when FPL's servers have a hiccup.

It never retries a request FPL rejected, because that would never work. Saving answers means re-running anything costs nothing.

Then we design the database, in `backend/fplai/data/schema.sql`. It's SQLite, one file, with 17 tables: players, prices for each gameweek, fixtures, player stats, predictions, locked picks, transfers, chips, results and more. The whole season fits in a few megabytes.

`backend/fplai/data/db.py` is the thin layer that opens the database. `backend/fplai/data/ingest.py` turns FPL's answers into rows in those tables. It also spots blank and double gameweeks, where a team plays zero or two times.

Now here's an important file: `backend/fplai/data/repository.py`. It's the *only* place that knows both the database and the rules. It turns database rows into the rules' building blocks. That keeps the rules pure, like we promised in step one.

And `repository.py` holds the honesty lock. Once a gameweek's deadline has passed, saving new picks for it is refused. Here's the real check:

```python
if when is None or moment >= when:
    raise LockedPicksExist(
```

If there's no deadline recorded, or the deadline has passed, it raises an error. So a past decision can never quietly "improve" with hindsight.

One small extra: `backend/fplai/data/assets.py` works out the web address of each club's shirt picture. FPL keys shirts by a club "code", not its normal ID, so this file gets that right.

## Testing with real FPL answers

How do we test all this without hitting FPL every time? We record real answers once.

`backend/tests/fixtures/record_fixtures.py` saves real FPL responses, trimmed down to 43 players so they stay small. Those are `bootstrap_static.json` (players, teams and settings), `fixtures.json` (the fixture list), `event_3_live.json` (live points for gameweek 3) and `recording_meta.json` (when and how they were recorded).

But some situations hadn't happened yet when the code was written, like blank and double gameweeks. So `backend/tests/fixtures/build_synthetic_fixtures.py` builds made-up ones. Those are `synthetic_bootstrap.json`, `synthetic_bootstrap_gw5.json`, `synthetic_fixtures.json`, `synthetic_event_3_live.json` and `synthetic_event_4_live.json`.

And the data tests: `test_client.py` checks the polite waiting, saving and retries, without ever touching the internet. `test_ingest.py` checks the recorded answers turn into the right rows. `test_repository.py` checks reading back and locking picks. `test_assets.py` checks the shirt addresses.

So let's pause. The rules are written and tested. The data comes in politely and is stored safely.

And past picks can't be changed. Good, that's the foundation. Now we can start predicting.

> **📁 Files we just created**
> - `backend/fplai/data/client.py`: the polite, caching FPL client.
> - `backend/fplai/data/schema.sql`: the 17 database tables.
> - `backend/fplai/data/db.py`: opens the database.
> - `backend/fplai/data/ingest.py`: turns FPL answers into rows.
> - `backend/fplai/data/repository.py`: turns rows into rule types, and locks picks after the deadline.
> - `backend/fplai/data/assets.py`: club shirt picture addresses.
> - `backend/tests/fixtures/record_fixtures.py`: records real FPL answers.
> - `backend/tests/fixtures/bootstrap_static.json`, `fixtures.json`, `event_3_live.json`, `recording_meta.json`: the recorded real answers.
> - `backend/tests/fixtures/build_synthetic_fixtures.py`: builds made-up answers for blank and double gameweeks.
> - `backend/tests/fixtures/synthetic_bootstrap.json`, `synthetic_bootstrap_gw5.json`, `synthetic_fixtures.json`, `synthetic_event_3_live.json`, `synthetic_event_4_live.json`: the made-up answers.
> - `backend/tests/test_client.py`, `test_ingest.py`, `test_repository.py`, `test_assets.py`: the data tests.

## Step three: predict each player's points

Now the predictions. We want "expected points": roughly how many points each player will score in each match.

Here's a key decision. We don't train a complicated model. Early in a season, there's far too little data to train one that beats a sensible, hand-built one. So we build it from simple parts, like a recipe.

First, `backend/fplai/model/player_rates.py`. For each player, it works out how often they do things per 90 minutes: goals, assists, saves, defensive actions, bonus points, cards. It also guesses the chance they start.

But there's a catch. Three gameweeks in, a midfielder with one goal from 180 minutes looks like a superstar. So every rate is pulled towards what's normal for a player of that price and position. As they play more minutes, their own numbers count more.

Next, `backend/fplai/model/team_strength.py`. How many goals will each team score and concede in a match? FPL used to publish team strength ratings, but this season they're all zero. So we rebuild them from FPL's fixture difficulty ratings plus real results.

Then `backend/fplai/model/xpts.py` adds it all up for one player in one match.

Will they play? Will they score or assist? Will their team keep a clean sheet? Bonus points?

Every part can be explained on the website, like "likely starter, easy home game, takes penalties".

And `backend/fplai/model/projections.py` runs that for every player, over the next 5 gameweeks, and saves it. And here's the no-hindsight rule again: every read in this file takes "before gameweek N" and only uses data from before that deadline.

Both teams read the *same* saved predictions. That's what makes the experiment fair.

Tests: `test_model.py` checks each part behaves sensibly, like a suspended player getting zero. `test_projections.py` checks the no-hindsight rule directly.

> **📁 Files we just created**
> - `backend/fplai/model/player_rates.py`: per-90 rates, pulled towards normal for the price.
> - `backend/fplai/model/team_strength.py`: goals for and against per match.
> - `backend/fplai/model/xpts.py`: expected points for one player in one match.
> - `backend/fplai/model/projections.py`: predictions for every player, 5 weeks ahead, with no hindsight.
> - `backend/tests/test_model.py` and `backend/tests/test_projections.py`: tests for the model.

## Step four: pick the best legal squad

Now we have a predicted score for every player. Easy, right? Just pick the top 15. No.

Here's the problem. Take the best forward, and you might not have enough money left for a fifth defender. And the "max 3 per club" rule bites exactly where the best players are. Picking one at a time, greedily, breaks the rules or wastes money.

So we write `backend/fplai/optimise/squad.py`. It uses something called integer linear programming, through a free tool called PuLP. You describe the goal, "get the most points", and all the rules, and it finds the true best answer. Here's the budget rule, written for the solver:

```python
problem += pulp.lpSum(prices[e] * in_squad[e] for e in candidates) <= budget
```

It just says: the prices of everyone in the squad must add up to no more than the budget. Every other rule gets written the same way. It's like packing a suitcase with a weight limit, done perfectly.

One small detail. The bench counts a little, at 0.12 of a starter's points. If the bench counted for nothing, the solver would fill it with the cheapest players, who never play.

Tested in `test_optimiser.py`, which pushes on each rule in turn.

> **📁 Files we just created**
> - `backend/fplai/optimise/squad.py`: finds the best legal squad, starting 11 and captain.
> - `backend/tests/test_optimiser.py`: tests every rule the optimiser must respect.

## Step five: the two teams

Now we build our two contestants.

`backend/fplai/strategy/best_xi.py` is Best XI. It's simple: every gameweek, ask the optimiser for the best squad from scratch. No memory, no transfers, no hits. Like playing a Free Hit every week.

`backend/fplai/strategy/manager.py` is The Manager, and this is where it gets interesting. It carries one squad all season. Each week, it asks: "What's the best squad I can reach with 0, 1, 2, 3, 4 or 5 transfers?" Then it takes off the cost of any hits.

But it's careful. A hit has to be worth more than its 4 points, plus a safety margin of 2. Here's the real check:

```python
if selection.objective - baseline < outcome.points_cost + hit_margin:
```

If the gain is less than the hit cost plus the margin, it doesn't take it. Predictions are uncertain, so it wants to be clearly sure.

It also decides on chips. It values each chip this week and plays the best one only if it beats a set bar. For example, a Wildcard needs to be worth 30 points.

And `backend/fplai/strategy/explain.py` turns the numbers into words. Best XI explains why each player is in. The Manager explains every transfer, hit and chip. So the website can say *why*, not just *what*.

Tested in `test_strategy.py`, which checks things like the Manager refusing a hit that doesn't pay.

> **📁 Files we just created**
> - `backend/fplai/strategy/best_xi.py`: Best XI, a fresh perfect squad each week.
> - `backend/fplai/strategy/manager.py`: The Manager, real rules, hits and chips.
> - `backend/fplai/strategy/explain.py`: turns decisions into readable reasons.
> - `backend/tests/test_strategy.py`: tests both teams' decisions.

Let's circle back. We have the rules, the data, the predictions, the optimiser and both teams. That's the brain, done. Now we need the *timing*: doing the right job at the right moment.

## Step six: the jobs

A gameweek has a rhythm. Before the deadline, refresh prices and pick. During matches, follow the live points. After, close it off and score it.

Each of those is a "job", and each gets its own file in `backend/fplai/jobs/`.

- `refresh.py`: pulls players, teams, prices and fixtures before a deadline, live points during a gameweek, and the final numbers after.
- `results.py`: pulls real points for gameweeks already played, for a database that's catching up.
- `pick.py`: locks one gameweek's picks before its deadline.
- `live.py`: keeps scores current during a gameweek, applying automatic subs as matches finish.
- `score.py`: scores locked picks against what really happened: captains, subs, hits.
- `backfill.py`: replays the season so far, one deadline at a time, using only what was known at each.
- `seed.py`: fills an empty database from scratch, so a fresh setup comes up with the whole season.
- `prune.py`: deletes old predictions for gameweeks already past, because that table grows fastest.
- `reset.py`: throws away every decision, so a season can be replayed. Normally that's exactly what must never happen, so it's a deliberate, separate tool.

Tests: `test_jobs.py` covers the refresh jobs, `test_live.py` the live updates, and `test_backfill_and_scoring.py` checks that replaying gameweek 7 never sees gameweek 7. `test_seed.py` runs a whole seed from nothing.

> **📁 Files we just created**
> - `backend/fplai/jobs/refresh.py`, `results.py`, `pick.py`, `live.py`, `score.py`, `backfill.py`, `seed.py`, `prune.py`, `reset.py`: one job each.
> - `backend/tests/test_jobs.py`, `test_live.py`, `test_backfill_and_scoring.py`, `test_seed.py`: tests for the jobs.

## Step seven: the scheduler, which decides what's due

Now, who decides which job to run, and when? That's `backend/fplai/jobs/scheduler.py`. And here's the clever bit: it has **no memory**.

Every time it runs, it just looks at the database and the clock and asks, "What's owed right now?" Here's the start of that function:

```python
def due_work(
    connection: sqlite3.Connection, now: datetime | None = None
) -> list[Job]:
    """Everything the database says is owed, at this moment.
```

If the database is empty, the answer is "seed". If the data is more than 3 hours old, "refresh". If a deadline is between 8 hours and 2 minutes away, "pick". And so on.

Why no memory? Because if a run crashes or gets skipped, nothing is lost. The work is simply still owed next time. It's a to-do list that rewrites itself.

And `backend/fplai/cli.py` turns every job into a command you can type by hand, like `fplai pick` or `fplai live`. That makes every step easy to run and debug yourself.

Tested in `test_scheduler.py`. The clock can be set to any moment, so timing rules are tested without real waiting.

> **📁 Files we just created**
> - `backend/fplai/jobs/scheduler.py`: decides what's due, with no memory.
> - `backend/fplai/cli.py`: every job as a typed command.
> - `backend/tests/test_scheduler.py`: tests the timing rules.

## Step eight: the API the website reads

Now we need to share results with a website. `backend/fplai/api/app.py` is a small, read-only API built with FastAPI. It only serves what the jobs already decided. It never predicts or picks anything.

Why? So a slow web request can never delay a deadline. The thinking and the showing are kept apart.

`backend/fplai/api/views.py` shapes the stored results into exactly what the website draws: who started, who came off the bench, the shirt picture, whether the score is final. So the website doesn't need to know any rules at all.

Tested in `test_api.py`.

> **📁 Files we just created**
> - `backend/fplai/api/app.py`: the read-only API.
> - `backend/fplai/api/views.py`: shapes results for the website.
> - `backend/tests/test_api.py`: tests what the API returns.

## Step nine: the website

The website lives in a `frontend` folder. It's built with React, a popular tool for building web pages out of pieces, and Vite, which bundles them up.

The setup files: `frontend/package.json` lists what to install, and `frontend/package-lock.json` pins the exact versions. `frontend/vite.config.js` holds the build settings. `frontend/index.html` is the empty page everything loads into. `frontend/.gitignore` stops the build output being saved.

Then the code. `frontend/src/main.jsx` starts the app. `frontend/src/App.jsx` is the main page, with a tab for each team and a gameweek picker.

`frontend/src/api.js` holds every call the website makes, all in one place. `frontend/src/styles.css` is the look.

And the pieces, in `frontend/src/components/`:

- `Pitch.jsx`: the half-pitch view of the squad.
- `Player.jsx`: one player on the pitch.
- `PlayerSheet.jsx`: the pop-up with a player's details.
- `ScoreCard.jsx`: the headline points, and how they compare with the official FPL average.
- `Transfers.jsx`: every transfer, with the numbers behind it.
- `Season.jsx`: the season chart.
- `Setup.jsx`: a "setting up" screen for a fresh install, showing progress while the season downloads and replays.

> **📁 Files we just created**
> - `frontend/package.json` and `frontend/package-lock.json`: what to install, with exact versions.
> - `frontend/vite.config.js`: build settings.
> - `frontend/index.html`: the page the app loads into.
> - `frontend/.gitignore`: stops build output being saved.
> - `frontend/src/main.jsx`: starts the app.
> - `frontend/src/App.jsx`: the main page.
> - `frontend/src/api.js`: every data call, in one place.
> - `frontend/src/styles.css`: the look.
> - `frontend/src/components/Pitch.jsx`, `Player.jsx`, `PlayerSheet.jsx`, `ScoreCard.jsx`, `Transfers.jsx`, `Season.jsx`, `Setup.jsx`: the page pieces.

## Step ten: no server needed

Now, here's a neat trick. Nobody types into this website. It only shows results. So why pay for a server running all day?

`backend/fplai/jobs/export.py` visits every page address the website could ever ask for, through the real API, and saves each answer as a file. Then the website can be hosted as plain files, for free, on GitHub Pages.

And because it goes through the *real* API, the saved files can never drift from what the API would say.

Tested in `test_export.py`. There's even a test that saves the whole site before and after pruning old data, and checks every file is identical. So pruning can never quietly break the site.

> **📁 Files we just created**
> - `backend/fplai/jobs/export.py`: saves every page's data as a file.
> - `backend/tests/test_export.py`: checks the export, and that pruning changes nothing visible.

## Step eleven: make it run by itself

Now the final piece. `.github/workflows/season.yml` tells GitHub Actions, a free service that runs code at set times, to run at 7 and 37 minutes past every hour.

Each run does this, in order:

1. Restore the database from a separate branch called `season-data`.
2. Run the scheduler, which does whatever is due.
3. Prune old predictions.
4. Save the database back to `season-data`, plus a 30-day backup copy.
5. Export every page, build the website, and publish it to GitHub Pages.

Only one run can happen at a time, because SQLite allows only one writer.

But GitHub's timer is best-effort. The project's notes record gaps of 2 to 7 hours between runs. That's why picking starts 8 hours before a deadline, and a run inside that window stays awake, up to 4 hours, until the deadline is safe.

There's also another way to run it: as an always-on server. `Dockerfile` packs everything into one box: it builds the website in a throwaway step, then copies it next to the back end. `.dockerignore` says what to leave out of the box.

And `render.yaml` sets it up on Render, a hosting service, with a disk to keep the database. It's a paid tier, because the free one has no disk and sleeps.

> **📁 Files we just created**
> - `.github/workflows/season.yml`: the robot that runs every 30 minutes.
> - `Dockerfile`: packs the whole app into one box.
> - `.dockerignore`: what to leave out of the box.
> - `render.yaml`: an optional always-on setup on Render.

## The last file: writing it all down

Last, `docs/how-it-works.md` explains the whole design in words.

One honest note: it's a little behind the code. It says 574 tests, while 610 pass now. It also lists older chip bars and "0 to 3 transfers", while the code uses higher bars and up to 5.

> **📁 Files we just created**
> - `docs/how-it-works.md`: the design, explained in words.

## So, how's it doing?

After 5 gameweeks, Best XI has 322 points and The Manager has 268. That's 54 points, or about 11 points a week. That's the rough cost of being stuck, so far.

Best XI beat the official average 2 times out of 5, and The Manager once.

But be careful. Five weeks is far too few to judge. And gameweeks 1 to 4 were re-picked after a model fix, so only gameweek 5 onwards is a truly live test. The accuracy of the point predictions hasn't been measured yet either.

## What's still missing?

- **No test on a past season**, so the predictions' accuracy isn't proven.
- **The model's settings were reasoned, not learned from data.**
- **Injury news leaks into replayed weeks.** FPL doesn't keep old injury news, so a rebuilt gameweek 1 knows about later injuries.
- **Blank and double gameweeks** are only tested on made-up fixtures.
- **One writer at a time**, so it can't scale up as-is.

## Let's put it all together

So let's look at the whole thing in one breath.

We **set up the workshop** with only four install needs. We wrote **the rules first**, pure and heavily tested, because one wrong rule breaks everything. We fetched **data politely** and stored it, with a **lock** so past picks can't change.

We built **expected points** from simple parts, pulled towards normal early in the season, with no hindsight. We used an **optimiser** to find the best legal squad. We built **two teams**: Best XI, free every week, and The Manager, careful about hits and chips.

Then we built **jobs** for each moment of a gameweek, and a **memory-free scheduler** that just asks "what's due now?". We built a **read-only API**, a **website**, and an **export** so it's all plain files on free hosting. Finally, a **robot** runs it every 30 minutes.

And notice how it links. The no-hindsight rule shows up in the predictions, the replays and the lock. The pure rules from step one get used by the optimiser, the teams and the scoring. Every piece has a job.

That's the project. Two teams, one set of predictions, and an honest measure of what being stuck really costs.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
