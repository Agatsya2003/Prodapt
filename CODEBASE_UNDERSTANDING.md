# GPU Lease — Developer Codebase Reference

> This document provides developer-level understanding of every API endpoint, the GPU access mechanism, the database schema, and all inter-component flows. Read this alongside the source files — references like `routes.py:143` point to exact line numbers.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Startup Sequence](#3-startup-sequence)
4. [Owner Identification](#4-owner-identification)
5. [How GPU Access Works](#5-how-gpu-access-works)
6. [Reservation System](#6-reservation-system)
7. [API Endpoints — Full Reference](#7-api-endpoints--full-reference)
8. [Sync vs Async Request Flows](#8-sync-vs-async-request-flows)
9. [Background Worker](#9-background-worker)
10. [Database Schema](#10-database-schema)
11. [Wait Time Calculation](#11-wait-time-calculation)
12. [Configuration Reference](#12-configuration-reference)
13. [Non-Core Files](#13-non-core-files)

---

## 1. Project Overview

**GPU Lease** is a shared-GPU access broker for [Ollama](https://ollama.com/) — an LLM inference server that runs on a GPU-equipped host.

### Problem it solves

A GPU can only process one LLM request at a time. Multiple teams/users sending concurrent requests would cause unpredictable failures or severely degraded throughput. GPU Lease sits in front of Ollama and:

- Enforces **exclusive access** via a database-backed distributed mutex.
- Provides a **synchronous proxy** for interactive use (fails fast if GPU is busy).
- Provides an **asynchronous queue** for batch use (waits until GPU is free).
- Supports **time-based reservations** so teams can claim dedicated GPU windows.
- Logs all **job metrics and events** for observability and chargeback.

### Tech stack

| Layer | Technology |
|-------|-----------|
| Web framework | FastAPI (ASGI) |
| ASGI server | Uvicorn |
| HTTP client (proxy) | httpx.AsyncClient |
| Database | MySQL 8 via PyMySQL |
| Data validation | Pydantic v2 |
| LLM backend | Ollama |

---

## 2. Repository Structure

```
GPU_Lease/
├── app.py          Entry point — FastAPI setup + startup hook
├── routes.py       All HTTP route handlers
├── auth.py         API key auth — generation, hashing, FastAPI dependency, permission checks
├── db.py           Database layer — all SQL + lease logic + auth key lookups
├── worker.py       Background async job processor (daemon thread)
├── init_db.py      One-time DB schema initializer (run before first start)
├── schemas.py      Pydantic models for request/response validation
├── issue_key.py    CLI tool to bootstrap the first ADMIN key and issue subsequent keys
├── wrapper.py      Usage documentation / template (not executable)
├── 12.py           Experimental AutoGen multi-agent snippet (unused)
├── test_res.py     Ad-hoc testing utility for reservations
├── .env            Environment variable configuration
├── requirements.txt Python dependencies
└── backend.log     Runtime log output
```

**Core files:** `app.py`, `routes.py`, `auth.py`, `db.py`, `worker.py`
**Run once:** `init_db.py`, `issue_key.py` (for first ADMIN key bootstrap)
**Reference only:** `wrapper.py`, `12.py`, `test_res.py`

---

## 3. Startup Sequence

```
python app.py
  │
  ├─ _configure_logging()          # reads LOG_LEVEL env var (app.py:8)
  ├─ FastAPI app created           # app.py:19
  ├─ router included               # app.py:20 — mounts all routes from routes.py
  │
  └─ uvicorn.run("app:app", ...)   # app.py:37
       │
       └─ ASGI startup event fires
            └─ on_startup()        # app.py:24
                 └─ start_worker() # worker.py:38
                      └─ Creates new asyncio event loop in a daemon thread
                           └─ worker_loop() runs indefinitely (polls every 2s)
```

**Implication:** The background worker is always running as long as the server is up. It shares the MySQL connection pool with the main thread but has its own asyncio event loop and httpx client session.

---

## 4. Owner Identification & Authentication

### Authentication

Every endpoint except `GET /healthz` requires an **`X-API-Key`** header. The dependency `get_api_key()` in `auth.py` validates the key on every request:

```
Incoming request
  │
  ├─ No X-API-Key header → 401
  ├─ SHA-256(key) not found in api_keys or is_active=0 → 401
  ├─ expires_at < NOW() → 401
  └─ ✅ Injects auth context into request.state, returns key record
```

After validation, `require_route_permission(key, route_group)` checks that the key type is allowed for the route. Returns 403 if not.

### Owner derivation

The `owner` string written to `gpu_jobs`, `gpu_reservations`, and `gpu_job_events` is derived exclusively from the **API key record** in `auth.py` — not from HTTP headers:

```python
# auth.py lines 113–121
team = record["team"]
user = record["user"]
if team and user:
    owner = f"{team}-{user}"    # e.g. "ml-alice"
elif team:
    owner = team                # e.g. "ml"
else:
    owner = f"general-{record['key_prefix']}"  # ASYNC_GENERAL / ADMIN without team
```

| Key type | team | user | Resulting owner |
|----------|------|------|----------------|
| `SYNC_SPECIFIC` | `ml` | `alice` | `ml-alice` |
| `ASYNC_SPECIFIC` | `ml` | `alice` | `ml-alice` |
| `ASYNC_GENERAL` | — | — | `general-glk_asg_…` |
| `ADMIN` (with team) | `ops` | — | `ops` |

This is set on `request.state.auth_owner` and accessed in route handlers as `owner = request.state.auth_owner`. The `X-Team` / `X-User` headers are no longer used for owner derivation.

---

## 5. How GPU Access Works

### The mutex: `gpu_lease` table

The GPU is represented by a **single row** (id=1) in the `gpu_lease` table. This row acts as a distributed mutex — acquiring the GPU means UPDATE-ing this row under a MySQL row-level lock.

```sql
-- gpu_lease table structure
id           TINYINT PRIMARY KEY   -- always 1 (singleton)
owner        VARCHAR(255)          -- current holder, NULL when free
active       TINYINT               -- 1 = held, 0 = free/cooldown
reserved_at  DATETIME              -- when the current lease started
reserved_until DATETIME            -- when the lease expires (IST)
```

### Acquisition: `db_try_reserve_gpu()` — `db.py:210`

```
1. BEGIN transaction
2. SELECT owner, active, reserved_until FROM gpu_lease WHERE id=1 FOR UPDATE
   └─ MySQL row-level lock: only one caller can be here at a time
3. If no row exists → INSERT (first ever call) → COMMIT → return (True, 0, until)
4. If active=0 OR reserved_until <= now → GPU is free
   └─ UPDATE: set owner, active=1, reserved_at=now, reserved_until=now+lease_seconds
   └─ COMMIT → return (True, 0, until)
5. If active=1 AND reserved_until > now → GPU is busy
   └─ ROLLBACK → return (False, retry_after_seconds, reserved_until)
```

**Key detail:** The `FOR UPDATE` lock ensures no two concurrent callers can both see the row as "free" simultaneously. MySQL releases the lock on COMMIT or ROLLBACK.

### Release: `db_release_gpu()` — `db.py:256`

```python
# Sets active=0 but keeps reserved_until = now + cooldown_seconds
# This means the "free" slot is still blocked for the cooldown window
UPDATE gpu_lease SET owner=NULL, active=0, reserved_at=NULL, reserved_until=%s WHERE id=1
```

During cooldown, `active=0` but `reserved_until` is still in the future, so `db_try_reserve_gpu()` treats it as occupied (condition at `db.py:236` requires both `active==0` AND `current_until <= now`).

### Lease duration

```python
# routes.py:44  (identical logic in worker.py:24)
def _get_lease_seconds(owner):
    default_s = int(os.getenv("LEASE_SECONDS_DEFAULT", "60"))
    mapping_raw = os.getenv("LEASE_SECONDS_BY_OWNER", "")
    # If LEASE_SECONDS_BY_OWNER={"ml-alice":300,"ml":120}
    # then owner "ml-alice" gets 300s, "ml" gets 120s, others get default
    return int(mapping.get(owner, default_s))
```

---

## 6. Reservation System

Reservations let a team pre-book exclusive GPU access for a date/time window.

### Storage: `gpu_reservations` table

```sql
id          INT AUTO_INCREMENT PRIMARY KEY
owner       VARCHAR(255)         -- reservation holder
start_date  DATE                 -- window start date (IST)
end_date    DATE                 -- window end date (IST)
start_time  TIME                 -- daily window start (IST)
end_time    TIME                 -- daily window end (IST)
UNIQUE KEY uniq_window (start_date, end_date, start_time, end_time)
```

A reservation covers the time range `[start_date+start_time, end_date+end_time]` in IST.

### Conflict check on insert — `db.py:161`

Before inserting, the code checks for any overlapping existing reservation:

```sql
SELECT 1 FROM gpu_reservations
WHERE NOT (end_date < new_start_date OR start_date > new_end_date)   -- date overlap
  AND NOT (end_time <= new_start_time OR start_time >= new_end_time)  -- time overlap
LIMIT 1
```

If an overlap exists, returns `False` → route returns HTTP 409.

### Enforcement on every request — `db.py:187`

Both the sync proxy and the async worker call `db_get_active_reservation()` before attempting GPU acquisition:

```sql
SELECT owner, start_time, end_time
FROM gpu_reservations
WHERE now_ist_date BETWEEN start_date AND end_date
  AND now_ist_time BETWEEN start_time AND end_time
LIMIT 1
```

- If a reservation is active AND the requester's owner != reservation owner → **HTTP 429** with `Retry-After: 60`.
- The async worker **skips** the job (sleeps 5s and retries) instead of failing it.

**Timezone note:** All reservation comparisons use IST (Asia/Kolkata) regardless of system timezone.

---

## 7. API Endpoints — Full Reference

Base URL: `http://<HOST>:<PORT>` (default `http://0.0.0.0:8000`)

---

### 7.1 Sync GPU Proxy

```
ANY /ollama/{path}
```

**Methods:** GET, POST, PUT, PATCH, DELETE
**File:** `routes.py:246`
**Auth required:** `SYNC_SPECIFIC` or `ADMIN` key (`X-API-Key` header)

**Request headers:**

| Header | Required | Description |
|--------|----------|-------------|
| `X-API-Key` | **Yes** | API key — must be `SYNC_SPECIFIC` or `ADMIN` type |
| `X-Estimated-MS` | Optional | Hint for queue wait time calculation |

**Request body:** Passed through as-is to Ollama. Must be valid JSON if model selection is needed (key `model` is extracted for logging).

**Flow:**

```
0. get_api_key() validates X-API-Key; owner = request.state.auth_owner
1. require_route_permission(key, "sync") — 403 if wrong key type
2. db_get_active_reservation() — check for active time reservation
   └─ If active and not owner → 429 {"detail": "GPU reserved for X between A and B IST"}
3. db_try_reserve_gpu(owner, lease_seconds) — acquire mutex
   └─ If busy → 429 {"detail": "GPU is reserved. Try again after Ns"}
              headers: Retry-After: N
4. Record acquired_dt; compute queue_wait_ms = acquired_dt - request_start
5. db_insert_job(job_id, owner, model, ...) — create job record (status=QUEUED)
6. db_update_job_running(job_id, acquired_dt) — status=RUNNING
7. db_insert_event(job_id, "GPU_ACQUIRED", details="queue_wait_ms=N")
8. httpx.AsyncClient.request(method, OLLAMA_BASE_URL/path, body, headers, params)
   timeout=None — waits as long as Ollama needs
9. db_update_job_done(job_id, status, metrics...)
10. db_insert_event(job_id, "GPU_RELEASED")
11. db_release_gpu(cooldown_seconds)   [always in finally block]
12. Return proxied response (status, headers, body) verbatim
```

**Error handling:**

| Scenario | HTTP Status | Body |
|----------|-------------|------|
| Active reservation, wrong owner | 429 | `{"detail": "GPU reserved for ... between ... IST."}` |
| GPU busy (lease held) | 429 | `{"detail": "GPU is reserved. Try again after Ns ..."}` |
| Ollama returns non-2xx | job status=FAILED, proxied upstream response | Ollama's error body |
| Exception during proxy | 500 (re-raised) | FastAPI default error format |

---

### 7.2 Async Enqueue

```
POST /async/ollama/{path}
```

**Methods:** GET, POST, PUT, PATCH, DELETE
**File:** `routes.py:402`
**Auth required:** `ASYNC_SPECIFIC`, `ASYNC_GENERAL`, or `ADMIN` key (`X-API-Key` header)

**Request headers:**

| Header | Required | Description |
|--------|----------|-------------|
| `X-API-Key` | **Yes** | API key — must be `ASYNC_SPECIFIC`, `ASYNC_GENERAL`, or `ADMIN` type |
| `X-Estimated-MS` | Optional | Caller's estimate of run time in ms. Defaults to `LEASE_SECONDS_DEFAULT * 1000` if absent. |

**Flow:**

```
0. get_api_key() validates X-API-Key; owner = request.state.auth_owner
1. require_route_permission(key, "async") — 403 if wrong key type
2. Generate job_id (UUID4)
3. Read full request body (stored for replay)
4. db_insert_job(..., req_method, req_path, req_body) — status=QUEUED
   └─ The entire raw request body is stored as LONGBLOB for later replay
5. db_insert_event(job_id, "JOB_ENQUEUED")
6. db_calculate_wait_time(job_id) — estimate queue position
7. Return immediately:
```

**Response (HTTP 200):**

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "QUEUED",
  "expected_wait_time_ms": 12000
}
```

**No reservation check here.** The job is queued; the worker enforces reservation checks when it picks up the job.

---

### 7.3 Async Job Status Poll

```
GET /async/jobs/{job_id}
```

**File:** `routes.py:460`
**Auth required:** `ASYNC_SPECIFIC`, `ASYNC_GENERAL`, or `ADMIN` key

**Flow:**

```
0. get_api_key() + require_route_permission(key, "async_poll")
1. db_get_job_status(job_id) → (status, estimated_ms, created_at)
2. If not found → 404
3. Ownership guard (non-ADMIN only):
   SELECT owner FROM gpu_jobs WHERE job_id = ?
   If job_owner != request.state.auth_owner → 404 (not 403 — avoids leaking job existence)
4. If DONE:
   └─ db_get_result(job_id) → (content_type, content)
   └─ Return Response(content=content, media_type=content_type)
      [same binary blob that Ollama returned, with original content-type]
4. If FAILED:
   └─ {"status": "FAILED", "message": "Job failed"}
5. If QUEUED or RUNNING:
   └─ db_calculate_wait_time(job_id) → wait_ms
   └─ {"job_id": ..., "status": ..., "expected_wait_time_ms": wait_ms}
```

**Polling pattern:** Clients should poll this endpoint until status is `DONE` or `FAILED`. There is no webhook/push notification.

---

### 7.4 Observability: Create Job Record

```
POST /jobs
```

**File:** `routes.py:94`
**Auth required:** `ADMIN` key
**Purpose:** Allows external callers (e.g., other scripts) to manually register a job record in the DB without going through the proxy.

**Request body (JSON):**

```json
{
  "job_id": "string",
  "owner": "string",
  "model_name": "string",
  "request_bytes": 1024,
  "estimated_ms": 5000
}
```

**Response:** `{"ok": true}`

---

### 7.5 Observability: Mark Job Running

```
POST /jobs/{job_id}/running
```

**File:** `routes.py:121`
**Auth required:** `ADMIN` key

**Request body (JSON):**

```json
{
  "started_at": "2024-01-01T10:00:00"   // optional, defaults to now (IST)
}
```

Sets job status to `RUNNING` and records `started_at`.

---

### 7.6 Observability: Mark Job Done

```
POST /jobs/{job_id}/done
```

**File:** `routes.py:142`
**Auth required:** `ADMIN` key

**Request body (JSON):**

```json
{
  "status": "DONE",
  "done_at": "2024-01-01T10:01:00",
  "http_status": 200,
  "queue_wait_ms": 50,
  "run_ms": 3200,
  "total_ms": 3250,
  "response_bytes": 4096,
  "error_code": null,
  "error_text": null
}
```

All fields except `status` are optional.

---

### 7.7 Observability: Log Job Event

```
POST /jobs/{job_id}/events
```

**File:** `routes.py:175`
**Auth required:** `ADMIN` key

**Request body (JSON):**

```json
{
  "event_type": "GPU_ACQUIRED",
  "owner": "ml-alice",
  "model_name": "llama3",
  "details": "queue_wait_ms=120"
}
```

Inserts a row into `gpu_job_events`. `event_type` is a free-form string; the app itself uses: `GPU_ACQUIRED`, `GPU_RELEASED`, `JOB_ENQUEUED`, `JOB_ERROR`.

---

### 7.8 Create Reservation

```
POST /reservations
```

**File:** `routes.py:206`
**Auth required:** `ADMIN` key

**Request body (JSON):**

```json
{
  "team": "NextGenLabs",
  "user": "Alice",
  "start_date": "2026-04-01",
  "end_date": "2026-04-01",
  "start_time": "10:00:00",
  "end_time": "12:00:00"
}
```

`user` is optional — omit it to reserve for the entire team. The `owner` string stored in the DB is constructed in the handler as `"{team}-{user}"` if `user` is present, otherwise just `"{team}"`.

**Responses:**

| Condition | HTTP | Body |
|-----------|------|------|
| Success | 200 | `{"ok": true}` |
| Overlapping window | 409 | `{"detail": "Reservation window overlaps with an existing reservation."}` |
| DB error | 500 | `{"detail": "...error message..."}` |

---

### 7.9 Admin Key Management

Three endpoints for the full API key lifecycle. All require an `ADMIN` key.

#### Issue a key — `POST /admin/keys`

**File:** `routes.py:519`

**Request body (JSON):**

```json
{
  "key_type": "ASYNC_SPECIFIC",
  "team": "ml",
  "user": "alice",
  "label": "alice notebook",
  "expires_at": null
}
```

- `key_type`: one of `ADMIN`, `ASYNC_GENERAL`, `ASYNC_SPECIFIC`, `SYNC_SPECIFIC`
- `team` + `user` are required for `*_SPECIFIC` types
- `expires_at`: ISO datetime or `null` (no expiry)

**Response (200):** Returns `raw_key` **once only** — it is never stored and cannot be retrieved again.

#### List keys for a team — `GET /admin/keys/{team}`

**File:** `routes.py:575`

Returns all key records for the team. `key_hash` is stripped from the response. Raw key values are never included — only `key_prefix`, type, status, and timestamps.

#### Revoke a key — `DELETE /admin/keys/{key_id}`

**File:** `routes.py:595`

Sets `is_active = 0` for the key with the given numeric DB id. The key stops working on the **next request** — no grace period, no cache. The row is kept for audit history. Returns 404 if the key is not found or already revoked.

---

### 7.10 Health Check

```
GET /healthz
```

**File:** `app.py:122`

**Response:** `{"ok": true}` — Always returns 200. Does not check DB or Ollama connectivity.

---

## 8. Sync vs Async Request Flows

### Synchronous Flow (inline execution)

```
Client                     GPU Lease API              MySQL             Ollama
  │                              │                      │                  │
  │── ANY /ollama/{path} ───────>│                      │                  │
  │   (X-API-Key header)         │─ validate API key ──>│                  │
  │                              │<─ key record ─────────│                  │
  │                              │─ check reservation ─>│                  │
  │                              │<─ (none active) ─────│                  │
  │                              │─ FOR UPDATE row ─────>│                  │
  │                              │<─ lease acquired ─────│                  │
  │                              │─ INSERT job (QUEUED) ─>│                 │
  │                              │─ UPDATE job (RUNNING) ─>│                │
  │                              │─ INSERT GPU_ACQUIRED ──>│                │
  │                              │                      │                  │
  │                              │── request ──────────────────────────────>│
  │                              │<─ response ─────────────────────────────│
  │                              │                      │                  │
  │                              │─ UPDATE job (DONE) ───>│                 │
  │                              │─ INSERT GPU_RELEASED ──>│                │
  │                              │─ release lease ────────>│                │
  │<─ proxied response ──────────│                      │                  │
```

**Failure mode:** If GPU is busy, the sync request is **immediately rejected** with HTTP 429. The client must retry manually.

---

### Asynchronous Flow (queued execution)

```
Client                     GPU Lease API              MySQL             Ollama
  │                              │                      │                  │
  │── POST /async/ollama/{path} ─>│                      │                  │
  │   (X-API-Key header)         │─ validate API key ──>│                  │
  │                              │<─ key record ─────────│                  │
  │                              │─ INSERT job+body ────>│                  │
  │                              │─ INSERT JOB_ENQUEUED ─>│                │
  │<─ {job_id, QUEUED, wait_ms} ─│                      │                  │
  │                              │                      │                  │
  │── GET /async/jobs/{id} ─────>│                      │                  │
  │<─ {QUEUED, wait_ms} ─────────│                      │                  │
  │                              │                      │                  │
  │         [Background Worker polls every 2s]          │                  │
  │                              │─ SELECT QUEUED jobs ─>│                  │
  │                              │<─ job row ────────────│                  │
  │                              │─ FOR UPDATE lease ───>│                  │
  │                              │<─ acquired ───────────│                  │
  │                              │─ UPDATE (RUNNING) ────>│                 │
  │                              │── request ──────────────────────────────>│
  │                              │<─ response ─────────────────────────────│
  │                              │─ UPDATE (DONE) ───────>│                 │
  │                              │─ INSERT result ───────>│                 │
  │                              │─ release lease ───────>│                 │
  │                              │                      │                  │
  │── GET /async/jobs/{id} ─────>│                      │                  │
  │                              │─ SELECT result ───────>│                 │
  │<─ Response(content, type) ───│                      │                  │
```

---

## 9. Background Worker

**File:** `worker.py`
**Started by:** `app.py:24` via `start_worker()` on FastAPI startup

### Architecture

- `start_worker()` (`worker.py:38`) creates a **new asyncio event loop** in a **daemon thread**.
- A single persistent `httpx.AsyncClient(timeout=None)` is reused across all job executions (efficiency, no per-job connection setup).
- The worker dies with the main process (daemon=True).

### `worker_loop()` — iteration logic

```python
while True:
    job = db_get_next_queued_job()         # oldest QUEUED job (ORDER BY created_at ASC)

    if not job:
        await asyncio.sleep(2)             # nothing to do, poll again
        continue

    if not job.req_path:
        mark FAILED (MISSING_DATA)         # corrupted/manual job record
        continue

    res = db_get_active_reservation()
    if res and res.owner != job.owner:
        await asyncio.sleep(5)             # reserved for someone else, skip
        continue                           # NOTE: job stays QUEUED, retried next cycle

    ok, _, _ = db_try_reserve_gpu(owner, lease_seconds)
    if not ok:
        await asyncio.sleep(2)             # GPU busy, retry
        continue

    # === GPU acquired ===
    db_update_job_running(job_id, now)
    db_insert_event(job_id, "GPU_ACQUIRED")

    resp = await client.request(method, url, content=req_body)

    db_update_job_done(job_id, status, metrics)
    db_store_result(job_id, content_type, resp.content)
    db_insert_event(job_id, "GPU_RELEASED")

    db_release_gpu(cooldown_seconds)       # always, even on exception
```

### Important behaviors

- **FIFO ordering:** Jobs are processed in `created_at ASC` order. Oldest job first.
- **One job at a time:** The worker does not process multiple jobs in parallel.
- **Reservation skipping:** If a reservation blocks the current job, the worker sleeps 5s and **retries the same job** next cycle. The job stays QUEUED.
- **No request headers forwarded:** Unlike the sync proxy, the worker does not forward original HTTP headers to Ollama (only body). `routes.py:227` strips headers; `worker.py:90` sends only `content=req_body`.

---

## 10. Database Schema

All tables are in the `observability` database.

---

### `gpu_jobs`

Primary key: `job_id` (UUID string)

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | VARCHAR(255) PK | UUID4 job identifier |
| `owner` | VARCHAR(255) | Derived from API key metadata (team, user, or key prefix) |
| `model_name` | VARCHAR(255) | Extracted from request body JSON `.model` |
| `status` | VARCHAR(50) | `QUEUED` → `RUNNING` → `DONE` / `FAILED` |
| `created_at` | DATETIME | Job creation time (IST) |
| `started_at` | DATETIME | GPU acquisition time (IST) |
| `done_at` | DATETIME | Completion time (IST) |
| `request_bytes` | BIGINT | Size of incoming request body |
| `response_bytes` | BIGINT | Size of Ollama response body |
| `http_status` | INT | HTTP status code from Ollama |
| `queue_wait_ms` | INT | Milliseconds from creation to GPU acquisition |
| `run_ms` | INT | Milliseconds from GPU acquire to completion |
| `total_ms` | INT | Milliseconds from creation to completion |
| `estimated_ms` | INT | Client-provided hint via X-Estimated-MS |
| `req_method` | VARCHAR(10) | HTTP method (async jobs only) |
| `req_path` | TEXT | Ollama path (async jobs only) |
| `req_body` | LONGBLOB | Raw request body stored for replay (async only) |
| `error_code` | VARCHAR(100) | `OLLAMA_HTTP_ERROR`, `WRAPPER_EXCEPTION`, `WORKER_EXCEPTION`, `MISSING_DATA` |
| `error_text` | TEXT | Error detail (max 512 chars) |

---

### `gpu_job_events`

Append-only audit log. Multiple events per job.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT AUTO_INCREMENT PK | |
| `job_id` | VARCHAR(255) | FK to gpu_jobs (no FK constraint) |
| `event_type` | VARCHAR(100) | `GPU_ACQUIRED`, `GPU_RELEASED`, `JOB_ENQUEUED`, `JOB_ERROR` |
| `event_ts` | DATETIME | Event timestamp (IST) |
| `owner` | VARCHAR(255) | Job owner at time of event |
| `model_name` | VARCHAR(255) | Model name |
| `details` | TEXT | Free-form detail string (e.g., `queue_wait_ms=120`) |

Index on `job_id` for fast per-job event lookup.

---

### `gpu_lease`

Singleton table (always exactly 1 row, id=1). Acts as the distributed GPU mutex.

| Column | Type | Description |
|--------|------|-------------|
| `id` | TINYINT PK | Always 1 |
| `owner` | VARCHAR(255) NULL | Current lease holder; NULL when free |
| `active` | TINYINT | `1` = GPU held, `0` = free or in cooldown |
| `reserved_at` | DATETIME NULL | When current lease started |
| `reserved_until` | DATETIME NULL | When current lease/cooldown expires |

**State machine:**

```
active=0, reserved_until=NULL or past  →  GPU is free, can be acquired
active=1, reserved_until in future     →  GPU is held
active=0, reserved_until in future     →  GPU in cooldown (post-release)
```

---

### `gpu_reservations`

Pre-scheduled access windows.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT AUTO_INCREMENT PK | |
| `owner` | VARCHAR(255) | Reservation holder |
| `start_date` | DATE | Window start date (IST) |
| `end_date` | DATE | Window end date (IST) |
| `start_time` | TIME | Daily start time (IST) |
| `end_time` | TIME | Daily end time (IST) |

Unique key on `(start_date, end_date, start_time, end_time)` prevents identical windows.

---

### `gpu_job_results`

Stores completed async job responses for later retrieval.

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | VARCHAR(255) PK | FK to gpu_jobs |
| `content_type` | VARCHAR(100) | MIME type from Ollama response header |
| `content` | LONGBLOB | Raw response body bytes |
| `created_at` | DATETIME | When result was stored |

Uses `INSERT ... ON DUPLICATE KEY UPDATE` for idempotent writes (`db.py:326`).

---

### `api_keys`

Stores API key records. Raw keys are **never stored** — only the SHA-256 hash.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT AUTO_INCREMENT PK | Numeric key ID (used for revocation) |
| `key_hash` | VARCHAR(64) UNIQUE | SHA-256 hex digest of the raw key — per-request lookup hot path |
| `key_prefix` | VARCHAR(14) | First 14 chars of the raw key — safe to display |
| `key_type` | ENUM | `ADMIN`, `ASYNC_GENERAL`, `ASYNC_SPECIFIC`, `SYNC_SPECIFIC`, `SYNC_GENERAL` |
| `team` | VARCHAR(255) NULL | Team binding (NULL for general/admin keys) |
| `user` | VARCHAR(255) NULL | User binding (NULL for general/admin/team-level keys) |
| `label` | VARCHAR(255) NULL | Human-readable description |
| `is_active` | TINYINT | `1` = valid, `0` = revoked. Setting to `0` rejects the key on the next request. |
| `created_at` | DATETIME | When key was issued |
| `expires_at` | DATETIME NULL | Expiry timestamp (NULL = never expires) |
| `last_used_at` | DATETIME NULL | Updated on every successful auth (best-effort) |
| `created_by` | VARCHAR(255) NULL | Key prefix of the issuer, or `"cli"` if issued via `issue_key.py` |

**Indexes:** `idx_api_keys_hash` (per-request lookup), `idx_api_keys_prefix` (admin display), `idx_api_keys_team` (team listing).

**State machine:** `is_active=1` → active; `is_active=0` → revoked (row kept for audit).

---

### `api_key_requests`

Table exists in the DB schema with status ENUM `PENDING` / `APPROVED` / `REJECTED`. **No routes or DB functions reference it yet** — reserved for a future self-service key request / admin approval workflow.

---

## 11. Wait Time Calculation

`db_calculate_wait_time(job_id)` — `db.py:278`

Returns an estimated wait in milliseconds for a QUEUED job. Used in both enqueue and status-poll responses.

**Formula:**

```
total_wait_ms = remaining_current_job_ms + sum_queued_ahead_ms + reservation_blocking_ms
```

**Step 1 — remaining time of currently RUNNING job:**

```sql
SELECT started_at, estimated_ms FROM gpu_jobs WHERE status='RUNNING' LIMIT 1
```

`remaining_ms = max(0, estimated_ms - elapsed_since_started_ms)`

**Step 2 — sum of QUEUED jobs ahead:**

```sql
SELECT SUM(estimated_ms) FROM gpu_jobs WHERE status='QUEUED' AND created_at < this_job.created_at
```

Jobs with `NULL` estimated_ms contribute 0 to the sum.

**Step 3 — active reservation blocking:**

If there's an active reservation owned by someone else, adds remaining reservation time:

```python
end_td = reservation['end_time']   # timedelta from DB TIME column
current_td = timedelta(hours=now.hour, minutes=now.minute, ...)
res_wait_ms = (end_td - current_td).total_seconds() * 1000
```

**Caveat:** This is an estimate. Accuracy depends on clients providing realistic `X-Estimated-MS` values. If `estimated_ms` is NULL for any job, that job contributes 0ms to the estimate.

---

## 12. Configuration Reference

All values read at runtime from environment variables (`.env` file loaded by OS/shell).

| Variable | Type | Default | Effect |
|----------|------|---------|--------|
| `MYSQL_HOST` | str | `127.0.0.1` | MySQL server host |
| `MYSQL_PORT` | int | `3306` | MySQL server port |
| `MYSQL_DB` | str | `observability` | Database name |
| `MYSQL_USER` | str | `root` | MySQL username |
| `MYSQL_PASS` | str | `password` | MySQL password |
| `OLLAMA_BASE_URL` | str | `http://127.0.0.1:11434` | Ollama backend base URL |
| `LEASE_SECONDS_DEFAULT` | int | `60` | GPU lease duration in seconds for all owners |
| `LEASE_SECONDS_BY_OWNER` | JSON str | _(empty)_ | Per-owner lease overrides, e.g. `{"ml-alice":300}` |
| `COOLDOWN_SECONDS` | int | `15` (code default) | Cooldown after GPU release; `.env` sets `0` |
| `HOST` | str | `0.0.0.0` | Server bind address |
| `PORT` | int | `8000` | Server listen port |
| `LOG_LEVEL` | str | `INFO` | Logging level (`DEBUG`/`INFO`/`WARNING`/`ERROR`) |

**Note on COOLDOWN_SECONDS:** The code default (`_get_cooldown_seconds()`, `routes.py:57`) is `15`. The `.env` file in this repo sets it to `0`, meaning no cooldown is enforced in the current deployment.

**Note on timezone:** All timestamps stored in DB are in **IST (Asia/Kolkata, UTC+5:30)**. This is hardcoded in `utcnow_dt3()` at `db.py:33`. Reservation time checks also use IST.

---

## 13. Non-Core Files

### `auth.py`

Core authentication module. Contains key generation (`generate_raw_key`), hashing (`hash_key`), the FastAPI dependency (`get_api_key`), and route-level permission enforcement (`require_route_permission`). Imported by `routes.py` and `issue_key.py`.

### `issue_key.py`

Server-side CLI tool for issuing API keys without an existing key. Used to bootstrap the very first `ADMIN` key. Reads the `.env` file and writes directly to the database. Raw key is printed once and never stored.

```bash
python issue_key.py --type ADMIN --label "ops admin"
python issue_key.py --type ASYNC_SPECIFIC --team ml --user alice --label "alice notebook"
python issue_key.py --type SYNC_SPECIFIC --team ml --user bob --expires 2026-12-31
```

### `wrapper.py`

A plain-text documentation file (not executable Python). Contains usage examples showing how to call the job and event observability endpoints. Not imported anywhere.

### `12.py`

An experimental script demonstrating a multi-agent LLM conversation using **AutoGen** (`autogen` library). It defines two agents (assistant + user proxy) and has them collaborate on a task. **Not integrated** into the main application and not referenced anywhere. Requires `autogen` which is not in `requirements.txt`.

### `test_res.py`

A small ad-hoc testing script that calls `db_get_active_reservation()` directly and prints the result. Used during development to verify reservation logic. Not part of the server.
