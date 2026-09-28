# Credit-Risk-Decisioning: Explained Simply

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A system that helps a lender decide whether to approve a loan: it estimates the chance someone won't pay back, weighs up the money at stake, explains every "no" in plain words, and checks that it's treating different groups of people fairly.

## The everyday comparison

Imagine lending money to friends.
- Lend to someone who pays you back, and you earn a little interest.
- Lend to someone who doesn't, and you lose most of what you lent.

Losing a loan hurts much more than earning a bit of interest helps. In my setup, one bad loan wipes out the profit from about eight good ones. So the question isn't "is this person more likely to pay than not?". It's "is the chance of them not paying low enough to be worth the risk?"

My system works that out for every application. It turns out that **where you draw the line** matters more than how clever the prediction is: using a common "textbook" line of 50% would actually **lose money**.

## What it does, step by step

1. **Estimates the risk** for each applicant from things like income, existing debts, how much of their credit they use, and missed payments.
2. **Makes sure the risk number is honest.** If it says "10% chance", roughly 10% of such people should really not pay.
3. **Decides yes or no** using the money involved, not a rule of thumb.
4. **Explains any "no"** in plain language, like "your debt repayments are high compared with your income", as lenders are required to.
5. **Checks fairness.** Even though it never looks at someone's group directly, it checks whether one group is turned down more than they should be, and finds out why.
6. **Watches for changes over time,** like more people failing to repay than expected.
7. **Keeps a permanent record** of every decision and why.

## An interesting finding

The system found that one group was being scored as riskier than they really were. The cause wasn't the model. It was that their **income was being recorded too low** in the data. The right fix is to record income correctly, not to lower the bar for one group. That's the kind of insight a lender actually needs.

## Honest limits

I built and tested it on **realistic made-up data**, because with real loans you never learn what would have happened to the people who were turned down. That means the exact numbers wouldn't carry over to a real bank, but the methods would.

## What this shows about me

- I understand lending as a business, not just as a prediction task.
- I care about fairness and explaining decisions.
- I measure things carefully and report surprising results honestly.
- I can build a complete system: modelling, decisions, a web service, an audit trail and deployment.
