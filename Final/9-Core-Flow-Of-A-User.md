# 9. Core Flow of a User

When an interviewer asks you to trace the complete lifecycle of a request, they want to see if you understand the boundary between the Frontend (React), the Network (HTTP/WebSockets), and the Backend (Express/Redis/Postgres). This document provides a massive, granular breakdown of the exact user journey.

---

## Section 1: The Initial Trigger (The User Action)

### 1.1 The React UI Trigger
1.  **The DOM Event:** A user sitting in Tokyo opens the React dashboard and clicks the "Run Heavy Task" button. The React `onClick` synthetic event handler fires.
2.  **The State Update:** React immediately sets a localized state (`isSubmitting: true`) to disable the button and prevent the user from double-clicking it.
3.  **The Network Request:** The `axios.post('https://api.queueflow.com/jobs')` function is executed.
4.  **The Browser Engine:** The Chrome V8 engine hands the request off to the browser's Network stack. The browser performs a DNS lookup, establishes a TCP handshake with the AWS Application Load Balancer (ALB), performs the TLS 1.3 handshake (SSL), and finally transmits the HTTP `POST` payload.

### 1.2 The API Ingestion (The Producer)
1.  **The Entry Point:** The ALB decrypts the HTTPS traffic and routes the raw HTTP request to an available `app.js` Docker container running in the private subnet.
2.  **The Validation:** Express middleware (`express.json()`) parses the body. 
3.  **The UUID Generation:** Node.js executes `crypto.randomUUID()` in memory. This takes $< 0.01$ milliseconds. We now have a unique identifier for distributed tracing.
4.  **The Database Lock-In:** `app.js` grabs a connection from `pg.Pool` and executes the `INSERT INTO jobs` query. The PostgreSQL writer node in `us-east-1` commits the transaction to its Write-Ahead Log.
5.  **The Routing:** `app.js` executes `redis.lPush` to push the payload into the `jobQueue:high` List.
6.  **The Acknowledgment:** The Express route reaches the bottom, executes `res.status(201).json({ id })`, and frees the HTTP connection. The user's browser receives the 201 Created response. The user sees a yellow "Queued" row appear on their screen.

> **Exact Interview Script:**
> "The user journey begins with extreme decoupling. When the user clicks the button, I don't want their browser waiting 30 seconds for an AI generation. The React client makes a simple HTTP POST. The Express API validates it, saves the UUID to Postgres for durability, pushes it to Redis for routing, and instantly returns a 201 Created. From the user's perspective, the UI feels incredibly snappy because the entire network round trip takes less than 50 milliseconds. They see a 'Queued' state, completely unaware that the heavy lifting has just begun asynchronously on the backend."

---

## Section 2: The WebSocket Deep Dive (The Real-Time UI)

The user is now staring at a yellow "Queued" bar. How does it turn blue ("Processing") and then green ("Completed") without the user hitting the Refresh button?

