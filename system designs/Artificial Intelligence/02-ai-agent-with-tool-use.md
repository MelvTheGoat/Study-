# AI Agent with Tool Use

An AI agent is a chatbot that can *do* things, not just talk. If you ask a smart assistant "Book a meeting with Sam next Tuesday at 2 p.m.", an agent can check your calendar, create the event and send the invite. It matters because it turns AI from "a clever writer" into "a helpful assistant that gets tasks done".

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Agent**: an LLM that can decide on actions and take them, step by step, to finish a task.
- **Tool**: a specific action the agent can use, like "check calendar" or "send email".
- **Tool call**: the agent asking to use a tool, with details, like "check calendar for Tuesday".
- **Agent loop**: the repeating cycle of think, act, look at the result, and think again.
- **Guardrails**: safety rules that limit what the agent is allowed to do.
- **Step limit**: the maximum number of actions the agent can take on one task.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide which tasks the agent should handle, and which it must never do. A narrow, clear job ("manage my calendar") is much easier to build well than "do anything".

**Step 2: Figure out the tools.** List the actions the agent needs. For each tool, write down its name, what it does, and what information it needs. The agent can only be as useful as its tools.

**Step 3: Sketch the loop.** Draw the agent loop: the LLM decides, a tool runs, the result comes back, and the LLM decides again. This loop is the heart of every agent.

**Step 4: Walk through one task.** Follow a real request from start to finish, counting the steps, because each one takes time and money.

**Step 5: Decide how to know it works.** Measure how often tasks are completed correctly, how long they take, and what they cost.

**Step 6: Plan for mistakes.** Agents can get confused, loop forever, or try something risky. Add limits and safety checks before giving the agent real power.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design a calendar assistant for a company.

- **10,000** tasks per day.
- Most tasks finish in **3-5 steps**.
- Finish most tasks in under **10 seconds**.
- It must **never** delete meetings without asking the user first.

### The Tools (Step 2)

Our agent gets four tools:

- **find_person**: looks up a colleague's email from their name.
- **check_calendar**: shows free and busy times for a date.
- **create_event**: books a meeting and sends invites.
- **delete_event**: removes a meeting (needs the user's "yes" first).

Each tool has a short description written for the LLM, like a label on a kitchen drawer. The clearer the label, the more likely the agent grabs the right tool.

### The Big Picture (Step 3)

```
   [User: "Book a meeting with Sam on Tuesday at 2pm"]
                     |
                     v
   [Agent (LLM)] <---------------------+
         |                             |
         | decides: use a tool         | result goes back
         v                             |
   [Guardrails check] --> [Tool Runner] --> (Calendar, Email)
         |
         | when finished
         v
   [Final reply to user]
```

Step by step, for this task:

1. The user asks to book a meeting with Sam.
2. The Agent decides it first needs Sam's email, and asks for **find_person("Sam")**.
3. The Guardrails check allows it, the Tool Runner (the part that actually runs tools) runs it, and the email comes back to the Agent.
4. The Agent asks for **check_calendar("Tuesday")** and sees 2 p.m. is free.
5. The Agent asks for **create_event(Sam, Tuesday, 2 p.m.)**. The meeting is booked.
6. The Agent replies: "Done! Meeting with Sam booked for Tuesday at 2 p.m."

That's 3 tool calls plus a final reply, which is 4 trips to the LLM.

### The Main Parts (Step 3)

**The Agent (LLM).** At each step, it reads the task, the tool descriptions and all results so far, then picks the next action. It's like a person following a recipe: read the next line, do it, check the result, move on.

**The Agent Loop.** The loop repeats *think → act → look at the result* until the task is done. It's like finding your way in a new city: check the map, walk a block, look around, check the map again.

**Tool Runner.** The LLM only *writes* a request like "create_event(Sam, Tuesday, 2 p.m.)". Normal code then runs the real action and sends back the result. It's like a restaurant: the waiter (the LLM) writes your order, but the kitchen (the Tool Runner) actually cooks it.

**Guardrails.** Before any tool runs, simple rules check it. Risky actions, like deleting a meeting, need the user to confirm. It's like a bank asking "Are you sure?" before a big transfer.

**Memory.** The agent keeps a record of the steps and results for the current task, so it doesn't repeat itself. Long-term memory across many days is a bigger topic (advanced - skip for now).

### Time and Cost (Step 4)

Let's estimate with simple numbers:

1. Each LLM call takes about **1.5 seconds** and costs about **1 cent**.
2. Our meeting task used **4** LLM calls.
3. Time: 4 × 1.5 = **6 seconds** (plus a little for the tools). That's under 10 seconds.
4. Cost: 4 × 1 cent = **4 cents** per task.
5. Per day: 10,000 tasks × 4 cents = **$400 per day**.

Fewer steps means faster *and* cheaper.

### How We Know It's Working (Step 5)

- **Task success rate**: the share of tasks done correctly. If we test 100 tasks and 90 are correct, that's 90%.
- **Steps per task**: fewer is better. A sudden rise often means the agent is getting confused.
- **Time and cost per task**: are we staying under 10 seconds and around 4 cents?
- **Safety issues**: how often guardrails had to block something. This should be rare, and each case should be reviewed.

Keep a set of test tasks and re-run them after every change.

### What Can Go Wrong (Step 6)

- **Looping forever.** The agent keeps checking the calendar again and again. Set a step limit, like 10, then stop and ask the user for help.
- **Wrong tool or wrong details.** It books "Sam Smith" instead of "Sam Jones". Check tool inputs, and ask the user when a name matches more than one person.
- **Risky actions.** Deleting or sending things can't always be undone. Require the user's confirmation for these.
- **A tool fails.** The calendar service is down. Retry once, then tell the user clearly instead of pretending it worked.

Someone could also hide instructions inside an email to trick the agent, called "prompt injection" (advanced - skip for now).

## Quick Recap

- An agent is an LLM that repeats a loop: think, use a tool, look at the result, repeat.
- The LLM only *asks* for actions. Normal code runs them, with guardrails checking first.
- Clear, well-described tools mean fewer steps, lower cost and fewer mistakes.
- Always set a step limit and require confirmation for risky actions.
- Measure task success, steps, time and cost.
