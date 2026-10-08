# Premier League Predictor: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Every now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. So here's our project. We want a system that predicts every Premier League match, every gameweek.

For each match, we want three chances: home win, draw and away win. And we want a likely score too, like 2-1.

But here's the thing that makes it interesting. We don't want to just look at the league table. A good pundit knows that a team that played in Europe on Thursday is tired on Sunday.

A team that just sacked its manager might play differently. A team safe in mid-table in May might not care much.

So our big idea is this: we don't write rules like "a new manager is worth 5%". Instead, we turn every one of those situations into a fact, a number, and we let the computer learn from history how much each fact actually matters. Facts, not rules. Keep that in your head, because everything we build serves that idea.

And one more thing. We want to be honest. Every prediction we publish, we keep forever.

We never change it. So anyone can check how good we really are.

## So what do we need?

Before we touch anything, let's list what a project like this needs. Think of it like cooking. Before you cook, you check what's in the cupboard.

We need:

1. **Data.** Past results, upcoming fixtures, and context like managers and squad quality.
2. **A place to keep it.** A database, so every later step reads from one place.
3. **A way to turn matches into facts.** Numbers the computer can learn from.
4. **A golden rule: no peeking at the future.** I'll explain this one properly, it's the most important thing in the whole project.
5. **Models.** The programs that learn from the facts and make predictions.
6. **A way to test honestly.** So we know if it's actually any good.
7. **A record of predictions.** Saved forever, never changed.
8. **A website.** So people can see the predictions.
9. **Something that runs it all by itself**, every day, without us pressing buttons.
10. **Checks.** Little alarms that tell us if something has quietly gone wrong.

That's the whole shopping list. Now let's build it, in the right order. And the order matters, because each step needs the one before it.

## Step zero: set up the workshop

Before we write any real logic, we set up the project itself. Think of it as clearing the table and laying out your tools before you start cooking.

First, a `README.md` at the top. That's the front page of the project. It says what the project is, how to run it, and what the results are.

Next, we write down which tools we need to install. And here's an early decision that pays off much later: we use **two** lists, not one.

`requirements.txt` holds only what the website needs, which is just Flask. `requirements-pipeline.txt` holds the heavy tools for data and models, like pandas, LightGBM and SciPy. Keep that in mind, because when we get to the website, you'll see why splitting them matters.

Then a `.gitignore` file. That tells git which files *not* to save, like the big working database and the raw downloads, because those get rebuilt anyway. There's one exception, and it's deliberate: the small website database *is* saved. Again, we'll see why later.

Now we create the main code folder, called `plpredict`. Inside it, we make small `__init__.py` files in each folder. They're nearly empty, but they tell Python "this folder is a package", so the code in one folder can use code in another.

And the last setup file is an important one: `plpredict/config.py`. This is the settings file for the whole project.

It holds where the data lives, which season we're predicting (2026-27), which season history starts from (2010-11), and every model setting. The idea is simple: no other file hard-codes a setting. If you want to change something, there's one place to change it.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `requirements.txt`: tools the website needs (just Flask).
> - `requirements-pipeline.txt`: the heavy tools for data and models.
> - `.gitignore`: files git should not save, like the big rebuilt database.
> - `plpredict/__init__.py`, plus an `__init__.py` in each sub-folder (`data`, `data/sources`, `features`, `models`, `pipeline`, `web`): mark each folder as a Python package.
> - `plpredict/config.py`: every setting in one place.

## Step one: we start with the data

Now the real work starts, and we start with data, because nothing else can happen without it. You can't build facts without results, and you can't train a model without facts.

Our main source is called openfootball. It's a free, public collection of English football results and fixtures. The nice thing is it's stored as a git repository, so the first time we download the whole thing, and after that we just pull the new bits each day. Quick and cheap.

To read it, we write `plpredict/data/sources/openfootball.py`. openfootball stores results as plain text files, one per competition per season, and the format has changed a bit over the years. This file reads those text files and turns them into clean match records. Crucially, it also reads the gameweek number for each block of fixtures, which we'll need for the golden rule.

