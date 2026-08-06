# 3. Deep-Dive File Breakdown

When an interviewer asks you to dive into the technical specifics of your codebase, you need to know exactly which file is responsible for what logic. This document breaks down the 5 core backend files of the `QueueFlow` architecture.

---

## 1. `app.js` (The Producer & API Gateway)

### Primary Responsibilities
*   Serve the REST API endpoints (`GET /jobs`, `POST /jobs`, `DELETE /jobs`).
*   Act as the **Producer** by writing initial job states to PostgreSQL and pushing them into Redis.
*   Serve the Socket.io WebSocket server to push real-time updates to the React UI.

### Core Mechanisms & Design Patterns
*   **The Duplicate Redis Client:** Notice how `app.js` imports both `redisClient` and `redisSubClient`. The standard client is used for `LPUSH` (sending jobs). The `SubClient` is exclusively dedicated to `SUBSCRIBE('job_updates')`. If you used one client for both, the `SUBSCRIBE` command would lock the client, crashing your `POST /jobs` endpoint.
*   **Database-First Guarantee:** In the `POST /jobs` route, look at the order of operations. It executes `pool.query('INSERT INTO jobs...')` **before** it calls `redis.lPush()`. If Postgres fails, the API returns a 500 error and nothing goes to the queue. If it pushed to Redis first and Postgres failed, workers would try to process a "ghost" job that doesn't exist in the database.

---

## 2. `worker.js` (The Consumer & Engine)

### Primary Responsibilities
*   Act as the **Consumer** by pulling jobs off the Redis lists and processing them.
*   Enforce **Idempotency** (prevent double processing) via PostgreSQL atomic locking.
*   Manage the **Failure Pipeline** (Exponential Backoff and Dead-Lettering).
*   Run the **Self-Healing Maintenance Loops**.

### Core Mechanisms & Design Patterns
*   **The Infinite `while(true)` Loop:** Inside `runWorkerLoop()`, the process is trapped in a permanent loop. It survives because `BRPOP` puts the thread to sleep, preventing a CPU spin-lock.
*   **The Blocking Client (`redisBlocker`):** Just like `app.js`, `worker.js` uses a separate Redis connection specifically for `BRPOP`. If it didn't, the worker wouldn't be able to use `ZADD` or `PUBLISH` while waiting for new jobs.
*   **The Atomic Claim (`UPDATE ... RETURNING *`):** Inside `handleJob()`, the worker generates a random `processingToken`. It attempts to update the job to `processing`. If `result.rowCount === 0`, it means another worker stole it, and this worker silently drops the payload.
*   **The Triple Maintenance Sweep:** Every single time the `while` loop iterates (which is at least every 5 seconds), it runs three background tasks:
    1.  `promoteDueRetries()`: Moves expired backoff jobs from the `ZSET` back to the active `LIST`.
    2.  `recoverStalledProcessingJobs()`: (Every 15s) Resets jobs that have been stuck in 'processing' for >60 seconds.
    3.  `recoverMissingQueuedJobs()`: (Every 15s) Pushes jobs back to Redis if Redis crashed but they are still marked as 'queued' in Postgres.

---

## 3. `redis.js` (The Cache Manager)

### Primary Responsibilities
*   Establish the connection to the Redis server.
*   Provide robust error handling and reconnection logic.
*   Export multiple client instances for different blocking/non-blocking contexts.

### Core Mechanisms & Design Patterns
*   **Singleton Pattern:** By connecting to Redis once in this file and exporting the connected clients, the rest of the application shares the exact same connection pool. You don't want `app.js` opening 500 new Redis connections for 500 incoming HTTP requests.
*   **Promisification:** Modern Node.js uses `async/await`. The `redis` v4 library used in this project natively supports Promises, preventing "callback hell."

---

## 4. `db.js` (The Database Driver)

### Primary Responsibilities
*   Establish the connection to the PostgreSQL database.
*   Manage the Connection Pool.

### Core Mechanisms & Design Patterns
*   **Connection Pooling (`pg.Pool`):** Instead of using a single `Client` (which would bottleneck under load) or creating a new `Client` for every request (which would cause massive TCP handshake latency), `db.js` exports a `Pool`. 
*   **How Pooling Works:** When `app.js` needs to insert a job, it transparently borrows a pre-opened connection from the pool, runs the query, and instantly returns the connection to the pool for the next request to use.

---

## 5. `queues.js` (The Central Nervous System)

### Primary Responsibilities
*   Store the central configuration for queue names and priorities.
*   Ensure that both the Producer (`app.js`) and Consumer (`worker.js`) agree on exactly what the Redis keys are called without importing each other.

### Core Mechanisms & Design Patterns
*   **The Array Order (`QUEUES_IN_PRIORITY_ORDER`):** This is the most important line in the file: `['jobQueue:high', 'jobQueue', 'jobQueue:low']`. 
*   **Why it matters:** When `worker.js` passes this array to `BRPOP`, Redis respects the exact order of the array. This single array natively implements strict Priority Queuing. Redis will absolutely refuse to look at the `normal` queue if there is even a single job in the `high` queue.
*   **The Delayed Queue (`jobQueue:delayed`):** Defines the single ZSET key used for all exponential backoff retries.

---

## 🗣️ Exact Interview Script: Explaining File Architecture

When the interviewer says: **"How did you structure your backend code to keep responsibilities separated?"**

> **Your Exact Script:**
> 
> "I strictly separated the Producer and Consumer logic into different files to mimic a true microservice architecture. 
> 
> My `app.js` acts purely as the API gateway and the Producer. Its only job is to validate incoming HTTP requests, write the initial state to Postgres via my `db.js` connection pool, and push the payload into Redis. It never processes anything.
> 
> My `worker.js` is the Consumer. It's a completely separate Node process that runs a `while(true)` loop. It uses a dedicated, blocking Redis client from my `redis.js` file to listen for jobs using `BRPOP`.
> 
> To prevent these two files from becoming tightly coupled or creating circular dependencies, I extracted all the queue logic—like priority levels and exact Redis key names—into a shared `queues.js` configuration file. This guarantees that my Producer is pushing to the exact same array of priority keys that my Consumer is listening to."
