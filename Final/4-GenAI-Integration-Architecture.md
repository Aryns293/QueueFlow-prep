# 4. GenAI Integration Architecture

Generative AI (LLMs, Image Generators) introduces unique challenges to backend architectures. AI inferences take a long time (high latency), fail unpredictably (hallucinations), and hit strict rate limits (OpenAI tokens per minute).

This document explains exactly how you would integrate GenAI into `QueueFlow`, and how to answer GenAI system design questions.

---

## 🧠 Why GenAI Demands a Job Queue

If you try to run an OpenAI API call synchronously inside an Express route:
1.  The user's browser will sit spinning for 30 seconds.
2.  If the user refreshes the page, you've just paid for the API call twice.
3.  If 100 users click the button at the same time, your server will freeze, and OpenAI will ban you for hitting rate limits (429 Too Many Requests).

### The Solution: Asynchronous Inference via QueueFlow
By putting GenAI calls into `QueueFlow`:
*   **Latency is hidden:** The API instantly returns a `jobId`, and the user sees a "Generating..." UI via WebSockets.
*   **Peak Shaving:** If 100 users click generate, the jobs are buffered in Redis. Your workers process them at a controlled pace (e.g., 5 at a time), ensuring you never hit the OpenAI rate limits.
*   **Retry on Failure:** If OpenAI goes down, the Exponential Backoff system safely retries the prompt 4 seconds later without failing the user's request.

---

## 🛠️ How to Add GenAI to Your Project

If an interviewer asks: *"How would you enhance this project to generate AI responses?"*

### 1. Update the Worker (`worker.js`)
Inside `processJob()`, you would add a switch case for AI tasks:

```javascript
async function processJob(job) {
    if (job.type === 'generate-text') {
        // 1. Call OpenAI API
        const response = await openai.chat.completions.create({
            model: "gpt-4",
            messages: [{ role: "user", content: job.payload.prompt }],
        });
        
        // 2. Save the AI response back to Postgres in the payload
        job.payload.ai_response = response.choices[0].message.content;
    }
}
```

### 2. Add AI-Specific Error Handling (The DLQ)
LLMs sometimes return garbage data or refuse to answer due to content moderation.
If the LLM returns an error or a hallucinated JSON structure, the `try/catch` block will throw.
*   The job will retry using **Exponential Backoff**.
*   If it fails 3 times, it goes to the **Dead-Letter Queue (DLQ)** (marked `failed` in Postgres). 
*   A human operator can look at the DLQ in the dashboard, fix the prompt, and click "Retry".

---

## 🗣️ Exact Interview Scripts for GenAI Questions

### Question 1: "How do you handle API Rate Limits (429 Errors) from OpenAI?"

> **Your Exact Script:**
> 
> "In my QueueFlow architecture, if a worker hits an OpenAI rate limit and receives a 429 error, the worker's `catch` block intercepts it. Because rate limits are usually temporary, I don't want to fail the job permanently. 
> 
> Instead, my worker dynamically calculates an Exponential Backoff delay—say, 4 seconds, then 8 seconds—and uses a Redis `ZADD` command to schedule the retry in a Delayed Queue. This completely removes the pressure from the OpenAI API, giving it time to recover, while ensuring the user's prompt is safely stored and eventually processed without them having to click 'Generate' again."

### Question 2: "AI inferences can take up to 60 seconds. Does this block your entire queue?"

> **Your Exact Script:**
> 
> "It doesn't block the queue because of how Node.js handles asynchronous I/O. When my worker calls the OpenAI API via `fetch` or `axios`, the Node.js event loop suspends that specific function and can immediately begin processing the next job.
> 
> However, to prevent a single worker from picking up 1,000 AI jobs and running out of RAM, I would implement a **Concurrency Limit** in `worker.js`. I would use a semaphore or an active job counter to ensure that a single worker process only pulls a maximum of, say, 10 concurrent jobs from Redis at a time."

### Question 3: "What if the AI hallucinates or generates toxic content? How do you handle Human-in-the-Loop (HITL)?"

> **Your Exact Script:**
> 
> "QueueFlow's state machine makes Human-in-the-Loop very easy to implement. I would add a new state to my PostgreSQL database called `pending_review`.
> 
> When the worker gets the response from the LLM, it runs it through a local sentiment analysis or moderation filter. If it gets flagged, the worker updates the PostgreSQL status to `pending_review` instead of `completed`. 
> 
> This job would appear in a special tab on the React dashboard. A human admin can read the AI's output, edit it if necessary, and click 'Approve'. That click hits an Express API endpoint that updates the status to `completed` and fires the WebSocket event to the end user."

### Question 4: "If your worker crashes while waiting for OpenAI to respond, how do you prevent the user from waiting forever?"

> **Your Exact Script:**
> 
> "Because OpenAI calls take a long time, the risk of a worker crashing mid-flight is high. That's exactly why I implemented the **Atomic Lock and Reaper Loop** in QueueFlow.
> 
> When the worker pulls the job, it locks it in Postgres with `status = 'processing'`. If the worker crashes, the job is stuck. But my background Reaper loop runs every 15 seconds, scanning Postgres. If it sees a job that has been `processing` for more than 2 minutes (which is longer than the max OpenAI timeout), it assumes the worker died. It instantly resets the job to `queued` and re-pushes it to Redis, so a healthy worker can pick it up and try the prompt again."
