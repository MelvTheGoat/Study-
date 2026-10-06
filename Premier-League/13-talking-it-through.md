# Premier League Predictor: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. Every now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

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

## Step one: we start with the data

We start with data, because nothing else can happen without it. You can't build facts without results, and you can't train a model without facts.

Our main source is called openfootball. It's a free, public collection of English football results and fixtures. The nice thing is it's stored as a git repository, so the first time we download the whole thing, and after that we just pull the new bits each day. Quick and cheap.

Now, here's a smart choice. We don't only read the Premier League. We also read the Championship, League One, the FA Cup and the EFL Cup.

Why? Two reasons. First, if a team played a cup match on Wednesday, we want to know they're tired on Saturday.

Second, and this is a big one, when a team gets promoted, we want to know how good they were in the division below. Hold onto that, we'll come back to it with Elo.

Then we add some extras:

- **Match stats** like shots, corners and cards, from another free source. Useful, but this source runs weeks behind the live season. We'll have to deal with that later.
- **Squad ratings** from the FIFA and EA FC video games. These tell us a squad's quality "on paper". They're built offline into one simple file per club per season.
- **Hand-written files** for things you can't get from results: manager spells, European competitions, and team name spellings. There's also a file for injured players, but honestly, it ships empty. Injuries are the hardest thing to automate, so for now the model works without them.

## Step two: clean up the team names

Before we store anything, we have a small but important job. Different sources spell clubs differently. One says "Man City", another says "Manchester City". If we don't fix that, the computer thinks they're two different clubs, and the history gets split in half.

So we have a little translator that makes sure every club always has one name. Here's a real line from it:

```python
"Manchester City": "Man City",
```

Boring? Yes. But skip it and everything after it is quietly wrong.

## Step three: put it all in one place

Now we load everything into a database. We use SQLite, which is a database that lives in a single file. No server to run, nothing to set up.

This working database ends up around 61 MB, and it gets rebuilt on each run. That's fine, it's quick, and it means we never have old junk lying around.

So let's pause and look at where we are. We've got our data. The names are clean.

It's all sitting in one database. Good. That part is set.

Now we can do the clever stuff.

## Step four: the golden rule, no peeking

Okay, before we build a single fact, we need to agree on a rule. This is the most important idea in the whole project, so let's take our time.

Imagine you're testing a model on last season. You ask it to predict a match in March. If, by accident, one of its facts uses results from April, the model looks amazing.

But that's cheating. In real life, in March, you don't know what happens in April. Experts call this **leakage**, and it's the most common way people fool themselves.

So how do we make sure it never happens? We don't just promise to be careful. We build the system so it *can't* happen.

Here's how. We walk through history in date order, one step at a time. Each team has a little "state", a running record of its recent results, its table position and so on.

And that state only ever moves forward. It's like reading a diary one page at a time. You literally can't see tomorrow's page because you haven't turned to it yet.

And there's one more subtle bit. A gameweek isn't one day. It might start on Friday and finish on Monday.

So should Monday's match know Saturday's results? No. Because when we publish predictions, we publish the whole gameweek before the first kick-off.

So every match in a gameweek uses the state from *before the first kick-off*.

Here's the real code. Look at the order:

```python
for match in group:
    rows.append(self._featurise(match, table, len(members)))
for match in group:
    _apply_result(match, self.states, self.elo, self.head_to_head)
```

See it? First we write down the facts for *every* match in the gameweek. Only after that do we feed in the results.

So no match in the gameweek can see another match's result. That's the golden rule, written in code.

## Step five: one ladder for every team

Now, remember I said hold onto the promoted teams? Here's where it pays off.

We give every team an **Elo rating**. Elo is a single strength number. You win, it goes up.

You lose, it goes down. Beat a strong team and it goes up more. Chess uses the same idea.

The clever part is that we put the Premier League, the Championship and League One all on *one* ladder. Lower divisions just start lower. Here are the real numbers from the settings:

```python
"premier_league": 0.0,
"championship": -180.0,
"league_one": -320.0,
```

