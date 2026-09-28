# Premier-League: Explained Simply

Repo: https://github.com/MelvTheGoat/Premier-League

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A program that predicts the result of every Premier League football match each week, gives the chance of a home win, draw or away win, learns from every new week of results, and publishes its predictions on a website that updates itself.

## The everyday comparison

Think of a good football pundit. They don't just look at the league table. They think:
- "This team played in Europe on Thursday, so they're tired."
- "They've just changed manager."
- "They're safe mid-table in May, so they have nothing to play for."

Most pundits carry these rules in their heads. My program is different. Instead of me telling it "a new manager is worth +5%", I give it the **facts** (how long the manager has been there, how the team did before and after) and let it **work out from 15 years of past matches** how much each fact actually matters.

## How it works, step by step

1. **It collects results** from a free public football database, including lower leagues and cups.
2. **It turns each match's situation into numbers:** recent form, league position gaps, days of rest, manager changes, squad quality from the FIFA video game ratings, past meetings, and more. Over 200 numbers per match.
3. **It learns from history**, but only from matches played *before* the week it's predicting. It never peeks at the future.
4. **It predicts each match** as three percentages (home win, draw, away win) plus a likely score.
5. **It saves every prediction forever.** It never changes a past prediction, so anyone can check its record.
6. **Every morning it checks** for new results, updates itself if needed, and even checks that the website is really showing the latest week.

## A clever bit

Newly promoted teams have no Premier League history, so most models treat them as a mystery. Mine rates teams across several divisions on one shared scale, so a promoted team's strong season in the division below still counts.

## How good is it?

When tested on three past seasons, one week at a time as if live, it picked the right result about **52%** of the time. Always guessing "home win" gets about 43%. Football is very unpredictable, so that's a solid result. Its percentages are also honest: when it says 65%, that outcome happens about 65% of the time.

This season it's only five weeks in, so it's too early to judge.

## What this shows about me

- I can turn "gut feeling" knowledge into measurable data.
- I'm strict about not cheating with hindsight.
- I build systems that run themselves and check that they're really working.
- I report results honestly, including the limits.
