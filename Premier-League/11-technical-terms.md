# Premier League Predictor: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md), and come back whenever a word trips you up.

The terms are grouped in the order the data travels: collecting it, turning it into facts, learning from it, checking the results, and publishing them.

---

## 1. Collecting the Data

### Data source
**What it means:** a place the project gets its information from, like a website or a public collection of files.

**Why it's needed here:** the model can only learn from what it's given. This project uses four: openfootball (results and fixtures), football-datasets (shots, corners and cards), FIFA / EA FC player ratings, and a few hand-written files (managers, European matches, team names).

### Fixture
**What it means:** a match that is scheduled but not played yet.

**Why it's needed here:** fixtures are the things being predicted. The project reads them from the same place as the results, so it always knows which gameweek is next.

### Gameweek
**What it means:** one round of league matches, usually 10 games spread over a few days.

**Why it's needed here:** the project predicts one gameweek at a time. It also uses the gameweek as its "cut-off line" for what the model is allowed to know (see *Point-in-time cut-off* below).

### Ingest
**What it means:** reading data from its sources and loading it into one place, in one tidy format.

**Why it's needed here:** each source has its own layout. Ingest turns them all into the same tables, so every later step reads from one place.

### Name normalisation
**What it means:** making sure one club always has one name. For example, "Man City" and "Manchester City" become the same team.

**Why it's needed here:** different sources spell clubs differently. Without this, one club would look like two, and its history would be split in half.

### Database
**What it means:** an organised store of data, arranged in tables, like a set of linked spreadsheets.

**Why it's needed here:** the project keeps matches, stats, features and predictions in one database file. There are two copies: a big working one (about 61 MB) and a small one for the website (about 1.4 MB).

---

## 2. Turning Matches into Facts

### Feature
**What it means:** one fact about a match, written as a number the model can use. "Days since the home team last played" is a feature.

**Why it's needed here:** a model can't read "they're tired after a European trip". It can read "3 days of rest". The project creates about 200 features per match.

### Context as features (not rules)
**What it means:** instead of a person writing a rule like "a new manager adds 5%", you give the model the raw fact and let it work out how much it matters.

**Why it's needed here:** this is the project's main idea. Hand-made rules are guesses. Letting the model weigh each fact from 15+ seasons of history is more honest, and it can ignore facts that don't help.

### Rolling form
**What it means:** how a team has done in its last few games, like points per game over the last 4, 6 or 10 matches.

**Why it's needed here:** recent results tell you more about a team today than results from years ago. Games older than 120 days drop out, so last season's run-in fades away as a new season starts.

### Fixture congestion
**What it means:** how many games a team has played recently, and how many days of rest it has had.

**Why it's needed here:** tired teams play worse. The project treats 7 days as normal rest, and reads cup and lower-league matches so it can see midweek games. It also records which European competition a team is in that season.

### Head-to-head (H2H)
**What it means:** how two teams have done against each other in the past.

**Why it's needed here:** some teams keep doing well against a particular opponent. It's one of the features the simpler model always uses.

### Home, away and difference columns
**What it means:** each fact is stored three times: for the home team, for the away team, and the gap between them.

**Why it's needed here:** what often matters most is the *gap*, like "the home team has 2 more days of rest". Giving the gap directly saves the model from having to work it out.

### Elo rating
**What it means:** one strength score per team. It goes up when a team wins and down when it loses, and beating a strong team moves it more. Chess uses the same idea.

**Why it's needed here:** it's the one measure that survives promotion. A newly promoted team has no Premier League form, but it does have an Elo score from the division below.

### Division offset
**What it means:** a head start (or handicap) in Elo points based on the division a team starts in.

**Why it's needed here:** without it, a League One team would start equal to a Premier League team. Here, a new Championship team starts 180 points lower, and a new League One team starts 320 points lower.

### Regression to the mean
**What it means:** pulling every score part of the way back towards average.

**Why it's needed here:** squads change every summer, so last season's rating shouldn't count in full. Between seasons, each team's Elo is pulled 25% of the way back to the average (1,500).

### Squad ratings
**What it means:** a club's player quality "on paper", taken from the FIFA and EA FC video game ratings.

**Why it's needed here:** results alone can't tell you a team has just signed great players. The ratings are a season old by design, so the project also records how old they are as a feature.

### Point-in-time cut-off
**What it means:** only using information that existed at a certain moment.

**Why it's needed here:** every match in a gameweek uses what was known before that gameweek's **first** kick-off. That's exactly what the project knows when it publishes, so learning and predicting see the same information.

### Leakage
**What it means:** when a model accidentally sees the future while learning. It then looks great in tests but fails in real life.

**Why it's needed here:** avoiding it is built into the design. Features are built in one pass, in date order, and each team's record only moves forward, so the future simply isn't there to see.

### Feature version
**What it means:** a label on the feature table that changes whenever the meaning of a column changes.

**Why it's needed here:** it stops old and new features from being mixed by accident.

---

## 3. Learning from the Data

### Model
**What it means:** a program that learns patterns from past examples, then uses them to guess something new.

**Why it's needed here:** there are two. One predicts the result (home win, draw or away win), and one predicts the score.

### Training and retraining
**What it means:** training is when the model learns from past matches. Retraining is doing it again with newer results added.

**Why it's needed here:** the project retrains after every gameweek, so the latest results always count.

