# nyc-taxi-demand-forecast: Explained Simply

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A system that predicts how many taxi pickups there will be in each area of New York, for each hour of the next day, so a taxi company can send drivers to the right places, and that looks after itself.

## The everyday comparison

Think about a bakery deciding how much bread to bake for tomorrow.
- Bake **too little**, and customers leave empty-handed and may not come back. That's bad.
- Bake **too much**, and you throw some away. Annoying, but not as bad.

A smart baker doesn't bake for an "average day". They bake a bit **extra**, because running out costs more than leftovers.

My system does the same for taxis. I set it up so that **not having enough drivers** counts as **three times worse** than having too many. With that setup, the maths says the best prediction to act on is a bit higher than the average: roughly the level you'd only exceed one day in four. The system checks this against past data and picks the best level automatically.

**The surprise:** using the "average" prediction instead of this smarter level would cost **25% more**. That's almost as big a difference as choosing a good prediction method over a bad one.

## How it works, step by step

1. **Collects trip records** automatically. If it already has a file, it doesn't download it again.
2. **Cleans the data** with nine clear rules (e.g. removing impossible trips), counting what it removed.
3. **Builds hour-by-hour totals** for each area, with information like time of day, day of week and public holidays.
4. **Predicts tomorrow** using only information available *today*. It's carefully tested so it never "peeks" at the answer.
5. **Tests itself on the past** as if it were predicting live, several times over.
6. **Watches for changes**, like some areas suddenly getting busier and others quieter, and **retrains** when needed.
7. **Only switches to a new version if it's clearly better** (at least 2% better).
8. **Answers requests** from a dispatch app through a simple web service.

## Honest limits

- It was tested on **realistic made-up data**, because the real New York data website was blocked where I built it.
- It doesn't yet use **weather or events**, which matter a lot in real life.

## What this shows about me

- I start from the business decision, not from the technology.
- I build complete systems that run and check themselves.
- I'm careful about testing fairly and honestly.
- I report what didn't work, not just what did.
