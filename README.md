# 🎟️ Concurrent Ticket Booking Engine

![Node.js](https://img.shields.io/badge/Node.js-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)
![BullMQ](https://img.shields.io/badge/BullMQ-FF4F00?logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=black)
![Zustand](https://img.shields.io/badge/Zustand-443E38?logo=react&logoColor=white)
![Grafana k6](https://img.shields.io/badge/Grafana_k6-7D64FF?logo=k6&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-2496ED?logo=docker&logoColor=white)
![SSE](https://img.shields.io/badge/Real--Time_Updates-SSE-6B7280)

## 🎯 Objective

Design and implement a concurrent seat reservation system that guarantees **zero double-bookings** under parallel request spikes using atomic Redis Lua locks, PostgreSQL optimistic concurrency control, BullMQ delayed expiration jobs, and Server-Sent Events (SSE).

> 🎬 **[Drag and drop your 10–15s Demo MP4 / GIF here on GitHub]**

---

## 📑 Table of Contents

- [🏗️ System Architecture](#️-system-architecture)
- [🔄 Reservation Lifecycle](#-reservation-lifecycle)
- [⚙️ Key Engineering Decisions](#️-key-engineering-decisions)
- [📊 Benchmarks & Load Test Metrics](#-benchmarks--load-test-metrics)
- [🚀 Docker Quickstart (Full Stack)](#-docker-quickstart-full-stack)
- [💻 Local Development Setup](#-local-development-setup)
- [🧪 Running Tests](#-running-tests)
- [📡 API Reference](#-api-reference)
- [📂 Project Structure](#-project-structure)

---

## 🏗️ System Architecture

```mermaid
flowchart LR
    Client["React + Zustand Client"]
    API["Express API Server"]
    RedisLock[("Redis 8\nAtomic Lua Locks")]
    BullMQ[["BullMQ Worker\nDelayed Expiry Queue"]]
    Postgres[("PostgreSQL 18\nOCC Versioning")]

    Client -- "POST /hold, /confirm\n(HttpOnly JWT)" --> API
    API -- "SSE Stream (/seats/stream)" --> Client
    API -- "1. EVAL acquire/release.lua" --> RedisLock
    API -- "2. Versioned SQL TX" --> Postgres
    API -- "3. Enqueue 120s TTL Job" --> BullMQ
    BullMQ -- "4. Auto-Release Expired Hold" --> Postgres
    BullMQ -- "5. Delete Lock & Notify" --> RedisLock
```

---

## 🔄 Reservation Lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor User as React Client
    participant API as Express API
    participant Redis as Redis (Lua)
    participant DB as PostgreSQL (OCC)
    participant Queue as BullMQ Worker

    User->>API: POST /api/seats/A1/hold
    API->>Redis: EVAL acquireLock.lua (SET NX EX 120)
    Redis-->>API: 1 (Lock Acquired)
    API->>DB: UPDATE seats SET status='HELD', version=version+1 WHERE version=N
    DB-->>API: Row Updated (version = N+1)
    API->>Queue: Add delayed job (120,000ms)
    API-->>User: 200 OK + SSE Broadcast (HELD)

    alt Payment Confirmed Within 120s
        User->>API: POST /api/seats/A1/confirm (idempotencyKey)
        API->>DB: BEGIN TX -> Insert Booking + UPDATE status='BOOKED', version=version+1
        API->>Redis: EVAL releaseLock.lua (Verify owner & DEL)
        API-->>User: 200 OK + SSE Broadcast (BOOKED)
    else 120s Hold Timer Expires
        Queue->>DB: UPDATE seats SET status='AVAILABLE', version=version+1 WHERE status='HELD'
        Queue->>Redis: EVAL releaseLock.lua (Verify owner & DEL)
        Queue-->>User: SSE Broadcast (AVAILABLE)
    end
```

---

## ⚙️ Key Engineering Decisions

| Layer | Mechanism | Purpose |
| :--- | :--- | :--- |
| **Distributed Locking** | **Redis Lua Scripts** (`acquireLock.lua`, `releaseLock.lua`) | Executes lock acquisition (`SET NX EX`) and ownership-verified deletion (`GET == userId` then `DEL`) atomically on Redis's single thread, preventing race conditions and ABA lock stealing. |
| **Database Integrity** | **Optimistic Concurrency Control (OCC)** | Every mutation increments the seat's `version` column (`WHERE id = $1 AND version = $2`). Acts as a database-level safety net if the cache layer fails. |
| **Hold Expiration** | **BullMQ Delayed Jobs** | Persists 120-second expiration timers in Redis AOF rather than volatile Node.js `setTimeout` memory, surviving server restarts. |
| **Live Sync** | **Server-Sent Events (SSE)** | Pushes unidirectional seat state transitions (`AVAILABLE`, `HELD`, `BOOKED`) over HTTP with native browser auto-reconnect. |
| **Idempotency & IDs** | **`crypto.randomUUID()` & Unique Keys** | Uses UUIDv4 for entity IDs and enforces a unique `idempotency_key` constraint on payment confirmations to prevent duplicate charges on network retries. |

---

## 📊 Benchmarks & Load Test Metrics

### 1. 500-Request Burst Benchmark (`loadtest/benchmark.ts`)
500 simultaneous HTTP `POST /api/seats/A1/hold` requests fired concurrently at Seat `A1`:

| Metric | Result |
| :--- | :--- |
| **Concurrent Virtual Users** | `500` simultaneous requests |
| **Target Resource** | Single Seat (`A1`) |
| **Locks Acquired (`200 OK`)** | **`1`** |
| **Race Losers Rejected (`409 Conflict`)** | **`499`** |
| **Unexpected Server Errors (`5xx`)** | **`0`** |
| **Throughput** | **`~1,205 req/sec`** |
| **p50 / p95 / p99 Latency** | **`288.08 ms` / `372.42 ms` / `372.75 ms`** |

### 2. Grafana k6 Stress Test (`loadtest/stress.k6.js`)
200 concurrent Virtual Users (VUs) executing 300 shared iterations against Seat `A1` via Docker:

```text
     scenarios: (100.00%) 1 scenario, 200 max VUs, 45s max duration (incl. graceful stop):
              * flash_sale_spike: 300 iterations shared among 200 VUs (maxDuration: 15s)

     ✓ is 200 (winner) or 409 (conflict)

     checks_total.......: 300     1004.10961/s
     checks_succeeded...: 100.00% 300 out of 300
     checks_failed......: 0.00%   0 out of 300

     http_req_duration..............: avg=61.82ms min=22.94ms med=65.04ms max=102.53ms p(90)=89.35ms p(95)=91.76ms
       { expected_response:true }...: avg=39.53ms min=22.94ms med=27.38ms max=68.26ms  p(90)=60.08ms p(95)=64.17ms
     http_req_failed................: 99.00% 299 out of 302
     http_reqs......................: 302    1010.803674/s
```

* **`302` Total Requests:** `2` setup requests (`POST /seats/reset` for test isolation and `POST /auth/login` for JWT cookie retrieval) + `300` concurrent hold requests.
* **`299` Rejected (`409 Conflict`):** k6 counts `4xx` responses under `http_req_failed` by default. `1` request acquired the seat (`200 OK`) and `299` were rejected (`409 Conflict`), achieving `100%` check success (`300/300`).

---

## 🚀 Docker Quickstart (Full Stack)

Run PostgreSQL, Redis, the Express API + BullMQ worker, and the React client in a single command:

```bash
git clone [https://github.com/](https://github.com/)<your-username>/concurrent-ticket-engine.git
cd concurrent-ticket-engine
docker compose up --build -d
```

* **Frontend UI:** `http://localhost:5173`
* **Backend API:** `http://localhost:4000/health`

**Pre-Seeded Test Accounts** (or register a new user via the UI):
* `user_alpha@test.com` / `password123`
* `user_beta@test.com` / `password123`
* `user_gamma@test.com` / `password123`

---

## 💻 Local Development Setup

### 1. Start PostgreSQL & Redis
```bash
docker compose -f docker-compose.dev.yaml up -d
```

### 2. Configure & Start Backend (`server/`)
Create `server/.env`:
```env
NODE_ENV=development
PORT=4000
CLIENT_URL=http://localhost:5173
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/concurrent_ticket_engine
REDIS_URL=redis://localhost:6379
JWT_SECRET=super_secret_jwt_signing_key_2026
LOAD_TEST_SECRET=benchmark_secret_bypass
HOLD_TTL_SECONDS=120
```

```bash
cd server
npm install
npm run dev
```

### 3. Start Frontend (`client/`)
```bash
cd client
npm install
npm run dev
```

---

## 🧪 Running Tests

```bash
# 1. Redis Lua atomic lock unit tests (Vitest)
cd server && npm test

# 2. 500-request concurrent burst benchmark
cd loadtest && npm install && npm run benchmark

# 3. Containerized Grafana k6 stress test (no local k6 install required)
cd loadtest && npm run stress:docker
```

---

## 📡 API Reference

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/register` | Public | Registers user and sets `HttpOnly` JWT cookie |
| `POST` | `/api/auth/login` | Public | Authenticates user and sets `HttpOnly` JWT cookie |
| `POST` | `/api/auth/logout` | Public | Clears session cookie |
| `GET` | `/api/auth/me` | Protected | Returns active session user |
| `GET` | `/api/seats` | Public | Returns all 50 auditorium seats (`A1`–`E10`) |
| `GET` | `/api/seats/stream` | Public | SSE endpoint for real-time seat status updates |
| `POST` | `/api/seats/:id/hold` | Protected | Acquires 120s Redis lock, increments DB version, queues BullMQ expiry job |
| `POST` | `/api/seats/:id/confirm` | Protected | Commits booking (`SUCCESS`) or releases hold (`FAIL`) with idempotency check |
| `POST` | `/api/seats/reset` | Dev/Test | Resets seats to `AVAILABLE` and flushes Redis locks |

---

## 📂 Project Structure

```text
concurrent-ticket-engine/
├── docker-compose.yaml          # Full-stack containerized setup
├── docker-compose.dev.yaml      # Postgres, Redis & ephemeral k6 runner
├── server/
│   ├── Dockerfile
│   ├── src/
│   │   ├── config/              # DB pool, Redis client, env schema
│   │   ├── redis/               # Atomic Lua scripts (acquireLock, releaseLock)
│   │   ├── queues/              # BullMQ delayed expiration worker
│   │   ├── services/            # SSE client manager
│   │   ├── controllers/         # Auth & Seat controllers
│   │   ├── middleware/          # JWT auth & rate limiter
│   │   └── scripts/             # Auto-schema & seed runner
│   └── tests/                   # Vitest unit tests
├── client/
│   ├── Dockerfile
│   └── src/
│       ├── api/                 # Fetch client
│       ├── store/               # Zustand stores (auth, seat)
│       ├── components/          # ProtectedRoute, PublicRoute, SeatGrid, CheckoutCard
│       └── pages/               # LoginPage, RegisterPage, BookingPage
└── loadtest/
    ├── benchmark.ts             # 500-request burst benchmark
    └── stress.k6.js             # Grafana k6 load test
```
