# What does it cost to be stuck with last week's team? I built two FPL bots to find out

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

## Why I built it

Every Fantasy Premier League manager lives with their past. The striker you bought in August is still in your team in October. You wanted three changes but only had one free transfer. You took a −4 hit and it didn't pay off.

All of that has a cost, but you can never see it, because you only ever play one team. There's no second version of you who got to start fresh each week.

So I built both versions. Two bots play the 2026/27 season side by side, using the exact same predictions:

- **The Manager** keeps one squad all season under the real rules: one free transfer a week, −4 points for each extra, a £100m budget with FPL's real selling prices, and two sets of chips.
- **Best XI of the Week** picks the best legal squad from scratch every gameweek. Same £100m, same 2/5/5/3 shape, same three-per-club limit, but no memory, no transfers, no chips.

They aren't competing. Best XI has every advantage and should win. The gap between them *is* the result: it's a measurement of what continuity costs. Both are scored with real FPL points from the API and compared with the official gameweek average, the same number every human manager is judged against.

## The problem, broken down

To do this properly I needed five things:

1. **The rules, exactly right.** Squad shapes, formations, auto-subs, captaincy, selling prices, free transfers, hits, chip windows, scoring. If one rule is wrong, every number after it is quietly wrong.
2. **A prediction** of how many points each of ~667 players will score in each of the next five gameweeks.
3. **A squad picker** that finds the best legal team within the budget.
4. **A strategy** for the Manager: when to transfer, when to take a hit, when to play a chip.
5. **A way to run it all season** without paying for a server and without ever missing a deadline.

I built them in that order: rules, then data, then prediction, then optimiser, then strategy, then the website, then deployment. The rules are facts, and everything above them is opinion. A wrong opinion costs points. A wrong fact makes every number meaningless.

## How it works

### Rules first, with no I/O

The `rules/` package is pure logic. It never touches the database or the network, which means every rule can be tested on its own. It has the most tests of any part of the project.

I checked each rule against the FPL API, which publishes the game's own settings. Some of my assumptions didn't survive that check. For 2026/27 a goalkeeper goal is worth 10 points, transfer chips start in GW2 not GW1, and FPL stopped publishing team attack and defence ratings. The scoring table is now read from the API instead of hard-coded, so the next rule change needs no code change.

### Predicting points from parts

I didn't train one big model. Three gameweeks into a season there isn't enough data to fit something that beats a sensible hand-built model. Parts are also easier to explain. "7.9 projected, home to Coventry, on penalties" is a sentence a person understands. One number out of a black box isn't.

So expected points are a sum: appearance, goals, assists, clean sheet, goals conceded, saves, defensive contribution, bonus and cards.

The key trick is **shrinkage**. A midfielder with one goal from 180 minutes has a raw rate of 0.5 goals per 90. That would make him the best player in the game. So every rate is pulled toward a prior based on position *and price* (price is FPL's own guess at a player's output), and the pull weakens as minutes build up:

```python
own_rate = pooled_total / pooled_minutes * 90.0
weight = pooled_minutes / (pooled_minutes + RATE_SHRINKAGE_MINUTES)
return weight * own_rate + (1 - weight) * prior_per_90
```

`RATE_SHRINKAGE_MINUTES` is 450, about five full matches. After that, a player's own record counts as much as the prior.

Team strength had to be rebuilt too, since FPL zeroed the ratings. I blend fixture difficulty (available from day one) with actual results (better as the season goes on). Clean-sheet chances come from a Poisson model on expected goals conceded.

### Picking the squad with an ILP

Picking greedily doesn't work. Take the best forward and you may not have money left for a fifth defender. The three-per-club limit bites exactly where the best players cluster. So the picker is an integer linear program (using PuLP and the CBC solver). Every player gets three yes/no variables: in the squad, starting, captain. The constraints are the literal FPL rules. The objective counts starters fully, the bench at 0.12, and the captain twice.

The bench weight matters. At zero, the solver buys four £4.0m players who never play. That's fine for one week and a disaster the moment someone gets injured.

### The Manager's choices

Each week the Manager asks the optimiser: "What's the best squad I can reach with 0 transfers? 1? 2? ... 5?" Then it takes whichever leaves the most points after hits. A hit has to beat four points *plus* a margin, because the predictions are often wrong:

```python
# A hit has to clear four points *plus* a margin for the projection
# being wrong, which it routinely is.
if outcome.points_cost and baseline is not None:
    if selection.objective - baseline < outcome.points_cost + hit_margin:
        continue
```

Rolling the transfer is a real answer, and often the right one.

## The hard parts, and how I solved them

### 1. No hindsight, enforced by the database