So a brand-new Championship team starts 180 points below an average Premier League team. Why does this matter? Because when a team gets promoted, most models know nothing about them.

No Premier League form, no table position, no history. But on our ladder, they bring their whole Championship season with them.

A few other small touches. Every summer, ratings get pulled 25% of the way back to average, because squads change. Home teams get a 60-point boost. And cup matches count a bit less, because teams often rest players in cups.

## Step six: turn every match into facts

Now we're ready to build the facts. For every match, we create about 200 of them. Let me give you the families, so you see the shape:

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

Let's circle back again. We have clean data in one place. We have a golden rule that's built into the code.

We have one ladder for all divisions. And we have about 200 honest facts for every match. That whole "facts" side is done.

Now we can finally teach the computer.

## Step seven: the result model

Now we build the model that predicts home, draw or away. And here we make a decision that I really like. We don't use one model. We use two, and we blend them.

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

## Step eight: the score model

Guessing exact scores is a different job, so it gets its own model. It's called **Dixon-Coles**, and the idea is simple once you hear it.

Every team gets an attack strength and a defence strength. The home team's expected goals is its attack, times the other team's defence, times a home advantage. Same the other way round. From those two numbers, you can work out the chance of every score, 0-0, 1-0, 2-1, up to 8 goals each.

A few smart touches:

- Plain maths gets low scores like 0-0 and 1-1 slightly wrong, so there's a small correction for them.
- Recent matches count more. A match about two years old counts about half as much as today's.
- We fit Championship matches too. Otherwise a promoted team that scored once in three games looks like it can't score at all.
- We gently pull extreme ratings back towards average, so one lucky afternoon doesn't create a superstar.

And one thing users appreciate: the score we show always agrees with the predicted result. So you never see "home win, 1-1" on the website.

## Step nine: what about data that arrives late?

Remember the match stats that run weeks behind? Here's where we deal with them.

Say we're predicting gameweek 5, and shots data only exists up to gameweek 2. If we let the model use the shots feature, it would be leaning on something it doesn't actually have for this week. So we have a simple rule: if a fact is missing for more than half of this week's matches, we drop it for this run.

And the lovely part is, we don't have to remember to switch it back on. The moment the data catches up, the fact comes back by itself on the next run. No code change needed.

## Step ten: are we actually any good?

Okay, so we have models. Now the most important question: are they good? And we have to answer it honestly.

So we do a **backtest**. We go back to past seasons and pretend we're living through them. Predict gameweek 4 using only what was known before gameweek 4.

Then gameweek 5. Then 6. All the way through.

The golden rule from step four keeps it honest.

We did this over three seasons, 2023-24 to 2025-26, from gameweek 4 onwards. That's 1,050 matches. Here's what came out:

- It picked the right result **52%** of the time. Always guessing "home win" gets about **43%**. Football is very unpredictable, so that's solid.
- Its chances are honest. When it said about 65%, that result happened about 65% of the time.
- It got the exact score right about **8%** of the time. That's normal, exact scores are hard.
- It also beat a simpler Elo-only version on log loss, 0.99 against 1.01.

One honest note: those numbers come from the project's own report. The backtest needs the source archives to re-run.

## Step eleven: predict for real, and keep the record

Now we put it all together into one chain, a pipeline. Every run goes: load the data, rebuild the facts, train both models, predict the next gameweek, and save it.

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

Let's circle back one more time. We've got data, facts, two models, an honest test, and a pipeline that predicts and keeps a permanent record. The brain of the project is finished. Now we just need to show it to the world, and make it run by itself.

## Step twelve: the website

The website is built with Flask, a small, simple Python tool for making web pages. It shows each gameweek, the season so far, and how the model works. It shows the three chances, the predicted score next to the real one, a tick or cross for whether the result was right, and the "late" marker.

Now, here's a problem we have to solve. The website is hosted for free on Vercel, which runs code only when someone visits. That's great, but it has two catches. The disk is read-only, and there's a size limit.

