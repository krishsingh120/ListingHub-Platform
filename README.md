# Classifieds Marketplace API (OLX Backend Clone)

A high-concurrency, production-grade RESTful API backend built in **Go (Golang)** for a scalable classifieds marketplace (inspired by OLX). Engineered with resilient architectural patterns including a **Transactional Outbox Worker**, **Non-blocking Keyset Pagination**, **Direct-to-Cloud Presigned S3/R2 Uploads**, **Token Bucket In-Memory Rate Limiting**, and **Constant-Time Timing-Attack Mitigation**.

---

## 🚀 Key Architecture & Engineering Highlights

```
                        +---------------------------------------+
                        |           Client / Frontend           |
                        +---+-------------------------------+---+
                            | (1) Upload Request            | (3) Direct Upload
                            v                               v
                +-----------------------+           +------------------+
                |  API Server (Go Mux)  |           |  Cloudflare R2   |
                +---+---------------+---+           |  / AWS S3 Bucket |
                    |               |               +--------+---------+
 (4) Create Listing |               | (2) Presigned URL      |
 with Temp Keys     |               +------------------------+
                    v
    +-------------------------------+
    | PostgreSQL (ACID Transaction) |
    | - Insert Listing              |
    | - Insert Images Metadata      |
    | - Insert Async Job (Outbox)   |
    +---------------+---------------+
                    |
                    | (5) Poll / Claim via FOR UPDATE SKIP LOCKED
                    v
    +-------------------------------+
    | Background Worker Goroutine   |
    | - Fetch Raw Object from R2    |
    | - Normalize & Resize Images   |
    | - Put Final Images to R2      |
    | - Update Listing (ready)      |
    | - Delete Completed Job        |
    +-------------------------------+
```

### 1. Reliable Asynchronous Processing (Transactional Outbox Pattern)
* Guarantees atomic consistency between relational database records (`listings`, `images`) and asynchronous work tasks (`jobs`).
* Workers claim jobs without blocking other concurrent workers using PostgreSQL **`FOR UPDATE SKIP LOCKED`**.
* Implements exponential backoff retry scheduling:
  $$\text{run\_after} = \text{NOW}() + \min(30s \times 2^{\text{attempts}}, 1hr)$$
* Permanently failing jobs or jobs exceeding maximum retry limits (`maxAttempts = 3`) are routed to a **Dead Letter** handler which marks the listing as `rejected`.

### 2. Zero-Bandwidth Direct Media Upload Pipeline
* Eliminates server network bottlenecks during large image uploads.
* Clients request secure **Presigned S3/R2 URLs** with bounded expiration TTLs and enforced file size/MIME constraints.
* Files are uploaded directly from the browser to Cloudflare R2 object storage.
* Before committing, the backend executes `HeadObject` to verify size limits and allowed content-types (`jpeg`, `png`, `webp`).

### 3. Keyset / Cursor-Based Pagination
* Replaces slow, resource-heavy SQL `OFFSET / LIMIT` operations ($O(N)$ scanning) with constant-time indexed cursor filtering ($O(1)$).
* Uses composite cursor `(created_at, id)` encoded as URL-safe Base64 JSON tokens.
* Backed by PostgreSQL partial composite index:
  ```sql
  CREATE INDEX listings_ready_pagination_idx
  ON listings (created_at DESC, id DESC)
  WHERE status = 'ready';
  ```

### 4. High-Performance Token Bucket Rate Limiting
* Fine-grained thread-safe in-memory rate limiting with `sync.Mutex` and `golang.org/x/time/rate`.
* Granular tiers configured across endpoints:
  * Global tier: 100 req/sec burst
  * Authentication (Sign-in): 5 tokens, 1 token / 5s (12 req/min)
  * User Registration (Sign-up): 3 req/min
  * Write Operations: 10 burst, 1 token / 5s
* Automated background ticker periodically purges stale client IP buckets after TTL expiration to prevent memory leaks.
* Accompanied by concurrency stress-test tool (`cmd/limittest`) proving thread-safety and race condition resilience.

### 5. Security & Timing-Attack Mitigation
* Password hashing with **Bcrypt** (cost 10).
* **Constant-Time Timing Attack Mitigation**: If a requested email is not found during sign-in, the server still evaluates `bcrypt.CompareHashAndPassword` against a precomputed `dummyHash`. This prevents malicious actors from enumerating valid account emails through response latency analysis.
* Stateless **JWT Authentication** (HS256) with registered claims, UUID identification, and context propagation.

### 6. Production Observability & Graceful Shutdown
* Structured JSON logging via Go standard library `log/slog` with request ID tracking across the entire HTTP middleware stack.
* Graceful server shutdown on `SIGINT` / `SIGTERM` with clean worker drain timeout and database connection pool release.

---

## 🛠️ Tech Stack

