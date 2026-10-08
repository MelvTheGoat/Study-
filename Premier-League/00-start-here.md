# Premier League Predictor: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

A football league table tells you what already happened. It doesn't tell you the situation a match is played in.

Maybe a team is tired after a European trip on Thursday. Maybe they just changed manager. Maybe they're safe in mid-table in May and have nothing left to play for.

Good football pundits think about all of this in their heads. This project asks a simple question: can a computer learn how much each of these things really matters, and use that to predict every Premier League match?

## 2. The Big Idea

The big idea is "facts, not rules". Most people would write a rule like "a new manager adds 5% to the chance of winning". This project doesn't do that. Instead, it gives the computer the raw facts, like how long the manager has been at the club, and lets it work out from history how much each fact matters.

The computer learns from about 15 seasons of past matches, starting in 2010-11. If a fact turns out not to matter, the computer simply learns to ignore it.

## 3. What It Does

Every gameweek (one round of about 10 matches), the project predicts every Premier League match. For each match, it gives three chances: home win, draw and away win. It also gives a likely score, like 2-1.

It publishes these predictions on a public website. It saves every prediction forever and never changes one after it's published. That way, anyone can check how good it really is.

## 4. How It Works, Step by Step

Here is the whole journey, from raw data to the website.

**Step 1: A robot wakes up every morning.** At 6 a.m. UTC (the world's standard clock), a scheduled job runs on GitHub Actions. GitHub Actions is a free service that runs code for you at set times. Think of it as an alarm clock that starts the whole process.

**Step 2: It collects new results.** The main data comes from openfootball, a free public collection of English football results and fixtures. It reads the Premier League, but also the Championship, League One and the cups. The extra competitions help it see how busy each team's schedule is.

**Step 3: It checks if anything is new.** If no new matches have been played since yesterday, it stops right there. This saves time and avoids pointless work.

**Step 4: It cleans up team names.** Different sources spell clubs differently, like "Man City" and "Manchester City". The project makes sure each club always has one name, so its history isn't split in two.

**Step 5: It turns each match into numbers.** A computer can't read "they're tired". It can read "3 days of rest". So for every match, the project creates about 200 facts, called features. These include recent form, league table gaps, days of rest, manager changes, squad quality, past meetings between the two teams, and how each team does at home or away.

**Step 6: It never peeks at the future.** This is very important. Every fact for a match only uses information from before that gameweek's first kick-off. The project walks through history in date order, like reading a diary one page at a time without flipping ahead. This stops "cheating", which experts call leakage.

**Step 7: It retrains the models.** A model is a program that learns patterns from past examples. After every gameweek, the models learn again, so the newest results count.

**Step 8: It predicts the next gameweek.** The models give the three chances and a likely score for each match.

**Step 9: It saves the predictions as a new record.** Old predictions are never overwritten. If a prediction is saved after a match has already started, it's marked "late", so you know it wasn't truly made in advance.

**Step 10: It updates the website and checks it.** A small copy of the data goes to the website. Then the robot visits the real, live website to check it's showing the new gameweek. This last check exists because once, everything looked fine while the website was quietly out of date.

## 5. The Clever Parts

**One rating ladder for all divisions.** The project gives every team an Elo rating. Elo is a single strength score that goes up when you win and down when you lose, like in chess.

The clever bit is that the Premier League, Championship and League One teams are all on the *same* ladder. Lower divisions just start lower. So when a team is promoted, its good season in the division below still counts. Most models treat promoted teams as a mystery.

**Two models working together.** The result prediction blends two models.

60% comes from LightGBM, a tool that builds many small yes/no flowcharts, each fixing the mistakes of the last. 40% comes from logistic regression, a simpler and steadier model. The simple one helps most early in the season, when there's little new data. It's like asking an expert and a sensible friend, then trusting the expert a bit more.

**A separate score model.** Guessing exact scores is a different job. A separate model, called Dixon-Coles, guesses how many goals each team will score.

It looks at each team's attack, the other team's defence, and home advantage. Recent matches count more than old ones. The score it shows always agrees with the predicted result, so you never see "home win, 1-1".

**Dropping late data.** Some data, like shots and corners, arrives weeks late. If a fact is missing for more than half of the week's matches, it's left out for that week. It comes back by itself once the data catches up.

## 6. The Important Words

- **Gameweek**: one round of league matches.
- **Feature**: one fact about a match, written as a number.
- **Model**: a program that learns patterns from the past to make a guess.
- **Retrain**: teaching the model again with the newest results.
- **Probability**: a chance, as a percentage.
- **Elo rating**: a team strength score that rises with wins and falls with losses.
- **Leakage**: accidentally letting the model see the future. It makes results look better than they really are.
- **Backtest**: testing the model on past seasons, one week at a time, as if it were live.
- **Calibration**: whether the chances are honest. If it says 65% many times, those things should happen about 65% of the time.
- **Pipeline**: a chain of steps that run in order, like a factory line.

## 7. The Tools, in One Line Each

- **Python**: the programming language everything is written in.
- **pandas and NumPy**: tools for working with tables and numbers.
- **LightGBM**: the main learning tool for predicting results.
- **scikit-learn**: provides the simpler, steadier model.
- **SciPy**: does the maths for the score model.
- **SQLite**: a simple database that lives in one file.
- **Flask**: a small tool for building the website.
- **Vercel**: free hosting that puts the website online.
- **GitHub Actions**: the free robot that runs everything every morning.
- **pytest**: runs 98 automatic checks on the code.

## 8. How Good Is It?

The project tested itself on three past seasons, one week at a time, as if it were live. That's 1,050 matches.

It picked the right result about 52% of the time. Always guessing "home win" gets about 43%. Football is very unpredictable, so that's a solid result.

Its chances are also honest. When it said about 65%, that result happened about 65% of the time. It guessed the exact score right about 8% of the time, which is normal, because exact scores are very hard.

This season's live record is very small: 21 right out of 50 matches (42%) over the first five gameweeks. But only one of those gameweeks was published fully before kick-off. Five weeks is far too little to judge.

## 9. What's Weak or Missing

- Injury news is now recorded every day from the FPL website (since 30 September 2026), but the model doesn't use it yet. A few weeks of records is too little to learn from.
- Manager history now comes from Wikidata, a free public database. It covers 84% of matches since 2010-11, but less of the older seasons.
- There's no "expected goals" data. It was tested in October 2026 and didn't measurably help, so shots on target are still used instead.
- The model almost never picks a draw. A draw is rarely the single most likely result, though the draw chance (usually 25–30%) is still shown.
- The project's own documents disagree slightly on the number of features.

## 10. What This Project Shows You Can Do

- Turn gut-feeling football knowledge into measurable data.
- Be strict about not cheating with future information.
- Build a system that runs itself every day and checks it really works.
- Combine two models to get honest, well-balanced chances.
- Report results honestly, including the limits.

## 11. Ten Things to Remember

1. It predicts every Premier League match: home, draw and away chances, plus a score.
2. The big idea is "facts, not rules": the computer learns how much each fact matters.
3. It uses about 200 facts per match.
4. It never uses information from after a gameweek's first kick-off.
5. One Elo ladder covers three divisions, which helps with promoted teams.
6. The result model is 60% LightGBM and 40% a simpler model.
7. A separate model predicts the score.
8. Every prediction is saved forever and never changed.
9. A robot runs everything every morning and checks the live website.
10. In testing it was right 52% of the time, against 43% for always picking home.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