### 2.1 The WebSocket Handshake
1.  When the user first loaded the React page, the `socket.io-client` library executed an HTTP Upgrade request to the API Gateway.
2.  The API Gateway agreed to the upgrade, switching protocols from HTTP/1.1 to a persistent WebSocket TCP tunnel (ws://).
3.  The specific `app.js` container handling that user holds that TCP connection open in memory.

### 2.2 The Pub/Sub Relay
1.  **The Worker Action:** Deep in the private subnet, a `worker.js` container pops the job from Redis. It executes the Postgres atomic lock: `UPDATE jobs SET status='processing'`.
2.  **The Event Broadcast:** The worker immediately executes `redis.publish('job_updates', JSON.stringify({ id, status: 'processing' }))`.
3.  **The Redis Backplane:** The Redis server, acting as a massive event router, duplicates this message and blasts it to every single `app.js` container that has a duplicate client running `SUBSCRIBE`.
4.  **The API Fan-Out:** The `app.js` container receives the message. It executes `io.emit('job_updated', payload)`. Socket.io iterates through all open WebSocket connections it holds in memory and streams the binary frame down the active TCP tunnels.
5.  **The React Reconciliation:** The Chrome browser receives the WebSocket frame. The React `useEffect` listener catches the event. It executes `setJobs(prev => prev.map(job => ...))`. React calculates the virtual DOM diff, realizes the status changed from 'queued' to 'processing', and re-renders the specific DOM node. The yellow bar instantly turns blue.

> **Exact Interview Script:**
> "To prevent the React dashboard from hammering the Postgres database with HTTP polling, I implemented a real-time event pipeline using Redis Pub/Sub and Socket.io. When a worker alters the database state, it blasts an ephemeral event to a Redis channel. The Express API, which maintains persistent WebSocket connections with the users, hears this event and funnels it down the socket. This creates a magical UX where the user sees the progress bar transition states in real-time, while the database CPU remains at absolute 0% utilization."

---

## Section 3: User Edge Cases (The Real World)

What happens when the user doesn't behave perfectly? 

### 3.1 The "Closed Laptop" Scenario
*   **The Problem:** The user clicks "Submit", gets the 201 Created response, and instantly slams their laptop shut, killing the internet connection.
*   **What Happens:** The WebSocket TCP tunnel is violently severed. When the worker finishes the job and fires the Redis Pub/Sub event, Socket.io attempts to send it, realizes the socket is dead, and silently drops the message.
*   **The Resolution:** The job is safely processing on the backend. When the user opens their laptop 5 hours later, the React `useEffect` hook fires an HTTP `GET /jobs` request. It pulls the absolute truth from Postgres, realizes the job is `completed`, and updates the UI accordingly. 

### 3.2 The "Double Click" (Rage Clicking)
*   **The Problem:** The network is slow. The user clicks "Submit", nothing happens immediately, so they rage-click it 5 more times.
*   **What Happens:** The React UI attempts to disable the button, but 5 requests slip through before the state re-renders. The API receives 5 identical requests.
*   **The Resolution:** If the job requires idempotency (e.g., charging a card), the API would generate an `Idempotency-Key` hash based on the payload. Postgres would have a `UNIQUE` constraint on this hash. The first request succeeds. The next 4 requests throw a PostgreSQL constraint violation error, which the Express `catch` block intercepts, returning a 409 Conflict. The queue is protected.

### 3.3 The "Refreshing the Page" Problem
*   **The Problem:** The user hits F5 (Refresh) right when the job is in the `processing` state.
*   **What Happens:** The entire React state is wiped from the browser's RAM. The WebSocket connection is killed and a new one is established.
*   **The Resolution:** Because the architecture is stateful at the database level, the React app's initial `GET /jobs` load fetches the exact current state from Postgres. It sees the job is `processing` and resumes listening on the new WebSocket for the final `completed` event. The UI state is perfectly recovered.

### 3.4 The "100-Hour Job" Timeout
*   **The Problem:** The user submits a video encoding task. The UI shows 'processing'. It takes 3 hours. The user leaves the tab open.
*   **What Happens:** Most Load Balancers sever idle connections after 60 seconds. However, WebSockets implement internal 'Ping/Pong' heartbeat frames. Every 25 seconds, Socket.io sends an invisible 1-byte ping to the API. The API sends a pong back. This keeps the AWS ALB firewall from terminating the connection, meaning the user can leave the tab open for 3 days and the final 'completed' event will still arrive perfectly.

---

## Section 4: The Final Review

If you are asked to walk through the system in an interview, combine Sections 1, 2, and 3 into this final narrative script.

> **Exact Interview Script:**
> "Let's trace a job end-to-end. The React UI triggers an HTTP POST. The Express API ingests it, instantly securing the data in Postgres for ACID durability, then pushes the UUID to a Redis List for high-speed routing. It returns a 201 to the user.
> 
> Across the network, a Node.js worker, suspended on a `BRPOP` socket, wakes up instantly. To prevent race conditions with other workers, it executes an Optimistic Concurrency Control lock in Postgres via `UPDATE ... RETURNING`. It wins the lock, processes the heavy LLM inference, and marks it completed in Postgres.
> 
> Finally, the worker fires a Redis `PUBLISH` event. The Express API catches it on a dedicated subscriber client, and streams it down an open Socket.io WebSocket tunnel to the React frontend. The user experiences a seamless, real-time progress bar without the system ever having to poll the database. And if the user closes their laptop mid-flight, the system's strict database-first architecture guarantees their job finishes safely in the background."
