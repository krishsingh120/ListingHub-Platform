# OLX API Project Revision Guide & Interview Masterclass

This revision guide is specifically designed to help you master, revise, and defend every engineering decision in this codebase during technical interviews (System Design, Backend Engineering, Go Core, Distributed Systems).

---

## 📑 Table of Contents
1. [Core Architectural Blueprint](#1-core-architectural-blueprint)
2. [Deep Dives into Engineering Implementations](#2-deep-dives-into-engineering-implementations)
   - [Transactional Outbox Pattern & SKIP LOCKED](#21-transactional-outbox-pattern--skip-locked)
   - [Cursor / Keyset Pagination](#22-cursor--keyset-pagination)
   - [Presigned Object Storage & Validation Pipeline](#23-presigned-object-storage--validation-pipeline)
   - [Thread-Safe Token Bucket Rate Limiter](#24-thread-safe-token-bucket-rate-limiter)
   - [Timing Attack Mitigation in Authentication](#25-timing-attack-mitigation-in-authentication)
   - [Graceful Shutdown Lifecycle](#26-graceful-shutdown-lifecycle)
3. [Top 20 Technical Interview Questions & Answers](#3-top-20-technical-interview-questions--answers)
4. [Live Code Walkthrough Strategy](#4-live-code-walkthrough-strategy)

---

## 1. Core Architectural Blueprint

### Component Interaction Diagram
```
[Client]
   |
   +---> (1) POST /uploads/presign (Authenticated)
   |        Creates upload key, returns presigned Cloudflare R2 PUT URL
   |
   +---> (2) PUT direct to R2 (Zero server bandwidth consumed)
   |
   +---> (3) POST /listings (Passes temporary image keys)
            |
            v
       [API Handler]
            |-- (3a) Validates request body & checks HeadObject on R2
            |-- (3b) Starts Postgres Transaction (BEGIN TX)
            |         |-- INSERT INTO listings (...)
            |         |-- INSERT INTO images (...)
            |         |-- INSERT INTO jobs (kind='process_image', payload=...)
            |        (COMMIT TX)
            |
            v
       [PostgreSQL DB]
            ^
            | (Poll every 3s via FOR UPDATE SKIP LOCKED)
            v
       [Background Worker Goroutine]
            |-- (4a) Fetches raw image from R2
            |-- (4b) Resizes & normalizes to 2048px @ 85% JPEG
            |-- (4c) Puts processed images back to R2 permanent keys
            |-- (4d) BEGIN TX:
            |         |-- UPDATE listings SET status = 'ready' WHERE id = ...
            |         |-- DELETE FROM jobs WHERE id = ...
            |        COMMIT TX
            |-- (4e) Dead-letter handler if failures exceed 3 attempts
```

---

## 2. Deep Dives into Engineering Implementations

### 2.1 Transactional Outbox Pattern & `SKIP LOCKED`
* **The Problem**: When a user creates a listing, if you process and resize images synchronously in the HTTP request:
  1. The HTTP request takes 2-5 seconds, causing high p99 latency and timeouts.
  2. If the server crashes mid-flight after saving the listing, images remain uncompressed or broken.
  3. If you write to Postgres and then publish to an external message broker (RabbitMQ/Kafka), a crash in between causes dual-write failure (database updated but message not sent).
* **The Solution**: 
  - Store both the listing and the background task inside PostgreSQL in the **same ACID transaction** (`BEGIN TX ... INSERT listings ... INSERT jobs ... COMMIT`).
  - Either both succeed or both fail.
* **Worker Execution with `FOR UPDATE SKIP LOCKED`**:
  ```sql
  UPDATE jobs
  SET attempts = attempts + 1,
      run_after = NOW() + LEAST((INTERVAL '30 seconds') * POWER(2, attempts), INTERVAL '1 hour')
  WHERE id = (
      SELECT id FROM jobs
      WHERE run_after <= NOW() AND attempts < $1
      ORDER BY run_after, id
      FOR UPDATE SKIP LOCKED
      LIMIT 1
  )
  RETURNING id, kind, payload, attempts
  ```
  *Why `SKIP LOCKED`?* Multiple worker instances or goroutines can poll simultaneously. Rather than waiting on a lock and stalling each other, each worker skips locked rows and immediately grabs the next available job without contention!

---

### 2.2 Cursor / Keyset Pagination
* **The Problem with `OFFSET / LIMIT`**:
  - `SELECT * FROM listings ORDER BY created_at DESC LIMIT 20 OFFSET 100000;`
  - The database has to scan 100,020 rows, sort them, throw away 100,000 rows, and return 20. Performance degrades linearly ($O(N)$).
  - Also suffers from the **page drift anomaly**: if new listings are posted while a user is browsing, items shift down and users see duplicates across pages.
* **The Solution (Keyset Pagination)**:
  - We use a composite cursor: `(created_at, id)`.
  - Condition: `WHERE (l2.created_at, l2.id) < ($created_at, $id)`.
  - Backed by partial index: `CREATE INDEX listings_ready_pagination_idx ON listings (created_at DESC, id DESC) WHERE status = 'ready';`.
  - The query jumps directly to the B-tree leaf node in $O(\log N)$ time.
  - Encoded into URL-safe Base64 token containing versioning:
    ```go
    type listingCursor struct {
        ID        uuid.UUID `json:"id"`
        CreatedAt time.Time `json:"created_at"`
        Version   int       `json:"version"`
    }
    ```

---

### 2.3 Presigned Object Storage & Validation Pipeline
* **Flow**:
  1. Client calls `POST /uploads/presign` specifying MIME type and size.
  2. Backend signs a PUT request using AWS S3 SDK v2 / Cloudflare R2 credentials with short expiration TTL (e.g., 15 minutes).
  3. Client uploads payload directly to Cloudflare R2.
  4. Client calls `POST /listings` providing the object key.
* **Defense in Depth**:
  - What if a malicious user uploads a 500MB video using the presigned URL?
  - Before writing to the database, the backend issues an S3 `HeadObject` call to inspect `ContentLength` and `ContentType`. If it exceeds 10MB or is not `jpeg/png/webp`, it rejects the request before queuing the job.

---

### 2.4 Thread-Safe Token Bucket Rate Limiter
* **Algorithm**: Token Bucket using Go's official `golang.org/x/time/rate`.
* **Concurreny & Race Conditions**:
  - The limiter maintains an in-memory map `map[string]*bucket` keyed by client IP (`X-Real-IP`).
  - In Go, simultaneous read/write to a map triggers a panic (`fatal error: concurrent map read and map write`).
  - We protect the map using a `sync.Mutex`.
  - In `cmd/limittest`, we simulate 1000 concurrent goroutines hitting the endpoint to prove our synchronization holds under load.
* **Memory Leak Prevention**:
  - A background cleanup goroutine runs on a 10-second ticker, evicting buckets whose `lastSeen` exceeds `bucketTTL` (10 minutes).

---

### 2.5 Timing Attack Mitigation in Authentication
* **The Vulnerability**:
  - Bcrypt hashing is computationally expensive by design (cost 10 takes ~50-80ms).
  - Naive login logic:
    ```go
    user, err := findByEmail(email)
    if err != nil {
        return "invalid credentials" // returns in 1ms!
    }
    if !bcrypt.Compare(user.Password, inputPassword) {
        return "invalid credentials" // returns in 80ms!
    }
    ```
  - An attacker measures response latency: 1ms means email does NOT exist; 80ms means email DOES exist! This enables user account enumeration.
* **The Fix in `auth.go`**:
  - When `sql.ErrNoRows` occurs, we execute:
    ```go
    _ = bcrypt.CompareHashAndPassword(dummyHash, []byte(req.Password))
    ```
  - Both paths take the exact same ~80ms execution time, completely eliminating the timing oracle.

---

### 2.6 Graceful Shutdown Lifecycle
* Listens to OS termination signals (`SIGINT`, `SIGTERM`) using `signal.NotifyContext`.
* When signal arrives:
  1. Stops HTTP server from accepting new connections: `srv.Shutdown(srvShutdownCtx)`.
  2. Gives active in-flight requests 15 seconds to finish.
  3. Signals the background worker context to finish current job: `select { case <-wrk.Done(): case <-time.After(10s): }`.
  4. Closes PostgreSQL connection pool safely.

---

## 3. Top 20 Technical Interview Questions & Answers

### Q1: Why did you use Go standard library `net/http` instead of Gin, Fiber, or Echo?
> **Answer:** In Go 1.22+, `net/http.ServeMux` received major enhancements including method-based routing (`GET /listings/{id}`) and path wildcards. Standard library reduces third-party dependencies, ensures zero allocation routing overhead, guarantees backwards compatibility, and keeps the binary minimal.

### Q2: How does the Transactional Outbox pattern prevent data inconsistency?
> **Answer:** When an operation involves updating a database and emitting an event/job, doing them separately risks dual-write anomalies (e.g. database commits, but network fails before queuing the job). By persisting both the listing and the job inside the exact same PostgreSQL ACID transaction, we guarantee atomicity—either both exist or neither does.

### Q3: Why `FOR UPDATE SKIP LOCKED` instead of an ordinary `SELECT ... FOR UPDATE`?
> **Answer:** Ordinary `FOR UPDATE` causes concurrent workers to queue and block waiting for locks on the same rows, leading to high latency and thread contention. `SKIP LOCKED` instructs the database engine to simply ignore any rows currently locked by other transactions and return the next unlocked row immediately, enabling horizontally scalable concurrent polling without lock contention.

### Q4: What happens if an image processing job fails multiple times?
> **Answer:** We maintain an `attempts` counter and calculate exponential backoff: `run_after = NOW() + LEAST(30s * 2^attempts, 1h)`. If errors are permanent (e.g. corrupted image file) or attempts hit `maxAttempts = 3`, the job routes to the dead-letter handler, which permanently deletes the job and marks the listing as `rejected` in a transaction.

### Q5: Why Keyset (Cursor) pagination instead of Offset pagination?
> **Answer:** Offset pagination executes an $O(N)$ scan because PostgreSQL must traverse all preceding $N$ tuples before returning the page. Keyset pagination uses indexed tuple comparisons `(created_at, id) < ($cur_created_at, $cur_id)`, navigating the B-Tree index directly in $O(\log N)$ time, yielding constant-time queries even on millions of rows. It also prevents duplicate or missing items caused by page drift.

### Q6: Why did you encode the cursor with Base64 JSON instead of passing raw timestamps?
> **Answer:** Base64 JSON hides underlying database schemas from client implementation, prevents client tampering, and includes a `version` field (e.g., `version: 1`). If we later change pagination criteria or sort fields, we can bump the version without breaking older API clients.

### Q7: Why use direct-to-cloud presigned URLs rather than streaming uploads through the Go backend?
> **Answer:** Uploading high-resolution images through the API server ties up server goroutines, network sockets, and memory buffers, turning the API into a bandwidth bottleneck. Generating presigned S3/R2 URLs offloads file transfer directly to the cloud storage infrastructure, keeping the API lightweight and focused on business logic.

### Q8: How do you prevent users from spoofing file sizes or MIME types with presigned URLs?
> **Answer:** We enforce validation on both ends:
> 1. Presigned URL generator enforces max size constraints and whitelisted MIME types.
> 2. When creating the listing, the API server executes an asynchronous `HeadObject` call against R2 to verify the actual uploaded object's byte size and MIME header before committing the database record.

### Q9: What was the race condition caught by your `cmd/limittest` tool?
> **Answer:** In Go, internal hash tables (`map`) are not thread-safe. When multiple requests arrive concurrently for different client IPs, the rate limiter attempts to read and write to `items map[string]*bucket` simultaneously. Without synchronization, the Go runtime throws a fatal panic: `concurrent map read and map write`. We solved this by serializing bucket lookups and writes using a `sync.Mutex`.

### Q10: How do you prevent memory leaks in your rate limiter?
> **Answer:** Without eviction, client IPs would accumulate forever. We run a background cleanup goroutine using a `time.NewTicker(10s)` that iterates over the map under mutex protection and evicts any bucket whose `lastSeen` timestamp is older than 10 minutes (`bucketTTL`).

### Q11: Explain timing attacks during user authentication and how you solved it.
> **Answer:** Bcrypt password verification takes ~80ms. If an invalid email returns immediately in 1ms while a valid email takes 80ms, attackers can infer email existence. We resolve this by running Bcrypt verification against a precomputed constant `dummyHash` even when no user record is found, ensuring both code paths have identical response latencies.

### Q12: Why did you use PostgreSQL instead of MongoDB or Redis for this service?
> **Answer:** Marketplace listings require strong relational constraints, ACID transactions (for outbox queues and image associations), complex indexing (composite partial B-Trees), and strict data integrity. PostgreSQL provides all of this natively along with row-level locking capabilities.

### Q13: How is graceful shutdown handled in this application?
> **Answer:** We utilize `signal.NotifyContext` on `os.Interrupt` and `syscall.SIGTERM`. On receipt, `server.Shutdown` stops accepting new HTTP connections and allows in-flight requests 15 seconds to terminate. Simultaneously, we trigger cancellation on the worker context, allowing it up to 10 seconds to finish its current job, followed by closing the database pool.

### Q14: How does the application handle structured logging and distributed tracing?
> **Answer:** We use Go’s `log/slog` with JSON formatting. A custom middleware generates or extracts a `X-Request-ID` UUID, stores it in the request `context.Context`, and injects it into every log message across handlers and workers, enabling end-to-end log correlation.

### Q15: Why use Docker multi-stage build with `distroless`?
> **Answer:** Multi-stage builds compile the Go binary with stripped debug symbols (`-ldflags="-s -w"`), resulting in an extremely lightweight container (~15-20MB). Running on Google’s `distroless/static:nonroot` provides zero shell or package manager vulnerabilities, drastically reducing attack surface in production.

### Q16: How do you handle password storage securely?
> **Answer:** Passwords are never stored in plaintext. We hash them using `golang.org/x/crypto/bcrypt` with work factor 10, which incorporates individual salts automatically to protect against rainbow table attacks.

### Q17: What is the purpose of image normalization in `internal/imaging`?
> **Answer:** Users upload arbitrary resolutions and orientations. The worker decodes the image, constrains its maximum dimension (width or height) to 2048px while maintaining aspect ratio, and re-encodes it as JPEG at 85% quality, reducing storage cost and optimizing frontend load times.

### Q18: What is the benefit of a partial index in PostgreSQL?
> **Answer:** Listings can have statuses like `draft`, `processing`, `rejected`, or `ready`. Since public queries only retrieve `ready` items, a partial index `WHERE status = 'ready'` consumes significantly less disk space and memory cache than indexing every row, resulting in higher cache hits.

### Q19: Could you replace this in-memory rate limiter with Redis?
> **Answer:** Yes. In a distributed multi-instance deployment behind a load balancer, an in-memory limiter isolates counts to individual nodes. Migrating to Redis with a sliding window or token bucket Lua script provides centralized rate limiting across all API replicas.

### Q20: If traffic spikes 100x, where are the initial bottlenecks and how would you scale?
> **Answer:** 
> 1. **Worker Job Polling**: Polling PostgreSQL every 3 seconds can introduce database load; transition from polling to PostgreSQL `LISTEN/NOTIFY` or an event broker like Redis Streams / SQS.
> 2. **Database Reads**: Introduce Redis caching for popular listings (`GET /listings/{id}`) and read replicas for search queries.
> 3. **Image Processing**: Separate the worker into an independent, auto-scaling deployment so compute-heavy image transformations do not impact API latency.

---

## 4. Live Code Walkthrough Strategy

When an interviewer asks you: *"Walk me through a request from start to finish"*, use the **Listing Creation Flow**:

1. **Step 1 (Pre-sign)**: Point to [`internal/handlers/uploads.go`](file:///c:/Users/Asus/Desktop/olx-api/internal/handlers/uploads.go) -> `Presign`. Explain how presigned URLs allow direct R2 uploads without loading the server.
2. **Step 2 (Validation & Tx)**: Point to [`internal/handlers/listings.go`](file:///c:/Users/Asus/Desktop/olx-api/internal/handlers/listings.go) -> `Create`. Show the `HeadObject` check and how the listing, images, and outbox job are inserted within a single SQL transaction.
3. **Step 3 (Outbox Worker)**: Point to [`internal/worker/worker.go`](file:///c:/Users/Asus/Desktop/olx-api/internal/worker/worker.go) -> `claimAndProcess`. Explain `FOR UPDATE SKIP LOCKED`, exponential backoff calculation, image normalization, and the atomic status update from `processing` to `ready`.
4. **Step 4 (Query & Pagination)**: Point to [`internal/handlers/listings_filters.go`](file:///c:/Users/Asus/Desktop/olx-api/internal/handlers/listings_filters.go) and [`internal/handlers/cursor.go`](file:///c:/Users/Asus/Desktop/olx-api/internal/handlers/cursor.go) to demonstrate keyset pagination and partial index optimization.
