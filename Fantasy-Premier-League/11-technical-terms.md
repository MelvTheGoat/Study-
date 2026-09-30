# FPL AI Manager: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md), and come back whenever a word trips you up.

The terms are grouped in the order things happen: the game's rules, collecting data, predicting points, picking the team, and running it all.

---

## 1. The Game

### Gameweek and deadline
**What it means:** a gameweek is one round of Premier League matches. The deadline is the moment, just before the first match, when your team locks.

**Why it's needed here:** everything is organised around deadlines. Both teams must be picked before each one, and nothing can change afterwards.

### Squad, starting XI and bench
**What it means:** your squad is 15 players (2 goalkeepers, 5 defenders, 5 midfielders, 3 forwards). 11 of them start and score points. The other 4 sit on the bench.

**Why it's needed here:** these are the rules every pick must follow. The optimiser has to respect them, or the team is illegal.

### Transfer and hit
**What it means:** a transfer swaps one player for another. You get 1 free transfer per week, and each extra one costs 4 points, which is called a hit.

**Why it's needed here:** this is the heart of the experiment. The Manager has to live with these costs, and Best XI doesn't.

### Chip
**What it means:** a one-off power-up. For example, Bench Boost (your bench scores too), Triple Captain, Free Hit (a one-week squad) and Wildcard (unlimited free transfers). There are two sets per season.

**Why it's needed here:** chips can be worth lots of points, so The Manager checks each week whether playing one is worth it. It only plays a chip when the gain passes a set bar, for example 30 points for a Wildcard.

### Selling price
**What it means:** what you get back when you sell a player. If their price went up, FPL keeps half of the rise. If it went down, you take the full fall.

**Why it's needed here:** The Manager's budget depends on it. Getting it wrong would make the team think it has more money than it really does.

### Blank and double gameweek
**What it means:** a blank gameweek is when a team has no match. A double gameweek is when a team plays twice.

**Why it's needed here:** a player with no match scores zero, and one with two matches can score double. The predictor handles both, though so far they've only been tested with made-up fixtures.

---

## 2. Collecting the Data

### API
**What it means:** a way for one program to ask another service for data. You send a request to a web address and get data back.

**Why it's needed here:** FPL's public API is the project's only data source: players, prices, fixtures, live points and even the scoring rules.

### JSON
**What it means:** a simple text format for data, made of labels and values, like `"price": 5.5`.

**Why it's needed here:** the FPL API sends its answers as JSON. The website also reads its data as JSON files.

### Rate limiting
**What it means:** deliberately slowing down how often you send requests.

**Why it's needed here:** the API is public with no login, so sending too many requests could get the project blocked. It waits at least 1 second between requests.

### Cache
**What it means:** a saved copy of an answer, so you don't have to ask again.

**Why it's needed here:** every API answer is saved on disk. Re-running a step is then fast and doesn't bother the API.

### Ingest
**What it means:** turning raw data from a source into tidy rows in your own database.

**Why it's needed here:** the API's layout is messy. Keeping all that mess in one place means the rest of the code only sees clean data.

### Database (SQLite)
**What it means:** an organised store of data in tables. SQLite keeps the whole database in a single file.

**Why it's needed here:** a whole season fits in a few megabytes across 17 tables. One file is easy to back up and to store between runs.

### Price snapshot
**What it means:** a saved record of every player's price at a particular gameweek.

**Why it's needed here:** prices change all season. Saving them per gameweek means a pick for gameweek 3 uses gameweek 3's prices, not today's.

---

## 3. Predicting Points

### Expected points (xPts)
**What it means:** a guess of the average number of points a player will score in a match.

**Why it's needed here:** the optimiser needs a number for every player to compare them. Both teams use the exact same expected points, so the test is fair.

### Per-90 rate
**What it means:** how often a player does something for every 90 minutes played. For example, 0.5 goals per 90.

**Why it's needed here:** it makes players fair to compare, even if one played 900 minutes and another only 90.

### Shrinkage (towards a prior)
**What it means:** a prior is a sensible starting guess. Shrinkage pulls a player's numbers towards it until there's enough real evidence.

**Why it's needed here:** early in a season, one lucky goal could make a cheap defender look like a superstar. Here the starting guess is based on the player's price and position, and the pull fades as they play more minutes.

### Chance of starting (minutes model)
**What it means:** a guess of how likely a player is to start, come off the bench, or not play at all.

**Why it's needed here:** a great player who doesn't play scores nothing. It uses recent starts, last season's minutes, and injury news from the API.

