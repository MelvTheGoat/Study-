# Premier League Predictor: System Design for Beginners

This project predicts every Premier League match, one gameweek at a time. For each game it gives three chances (home win, draw, away win) and a likely score, and shows them on a public website. It has all the classic parts of a system that learns from data, so it's great to study.

## Key Terms

- **Model**: a program that learns patterns from past examples to guess something new.
- **Feature**: one fact about a match as a number, like "days since this team last played".
- **Gameweek**: one round of league matches, usually 10 games.
- **Elo rating**: a team strength score that rises with wins and falls with losses, like a chess rating.
- **Probability**: a chance, as a percentage. "Home win 50%" means it should happen about half the time.
- **Retrain**: teaching the model again with the newest results.
- **Pipeline**: steps that run in order, each feeding the next, like a factory line.
- **Leakage**: when a model accidentally sees the future while learning, so it looks better in tests than it really is.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** We want honest chances, not just one pick: "When it says 60%, does that happen about 60% of the time?"

**Step 2: Figure out the data.** We need past results, upcoming fixtures, and context like manager changes. Ask which data is free, and which arrives late.

**Step 3: Sketch the main parts.** Data comes in, becomes features, the model learns, and predictions are saved and shown. Draw these as boxes.

**Step 4: Walk through one gameweek.** Follow the next round from "new results arrive" to "predictions on the website", checking nothing from the future sneaks in.

**Step 5: Decide how to know it works.** Test on past seasons, one week at a time, and compare against a simple guess like "always pick the home team".

**Step 6: Plan for problems.** Data can arrive late, the website can stop updating, and old predictions could be overwritten. Plan a check for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Predict **10** matches per gameweek, across **38** gameweeks.
- Learn from past seasons, starting from **2010-11** by default.
- Check for new results **once a day**, at 6 a.m. UTC (the world's standard clock).
- **Never** change a prediction once it's published.

How much history? Step by step:

1. Matches per season: 10 × 38 = **380**.
2. Seasons of history from 2010-11 to 2025-26: **16**.
3. 16 × 380 = 6,080, so about **6,000** Premier League matches to learn from.

### The Big Picture (Step 3)

```
   [Football data sources]
            |
            v
   [Ingest: load and clean] ---> [Database]
            |
            v
   [Feature Builder: ~200 facts per match]
            |
            v
   [Models: result + score]
            |
            v
   [Prediction Store: never overwritten]
            |
            v
   [Website] <--- [Daily Robot checks it all]
```

Step by step:

1. Every morning, the Daily Robot wakes up. It runs on GitHub Actions (a free service that runs code at set times).
2. It pulls the latest results from openfootball, a free public collection of English football results.
3. Ingest loads them into the database (an organised store of data). If nothing is new, the robot stops.
4. The Feature Builder records about 200 facts per match, using only what was known before the gameweek's first kick-off.
5. The models retrain and predict the next gameweek.
6. The predictions are saved as a new record. Old ones are never touched.
7. A small copy of the database goes to the website, and the robot checks the live site.

### The Main Parts (Step 3)

**Data Sources and Ingest.** Results come from several leagues and cups, plus match stats (shots, cards) and squad ratings from the FIFA video games. Ingest also fixes club names that sources spell differently. It's like a post room: letters arrive in different formats, and it sorts them into one place with the same labels.

**Feature Builder.** This turns each match's situation into numbers: recent form, table gaps, days of rest, manager changes, squad quality and past meetings. It's like a scout filling in a report card before every match. It walks through history in date order, so it can't see the future, like reading a diary without flipping ahead.

**Elo Ratings.** Teams in the Premier League, Championship and League One share one scale, with lower divisions starting lower. So a promoted team's good season in the division below still counts. It's like one ranking ladder for a whole tennis club, not one per court.

**Result Model.** Two models guess, and their answers are blended. 60% comes from LightGBM (a tool that builds many small yes/no flowcharts, each fixing the last one's mistakes), and 40% from a simpler, steadier model that helps early in the season. It's like asking an expert and a sensible friend, then trusting the expert a bit more.

**Score Model.** A separate model guesses each team's goals from its attack, the other team's defence and home advantage. It's like a weather forecast, but for goals. The score shown always agrees with the pick, so you never see "home win, 1-1".

It also has a small correction for low scores like 0-0 and 1-1 (advanced - skip for now).

**Prediction Store.** Every retrain adds a new record and nothing is overwritten. Predictions saved after kick-off are marked "late". It's like writing your predictions in pen, in a dated notebook.

**Website.** The website is built with Flask (a simple Python tool for making websites). It never runs the heavy maths, and only reads a 1.4 MB copy of the 61 MB database. It's like a shop window: it shows the finished goods, while the workshop stays out back.

### How We Know It's Working (Step 5)

These come from the project's own report, testing three past seasons week by week (1,050 matches).

- **Accuracy**: right **52%** of the time, versus **43%** for always guessing "home win". Step by step: 52% of 1,050 ≈ 550 right, versus 43% of 1,050 ≈ 450.
- **Calibration**: when it said about 65%, that result happened about 65% of the time.
- **Log loss**: a score that punishes confident wrong guesses (lower is better). The model scored **0.99**, and a simpler Elo-only version **1.01**.
- **Exact score**: right about **8%** of the time, since exact scores are very hard.

### What Can Go Wrong (Step 6)

- **Late data.** Match stats arrive weeks late. Any fact missing for over half the week's matches is dropped for that run, and returns once the data catches up.
- **The website quietly goes stale.** Once, every job looked fine while the live site sat out of date. Now the robot checks the real website every run.
- **Old predictions wiped.** A rebuild could erase the record, so published history is restored before predicting.
- **Missing context.** Injuries aren't collected yet, and manager history only starts in 2025-26.

## Quick Recap

- It's a pipeline: collect data, build features, retrain, predict, save, publish.
- Features only use what was known before each gameweek: no peeking ahead.
- One shared Elo scale lets promoted teams bring their record with them.
- Predictions are written "in pen": saved forever and marked if late.
- A daily robot checks everything, right up to the live website.
