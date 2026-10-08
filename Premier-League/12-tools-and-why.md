# Premier League Predictor: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like Elo or leakage, are explained in [11-technical-terms.md](11-technical-terms.md).

A helpful way to picture it: the project is a small factory. Some tools are the machines that do the work, some are the storage room, and some are the delivery van that gets the results to the public.

---

## The Language

### Python
**What it is:** a popular programming language that's known for being easy to read.

**Why it's used here:** the whole project is written in it. Python has the best free tools for working with data and building models, so it's the usual choice for this kind of work.

---

## Working with Data

### pandas
**What it is:** a Python tool for working with tables of data, like a spreadsheet you control with code.

**Why it's used here:** matches, features and predictions are all tables. pandas is used to sort matches by date, pick out columns, and pass tables to the models.

### NumPy
**What it is:** a Python tool for fast maths on long lists of numbers.

**Why it's used here:** the models' outputs are lists of chances. NumPy blends the two models' chances (60% + 40%) and makes sure each match's three chances add up to 100%.

### git (to fetch the data)
**What it is:** a tool that keeps track of versions of files. It can also download a whole collection of files and later fetch only what changed.

**Why it's used here:** the main football data (openfootball) is shared as a git collection. The project downloads it once, then just fetches the new results each day, which is quick.

---

## Building the Models

### LightGBM
**What it is:** a fast tool for gradient boosting, which means building many small yes/no flowcharts, each one fixing the mistakes of the ones before.

**Why it's used here:** it provides 60% of the result prediction. It copes well with lots of different facts and with gaps in the data, which matters because some facts arrive late.

### scikit-learn
**What it is:** Python's most common toolbox for everyday machine learning (programs that learn patterns from data).

**Why it's used here:** it provides the simpler model (logistic regression) that makes up the other 40%. It also fills in missing values with the middle value and puts all the facts on the same scale before that model learns.

### SciPy
**What it is:** a Python tool for scientific maths, like finding the best settings for a formula.

**Why it's used here:** the score model (Dixon-Coles) needs an attack and defence number for every club. SciPy searches for the numbers that best explain past scores, and also gives the chance of each goal count (0, 1, 2 and so on).

A small note: the project's install list also includes statsmodels and requests, but I didn't find either being used in the code.

---

## Storing Data

### SQLite
**What it is:** a database (an organised store of data in tables) that lives in a single file. There's no separate server to run.

**Why it's used here:** it's simple and free. The big working database (about 61 MB) is rebuilt on each run. A slim copy for the website (about 1.4 MB) is saved inside the project itself, so the website needs no separate database service.

---

## The Website

### Flask
**What it is:** a small Python tool for building websites. You write a little code for each page, and it sends the page to the visitor.

**Why it's used here:** the site only has a few pages (gameweek, season history, and how the model works). Flask is the *only* thing the website installs. This keeps it tiny, because the heavy model tools never run on the website.

### Vercel
**What it is:** a hosting service (a company that puts your website online). It runs your code only when someone visits, which is called "serverless".

**Why it's used here:** it's free, and it updates the site automatically whenever new code is pushed. The trade-off is a size limit, which is why the website is kept to Flask and a small read-only database.

---

## Running Itself

### GitHub Actions
**What it is:** a free service from GitHub (the website where the code is stored) that runs tasks for you, on a timetable or on request.

**Why it's used here:** it's the project's daily robot. Every morning at 6 a.m. UTC it:

1. Checks for new results, and stops if there are none.
2. Retrains the models and predicts the next gameweek.
3. Makes the website copy of the database and checks the pages load.
4. Saves the update and moves the website's branch forward safely.
5. Checks the live website really shows the new gameweek.

It needs no passwords or secret keys, because it uses the access GitHub already gives it.

---

## Testing

### pytest
**What it is:** a Python tool for running tests, which are small programs that check your code still does what it should.

**Why it's used here:** the project has 98 tests. They cover the features, the models, reading openfootball's files, the pipeline and website, the publishing checks, and club names. If a change breaks something, a test fails before it reaches the public.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python | The language everything is written in |
| pandas | Handles the tables of matches and features |
| NumPy | Blends the two models' chances |
| git | Downloads the football data, then fetches updates |
| LightGBM | The main result model (60%) |
| scikit-learn | The simpler, steadier result model (40%) |
| SciPy | Fits the score model |
| SQLite | Stores everything in one file |
| Flask | Builds the website pages |
| Vercel | Puts the website online for free |
| GitHub Actions | Runs the whole thing every morning |
| pytest | Checks the code still works |
