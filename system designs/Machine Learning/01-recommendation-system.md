# Recommendation System

A recommendation system is software that suggests things you might like. When Netflix shows "Because you watched...", or YouTube fills your home page with videos, that is a recommendation system at work. It matters because most of what people watch, read or buy online is found through these suggestions.

## Key Terms

- **User**: a person using the app.
- **Item**: anything we can recommend, like a video, a song or a product.
- **Interaction**: something a user does with an item, like a click, a watch, a like or a purchase.
- **Candidate generation**: quickly picking a few hundred possible items out of millions.
- **Ranking**: carefully scoring those few hundred items and putting the best ones on top.
- **Embedding**: a list of numbers that describes a user or an item, so that similar things get similar numbers.
- **Latency**: how long the user waits for the page to load.
- **Cold start**: when a user or item is brand new, so we know almost nothing about it yet.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Ask what "a good recommendation" means for this app. On a video app it might be "videos people actually watch", not just "videos people click". Getting the goal right matters more than any clever model (a program that learned patterns from past data).

**Step 2: Figure out the data.** List what you know: what each user watched, liked or skipped, and basic facts about each item. This history is the fuel for everything else. No data means no personal suggestions.

**Step 3: Sketch the main parts.** Most recommendation systems have two stages: a fast stage that finds a shortlist, and a careful stage that orders it. Draw these as boxes before thinking about details.

**Step 4: Walk through one request.** Imagine one user opening the app and follow what happens, box by box. This shows you where time is spent and what data each part needs.

**Step 5: Decide how to know it works.** Pick a few simple numbers to watch, like how often people click a suggestion. Without these, you are guessing.

**Step 6: Plan how to keep it working.** Tastes change and new items arrive every day. Think about how the system stays fresh and what happens when something breaks.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design the home page for a video app.

- **10 million** users and **1 million** videos.
- Show **20** suggested videos when a user opens the app.
- Load in under **200 milliseconds** (0.2 seconds), so it feels instant.

How busy will it be? Let's do the math step by step:

1. 10 million users each open the app about 5 times a day.
2. 10,000,000 × 5 = 50,000,000 requests per day.
3. A day has about 100,000 seconds (86,400, rounded up).
4. 50,000,000 ÷ 100,000 = **500 requests per second** on average.

### The Data We Use (Step 2)

- **Watch history**: which videos each user watched, and for how long.
- **Item facts**: the topic, the length and the upload date of each video.
- **Context**: the time of day and the device (phone or TV).

Every time we show a video, we also record whether the user watched it. That record teaches the system what "good" looks like.

### The Big Picture (Step 3)

```
   [User opens app]
          |
          v
   [Recommendation Service]
          |
          v
   [Candidate Generation]  -- 1,000,000 videos -> 500
          |
          v
   [Ranking Model] <------- [Feature Store]
          |
          v
   [Final Filter]          -- 500 -> best 20
          |
          v
   [Home page shows 20 videos]
```

Step by step:

1. The user opens the app, and the app asks the Recommendation Service (a program on our servers that answers these requests) for suggestions.
2. Candidate Generation quickly grabs about 500 videos this user might like, out of 1 million.
3. The Ranking Model gives each of the 500 videos a score, using facts from the Feature Store.
4. The Final Filter removes videos the user already watched and mixes in some variety.
5. The top 20 videos appear on the home page, all in under 0.2 seconds.

### The Main Parts (Step 3)

**Candidate Generation.** Its job is to be fast, not perfect. Think of a shop assistant who quickly pulls a few racks of clothes in your style before you look closely. A common trick is to give every user and video an embedding, then grab the videos whose numbers sit closest to the user's.

An embedding works like a map: similar videos sit close together, and "find videos near this user" is a very fast question to answer.

**Feature Store.** This is storage for ready-made facts, like "this user watched 12 cooking videos this week". It is like a well-stocked pantry: the cook doesn't grow vegetables when an order arrives, the ingredients are already waiting.

**Ranking Model.** This model looks carefully at each of the 500 candidates and predicts how likely this user is to watch it. It's like a talent-show judge who only scores the finalists, not everyone who applied. With only 500 videos to judge, it can afford to be smarter.

**Final Filter.** This step hides videos already watched and avoids showing five videos from the same channel. It is like a good DJ who won't play the same artist five times in a row.

**Training (behind the scenes).** Once a day, we retrain the models on yesterday's watch history, so they learn new tastes. This happens offline (in the background, not while users wait).

A popular model for candidate generation is called a "two-tower model" (advanced - skip for now).

### How It Answers a Request (Step 4)

The time budget is 200 milliseconds, split roughly like this:

- 30 ms to find candidates.
- 30 ms to fetch facts from the Feature Store.
- 100 ms to score 500 videos.
- 40 ms left over for filtering and sending the page.


### How We Know It's Working (Step 5)

- **Click-through rate (CTR)**: the share of shown videos that get clicked. If we show 20 and the user clicks 2, CTR is 2 ÷ 20 = 10%.
- **Watch time**: how many minutes people watch from our suggestions. This catches "clickbait" that gets clicks but no watching.
- **A/B test**: show the new system to half the users and the old one to the other half, then compare. It is like a fair taste test between two recipes.

### What Can Go Wrong (Step 6)

- **Cold start.** A new user has no history. Show popular videos first, or ask them to pick a few interests.
- **Too much of the same.** If someone watches one cat video, they may get only cats forever. Mix in some variety on purpose.
- **Too slow.** Save ready-made results for very active users in Redis (a very fast storage for data you need instantly).
- **Out-of-date model.** Tastes change. Retrain daily, and watch the click rate so you notice when it drops.

## Quick Recap

- Recommendation systems use two stages: a fast shortlist (candidate generation), then careful ordering (ranking).
- Embeddings turn users and videos into numbers, so "similar" becomes "close together".
- Measure what users really value, like watch time, not just clicks.
- Plan for new users, variety and freshness from day one.