Now, here's a smart choice. We don't only read the Premier League. We also read the Championship, League One, the FA Cup and the EFL Cup.

Why? Two reasons. First, if a team played a cup match on Wednesday, we want to know they're tired on Saturday.

Second, and this is a big one, when a team gets promoted, we want to know how good they were in the division below. Hold onto that, we'll come back to it with Elo.

Then we add some extras. Each one gets its own reader file.

**Match stats** like shots, corners, fouls, cards and the referee come from another free source, read by `plpredict/data/sources/footballdata.py`. Shots on target is the closest free stand-in for "expected goals".

The catch: this source runs weeks behind the live season. We'll deal with that later.

**Squad ratings** come from the FIFA and EA FC video games. Why a video game? Because EA rates every squad once a season, on the same scale, going back about twenty years. It's the only free way to measure a squad's quality "on paper".

The player files are huge, so we don't keep them in the project. Instead, `scripts/build_squad_ratings.py` boils them down once into one small file, `data/external/squad_ratings.csv`, with one row per club per season. Then `plpredict/data/sources/fifa_ratings.py` reads that file.

And then there are things you simply can't get from results, so we write them by hand into small CSV files in `data/manual/`:

- `managers.csv`: when each manager started and left. These only went back to 2025-26, so later we'll replace them with a better source.
- `european_participation.csv`: which clubs are in Europe each season, because a Thursday night in the Europa League changes Sunday.
- `unavailability.csv`: key players missing each gameweek. Honestly, it shipped empty. Injuries are the hardest thing to automate, so we'll come back to them.

We also write a test straight away: `tests/test_openfootball_parser.py`. It checks the text reader handles the different file formats correctly. If the reader is wrong, everything after it is wrong, so it's worth testing first.

> **📁 Files we just created**
> - `plpredict/data/sources/openfootball.py`: reads results and fixtures from the free text archive.
> - `plpredict/data/sources/footballdata.py`: reads match stats like shots and cards.
> - `scripts/build_squad_ratings.py`: boils the huge video game rating files down to one small file.
> - `data/external/squad_ratings.csv`: squad quality, one row per club per season.
> - `plpredict/data/sources/fifa_ratings.py`: reads the squad ratings file.
> - `data/manual/managers.csv`: manager start and end dates.
> - `data/manual/european_participation.csv`: which clubs play in Europe each season.
> - `data/manual/unavailability.csv`: missing players (empty for now).
> - `tests/test_openfootball_parser.py`: checks the results reader works.

## Step two: clean up the team names

Before we store anything, we have a small but important job. Different sources spell clubs differently. One says "AFC Bournemouth", another says "Bournemouth". If we don't fix that, the computer thinks they're two different clubs, and the history gets split in half.

So we write `plpredict/data/teams.py`. It's a little translator that makes sure every club always has one name. It uses simple rules, like dropping "AFC" or "FC" and tidying punctuation, plus a list of special cases the rules can't handle. Here's a real line from it:

```python
"Manchester City": "Man City",
```

And for spellings that are just odd, we keep a hand-written list in `data/manual/team_aliases.csv`, with lines like "Spurs" meaning "Tottenham Hotspur". Plus a test, `tests/test_teams.py`, to make sure the translator gets them right.

Boring? Yes. But skip it and everything after it is quietly wrong.

> **📁 Files we just created**
> - `plpredict/data/teams.py`: gives every club one single name.
> - `data/manual/team_aliases.csv`: a hand-written list of odd spellings.
> - `tests/test_teams.py`: checks the name cleaning works.

## Step three: put it all in one place

Now we need somewhere to keep everything. We use SQLite, which is a database that lives in a single file. No server to run, nothing to set up.

First we design the tables, in `plpredict/db.py`. Think of it as drawing the shelves before you fill the cupboard. There's a table for matches, one for match stats, one each for managers, missing players, squad ratings and European clubs.

There's also a table for the facts we'll build later, and tables for model runs and predictions. Everything has its own shelf.