* **Language**: Go (Golang) 1.26+
* **HTTP Server**: Go Standard Library `net/http` (Go 1.22+ Enhanced Routing `METHOD /pattern/{id}`)
* **Database**: PostgreSQL with `jackc/pgx/v5`
* **Migrations**: `golang-migrate/migrate/v4`
* **Object Storage**: AWS SDK for Go v2 (`aws-sdk-go-v2/service/s3`) connected to Cloudflare R2 / AWS S3
* **Image Processing**: `golang.org/x/image` (Bi-linear / Nearest-Neighbor resize, JPEG compression)
* **Auth**: JWT (`golang-jwt/jwt/v5`) & Bcrypt (`golang.org/x/crypto/bcrypt`)
* **Deployment**: Multi-stage Docker build with Google Container Tools `distroless/static:nonroot`

---

## 📁 Project Structure

```
olx-api/
├── cmd/
│   ├── api/            # API server entrypoint & graceful shutdown lifecycle
│   ├── healthcheck/    # Container healthcheck binary for Kubernetes/Docker
│   ├── limittest/      # Concurrent race-condition & rate-limiter stress test tool
│   └── migrate/        # Database migration CLI runner (up/down)
├── internal/
│   ├── config/         # Environment configuration loader with panic validations
│   ├── db/             # PostgreSQL connection pool initializer
│   ├── handlers/       # HTTP route handlers, DTOs, validations & cursor encoding
│   ├── httpx/          # Standardized JSON response envelope & error schemas
│   ├── imaging/        # In-memory image aspect-ratio normalization & compression
│   ├── jobs/           # Outbox payload types and job specifications
│   ├── middleware/     # Rate limiter, JWT Auth, RequestID, and CORS middleware
│   ├── storage/        # Cloudflare R2 / S3 Presigner & Object client
│   └── worker/         # Transactional outbox background processor & dead-letter queue
├── migrations/         # PostgreSQL versioned UP/DOWN SQL migrations
├── seeds/              # Development seed scripts
├── .env.example        # Environment variable blueprint
├── Dockerfile          # Multi-stage scratch/distroless production container
└── Makefile            # Common build & migration targets
```

---

## ⚙️ Environment Variables

Create your `.env` file from the provided template:

```bash
cp .env.example .env
```

| Variable | Description | Example |
| :--- | :--- | :--- |
| `PORT` | API Server listening port | `8088` |
| `ENV` | Environment (`development`, `production`) | `development` |
| `FRONTEND_ORIGIN`| Allowed CORS Origin | `http://localhost:3000` |
| `DATABASE_URL` | PostgreSQL connection string | `postgres://postgres:postgres@localhost:5432/olx_dev?sslmode=disable` |
| `JWT_KEY` | Secret key for signing HS256 tokens | `min-32-chars-secret-key` |
| `STORAGE_ACCOUNT_ID` | Cloudflare Account ID | `cloudflare_account_id` |
| `STORAGE_ACCESS_KEY` | R2 / S3 Access Key ID | `r2_access_key` |
| `STORAGE_ACCESS_SECRET`| R2 / S3 Secret Access Key | `r2_secret_key` |
| `BUCKET` | Cloudflare R2 Bucket name | `olx-listings-bucket` |

---

## 🏃 Running the Project Locally

### Prerequisites
* Go 1.26+ installed
* PostgreSQL 15+ installed and running
* Cloudflare R2 bucket credentials (or AWS S3 / MinIO)

### 1. Database Setup & Migrations
Create your PostgreSQL database:
```sql
CREATE DATABASE olx_dev;
```

Run database migrations:
```bash
make migrate-up
# or: go run ./cmd/migrate up
```

### 2. Start the API Server
```bash
make run
# or: go run ./cmd/api
```

The server will initialize the database pool, connect to Cloudflare R2 storage, launch the background worker goroutine, and listen on the configured port.

### 3. Run Rate Limiter Stress Test
In a separate terminal, test the rate limiter under 1000 concurrent goroutine requests:
```bash
go run ./cmd/limittest
```

---

## 🐳 Docker Deployment

Build and run using the optimized multi-stage distroless Docker image:

```bash
# Build Docker image
docker build -t olx-api:latest .

# Run container
docker run -p 8088:8088 --env-file .env olx-api:latest
```

---

## 📡 API Reference & Standard Response Envelope

All API error responses follow a uniform JSON schema:

```json
{
  "code": "validation_failed",
  "message": "title is required",
  "field": "title"
}
```

### Core Endpoints

| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `GET` | `/healthz` | No | Liveness/Readiness probe |
| `POST` | `/signup` | No | Register new user account (rate-limited) |
| `POST` | `/signin` | No | Authenticate user & issue JWT token |
| `GET` | `/me` | Yes | Get authenticated user profile |
| `POST` | `/uploads/presign` | Yes | Request presigned R2/S3 upload URLs |
| `GET` | `/listings` | No | List ready items with cursor pagination & city/price filters |
| `GET` | `/listings/{id}` | No | Get listing details with processed images |
| `POST` | `/listings` | Yes | Create listing (triggers async image processing) |
| `PUT` | `/listings/{id}` | Yes | Update owned listing |
| `DELETE`| `/listings/{id}` | Yes | Delete owned listing |
| `GET` | `/me/listings` | Yes | Retrieve listings created by current user |
