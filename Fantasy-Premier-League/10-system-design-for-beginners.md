# FPL AI Manager: System Design for Beginners

This project plays Fantasy Premier League (FPL) with two robot teams that use the same predictions. One team follows the real rules and keeps its squad week to week. The other gets a brand-new perfect squad every week. The gap between their scores shows how much being "stuck" with last week's choices costs.

## Key Terms

- **FPL**: a free online game where you pick 15 real Premier League players and score points from how they play.
- **Gameweek**: one round of matches. Each has a **deadline**, after which your team is locked.
- **Transfer**: swapping a player out of your squad. You get one free per week, and each extra costs 4 points (a **hit**).
- **Chip**: a one-off power-up, like Triple Captain, where your captain scores triple points instead of double.
- **Expected points**: a guess of how many points a player will score, based on their past games and their next opponent.
- **Optimiser**: a program that searches every legal combination and finds the best one. Like solving a giant Sudoku.
- **API**: a way for programs to ask another service for data. FPL has a public one.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** We want a fair test: two teams with the same predictions, where only the rules differ. Any score gap then comes from the rules, not from better guesses.

**Step 2: Figure out the data.** We need players, prices, fixtures and live points. The FPL API gives all of it for free, with no login.

**Step 3: Sketch the main parts.** Data comes in, gets stored, turns into predicted points, and the optimiser picks the teams. A website shows the results.

**Step 4: Walk through one gameweek.** Follow one week from "new prices" to "picks locked" to "final points". Check that nothing can be changed after the deadline.

**Step 5: Decide how to know it works.** Compare each team's real points with the official average for that gameweek.

**Step 6: Plan for problems.** The API could block us, a scheduled run could be missed, and the game's rules could change. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Choose from about **650** players.
- Pick a legal squad: **15** players, a **£100m** budget, at most **3** from one club.
- Lock both teams before every one of the **38** deadlines.
- Check what needs doing about every **30 minutes**.

How often does it check? Step by step:

1. 2 checks per hour (at 7 and 37 minutes past).
2. 2 × 24 hours = **48 checks per day**.
3. Most do almost nothing. They only act when something is due.

### The Big Picture (Step 3)

```
   [FPL public API]
          |
          v
   [Data Collector: polite + cached] ---> [Database]
                                              |
                                              v
                                   [Points Predictor]
                                              |
                                              v
                                   [Optimiser: best legal squad]
                                              |
                                              v
                                   [The Manager + Best XI]
                                              |
                                              v
   [Scheduler] ....................>  [Website]
```

Step by step:

1. Every 30 minutes, the Scheduler runs on GitHub Actions (a free service that runs code at set times). It asks the database "what's due now?"
2. If data is old, the Data Collector fetches fresh prices and fixtures, at most 1 request per second.
3. The Points Predictor guesses each player's points for the next 5 gameweeks.
4. Inside 8 hours of a deadline, the Optimiser picks both teams from the same guesses.
5. Both teams are saved as "locked picks".
6. After the matches, real points are saved and each team is scored against the average.
7. The website is rebuilt from the results.

### The Main Parts (Step 3)

**Data Collector.** It asks the FPL API for data, waits at least 1 second between requests, and saves every answer on disk. It's like a polite customer who writes things down, instead of asking the shop assistant the same question again and again.

**Rules Engine.** All of FPL's rules live in one place: squad shape, formations, captains, selling prices and chips. It's like the referee's rulebook. If one rule is wrong, every number after it is wrong, so this part has the most tests.

**Points Predictor.** It builds each player's expected points from simple parts: will they play, will they score or assist, will their team keep a clean sheet? It's like a pundit adding up "likely starter, easy home game, on penalties". New-season numbers are thin, so each player's rates are pulled towards what's normal for their price.

**Optimiser.** It uses PuLP (a free Python tool for solving "pick the best combination" puzzles). Picking the top players one by one fails, because budget, positions and the 3-per-club rule all affect each other. The optimiser checks them all at once, like packing a suitcase with a weight limit.

The maths behind it is called integer linear programming (advanced - skip for now).

**The Two Teams.** **Best XI** asks the optimiser for the best squad from scratch each week. **The Manager** asks, "What's the best squad I can reach with 0 to 5 transfers?", then takes off 4 points for each extra one. A hit only happens if it's worth at least 6 points (4 for the hit, plus 2 to be safe).

**Scheduler.** It keeps no memory between runs. Each time, it compares the database with the clock and does whatever is owed. It's like a to-do list that rewrites itself: if a run is missed, the work is simply still on the list next time.

**Website.** Every page is saved as a file ahead of time and hosted on GitHub Pages (free website hosting). No server needs to be running. It's like printing a newspaper instead of answering every reader's phone call.

### How We Know It's Working (Step 5)

- **Beat the average**: after 5 gameweeks, Best XI beat the official average 2 times and The Manager 1 time.
- **Total points**: Best XI 322, The Manager 268. Step by step: 322 − 268 = 54 points, and 54 ÷ 5 ≈ **11 points per gameweek**. That's the rough cost of being stuck so far.
- **Tests**: 610 automatic checks pass, with no internet needed.

Five weeks is far too few to judge. Also, how accurate the point guesses are isn't measured in the project yet.

### What Can Go Wrong (Step 6)

- **The API blocks us.** Too many requests look like an attack. So the collector waits between requests and saves answers to reuse.
- **A scheduled run is missed.** GitHub's timer sometimes skips 2 to 7 hours. So picks start 8 hours before the deadline, and a run stays awake until the deadline is safe.
- **Changing picks after the deadline.** That would be cheating. Once the deadline passes, saving new picks for that week causes an error.
- **The rules change.** This season a goalkeeper's goal became worth 10 points. So points per action are read from the API, not typed into the code.

## Quick Recap

- Two teams, one set of predictions: the only difference is the rules they follow.
- Expected points are built from simple, explainable parts.
- An optimiser finds the best legal squad, because greedy picking breaks the rules.
- A memory-free scheduler just asks, "What's due now?"
- Picks are locked at the deadline, so the record is fair.