Remember our database is 61 MB? Well, about 53 MB of that is the facts table, and no web page ever looks at it.

So we export a slim copy with only what the pages need. That copy is about 1.4 MB. It's small enough to save with the code, and safe to open read-only.

And for the same size reason, the website only installs Flask. None of the heavy learning tools. The website never does any maths. It's like a shop window: it shows the finished goods, and the workshop stays out back.

## Step thirteen: make it run by itself

Now the final piece. We want this to run every day without us.

So we use GitHub Actions, a free service that runs code at set times. It runs every morning at 6 a.m. UTC.

Why every day, and not once a week? Because fixtures don't follow a neat weekly timetable. International breaks, midweek games, gameweeks spread over four days. So rather than guess, it checks every morning.

Here's what the morning robot does, in order:

1. Pull the new results.
2. Ask: has anything changed since the website was last updated? If not, stop. That run takes about a minute and changes nothing.
3. If yes, retrain and predict the next gameweek.
4. Export the slim website database.
5. Test that the website can actually open it and show its pages, before saving anything.
6. Save and push the update.
7. Check that every played gameweek has a prediction, and that the next one is forecast.
8. Visit the real, live website and check it's showing the right gameweek.

## The story behind that last check

That last step has a story, and it's a great lesson, so let me tell it.

At one point, the project's main branch got renamed. The morning robot carried on, pushing updates to the new name, and every run went green. Success, success, success.

But the website host was still watching the *old* branch name. So the live site just sat there, frozen, out of date.

The project's own notes describe it perfectly: "Eleven consecutive successful runs, a static site." From the inside, everything looked perfect.

So two fixes came out of it. First, after pushing, the robot also moves any other deployment branch forward, but only safely, never forcing over someone else's work. Second, and this is the real lesson, the robot now checks the far end of the chain: the live website itself.

Because "all green" doesn't mean "working". The only way to know the site is current is to go and look at it.

## And the tests

Running underneath all of this are 67 automatic tests. They check the facts, the models, the openfootball file reader, the pipeline and website, the publication checks, and the team names. If a change breaks something, a test fails before it reaches the public.

## So, how's it doing live?

Let's be honest about the live season. After the first five gameweeks of 2026-27, it got 21 out of 50 matches right, which is 42%. But here's the catch: only gameweek 4 was truly published before kick-off.

Gameweeks 1 to 3 were filled in afterwards, and gameweek 5 was saved after its first match. So it's far too early to judge anything. That's exactly why we have the "late" marker.

It keeps us honest.

## What's still missing?

Every project has gaps, and it's good to know them:

- **Injuries** aren't collected automatically. That file is empty for now.
- **Manager history** only starts from 2025-26, so there's not much to learn from yet.
- **There's no "expected goals" data.** Shots on target stand in for it.
- **Draws** are almost never picked, though the chance is always shown.
- **It's Premier League only**, and it rebuilds all the facts from scratch every run. Fine now, but at a bigger scale you'd want to update only what changed.

## Let's put it all together

So let's step back and look at the whole thing in one breath.

We started with **data** from free sources, including lower leagues and cups. We **cleaned the team names** and put everything in **one database**. We agreed on a **golden rule**, no peeking at the future, and we built it into the code so it can't be broken.

We put every team on **one Elo ladder**, so promoted teams bring their history with them. We turned every match into about **200 facts**.

Then we built **two result models** and blended them, plus a **separate score model**. We **dropped late data** automatically. We **tested honestly** on three past seasons and got 52% against 43%. We built a **pipeline** that predicts and keeps a **permanent record**, with a "late" marker for honesty.

Finally, we built a **small website**, made a **morning robot** to run everything, and added **checks** right up to the live website, because we learned the hard way that green doesn't mean working.

And notice how it all links together. The golden rule from step four shows up again in the facts, in the backtest, and in the pipeline. The promoted-team problem from step one gets solved by Elo in step five and by the score model in step eight. Every piece is there for a reason.

That's the project. Facts, not rules. Honest from start to finish.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