### Probability
**What it means:** a chance, written as a percentage.

**Why it's needed here:** the project publishes three chances per match, not just a pick. That's more honest, because football is uncertain.

### Gradient boosting (LightGBM)
**What it means:** a method that builds many small yes/no flowcharts ("decision trees"). Each new one tries to fix the mistakes of the ones before.

**Why it's needed here:** it can spot facts that matter together, like "being tired matters more when key players are missing". It provides 60% of the result prediction.

### Logistic regression
**What it means:** a simpler model that gives each fact a weight, adds them up, and turns the total into chances.

**Why it's needed here:** it's steadier than LightGBM, especially early in a season when there's little new data. It uses a small set of 12 always-available features and provides the other 40%.

### Blend (ensemble)
**What it means:** combining the answers of more than one model.

**Why it's needed here:** the two models make different kinds of mistakes. Blending them (60% + 40%) gives better-balanced chances across a whole season than either model alone.

### Early stopping
**What it means:** stopping training at the point where the model stops getting better on recent matches it hasn't learned from.

**Why it's needed here:** training too long makes a model memorise the past instead of learning from it. The code notes that a fixed 400 rounds did worse than stopping at around 60.

### Poisson model
**What it means:** a common way to model counts of rare events, like goals in a match.

**Why it's needed here:** a score is two counts (home goals and away goals), not a label. So the score model uses a Poisson-based method called **Dixon-Coles**, which gives every club an attack strength and a defence strength.

### Time decay
**What it means:** making older matches count less than recent ones.

**Why it's needed here:** a team from two years ago may be very different today. In the score model, a match about two years old counts about half as much as today's.

### Ridge (shrinkage)
**What it means:** gently pulling extreme numbers back towards average.

**Why it's needed here:** a team with only a few matches could end up with a crazy rating after one lucky day. Ridge stops that. The score model also learns from Championship matches, so promoted teams have a full season behind them.

---

## 4. Checking the Results

### Baseline
**What it means:** a very simple guess used as a comparison.

**Why it's needed here:** a model is only useful if it beats something simple. Here the baselines are "always pick the home team" (43% right) and an Elo-only model.

### Walk-forward backtest
**What it means:** testing on the past as if it were live. You predict week 1 using only earlier data, then week 2, and so on.

**Why it's needed here:** it gives an honest estimate of how the model would really have done. The project's report says 52% of results right over 1,050 past matches.

### Accuracy
**What it means:** the share of predictions that were right.

**Why it's needed here:** it's the easiest number to understand, but it only looks at the pick, not the chances.

### Log loss
**What it means:** a score for chances that punishes being confidently wrong. Lower is better.

**Why it's needed here:** the project cares about honest chances, so this is what the models are trained to improve.

### Calibration
**What it means:** whether the chances match reality. If a model says 70% many times, those things should happen about 70% of the time.

**Why it's needed here:** published chances are only useful if you can trust them. The project's check found predicted and real rates within about 4 percentage points.

### MAE (mean absolute error)
**What it means:** the average size of a miss, ignoring whether it was too high or too low.

**Why it's needed here:** it measures the score model. On average, its goal guesses were off by about 0.93 goals per team.

---

## 5. Running and Publishing

### Pipeline
**What it means:** a chain of steps that run in order, each feeding the next.

**Why it's needed here:** every run does the same thing: ingest, build features, retrain, predict, save. Putting it in one chain means nothing gets skipped.

### Coverage check
**What it means:** only using a feature if it's filled in for enough of the matches being predicted.

**Why it's needed here:** match stats arrive weeks late. Any feature missing for more than half the gameweek's matches is dropped for that run, and it comes back by itself once the data catches up.

### Append-only record
**What it means:** you can add new entries, but never change or delete old ones.

**Why it's needed here:** it makes the prediction history trustworthy. Each retrain adds a new record, and before predicting, a rebuilt database restores the published history so nothing is lost.

### Late flag
**What it means:** a mark on any prediction that was saved after its match had kicked off.

**Why it's needed here:** honesty. It shows which predictions were truly made in advance.

### Read-only export
**What it means:** a small copy of the database that the website can read but never change.

**Why it's needed here:** the website's host doesn't allow saving files. So the project builds a slim copy with only what the pages need.

### Serverless hosting
**What it means:** the host runs your website code only when someone visits, so you don't manage a computer yourself.

**Why it's needed here:** it's free and simple, but it has a size limit. That's why the website installs only Flask, and none of the heavy learning tools.

### Scheduled job
**What it means:** a task that runs by itself at a set time.

**Why it's needed here:** matches don't follow a neat weekly timetable. So the job runs every day at 6 a.m. UTC, and stops straight away if there are no new results.

### Smoke check
**What it means:** a quick test that the main pages load at all before publishing. The name comes from switching on a machine and checking it doesn't start smoking.

**Why it's needed here:** a broken database file would take the whole website down. This check catches it before it's published.

### Live-site check
**What it means:** visiting the real, public website to confirm it shows the latest gameweek.

**Why it's needed here:** once, every step reported success while the live site sat out of date, because the site was being published from a different branch. Checking the real website is the only way to be sure.

### Fast-forward
**What it means:** moving a branch (a separate line of work in git) forward to include new changes, without overwriting anything it already has.

**Why it's needed here:** it keeps the website's branch up to date safely. If a branch has its own changes, it's left alone instead of being forced.
