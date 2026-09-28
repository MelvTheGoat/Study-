# LLM Serving and Inference Platform

An LLM serving platform is the machinery that runs a large AI model (a program that learned patterns from huge amounts of text) so thousands of people can use it at once. When you type into ChatGPT and words start appearing, a platform like this is working behind the scenes. It matters because a brilliant model is useless if it's slow, keeps crashing or costs too much to run.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Inference**: using an already-trained model to answer a request (as opposed to training it).
- **GPU**: a special computer chip that does lots of maths at the same time. LLMs need GPUs to run fast.
- **Token**: a small piece of text, roughly three quarters of a word. "Hello there!" is about 3 tokens.
- **Latency**: how long the user waits for a response.
- **Time to first token**: how long until the first word appears on screen.
- **Throughput**: how much work the system finishes per second, for example tokens per second.
- **Batching**: handling many users' requests together on one GPU, instead of one at a time.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide how many users you expect, how long their answers are, and how fast they must feel. A chatbot needs the first words quickly, while a nightly report can wait.

**Step 2: Figure out the workload.** Estimate requests per minute and tokens per answer. From that you can work out how many tokens per second you must produce. This single number drives how many GPUs you need.

**Step 3: Sketch the main parts.** Requests need a front door, a way to share the work, and GPU servers running the model. Draw these as boxes.

**Step 4: Follow one request.** Trace a message from the user to the GPU and back. Notice that the answer is sent back word by word, not all at the end.

**Step 5: Decide how to know it works.** Watch speed, errors, how busy the GPUs are, and cost.

**Step 6: Plan for busy times and failures.** Traffic jumps at certain hours, and GPUs can crash. Plan how to add capacity and how to fail gracefully.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design serving for a company chatbot.

- **1,000** requests per minute at the busiest time.
- About **500 tokens** per answer.
- First words on screen within **1 second**.

### How Many GPUs? (Step 2)

Let's do the math step by step:

1. 1,000 requests per minute × 500 tokens = 500,000 tokens per minute.
2. A minute has 60 seconds: 500,000 ÷ 60 ≈ **8,300 tokens per second** needed.
3. Say one GPU server, using batching, produces about **2,000 tokens per second**.
4. 8,300 ÷ 2,000 ≈ 4.2, so we need **5 servers**.
5. Add spare capacity for surprises and crashes: plan for **7 servers**.

These are example numbers. Always measure your own model on your own GPUs.

### The Big Picture (Step 3)

```
   [Users' apps]
         |
         v
   [API Gateway]        -- checks keys, applies rate limits
         |
         v
   [Load Balancer]      -- spreads requests across servers
         |
    +----+----+
    v         v
 [GPU Server] [GPU Server]  ... (7 in total)
         |
         v
   [Streamed answer back to user, token by token]
         |
   [Monitoring] watches speed, errors and GPU use
```

Step by step:

1. A user sends a message from the app.
2. The API Gateway checks the user's key (a secret code that proves who they are) and makes sure they aren't sending too many requests.
3. The Load Balancer sends the request to whichever GPU Server is least busy.
4. The GPU Server adds it to a batch with other users' requests and starts generating tokens.
5. Each token is streamed (sent immediately) back to the user, so words appear one by one.
6. Monitoring records how fast it was and whether anything failed.

### The Main Parts (Step 3)

**API Gateway.** An API is the way apps talk to our service, and the gateway is its front door. It checks who you are and applies rate limits (a cap on how many requests one user can send per minute). It's like a nightclub bouncer: checks your ID and stops one group from rushing in all at once.

**Load Balancer.** It spreads requests across the GPU servers so no single server gets overloaded. It's like a supermarket manager sending the next shopper to the shortest checkout queue.

**GPU Servers.** Each server loads the model into GPU memory and generates tokens. Loading a big model can take minutes, so servers are kept running, not started fresh for each request. Think of an oven that's already hot and ready.

**Batching.** A GPU can work on many requests at once almost as fast as on one. So instead of serving users one by one, the server groups them. It's like a bus instead of taxis: one trip carries many people, and the cost per person drops a lot.

Modern servers keep adding and removing requests from a batch while it runs (advanced - skip for now).

**Streaming.** Answers are sent token by token as they're made. The full answer might take 5 seconds, but the user sees the first words in under 1 second. It feels like a live conversation instead of a long wait.

**Autoscaling.** When traffic grows, extra GPU servers are started automatically. When it drops, some are switched off to save money. It's like a shop opening more tills at lunchtime.

### How We Know It's Working (Step 5)

- **Time to first token**: is it under 1 second for most users? This is what "fast" feels like in chat.
- **Tokens per second per user**: how quickly the rest of the answer arrives. About 20-30 feels smooth to read.
- **Error rate**: the share of requests that fail. If 5 out of 1,000 fail, that's 0.5%.
- **GPU use and cost**: how busy the GPUs are, and the cost per 1,000 tokens. Idle GPUs are expensive.

### What Can Go Wrong (Step 6)

- **Sudden traffic spike.** Requests pile up. Use rate limits, a short waiting queue, and autoscaling to add servers.
- **A GPU server crashes.** The load balancer runs regular health checks (quick "are you OK?" pings) and stops sending work to broken servers.
- **Very long prompts.** A huge document uses lots of GPU memory and slows everyone. Set a maximum prompt length.
- **Too expensive.** Use batching, switch off idle servers, and consider a smaller model for simple questions.

Models can also be shrunk so they need less GPU memory, called quantisation (advanced - skip for now).

## Quick Recap

- Serving means running a trained model for many users, fast and reliably.
- Work out tokens per second first. That number tells you how many GPUs you need.
- Batching is the biggest cost saver: one GPU serves many users at once.
- Streaming makes answers feel fast, even when the full reply takes a few seconds.
- Plan for spikes and crashes with rate limits, health checks and autoscaling.
