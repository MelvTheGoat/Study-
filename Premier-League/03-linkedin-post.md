# Premier-League: LinkedIn Post

*About 165 words. Copy from the line below.*

---

A league table tells you what happened. It doesn't tell you a team played four days after a European away trip, with a new manager.

I built a Premier League predictor that turns that context into numbers instead of rules. Congestion, table pressure, manager tenure, squad ratings, head-to-head: 200+ features per match, and the model decides what matters.

The interesting bit is promoted clubs. They arrive with no Premier League history. So Elo is rated across the Premier League, Championship, League One and cups on one scale, and a promoted side's season of results carries over.

It retrains after every gameweek, never overwrites a forecast, and publishes itself daily.

In the repo's walk-forward backtest over 1,050 matches: 52.1% outcome accuracy vs 43.2% for always picking the home side, and well-calibrated probabilities. The live season is only 5 gameweeks in, so it's far too early to judge.

One lesson: CI stayed green 11 times while the site was stuck two gameweeks back. Now the job checks the live page.

https://github.com/MelvTheGoat/Premier-League

#MachineLearning #Football #Python #DataScience