### Team strength
**What it means:** how many goals a team is expected to score and concede in a particular match.

**Why it's needed here:** a striker facing a weak defence should get a higher guess. FPL stopped publishing its own team ratings this season, so the project rebuilds them from fixture difficulty and real results.

### Poisson distribution
**What it means:** a standard way to work out the chance of 0, 1, 2 or more of something rare happening, like goals.

**Why it's needed here:** it gives the chance of a clean sheet (0 goals conceded), and the chance of conceding enough goals to lose points.

### Component model (not trained)
**What it means:** building a prediction by adding up simple parts using reasoned rules, instead of letting a program learn the weights from data.

**Why it's needed here:** there's too little early-season data to train something better. Each part can also be explained on the website, like "on penalties, easy home game".

### Horizon
**What it means:** how far ahead you look. Here, 5 gameweeks.

**Why it's needed here:** a transfer should help for several weeks, not just one. The Manager counts future weeks a little less than this week.

---

## 4. Picking the Team

### Optimiser (integer linear programming)
**What it means:** a method for finding the best choice when there are many rules at once. "Integer" means each answer is a whole number: a player is in (1) or out (0).

**Why it's needed here:** budget, positions and the 3-per-club limit all affect each other. Picking the best players one by one breaks the rules or wastes money. The optimiser finds the true best legal squad.

### Constraint
**What it means:** a rule the answer must obey, like "exactly 15 players" or "no more than £100m".

**Why it's needed here:** every FPL rule becomes a constraint, so the optimiser can never return an illegal team.

### Greedy approach
**What it means:** always grabbing the best option right now, without thinking about the whole picture.

**Why it's needed here:** it's what the project avoids. Greedy picking might spend the whole budget on 3 stars and leave no money for the rest.

### Hit margin
**What it means:** a safety buffer on top of the hit cost.

**Why it's needed here:** predictions are uncertain. So a transfer that costs a hit must be expected to gain at least 6 points (4 for the hit plus 2 extra).

### Bench weight
**What it means:** how much the bench players' points count when choosing the squad. Here it's 0.12, so a bench point is worth about an eighth of a starter's.

**Why it's needed here:** if the bench counted for nothing, the optimiser would fill it with the cheapest players, who never play. A small weight keeps the bench useful if a starter is injured.

### Locked picks
**What it means:** the saved team for a gameweek, which can't be changed after the deadline.

**Why it's needed here:** it keeps the record honest. Trying to save new picks after the deadline causes an error.

### Leakage
**What it means:** accidentally using information from the future when making a decision in the past.

**Why it's needed here:** picks must only use what was known before the deadline. The project guards against this, but there's one known gap: injury news for rebuilt past gameweeks is today's news, because FPL doesn't keep old injury news.

---

## 5. Running and Publishing

### Scheduler (stateless)
**What it means:** a program that decides which jobs to run. "Stateless" means it keeps no memory between runs.

**Why it's needed here:** each time, it simply compares the database with the clock and does what's owed. If a run crashes or is missed, the work is still owed next time.

### Pick window
**What it means:** the time before a deadline when picking is allowed. Here, from 8 hours before until 2 minutes before.

**Why it's needed here:** GitHub's timer can skip hours. A wide window means a pick still happens even if a run is late.

### Cron
**What it means:** a way of writing a repeating timetable for a computer, like "at 7 and 37 minutes past every hour".

**Why it's needed here:** it sets when the scheduler runs, about every 30 minutes.

### Static export
**What it means:** saving every page's data as a file ahead of time, so no live server is needed.

**Why it's needed here:** the site is hosted free as plain files. The files are made by calling the project's real data service, so they always match it.

### Read-only service
**What it means:** a service that only shows data and never changes it.

**Why it's needed here:** the picking happens in separate jobs. The website's data service only serves what was decided, so a slow page can never delay a deadline.

### Force-push
**What it means:** replacing what's stored on a git branch (a separate line of saved work) instead of adding to it.

**Why it's needed here:** the database changes every 30 minutes. Replacing it each time keeps the project from growing huge. A 30-day backup copy is kept as well.

### Concurrency lock
**What it means:** a rule that only one run can happen at a time.

**Why it's needed here:** SQLite only allows one writer at a time. Two runs at once could damage the database, so the timetable blocks it.

### Tests
**What it means:** small programs that check the real code still works as expected.

**Why it's needed here:** a wrong rule quietly breaks every number after it. The project has 610 tests that run without the internet, many using real recorded API answers.
