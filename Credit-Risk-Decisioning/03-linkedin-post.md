# Credit-Risk-Decisioning: LinkedIn Post

*About 170 words. Copy from the line below.*

---

In credit, the cutoff is worth more than the model.

I built a credit decisioning system, not a default-prediction notebook: calibrated probabilities, approve/decline at the lowest expected cost, adverse-action reasons, a fairness audit and drift monitoring.

On an out-of-time test set, the same LightGBM model with different cutoffs:
- cost-optimal cutoff (0.107): profit of 2.5M
- F1-optimal cutoff: 730K less profit
- the "default" 0.5 cutoff: loses money outright

Choosing F1 cost more than twice the whole gap between the best and worst model.

Other things I measured instead of assuming:
- SMOTE and class weights destroyed calibration for no gain in ranking
- reject inference helped in one selection setting and hurt in another, and a real book can't tell you which you're in
- PSI stayed under its alarm while defaults rose 75%

I built it on a synthetic loan book with known ground truth, because fairness and reject inference can't be checked any other way. The numbers won't transfer to a real portfolio. The methods do.

https://github.com/MelvTheGoat/Credit-Risk-Decisioning

#CreditRisk #MachineLearning #Fintech #ResponsibleAI
