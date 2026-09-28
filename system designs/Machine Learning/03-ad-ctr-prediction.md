# Ad Click Prediction (CTR Prediction)

Every time you scroll Instagram or search on Google, the site has a split second to choose which ad to show you. To choose well, it predicts how likely you are to click each ad. This matters because ads pay for most free apps, and showing the right ad keeps both users and advertisers happy.

## Key Terms

- **Impression**: one time an ad is shown to a user.
- **Click-through rate (CTR)**: clicks divided by impressions. 2 clicks from 100 impressions = 2% CTR.
- **pCTR**: the *predicted* CTR, meaning our model's guess of how likely this user is to click this ad. (A model is a program that learned patterns from past data.)
- **Bid**: how much an advertiser will pay for one click.
- **Auction**: the quick contest that decides which ad wins the spot.
- **Feature**: one useful fact given to the model, like "user's age group" or "ad category".
- **Calibration**: whether our predicted chances match reality. If we say 2%, about 2 in 100 should really click.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The site wants to show ads that users actually care about and that earn money. That means we need a good guess of the click chance for every possible ad.

**Step 2: Figure out the data.** Every impression is a lesson: the ad was shown, and it was either clicked or not. We have millions of these lessons every day. Most of them are "not clicked", which makes learning tricky.

**Step 3: Sketch the main parts.** We need to find ads that fit the user, predict the click chance for each, and pick a winner. Draw these as boxes.

**Step 4: Walk through one ad request.** Follow what happens when the user scrolls to an ad slot. The whole decision must happen in a blink, around 0.1 seconds.

**Step 5: Decide how to know it works.** Check both "are the predictions accurate?" and "did money and clicks go up?". Both matter.

**Step 6: Plan for problems.** People's interests shift, new ads arrive with no history, and some clicks are fake. Think about each before launch.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design ad selection for a social media app.

- **10,000** ad requests per second.
- For each request, consider about **100** eligible ads.
- Decide in under **100 milliseconds** (0.1 seconds).

How many predictions is that? Step by step:

1. 10,000 requests per second.
2. × 100 ads per request.
3. = **1,000,000 predictions per second.**

So the model must be fast. Each prediction gets only a tiny slice of time.

### Why the Prediction Matters (Step 1)

The site usually earns money only when someone clicks. So the value of showing an ad is:

**Expected value = bid × pCTR**

Here's an example with two ads:

- **Ad A**: bid $2.00, pCTR 1%. Value = $2.00 × 0.01 = **2 cents** per showing.
- **Ad B**: bid $0.50, pCTR 5%. Value = $0.50 × 0.05 = **2.5 cents** per showing.

Ad B wins, even though it pays less per click, because people are much more likely to click it. This is why accurate click predictions are so valuable: they help both the site and the user.

### The Data We Use (Step 2)

- **User facts**: age group, interests, recent activity.
- **Ad facts**: category, advertiser, image or text, past CTR.
- **Context**: time of day, device, where on the page the ad appears.
- **The answer**: was it clicked (1) or not (0)?

Clicks are rare. With a 1% CTR, 99 out of 100 examples are "no click". The model has to learn from a small number of "yes" examples.

### The Big Picture (Step 3)

```
   [User scrolls to ad slot]
              |
              v
   [Ad Server]
              |
              v
   [Candidate Ads]  -- ads that target this user (~100)
              |
              v
   [CTR Model] <--------- [Feature Store]
              |
              v
   [Auction: bid x pCTR]
              |
              v
   [Winning ad shown] ----> [Click Logs] -> retrain model
```

Step by step:

1. The user scrolls, and the app asks the Ad Server (a program on our servers that handles ad requests) for an ad.
2. Candidate Ads finds about 100 ads whose targeting fits this user, for example "sports fans aged 18-30".
3. The CTR Model predicts the click chance for each ad, using facts from the Feature Store.
4. The Auction multiplies each bid by its pCTR and picks the highest value.
5. The winning ad is shown. Whether it gets clicked is saved in the Click Logs, which are used to retrain the model.

### The Main Parts (Step 3)

**Candidate Ads.** Advertisers choose who they want to reach. This step keeps only ads allowed for this user. It's like a bouncer checking the guest list before anyone gets into the party.

**Feature Store.** A fast store of ready-made facts about users and ads, such as "this user clicked 3 sports ads this week". It's like a chef's prepped ingredients: everything is chopped and waiting, so cooking is quick.

**CTR Model.** This model turns the features into a click chance. It's like an experienced shopkeeper who can glance at a customer and guess what they'll pick up. Simple, fast models are common here because of the huge number of predictions.

Some systems use deep learning models that learn feature combinations automatically (advanced - skip for now).

**Auction.** It ranks ads by bid × pCTR and picks the winner. Think of it as a fair referee: the highest *expected* value wins, not just the biggest bid.

**Click Logs and Retraining.** Every impression and click is saved. The model is retrained regularly, often daily, so it keeps up with changing interests.

### How We Know It's Working (Step 5)

- **Calibration check**: group all ads we predicted at about 2%. Did about 2% of them get clicked? If we say 2% but reality is 1%, the auction makes bad choices.
- **CTR**: are more of the shown ads getting clicked than before?
- **Revenue per 1,000 impressions**: how much money we earn every 1,000 ad showings. This is a simple, common business number.
- **A/B test**: give half the users the new model and half the old one, then compare. It's like a fair taste test between two recipes.

### What Can Go Wrong (Step 6)

- **New ads with no history.** The model has little to go on. Give new ads a fair starting guess (like the average CTR for their category) and a small amount of exposure to learn.
- **Interests change.** A model trained last month may be out of date. Retrain often and watch calibration every day.
- **Fake clicks.** Bots can click ads to waste budgets. Filter out suspicious clicks before they reach training data.
- **Slow predictions.** At a million predictions per second, even small delays add up. Keep the model simple and features pre-computed.

## Quick Recap

- CTR prediction guesses how likely a user is to click each ad.
- The winner is chosen by **bid × pCTR**, so accurate predictions matter as much as big bids.
- Clicks are rare (about 1 in 100), so the model learns from few "yes" examples.
- Check that predictions are calibrated (2% really means about 2%), not just that clicks go up.