Then we write `plpredict/data/ingest.py`, which does "fetch, clean, store". It calls the readers from step one, runs every name through the translator from step two, and saves everything into the database.

Here's a nice design choice: this file knows about data sources and the database, and nothing else. It does no feature work. So you could swap openfootball for a paid data feed one day without touching the model at all.

And we add a little command, `scripts/ingest.py`, so you can run just this part on its own. That's handy for checking a data source, or filling the database for the first time.

This working database ends up around 61 MB, and it gets rebuilt on each run. That's fine, it's quick, and it means we never have old junk lying around.

So let's pause and look at where we are. We've got our data. The names are clean.

It's all sitting in one database. Good. That part is set.

Now we can do the clever stuff.

> **📁 Files we just created**
> - `plpredict/db.py`: designs all the database tables.
> - `plpredict/data/ingest.py`: fetches, cleans and stores all the data.
> - `scripts/ingest.py`: a command to run just the data loading.

## Step four: the golden rule, no peeking

Okay, before we build a single fact, we need to agree on a rule. This is the most important idea in the whole project, so let's take our time.

Imagine you're testing a model on last season. You ask it to predict a match in March. If, by accident, one of its facts uses results from April, the model looks amazing.

But that's cheating. In real life, in March, you don't know what happens in April. Experts call this **leakage**, and it's the most common way people fool themselves.

So how do we make sure it never happens? We don't just promise to be careful. We build the system so it *can't* happen.

Here's how. We write `plpredict/features/state.py`. In it, each team has a little "state": a running record of its recent results, its points, its goals, its table position. And that state only ever moves forward, in date order.

It's like reading a diary one page at a time. You literally can't see tomorrow's page because you haven't turned to it yet. The file's own description puts it perfectly: everything in it answers one question, "what was knowable about this club the day before this match kicked off?"

And there's one more subtle bit. A gameweek isn't one day. It might start on Friday and finish on Monday.

So should Monday's match know Saturday's results? No. Because when we publish predictions, we publish the whole gameweek before the first kick-off.

So every match in a gameweek uses the state from *before the first kick-off*. Here's the real code that does it. Look at the order:

```python
for match in group:
    rows.append(self._featurise(match, table, len(members)))
for match in group:
    _apply_result(match, self.states, self.elo, self.head_to_head)
```

See it? First we write down the facts for *every* match in the gameweek. Only after that do we feed in the results.

So no match in the gameweek can see another match's result. That's the golden rule, written in code. That loop lives in the facts builder, which we'll meet in step six.

> **📁 Files we just created**
> - `plpredict/features/state.py`: each team's running record, which only ever moves forward in time.

## Step five: one ladder for every team

Now, remember I said hold onto the promoted teams? Here's where it pays off.

We write `plpredict/features/elo.py`. It gives every team an **Elo rating**. Elo is a single strength number. You win, it goes up.

You lose, it goes down. Beat a strong team and it goes up more. Chess uses the same idea.

The clever part is that we put the Premier League, the Championship and League One all on *one* ladder. Lower divisions just start lower. Here are the real numbers, from `config.py`:

```python
"premier_league": 0.0,
"championship": -180.0,
"league_one": -320.0,
```

So a brand-new Championship team starts 180 points below an average Premier League team. Why does this matter? Because when a team gets promoted, most models know nothing about them.

No Premier League form, no table position, no history. But on our ladder, they bring their whole Championship season with them.

A few other small touches. Every summer, ratings get pulled 25% of the way back to average, because squads change. Home teams get a 60-point boost.

Big wins count more than narrow ones. And cup matches count a bit less, because teams often rest players in cups.

> **📁 Files we just created**
> - `plpredict/features/elo.py`: one strength rating for every team, across three divisions.

## Step six: turn every match into facts

Now we're ready to build the facts, in `plpredict/features/build.py`. This is where the state from step four and the Elo from step five come together. It walks through all of history in one pass, in date order, and for every match it creates about 200 facts. Let me give you the families, so you see the shape:

- **Strength**: the Elo ratings.
- **Form**: how they've done in the last 4, 6 and 10 matches. Games older than 120 days drop out, so last season fades away as a new one starts.
- **Table context**: where they are in the table, and the gaps above and below.
- **Congestion**: how many days of rest they've had. 7 days counts as normal.
- **Manager**: how long the manager has been there, and how the team did before and after a change.
- **Availability**: key players missing. Remember, this file is empty for now.
- **Squad quality**: the video game ratings, plus how old those ratings are.
- **Head-to-head**: how these two teams have done against each other.
- **Venue**: how each team does at home or away.
- **Season shape**: how far through the season we are.

And here's a nice trick. Every fact is stored three times: for the home team, for the away team, and the *difference* between them. Because often what matters is the gap, like "the home team has 2 more days of rest". Giving the model the gap directly saves it from having to work it out.

One more small but wise thing. The facts table carries a version label. Whenever the meaning of a fact changes, the label changes too. So old and new facts can never get mixed up by accident.

And of course, a test: `tests/test_features.py`. It checks the facts are built correctly and, most importantly, that none of them can see the future. To keep the tests fast, `tests/conftest.py` builds a small pretend league for them to use, so they don't need the real data downloaded.

Let's circle back again. We have clean data in one place. We have a golden rule that's built into the code.

We have one ladder for all divisions. And we have about 200 honest facts for every match. That whole "facts" side is done.

Now we can finally teach the computer.

> **📁 Files we just created**
> - `plpredict/features/build.py`: turns every match into about 200 facts, in one pass through history.
> - `tests/conftest.py`: builds a small pretend league for the tests.
> - `tests/test_features.py`: checks the facts are right and never see the future.

## Step seven: the result model

Now we build the model that predicts home, draw or away, in `plpredict/models/outcome.py`. And here we make a decision that I really like. We don't use one model. We use two, and we blend them.

The first is **LightGBM**. Think of it as building lots of small yes-or-no flowcharts, where each new one fixes the mistakes of the last. It's great at spotting combinations, like "being tired matters more when key players are missing". But early in a season, when there's hardly any new data, it can get over-confident.

The second is **logistic regression**. It's much simpler. It gives each fact a weight and adds them up.

It only uses 12 core facts that are always available. It can't spot clever combinations, but it's steady and sensible, especially early in the season.

So we blend them: 60% LightGBM, 40% the simple one. Here's the real line:

```python
blended = weight * boosted + (1 - weight) * linear
```

Where `weight` is 0.6. It's like asking an expert and a sensible friend, then trusting the expert a bit more.

Two more details worth knowing. First, how long should LightGBM train? Too long and it memorises the past.

So we hold back the most recent 15% of matches, train until it stops getting better on those, then train the final model on everything. The code notes that a fixed number of rounds did noticeably worse.

Second, what do we train it to get right? Not accuracy, but something called **log loss**, which rewards honest chances and punishes being confidently wrong. Why?

Because we publish chances, not just picks. And here's a funny side effect: the model almost never picks a draw. A draw is rarely the single most likely result.

But that's fine, because we still show the draw chance, usually 25 to 30%.

> **📁 Files we just created**
> - `plpredict/models/outcome.py`: the result model, a 60/40 blend of LightGBM and a simpler model.

## Step eight: the score model

Guessing exact scores is a different job, so it gets its own model and its own file: `plpredict/models/scoreline.py`. The method is called **Dixon-Coles**, and the idea is simple once you hear it.

Every team gets an attack strength and a defence strength. The home team's expected goals is its attack, times the other team's defence, times a home advantage. Same the other way round. From those two numbers, you can work out the chance of every score, 0-0, 1-0, 2-1, up to 8 goals each.

A few smart touches:

- Plain maths gets low scores like 0-0 and 1-1 slightly wrong, so there's a small correction for them.
- Recent matches count more. A match about two years old counts about half as much as today's.
- We fit Championship matches too. Otherwise a promoted team that scored once in three games looks like it can't score at all.
- We gently pull extreme ratings back towards average, so one lucky afternoon doesn't create a superstar.

