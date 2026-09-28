# Fantasy-Premier-League: LinkedIn Post

*About 160 words. Copy from the line below.*

---

Every Fantasy Premier League manager pays for last week's decisions. Nobody can see the bill.

So I built two bots that play the 2026/27 season from the same predictions.

The Manager lives by the real rules: one free transfer a week, −4 for each extra, £100m, chips that expire.
Best XI rebuilds the perfect squad from scratch every gameweek.

The gap between them is the cost of continuity.

The interesting bit: the squad picker is an integer linear program, not a greedy pick. Budget, 2/5/5/3 positions and "max 3 per club" all interact, and the ILP finds the true best legal squad.

The hardest rule was no hindsight. A past gameweek can only use data from before its own deadline, and the database refuses to rewrite a locked squad.

After 5 gameweeks: Best XI 322, Manager 268. Far too early to judge, and neither is beating the average often yet.

Runs free on GitHub Actions. 610 tests.

https://github.com/MelvTheGoat/Fantasy-Premier-League

#Python #Optimisation #FPL #DataScience
