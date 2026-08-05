# QueueFlow — Complete Interview Prep Guide

Everything you need to learn, understand, and confidently answer **any** question about this project.

---

## 1. High-Level System Design Concepts

- Producer-Consumer Pattern
- Message Queue / Job Queue
- Distributed Systems
- Asynchronous Processing
- Decoupling Producers from Consumers
- Horizontal Scaling
- Vertical Scaling
- Event-Driven Architecture
- Background Job Processing
- Task Scheduling vs Job Queuing
- Fan-out / Fan-in Patterns
- Throughput vs Latency Tradeoffs
- Backpressure in Queue Systems
- At-Least-Once Delivery
- At-Most-Once Delivery
- Exactly-Once Delivery (and why it's hard)
- Idempotency (why it matters for retries)

---

## 2. Job Lifecycle & State Machine

- Job States: `queued → processing → completed`
- Job States: `queued → processing → failed` (after exhausting retries)
- State Transitions as a Finite State Machine
- Why status is tracked in PostgreSQL, not Redis
- Atomic status claims (`UPDATE ... WHERE status = 'queued' RETURNING id`)
- Preventing double-processing of the same job
- Stale Redis entry handling (worker pops a job that was already processed)
- Dead-Letter Queue concept (jobs that permanently fail)
- Job Retry Count tracking
- Maximum Retries (`max_retries` default = 3)
- Error message persistence in PostgreSQL

---

## 3. Redis — Concepts & Commands Used

### Core Concepts
- Redis as an In-Memory Data Store
- Redis Data Structures (Strings, Lists, Sets, Sorted Sets, Hashes)
- Redis LIST data structure (used as queue backbone)
- FIFO ordering within a single list
- Redis Single-Threaded Model
- Redis Persistence (RDB snapshots, AOF)
- Redis Pub/Sub (not used here, but know the difference)
- Redis Streams vs Lists (tradeoff discussion)
- Why raw Redis commands instead of BullMQ

### Commands Used
- `LPUSH` — push to the left (head) of a list, O(1)
- `RPOP` — pop from the right (tail) of a list
- `BRPOP` — blocking pop (waits until an element is available), timeout parameter
- `DEL` — delete keys (used in purge)
- `LLEN` — get list length (know for depth metrics)

### Redis in This Project
- Three separate Redis LIST keys for priority (`jobQueue:high`, `jobQueue`, `jobQueue:low`)
- BRPOP across multiple keys checks them in order (priority implementation)
- BRPOP with timeout 0 (block forever, no CPU polling)
- Atomic pop guarantee (safe for multiple concurrent workers)
- Why a duplicate Redis client is needed (`redis.duplicate()`)
- Blocking client vs non-blocking client separation
- Redis connection string format (`redis://host:port`)
- `createClient()` from `redis` npm package (Node Redis v4+)
- Redis error event handling (`client.on('error', ...)`)
- Lazy singleton pattern for Redis client (`getClient()`)

---

## 4. PostgreSQL — Concepts & SQL Used

### Core Concepts
- Relational Database
- ACID Properties (Atomicity, Consistency, Isolation, Durability)
- Why PostgreSQL for persistent state (survives crashes)
- Connection Pooling (`pg.Pool`)
- Connection String format (`postgresql://user:password@host:port/db`)
- SSL configuration for production (`rejectUnauthorized: false`)
- Parameterized Queries (`$1, $2, $3` — prevents SQL injection)
- `JSONB` column type (storing arbitrary job payloads)
- `UUID` primary keys (`gen_random_uuid()`)
- `pgcrypto` extension (for `gen_random_uuid()`)
- Database Indexing and why it matters
- `TIMESTAMP` columns (`created_at`, `updated_at`)
- Default values in CREATE TABLE
- `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` (safe idempotent migration)
- `CREATE INDEX IF NOT EXISTS`
- `TRUNCATE TABLE` (vs DELETE — faster, not row-by-row)

### SQL Queries Used
- `INSERT INTO jobs (...) VALUES ($1, $2, ...)` — job creation
- `UPDATE jobs SET status = 'processing' WHERE id = $1 AND status = 'queued' RETURNING id` — atomic claim
- `UPDATE jobs SET status = 'completed' WHERE id = $1` — mark complete
- `UPDATE jobs SET status = 'failed', error = $1 WHERE id = $2` — mark failed
- `UPDATE jobs SET retry_count = $1, status = 'queued', error = $2 WHERE id = $3` — retry
- `UPDATE jobs SET status = 'queued', retry_count = 0, error = NULL WHERE id = $1 AND status = 'failed' RETURNING *` — manual retry from dashboard
- `SELECT * FROM jobs ORDER BY created_at DESC LIMIT 200` — dashboard listing
- `SELECT * FROM jobs WHERE id = $1` — single job fetch
- `SELECT id, type, payload, priority FROM jobs WHERE status = 'queued'` — orphan recovery
- `SELECT retry_count, max_retries FROM jobs WHERE id = $1` — retry check
- Aggregate query with `COUNT(*) FILTER (WHERE ...)` — stats endpoint
- `COALESCE()` — null-safe default values
- `AVG()` with FILTER clause
- `EXTRACT(EPOCH FROM ...)` — timestamp arithmetic
- `::int`, `::float` type casting in PostgreSQL

### Database Schema
- `id` — UUID PRIMARY KEY
- `type` — VARCHAR(50) NOT NULL
- `payload` — JSONB NOT NULL
- `status` — VARCHAR(20) DEFAULT 'queued'
- `retry_count` — INTEGER DEFAULT 0
- `max_retries` — INTEGER DEFAULT 3
- `error` — TEXT (nullable)
- `priority` — VARCHAR(10) NOT NULL DEFAULT 'normal'
- `created_at` — TIMESTAMP DEFAULT NOW()
- `updated_at` — TIMESTAMP DEFAULT NOW()

### Indexes
- `idx_jobs_status` — on `status` column (dashboard and /stats filter by status constantly)
- `idx_jobs_created_at` — on `created_at DESC` (ORDER BY optimization)

---

## 5. Priority Queue Implementation

- Three priority levels: `high`, `normal`, `low`
- Three Redis LIST keys: `jobQueue:high`, `jobQueue`, `jobQueue:low`
- BRPOP checks keys in order → high-priority jobs always dequeued first
- Within a single priority level: strict FIFO
- `queueKeyForPriority()` — maps priority string to Redis key name
- `QUEUES_IN_PRIORITY_ORDER` — ordered array of queue names
- No extra infrastructure needed (same LPUSH/BRPOP primitives, just more keys)
- Priority stored in PostgreSQL row alongside other job metadata
- Priority validated against `PRIORITIES` enum before DB insert

---

## 6. Retry Logic & Exponential Backoff

- Automatic retry on failure
- Max 3 retries (configurable via `max_retries`)
- Exponential backoff formula: `2000 * 2^retry_count` (2s → 4s → 8s)
- `scheduleRetry()` — schedules ZADD with due timestamp (score) to DELAYED_QUEUE
- `promoteDueRetries()` — cron-like check to move due jobs from ZSET back to LIST via ZRANGEBYSCORE
- Retry count incremented in PostgreSQL before re-queue
- Job status set back to `queued` during retry
- Error message stored even on retry (overwritten each attempt)
- After exhausting retries → `status = 'failed'` (dead-letter)
- 100% durable delayed queue using Redis Sorted Sets (`ZSET`) prevents backoff loss on worker crash

---

## 7. Crash Recovery

- `recoverOrphanedJobs()` function in `worker.js`
- Runs once on every worker startup
- Queries PostgreSQL for all jobs with `status = 'queued'`
- Re-pushes each to the correct Redis priority queue
- Covers the case: process died between DB insert and Redis push
- Covers the case: Redis was temporarily unavailable
- Safe because handleJob's atomic `WHERE status = 'queued'` prevents re-processing
- Logs how many orphaned jobs were recovered

---

## 8. Worker Process — Deep Dive

### Functions
- `processJob(job)` — simulates work (200ms sleep), 85% random failure rate
- `handleJob(rawData, redis)` — parses JSON, claims job atomically, runs processJob, handles success/failure
- `scheduleRetry(redis, job, delayMs)` — durable delayed re-queue via Redis ZSET
- `promoteDueRetries(redis)` — promotes ready jobs from ZSET to standard queues
- `recoverOrphanedJobs(redis)` — startup crash recovery
- `startWorker()` — main entry: connects Redis, recovers orphans, enters infinite loop

### Concepts
- Infinite `while (true)` loop with `BRPOP`
- Polling fallback with `rPop` + `sleep(1000)` (for serverless Redis compatibility like Upstash)
- Non-blocking I/O (Node.js event loop processes other jobs during backoff wait)
- Duplicate Redis client (`redis.duplicate()`) — one for blocking pop, one for LPUSH
- JSON.parse safety (try/catch for malformed queue entries)
- Atomic job claim (`UPDATE WHERE status = 'queued'`) — prevents duplicate processing
- Stale entry detection (if claim returns 0 rows, skip the job)
- `require.main === module` pattern — auto-start only when run directly, not when imported
- `startWorker()` exported as module — allows `RUN_WORKER_INLINE` mode

---

## 9. API Server (Express) — Deep Dive

### Modules & Middleware
- `express()` — create Express app
- `cors()` — enable Cross-Origin Resource Sharing
- `express.json()` — parse JSON request bodies
- `require('crypto').randomUUID` — generate UUIDs (no external uuid package)

### Routes & Handlers
- `GET /` — root health message
- `GET /health` — JSON health check `{ status: 'ok' }`
- `POST /jobs` — create a new job (producer)
- `GET /jobs` — list all jobs (dashboard, LIMIT 200, ORDER BY created_at DESC)
- `GET /jobs/:id` — get single job by UUID
- `GET /stats` — aggregate queue metrics
- `POST /jobs/:id/retry` — retry a failed job
- `DELETE /jobs` — purge all jobs (TRUNCATE + DEL Redis keys)

### Validation
- `type` is required (400 if missing)
- `type` must be in `JOB_TYPES` enum: `email`, `sms`, `image-resize`, `pdf-generate`, `analytics`
- `priority` must be in `PRIORITIES` enum: `high`, `normal`, `low`
- `priority` defaults to `'normal'` if omitted
- UUID format validation with regex (`UUID_RE`)
- 404 for well-formed but non-existent job IDs
- 400 for malformed (non-UUID) job IDs

### Stats Endpoint Metrics
- `total` — total job count
- `queued` — jobs waiting
- `processing` — jobs being worked on
- `completed` — successfully finished
- `failed` — permanently failed (dead-letter)
- `avgRetries` — average retry count across completed + failed
- `avgCompletionSeconds` — average time from created_at to updated_at for completed jobs
- `successRate` — `completed / (completed + failed) * 100`, rounded to 1 decimal

### Error Handling
- try/catch on every route handler
- 500 response with `error: err.message`
- Console error logging

---

## 10. Entry Point (`index.js`)

- `dotenv.config()` — load `.env` file
- Connects Redis before starting Express
- `app.listen(PORT)` — start HTTP server
- `RUN_WORKER_INLINE` env var — runs worker inside API process
- `require('./worker').startWorker()` — inline worker for single-process deployment
- `process.exit(1)` on startup failure

---

## 11. Shared Module (`queues.js`)

- `JOB_TYPES` — enum array of valid job types
- `PRIORITIES` — enum array of valid priorities
- `QUEUES_IN_PRIORITY_ORDER` — ordered queue key names
- `queueKeyForPriority(priority)` — maps priority string to Redis key
- Shared between producer (app.js) and consumer (worker.js)
- Prevents the two sides from disagreeing on queue names

---

## 12. Database Connection (`db.js`)

- `pg.Pool` — connection pool (not single client)
- Connection pooling concept (reuse connections, avoid overhead)
- `connectionString` from `DATABASE_URL` env var
- SSL enabled only in production (`NODE_ENV === 'production'`)
- `rejectUnauthorized: false` for cloud-hosted Postgres (Neon, Supabase, etc.)
- Pool exported as module singleton

---

## 13. Redis Connection (`redis.js`)

- Lazy singleton pattern — `getClient()` creates on first call, returns cached after
- `createClient({ url: process.env.REDIS_URL })` — Node Redis v4+ API
- Error event listener (logs but doesn't crash)
- `client.connect()` — explicit connection (Node Redis v4+ requires this)
- Module exports `{ getClient }` (not the raw client)

---

## 14. Migration Script (`migrate.js`)

- `CREATE EXTENSION IF NOT EXISTS "pgcrypto"` — enables gen_random_uuid()
- `CREATE TABLE IF NOT EXISTS jobs (...)` — idempotent table creation
- `ALTER TABLE ... ADD COLUMN IF NOT EXISTS priority` — safe column addition
- `CREATE INDEX IF NOT EXISTS` — idempotent index creation
- `process.exit(0)` on success, `process.exit(1)` on failure
- Runs as one-shot script, not as part of the app startup
- Docker Compose runs it as a separate service that exits after completion
- `service_completed_successfully` condition — API and worker wait for migration

---

## 15. Simulation Script (`simulate.js`)

- Fires massive concurrency payloads (1,000 to 10,000+ jobs) with random types and weighted priorities
- `PRIORITY_WEIGHTS` — 20% high, 70% normal, 10% low
- `randomPriority()` — weighted random selection using cumulative probability
- Uses `axios.post()` to hit the API (not direct DB/Redis access)
- 100ms gap between each job (`setTimeout`)
- Demonstrates retry, backoff, and dead-lettering under high failure rate
- `process.exit(0)` after completion

---

## 16. Frontend — React Dashboard

### React Concepts Used
- Functional Components
- `useState` — local state management
- `useEffect` — side effects (polling interval)
- `useCallback` — memoized function references
- `useMemo` — derived/computed values
- `useRef` — mutable ref for previous jobs (no re-render)
- React.StrictMode
- `createRoot` (React 18+ concurrent API)
- JSX syntax
- Conditional rendering (`&&`, ternary)
- List rendering with `.map()` and `key` prop
- Controlled form inputs
- Event handling (`onClick`, `onSubmit`, `onChange`)
- `event.preventDefault()` (form submission)
- `event.stopPropagation()` (modal click-through)
- Component composition and props

### Components
- `App` — root component, state & data fetching
- `MetricCard` — displays a single metric with tone color
- `StatusPill` — colored status badge
- `PriorityPill` — colored priority badge
- `JobCard` — individual job card in pipeline lane
- `PipelineLane` — column of jobs for one status
- `JobModal` — full detail overlay for a job

### Utility Functions
- `normalizeJob(job)` — sanitize status/priority to known values
- `timeAgo(dateStr)` — exact-second relative time string (0s, 1s, 2s) for visual backoff validation
- `relativeTime(dateStr)` — "5s ago" or "just now"
- `formatDateTime(dateStr)` — locale-aware full date string
- `formatDuration(seconds)` — human-readable duration
- `typeLabel(type)` — map type value to display label
- `typeAccent(type)` — map type to color
- `payloadSummary(payload)` — truncated summary of payload
- `deriveStats(jobs, apiStats)` — merge local counts with API stats
- `buildActivity(previous, next)` — diff jobs arrays to build activity feed

### State Variables
- `jobs` — array of all jobs from API
- `stats` — aggregate stats from `/stats`
- `activity` — activity feed events (state changes)
- `lastSync` — timestamp of last successful fetch
- `error` — current error message or null
- `loading` — initial loading state
- `creating` — creating a job in progress
- `bursting` — demo burst in progress
- `filters` — `{ status, priority, query }` filter state
- `selectedJob` — job shown in modal (or null)
- `form` — `{ type, priority, payload }` job creation form
- `previousJobsRef` — ref holding previous jobs array for diff

### Dashboard Features
- **WebSockets** for zero-latency real-time state synchronization (Socket.io)
- Strict `LIMIT 200` UI safeguard preventing browser crash on massive 10,000+ job datasets
- Pipeline view (Kanban-style: Queued → Processing → Completed → Failed)
- Metric cards (Queued, Processing, Success Rate, Avg Time)
- Priority mix bar chart (proportional bar per priority)
- Status filter (segmented control)
- Priority filter (dropdown)
- Text search (across id, type, status, priority, payload, error)
- Job table (latest 8 visible jobs)
- Activity feed (state change events, max 10)
- Job detail modal (click any card)
- Create job form (type, priority, payload)
- Demo burst button (12 jobs at once)
- Purge All button (with confirm dialog)
- Retry button on failed jobs
- Live/Offline connection indicator
- Toast notifications (react-hot-toast)
- Auto-animate transitions (@formkit/auto-animate)
- Queued lane sorted by priority then FIFO
- Other lanes sorted by most recently updated

### CSS / Styling Concepts
- CSS Custom Properties (`--metric-tone`, `--pill-tone`, etc.)
- Glassmorphism (`backdrop-filter: blur()`, semi-transparent backgrounds)
- CSS Grid Layout (`grid-template-columns`, `gap`)
- CSS Flexbox (`display: flex`, `align-items`, `gap`)
- `min()` function for responsive width
- `minmax()` in grid
- `color-mix()` CSS function
- Custom scrollbar styling (`::-webkit-scrollbar`)
- CSS Animations (`@keyframes card-in, live-pulse, processing-glow, fade-in, slide-up`)
- CSS Transitions (`transition: transform, box-shadow, background`)
- Hover effects (`transform: translateY()`, `scale()`)
- `:not(:disabled)` pseudo-selector
- `border-radius: 999px` (pill shape)
- Responsive breakpoints (`@media max-width: 1180px, 760px`)
- Dark theme color palette (slate/gray tones)
- `::selection` pseudo-element styling
- `radial-gradient()` background (subtle ambient glow)
- `linear-gradient()` background
- `background-attachment: fixed`
- `box-shadow` for depth and glow effects
- `text-transform: uppercase`, `letter-spacing`
- `font-synthesis: none`, `-webkit-font-smoothing: antialiased`

---

## 17. Libraries & Dependencies

### Backend
- `express` — HTTP server framework
- `cors` — Cross-Origin Resource Sharing middleware
- `dotenv` — load .env file into process.env
- `pg` — PostgreSQL client for Node.js (node-postgres)
- `redis` — Node Redis v4+ client
- `axios` — HTTP client (used in simulate.js)
- `nodemon` (dev) — auto-restart on file change
- `jest` (dev) — testing framework
- `supertest` (dev) — HTTP assertions for Express

### Frontend
- `react` — UI library
- `react-dom` — React DOM renderer
- `axios` — HTTP client for API calls
- `@formkit/auto-animate` — automatic animation for list additions/removals
- `react-hot-toast` — toast notification library
- `vite` (dev) — build tool and dev server
- `@vitejs/plugin-react` (dev) — Vite React plugin
- `eslint` (dev) — linter
- `eslint-plugin-react-hooks` (dev) — React hooks lint rules
- `eslint-plugin-react-refresh` (dev) — React Fast Refresh lint rules
- `globals` (dev) — global variable definitions for ESLint

---

## 18. Testing — Concepts & Implementation

### Testing Concepts
- Integration Testing (vs Unit Testing vs E2E)
- Testing against real infrastructure (not mocks)
- `supertest` — test Express routes without starting a server
- `jest` — test runner, assertions, lifecycle hooks
- `describe()` and `test()` blocks
- `afterAll()` — cleanup (close Redis + Postgres connections)
- Test isolation and ordering (`--runInBand` — sequential execution)
- `testTimeout: 15000` — 15s timeout for integration tests

### Test Cases Covered
- `GET /health` returns 200 with `{ status: 'ok' }`
- `POST /jobs` rejects missing `type` (400)
- `POST /jobs` rejects unknown `type` (400)
- `POST /jobs` rejects invalid `priority` (400)
- `POST /jobs` creates a job, defaults to normal priority (200)
- `POST /jobs` accepts explicit high priority (200)
- `GET /jobs/:id` returns the created job (200)
- `GET /jobs/:id` returns 400 for malformed UUID
- `GET /jobs/:id` returns 404 for well-formed but missing UUID
- `GET /jobs` returns array including created job
- `GET /stats` returns aggregate counts with successRate and avgRetries

### Assertions Used
- `expect(res.status).toBe()`
- `expect(res.body.status).toBe()`
- `expect(res.body.success).toBe(true)`
- `expect(res.body.jobId).toBeDefined()`
- `expect(Array.isArray(res.body)).toBe(true)`
- `expect(res.body.some(...))`
- `expect(res.body.total).toBeGreaterThanOrEqual()`
- `expect(res.body).toHaveProperty()`

---

## 19. Docker & Docker Compose

### Docker Concepts
- Containerization
- Docker Image vs Container
- Dockerfile
- Build context
- Layer caching
- Multi-stage builds (frontend Dockerfile)
- Alpine images (smaller footprint)
- `WORKDIR`
- `COPY`
- `RUN`
- `CMD`
- `ARG` vs `ENV`
- `EXPOSE`
- `.dockerignore` file (exclude files from build context)
- `npm ci` vs `npm install` (CI-optimized, deterministic)
- `--omit=dev` (skip devDependencies in production)

### Docker Compose Concepts
- `services` — define multiple containers
- `depends_on` — startup ordering
- `condition: service_healthy` — wait for healthcheck
- `condition: service_completed_successfully` — wait for one-shot service
- `healthcheck` — command, interval, timeout, retries
- `volumes` — persistent data (`pgdata`)
- Named volumes
- `ports` — host:container port mapping
- `environment` — container env vars
- `build` — build from Dockerfile
- `command` — override default CMD
- `--scale worker=N` — horizontal scaling

### Services in docker-compose.yml
- `postgres` — PostgreSQL 16 Alpine
- `redis` — Redis 7 Alpine
- `migrate` — one-shot migration service (exits after completion)
- `api` — Express API server (port 3000)
- `worker` — worker process (can be scaled)
- `frontend` — Vite build served by Nginx (port 5173→80)

### Frontend Multi-Stage Build
- Stage 1: `node:20-alpine` — install deps, run `vite build`
- Stage 2: `nginx:alpine` — serve static `dist/` files
- `ARG VITE_API_URL` — build-time variable (Vite inlines env vars at build time)
- Why build arg not runtime env (Vite replaces `import.meta.env.VITE_*` at build time)

---

## 20. CI/CD — GitHub Actions

### Concepts
- Continuous Integration
- GitHub Actions workflow syntax
- `on: push/pull_request` triggers
- `jobs` — parallel execution by default
- `runs-on: ubuntu-latest`
- `services` — sidecar containers (Postgres, Redis)
- Service container health checks
- `actions/checkout@v4` — clone repo
- `actions/setup-node@v4` — install Node.js
- `cache: npm` — cache node_modules between runs
- `cache-dependency-path` — point to correct package-lock.json
- `working-directory` — run commands in subdirectory

### CI Pipeline
- **backend-test job**: install → migrate → `npm test` (against real Postgres + Redis)
- **frontend-build job**: install → `npm run lint` → `npm run build`
- Both jobs run on every push to `main` and every PR to `main`

---

## 21. Deployment Architecture

### Platforms
- Render — Web Service (API + inline worker), Static Site (frontend)
- Neon — Managed PostgreSQL (permanent free tier)
- Redis Cloud / Render Key Value — Managed Redis
- Vercel — Alternative for frontend static hosting

### Concepts
- `RUN_WORKER_INLINE=true` — run worker in API process (single-process deployment)
- Why: Render free tier only supports Web Services, not Background Workers
- Tradeoff: worker pauses during free-tier inactivity spin-down
- `Procfile` — Heroku/Render process type declarations (`web:`, `worker:`)
- Separation of API and Worker as distinct processes in production
- Neon vs Render Postgres — Neon doesn't hard-delete after 30 days
- Environment variables management across services

---

## 22. Node.js Core Concepts

- Event Loop
- Non-blocking I/O
- Async/Await
- Promises
- `setTimeout` and the event loop (non-blocking delay)
- `process.env` — environment variables
- `process.exit(0)` / `process.exit(1)` — clean/error exit
- `require()` — CommonJS module system
- `module.exports`
- `require.main === module` — detect if file is run directly
- `require('crypto').randomUUID` — built-in UUID generation
- `JSON.parse()` / `JSON.stringify()`
- `Math.random()` — pseudo-random number generation
- `console.log()` / `console.error()`
- `try/catch` error handling with async/await
- `Promise.all()` — concurrent async operations (demo burst)
- `Array.from()` — create arrays from iterables
- `Array.prototype.flatMap()` — map + flatten
- CommonJS vs ES Modules (`require` vs `import`)

---

## 23. Express.js Concepts

- `express()` — create app instance
- Middleware chain
- `app.use()` — register middleware
- `app.get()`, `app.post()`, `app.delete()` — route handlers
- Request object (`req.body`, `req.params`, `req.query`)
- Response object (`res.json()`, `res.send()`, `res.status()`)
- Route parameters (`:id`)
- HTTP status codes (200, 400, 404, 500)
- Content-Type: application/json
- `app.listen(PORT, callback)` — start server
- Separating app (app.js) from server (index.js) — testability pattern

---

## 24. REST API Design Concepts

- RESTful Endpoints
- HTTP Methods (GET, POST, DELETE)
- Resource-based URLs (`/jobs`, `/jobs/:id`)
- Status Codes (200 OK, 400 Bad Request, 404 Not Found, 500 Internal Server Error)
- Request/Response JSON format
- Input validation
- Error responses
- Idempotent operations
- CRUD operations mapping
- Query parameters vs Path parameters
- Pagination (LIMIT 200)

---

## 25. Vite — Build Tool Concepts

- Vite as a dev server and bundler
- ES Module-based dev server (fast HMR)
- Rollup-based production build
- `import.meta.env.VITE_*` — environment variables (build-time injection)
- `vite.config.js` — configuration file
- `defineConfig()` — type-safe config helper
- `@vitejs/plugin-react` — React support
- `npm run dev` — development server
- `npm run build` — production build
- `npm run preview` — preview production build
- `npm run lint` — run ESLint

---

## 26. Security Concepts in This Project

- Input validation (type, priority enums)
- UUID format validation (regex)
- Parameterized SQL queries (prevent SQL injection)
- CORS middleware (controlled cross-origin access)
- `.env` files (secrets not committed to git)
- `.gitignore` / `.dockerignore` (exclude sensitive files)
- SSL for PostgreSQL in production
- No raw user input in SQL queries
- `window.confirm()` before destructive actions (purge)

---

## 27. Design Patterns Used

- Producer-Consumer Pattern
- Singleton Pattern (Redis client, Postgres pool)
- Module Pattern (CommonJS exports)
- Separation of Concerns (app.js vs index.js vs worker.js vs db.js vs redis.js vs queues.js)
- State Machine (job lifecycle)
- Observer Pattern (activity feed diffs)
- Atomic Operations (WHERE status = 'queued' claim)
- Graceful Degradation (crash recovery, orphan re-queue)
- Lazy Initialization (Redis getClient)
- Strategy Pattern (queueKeyForPriority maps priority to queue)
- Component-based Architecture (React components)

---

## 28. Concurrency & Scaling Concepts

- Horizontal scaling with `docker compose up --scale worker=N`
- BRPOP atomic pop — multiple workers safely share the same queues
- No code changes needed for multi-worker setup
- Race condition prevention (`UPDATE WHERE status = 'queued'`)
- Connection pooling (pg.Pool for multiple DB connections)
- `Promise.all()` for concurrent requests (demo burst)
- Non-blocking backoff (worker keeps processing during setTimeout)
- Blocking vs polling trade-off (BRPOP vs rPop + sleep)
- Upstash serverless Redis compatibility (polling fallback)

---

## 29. Networking & Protocols

- HTTP/HTTPS
- TCP (underlying Redis and Postgres connections)
- WebSocket (mentioned as future improvement, currently polling)
- CORS (Cross-Origin Resource Sharing)
- DNS resolution in Docker Compose (service names as hostnames)
- Port mapping (host:container)
- `localhost` vs container service names (e.g., `postgres:5432` inside Docker, `localhost:5432` outside)

---

## 30. Environment Configuration

### Environment Variables
- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `PORT` — API server port (default 3000)
- `NODE_ENV` — development/production/test
- `RUN_WORKER_INLINE` — true/false (inline worker mode)
- `API_URL` — used by simulate.js (default http://localhost:3000)
- `VITE_API_URL` — frontend build-time API URL

### dotenv
- `.env` file format
- `dotenv.config()` — load into `process.env`
- `.env.example` — template for required variables (committed to git)
- `.env` in `.gitignore` (never committed)

---

## 31. File-by-File Module Map

### Backend Files
- `index.js` — entry point, starts Express server + optional inline worker
- `app.js` — Express app definition, all routes and middleware
- `worker.js` — worker process, job processing loop, retry logic, crash recovery
- `db.js` — PostgreSQL connection pool
- `redis.js` — Redis client singleton
- `queues.js` — shared constants (job types, priorities, queue key mapping)
- `migrate.js` — database migration script
- `simulate.js` — load testing / demo script
- `package.json` — dependencies and scripts
- `jest.config.js` — test configuration
- `Dockerfile` — backend Docker image
- `.dockerignore` — excluded files from Docker build
- `.env.example` — environment variable template
- `Procfile` — Heroku/Render process declarations
- `__tests__/api.test.js` — integration test suite

### Frontend Files
- `index.html` — HTML entry point
- `src/main.jsx` — React entry point (createRoot, StrictMode)
- `src/App.jsx` — main dashboard component
- `src/App.css` — all dashboard styles
- `src/index.css` — global/reset styles
- `package.json` — dependencies and scripts
- `vite.config.js` — Vite configuration
- `eslint.config.js` — ESLint flat config
- `Dockerfile` — multi-stage frontend Docker image
- `.dockerignore` — excluded files from Docker build
- `.gitignore` — excluded files from git

### Root Files
- `docker-compose.yml` — multi-service orchestration
- `.gitignore` — root-level git exclusions
- `README.md` — project documentation
- `LICENSE` — MIT License

---

## 32. npm Scripts

### Backend
- `npm start` — `node index.js`
- `npm run dev` — `nodemon index.js` (auto-restart)
- `npm run worker` — `node worker.js`
- `npm run worker:dev` — `nodemon worker.js`
- `npm run migrate` — `node migrate.js`
- `npm run simulate` — `node simulate.js`
- `npm test` — `jest --runInBand`

### Frontend
- `npm run dev` — `vite` (dev server)
- `npm run build` — `vite build` (production)
- `npm run lint` — `eslint .`
- `npm run preview` — `vite preview`

---

## 33. Interview Questions You Should Be Able to Answer

### Architecture
- Why did you choose Redis over RabbitMQ / Kafka / SQS?
- Why use PostgreSQL alongside Redis? Why not just Redis?
- What happens if Redis goes down? How do you recover?
- What happens if a worker crashes mid-processing?
- How does your system guarantee a job isn't processed twice?
- How would you scale this to handle millions of jobs per day?
- What are the limitations of your current architecture?
- How would you add exactly-once delivery semantics?
- Why not use BullMQ? What did you learn by building from scratch?
- How would you add scheduled/delayed jobs (cron-style)?

### Redis-Specific
- Explain LPUSH and BRPOP — how do they work together as a queue?
- Why is BRPOP better than polling (RPOP in a loop)?
- How does BRPOP implement priority queues?
- What guarantees does Redis give for atomic operations?
- What happens to in-flight jobs if Redis restarts?
- Why do you need two Redis clients (duplicate)?
- What's the difference between Redis Lists, Streams, and Pub/Sub for queuing?

### PostgreSQL-Specific
- Why use JSONB for the payload column?
- What indexes did you create and why?
- Explain the atomic claim query (UPDATE WHERE status = 'queued' RETURNING id)
- Why parameterized queries instead of string interpolation?
- What is connection pooling and why does it matter?
- How does TRUNCATE differ from DELETE?
- What does the FILTER clause do in aggregate functions?

### Retry Logic
- Explain your exponential backoff formula
- Why is setTimeout non-blocking? What happens in the event loop?
- What's the limitation of setTimeout-based backoff?
- How would you make retries survive a worker restart?
- What is a dead-letter queue and why do you need one?
- How would you add idempotency keys for safe retries?

### Docker / DevOps
- Walk me through your docker-compose.yml
- What is a multi-stage Docker build and why did you use one?
- How does service dependency ordering work in Docker Compose?
- What does `--scale worker=3` do and why is it safe?
- How does the migration service work as a one-shot container?
- What's in your .dockerignore and why?

### Frontend
- How does the dashboard get real-time data?
- Why polling instead of WebSockets? What would you change?
- How do you build the activity feed from job diffs?
- What React hooks did you use and why?
- How does the pipeline view sort jobs?
- What is useRef used for here? Why not useState?

### Testing
- Why integration tests instead of unit tests?
- What does supertest do differently from axios?
- How do you set up test infrastructure (Postgres/Redis)?
- What does `--runInBand` do in Jest?

### Deployment
- How did you deploy this for free?
- What is RUN_WORKER_INLINE and why does it exist?
- Why did you choose Neon over Render's Postgres?
- What is a Procfile?

---

## 34. Real-World Analogies & Use Cases

- Email sending (Mailgun, SendGrid) — queue emails, retry on SMTP failure
- SMS delivery — queue messages, backoff on carrier rate limits
- Image processing — thumbnail generation, resize, watermark
- PDF generation — invoices, reports
- Payment processing — retry failed charges with exponential backoff
- Ride-hailing dispatch — queue ride requests, match drivers
- Order confirmation — e-commerce order pipeline
- Analytics event ingestion — buffer events, batch process
- Webhook delivery — retry on endpoint timeout
- Video transcoding — long-running background jobs

---

## 35. Concepts to Know Beyond This Project

- Message Brokers: RabbitMQ, Apache Kafka, AWS SQS, Google Pub/Sub
- Redis Sorted Sets for delayed queues (`ZADD`, `ZRANGEBYSCORE`)
- Redis Streams (`XADD`, `XREADGROUP`) — consumer groups, acknowledgment
- WebSockets for real-time push (replace polling)
- Worker Threads in Node.js
- Cluster module in Node.js
- Rate Limiting / Throttling
- Circuit Breaker Pattern
- Saga Pattern (distributed transactions)
- CAP Theorem
- Eventual Consistency
- Database Transactions (BEGIN/COMMIT/ROLLBACK)
- Database Locking (row-level, advisory locks)
- Observability (logging, metrics, tracing)
- Prometheus + Grafana for queue monitoring
- Health checks and readiness probes
- Kubernetes for container orchestration (next step beyond Docker Compose)
- Load Balancing
- Reverse Proxy (Nginx — used in frontend Dockerfile)

---

## 36. Git / Version Control Concepts

- `.gitignore` — exclude files from tracking
- GitHub Actions — CI/CD automation
- Branch protection rules
- Pull Request workflow
- MIT License
- README badges (CI status, license, Node version)

---

## 37. Key Code Patterns to Memorize

```
// Atomic job claim — prevents double-processing
UPDATE jobs SET status = 'processing' WHERE id = $1 AND status = 'queued' RETURNING id

// Exponential backoff
const backoffMs = 2000 * 2 ** retry_count;  // 2s, 4s, 8s

// Priority queue with BRPOP
BRPOP jobQueue:high jobQueue jobQueue:low 0

// Non-blocking requeue
setTimeout(() => redis.lPush(queue, data), delayMs);

// Singleton lazy init
let client;
const getClient = async () => { if (client) return client; ... }

// Detect direct execution
if (require.main === module) { startWorker(); }

// Weighted random selection
const r = Math.random();
let cumulative = 0;
for (const [val, weight] of weights) { cumulative += weight; if (r <= cumulative) return val; }
```

---

## 38. Performance & Optimization Topics

- `BRPOP` vs polling — zero CPU usage while waiting
- `LPUSH` is O(1) — constant time regardless of list size
- Connection pooling — avoid per-request connection overhead
- Database indexes — O(log n) lookups on status and created_at
- `LIMIT 200` — prevent unbounded query results
- `npm ci --omit=dev` — smaller production Docker images
- Alpine base images — minimal container size
- Multi-stage Docker build — final image only has compiled output + nginx
- `--runInBand` in tests — prevents parallel test interference

---

_Use this as your checklist. Search each topic on Google / YouTube. Every line here maps to something you used in the project and could be asked about in an interview._