And one thing users appreciate: the score we show always agrees with the predicted result. So you never see "home win, 1-1" on the website.

Both models get tested together in `tests/test_models.py`.

> **📁 Files we just created**
> - `plpredict/models/scoreline.py`: the score model (Dixon-Coles).
> - `tests/test_models.py`: checks both models behave correctly.

## Step nine: what about data that arrives late?

Remember the match stats that run weeks behind? Here's where we deal with them. This bit lives inside `outcome.py`, in a small function called `usable_columns`.

Say we're predicting gameweek 5, and shots data only exists up to gameweek 2. If we let the model use the shots feature, it would be leaning on something it doesn't actually have for this week. So we have a simple rule: if a fact is missing for more than half of this week's matches, we drop it for this run.

And the lovely part is, we don't have to remember to switch it back on. The moment the data catches up, the fact comes back by itself on the next run. No code change needed.

> **📁 No new files here.** This rule lives inside `plpredict/models/outcome.py`, which we made in step seven.

## Step ten: are we actually any good?

Okay, so we have models. Now the most important question: are they good? And we have to answer it honestly.

So we write `plpredict/models/evaluate.py`, which does a **backtest**. We go back to past seasons and pretend we're living through them. Predict gameweek 4 using only what was known before gameweek 4.

Then gameweek 5. Then 6. All the way through.

The golden rule from step four keeps it honest. As the file itself says, a random split would flatter the model, because it would let it learn from matches that hadn't happened yet.

And we add a command to run it: `scripts/backtest.py`. You tell it which seasons, and it does the rest.

We did this over three seasons, 2023-24 to 2025-26, from gameweek 4 onwards. That's 1,050 matches. Here's what came out:

- It picked the right result **52%** of the time. Always guessing "home win" gets about **43%**. Football is very unpredictable, so that's solid.
- Its chances are honest. When it said about 65%, that result happened about 65% of the time.
- It got the exact score right about **8%** of the time. That's normal, exact scores are hard.
- It also beat a simpler Elo-only version on log loss, 0.99 against 1.01.

One honest note: those numbers come from the project's own report. The backtest needs the source archives to re-run.

> **📁 Files we just created**
> - `plpredict/models/evaluate.py`: the honest walk-forward test.
> - `scripts/backtest.py`: a command to run that test on chosen seasons.

## Step eleven: predict for real, and keep the record

Now we put it all together into one chain, a pipeline, in `plpredict/pipeline/run.py`. Every run goes: load the data, rebuild the facts, train both models, predict the next gameweek, and save it.

Here's the heart of it in real code. Look how it decides what the model is allowed to learn from:

```python
cutoff = gameweek_cutoff(features, season, matchday)
train = league[
    (pd.to_datetime(league["match_date"]).dt.date < cutoff) & league["result"].notna()
]
```

The cutoff is the gameweek's first kick-off. The model only trains on matches before it. That's the golden rule again, showing up one more time. You see how it keeps linking back?

Then we save. And here's the honesty part. Every run adds a *new* record.

We never overwrite an old prediction. If a prediction is saved after a gameweek has already kicked off, it gets marked "late", so nobody's fooled into thinking it was made in advance.

There's one more clever safety net. Remember, the working database gets rebuilt every run. So what stops a rebuild from wiping out our history?

Before predicting, the pipeline restores the published history first. So a rebuild can never erase the record.

And the command that runs all of this is `scripts/run_pipeline.py`. It's "the job to run after every gameweek". It also has a handy setting that just stops if nothing has changed since the last run. Remember that, the morning robot will use it.

Let's circle back one more time. We've got data, facts, two models, an honest test, and a pipeline that predicts and keeps a permanent record. The brain of the project is finished. Now we just need to show it to the world, and make it run by itself.

> **📁 Files we just created**
> - `plpredict/pipeline/run.py`: the full chain: load, build facts, train, predict, save.
> - `scripts/run_pipeline.py`: the command to run the pipeline after each gameweek.

## Step twelve: the website