The season had already started when I built this, so gameweeks 1 onward had to be rebuilt. The rule: **a gameweek can only use data that existed before its own deadline.** I didn't want that to depend on me being careful, so the database enforces it:

- Prices are stored per gameweek, so a GW7 decision uses GW7 prices.
- Projections are keyed by which deadline they were made before.
- Once a deadline has passed, locked picks can't be rewritten:

```python
if picks_are_locked(connection, model_id, gameweek):
    moment = now or datetime.now(UTC)
    when = deadline(connection, gameweek)
    if when is None or moment >= when:
        raise LockedPicksExist(...)
```

Before the deadline a squad can be refined as team news comes in, just like a human would. After it, never.

There is one honest gap. The API doesn't publish the history of injury flags, so a squad rebuilt for GW1 today knows about injuries announced later. Prices, results and projections are time-boxed. Team news isn't.

### 2. Chips that fired in week one

My first version used one shared threshold of 6 points for every chip. It played Triple Captain in GW3 and a Wildcard in GW2. That's clearly wrong, but it wasn't obvious in advance. A top captain scores six to eight points in a *normal* week, so a six-point bar fires straight away. Each chip needs its own bar, set where *its* good week starts:

```python
CHIP_THRESHOLDS: dict[Chip, float] = {
    Chip.BENCH_BOOST: 22.0,
    Chip.TRIPLE_CAPTAIN: 12.0,
    Chip.FREE_HIT: 18.0,
    Chip.WILDCARD: 30.0,
}
```

These moved again when I fixed the minutes model, because a threshold only means something relative to the model that produces the numbers. The Manager also can't choose a Wildcard before GW5, when the predictions are still mostly priors.

### 3. Running free, on a clock I don't control

The site is read-only, so it doesn't need a server. It only needs something that thinks a few times a day and leaves files behind. That's GitHub Actions. The workflow runs the jobs, stores the database, exports every API response as a static JSON file, and publishes to GitHub Pages.

The catch is that GitHub's cron is best-effort. I asked for every 30 minutes and saw gaps of two to seven hours. A two-hour pick window was missed about two times in five. So:

- The pick window now opens **8 hours** before the deadline.
- A run that lands inside the window **stays alive** (up to 4 hours) and keeps refining until the deadline.
- The scheduler keeps **no memory**. Every tick it asks the database "what is owed right now?" A restart, a missed run or a week offline all resolve the same way.

The season's SQLite file lives on a `season-data` branch, force-pushed as one commit each run. Keeping every half-hourly version of a binary file would add gigabytes. Each run also uploads the file as a 30-day backup.

### 4. Bugs that looked like results

Four bugs reached the live site, and they all had the same shape: **they gave believable output instead of an error.**

- The first deploy built a whole season and lost it. The database path pointed inside `site-packages`, which the runner throws away.
- The site showed GW1–3 as final at **zero** points. Nothing in the setup path fetched the real results, and scoring a squad against no results doesn't fail. It just returns zero.
- The fix for that fetched the results, and then the next line skipped the scoring the results were fetched for.
- An unknown API path returned the website's HTML with a 200.

What came out of it: stale-score detection (re-score when results are newer than the score), setup stages that refuse to publish squads without results, and an end-to-end seed test. The pieces had been tested. The thing people actually see had not.

## Where it stands

From the season database on 28 September 2026, after five gameweeks:

| | Manager | Best XI |
|---|---|---|
| Points | 268 | 322 |
| Beat the average | 1 of 5 | 2 of 5 |

That's a 54-point gap. Five weeks is far too few to mean much, and neither model is beating the average reliably yet. I'd rather say that plainly than explain it away. Note too that the first deploy lost its database, so GW1–3 were re-decided later, using only pre-deadline data.

The test suite has 610 tests, and all of them run with no network.

## What I learned

- **Put the rules first, and make them pure.** Keeping them away from the database made them easy to test.
- **Make the database refuse the wrong thing.** A rule enforced by the schema beats one enforced by discipline.
- **Silent wrong answers are worse than crashes.** I now check the output people see, not just each piece.
- **Thresholds belong to a model.** Change the model and every threshold on top of it needs checking.
- **Stateless schedulers are easy to trust.** "What is owed right now?" survives anything.

## What's next

- **A real backtest** on a past season. Right now the model's accuracy is not measured. The weights are reasoned, not fitted.
- **Record team news every gameweek**, to close the one leak for future seasons.
- **Test blank and double gameweeks on real fixtures** once they're announced.
- **Validate the chip thresholds** over a full season.
- **Effective ownership**, so the Manager can play *against* the field, not just measure itself against it.
