# Search Ranking System

A search ranking system takes what you type into a search box and decides which results to show, and in what order. When you search "running shoes" on Amazon, it has millions of products but shows you the best few at the top. It matters because people rarely scroll: if the right result isn't near the top, it might as well not exist.

## Key Terms

- **Query**: the words the user types into the search box.
- **Document**: one thing that can be found, like a product, a web page or a video.
- **Index**: a prepared lookup table that lets us find matching documents quickly.
- **Retrieval**: quickly finding a few hundred documents that match the query.
- **Ranking**: carefully ordering those documents so the most useful are on top.
- **Relevance**: how well a result matches what the user actually wanted.
- **Latency**: how long the user waits for results to appear.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide what a "good result" means here. On a shop, it's a product the user clicks and buys. On a help site, it's a page that answers the question.

**Step 2: Figure out the data.** You need the documents themselves (titles, descriptions, prices) and a history of searches. That history tells you which results people clicked for each query, which is gold for learning.

**Step 3: Sketch the main parts.** Like most large systems, search uses two stages: a fast stage that finds matches, and a careful stage that orders them. Draw these boxes first.

**Step 4: Follow one search.** Trace a single query from the search box to the results page. Give each part a slice of the time budget so you know what "too slow" means.

**Step 5: Decide how to know it works.** Choose simple measures, like "how often people click one of the top 3 results". Also plan for people rating results by hand.

**Step 6: Plan for problems.** Think about typos, searches with no results, and very popular items that crowd out everything else.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design search for an online shop.

- **10 million** products.
- **1,000** searches per second at busy times.
- Show the top **20** results in under **300 milliseconds** (0.3 seconds).

A time budget for those 300 ms:

1. 50 ms to understand the query.
2. 100 ms to find matching products.
3. 100 ms to rank them.
4. 50 ms spare for building the page.

50 + 100 + 100 + 50 = 300 ms.

### The Data We Use (Step 2)

- **Product data**: title, description, category, price, rating.
- **Search logs**: what people searched for, what they saw, and what they clicked or bought.
- **Popularity**: how often each product is bought.

The search logs matter most. If thousands of people search "phone case" and buy the same three items, that is a strong clue about what's relevant.

### The Big Picture (Step 3)

```
   [User types "running shoes"]
              |
              v
   [Query Understanding]   -- fix typos, spot "shoes" = category
              |
              v
   [Retrieval] <---------- [Search Index]
              |               (10 million products)
              v
   [Ranking Model]         -- scores ~500 matches
              |
              v
   [Results Page: top 20]
```

Step by step:

1. The user types "runing shoes" (with a typo) and presses search.
2. Query Understanding fixes the typo to "running shoes" and notices "shoes" is a category.
3. Retrieval looks up the Search Index and quickly finds about 500 matching products.
4. The Ranking Model (a program that learned patterns from past searches) scores those 500 and puts them in order.
5. The top 20 appear on the results page, all within 0.3 seconds.

### The Main Parts (Step 3)

**Query Understanding.** This step cleans up what the user typed: fixing spelling, and spotting useful hints like a brand or a size. It's like a friendly shop assistant who hears "runing shoos" and knows exactly what you mean.

**Search Index.** The index is prepared ahead of time, like the index at the back of a textbook. Instead of reading every page to find "photosynthesis", you look it up and jump straight to pages 42 and 97. Our index maps each word to the products that contain it, so finding matches takes milliseconds.

**Retrieval.** Using the index, this step collects every product that reasonably matches, usually a few hundred. It's like a librarian quickly pulling 500 books off the shelf before you choose the best 20. Speed matters more than perfect order here.

**Ranking Model.** This model scores each of the 500 products. It looks at clues such as:

- How well the words match the query.
- How often people who searched this clicked or bought the product.
- The product's rating and price.

It's like a careful judge comparing 500 finalists. The model learns from search logs which clues matter most.

A common approach here is called "learning to rank" (advanced - skip for now).

**Results Page.** The top 20 results are shown, often with small rules on top, like "don't show out-of-stock items first".

### How We Know It's Working (Step 5)

- **Click-through rate (CTR)**: the share of searches where the user clicks a result. If 1,000 searches lead to 600 clicks, CTR is 600 ÷ 1,000 = 60%.
- **Top-3 click rate**: how often the click lands in the first 3 results. High means the best results really are at the top.
- **Zero-result rate**: how often a search finds nothing at all. Lower is better.
- **Human ratings**: people rate a sample of results as "great", "okay" or "wrong". This catches problems that clicks miss.

To test a change, run an A/B test: half the users get the new ranking, half get the old one, and you compare the numbers. It's like a fair taste test.

### What Can Go Wrong (Step 6)

- **Typos and odd words.** "iphon case" should still work. Spelling correction and synonyms ("sofa" = "couch") help a lot.
- **No results.** Rather than an empty page, loosen the search (drop one word) or show popular items in that category.
- **Position bias.** People click the first result partly just because it is first. If you learn only from clicks, the same items stay on top forever. Showing a little variety helps the system learn fairly.
- **Slow searches at busy times.** Keep popular searches ready in Redis (a very fast storage for data you need instantly), so repeat searches return instantly.

## Quick Recap

- Search has two stages: fast retrieval with an index, then careful ranking with a model.
- An index is like the back of a textbook: look up a word, jump straight to matches.
- Search logs (what people clicked and bought) teach the ranking model what "relevant" means.
- Watch click rates, zero-result searches and human ratings to know it's working.
