# System Designs for Beginners

Welcome! This folder is a gentle introduction to system design (planning how the parts of a large piece of software fit together). You don't need any background to start. Every new word is explained the first time it appears, and every design uses everyday comparisons.

Go at your own pace. It's completely normal to read a design twice before it clicks. Each one follows the same shape, so it gets easier as you go:

1. **Key Terms**: the new words, in plain English.
2. **Part 1: How to Approach It**: the steps to think through any design.
3. **Part 2: The Design**: a worked example with one simple diagram.
4. **Quick Recap**: the few ideas worth remembering.

## Suggested Learning Order (easiest first)

1. [Recommendation System](Machine%20Learning/01-recommendation-system.md): suggesting videos people will enjoy.
2. [Search Ranking System](Machine%20Learning/02-search-ranking-system.md): showing the best results for what someone typed.
3. [Fraud and Anomaly Detection](Machine%20Learning/04-fraud-anomaly-detection.md): spotting suspicious payments quickly.
4. [Ad Click Prediction](Machine%20Learning/03-ad-ctr-prediction.md): guessing which ad someone will click, and picking the winner.
5. [Conversational Assistant](Artificial%20Intelligence/04-conversational-assistant.md): a chatbot that remembers the conversation.
6. [RAG System: Building the App](Artificial%20Intelligence/01-rag-system.md): a chatbot that answers from your own documents, with sources.
7. [AI Agent with Tool Use](Artificial%20Intelligence/02-ai-agent-with-tool-use.md): an AI that can take actions, like booking a meeting.
8. [LLM Evaluation and Observability](Artificial%20Intelligence/05-llm-evaluation-observability.md): checking an AI app is good, before and after release.
9. [LLM Serving and Inference Platform](Artificial%20Intelligence/03-llm-serving-inference-platform.md): running a big AI model for thousands of users at once.
10. [LLM RAG System: The Models Behind the Search](Machine%20Learning/05-llm-rag-system.md): how the search models in RAG work and get better. Read #6 first.

## All Designs by Folder

### Machine Learning

| Design | One-line summary |
|---|---|
| [01 Recommendation System](Machine%20Learning/01-recommendation-system.md) | Suggest items people will like, using a fast shortlist and a careful ranking. |
| [02 Search Ranking System](Machine%20Learning/02-search-ranking-system.md) | Turn a search into the best 20 results, using an index and a ranking model. |
| [03 Ad Click Prediction](Machine%20Learning/03-ad-ctr-prediction.md) | Predict the chance of a click, then pick the ad with the best bid × chance. |
| [04 Fraud and Anomaly Detection](Machine%20Learning/04-fraud-anomaly-detection.md) | Score every payment and approve, review or block it in a split second. |
| [05 LLM RAG System (models)](Machine%20Learning/05-llm-rag-system.md) | How embedding and re-ranking models find the right passages, and how to improve them. |

### Artificial Intelligence

| Design | One-line summary |
|---|---|
| [01 RAG System (the app)](Artificial%20Intelligence/01-rag-system.md) | Split documents, store them, find the right pieces, and answer with sources. |
| [02 AI Agent with Tool Use](Artificial%20Intelligence/02-ai-agent-with-tool-use.md) | An AI that loops through think, act and check to finish real tasks safely. |
| [03 LLM Serving Platform](Artificial%20Intelligence/03-llm-serving-inference-platform.md) | Run a large model on GPUs for many users: batching, streaming and scaling. |
| [04 Conversational Assistant](Artificial%20Intelligence/04-conversational-assistant.md) | A support chatbot with memory, safety checks and handoff to humans. |
| [05 LLM Evaluation and Observability](Artificial%20Intelligence/05-llm-evaluation-observability.md) | Test AI answers before release and watch them after, with a feedback loop. |

## A Tip Before You Start

Try sketching the diagram yourself on paper after reading each design, without looking. If you can draw the boxes and explain each one in a sentence, you've understood the core idea. That's all system design really is: boxes, arrows, and good reasons.