Now the website. It's built with Flask, a small, simple Python tool for making web pages. It shows each gameweek, the season so far, and how the model works. It shows the three chances, the predicted score next to the real one, a tick or cross for whether the result was right, and the "late" marker.

We split it into a few files, each with one job.

`plpredict/web/queries.py` fetches what each page needs from the database. It only ever reads. It never trains or predicts anything. That's what lets the website keep showing last week's predictions, even while the pipeline is busy retraining.

`plpredict/web/app.py` is the website itself. It's deliberately thin. No model is loaded, nothing is calculated when someone visits. Every page is just a question to the database.

Then the look of the pages. In `plpredict/web/templates/`, `base.html` is the shared frame every page sits in. On top of that sit `gameweek.html` for one gameweek's predictions, `season.html` for the season so far, `model.html` explaining how the model works, and `404.html` for "page not found".

There's also `_match.html`, a small reusable block that draws one match, so every page shows matches the same way. And `plpredict/web/static/style.css` holds the colours and layout.

To try it on your own computer, there's `scripts/serve.py`. It starts the website locally.

Now, here's a problem we have to solve. The website is hosted for free on Vercel, which runs code only when someone visits. That's great, but it has two catches. The disk is read-only, and there's a size limit.

Remember our database is 61 MB? Well, about 53 MB of that is the facts table, and no web page ever looks at it.

So we write `plpredict/web/export.py`, plus a command for it, `scripts/export_web_db.py`. Together they make a slim copy with only what the pages need: Premier League fixtures, the predictions, and the model details. That copy is `data/web/plpredict-web.db`, and it's about 1.4 MB.

It's small enough to save with the code, and it's made in a way that's safe to open read-only. Remember the `.gitignore` exception in step zero? This is the one database we *do* save, because the website needs it.

And remember the two install lists from step zero? Here's the payoff. Vercel only installs `requirements.txt`, which is just Flask.

None of the heavy learning tools. The website never does any maths. It's like a shop window: it shows the finished goods, and the workshop stays out back.

Finally, two small files tell Vercel how to run it. `api/index.py` is the front door Vercel uses to start the app. And `vercel.json` tells Vercel which files to include, like the slim database, the templates and the stylesheet, and to send every web address to that front door.

We test the pipeline and website together in `tests/test_pipeline_and_web.py`.

> **📁 Files we just created**
> - `plpredict/web/queries.py`: fetches what each page needs, read-only.
> - `plpredict/web/app.py`: the website itself, kept very thin.
> - `plpredict/web/templates/base.html`: the shared frame for every page.
> - `plpredict/web/templates/gameweek.html`: one gameweek's predictions.
> - `plpredict/web/templates/season.html`: the season so far.
> - `plpredict/web/templates/model.html`: how the model works.
> - `plpredict/web/templates/_match.html`: a reusable block that draws one match.
> - `plpredict/web/templates/404.html`: the "page not found" page.
> - `plpredict/web/static/style.css`: colours and layout.
> - `scripts/serve.py`: runs the website on your own computer.
> - `plpredict/web/export.py`: builds the slim website database.
> - `scripts/export_web_db.py`: the command to build it.
> - `data/web/plpredict-web.db`: the slim 1.4 MB database the website reads.
> - `api/index.py`: the front door Vercel uses to start the website.
> - `vercel.json`: tells Vercel what to include and how to route pages.
> - `tests/test_pipeline_and_web.py`: tests the pipeline and website together.

## Step thirteen: make it run by itself

Now the final piece. We want this to run every day without us.

So we write `.github/workflows/update-predictions.yml`. That's a set of instructions for GitHub Actions, a free service that runs code at set times. It runs every morning at 6 a.m. UTC.

Why every day, and not once a week? Because fixtures don't follow a neat weekly timetable. International breaks, midweek games, gameweeks spread over four days. So rather than guess, it checks every morning.

Here's what the morning robot does, in order:

