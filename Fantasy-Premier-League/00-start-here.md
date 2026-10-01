# FPL AI Manager: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

Fantasy Premier League (FPL) is a free online game. You pick 15 real Premier League players, and you score points from how they play in real matches each week.

Every FPL manager is stuck with last week's choices. You only get one free transfer (player swap) a week, and extra swaps cost points. So you can't just pick the perfect team every week.

That costs you points, but nobody knows how many, because you only ever play one team. This project measures that hidden cost.

## 2. The Big Idea

The project runs two robot teams through the 2026/27 season. Both use exactly the same predictions about how many points each player will score.

**The Manager** plays by the real rules. It keeps its squad week to week, gets one free transfer, and pays 4 points for each extra one.

**Best XI** cheats on purpose. It picks a brand-new perfect squad every single week, with no limits.

Because both teams use the same predictions, the only difference is the rules. So the gap between their scores shows what being "stuck" really costs.

## 3. How It Works, Step by Step

**Step 1: A robot checks every 30 minutes.** A scheduled job runs on GitHub Actions, a free service that runs code at set times. It runs at 7 and 37 minutes past every hour.

**Step 2: It asks, "What needs doing now?"** The robot has no memory between runs. Each time, it compares the database with the clock and does whatever is due. If a run is missed, the job is simply still waiting next time, like a to-do list that rewrites itself.

**Step 3: It collects data politely.** The data comes from FPL's free public API (a way for programs to ask a service for data). It gets players, prices, fixtures and live points. It waits at least 1 second between requests and saves every answer, so it never gets blocked.

**Step 4: It saves prices for each week.** Prices change all season. So the project saves every player's price for each gameweek. A pick for gameweek 3 always uses gameweek 3's prices.

**Step 5: It predicts each player's points.** For every player, it estimates "expected points": how many points they'll probably score. It looks at the next 5 gameweeks.

**Step 6: It picks both teams.** About 8 hours before each deadline, an optimiser picks the best legal squad for each team. An optimiser is a program that checks every possible combination and finds the best one.

**Step 7: It locks the picks.** Both teams are saved before the deadline. After the deadline, the code refuses to change them. This keeps the record honest.

**Step 8: It scores the week.** After the matches, it saves the real points. It compares each team with the official average score for that gameweek.

**Step 9: It updates the website.** Every page is saved as a file ahead of time and put on GitHub Pages (free website hosting). No server needs to be running.

## 4. The Clever Parts

**Expected points built from simple parts.** It doesn't use a complicated trained model. It adds up simple parts: will the player start, will they score or assist, will their team keep a clean sheet, and will they get bonus points?

This means every prediction can be explained on the website, like "likely starter, easy home game, takes penalties". Early in a season there's too little data for anything fancier.

**Pulling guesses towards normal.** Early in the season, one lucky goal could make a cheap defender look like a superstar. So each player's numbers are pulled towards what's normal for their price and position. As they play more minutes, their real numbers count more.

**An optimiser instead of greedy picking.** Picking the top players one by one doesn't work. You might spend all your money on three stars and break the "max 3 per club" rule.

The optimiser checks budget, positions and club limits all at once. It's like packing a suitcase with a weight limit.

**Careful about extra transfers.** The Manager asks, "What's the best squad I can reach with 0, 1, 2, 3, 4 or 5 transfers?" Each extra transfer costs 4 points. It only takes the hit if it expects to gain at least 6 points: 4 for the hit, plus 2 to be safe.

**Rules read from FPL itself.** This season, a goalkeeper's goal became worth 10 points. The project reads the points for each action straight from FPL, so a rule change needs no code change.

## 5. The Important Words

- **Gameweek**: one round of matches, with a deadline when teams lock.
- **Transfer**: swapping one player for another.
- **Hit**: the 4-point cost of each extra transfer.
- **Chip**: a one-off power-up, like Triple Captain, where your captain scores triple points.
- **Expected points**: a guess of how many points a player will score.
- **Optimiser**: a program that finds the best legal combination.
- **API**: a way for programs to ask a service for data.
- **Stateless scheduler**: a job runner with no memory, which just asks "what's due now?"
- **Locked picks**: a team that can't be changed after the deadline.
- **Leakage**: accidentally using future information.

## 6. The Tools, in One Line Each

- **Python**: the language the back end is written in.
- **httpx**: fetches data from the FPL API.
- **SQLite**: a simple database that keeps the whole season in one file.
- **PuLP and CBC**: the optimiser tools that find the best squad.
- **FastAPI**: serves the results as data for the website.
- **React and Vite**: build the website's pages.
- **GitHub Actions**: the free robot that runs every 30 minutes.
- **GitHub Pages**: free hosting for the website.
- **pytest**: runs 610 automatic checks, with no internet needed.

The back end only needs four outside tools to run. That's on purpose, because the points model is simple arithmetic.

## 7. How Good Is It?

After 5 gameweeks, Best XI had 322 points and The Manager had 268. That's a gap of 54 points, or about 11 points per gameweek. That's the rough cost of being stuck so far.

Best XI beat the official average 2 times out of 5. The Manager beat it once.

Five weeks is far too few to judge. Also, how accurate the point predictions are hasn't been measured yet, and it hasn't been tested on a past season.

## 8. What's Weak or Missing

- There's no test on a past season, so the model's accuracy isn't proven.
- The model's settings were chosen by reasoning, not learned from data.
- FPL doesn't keep old injury news. So rebuilt past weeks accidentally know about later injuries.
- The model was fixed after gameweeks 1 to 4 had been played, so only gameweek 5 onwards is a truly fair test.
- The project's own documents are a bit behind the code, for example on the number of tests.

## 9. What This Project Shows You Can Do

- Turn a real game's rules into exact, tested code.
- Use an optimiser to solve a "best choice under many rules" problem.
- Design a fair experiment, where only one thing differs.
- Build a cheap system that runs itself, with no server.
- Keep a record honest by locking decisions at the deadline.

## 10. Ten Things to Remember

1. Two robot teams use the same predictions, and only the rules differ.
2. The Manager follows real rules, and Best XI rebuilds freely each week.
3. The score gap measures the cost of being stuck.
4. Data comes from FPL's free public API, collected politely.
5. Expected points are built from simple, explainable parts.
6. An optimiser picks the best legal squad.
7. An extra transfer must be worth at least 6 points.
8. The scheduler has no memory and just asks "what's due now?"
9. Picks are locked at the deadline.
10. After 5 weeks: Best XI 322, The Manager 268.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
