# Fantasy-Premier-League: Explained Simply

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

*For a family member or a recruiter with no tech background.*

---

## First, what is Fantasy Premier League?

It's a free online game played by millions of football fans. You get a pretend budget of £100 million and pick 15 real Premier League players. Each weekend, your players earn points for what they actually do on the pitch: scoring, assisting, keeping a clean sheet. Each week you're allowed one free change to your team. Every extra change costs you 4 points.

## What I built, in one sentence

I built two computer players that play this game on their own for the whole season, so I can measure how much it costs to be stuck with your old decisions.

## The everyday comparison

Imagine two people doing the weekly food shop on the same budget.

- **Person A** has a fridge already full from last week. They can only swap one item for free. Swapping more costs extra. If they bought too much chicken last week, they're living with that chicken.
- **Person B** gets an empty fridge every week and can buy the perfect shop from scratch.

Person B will always eat a bit better. But *how much* better? That difference is the cost of being tied to last week's choices.

My project does exactly this with football teams:
- **"The Manager"** is Person A. It keeps its team all season and plays by the real rules.
- **"Best XI"** is Person B. It builds the perfect team from scratch every week.

Both use the same predictions, so the only difference is the freedom they have. The gap in their scores is the answer.

## How it works, step by step

1. **It collects information.** Several times a day it reads the official game's website data: players, prices, upcoming matches and scores. It asks politely, at most once a second, so it doesn't overload the website.
2. **It predicts points.** For every player it estimates how many points they're likely to score in the next five weeks. It looks at things like how often they start, how often they score, and how hard their next opponents are.
3. **It picks the best team.** A maths tool called an *optimiser* (a program that tries every legal combination in a smart way and finds the best one) picks 15 players within the budget and the game's rules.
4. **The Manager decides on changes.** It checks whether making 0, 1, 2 or more changes is worth the points they cost. Often the answer is "make no change and save the free one for next week".
5. **It locks the team before the deadline.** Once the real deadline passes, the team can't be changed, just like for a human player.
6. **It scores itself honestly.** After the matches, it uses the real official points, not its own guesses, and compares itself with the average player's score that week.
7. **It shows everything on a website**, with the team on a pitch, the reasons for each pick, and a season chart.

## Why it's harder than it sounds

- **No cheating with hindsight.** The season had already started when I built it, so the first few weeks had to be replayed. I made sure each replayed week only used information that existed *before* that week's deadline, like judging a weather forecast using only what was known the day before. The system physically refuses to change a team after its deadline.
- **The rules are fiddly.** Changes in the rules for this season (like goalkeepers now getting 10 points for a goal) had to be checked against the official data, not guessed.
- **It runs for free.** It doesn't use a paid server. It uses GitHub, a free service where programmers store their work, which can also run small jobs on a timer and host a simple website. The timer isn't always on time, so I built in extra margin so it never misses a deadline.

## How it's doing

After the first five weeks of the season:
- **Best XI** (fresh team every week): 322 points
- **The Manager** (real rules): 268 points

So far, being stuck with past choices has cost about 54 points. Five weeks is still too early to draw conclusions. Neither computer player is regularly beating the average human player yet, and I'm open about that.

## What this shows about me

- I can take a fuzzy question ("what does sticking with old decisions cost?") and turn it into something you can measure.
- I care about fair measurement: no hindsight and no marking my own homework.
- I can build something complete: collecting data, making predictions, making decisions, running on a schedule, and showing results on a website.
- I test my work. There are 610 automatic checks that all pass.
