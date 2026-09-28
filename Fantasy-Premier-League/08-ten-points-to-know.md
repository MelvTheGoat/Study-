# Fantasy-Premier-League: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

---

1. **Two bots, one projection: the Manager (real rules) vs Best XI (fresh squad every week).**
   *Why it matters:* they believe the same things, so the score gap measures only the cost of continuity.

2. **Both are scored with real FPL points against the official gameweek average (`average_entry_score`).**
   *Why it matters:* the system never marks its own homework with its own predictions.

3. **The rules engine is pure logic with no database or network (`fplai/rules/`).**
   *Why it matters:* a wrong rule silently breaks every number, so the rules must be easy to test. They have the most tests.

4. **Expected points are a sum of parts: appearance, goals, assists, clean sheet, goals conceded, saves, defensive contribution, bonus, cards.**
   *Why it matters:* each pick can be explained in football terms, and there's too little early-season data to train a model.

5. **Rates are shrunk toward a price-based prior. 450 minutes of play equals the prior's weight. Last season counts at 0.45.**
   *Why it matters:* it stops one lucky goal in two games making a cheap midfielder look like the best player.

6. **The squad picker is an integer linear program (PuLP + CBC): 15 in squad, 11 starting, 1 captain, 2/5/5/3, max 3 per club, within budget.**
   *Why it matters:* the constraints interact, so greedy picking can't find the true best legal squad.

7. **The Manager tries 0 to 5 transfers. A hit must beat 4 points plus a 2-point margin. Chips have their own bars: BB 22, TC 12, FH 18, WC 30.**
   *Why it matters:* it copies how a careful human thinks, and it fixed a bug where one shared bar burned chips in week 2–3.

8. **No hindsight is enforced by the database: per-gameweek prices, projections keyed by deadline, and `LockedPicksExist` after a deadline.**
   *Why it matters:* the backfill is honest by design, not by discipline. The one leak is historical injury flags.

9. **A stateless scheduler on GitHub Actions: 8-hour pick window, runs stay alive until the deadline, and the DB is stored on a force-pushed `season-data` branch.**
   *Why it matters:* it runs for free and survives GitHub's late cron (2–7 hour gaps were seen).

10. **Results after GW5 (28 Sep 2026): Best XI 322, Manager 268. Beat the average: 2/5 vs 1/5. 610 tests pass. No backtest yet.**
    *Why it matters:* know the real numbers, and know what isn't measured (prediction accuracy), so you never overclaim.