1. Pull the new results.
2. Ask: has anything changed since the website was last updated? If not, stop. That run takes about a minute and changes nothing. (That's the "stop if nothing changed" setting in `run_pipeline.py`.)
3. If yes, retrain and predict the next gameweek.
4. Export the slim website database.
5. Test that the website can actually open it and show its pages, before saving anything.
6. Save and push the update.
7. Check that every played gameweek has a prediction, and that the next one is forecast. That's `scripts/check_publication.py`.
8. Visit the real, live website and check it's showing the right gameweek. That's `scripts/check_live_site.py`.

And the two checking scripts get their own test file, `tests/test_publication_checks.py`.

> **📁 Files we just created**
> - `.github/workflows/update-predictions.yml`: the morning robot's instructions.
> - `scripts/check_publication.py`: checks every gameweek has a prediction.
> - `scripts/check_live_site.py`: visits the live website to check it's up to date.
> - `tests/test_publication_checks.py`: tests those two checks.

## The story behind those last checks

Those last two checks have a story, and it's a great lesson, so let me tell it.

At one point, the project's main branch got renamed. The morning robot carried on, pushing updates to the new name, and every run went green. Success, success, success.

But the website host was still watching the *old* branch name. So the live site just sat there, frozen, out of date.

The project's own notes describe it perfectly: "Eleven consecutive successful runs, a static site." From the inside, everything looked perfect.

So fixes came out of it. After pushing, the robot also moves any other deployment branch forward, but only safely, never forcing over someone else's work. `check_publication.py` makes sure "nothing published" really means "nothing to do", not "quietly broken".

And `check_live_site.py` is the real lesson: it checks the far end of the chain, the live website itself. Because "all green" doesn't mean "working". The only way to know the site is current is to go and look at it.

## And the tests

Let's gather the tests in one place, since we made them step by step. There are 98 of them across nine test files: the results reader, the team names, the facts, the models, the pipeline and website, the publication checks, and (added later) gameweeks, managers and FPL records. They all run on the small pretend league from `conftest.py`. If a change breaks something, a test fails before it reaches the public.

## The last file: writing it all down

There's one more file, and it's written last: `docs/HOW_IT_WAS_BUILT.md`. It's the project's own record of every design decision: what each requirement demanded, how it was met, and where it lives in the code. A lot of what I've told you today comes from it.

One small honest note: it was written a little earlier than the final code, so some numbers are slightly behind. For example, it says 55 tests and 213 facts, while the code now has 98 tests and the README says 216 facts.

> **📁 Files we just created**
> - `docs/HOW_IT_WAS_BUILT.md`: the project's own record of its design decisions.

## What we added after launch

Once the season started, three of the gaps got worked on. Let's walk through them.

**First, postponed matches.** A match is listed under the gameweek it was first scheduled in. But when it's postponed, it's actually played weeks or months later. So its result was showing up in the facts for matches that came *before* it, which breaks our golden rule.

So we write `plpredict/data/gameweeks.py`. It puts every match in the gameweek it's actually *played* in, and keeps the original one as `original_matchday`. That moved 190 matches, and cut the leaks from 258 (the worst was 185 days into the future) down to 6, none more than four days.

It fixed two other things too. A postponed match no longer freezes the website on an old gameweek, and the site now shows "Rearranged from GW8" next to it. We also get a new fact, `gameweek_fixtures`, so the model knows when a team plays twice in one gameweek.

**Second, manager history.** Our hand-written `managers.csv` only covered about a season and a half, and it had a mistake in it. So we pull the history from Wikidata instead, a free public database anyone can edit.

`plpredict/data/sources/wikidata_managers.py` asks Wikidata for every club's managers and keeps only the head coaches. `data/manual/team_wikidata.csv` tells it each club's Wikidata ID, and the answers are saved in `data/external/managers_wikidata.csv`. `scripts/sync_managers.py` runs this every morning, before the rest of the robot's work.

Now 84% of matches since 2010-11 have a known manager, and nearly all of them from 2018-19 on. The old `managers.csv` stays, but only fills in what Wikidata misses. And because Wikidata can be slow to update, `plpredict/data/sources/manager_news.py` and `scripts/check_manager_news.py` watch BBC Sport headlines for a sacking. That only raises a flag for a person to check; it never changes the data.

**Third, injuries.** The FPL website (the official fantasy football game) shows every player's status, like "injured" or "75% chance of playing". The catch is it keeps no history: once the news changes, the old news is gone.

So `plpredict/data/sources/fpl.py` reads it, and `scripts/snapshot_fpl.py` saves it every day into `data/snapshots/fpl_availability.csv`. It only writes down what *changed*, so the file stays small. This started on 30 September 2026, and it isn't a model fact yet: a few weeks of records is too little to learn from.

**And what didn't make it.** We also tried adding expected goals (xG) from Understat, an injury guess, and a longer-memory xG rating. We tested each one over 2,660 past matches, from 2019-20 to 2025-26, and none was measurably better. So they weren't shipped. Being willing to drop your own ideas is part of testing honestly.

> **📁 Files we just created**
> - `plpredict/data/gameweeks.py`: puts each match in the gameweek it's actually played in.
> - `plpredict/data/sources/wikidata_managers.py`: pulls every club's managers from Wikidata.
> - `data/manual/team_wikidata.csv`: each club's Wikidata ID.
> - `data/external/managers_wikidata.csv`: the saved manager history.
> - `scripts/sync_managers.py`: refreshes the manager history every morning.
> - `plpredict/data/sources/manager_news.py`: reads BBC Sport headlines about managers.
> - `scripts/check_manager_news.py`: flags a manager change Wikidata hasn't caught yet.
> - `plpredict/data/sources/fpl.py`: reads player availability from the FPL website.
> - `scripts/snapshot_fpl.py`: saves the day's availability changes.
> - `data/snapshots/fpl_availability.csv`: the daily availability record.
> - `tests/test_gameweeks.py`, `tests/test_managers.py`, `tests/test_fpl_snapshots.py`: tests for all three.

## So, how's it doing live?

Let's be honest about the live season. After the first five gameweeks of 2026-27, it got 21 out of 50 matches right, which is 42%. But here's the catch: only gameweek 4 was truly published before kick-off.

Gameweeks 1 to 3 were filled in afterwards, and gameweek 5 was saved after its first match. So it's far too early to judge anything. That's exactly why we have the "late" marker. Gameweek 6, starting on 10 October after the international break, was forecast well before kick-off.

It keeps us honest.

## What's still missing?

Every project has gaps, and it's good to know them:

- **Injuries** are now recorded daily, but there's too little history for the model to use yet.
- **Manager history** is thinner before 2016.
- **There's no "expected goals" data.** It was tested and didn't measurably help, so shots on target still stand in for it.
- **Draws** are almost never picked, though the chance is always shown.
- **It's Premier League only**, and it rebuilds all the facts from scratch every run. Fine now, but at a bigger scale you'd want to update only what changed.

## Let's put it all together

So let's step back and look at the whole thing in one breath.

We **set up the workshop**: the README, two install lists, and one settings file. We started with **data** from free sources, including lower leagues and cups, each with its own reader. We **cleaned the team names** and put everything in **one database**. We agreed on a **golden rule**, no peeking at the future, and we built it into the code so it can't be broken.

We put every team on **one Elo ladder**, so promoted teams bring their history with them. We turned every match into about **200 facts**.

Then we built **two result models** and blended them, plus a **separate score model**. We **dropped late data** automatically. We **tested honestly** on three past seasons and got 52% against 43%. We built a **pipeline** that predicts and keeps a **permanent record**, with a "late" marker for honesty.

Finally, we built a **small website** with a slim database of its own, made a **morning robot** to run everything, and added **checks** right up to the live website, because we learned the hard way that green doesn't mean working.

And notice how it all links together. The golden rule from step four shows up again in the facts, in the backtest, and in the pipeline. The promoted-team problem from step one gets solved by Elo in step five and by the score model in step eight.

The two install lists from step zero are what let the website fit on a free host in step twelve. Every piece is there for a reason.

That's the project. Facts, not rules. Honest from start to finish.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
