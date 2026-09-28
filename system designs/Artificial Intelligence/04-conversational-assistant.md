# Conversational Assistant

A conversational assistant is a chatbot that can hold a back-and-forth conversation and remember what was said earlier. A customer-support chat on a bank's website, or ChatGPT itself, are good examples. It matters because a good assistant can answer common questions instantly, day or night, and hand tricky cases to a human.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Message**: one thing said in the chat, by the user or the assistant.
- **Conversation history**: all the messages so far in one chat.
- **System prompt**: hidden instructions that tell the LLM how to behave, like "You are a polite bank assistant".
- **Token**: a small piece of text, roughly three quarters of a word.
- **Context window**: the maximum amount of text (in tokens) the LLM can read at once. It's the LLM's short-term memory.
- **Handoff**: passing the chat to a human agent when the bot can't help.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide what the assistant is for, like answering questions about bank accounts. Also decide what it must never do, like giving financial advice. A clear, narrow job makes a better assistant.

**Step 2: Figure out the data.** You need the conversation history for each chat, some facts about the user (if they're logged in), and the company's help articles. Decide what to store and for how long.

**Step 3: Sketch the main parts.** Every new message needs the history loaded, a prompt built, safety checks, the LLM's reply, and the history saved again. Draw these as boxes.

**Step 4: Walk through one message.** Follow a single user message through the system. Notice what the LLM "sees" each time, because it only knows what's in the prompt.

**Step 5: Decide how to know it works.** Measure whether people's problems actually get solved, and how happy they are.

**Step 6: Plan for problems.** Long chats overflow the LLM's memory, users ask off-topic or harmful things, and sometimes the bot is simply wrong. Plan a response to each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design a support assistant for a bank.

- **100,000** conversations per day.
- About **10** messages per conversation.
- A reply within about **3 seconds**.
- Hand off to a human when the user asks, or when the bot is stuck.

How many LLM replies per day? Step by step:

1. 100,000 conversations × 10 messages = 1,000,000 messages.
2. About half are from the assistant: **500,000 replies per day**.

### The Big Picture (Step 3)

```
   [User sends message]
            |
            v
   [Chat Service] <-----> [Conversation Store]
            |                (history of each chat)
            v
   [Build Prompt: system prompt + history + new message]
            |
            v
   [Safety Check (in)]
            |
            v
   [LLM] ---> [Safety Check (out)] ---> [Reply shown + saved]
```

Step by step:

1. The user types, "Why was I charged a fee?"
2. The Chat Service (the program that manages each conversation) loads this chat's history from the Conversation Store.
3. It builds a prompt: the system prompt, the history, and the new message.
4. A Safety Check looks at the incoming message for harmful or off-topic requests.
5. The LLM writes a reply.
6. A second Safety Check reviews the reply, for example to make sure it doesn't reveal private data.
7. The reply is shown to the user and saved to the history.

### The Main Parts (Step 3)

**Conversation Store.** A database that keeps every chat's messages. Each chat has an ID, so we can load the right history. It's like a receptionist's notebook: before answering, they glance at what the caller said earlier.

**The LLM Doesn't Remember.** This surprises many beginners. The LLM forgets everything between messages. We make it *seem* to remember by sending the whole history every time. It's like replaying the whole phone call to someone before asking your next question.

**System Prompt.** These hidden instructions set the assistant's personality and rules. For example: "You are a friendly bank assistant. Only discuss our products. If unsure, offer a human." It's like the training a new employee gets on their first day.

**Handling Long Chats.** The context window has a size limit. Let's do the math:

1. Say the context window holds **8,000 tokens**.
2. Each message is about **100 tokens**.
3. 8,000 ÷ 100 = about **80 messages** before it's full (less, once the system prompt is included).

For long chats, we keep the recent messages in full and replace older ones with a short summary. It's like keeping meeting notes instead of the full recording.

**Safety Checks.** These are filters before and after the LLM. They block harmful requests, stop the bot from sharing private data, and keep it on topic. Think of them as a polite security guard at each door.

**Handoff to a Human.** If the user asks for a person, or the bot is unsure twice in a row, the chat moves to a human agent, along with the full history. The customer never has to repeat themselves.

The assistant can also look up help articles before answering. That's the RAG pattern, which has its own guide: [01-rag-system.md](01-rag-system.md).

### How We Know It's Working (Step 5)

- **Resolution rate**: the share of chats solved without a human. If 70,000 of 100,000 chats end solved, that's 70%.
- **Handoff rate**: the share passed to humans. Some handoff is healthy, but a sudden jump means something is wrong.
- **User rating**: a simple thumbs-up or thumbs-down after each chat.
- **Response time**: are replies arriving within about 3 seconds?

Also read a small sample of real chats every week. Numbers can hide problems that are obvious when you read the words.

### What Can Go Wrong (Step 6)

- **It "forgets" earlier details.** Long chats get summarized badly. Keep important facts (like the account type) in a short "facts so far" note that is always included.
- **Confident wrong answers.** The LLM makes up a fee policy. Give it real help articles to answer from, and let it say "I'm not sure, let me get a colleague."
- **Harmful or off-topic requests.** Users try to make it say silly or dangerous things. The safety checks and a clear system prompt handle most of this.
- **Frustrated users.** Repeating "I don't understand" makes people angry. Offer a human early, especially after two failed attempts.

## Quick Recap

- The LLM has no memory. We send the conversation history with every message.
- The system prompt sets the assistant's rules and personality.
- Context windows have limits, so summarize old messages in long chats.
- Safety checks go before *and* after the LLM.
- Measure whether problems get solved, and hand off to humans gracefully.
