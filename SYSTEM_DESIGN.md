# Antriaga — System Design & Implementation Blueprint

> **Distributed Background Job Engine**  
> *Every Job Matters. Every Failure Has a Recovery Path.*

| Metadata | Nilai |
|---|---|
| Project | **Antriaga** |
| Jenis | Distributed, fault-tolerant background job processing system |
| Bahasa | Go (Golang) |
| HTTP framework | **Fiber v3** |
| Arsitektur kode | **Clean Architecture** + ports/adapters |
| Persistence | **MySQL 8.4**, GORM, SQL migrations |
| Message broker | **Redis Streams** (Redis 7+) |
| Dokumentasi API | **Swaggo** (`swaggo/swag`) + Fiber Swagger UI |
| Testing | Go `testing`, `testify`, `testcontainers-go`, `go test -race` |
| Logging | Go `log/slog`, structured JSON events, per-job correlation & redaction |
| Authentication | **Client-based API keys** (admin generated, scoped, revocable, stored as keyed digest) |
| Resource governance | Backpressure, bounded worker slots, per-client isolation |
| Payload schema | Business `payload_version` terpisah dari Redis `schema_version` |
| Status dokumen | Rancangan implementasi — belum menyatakan kode sudah dibangun |
| AI-assisted workflow | Spec-driven tasks, repository `AGENTS.md`, task handoff, skill selection, test-first verification |
| Revisi | **v4.1 — dokumentasi pengguna, release gates, dan penegasan kontrak lease worker** |
| Tanggal | 23 November 2026 |

---

## 1. Executive Summary

**Antriaga** adalah mesin pemrosesan pekerjaan latar belakang (background job) yang memungkinkan service mengirim pekerjaan melalui HTTP API tanpa menunggu pekerjaan selesai. Antriaga menyimpan state dan histori pekerjaan secara persisten di **MySQL**, mendistribusikan pekerjaan ke sejumlah worker melalui **Redis Streams**, kemudian mengelola retry, scheduling, dan pemulihan saat terjadi kegagalan.

**Developer workflow:** implementasi dapat dibantu AI coding agent dengan task kecil, aturan `AGENTS.md`, test-first, dan bukti verifikasi (Bagian 36–44). **Dokumentasi pengguna:** `README.md` dan berkas `docs/` untuk instalasi, API, autentikasi, integrasi, troubleshooting, operasi, dan keamanan (Bagian 45–47). **V4 mempertahankan seluruh keputusan arsitektur v3**; repository skills merupakan alat bantu, bukan sumber kebenaran di atas dokumen ini.

**Observability:** seluruh lifecycle job (submit, publish, claim, execute, retry, scheduling, dead-letter, ACK, dan crash recovery) didokumentasikan dengan event log JSON terstruktur; lihat Bagian 16. **Auth:** setiap permintaan client memakai API key yang diterbitkan administrator, di-scope per client; key dapat dirotasi dan dicabut (Bagian 27). **Ketahanan:** worker menerapkan backpressure, recovery menghindari race, dan payload memiliki versi (Bagian 28–30).

**Positioning:** *Antriaga is a fault-tolerant distributed background job engine built with Go, Fiber, MySQL, and Redis Streams. It provides persistent job tracking, scheduled execution, bounded retries, dead-letter handling, worker crash recovery, idempotency support, and horizontally scalable workers.*

**Bukan** workflow orchestrator general-purpose, message bus untuk semua event, atau layanan eksekusi arbitrary code. MVP hanya mendukung job type yang telah terdaftar, dan URL external system dikonfigurasi oleh server.

### 1.1 Masalah yang diselesaikan

- Request API tidak perlu menunggu pengiriman email, webhook, atau pembuatan laporan.
- Workload bisa didistribusikan ke banyak worker yang di-scale secara independen.
- Job tidak hanya disimpan di memori; status, attempts, dan output tercatat di MySQL.
- Kegagalan sementara bisa dicoba ulang tanpa membuat retry loop tak terbatas.
- Job yang tertunda dapat dijalankan sesuai `scheduled_at`.
- Worker yang crash dapat dipulihkan menggunakan lease DB dan Redis Pending Entries List (PEL).
- Operator bisa mengobservasi queue depth, failure rate, histori, dan dead jobs.
- Developer bisa menelusuri satu `job_id` lintas API, dispatcher, worker, scheduler, dan reaper melalui structured event logs.

### 1.2 Prinsip desain

1. **MySQL adalah source of truth.** Redis berperan sebagai transport/notifikasi kesiapan job.
2. **At-least-once delivery, bukan exactly-once.** Redelivery dan pengulangan side effect mungkin terjadi.
3. **Idempotency end-to-end.** Untuk side effect kritis, worker wajib menyampaikan idempotency key yang stabil ke external system.
4. **State transition terkontrol.** Transisi job dilakukan lewat use case/repository yang memenuhi prasyarat state.
5. **Atomic DB changes.** Penyimpanan job dan intent publish menggunakan transactional outbox.
6. **Framework-independent core.** Domain dan application tidak mengimpor Fiber, GORM, atau Redis SDK.
7. **Bounded retry & recovery.** Tidak ada retry tanpa batas untuk error permanen atau poison message.
8. **Observable and testable.** Setiap jalur sukses, gagal, retry, dan crash dapat diuji.
9. **Client identity is enforced.** API key menentukan `client_id` dan scopes. Setiap job/queries terisolasi per client, termasuk idempotency.
10. **Bounded admission and concurrency.** Jangan overconsume Redis atau menambah backlog tanpa batas.
11. **Versioned payloads.** Business `payload_version` immutable dan kompatibel selama rolling deployment.

---

## 2. Scope, Requirement, dan Non-goal

### 2.1 MVP (wajib)

| Area | Kemampuan |
|---|---|
| Job API | Create job, get by ID, list/filter, get attempts — semuanya client-scoped |
| Client auth (P0) | Unique client identity, admin CLI generate/rotate/revoke API key, scopes, hashed credential, owner isolation |
| Capacity (P0) | Worker slot-based consume, limiter API independen dari Redis, serta batas job aktif dan backlog outbox |
| Compatibility (P0) | Business `payload_version` dan rolling deploy policy |
| Job types | `email.send`, `webhook.deliver`, `payment.capture`, `report.generate` |
| Database | Jobs, job attempts, transactional outbox |
| Queue | Redis Streams dan consumer groups per queue |
| Worker | Handler registry, concurrent workers, graceful shutdown |
| Scheduler | Delayed job dan retry due dengan polling database |
| Reliability | At-least-once, exponential backoff + jitter, bounded attempts, dead jobs |
| Recovery | Redis PEL reclaim (`XPENDING` + `XCLAIM` guard), DB lease expiry, stale-message handling |
| API docs | Swaggo annotations dan Swagger UI |
| Testing | Unit, HTTP handler, repository/broker integration, end-to-end crash scenarios |
| Logging | `slog` JSON, lifecycle event catalog, per-job correlation, error/retry/recovery logs, redaction, log unit tests |
| Delivery | Docker Compose, migrations, Makefile, CI dasar |

### 2.2 Setelah MVP

Manual dead-job requeue (P1), retention/cleanup worker (P1), operational runbooks (P1), durable `job_events` timeline (P2), priority yang benar-benar ditegakkan, cancel cooperative, pause/resume queue, advanced per-tenant quotas, RBAC manusia, web dashboard, RabbitMQ adapter, SSE/WebSocket, OpenTelemetry tracing, Kubernetes/Helm, dan multi-region architecture. **Per-client API key + basic rate limit adalah MVP**, tidak ditunda ke fitur multi-tenancy lanjutan.

> **Keputusan MVP:** kolom `priority` boleh dipersiapkan di database, tetapi **tidak dijanjikan berpengaruh pada urutan Redis Streams**. Jangan mengklaim priority queue sebelum scheduling prioritas benar-benar diimplementasikan.

### 2.3 Non-functional requirement dan target awal

| Indikator | Target awal / catatan |
|---|---|
| Delivery semantics | At-least-once |
| Job creation | HTTP `202 Accepted` setelah transaksi MySQL berhasil |
| DB consistency | Job dan outbox intent tersimpan dalam satu transaksi |
| Schedule precision | Best effort, bergantung pada polling dan backlog, bukan real-time guarantee |
| Scheduler polling | 1 detik (konfigurabel) |
| Worker concurrency | Awal 10 per instance, configurable; `XREADGROUP COUNT <= available slots` |
| Redis consume batch | Awal maksimum 10, atau sebesar sisa slot worker; tidak boleh over-prefetch |
| Default max attempts | 5 |
| Retry base / maximum | 5 detik / 15 menit |
| Retry jitter | 0–20% |
| Execution timeout | 5 menit default, per job type bisa berbeda |
| Heartbeat interval | 30 detik |
| DB execution lease | 90 detik, diperpanjang heartbeat selama job aktif |
| Reaper polling | 15 detik |
| Graceful shutdown | Maksimum 30 detik untuk drain; lanjut recovery jika belum selesai |
| Payload limit | 256 KiB; file besar ke object storage |
| Timezone | Internal UTC, tampilkan sesuai kebutuhan client |
| API authentication | Scoped API key per unique client, admin-generated, revoked/expired checked |
| API key entropy | 32 cryptographically secure random bytes per key, plaintext only-once |
| Backpressure | Per-client rate limit + overload policy; admission dapat `429` atau `503` |
| Business payload version | `payload_version` default 1, disimpan di jobs |

Angka di atas adalah **konfigurasi awal untuk diuji**, bukan hasil benchmark. Tuning dilakukan berdasarkan metrik dan load test.

---

## 3. Technology Stack

| Layer | Teknologi | Catatan |
|---|---|---|
| HTTP transport | `github.com/gofiber/fiber/v3` | API, middlewares, health checks |
| Swagger generator | `github.com/swaggo/swag` | Menghasilkan Swagger 2.0 spec dari annotation |
| Swagger UI | `github.com/gofiber/contrib/v3/swaggo` | Adapter Fiber v3 untuk UI Swaggo |
| ORM | `gorm.io/gorm` + `gorm.io/driver/mysql` | Hanya di infrastructure |
| SQL migrations | `golang-migrate/migrate` | Migrasi eksplisit; hindari `AutoMigrate()` di production |
| Broker client | `github.com/redis/go-redis/v9` | Redis Streams |
| Database | MySQL 8.4 | InnoDB, transactions, `SKIP LOCKED` |
| Redis | Redis 7+ | AOF persistence; consumer groups, PEL, guarded `XPENDING`/`XCLAIM` recovery; `XAUTOCLAIM` optional |
| IDs | UUIDv7 library dengan dukungan yang sesuai | Sorting mendekati urutan waktu; tidak menjadi security boundary |
| Validation | `go-playground/validator/v10` atau validasi eksplisit | DTO validation di inbound adapter |
| Logging | `log/slog` | JSON lifecycle events, correlation keys, safe redaction; see §16 |
| API key generation | `crypto/rand`, `crypto/hmac`, `crypto/sha256` | Standard library; separate admin CLI and secret manager |
| Metrics | Prometheus client library | Internal `/metrics` |
| Unit testing | Go `testing`, `testify` | Fake/mock berbasis ports |
| Integration testing | `testcontainers-go` | MySQL/Redis nyata di test |
| Deployment | Docker Compose | MVP/demo lokal |

**Version policy:** kunci versi dependency di `go.mod` dan image digest/tag yang teruji. Pastikan kompatibilitas versi Go dengan Fiber v3 dan middleware pilihan sebelum membuat release. Jangan menggunakan `@latest` tanpa pinning di CI production.

---

## 4. Arsitektur Sistem

### 4.1 System context

```mermaid
flowchart LR
    C[Client / Backend Service] -->|HTTPS + X-API-Key| AUTH[API Key Middleware]
    AUTH --> A[Antriaga API]
    O[Operator] -->|Inspect job / metrics| A
    A --> D[(MySQL: clients, keys, jobs, attempts, outbox)]
    A -->|Transactional intent| D
    P[Outbox Dispatcher] -->|Read intent| D
    P -->|XADD| R[(Redis Streams)]
    R -->|XREADGROUP| W[Worker Pools]
    W -->|State, result, attempts| D
    W --> E[Mock External Services]
    S[Scheduler] -->|Due scheduled/retry jobs| D
    S -->|Create outbox intent| D
    X[Reaper / Reconciler] -->|PEL + lease recovery| R
    X -->|Reconcile state| D
    M[Prometheus] -->|Scrape| A
```

Catatan: arrow representasi intent menuju DB bukan berarti transaksi MySQL dan Redis bisa di-commit atomik.

### 4.2 Runtime containers

```mermaid
flowchart TB
    subgraph Inbound[Public entry]
        LB[Reverse Proxy / Load Balancer]
        API[Antriaga API: Fiber]
        LB --> API
    end
    subgraph Control[Control plane]
        SCH[Scheduler]
        OUT[Outbox Dispatcher]
        REC[Reaper + Reconciler]
    end
    subgraph DataPlane[Execution plane]
        EM[Email Workers]
        WH[Webhook Workers]
        PM[Payment Workers]
        RP[Report Workers]
    end
    subgraph State[State & transport]
        MYSQL[(MySQL)]
        REDIS[(Redis Streams)]
        S3[(MinIO / Object storage)]
    end
    API --> MYSQL
    SCH --> MYSQL
    OUT --> MYSQL
    OUT --> REDIS
    REC --> MYSQL
    REC --> REDIS
    REDIS --> EM & WH & PM & RP
    EM & WH & PM & RP --> MYSQL
    RP --> S3
    EM --> MAIL[Mock Mailer]
    WH --> WEBHOOK[Mock Webhook Receiver]
    PM --> PAYMENT[Mock Payment Gateway]
```

**Scaling boundary:** binary/API instances, worker instances, scheduler, dispatcher, dan reaper dapat dijalankan terpisah. Pada MVP satu binary `maintenance` boleh menjalankan scheduler + dispatcher + reaper sebagai proses terpisah secara logis, tetapi deployment production sebaiknya menyediakan scaling/HA secara jelas.

### 4.3 Clean Architecture dependency rule

```mermaid
flowchart LR
    I[Inbound adapter: Fiber HTTP] --> U[Application: use cases]
    U --> D[Domain: entities + rules]
    U --> P[Ports: contracts]
    O[Outbound adapters: GORM / Redis / Clock / External Clients] -.implement.-> P
    B[Bootstrap / DI] --> I
    B --> O
```

Aturan ketergantungan:

- `domain` hanya menggunakan Go standard library jika memungkinkan.
- `application` bergantung pada `domain` dan **interfaces/ports**, bukan detail adapter.
- `adapters/inbound/http` menggunakan Fiber serta memanggil use case.
- `adapters/outbound/mysql` menggunakan GORM; adapter Redis menggunakan go-redis.
- `cmd/*` membangun dependency graph (manual dependency injection).
- Tidak ada `fiber.Ctx`, `*gorm.DB`, atau `redis.Client` sebagai parameter use case/domain.

---

## 5. Struktur Repository

```text
antriaga/
├── cmd/
│   ├── api/main.go
│   ├── worker/main.go
│   ├── scheduler/main.go
│   ├── dispatcher/main.go
│   ├── reaper/main.go
│   └── admin/main.go
├── internal/
│   ├── domain/
│   │   ├── job.go
│   │   ├── client.go
│   │   ├── api_key.go
│   │   ├── job_status.go
│   │   ├── job_error.go
│   │   ├── attempt.go
│   │   └── job_test.go
│   ├── application/
│   │   ├── ports/
│   │   │   ├── job_repository.go
│   │   │   ├── client_repository.go
│   │   │   ├── api_key_repository.go
│   │   │   ├── attempt_repository.go
│   │   │   ├── outbox_repository.go
│   │   │   ├── unit_of_work.go
│   │   │   ├── broker.go
│   │   │   ├── clock.go
│   │   │   └── job_handler.go
│   │   ├── authentication/
│   │   │   ├── authenticate_api_key.go
│   │   │   └── principal.go
│   │   ├── administration/
│   │   │   ├── create_client.go
│   │   │   ├── issue_api_key.go
│   │   │   └── revoke_api_key.go
│   │   ├── jobs/
│   │   │   ├── create_job.go
│   │   │   ├── get_job.go
│   │   │   ├── list_jobs.go
│   │   │   ├── get_attempts.go
│   │   │   └── requeue_job.go       # P1
│   │   ├── capacity/
│   │   │   ├── admission_policy.go
│   │   │   └── concurrency_limits.go
│   │   ├── execution/
│   │   │   ├── claim_job.go
│   │   │   ├── execute_job.go
│   │   │   ├── finish_job.go
│   │   │   └── retry_policy.go
│   │   ├── scheduling/
│   │   │   └── dispatch_due_jobs.go
│   │   ├── dispatching/
│   │   │   └── publish_outbox.go
│   │   └── recovery/
│   │       ├── recover_stale_jobs.go
│   │       └── reconcile_queued_jobs.go
│   ├── adapters/
│   │   ├── inbound/http/
│   │   │   ├── router.go
│   │   │   ├── handlers/job_handler.go
│   │   │   ├── handlers/health_handler.go
│   │   │   ├── dto/job_request.go
│   │   │   ├── dto/job_response.go
│   │   │   ├── middleware/auth.go
│   │   │   ├── middleware/scope.go
│   │   │   ├── middleware/rate_limit.go
│   │   │   ├── middleware/request_id.go
│   │   │   └── middleware/access_log.go
│   │   └── outbound/
│   │       ├── mysql/
│   │       │   ├── connection.go
│   │       │   ├── models/client_model.go
│   │       │   ├── models/api_key_model.go
│   │       │   ├── models/job_model.go
│   │       │   ├── models/attempt_model.go
│   │       │   ├── models/outbox_model.go
│   │       │   ├── client_repository.go
│   │       │   ├── api_key_repository.go
│   │       │   ├── job_repository.go
│   │       │   ├── attempt_repository.go
│   │       │   └── unit_of_work.go
│   │       ├── redisstreams/
│   │       │   ├── broker.go
│   │       │   ├── pending_inspector.go
│   │       │   └── consumer.go
│   │       ├── external/
│   │       │   ├── mailer.go
│   │       │   ├── webhook.go
│   │       │   └── payment.go
│   │       ├── storage/minio.go
│   │       └── system/clock.go
│   ├── worker/
│   │   ├── runtime.go
│   │   ├── registry.go
│   │   ├── heartbeat.go
│   │   └── handlers/
│   │       ├── email.go
│   │       ├── webhook.go
│   │       ├── payment.go
│   │       └── report.go
│   ├── platform/
│   │   ├── config/config.go
│   │   ├── credentials/
│   │   │   ├── generate.go
│   │   │   ├── verify.go
│   │   │   └── credentials_test.go
│   │   ├── logger/
│   │   │   ├── logger.go
│   │   │   ├── events.go
│   │   │   ├── redact.go
│   │   │   └── logger_test.go
│   │   └── metrics/metrics.go
│   └── bootstrap/
│       ├── admin.go
│       ├── api.go
│       ├── worker.go
│       └── maintenance.go
├── docs/                       # User docs + generated Swaggo (docs.go, swagger.{json,yaml})
│   ├── README.md              # Public documentation index
│   ├── getting-started/       # installation.md, quickstart.md
│   ├── guides/                # authentication, submitting-jobs, job-types, tracking-jobs
│   ├── reference/             # api, configuration, errors
│   ├── examples/              # go-client.md
│   ├── operations/            # deployment, monitoring, troubleshooting, security
│   ├── faq.md
│   ├── runbooks/              # Incident response (P1)
│   └── ai/                    # AI-assisted implementation artifacts (v4)
│       ├── task-template.md
│       ├── worklog.md
│       └── decisions.md
├── design/                     # ADRs, diagrams, failure scenarios
├── .agents/skills/             # Optional; selective, reviewed skills only
├── AGENTS.md                   # Root agent instructions / architectural guardrails
├── migrations/
│   ├── 000001_create_clients.up.sql
│   ├── 000001_create_clients.down.sql
│   ├── 000002_create_api_keys.up.sql
│   ├── 000002_create_api_keys.down.sql
│   ├── 000003_create_jobs.up.sql
│   ├── 000003_create_jobs.down.sql
│   ├── 000004_create_job_attempts.up.sql
│   ├── 000004_create_job_attempts.down.sql
│   ├── 000005_create_outbox_events.up.sql
│   ├── 000005_create_outbox_events.down.sql
│   ├── 000006_create_job_events.up.sql    # P2, not applied in MVP
│   └── 000006_create_job_events.down.sql  # P2, not applied in MVP
├── examples/
│   ├── mock-mailer/
│   ├── mock-webhook/
│   ├── mock-payment/
│   └── load-generator/
├── tests/
│   ├── integration/
│   ├── e2e/
│   ├── security/                # client isolation, key rotation/revocation
│   └── fixtures/
├── deployments/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/workflows/ci.yml
├── .env.example
├── .golangci.yml
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

**Catatan:** pisahkan package berdasarkan tanggung jawab, bukan membuat satu package `utils` besar. Unit test berdekatan dengan kode yang diuji, sementara integration/e2e test berada di `tests/`.

---

## 6. Domain Model dan State Machine

### 6.1 Entity Job

| Field | Tipe konsep | Deskripsi |
|---|---|---|
| `id` | UUIDv7 | Primary identifier |
| `client_id` | UUIDv7 | Owner client; immutable, extracted from validated API key |
| `queue` | string | `email`, `webhook`, `payment`, `report` |
| `type` | string | Salah satu job type statis |
| `payload` | Teks JSON UTF-8 | Input tervalidasi, maksimal 256 KiB. Token angka dipertahankan |
| `payload_version` | uint16 | Version of business JSON payload, independent from stream schema |
| `result` | nullable JSON | Ringkasan output, bukan file besar |
| `status` | enum | State job |
| `attempts` | int | Jumlah eksekusi yang sudah di-claim |
| `max_attempts` | int | Maksimum handler execution attempt |
| `enqueue_seq` | integer | Generation/fencing untuk stream message |
| `scheduled_at` | nullable UTC time | Waktu eligible untuk enqueue |
| `lease_owner` | nullable string | Worker pemilik eksekusi |
| `lease_token` | nullable UUID | Token unique untuk menolak update stale worker |
| `lease_expires_at` | nullable UTC time | Batas lease aktif |
| `idempotency_key` | nullable string | Dedup saat create request |
| `last_error_code` | nullable string | Klasifikasi error terakhir |
| `last_error_message` | nullable string | Error teredaksi |
| `created_at`, `updated_at` | UTC timestamp | Audit timestamps |

### 6.2 Status

| Status | Makna |
|---|---|
| `scheduled` | Job menunggu jadwal pertama |
| `queued` | Sudah eligible diproses; bisa masih menunggu outbox dispatcher/broker |
| `processing` | Dieksekusi satu worker dengan DB lease |
| `retrying` | Gagal sementara, menunggu backoff |
| `completed` | Handler selesai dan hasil DB commit |
| `dead` | Permanent error atau max attempts habis |
| `cancelled` | Tidak dilanjutkan; implementasi API cancel masuk fase setelah MVP |

**Tidak perlu** status `failed` terpisah pada MVP: kegagalan percobaan dicatat di `job_attempts`; job keseluruhan masuk `retrying` atau `dead`. Ini menghindari ambiguitas antara kegagalan sementara dan terminal.

```mermaid
stateDiagram-v2
    [*] --> scheduled: create dengan scheduled_at masa depan
    [*] --> queued: create immediate
    scheduled --> queued: scheduler + outbox
    queued --> processing: worker claim + lease
    processing --> completed: success
    processing --> retrying: transient failure, attempts tersisa
    processing --> dead: permanent failure / attempts habis, termasuk saat recovery
    retrying --> queued: backoff due + outbox
    processing --> queued: lease expired + recovery
    scheduled --> cancelled: cancel (fase 2)
    queued --> cancelled: cancel (fase 2)
    retrying --> cancelled: cancel (fase 2)
    completed --> [*]
    dead --> [*]
    cancelled --> [*]
```

**Requeue operator (P1):** kontrak di §31 menggunakan command eksplisit dan audit trail. Rancangan yang diuji di §31.2 membuat job baru yang terhubung ke job `dead`, sehingga histori attempt lama tetap utuh. MVP berhenti pada state terminal.

### 6.3 Domain invariants

1. Job type harus dikenal registry dan cocok dengan queue yang dipetakan server.
2. `attempts <= max_attempts`. Hanya conditional claim yang menaikkan `attempts`. Recovery menutup attempt yang sudah dihitung saat claim, lalu antrekan ulang job jika masih ada jatah attempt.
3. Hanya job dengan `status=queued` dan `enqueue_seq` cocok yang dapat di-claim.
4. Worker hanya boleh memperpanjang lease atau menutup attempt jika job `processing`, `lease_token` cocok, dan `lease_expires_at > db_now` menurut waktu MySQL setelah lock baris diperoleh. Pada `lease_expires_at <= db_now`, lease sudah berakhir dan recovery dapat mengambil alih (§9.2, §29.2).
5. Terminal job tidak dapat diproses ulang tanpa command requeue yang terautentikasi.
6. `idempotency_key` untuk submit request yang sama mengembalikan job existing; request berkonflik dengan payload berbeda ditolak.
7. Satu `attempt_number` hanya mempunyai satu baris attempt per job.
8. Semua operasi waktu menggunakan UTC.
9. Semua job mempunyai `client_id` yang berasal dari validated API key; data access dibatasi owner.
10. `payload_version` job immutable; setiap handler hanya memproses versi yang kompatibel.
11. Idempotency request scoped per `client_id`, bukan global.

---

## 7. Database Design (MySQL)

**InnoDB** adalah satu-satunya transactional source of truth. Gunakan migrations versioned, foreign keys yang relevan, dan index yang mengikuti pola query aktual.

### 7.1 ERD

```mermaid
erDiagram
    CLIENTS ||--o{ API_KEYS : owns
    CLIENTS ||--o{ JOBS : submits
    JOBS ||--o{ JOB_ATTEMPTS : has
    JOBS ||--o{ OUTBOX_EVENTS : produces
    CLIENTS {
        char_36 id PK
        varchar_80 code UK
        varchar_16 status
    }
    API_KEYS {
        char_36 id PK
        char_36 client_id FK
        binary_32 secret_digest
        smallint pepper_version
        json scopes
        varchar_16 status
    }
    JOBS {
        char_36 id PK
        char_36 client_id FK
        varchar_100 queue
        varchar_100 type
        mediumtext payload
        smallint payload_version
        json result
        varchar_24 status
        smallint priority
        int attempts
        int max_attempts
        bigint enqueue_seq
        datetime scheduled_at
        datetime started_at
        varchar_128 lease_owner
        char_36 lease_token
        datetime lease_expires_at
        varbinary_150 idempotency_key
        char_64 request_hash
    }
    JOB_ATTEMPTS {
        bigint id PK
        char_36 job_id FK
        int attempt_number
        varchar_128 worker_id
        varchar_24 status
        datetime started_at
        datetime finished_at
    }
    OUTBOX_EVENTS {
        char_36 id PK
        char_36 job_id FK
        bigint enqueue_seq
        varchar_24 status
        datetime available_at
        varchar_41 published_stream_id
        char_36 lease_token
        datetime lease_expires_at
    }
```

### 7.2 Migration `jobs` (setelah `clients` dan `api_keys`; lihat §27.2)

```sql
CREATE TABLE jobs (
    id CHAR(36) NOT NULL PRIMARY KEY,
    client_id CHAR(36) NOT NULL,
    queue VARCHAR(100) NOT NULL,
    type VARCHAR(100) NOT NULL,
    payload MEDIUMTEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
    payload_version SMALLINT UNSIGNED NOT NULL DEFAULT 1,
    result JSON NULL,
    status VARCHAR(24) NOT NULL,
    priority SMALLINT NOT NULL DEFAULT 5,
    attempts INT NOT NULL DEFAULT 0,
    max_attempts INT NOT NULL DEFAULT 5,
    enqueue_seq BIGINT NOT NULL DEFAULT 1,
    scheduled_at DATETIME(3) NULL,
    started_at DATETIME(3) NULL,
    completed_at DATETIME(3) NULL,
    lease_owner VARCHAR(128) NULL,
    lease_token CHAR(36) NULL,
    lease_expires_at DATETIME(3) NULL,
    idempotency_key VARBINARY(150) NULL,
    request_hash CHAR(64) NULL,
    last_error_code VARCHAR(80) NULL,
    last_error_message TEXT NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uq_jobs_client_idempotency (client_id, idempotency_key),
    KEY idx_jobs_client_created (client_id, created_at),
    KEY idx_jobs_client_status (client_id, status, created_at),
    KEY idx_jobs_due (status, scheduled_at),
    KEY idx_jobs_queue_status_created (queue, status, created_at),
    KEY idx_jobs_expired_leases (status, lease_expires_at),
    KEY idx_jobs_created (created_at),
    CONSTRAINT fk_jobs_client FOREIGN KEY (client_id) REFERENCES clients(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`payload` menyimpan teks JSON UTF-8 yang sudah divalidasi. Binding INSERT menggunakan teks asli payload tanpa CAST ke `JSON` dan tanpa konversi angka ke floating point. Worker membaca teks itu kembali, sehingga token angka pada `webhook.deliver.data` bertahan sampai request outbound. `result` tetap memakai tipe JSON.

`idempotency_key` nullable: MySQL mengizinkan lebih dari satu `NULL` pada unique index. **v3 sudah melakukan namespace per client** dengan unique `(client_id, idempotency_key)`, tanpa mengandalkan `client_id` dari input pengguna. Jika kelak ada organisasi dengan beberapa client, tenant hierarchy adalah fitur terpisah.

### 7.3 Migration `job_attempts`

```sql
CREATE TABLE job_attempts (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    job_id CHAR(36) NOT NULL,
    attempt_number INT NOT NULL,
    worker_id VARCHAR(128) NOT NULL,
    lease_token CHAR(36) NOT NULL,
    status VARCHAR(24) NOT NULL,
    error_code VARCHAR(80) NULL,
    error_message TEXT NULL,
    started_at DATETIME(3) NOT NULL,
    finished_at DATETIME(3) NULL,
    duration_ms BIGINT NULL,
    created_at DATETIME(3) NOT NULL,
    UNIQUE KEY uq_attempt_job_number (job_id, attempt_number),
    KEY idx_attempts_job (job_id, started_at),
    CONSTRAINT fk_attempts_job FOREIGN KEY (job_id) REFERENCES jobs(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Attempt status yang direkomendasikan: `processing`, `completed`, `retryable_failed`, `permanent_failed`, `lease_expired`. Recovery akan menutup attempt lama sebagai `lease_expired`, **tidak membuat attempt number kedua yang sama**. Jika handler sempat melakukan side effect sebelum crash, database tidak dapat memastikan efek eksternal tersebut belum terjadi.

### 7.4 Migration `outbox_events`

```sql
CREATE TABLE outbox_events (
    id CHAR(36) NOT NULL PRIMARY KEY,
    job_id CHAR(36) NOT NULL,
    enqueue_seq BIGINT NOT NULL,
    event_type VARCHAR(40) NOT NULL DEFAULT 'job.ready',
    status VARCHAR(24) NOT NULL DEFAULT 'pending',
    available_at DATETIME(3) NOT NULL,
    published_at DATETIME(3) NULL,
    published_stream_id VARCHAR(41) CHARACTER SET ascii COLLATE ascii_bin NULL,
    lease_owner VARCHAR(128) NULL,
    lease_token CHAR(36) NULL,
    lease_expires_at DATETIME(3) NULL,
    publish_attempts INT NOT NULL DEFAULT 0,
    last_error TEXT NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uq_outbox_job_generation (job_id, enqueue_seq),
    KEY idx_outbox_due (status, available_at),
    KEY idx_outbox_expired (status, lease_expires_at),
    CONSTRAINT fk_outbox_job FOREIGN KEY (job_id) REFERENCES jobs(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Outbox status:** `pending -> publishing -> published`. Dispatcher lain dapat mengklaim ulang row `publishing` setelah lease kedaluwarsa. Setiap claim memiliki token baru, dan finalisasi harus membuktikan token serta lease masih aktif (§9.1). Redis mungkin sudah menerima event sebelum dispatcher crash; duplikasi dianggap normal dan ditangani oleh `enqueue_seq` + DB conditional claim.

### 7.5 GORM mapping

**Urutan SQL migrations greenfield (v3):** `000001_clients`, `000002_api_keys`, `000003_jobs`, `000004_job_attempts`, `000005_outbox_events`; `000006_job_events` adalah P2. SQL untuk `clients`/`api_keys` dan audit auth ada di §27.2. Jangan langsung menerapkan migrasi ini ke database v2 yang sudah terisi tanpa backfill teruji.


- GORM model hanya di `adapters/outbound/mysql/models`.
- Mapping `JobEntity <-> JobModel` dilakukan di repository adapter, bukan di domain.
- Map kolom `payload` sebagai string atau bytes yang dikirim driver sebagai teks UTF-8. `datatypes.JSON` boleh dipakai untuk `result`. Validator atau handler yang perlu membaca nilai numerik memakai token mentah atau `json.Number` sesuai schema bisnis.
- Gunakan `db.WithContext(ctx)` untuk operasi DB.
- Definisikan batas transaksi di unit-of-work/application service.
- Gunakan `RowsAffected` pada conditional update untuk membuktikan claim berhasil.
- Untuk scheduled/retry polling, gunakan `SELECT ... FOR UPDATE SKIP LOCKED` di transaksi pendek.
- Jangan memakai `AutoMigrate()` untuk production schema change; gunakan migration up/down yang ter-review.

---

## 8. Redis Streams Contract

### 8.1 Konvensi nama

```text
antriaga:queue:email
antriaga:queue:webhook
antriaga:queue:payment
antriaga:queue:report

consumer group = antriaga-workers
consumer name  = <hostname>-<process-id>-<instance-uuid>
```

Setiap queue adalah satu Redis Stream, dengan **satu consumer group per worker pool**. Jangan membuat group baru per instance; itu akan membuat setiap group menerima salinan pesan masing-masing.

### 8.2 Message payload

Pesan v1 terdiri dari field Redis Stream `event_id`, `job_id`, `queue`, `enqueue_seq`, dan `schema_version`. Semua field wajib. `event_id` adalah ID outbox, `job_id` adalah ID job, dan `queue` harus cocok dengan stream serta job di MySQL. Encode `enqueue_seq` sebagai integer desimal positif dan `schema_version` sebagai string `1`. Contoh berikut menampilkan nilai logis setelah decoding:

```json
{
  "event_id": "019a9e11-330b-7e1e-bb8a-119511778802",
  "job_id": "019a9e11-330b-7e1e-bb8a-119511778801",
  "queue": "email",
  "enqueue_seq": 1,
  "schema_version": 1
}
```

Worker selalu membaca payload, **payload_version bisnis**, **client_id owner**, status, attempts, dan lease dari MySQL. `event_id` dipakai untuk tracing dan rekonsiliasi. Redis Stream ID digunakan untuk `XACK`.

Worker dan reaper memakai decoder transport yang sama. Jika field wajib hilang, nilainya invalid, atau `schema_version` belum didukung, catat error teredaksi dan terukur dengan stream/message ID. Jangan klaim job, jalankan handler, ubah MySQL, atau ACK pesan berdasarkan envelope yang belum tervalidasi. Biarkan pesan di PEL dan lanjutkan scan kandidat lain dengan cursor. Operator dapat memperbaiki versi consumer agar pesan yang valid dibaca pada scan PEL berikutnya. Untuk envelope yang dipastikan rusak permanen, ikuti [runbook pemulihan](docs/operations/troubleshooting.md#pemulihan-envelope-redis-rusak). Error decode saja tidak mengizinkan rekonsiliasi otomatis.

### 8.3 Operasi Redis

| Command | Penggunaan |
|---|---|
| `XGROUP CREATE ... 0 MKSTREAM` | Inisialisasi consumer group saat startup dan pemulihan `NOGROUP` saat runtime |
| `XADD` | Outbox dispatcher publish ready-job message |
| `XREADGROUP` | Worker consume pesan dari group |
| `XACK` | Acknowledge setelah state DB aman/terminal atau pesan stale |
| `XPENDING` | Enumerasi PEL read-only; preflight verifikasi lease DB |
| `XCLAIM` | Claim satu/lebih PEL message dengan guard min idle setelah DB preflight |
| `XAUTOCLAIM` | Alternatif optimized scanning; ownership **langsung berpindah**, perlu DB recheck |
| `XINFO GROUPS` | Diagnostik backlog/consumer group |
| `XRANGE stream id id COUNT 1` | Periksa apakah pesan untuk outbox `published` masih ada. Gunakan Redis primary untuk keputusan rekonsiliasi |
| `XTRIM` | Retention terencana, dengan perhatian terhadap pending/unacked data |

Saat `XREADGROUP` atau scan PEL mendapat `NOGROUP`, jalankan ulang inisialisasi group untuk stream terpengaruh dengan `XGROUP CREATE <stream> <group> 0 MKSTREAM`, lalu lanjutkan loop tanpa restart proses. Group yang dibuat lebih dulu oleh instance lain (`BUSYGROUP`) dianggap siap. Jangan mereset cursor group yang sudah ada. Error lain mengikuti bounded backoff dan membuat readiness consumer/reaper degraded sampai group siap. Mulai dari `0` agar pesan yang masih ada bisa dibaca. Conditional claim MySQL tetap menahan pesan lama dan duplikat.

Contoh operasi konseptual:

```redis
XGROUP CREATE antriaga:queue:email antriaga-workers 0 MKSTREAM
XADD antriaga:queue:email * event_id 019a9e11-330b-7e1e-bb8a-119511778802 job_id 019a9e11-330b-7e1e-bb8a-119511778801 queue email enqueue_seq 1 schema_version 1
XREADGROUP GROUP antriaga-workers worker-01 COUNT 10 BLOCK 5000 STREAMS antriaga:queue:email >
XACK antriaga:queue:email antriaga-workers 1730000000000-0
XPENDING antriaga:queue:email antriaga-workers IDLE 120000 - + 20
# Setelah memeriksa lease DB, klaim ID yang eligible:
XCLAIM antriaga:queue:email antriaga-workers reaper-01 120000 1730000000000-0
```

Command yang mengubah stream, termasuk `XTRIM`, harus didesain agar tidak menghilangkan satu-satunya petunjuk recovery untuk job yang masih aktif. DB reconciler adalah pengaman tambahan, bukan alasan mengabaikan retention Redis. **Redis 7 tidak memiliki `XTRIM ... ACKED`** (fitur Redis 8.2); lihat kebijakan aman §32.

### 8.4 Acknowledgment invariant

- **Success:** DB commit `completed` + attempt result, kemudian `XACK`.
- **Permanent error / exhausted:** DB commit `dead` + attempt result, kemudian `XACK`.
- **Retryable error:** DB commit `retrying` + `scheduled_at` + attempt result, kemudian `XACK`. Scheduler nanti membuat generation baru.
- **Redelivery yang aman di-ACK:** `XACK` setelah MySQL membuktikan job terminal, generation pesan lebih lama, atau job `retrying` dengan generation sama dan `scheduled_at` sudah tersimpan. Pada `retrying`, ACK tidak memanggil handler, mengubah jadwal/counter, atau membuat outbox. Scheduler tetap merilis generation berikutnya saat due.
- **Generation sama untuk `queued` atau `processing`:** pertahankan pesan di PEL hingga worker menyimpan hasil atau reaper menyimpan keputusan recovery sesuai §29.2.
- **Lease hilang pada heartbeat/finalisasi:** kembalikan `ErrLostLease` dan jangan `XACK` pada jalur worker tersebut.
- **DB gagal commit:** **jangan** `XACK`, sehingga Redis PEL memungkinkan recovery.

---

## 9. End-to-End Data Flow

### 9.1 Create job — aman terhadap DB/Redis split

```mermaid
sequenceDiagram
    participant Client
    participant API as Fiber API
    participant DB as MySQL
    participant OUT as Dispatcher
    participant Redis
    participant Worker
    Client->>API: POST /api/v1/jobs
    API->>API: Validate job type + payload
    API->>DB: BEGIN TRANSACTION
    API->>DB: INSERT jobs(status=queued/scheduled)
    opt Immediate job
        API->>DB: INSERT outbox_events(pending, seq=1, available_at=db_now)
    end
    API->>DB: COMMIT
    API-->>Client: 202 Accepted + job ID
    OUT->>DB: Claim outbox and commit a fresh lease token
    OUT->>Redis: XADD(job ID + enqueue_seq)
    Redis-->>OUT: Stream ID
    OUT->>DB: Mark published only for the active claim
    Redis-->>Worker: XREADGROUP
```

**Rule:** never `XADD` sebelum berhasil menyimpan DB transaction. Jika DB commit sukses tetapi Redis gagal, event tetap di outbox `pending`/`publishing` dan dapat diterbitkan ulang. Jika Redis sukses tetapi penandaan `published` gagal, mungkin terjadi duplicate publish, ditangani oleh conditional worker claim.

Untuk `scheduled_at` masa depan, API menyimpan job `scheduled` **tanpa** outbox ready event; scheduler membuat outbox saat due. Untuk immediate job, API membuat job `queued` dan outbox atomik. Sebelum insert baru, terapkan admisi MySQL pada §28.2. Job terjadwal ikut memakai kapasitas job aktif sejak insert walaupun outbox belum dibuat.

#### Waktu awal event outbox siap publish

Setiap insert event `job.ready` memakai `status='pending'` dan satu timestamp `db_now = UTC_TIMESTAMP(3)` dari MySQL untuk `available_at`, `created_at`, dan `updated_at`. Terapkan aturan yang sama pada job immediate, release scheduled/retrying, recovery lease, recovery sebelum claim, rekonsiliasi, dan perbaikan operator. Insert outbox dan perubahan state job harus commit dalam satu transaksi. Scheduled job awal menunggu scheduler membuat event saat due. Dispatcher boleh mengklaim event baru pada polling pertama setelah commit. Backoff publish hanya mengubah `available_at` pada update untuk percobaan berikutnya.

#### Claim dan finalisasi dispatcher

1. Lock satu row outbox yang `pending` dan sudah due atau yang `publishing` dengan lease kedaluwarsa. Gunakan `FOR UPDATE SKIP LOCKED` untuk mengambil kandidat. Baca `UTC_TIMESTAMP(3)` dalam statement terpisah setelah lock dan periksa kembali kelayakan. Set `publishing`, simpan owner dan token unik per claim, isi expiry dari `OUTBOX_LEASE_TTL`, tambah `publish_attempts`, dan commit.
2. Jalankan `XADD` di luar transaksi dengan `OUTBOX_PUBLISH_TIMEOUT` yang positif dan lebih pendek dari TTL. Jika sukses, simpan `published`, `published_at`, dan Redis Stream ID hasil `XADD` pada `published_stream_id` dalam finalisasi atomik. Status `pending` atau `publishing` tidak memiliki ID publish durable. Jika gagal, kembalikan row ke `pending` dan set `available_at` dengan backoff publish yang dimulai dari 1 detik dan dibatasi 60 detik. Event tetap disimpan untuk percobaan berikutnya. Retry publish tidak mengubah attempt job.
3. Dalam kedua jalur finalisasi, lock row berdasarkan ID dan baca `db_now` dari MySQL setelah lock. Update hanya jika guard ini cocok, lalu kosongkan owner, token, dan expiry dalam transaksi yang sama.

```sql
WHERE id = ?
  AND status = 'publishing'
  AND lease_token = ?
  AND lease_expires_at > ?  -- db_now setelah lock
```

4. Jika guard ditolak, rollback dan catat `outbox.lease_lost`. Dispatcher lama tidak mengubah status, `published_at`, `published_stream_id`, jadwal, atau counter. Error query dan commit adalah error infrastruktur. Catat `outbox.published` hanya setelah commit finalisasi berhasil. `XADD` yang sudah diterima Redis bisa diulang. Worker mengendalikan duplikasi lewat conditional claim berdasarkan job dan generation.

### 9.2 Claim dan eksekusi job

1. Worker membaca message Redis (`XREADGROUP`) hanya ketika slot concurrency tersedia; memperoleh `job_id`, `enqueue_seq`, `stream_id`.
2. Worker menjalankan transaksi DB untuk conditional claim. Lock baris job, periksa `status=queued`, `enqueue_seq` cocok, dan `attempts < max_attempts`, lalu baca waktu MySQL setelah lock untuk menetapkan awal lease.
3. Dalam transaksi yang sama: ubah status ke `processing`, increment attempts, buat `lease_token`, atur `lease_expires_at`, insert `job_attempts(processing)`.
4. **Commit sebelum side effect eksternal.** Jika claim tidak berhasil, klasifikasi message sebagai duplicate/stale atau sementara belum eligible.
5. Jalankan handler terdaftar untuk `(job_type, payload_version)` dengan timeout dan heartbeat. Worker memakai payload dari MySQL. Worker tidak menerima plaintext API key.
6. `FinishJob` menerapkan guard lease aktif di bawah untuk semua hasil. Hasil sukses menjadi `completed`. Kegagalan sementara dengan jatah attempt tersisa menjadi `retrying`. Kegagalan permanen atau jatah habis menjadi `dead`. Simpan hasil/error dan jadwal retry sesuai status, tutup attempt yang sedang aktif, lalu kosongkan lease dalam satu transaksi.
7. Setelah commit finalisasi berhasil, tulis event transisi dan lakukan `XACK`. Jika ACK gagal, catat `redis.ack_failed`. Untuk pesan yang sudah terbukti terminal atau stale berdasarkan state durable, ikuti aturan §8.4 dan §29.2.

#### Kontrak heartbeat dan finalisasi

1. Mulai transaksi singkat dan lock baris job berdasarkan primary key dengan `SELECT ... FOR UPDATE`. Setelah lock diperoleh, jalankan `SELECT UTC_TIMESTAMP(3)` sebagai statement terpisah pada transaksi yang sama untuk mendapatkan `db_now`. Gunakan waktu itu sebagai titik keputusan. [MySQL mengevaluasi `UTC_TIMESTAMP()` di awal statement](https://dev.mysql.com/doc/refman/8.4/en/date-and-time-functions.html), sehingga timestamp dari statement yang menunggu lock dapat mendahului expiry. Jangan gunakan jam aplikasi atau timestamp dari sebelum lock.
2. Gunakan guard berikut pada update job untuk **heartbeat dan ketiga jalur finalisasi**. Bind `db_now` dari langkah 1. `lease_expires_at == db_now` sudah kedaluwarsa. Reaper dan sweeper memakai kebalikannya (`lease_expires_at <= db_now`) untuk job `processing` yang cocok dengan snapshot token dan generation.

```sql
WHERE id = ?
  AND status = 'processing'
  AND lease_token = ?
  AND lease_expires_at > ?  -- db_now setelah lock
```

3. Heartbeat menetapkan expiry baru ke `db_now + JOB_LEASE_TTL`. Untuk heartbeat, cek kelayakan dari baris yang dikunci, sehingga dua timestamp pada milidetik yang sama tidak salah diklasifikasikan sebagai lease hilang hanya karena MySQL melaporkan nol baris berubah. Finalisasi wajib memvalidasi satu job dan satu attempt yang aktif untuk job, nomor attempt, dan token yang sama. Jika update attempt gagal atau tidak cocok, rollback transaksi. Semua operasi MySQL yang menyentuh kedua baris mengambil lock job sebelum attempt. Pengecekan lease berlaku pada waktu keputusan setelah lock diperoleh. Bila lease valid saat keputusan dan transaksi commit setelah deadline, transisi tersebut tetap sah karena recovery baru dapat memeriksa state setelah lock dilepas.
4. Jika guard ditolak, rollback dan kembalikan `ErrLostLease`, termasuk ketika token masih cocok tetapi lease sudah kedaluwarsa. Hentikan heartbeat, batalkan context handler jika masih berjalan, abaikan hasil lokal, dan catat `worker.lease_lost`. Jalur ini tidak mengubah job, attempt, outbox, atau counter, serta tidak mengirim `XACK` atau event sukses/retry/dead. Reaper atau sweeper menutup attempt lama. Kegagalan query atau commit merupakan error infrastruktur dan tidak boleh dipetakan ke `ErrLostLease` atau menghasilkan ACK. Efek eksternal yang sudah diterima provider tetap memerlukan idempotency key stabil.

```mermaid
sequenceDiagram
    participant Redis
    participant W as Worker
    participant DB as MySQL
    participant E as External Service
    Redis-->>W: Message(job_id, enqueue_seq)
    W->>DB: Conditional claim + insert attempt (TX)
    alt Claim sukses
        DB-->>W: attempt_number + lease_token
        W->>E: Execute handler with stable idempotency key
        E-->>W: Success / error
        W->>DB: Lock, read MySQL time, guarded finish + attempt update (TX)
        alt Committed
            DB-->>W: Finish committed
            W->>Redis: XACK
        else Lease rejected or DB failure
            DB-->>W: ErrLostLease or infrastructure error
            Note over W,Redis: No XACK from this worker path
        end
    else Safe to ACK per section 8.4
        DB-->>W: Durable terminal, stale generation, or stored retry schedule
        W->>Redis: XACK without executing
    end
```

Pseudocode conditional claim setelah lock baris dan pembacaan `db_now` seperti di atas. Bind expiry dari `db_now + JOB_LEASE_TTL`, serta `db_now` untuk `started_at` dan `updated_at`.

```sql
UPDATE jobs
SET status = 'processing',
    attempts = attempts + 1,
    lease_owner = ?,
    lease_token = ?,
    lease_expires_at = ?,
    started_at = ?,
    updated_at = ?
WHERE id = ? AND status = 'queued'
  AND enqueue_seq = ? AND attempts < max_attempts;
```

Pastikan `RowsAffected == 1`, lalu ambil attempt number terkini dan insert attempt di **transaksi yang sama**. Saat gagal claim karena attempts habis tetapi status masih `queued`, repair state menjadi `dead` melalui transaksi agar job tidak menggantung.

### 9.3 Scheduling dan retry due

Scheduler menjalankan batch transaction:

```sql
SELECT id, status, enqueue_seq
FROM jobs
WHERE status IN ('scheduled', 'retrying')
  AND scheduled_at <= UTC_TIMESTAMP(3)
ORDER BY scheduled_at, id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

Untuk masing-masing job pada batch, dalam transaksi yang sama:

1. `enqueue_seq += 1`. Job terjadwal dibuat dengan `enqueue_seq = 1` tanpa outbox, jadi outbox ready pertamanya memakai seq 2 saat release. Job immediate memakai seq 1 pada outbox pertamanya. Setiap outbox `(job_id, enqueue_seq)` bersifat unik.
2. Ubah `status` menjadi `queued`, `scheduled_at = NULL` (atau simpan immutable scheduled time di audit event bila diperlukan).
3. Insert `outbox_events(event_type='job.ready', status='pending', enqueue_seq=<new_seq>, available_at=db_now)` dengan waktu MySQL sesuai §9.1.
4. Commit, biarkan outbox dispatcher melakukan publish.

**Tidak** menyimpan `time.Sleep` per job pada goroutine. Semua jadwal harus persist di database.

### 9.4 Retry policy

Rumus:

```text
delay = min(base_delay * 2^(attempt_number - 1), max_delay)
actual_delay = delay + uniform_random(0, delay * 0.20)
```

| Attempt gagal | Delay dasar (sebelum jitter) | Tindakan |
|---|---:|---|
| 1 | 5 detik | Retry jika masih ada kesempatan |
| 2 | 10 detik | Retry jika masih ada kesempatan |
| 3 | 20 detik | Retry jika masih ada kesempatan |
| 4 | 40 detik | Retry jika masih ada kesempatan |
| 5 | — | `dead` bila `max_attempts=5` |

Retry hanya dibuat jika `attempts < max_attempts`; `Retry-After` untuk HTTP 429 dapat dipakai sebagai batas minimal delay, lalu clamp pada konfigurasi policy yang eksplisit.

| Kegagalan | Kategori | Default |
|---|---|---|
| HTTP 429 | Retryable | Hormati `Retry-After` (dengan cap) |
| HTTP 500/502/503/504 | Retryable | Backoff |
| Network reset, transient connection error | Retryable | Backoff |
| Deadline/timeout external | Retryable | Backoff; side effect mungkin sudah terjadi |
| Invalid JSON atau schema payload | Permanent | Langsung `dead` |
| Mock payment `409 PAYMENT_IDEMPOTENCY_CONFLICT` | Permanent | Tidak melakukan capture tambahan |
| HTTP 400/401/403/404 | Umumnya permanent | Kecuali contract provider mendefinisikan pengecualian |
| Unknown job type | Permanent | `dead` + alert konfigurasi |
| Panic di handler | Retryable terbatas | Recover panic, catat, enforce max attempts |
| MySQL unavailable pada finalize | Infrastructure error | Jangan ACK; recovery, bukan retry handler langsung |

**Penting:** retry handler, broker redelivery, expired-lease recovery, dan publish reconciliation adalah empat mekanisme berbeda; jangan dicampur menjadi satu counter dan satu error flow. Gunakan event `job.retry_scheduled`, `redis.ack_failed`, `reaper.job_recovered`, dan `reconciler.job_republished` yang berbeda untuk diagnosis (Bagian 16).

### 9.5 Crash recovery / reaper — aman terhadap race

**Baseline v3 (Redis 7+):** reaper melakukan enumerasi kandidat dengan `XPENDING` (read-only), membaca job dari MySQL untuk mengecek status/generation/lease, lalu hanya menjalankan `XCLAIM` untuk pesan yang eligible dan sudah melewati `min-idle`. **Setelah XCLAIM, reaper memeriksa lagi state dalam transaksi MySQL**, karena lease dan status bisa berubah bersamaan. `XAUTOCLAIM` tetap boleh menjadi optimasi **setelah** implementasi race-safe tervalidasi; command tersebut memindahkan PEL ownership sebelum DB dapat diperiksa.

```mermaid
flowchart TD
    A[XPENDING: enumerate idle pending IDs] --> B[Load MySQL job, generation, lease]
    B --> C{Safe to ACK per section 8.4?}
    C -->|Yes| D[XACK without executing]
    C -->|No| E{DB lease active?}
    E -->|Yes| F[Skip without state change or ACK]
    E -->|No| G[XCLAIM with minimum idle condition]
    G --> H[DB TX: recheck state, lease token, generation]
    H --> I{Still recovery eligible?}
    I -->|No| J[No requeue; preserve active worker]
    I -->|Yes| K[Recover queued or expired processing per section 29.2]
    K --> L[DB commit]
    L -->|Success| M[XACK old message]
    L -->|Failure| N[No ACK; retry later]
```

- `XPENDING` idle bukan bukti worker crash. Worker sehat yang memakai long-running handler harus memperpanjang DB lease dengan heartbeat.
- `XCLAIM` tidak memberikan transaksi atomik dengan MySQL. Jika ternyata lease aktif pada DB recheck, **jangan requeue atau ACK**; worker original masih dapat menyelesaikan pekerjaan dan `XACK` ke group meskipun ownership PEL berubah.
- Setelah lock baris job, ukur ulang waktu MySQL sesuai §9.2. Recovery hanya menutup attempt `processing` yang token dan generation-nya masih cocok ketika `lease_expires_at <= db_now`. Jika attempts tersisa, commit penutupan attempt bersama generation dan outbox baru. Jika attempts habis, commit `dead` tanpa outbox baru.
- `XAUTOCLAIM` dapat digunakan untuk scan cepat tetapi tidak boleh dikombinasikan dengan asumsi bahwa ownership lama masih terjaga hingga setelah validasi DB.
- Tambahkan **DB sweeper/reconciler** untuk kasus PEL/stream tidak ada, Redis restart kehilangan write, atau queued job terjebak. Jangan mengambil alih `processing` yang lease-nya masih aktif.
- Timeout/fencing DB mencegah dua worker menyelesaikan state yang sama tetapi **tidak** menjamin side effect external tidak ganda; gunakan downstream idempotency key yang stabil.

Algoritme rinci, log event, dan race-injection test di §29. Event logging `reaper.message_reclaimed` harus berarti Redis claim benar-benar sukses, **bukan** hanya `XPENDING` candidate ditemukan.

### 9.6 Crash windows dan invariants

| Titik crash | Keadaan | Respons pemulihan |
|---|---|---|
| Sebelum DB commit create | Tidak ada job durable | Client retry create; idempotency melindungi duplikasi |
| Setelah DB commit, sebelum `XADD` | Job+outbox pending | Dispatcher publish ulang |
| Setelah `XADD`, sebelum outbox mark published | Mungkin double publish | Duplicate conditional claim aman |
| Sesudah XREADGROUP, sebelum claim DB | PEL pending, job `queued` | Commit generation dan outbox baru tanpa menaikkan `attempts`, lalu ACK pesan lama |
| Setelah DB claim, sebelum external call | Job processing, lease aktif | Lease expiry + requeue |
| Setelah external side effect, sebelum DB commit | Side effect mungkin sudah terjadi | Recovery dapat mengulang; downstream idempotency wajib |
| Setelah DB commit completed, sebelum XACK | Job terminal tapi PEL pending | Reaper ACK tanpa memanggil handler |
| Setelah DB commit retrying, sebelum XACK | Jadwal retry tersimpan dengan generation yang sama | Reaper ACK tanpa mengubah jadwal atau counter. Scheduler merilis retry saat due |
| Worker hidup tetapi heartbeat berhenti | Lease berakhir walau code masih berjalan | Fencing hasil DB, timeout, downstream idempotency |
| Redis restart dan kehilangan message | DB queued tanpa progres | Outbox/reconciler detect + republish |

---

## 10. Job Types dan Handler Registry

### 10.1 Job registry mapping

Job handlers terdaftar berdasarkan **`(job_type, payload_version)`**. `client.allowed_job_types` dapat membatasi jenis pekerjaan yang boleh diminta client; queue tetap ditentukan registry, bukan body client. Lihat §30.

| Job type | Queue | External target | Demo capability |
|---|---|---|---|
| `email.send` | `email` | Mock Mailer | 202, 400, 429, 503, timeout |
| `webhook.deliver` | `webhook` | Mock Webhook Receiver | Signature, dedup, backoff |
| `payment.capture` | `payment` | Mock Payment Gateway | Retry tidak double capture |
| `report.generate` | `report` | Report worker + MinIO | Long-running, heartbeat, result |

```go
// internal/application/ports/job_handler.go
package ports

import "context"

type JobExecution struct {
    ClientID       string
    JobID          string
    AttemptNumber  int
    Payload        []byte
    IdempotencyKey string
}

type JobHandler interface {
    Execute(ctx context.Context, execution JobExecution) ([]byte, error)
}
```

Registry memilih handler berdasarkan `(type, payload_version)`. `ClientID`, `JobID`, dan `AttemptNumber` berasal dari hasil claim MySQL yang sudah commit. `IdempotencyKey` diturunkan aturan handler; payload client tidak bisa menentukan namespace storage. Setiap handler:

1. Validasi payload spesifik.
2. Peroleh dependency external client melalui constructor injection.
3. Terapkan timeout/rate-limit sesuai target.
4. Klasifikasikan error sebagai `Retryable` atau `Permanent`.
5. Mengembalikan result metadata JSON kecil, bukan file bytes.

### 10.2 Contoh request body per job

**`email.send`**

```json
{
  "queue": "email",
  "type": "email.send",
  "payload_version": 1,
  "payload": {
    "to": "sample@example.com",
    "subject": "Welcome",
    "template": "welcome"
  },
  "max_attempts": 5
}
```

**`webhook.deliver`**

```json
{
  "queue": "webhook",
  "type": "webhook.deliver",
  "payload": {
    "event": "order.completed",
    "reference_id": "ORD-10021",
    "data": { "order_id": "ORD-10021", "total": 250000 }
  }
}
```

Target webhook berasal dari konfigurasi server. Signature mencakup byte outbound yang benar-benar dikirim. Pertahankan token angka JSON dari `data` melalui submit, penyimpanan, retry, dan pengiriman dengan `json.RawMessage` atau `json.Number`. Secret mock receiver hanya berada di konfigurasi worker.

**`payment.capture`**

```json
{
  "queue": "payment",
  "type": "payment.capture",
  "payload": {
    "payment_id": "PAY-10092",
    "order_id": "ORD-10021",
    "amount": 250000,
    "currency": "IDR"
  }
}
```

Untuk mock gateway MVP, turunkan key dari SHA-256 atas array tiga string `["antriaga.payment.capture.v1", client_id, payment_id]` dengan encoding JSON UTF-8 compact sesuai aturan escaping §27.7. Encode hash dalam hex huruf kecil. Gunakan `client_id` canonical dari claim MySQL dan `payment_id` case-sensitive dari payload yang tervalidasi. Key tetap sama pada retry, recovery, atau job lain untuk pembayaran logis yang sama dari client tersebut. Mock memeriksa dan menyimpan key, tuple tervalidasi `(order_id, amount, currency)`, dan hasil secara atomik. Key sama dengan tuple sama mengembalikan hasil awal. Key sama dengan tuple berbeda menghasilkan `409 PAYMENT_IDEMPOTENCY_CONFLICT` sebagai kegagalan permanen tanpa capture tambahan. Client berbeda dapat mempunyai `payment_id` yang sama.

**`report.generate`**

```json
{
  "queue": "report",
  "type": "report.generate",
  "payload": {
    "report_type": "monthly_sales",
    "period": "2026-10",
    "format": "csv"
  }
}
```

Worker membuat key privat `reports/{client_id}/{job_id}/{attempt_number}.csv` dari hasil claim durable di MySQL. Kontrak payload v1 menolak `output_key`. Attempt menyimpan file dengan key miliknya sendiri dan mengembalikan metadata `bucket`, `object_key`, `content_type`, `size_bytes`. Hanya finalisasi sukses dengan lease yang sah (§9.2) boleh menyimpan referensi itu sebagai `jobs.result`. Jika lease hilang atau commit belum terbukti, pertahankan object untuk rekonsiliasi. Cleanup hanya boleh menghapus object setelah attempt berhenti dan dipastikan tidak direferensikan oleh hasil yang commit. Akses file menggunakan client scope yang sama dengan pembacaan job.

---

## 11. Application Ports dan Use Cases

### 11.1 Port interfaces (contoh desain)

```go
// internal/application/ports/job_repository.go
package ports

import (
    "context"
    "time"

    "github.com/lathief/Antriaga/internal/domain"
)

type JobRepository interface {
    FindByID(ctx context.Context, clientID, id string) (*domain.Job, error)
    FindByIdempotencyKey(ctx context.Context, clientID, key string) (*domain.Job, error)
    List(ctx context.Context, clientID string, filter JobFilter) ([]domain.Job, int64, error)
}

type JobFilter struct {
    Queue  string
    Status string
    Page   int
    Limit  int
}

type Clock interface {
    Now() time.Time
}

type Message struct {
    EventID    string
    JobID      string
    Queue      string
    EnqueueSeq int64
}

type Delivery struct {
    Message  Message
    StreamID string // Redis Streams entry ID for XACK
}

type Broker interface {
    Publish(ctx context.Context, msg Message) (streamID string, err error)
    Consume(ctx context.Context, queue, consumer string, fn func(Delivery) error) error
    Ack(ctx context.Context, queue, streamID string) error
    ClaimPending(ctx context.Context, queue, consumer string, minIdle time.Duration, count int64) ([]Delivery, error)
    Close() error
}
```

**Penting:** Semua query API menerima `clientID` dari principal tervalidasi, bukan dari request bebas. Interfaces di atas adalah ilustrasi *port contract*, bukan implementasi lengkap yang siap dijalankan. `Delivery` membawa Redis Stream ID agar worker/reaper dapat meng-ACK pesan yang tepat. Detail unit-of-work dan repository transaction methods perlu ditambahkan tanpa mengekspos `*gorm.DB`.

### 11.2 Application services

| Use case | Perilaku |
|---|---|
| `AuthenticateAPIKey` | Verifikasi key HMAC, status client/key, expiry, scopes; return principal |
| `CreateClient` / `IssueAPIKey` / `RevokeAPIKey` | Admin CLI saja, audit perubahan identitas/kredensial |
| `CreateJob` | Validasi principal, type + payload_version, client-scoped dedup key, insert jobs dan outbox dalam TX |
| `GetJob` | Baca job by ID, redaksi fields sensitif |
| `ListJobs` | Filter queue/status, pagination, bounded limit |
| `GetJobAttempts` | Riwayat attempts + error codes |
| `ClaimJob` | Validate generation, conditional state transition, create lease + attempt |
| `ExecuteJob` | Handler registry `(type, payload_version)`, slot-concurrency, timeout, heartbeat, error classification |
| `FinishJob` | Guard status + token + expiry dengan waktu MySQL setelah lock (§9.2). Simpan job dan attempt dalam satu transaksi, lalu clear lease |
| `DispatchDueJobs` | Scan `scheduled/retrying`, queue + outbox secara atomik |
| `PublishOutbox` | Claim dengan token unik, XADD di luar transaksi, finalisasi kedua hasil dengan status, token, dan expiry (§9.1) |
| `RecoverStaleJobs` | Reclaim PEL, reconcile DB lease, create recovery outbox |
| `ReconcileQueuedJobs` | Periksa publish durable generation aktif dan republish hanya jika Redis primary mengonfirmasi pesan hilang (§29.4) |

`CreateJob` dan `FinishJob` adalah contoh unit-of-work lintas repository. Jangan memecah satu transaksi DB ke beberapa use case bebas tanpa satu coordinator.

### 11.3 Error types

```go
package domain

import "errors"

var (
    ErrJobNotFound        = errors.New("job not found")
    ErrUnsupportedJobType = errors.New("unsupported job type")
    ErrInvalidTransition  = errors.New("invalid job state transition")
    ErrIdempotencyConflict = errors.New("idempotency key conflict")
    ErrLostLease          = errors.New("job execution lease lost")
)

type ExecutionError struct {
    Code      string
    Retryable bool
    Cause     error
}

func (e *ExecutionError) Error() string { return e.Code }
func (e *ExecutionError) Unwrap() error { return e.Cause }
```

Jangan memasukkan raw stack trace, credentials, atau rahasia external service ke response client. Simpan error yang sudah diredaksi pada histori dan log internal.

---

## 12. HTTP API Contract (Fiber)

Base path: `/api/v1`  
Content type: `application/json`  
Auth: **wajib client-based `X-API-Key`** untuk semua endpoint `/api/v1/*`. Middleware verifikasi key dan menentukan `client_id` + scope; seluruh query owner-scoped. Admin key management via CLI, bukan REST publik (§27). RBAC manusia/UI operator ditunda fase berikutnya.

### 12.1 Endpoint MVP

| Method | Path | Status | Deskripsi |
|---|---|---|---|
| `POST` | `/api/v1/jobs` | `202` | Submit job immediate atau scheduled; scope `jobs:create` |
| `GET` | `/api/v1/jobs/:id` | `200` | Get job detail milik client; scope `jobs:read` |
| `GET` | `/api/v1/jobs` | `200` | List/filter job client tersebut + pagination; scope `jobs:read` |
| `GET` | `/api/v1/jobs/:id/attempts` | `200` | Get execution history client tersebut; scope `jobs:read` |
| `GET` | `/health/live` | `200` | Liveness process |
| `GET` | `/health/ready` | `200`/`503` | Dependencies critical ready |
| `GET` | `/metrics` | `200` | Internal Prometheus metrics |
| `GET` | `/swagger/*` | `200` | Swagger UI (non-public in production) |

**Isolasi data:** API response tidak mengekspos job milik client lain; jika ID ditemukan tetapi owner berbeda, kembalikan `404` generik. Admin CLI tidak melalui `/api/v1`.

### 12.2 Endpoint MVP+ / lanjutan

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/v1/jobs/:id/requeue` | Manual requeue (P1), scope `jobs:requeue`, owner-check, immutable history/audit |
| `GET` | `/api/v1/jobs/:id/events` | Durable timeline (P2), `jobs:read` + owner-check |
| `POST` | `/api/v1/jobs/:id/cancel` | Cancel cooperative, fase 2 |
| `GET` | `/api/v1/queues` | Queue statistics, fase 2 |
| `POST` | `/api/v1/queues/:name/pause` | Pause queue, fase 2 |
| `POST` | `/api/v1/queues/:name/resume` | Resume queue, fase 2 |

**Hindari** endpoint `POST /jobs/:id/retry` dan `requeue` bersamaan tanpa perbedaan semantik. Retry otomatis adalah kebijakan internal. Untuk operator, gunakan satu command `requeue` yang dipisahkan secara eksplisit.

### 12.3 Contoh create request

```http
POST /api/v1/jobs HTTP/1.1
Content-Type: application/json
X-API-Key: <API_KEY_DARI_SECRET_MANAGER>
Idempotency-Key: email-welcome-user-10021

{
  "type": "email.send",
  "payload_version": 1,
  "payload": {
    "to": "sample@example.com",
    "subject": "Welcome",
    "template": "welcome"
  },
  "max_attempts": 5
}
```

Queue **diturunkan dari registry**. Untuk kompatibilitas dengan request desain awal, field `queue` bisa diterima secara opsional, tetapi jika ada **harus sesuai** mapping `type -> queue`; tidak boleh client memilih arbitrary queue.

**Response:**

```json
{
  "data": {
    "id": "019a9e11-330b-7e1e-bb8a-119511778801",
    "queue": "email",
    "type": "email.send",
    "payload_version": 1,
    "status": "queued",
    "attempts": 0,
    "max_attempts": 5,
    "created_at": "2026-11-18T05:00:00Z"
  }
}
```

### 12.4 Scheduled request

```json
{
  "type": "report.generate",
  "payload_version": 1,
  "scheduled_at": "2026-11-19T01:00:00Z",
  "payload": {
    "report_type": "monthly_sales",
    "period": "2026-10",
    "format": "csv"
  }
}
```

Server menolak scheduled time yang salah format, yang tidak lebih besar dari waktu MySQL saat insert baru (`422`), atau job type yang belum didukung. Retry identik dengan `Idempotency-Key` yang sama tetap menemukan job asal setelah jadwal lewat (§27.7).

### 12.5 Pagination dan error envelope

```json
{
  "data": [],
  "meta": { "page": 1, "limit": 20, "total": 124 }
}
```

```json
{
  "error": {
    "code": "UNSUPPORTED_JOB_TYPE",
    "message": "job type is not supported",
    "request_id": "req-019a9e11"
  }
}
```

| HTTP | Kondisi |
|---:|---|
| `400` | Bad JSON, invalid field |
| `401` | Missing/invalid/revoked/expired API key atau disabled client (generic response) |
| `403` | Auth valid tetapi scope kurang |
| `404` | Job tidak ditemukan **atau bukan milik client terautentikasi** |
| `409` | Idempotency conflict atau illegal state transition |
| `413` | Payload terlalu besar |
| `422` | Valid JSON, tetapi domain validation gagal |
| `429` | Per-client rate limit atau outstanding-job limit |
| `500` | Unexpected internal error |
| `503` | Dependency unavailable/readiness failed, DB auth lookup error, atau global backpressure |

Semua request `POST` perlu timeout yang berbeda dari timeout eksekusi handler; HTTP tidak menunggu job selesai.

---

## 13. Swaggo / Swagger Documentation

### 13.1 Instalasi dependency

```bash
go mod init github.com/lathief/Antriaga

go get github.com/gofiber/fiber/v3
go get github.com/gofiber/contrib/v3/swaggo
go get github.com/swaggo/swag
go get gorm.io/gorm gorm.io/driver/mysql
go get github.com/redis/go-redis/v9

go install github.com/swaggo/swag/cmd/swag@v1.16.6
```

Pin versi sejak awal. Per November 2026, Fiber v3.7.0 dan `github.com/gofiber/contrib/v3/swaggo` mensyaratkan **Go 1.26.0**, dengan `swag` v1.16.6 dan `swaggo/files/v2` v2.0.2. Verifikasi ulang `go.mod` upstream sebelum menaikkan versi, dan jangan memakai `@latest` pada workflow/Makefile CI agar generated docs reproducible.

### 13.2 General API annotation

Di `cmd/api/main.go`:

```go
// @title Antriaga API
// @version 1.0
// @description Distributed background job processing API.
// @BasePath /api/v1
// @schemes http https
// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name X-API-Key
package main
```

Alternatif: pakai file `cmd/api/docs.go` yang dipilih eksplisit oleh `swag init -g ...`. Pastikan annotation general benar-benar ditemukan oleh generator.

### 13.3 Annotation handler

```go
// CreateJob godoc
// @Summary Submit a background job
// @Description Persist a job for the authenticated client and schedule asynchronous processing.
// @Tags Jobs
// @Accept json
// @Produce json
// @Param Idempotency-Key header string false "Optional deduplication key"
// @Param request body dto.CreateJobRequest true "Job creation payload"
// @Success 202 {object} dto.JobResponse
// @Failure 400 {object} dto.ErrorResponse
// @Failure 409 {object} dto.ErrorResponse
// @Failure 422 {object} dto.ErrorResponse
// @Failure 500 {object} dto.ErrorResponse
// @Security ApiKeyAuth
// @Router /jobs [post]
func (h *JobHandler) CreateJob(c fiber.Ctx) error {
    // Bind DTO -> validate -> application use case -> response.
    return nil // illustrative: replace with real implementation
}
```

Di DTO, `payload_version` wajib terdokumentasi dan `client_id` **tidak** menjadi parameter terpercaya. Dokumentasi setiap endpoint menyatakan scope yang dibutuhkan (meski Swagger 2.0 security definition API key tidak otomatis mengekspresikan scopes). `dto.CreateJobRequest`, `dto.JobResponse`, dan `dto.ErrorResponse` harus benar-benar didefinisikan di `adapters/inbound/http/dto`. Dokumentasi response/field mengikuti contract HTTP, **bukan model database**.

### 13.4 Generate Swagger spec

```bash
# Dari project root
swag init -g cmd/api/main.go -d ./ --parseInternal -o ./docs
```

Output umumnya `docs/docs.go`, `docs/swagger.json`, dan `docs/swagger.yaml`. Setelah generate, import package `docs` agar spec terdaftar dalam binary.

### 13.5 Mount Swagger UI di Fiber v3

```go
package main

import (
    "github.com/gofiber/fiber/v3"
    swaggo "github.com/gofiber/contrib/v3/swaggo"

    _ "github.com/lathief/Antriaga/docs"
)

func main() {
    app := fiber.New()
    // Register production routes and middleware via bootstrap.
    app.Get("/swagger/*", swaggo.HandlerDefault)
    if err := app.Listen(":8080"); err != nil {
        panic(err)
    }
}
```

Akses: `http://localhost:8080/swagger/index.html`. Batasi akses atau matikan route UI untuk deployment public. Jika path middleware diubah, sesuaikan dengan dokumentasi versi dependency yang sudah dipin.

**Batasan:** Swaggo `swaggo/swag` menghasilkan **Swagger 2.0**, bukan janji otomatis OpenAPI 3.0. Jika nanti butuh OpenAPI 3, jadikan migrasi tooling sebagai ADR tersendiri.

---

## 14. Fiber Handler, Routing, dan Validation

### 14.1 Contoh route registration

```go
func RegisterRoutes(app *fiber.App, h *JobHandler, health *HealthHandler) {
    v1 := app.Group("/api/v1")
    v1.Post("/jobs", h.CreateJob)
    v1.Get("/jobs", h.ListJobs)
    v1.Get("/jobs/:id", h.GetJob)
    v1.Get("/jobs/:id/attempts", h.GetAttempts)

    app.Get("/health/live", health.Live)
    app.Get("/health/ready", health.Ready)
}
```

Handler Fiber v3 menerima `fiber.Ctx` (bukan `*fiber.Ctx` Fiber v2). Parsing body menggunakan `c.Bind().Body(&request)`; konversi DTO ke use case command tetap dilakukan di adapter.

### 14.2 Inbound validation rules

- `type` harus salah satu registered types.
- Payload JSON harus valid dan memenuhi schema untuk jenis job tersebut.
- `max_attempts` berada dalam batas konfigurasi, misalnya `1..10`.
- `scheduled_at`, jika tersedia, harus RFC3339/UTC normalized.
- Terapkan panjang, charset, dan canonicalization `Idempotency-Key` sesuai §27.7.
- `page>=1`, `1<=limit<=100` untuk list.
- Whitelist status dan queue untuk filter.
- Batasi body size maksimal 256 KiB untuk job request.
- Jangan memasukkan header/secret authentication ke `payload` yang dipersist.

**Separation:** validation syntax di DTO adapter; rule type/queue mapping dan state invariants juga divalidasi di use case/domain.

### 14.3 Middleware

Request ID, structured access logging (event `http.request_completed`), panic recovery, **client-based API key middleware + scope middleware + per-client rate limit (P0)**, request body limit, timeout yang sesuai, dan standar error mapper. `GET /health/live` tidak boleh bergantung pada MySQL. Karena job submission hanya membutuhkan MySQL + outbox, **API readiness bergantung pada MySQL dan status draining**. Redis yang putus dilaporkan sebagai degraded. Saat kapasitas MySQL tercapai, submit job baru ke `POST /api/v1/jobs` mengembalikan `503 SYSTEM_BACKPRESSURE`; pembacaan dan replay key yang sudah tercatat tetap memakai MySQL. Worker dan dispatcher memerlukan Redis untuk delivery.

---

## 15. Unit Testing & Test Strategy

### 15.1 Testing pyramid

| Jenis | Scope | Dependencies | Target |
|---|---|---|---|
| Unit domain | State transitions, retry, validation | None | Banyak, cepat |
| Unit application | Use cases, transaction boundaries, errors | Fake ports/mock | Banyak, deterministik |
| HTTP handler | Routing, DTO, errors, Swagger routes | Fiber `app.Test`; fake use cases | Tiap endpoint |
| Integration | GORM repository, MySQL transactions, Redis group | Testcontainers MySQL/Redis | DB/Redis behavior nyata |
| End-to-end | API -> outbox -> Redis -> worker -> mock | Docker Compose/Testcontainers | Happy path + failure injection |
| Performance | 1k–10k jobs, throughput, latency, lag | Separate environment | Baseline terukur |

### 15.2 Unit test: retry policy

Contoh fungsi pure yang mudah diuji:

```go
package execution

import "time"

// Backoff is deliberately deterministic; inject jitter separately.
func Backoff(attempt int, base, maximum time.Duration) time.Duration {
    if attempt < 1 {
        attempt = 1
    }
    delay := base
    for i := 1; i < attempt; i++ {
        if delay >= maximum/2 {
            return maximum
        }
        delay *= 2
    }
    if delay > maximum {
        return maximum
    }
    return delay
}
```

```go
package execution

import (
    "testing"
    "time"

    "github.com/stretchr/testify/require"
)

func TestBackoff(t *testing.T) {
    tests := []struct {
        name    string
        attempt int
        want    time.Duration
    }{
        {"first", 1, 5 * time.Second},
        {"second", 2, 10 * time.Second},
        {"third", 3, 20 * time.Second},
        {"clamped", 20, 15 * time.Minute},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Backoff(tt.attempt, 5*time.Second, 15*time.Minute)
            require.Equal(t, tt.want, got)
        })
    }
}
```

Untuk jitter, inject interface `Random` atau seed terkontrol; jangan mengandalkan randomness global dalam unit test. Untuk scheduling, inject `Clock` sehingga time-dependent tests deterministik.

### 15.3 Unit test: `CreateJob` use case

Mock/fake ports harus bisa memastikan:

1. Unsupported job type ditolak **sebelum** persist.
2. Immediate job membuat row job dan outbox dalam satu transaction boundary.
3. Scheduled job membuat row job tetapi **belum** ready outbox.
4. Jika insert outbox gagal, transaksi rollback job.
5. Idempotency key sama + request hash sama mengembalikan existing job.
6. Idempotency key sama + request hash beda menghasilkan `409`.
7. DB error dikembalikan dan tidak dilaporkan sebagai `202`.
8. Tidak ada call Redis langsung pada path HTTP create.

### 15.4 Unit test: worker / state machine

| Test | Kondisi | Hasil diharapkan |
|---|---|---|
| `ClaimQueuedJob` | Status queued + seq cocok | Processing + attempt baru |
| `IgnoreDuplicateMessage` | Attempt masih processing | Tidak menjalankan handler dua kali |
| `RejectOldGeneration` | Message seq lebih rendah | Stale message di-ACK secara aman |
| `CompleteWithValidLease` | Token benar dan expiry setelah waktu keputusan DB | Completed |
| `RejectExpiredLeaseBeforeRecovery` | Token masih benar, waktu keputusan sama dengan atau melewati expiry | `ErrLostLease` untuk heartbeat dan finalisasi sukses/retry/dead. Job dan attempt tidak berubah, tanpa ACK |
| `RejectLateWorkerResult` | Token lama setelah recovery | `ErrLostLease`, DB state tidak berubah, tanpa ACK |
| `RetryTemporaryFailure` | 503, attempts tersisa | Retrying + scheduled_at |
| `DeadPermanentFailure` | Payload invalid | Dead, tidak retry |
| `DeadAfterMaxAttempts` | Attempts habis | Dead |
| `SchedulerConcurrency` | Dua scheduler | Satu generation baru saja |
| `DispatcherDoublePublish` | Publish sukses, mark gagal | Conditional claim tetap satu worker |

### 15.5 Fiber handler tests

Gunakan `httptest.NewRequest()` dengan `app.Test()` dari Fiber v3. Periksa status, headers, JSON envelope, auth, validasi DTO, dan edge case. Contoh minimal smoke test:

```go
func TestLiveEndpoint(t *testing.T) {
    app := fiber.New()
    app.Get("/health/live", func(c fiber.Ctx) error {
        return c.SendStatus(fiber.StatusOK)
    })

    req := httptest.NewRequest(http.MethodGet, "/health/live", nil)
    res, err := app.Test(req)
    require.NoError(t, err)
    defer res.Body.Close()
    require.Equal(t, http.StatusOK, res.StatusCode)
}
```

Test file memerlukan imports standar (`net/http`, `net/http/httptest`, `testing`) dan Fiber/testify. Kode di atas adalah contoh pola test, bukan file test lengkap.

### 15.6 Integration testing dengan Testcontainers

- Start MySQL 8.4 dan Redis 7 container terisolasi per suite/kelas pengujian.
- Terapkan SQL migrations sesungguhnya ke container MySQL.
- Test `FOR UPDATE SKIP LOCKED` untuk dua scheduler/dispatcher serentak.
- Test unique idempotency key dan transaction rollback.
- Uji bootstrap Compose dari volume kosong dan migrasi gagal. API tidak mulai jika migrasi gagal, dan jalur submit tetap tersedia setelah migrasi sukses saat Redis sengaja tidak dijalankan.
- Uji event outbox baru dari immediate job, release scheduler, dan recovery. Masing-masing harus langsung bisa diklaim dispatcher setelah commit tanpa menunggu grace period.
- Test outbox crash: publish sukses tetapi mark `published` sengaja gagal.
- Uji dispatcher A tertahan hingga expiry, lalu dispatcher B mengklaim row dengan token baru. Finalisasi sukses dan gagal dari A ditolak termasuk jika owner sama. Uji juga token masih cocok tetapi lease sudah lewat atau tepat pada expiry. Jika XADD A sudah diterima dan B mempublish ulang, verifikasi hanya satu claim worker untuk generation itu.
- Test Redis `XGROUP`, `XREADGROUP`, `XACK`, `XPENDING`, guarded `XCLAIM`, dan optional `XAUTOCLAIM` optimization.
- Test worker claim saat dua consumers menerima duplicate generation.
- Uji round-trip dari dispatcher nyata ke decoder worker melalui Redis Streams. Verifikasi lima field wajib, versi `1`, serta penolakan field hilang atau versi lain tanpa claim atau ACK. Gunakan contoh `XADD` §8 sebagai fixture kontrak.
- Test recovery ketika worker crash sebelum XACK.
- Jalankan matriks lease pada §29.3 terhadap MySQL yang sesungguhnya. Gunakan fixture timestamp dan koordinasi transaksi agar pengujian tidak bergantung pada `sleep`.
- Gunakan Redis AOF pada e2e konfigurasi untuk memeriksa restart persistence secara terpisah.

### 15.7 End-to-end acceptance scenarios

| ID | Skenario | Acceptance |
|---|---|---|
| E2E-01 | Email success | 202 -> queued -> processing -> completed |
| E2E-02 | Webhook 503 lalu 200 | Retry terjadi; attempts tercatat |
| E2E-03 | Email 400 invalid | Dead tanpa retry berulang |
| E2E-04 | Payment response hilang setelah capture. Ulangi dengan client lain pada payment ID yang sama, dan tuple berbeda dengan key yang sama | Satu capture per client dan payment ID. Retry identik mendapat hasil awal. Tuple berbeda memberi conflict permanen |
| E2E-04b | Webhook membawa integer besar dan desimal berpresisi tinggi | Bandingkan token angka dari submit, baca MySQL, dan body outbound, termasuk retry |
| E2E-05 | Worker dibunuh setelah claim | Lease expiry -> recovery -> terminal |
| E2E-06 | Worker mati setelah result DB commit, sebelum ACK | PEL reclaimed dan ACK tanpa re-execute |
| E2E-07 | Redis down saat create job | MySQL create masih aman; outbox publish saat Redis pulih |
| E2E-08 | Dua worker + duplicate message | Satu DB claim aktif per generation |
| E2E-09 | Handler berjalan melewati expiry awal dengan heartbeat aktif | Gunakan TTL dan interval khusus test, tahan handler dengan barrier, lalu verifikasi expiry maju, reaper melewati job, dan worker menyelesaikan attempt yang sama |
| E2E-10 | Scheduled job | Tidak di-execute sebelum due; eventually processed |
| E2E-11 | Restart API dan worker | Status/histori tetap tersimpan |
| E2E-12 | Load 1.000 job, scale 1 -> 5 workers | Throughput terukur dan queue lag menurun saat kapasitas cukup |

### 15.8 Logging assertions pada unit/integration/E2E tests

Tambahkan pengecekan log seperti yang dirinci dalam **§16.8**: event retry/dead, XACK fail, commit fail, recovered lease, sanitized errors, dan correlation `request_id -> job_id`. Pada E2E, gunakan log JSON sebagai bukti observability **di samping** verifikasi state di MySQL/Redis, bukan sebagai satu-satunya assertion correctness.

### 15.9 Test commands dan coverage

```bash
go test ./... -count=1
go test ./... -race -count=1
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out

go test ./tests/integration/... -tags=integration -count=1
go test ./tests/e2e/... -tags=e2e -count=1
```

Gunakan build tag yang konsisten pada file integration/e2e. Coverage threshold awal yang masuk akal: **domain + application >= 80% line coverage**, tetapi jangan mengejar angka dengan mengorbankan failure-path coverage. Integration/E2E harus tetap menjadi gate untuk fitur concurrency dan transactional safety.

### 15.10 Security, capacity, versioning, dan race-condition tests (v3)

- **API key:** generate/verify HMAC, key ID uniqueness, revocation, expiry, disabled client, scope, fail-closed saat MySQL unavailable, redaction, dan rotasi tanpa downtime.
- **Client isolation:** dua client dengan `Idempotency-Key` sama tidak konflik, cross-client GET/list/attempts/events/requeue tidak membaca/mengubah data pihak lain.
- **Backpressure:** uji outage Redis dengan job langsung dan terjadwal, submit serentak pada batas per client, replay idempoten ketika antrean penuh, dan pemulihan dispatcher. Property test memastikan `active_handlers <= configured_concurrency`, `XREADGROUP COUNT <= free_slots`; overload menghasilkan `429`/`503` sesuai kebijakan.
- **Recovery races:** heartbeat bersamaan PEL scan, dua reaper claim sama, DB commit gagal setelah Redis claim, late result dari stale worker (lihat §29.3).
- **Payload versioning:** handler v1/v2 coexist, unsupported submit ditolak, dan backlog v1 bertahan selama rolling deploy (lihat §30).
- **P1/P2 tests:** dead-job requeue idempotent, safe cleanup FK/PEL, dan transactional event timeline tidak menulis false success.

---

## 16. Observability & Detailed Process Logging

Observability MVP terdiri atas **structured application logs + durable job/attempt/outbox state di MySQL + Prometheus metrics**. Tujuan logging: seorang developer atau operator dapat menjawab *job diterima kapan, dipublish ke Redis kapan, worker mana yang mengeksekusi, attempt ke berapa, mengapa gagal, kapan retry, dan bagaimana job pulih setelah crash* hanya dengan menelusuri `job_id`.

**Keputusan:** gunakan `log/slog` (Go standard library), output JSON satu baris per event ke **stdout**, dan logger yang di-inject ke API, worker, scheduler, dispatcher, serta reaper. Domain **tidak** mengerjakan I/O logging. Structured log bukan database transaksi dan **bukan** pengganti `jobs`, `job_attempts`, atau `outbox_events`.

### 16.1 Aturan logging, level, dan konsistensi

| Level | Kapan dipakai | Contoh event |
|---|---|---|
| `DEBUG` | Detail diagnostik volume tinggi; diaktifkan sementara | `redis.message_received`, `redis.ack_succeeded`, `scheduler.batch_empty` (opsional) |
| `INFO` | Milestone lifecycle yang berhasil/berubah state secara nyata | `job.accepted`, `job.claimed`, `job.completed`, `outbox.published`, `scheduler.job_released` |
| `WARN` | Gangguan yang bisa pulih atau anomali tanpa kehilangan invariant | `job.attempt_failed`, `job.retry_scheduled`, `worker.lease_lost`, `redis.ack_failed`, `outbox.publish_failed` |
| `ERROR` | Kegagalan terminal atau kegagalan infrastruktur serius yang perlu investigasi | `job.dead`, `db.commit_failed`, `reaper.recovery_failed` |

Aturan implementasi:

1. Setiap log mempunyai **event name stabil** dalam format `subject.action` (misalnya `job.retry_scheduled`). Teks `msg` dibuat manusiawi, sedangkan `event` digunakan untuk filtering/alert.
2. Event `job.accepted`, `job.completed`, `job.dead`, `job.retry_scheduled`, `scheduler.job_released`, dan `reaper.job_recovered` hanya ditulis **setelah transaksi MySQL berhasil commit**. Gagal commit berarti log kegagalan, **bukan** log transisi berhasil. Jangan membuat log sukses sebelum side effect durable.
3. Logging adalah **best effort**: log sink yang gagal tidak membuat DB transaction di-rollback, dan tidak boleh memblokir worker tanpa batas. Metrics + durable state tetap menjadi sumber pengamatan tambahan.
4. Hindari satu log per polling kosong, per heartbeat normal, atau per message ACK sukses di production `INFO`; gunakan counter/metrics, `DEBUG` atau periodic aggregated summary. Selalu log lease loss, publish failure, retry, dead, serta recovery penting.
5. Untuk satu kegagalan, bedakan: error **handler**, kegagalan **database commit**, kegagalan **Redis ACK**, dan **worker crash recovery**; jangan melabeli semuanya `job.retry_scheduled`.
6. Panic ditangkap di worker boundary; catat event `worker.handler_panic` dengan error code teredaksi dan identitas job; setelah itu ikuti klasifikasi dan retry policy. Jangan mengirim stack trace mentah ke klien.
7. Semua timestamp di UTC (RFC3339). Gunakan durasi dalam **milidetik**, jumlah attempt sebagai integer, boolean sebagai boolean, dan logger records satu baris JSON.
8. Pada produksi, `LOG_LEVEL=info` sebagai default. Gunakan `debug` sementara hanya di lingkungan yang akses lognya terlindungi.

### 16.2 Standar schema JSON (wajib dan opsional)

`log/slog` JSON handler menyediakan `time`, `level`, `msg`. Tambahkan field berikut secara konsisten:

| Field | Tipe | Sumber / aturan |
|---|---|---|
| `event` | string | Kode tetap, contoh `job.retry_scheduled` — **wajib** untuk lifecycle events |
| `service` | string | `antriaga` dari bootstrap |
| `component` | string | `api`, `worker`, `scheduler`, `dispatcher`, `reaper` |
| `environment` | string | `local`, `staging`, `production` |
| `version` | string | App release/commit SHA jika tersedia |
| `instance_id` | string | Identitas instance proses, **bukan** data user |
| `request_id` | string | Hanya saat request HTTP dan event yang memperoleh konteks tersebut |
| `client_id` | string | Identity owner dari API key / persisted job; aman untuk log akses terkendali |
| `api_key_id` | string | Credential identifier non-secret, hanya untuk auth/audit event; tidak pernah plaintext key |
| `job_id` | string | **Primary correlation key** lintas seluruh background-job lifecycle |
| `queue`, `job_type` | string | Nama queue / registered job type |
| `event_id` | string | ID intent outbox ketika relevan |
| `stream_id` | string | ID pesan Redis Streams untuk consume/ACK/reclaim |
| `enqueue_seq` | int64 | Generation message untuk mendeteksi stale/duplicate |
| `attempt_number`, `max_attempts` | int | Percobaan ke berapa dan batas total |
| `worker_id` | string | Pemilik execution attempt |
| `lease_token` | string | **Jangan log token mentah**; gunakan `lease_status` atau `lease_generation` bila perlu |
| `from_status`, `to_status` | string | State transition yang benar-benar di-commit |
| `error_code`, `error_kind` | string | Kategori aman (`UPSTREAM_UNAVAILABLE`, `retryable`, `permanent`, `infrastructure`) |
| `http_status` | int | HTTP status dari mock/provider bila aman dan tersedia |
| `retry_delay_ms` | int64 | Delay backoff aktual setelah jitter |
| `next_run_at` | string | Jadwal retry/delayed execution UTC |
| `duration_ms` | int64 | Durasi kerja/attempt; definisi harus konsisten |
| `trace_id` | string | Opsional jika OpenTelemetry ditambahkan nanti |

**Korelasi lintas proses:** API dapat mengeluarkan `request_id`; setelah HTTP response selesai konteks itu tidak otomatis ikut ke worker. Gunakan `job_id` untuk mengikuti seluruh lifecycle, plus `event_id`, `enqueue_seq`, dan `stream_id` untuk delivery. Jika ingin relasi request ke worker, cari `job.accepted` yang menyimpan **kedua** `request_id` dan `job_id` dalam log API. Jangan mengisi `request_id` worker dengan nilai rekaan.

**Data sensitif:** jangan log `payload`, `result` penuh, email penerima, nama pelanggan, payment ID mentah, `Authorization`, `X-API-Key`, password, API key, DSN, signed URL, full webhook body, signature, idempotency key, atau error string dari provider secara mentah. Jika perlu, gunakan hash/fingerprint yang tak dapat dibalik dan sudah ditinjau kebutuhannya. `error_code` dan pesan ringkas yang telah disanitasi lebih aman daripada `err.Error()` mentah.

### 16.3 Event catalog untuk seluruh proses

| Komponen | Event | Level default | Makna, termasuk titik emit |
|---|---|---|---|
| API | `http.request_completed` | INFO | Satu log di akhir request: method, route template, status, latency, request_id; filter health/metrics bila noisy |
| API | `auth.key_rejected` | WARN | Missing/expired/revoked/invalid key; tanpa raw token atau secret |
| API | `auth.scope_denied` | WARN | Key valid, scope tidak cukup, sertakan client_id + key_id |
| Admin CLI | `api_key.created`, `api_key.revoked` | INFO/WARN | Setelah transaksi credential sukses, tanpa secret/digest |
| API | `job.rejected` | WARN | Input invalid, unsupported type, idempotency conflict; tanpa data sensitif |
| API | `job.accepted` | INFO | Job + immediate outbox sudah commit, response siap `202` |
| API | `job.deduplicated` | INFO | Idempotent request berulang dipetakan ke existing job, tanpa membuat job baru |
| API | `db.commit_failed` | ERROR | Commit create/finish gagal; **jangan** mengklaim job tersimpan/selesai |
| Dispatcher | `outbox.claimed` | DEBUG | Mengambil outbox row untuk publish |
| Dispatcher | `outbox.published` | INFO | `XADD` sukses dan mark `published` di DB sukses; sertakan event_id, stream_id |
| Dispatcher | `outbox.publish_failed` | WARN | Redis/XADD gagal, intent masih ada untuk retry (berdasarkan state DB) |
| Dispatcher | `outbox.lease_lost` | WARN | Guard status, token, atau expiry menolak finalisasi tanpa mengubah outbox |
| Dispatcher | `outbox.mark_failed` | ERROR | `XADD` mungkin berhasil tetapi penandaan `published` gagal; **duplikasi mungkin terjadi** |
| Worker | `redis.message_received` | DEBUG | Message diterima dari Redis, belum berarti DB claim berhasil |
| Worker | `job.claimed` | INFO | Conditional DB claim berhasil; attempt + lease sudah commit |
| Worker | `job.started` | DEBUG | Handler mulai, bukan pengganti event `job.claimed` |
| Worker | `job.attempt_failed` | WARN | Handler gagal; catat error_kind/error_code dan attempt_number |
| Worker | `job.retry_scheduled` | WARN | DB commit `processing -> retrying`, catat delay dan `next_run_at` |
| Worker | `job.completed` | INFO | DB commit `completed` sukses, catat durasi |
| Worker | `job.dead` | ERROR | DB commit `dead` karena permanent error atau attempts habis |
| Worker | `worker.handler_panic` | ERROR | Panic handler tertangkap; log aman, kemudian ikuti policy bounded retry |
| Worker | `worker.lease_lost` | WARN | Guard status/token/expiry menolak hasil atau heartbeat. Jangan log transisi, dan biarkan recovery berjalan |
| Worker | `redis.ack_failed` | WARN | DB sudah aman tetapi `XACK` gagal; reaper/reconciliation kelak menangani |
| Worker | `redis.ack_succeeded` | DEBUG | ACK berhasil, opsional untuk diagnosa |
| Scheduler | `scheduler.job_released` | INFO | DB commit due `scheduled/retrying -> queued` dan outbox baru |
| Scheduler | `scheduler.batch_failed` | ERROR | Query/transaction scheduler gagal |
| Reaper | `reaper.pending_examined` | DEBUG | `XPENDING` menemukan kandidat, **belum** pindah ownership |
| Reaper | `reaper.message_reclaimed` | INFO | `XCLAIM` berhasil mengalihkan ownership pesan, **belum** berarti DB job direqueue |
| Reaper | `reaper.active_lease_skipped` | DEBUG | PEL idle tetapi lease MySQL masih aktif; jangan ambil alih |
| Reaper | `reaper.job_recovered` | WARN | Lease expired; DB commit old attempt closed + next generation outbox |
| Reaper | `reaper.job_dead` | ERROR | Recovery memutuskan terminal karena attempts habis |
| Reaper | `reaper.stale_message_acked` | DEBUG | Event obsolete/terminal di-ACK tanpa menjalankan handler ulang |
| Reaper | `reaper.recovery_failed` | ERROR | Klaim PEL/DB recovery gagal; kirim metrics/alert |
| Reconciler | `reconciler.job_republished` | WARN | Stuck queued job dibuat generation/outbox baru setelah verifikasi DB |
| Semua | `component.started`, `component.stopping`, `component.stopped` | INFO | Proses startup, menerima shutdown, selesai drain; sertakan service/component/instance_id |

Event success ber-volume besar dapat di-sample *hanya bila* metrik agregat tetap akurat dan informasi audit durable (`jobs`/`job_attempts`) tetap tersimpan. Event error, `dead`, retry, lease lost, dan recovery jangan di-sample secara acak pada MVP.

### 16.4 Contoh timeline log untuk satu job gagal lalu berhasil

Berikut JSONL ilustratif. Semua log dengan `job_id` yang sama dapat digabung meskipun berasal dari container berbeda. Nilai `msg` sengaja ringkas; `event` adalah contract yang stabil.

```jsonl
{"time":"2026-11-18T05:00:00Z","level":"INFO","msg":"Job accepted after MySQL commit","event":"job.accepted","service":"antriaga","component":"api","request_id":"req-123","job_id":"job-abc","queue":"webhook","job_type":"webhook.deliver","enqueue_seq":1}
{"time":"2026-11-18T05:00:00.080Z","level":"INFO","msg":"Outbox message published","event":"outbox.published","service":"antriaga","component":"dispatcher","job_id":"job-abc","event_id":"evt-1","stream_id":"1794978000080-0","enqueue_seq":1}
{"time":"2026-11-18T05:00:00.100Z","level":"INFO","msg":"Job claimed by worker","event":"job.claimed","service":"antriaga","component":"worker","job_id":"job-abc","worker_id":"worker-01","attempt_number":1,"max_attempts":5,"enqueue_seq":1}
{"time":"2026-11-18T05:00:00.380Z","level":"WARN","msg":"Job handler returned retryable error","event":"job.attempt_failed","service":"antriaga","component":"worker","job_id":"job-abc","attempt_number":1,"error_code":"UPSTREAM_UNAVAILABLE","error_kind":"retryable","http_status":503}
{"time":"2026-11-18T05:00:00.390Z","level":"WARN","msg":"Retry scheduled after MySQL commit","event":"job.retry_scheduled","service":"antriaga","component":"worker","job_id":"job-abc","attempt_number":1,"from_status":"processing","to_status":"retrying","retry_delay_ms":5500,"next_run_at":"2026-11-18T05:00:05.890Z"}
{"time":"2026-11-18T05:00:06.010Z","level":"INFO","msg":"Retry job released to queue","event":"scheduler.job_released","service":"antriaga","component":"scheduler","job_id":"job-abc","from_status":"retrying","to_status":"queued","enqueue_seq":2}
{"time":"2026-11-18T05:00:06.075Z","level":"INFO","msg":"Outbox message published","event":"outbox.published","service":"antriaga","component":"dispatcher","job_id":"job-abc","event_id":"evt-2","stream_id":"1794978006075-0","enqueue_seq":2}
{"time":"2026-11-18T05:00:06.100Z","level":"INFO","msg":"Job claimed by worker","event":"job.claimed","service":"antriaga","component":"worker","job_id":"job-abc","worker_id":"worker-02","attempt_number":2,"max_attempts":5,"enqueue_seq":2}
{"time":"2026-11-18T05:00:06.420Z","level":"INFO","msg":"Job completed after MySQL commit","event":"job.completed","service":"antriaga","component":"worker","job_id":"job-abc","attempt_number":2,"duration_ms":320,"from_status":"processing","to_status":"completed"}
```

Urutan asinkron nyata tidak selalu persis sama dengan contoh (scheduling lag dan urutan proses berbeda mungkin terjadi); gunakan `job_id` dan data state, bukan mengasumsikan timestamp lintas host sebagai urutan transaksi global.

**Contoh lain, terminal failure (tidak ada retry):**

```json
{"time":"2026-11-18T05:10:01Z","level":"ERROR","msg":"Job moved to dead state","event":"job.dead","component":"worker","job_id":"job-def","job_type":"email.send","attempt_number":1,"max_attempts":5,"error_code":"INVALID_EMAIL","error_kind":"permanent","from_status":"processing","to_status":"dead"}
```

**Contoh Redis unavailable setelah API menerima job:**

```json
{"time":"2026-11-18T05:12:02Z","level":"WARN","msg":"Outbox publish failed; retry pending","event":"outbox.publish_failed","component":"dispatcher","job_id":"job-ghi","event_id":"evt-3","error_code":"REDIS_UNAVAILABLE","error_kind":"infrastructure","publish_attempts":1}
```

### 16.5 Desain implementasi `log/slog` dalam Clean Architecture

Tempatkan factory di `internal/platform/logger/logger.go` dan bangun satu logger di **composition root** (`internal/bootstrap/*`) untuk masing-masing proses. Logger memiliki atribut statis (`service`, `component`, `environment`, `version`, `instance_id`). Use case dapat menerima `*slog.Logger` melalui constructor karena `slog` adalah standard library Go, atau gunakan `application/ports/logger.go` jika ingin memock interface tipis. Domain entity **tidak** memanggil logger global.

Tambahkan file berikut pada struktur repository bagian 5:

```text
internal/platform/logger/
├── logger.go         # JSON slog factory, level, static metadata
├── redact.go         # Safe-field policy dan defense-in-depth redaction
├── logger_test.go    # Format, level filtering, secret redaction
└── events.go         # Constants event-name, shared field conventions
internal/adapters/inbound/http/middleware/
└── access_log.go     # Request ID, sanitized HTTP access event
```

Contoh implementasi factory (snippets bersifat desain yang dapat dikembangkan):

```go
// internal/platform/logger/logger.go
package logger

import (
    "log/slog"
    "os"
    "strings"
)

func New(service, component, environment, version, instanceID string, level slog.Level) *slog.Logger {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: level,
        ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
            switch strings.ToLower(a.Key) {
            case "authorization", "x-api-key", "api_key", "password", "secret",
                "token", "lease_token", "idempotency_key", "payload",
                "request_body", "dsn", "payment_id", "email", "signature":
                return slog.String(a.Key, "[REDACTED]")
            default:
                return a
            }
        },
    })
    return slog.New(handler).With(
        "service", service,
        "component", component,
        "environment", environment,
        "version", version,
        "instance_id", instanceID,
    )
}
```

`ReplaceAttr` hanya membantu untuk **nama field yang dikenal**; ini tidak membersihkan PII yang sudah dimasukkan ke `msg`, nested arbitrary strings, atau `error_message`. Karena itu **jangan** log payload, URL berisi token, header mentah, atau `err.Error()` dari provider. Aturan utama tetap allowlist field yang aman pada titik logging.

Contoh helper emit event setelah state retry **sudah commit**:

```go
// Dipanggil oleh application service setelah transaksi FinishJob berhasil.
func LogRetryScheduled(
    ctx context.Context,
    log *slog.Logger,
    jobID string,
    attempt int,
    code string,
    delay time.Duration,
    nextRun time.Time,
) {
    log.WarnContext(ctx, "Retry scheduled after MySQL commit",
        "event", "job.retry_scheduled",
        "job_id", jobID,
        "attempt_number", attempt,
        "error_code", code, // mapped safe error code, bukan err.Error() mentah
        "error_kind", "retryable",
        "retry_delay_ms", delay.Milliseconds(),
        "next_run_at", nextRun.UTC().Format(time.RFC3339Nano),
        "from_status", "processing",
        "to_status", "retrying",
    )
}
```

Imports untuk helper: `context`, `log/slog`, `time`. Call site yang ideal: **DB transaction selesai -> emit log -> XACK**. Jika finalisasi commit gagal, emit `db.commit_failed` dan biarkan message pending; **jangan** emit `job.retry_scheduled` atau `job.completed`. Jika `XACK` kemudian gagal, emit `redis.ack_failed` terpisah sambil mempertahankan status DB yang sudah committed.

**Fiber access log:** pasang middleware yang menghasilkan/gunakan `X-Request-ID` yang aman, catat route template (bukan path raw berisi identifier sensitif), HTTP method, response status, duration, dan `request_id`. Middleware mengeluarkan `http.request_completed` sekali setelah downstream selesai, termasuk ketika error mapper/panic recovery menghasilkan response. `job.accepted` menyertakan `request_id` yang sama dan `job_id` hasil use case. Jangan log body request/response atau API key. Inisialisasi request-id middleware **sebelum** access logger, dan pastikan urutan recovery/error handler menjaga log completion. Detail pemanggilan Fiber v3 dipin sesuai versi dependency sebelum coding.

### 16.6 Logging di tiap mekanisme kegagalan dan recovery

| Skenario | Event minimum yang diharapkan | Hal penting |
|---|---|---|
| Handler sukses | `job.claimed` → `job.completed` | Emit `completed` hanya setelah DB commit |
| Handler 503, retry tersedia | `job.attempt_failed` → `job.retry_scheduled` → `scheduler.job_released` | Catat error_code, retry_delay_ms, next_run_at, attempt_number |
| Handler 400/invalid payload | `job.attempt_failed` → `job.dead` | `error_kind=permanent`; tidak ada `job.retry_scheduled` |
| Attempts habis | `job.attempt_failed` → `job.dead` | `error_code=MAX_ATTEMPTS_EXHAUSTED` atau failure reason aman |
| Worker panic | `worker.handler_panic` → retry/dead sesuai policy | Tidak mengubah panic menjadi sukses palsu |
| DB commit gagal | `db.commit_failed` | Jangan ACK Redis, jangan log transisi sukses |
| `XACK` gagal setelah completed | `job.completed` → `redis.ack_failed` | Reaper nanti ACK event terminal; **bukan** rerun handler |
| Redis publish gagal | `outbox.publish_failed` | Outbox tetap menjadi intent untuk dicoba kembali |
| `XADD` sukses, mark DB gagal | `outbox.mark_failed` | Duplicate publish mungkin terjadi, CAS menahan duplikasi eksekusi |
| Worker crash dan lease kedaluwarsa | `reaper.pending_examined` → `reaper.message_reclaimed` → `reaper.job_recovered` | Periksa lease DB, tutup attempt, generation baru, bukan sekadar XAUTOCLAIM |
| PEL idle tetapi lease masih aktif | `reaper.active_lease_skipped` (DEBUG) | Jangan ambil alih worker yang masih berjalan |
| Stale/duplicate generation | `reaper.stale_message_acked` atau worker skip debug | Tidak memanggil handler lagi |
| Graceful shutdown | `component.stopping` → `component.stopped` | Catat in-flight count dan drained vs timeout |

Tiap event transisi yang membutuhkan MySQL commit harus mencerminkan **hasil final transaction**, bukan harapan sebelum commit. State machine adalah sumber kebenaran untuk menilai urutan event yang masuk akal.

### 16.7 Logging versus history/audit, storage, dan security

- **MySQL** tetap menyimpan state terakhir, attempt number, attempt error code, waktu mulai/akhir, dan outbox. Log menunjukkan narasi operasional lintas komponen; jika sink log drop, job tetap harus pulih dari MySQL dan Redis.
- Jika kelak ingin endpoint **timeline job yang queryable** terlepas dari retention log platform, tambahkan tabel `job_events` dalam transaksi bersama state change sebagai **MVP+** (`job_id`, `event_type`, `attempt_number`, `from_status`, `to_status`, `metadata JSON aman`, `occurred_at`). Jangan menganggap stdout log otomatis audit trail durable/transactional.
- Produksi: stdout JSON → container log collector (contoh Fluent Bit / Grafana Alloy / OpenTelemetry Collector) → Loki atau log backend lain. Ini **opsional** pada MVP; tidak wajib membangun ELK hanya untuk portfolio.
- Konfigurasikan retention dan access control pada log backend sesuai kebijakan organisasi; jangan simpan data personal rahasia. Redaction dan izin akses lebih penting daripada menyimpan semua data.
- **GORM/SQL debug logging dan HTTP client debug trace harus nonaktif secara default di production**, karena query parameters/URLs dapat berisi data sensitif meskipun application logger sudah memiliki redaction. Jangan log payload Redis secara penuh.
- Tetapkan limit ukuran log per record (misalnya 4 KiB untuk app messages) dan rate-limit/noise policy; batasi error stack dan teks response eksternal. Per event yang tidak bisa ditulis karena sink error, metrics agregat dan durable state membantu diagnosa.
- Jangan gunakan `job_id`, `request_id`, `event_id` sebagai Prometheus label. Cukup gunakan queue, job_type, reason/error category dengan **cardinality terbatas**.

### 16.8 Unit test dan acceptance untuk logging

**Unit test factory + event emission** memakai `slog.NewJSONHandler(&bytes.Buffer, ...)` yang di-inject, parse JSON per baris, lalu assert `event`/`job_id`/`attempt_number`/`error_code`; jangan hanya memeriksa substring pesan. Gunakan logger sink in-memory untuk test tanpa stdout global. Contoh:

```go
func TestRetryScheduledLogIsStructured(t *testing.T) {
    var buf bytes.Buffer
    log := slog.New(slog.NewJSONHandler(&buf, nil))
    LogRetryScheduled(
        context.Background(), log, "job-abc", 1, "UPSTREAM_UNAVAILABLE",
        5*time.Second, time.Date(2026, 11, 18, 5, 0, 5, 0, time.UTC),
    )

    var record map[string]any
    require.NoError(t, json.Unmarshal(bytes.TrimSpace(buf.Bytes()), &record))
    require.Equal(t, "job.retry_scheduled", record["event"])
    require.Equal(t, "job-abc", record["job_id"])
    require.Equal(t, float64(1), record["attempt_number"])
    require.Equal(t, float64(5000), record["retry_delay_ms"])
    require.NotContains(t, buf.String(), "Authorization")
}
```

Imports: `bytes`, `context`, `encoding/json`, `log/slog`, `testing`, `time`, dan `github.com/stretchr/testify/require`. Fungsi contoh harus ditempatkan dalam package yang sesuai dengan helper-nya; ini contoh pola assertion, bukan implementasi lengkap repository.

Test cases tambahan yang **wajib**:

1. Handler 503 → `job.attempt_failed` dan `job.retry_scheduled` berisi `attempt_number`, `error_code`, `retry_delay_ms`, `next_run_at` yang tepat.
2. Handler invalid payload → `job.dead` saja setelah failed attempt; **tidak** ada `job.retry_scheduled`.
3. Gagal DB commit finalize → muncul `db.commit_failed`; **tidak** ada `job.completed` / `job.retry_scheduled`; Redis message belum ACK.
4. DB commit completed berhasil tetapi `XACK` gagal → `job.completed` tetap benar dan `redis.ack_failed` muncul; reaper tidak mengeksekusi ulang handler.
5. Heartbeat/finalisasi dengan lease kedaluwarsa → `worker.lease_lost`, tanpa event transisi atau ACK. Setelah recovery commit → `reaper.job_recovered`. Reaper melewati lease yang masih aktif.
6. Duplicate publish / stale generation → tidak membuat event `job.claimed`/`job.completed` tambahan untuk eksekusi palsu.
7. Redaction: `Authorization`, API key, DSN, idempotency key, email dan payload sensitif **tidak** muncul di JSON output, termasuk pada error path.
8. Correlation: event `job.accepted` memiliki `request_id + job_id`, lalu semua worker/scheduler/dispatcher/reaper lifecycle events terkait memiliki `job_id` sama.
9. Filtering: INFO production tidak menampilkan DEBUG seperti `redis.ack_succeeded` atau heartbeat setiap 30 detik.
10. Saat shutdown, log menyebut `inflight_jobs`, `drained`, dan `shutdown_duration_ms` agar insiden restart dapat direkonstruksi.

### 16.9 Cara menelusuri satu job lewat Docker Compose

Untuk development, semua service mencetak JSONL ke stdout. Perintah contoh (memerlukan `jq` di host; jalankan dari root repository):

```bash
# Seluruh event untuk satu job, lintas API/dispatcher/worker/scheduler/reaper.
docker compose -f deployments/docker-compose.yml logs --no-log-prefix \
  api dispatcher worker scheduler reaper \
  | jq -c 'select(.job_id == "job-abc") | {time, component, event, attempt_number, error_code, retry_delay_ms, next_run_at}'

# Cari masalah yang perlu diagnosis pada seluruh runtime components.
docker compose -f deployments/docker-compose.yml logs --no-log-prefix \
  api dispatcher worker scheduler reaper \
  | jq -c 'select(.level == "WARN" or .level == "ERROR") | {time, component, event, job_id, error_code}'
```

Contoh perintah hanya berfungsi bila semua output proses terkait benar-benar JSONL dan `jq` terpasang. Jika ada container yang mencetak non-JSON, filter/pisahkan stdout sebelum parsing. Untuk produksi, gunakan query berdasarkan `event`, `component`, dan `job_id` pada log backend alih-alih mengandalkan `docker compose logs` sebagai arsip.

### 16.10 Prometheus metrics dan alert

```text
antriaga_jobs_created_total{type,queue}
antriaga_jobs_completed_total{type,queue}
antriaga_jobs_dead_total{type,queue,reason}
antriaga_jobs_retried_total{type,queue}
antriaga_job_duration_seconds_bucket{type,queue,le}
antriaga_queue_depth{queue}
antriaga_oldest_queued_job_seconds{queue}
antriaga_active_workers{queue}
antriaga_outbox_pending{queue}
antriaga_outbox_oldest_pending_seconds
antriaga_recovery_total{queue,reason}
antriaga_redis_publish_errors_total
antriaga_redis_ack_errors_total{queue}
antriaga_db_commit_errors_total{component}
antriaga_worker_lease_lost_total{queue}
antriaga_worker_inflight{queue}
antriaga_worker_available_slots{queue}
antriaga_api_rejected_total{reason}
antriaga_api_auth_failed_total{reason}
antriaga_queue_wait_seconds_bucket{queue,le}
```

Jangan gunakan `job_id`, `request_id`, atau `idempotency_key` sebagai Prometheus labels karena cardinality tinggi. Gunakan log/tracing untuk korelasi per job.

Alert awal (perlu tuning setelah load test):

- Outbox oldest pending > 60 detik dan meningkat.
- Failure/dead rate > 5% selama 5 menit pada workload signifikan.
- Queue oldest wait > 10 menit.
- Tidak ada worker aktif untuk queue yang berisi job.
- Recovery count terus meningkat atau worker kehilangan lease berulang.
- MySQL/Redis readiness gagal, atau DB commit/Redis ACK error meningkat.
- Lease expiration tinggi untuk job normal.

**Minimum dashboard:** jumlah job per status; success/dead/retry rate; P95 handler duration; queue depth/oldest wait; outbox pending age; recovery count; Redis publish/ACK errors. Klik dari metrik agregat ke logs menggunakan `job_id` dari pencarian log bila diperlukan, **bukan** label metrik per-job.

---
## 17. Security & Trust Boundaries

1. **API auth wajib (MVP):** `X-API-Key` di-generate oleh admin CLI, terikat ke unique client ID, diverifikasi menggunakan stored HMAC digest + secret pepper; key dapat dirotasi/revoke/expire (§27). Tidak ada public API untuk membuat key.
2. **Network isolation:** MySQL dan Redis tidak diekspos ke internet; API melalui reverse proxy HTTPS.
3. **External target allowlist:** Mock webhook/payment/email URL dikonfigurasi server untuk mencegah SSRF.
4. **Credentials:** Dedicated least-privilege DB user; Redis ACL bila tersedia.
5. **Payload controls:** JSON validation, body limit, per-job-type field allowlist, secret redaction.
6. **Logging:** Hindari menyimpan API keys, private payment information, full payload sensitif, token lease, atau raw provider errors. Wajib gunakan allowlisted fields, redaction, dan test kebocoran (lihat §16.2, §16.8).
7. **Swagger:** Nonaktifkan di public production atau lindungi aksesnya.
8. **Metrics:** Bind internal network, jangan membuka `/metrics` secara publik.
9. **Database:** Parameterized queries via GORM/raw parameterized SQL; tidak membangun SQL dari user input.
10. **Idempotency:** Duplicate create dan duplicate external side effect ditangani dengan key stabil.
11. **Object storage:** Worker membuat key privat per client, job, dan attempt dari claim MySQL. Otorisasi pembacaan hasil mengikuti client scope job.
12. **Audit:** Simpan siapa dan kapan melakukan manual requeue/cancel ketika fitur operator tersedia.
13. **Client isolation:** Semua CRUD jobs/attempts/events/requeue dan idempotency dibatasi `client_id` dari verified key, bukan request body.
14. **Bounded resource use:** Slot-based worker consume, per-client throttling, API admission backpressure (§28).
15. **Key lifecycle:** Key hanya ditampilkan sekali, status/expiry checked, no plaintext/digest/pepper in logs; audit rotasi dan pencabutan.

---

## 18. Deployment dan Configuration

### 18.1 Runtime services

```text
api          -> Fiber REST API + Swagger + health
worker       -> Consume Redis Streams, execute jobs
scheduler    -> Release scheduled/retrying jobs
dispatcher   -> Publish pending DB outbox to Redis
reaper       -> XPENDING/guarded XCLAIM + recover expired leases + reconcile stuck jobs
admin (CLI)  -> Create/disable clients, issue/rotate/revoke API keys (no public admin HTTP)
mysql        -> Durable source of truth
redis        -> Message transport / consumer groups
minio        -> Report output storage
mock-mailer  -> Demo email service
mock-webhook -> Demo webhook receiver
mock-payment -> Demo idempotent payment provider
```

### 18.2 `.env.example`

```dotenv
APP_ENV=local
APP_NAME=antriaga
HTTP_ADDR=:8080
ENABLE_SWAGGER=true
LOG_LEVEL=debug
LOG_FORMAT=json
LOG_OUTPUT=stdout
LOG_INCLUDE_SOURCE=false
LOG_ACCESS_HEALTHCHECK=false
LOG_MAX_MESSAGE_BYTES=4096

MYSQL_DSN=antriaga:local-password@tcp(mysql:3306)/antriaga?parseTime=true&loc=UTC
MYSQL_MIGRATION_URL=mysql://antriaga:local-password@tcp(mysql:3306)/antriaga
MYSQL_MAX_OPEN_CONNS=30
MYSQL_MAX_IDLE_CONNS=10
MYSQL_CONN_MAX_LIFETIME=30m

REDIS_ADDR=redis:6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_CONSUMER_GROUP=antriaga-workers
REDIS_STREAM_PREFIX=antriaga:queue:

WORKER_CONCURRENCY=10
REDIS_CONSUME_BATCH_MAX=10
API_RATE_LIMIT_PER_CLIENT_RPS=20
API_RATE_LIMIT_MAX_CLIENTS=10000
API_MAX_OUTSTANDING_PER_CLIENT=1000
API_OUTSTANDING_HIGH_WATERMARK=20000
API_OUTBOX_HIGH_WATERMARK=10000
JOB_DEFAULT_MAX_ATTEMPTS=5
JOB_MAX_PAYLOAD_BYTES=262144
JOB_EXECUTION_TIMEOUT=5m
JOB_LEASE_TTL=90s
JOB_HEARTBEAT_INTERVAL=30s
WORKER_SHUTDOWN_GRACE=30s

SCHEDULER_INTERVAL=1s
DISPATCHER_INTERVAL=500ms
OUTBOX_LEASE_TTL=30s
OUTBOX_PUBLISH_TIMEOUT=5s
REAPER_INTERVAL=15s
REAPER_MIN_IDLE=120s
RETRY_BASE_DELAY=5s
RETRY_MAX_DELAY=15m
RETRY_JITTER_PERCENT=20

MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=local-access
MINIO_SECRET_KEY=local-secret
MINIO_BUCKET=antriaga-output

MOCK_MAILER_URL=http://mock-mailer:8080
MOCK_WEBHOOK_URL=http://mock-webhook:8080
MOCK_PAYMENT_URL=http://mock-payment:8080
# Wajib diisi via secret manager/runtime env (base64 raw random 32+ bytes), JANGAN commit nilainya.
AUTH_KEY_PEPPER_B64=
API_KEY_ENVIRONMENT=test
API_KEY_DEFAULT_EXPIRY_DAYS=90
AUTH_MAX_API_KEY_LENGTH=256
```

**Penting:** nilai di atas bersifat development/example. Pada production set `LOG_LEVEL=info`, pertahankan JSON ke stdout, dan atur retention di log collector. `LOG_MAX_MESSAGE_BYTES` adalah policy yang perlu diimplementasikan pada logger wrapper, **bukan** fitur otomatis `slog`. Jangan menyimpan production secrets di `.env` yang di-commit. `AUTH_KEY_PEPPER_B64` harus ditolak jika kosong/invalid pada startup API atau admin CLI; pengisian secret melalui secret manager. `API_KEY` statis tunggal dari v2 **dihapus**: kredensial client dicari berdasarkan `api_keys` di MySQL. Untuk scheduler/outbox/reaper gunakan UUID instance yang unik pada lease ownership.

### 18.3 Docker Compose blueprint

File `deployments/docker-compose.yml` disarankan memiliki service `api`, `worker`, `scheduler`, `dispatcher`, `reaper`, `mysql`, `migrate`, `redis`, `minio`, tiga mocks, dan opsional `prometheus`, `grafana`. Tiap aplikasi Antriaga menggunakan image yang sama dengan entrypoint/binary berbeda. Contoh bagian utama:

```yaml
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: antriaga
      MYSQL_USER: antriaga
      MYSQL_PASSWORD: local-password
      MYSQL_ROOT_PASSWORD: local-root-password
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD"]
      interval: 5s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

  migrate:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
      target: migrator
    env_file: ["../.env"]
    command: ["up"]
    restart: "no"
    depends_on:
      mysql: {condition: service_healthy}

  api:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
    command: ["/app/api"]
    env_file: ["../.env"]
    ports: ["8080:8080"]
    depends_on:
      migrate: {condition: service_completed_successfully}

  worker:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
    command: ["/app/worker"]
    env_file: ["../.env"]
    depends_on:
      migrate: {condition: service_completed_successfully}
      redis: {condition: service_healthy}

  scheduler:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
    command: ["/app/scheduler"]
    env_file: ["../.env"]
    depends_on:
      migrate: {condition: service_completed_successfully}

  dispatcher:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
    command: ["/app/dispatcher"]
    env_file: ["../.env"]
    depends_on:
      migrate: {condition: service_completed_successfully}
      redis: {condition: service_healthy}

  reaper:
    build:
      context: ..
      dockerfile: deployments/Dockerfile
    command: ["/app/reaper"]
    env_file: ["../.env"]
    depends_on:
      migrate: {condition: service_completed_successfully}
      redis: {condition: service_healthy}

volumes:
  mysql_data:
  redis_data:
```

Service satu kali `migrate` berjalan setelah MySQL sehat. Stage `migrator` harus menyertakan CLI migrasi terpin dan SQL versi checkout. Entrypoint membaca `MYSQL_MIGRATION_URL` dari environment dan meneruskan operasi `up` ke CLI tanpa mencetak kredensial. Migrasi tanpa perubahan keluar `0`; error keluar nonnol dan menahan service aplikasi. API dan scheduler bergantung pada MySQL yang sudah bermigrasi. Worker, dispatcher, dan reaper juga menunggu Redis sehat. Dari root proyek, jalankan `make up` untuk boot normal atau `docker compose -f deployments/docker-compose.yml up --build --scale worker=3` untuk demo skala. Blueprint ini masih perlu service `minio`, mocks, dan metrics sesuai milestone.

### 18.4 Dockerfile multi-stage

Gunakan satu multi-stage Dockerfile yang meng-compile semua binary (`api`, `worker`, `scheduler`, `dispatcher`, `reaper`, `admin`) untuk target OS/arch, lalu copy ke runtime image minimal. Jalankan sebagai non-root, set CA certificates, dan gunakan image/toolchain versi pin. Sediakan stage `migrator` khusus. Pertahankan runtime aplikasi sebagai target akhir default, dan pilih stage `migrator` hanya untuk service `migrate`.

### 18.5 Shutdown

- Stop menerima job baru saat API shutdown (readiness false).
- Worker berhenti consume pesan baru, beri kesempatan running jobs selesai sampai grace period.
- Jika deadline terlampaui, batalkan context handler dan **jangan ACK** eksekusi yang belum sukses commit; lease+reaper melanjutkan recovery.
- Dispatcher menyelesaikan/meninggalkan outbox lease secara aman.
- Scheduler/reaper menghentikan polling, menyelesaikan transaksi singkat, lalu exit.

---

## 19. Local Development Commands

### 19.1 Makefile target yang disarankan

```makefile
.PHONY: help deps swagger test test-race test-integration lint build up down logs migrate-up migrate-down

help:
	@echo "deps swagger test test-race test-integration lint build up down logs migrate-up migrate-down"

deps:
	go mod download

swagger:
	swag init -g cmd/api/main.go -d ./ --parseInternal -o ./docs

test:
	go test ./... -count=1

test-race:
	go test ./... -race -count=1

test-integration:
	go test ./tests/integration/... -tags=integration -count=1

lint:
	golangci-lint run ./...

build:
	go build -o bin/api ./cmd/api
	go build -o bin/worker ./cmd/worker
	go build -o bin/scheduler ./cmd/scheduler
	go build -o bin/dispatcher ./cmd/dispatcher
	go build -o bin/reaper ./cmd/reaper
	go build -o bin/antriaga-admin ./cmd/admin

up:
	docker compose -f deployments/docker-compose.yml up --build -d

down:
	docker compose -f deployments/docker-compose.yml down

logs:
	docker compose -f deployments/docker-compose.yml logs -f

migrate-up:
	docker compose -f deployments/docker-compose.yml run --rm --build migrate up

migrate-down:
	docker compose -f deployments/docker-compose.yml run --rm --build migrate down 1
```

**Catatan:** `MYSQL_MIGRATION_URL` mengikuti sintaks DSN dari `golang-migrate` (berbeda dari DSN GORM). Pin versi `golangci-lint`, `migrate`, `swag` CLI pada developer tooling/CI.

### 19.2 Urutan startup lokal

1. Copy `.env.example` ke `.env`, ubah password development.
2. Jalankan `make up`. Compose menunggu MySQL sehat, menjalankan `migrate`, dan memulai service aplikasi setelah migrasi sukses.
3. API dan scheduler dapat siap tanpa Redis. Consumer menyiapkan group secara idempoten setelah Redis tersedia.
4. Cek status service `migrate`, `/health/ready`, dan `/swagger/index.html` jika diaktifkan. Jika migrasi gagal, periksa `docker compose -f deployments/docker-compose.yml logs migrate` dan perbaiki sebelum mengulang startup.
5. Submit sample job, periksa status, attempts, Redis PEL, dan outbox, lalu jalankan failure-injection test pada mock.

---

## 20. CI/CD Blueprint

### 20.1 Pull request pipeline

```mermaid
flowchart LR
    PR[Pull Request] --> F[go fmt / go vet]
    F --> L[golangci-lint]
    L --> U[Unit tests + race]
    U --> I[Integration tests: MySQL + Redis]
    I --> S[Swagger generation consistency check]
    S --> B[Build 5 binaries + container]
    B --> G[Security + dependency scan]
```

Rekomendasi job CI:

1. `go mod verify` dan dependency cache.
2. `gofmt` + `go vet` + `golangci-lint`.
3. `go test ./... -race -count=1`.
4. MySQL/Redis integration tests (Testcontainers dalam runner dengan Docker).
5. Jalankan `swag init` lalu `git diff --exit-code docs/` untuk memastikan docs tersinkronisasi.
6. Build semua binary; build Docker image.
7. `govulncheck`/dependency scanner dan container image scanner.
8. Pada release: tag immutable, migration review/backup, deployment canary/rolling, smoke test.

### 20.2 Test quality gate

- Tidak ada failing test atau race detector warning.
- Semua migration baru punya rollback policy (rollback bisa manual bila destruktif).
- Tidak ada breaking change API tanpa versioning.
- Swaggo spec up-to-date dengan handler + DTO.
- Perubahan retry/recovery memiliki concurrency/failure-path test.

---

## 21. Milestone Implementasi

### Milestone 1 — API + Persistence + Clean Architecture

- Inisialisasi repo, Go modules, config, `slog` JSON factory + redaction, manual DI.
- Pasang HTTP request-ID/access logging dan event `job.accepted`, `job.rejected`, `db.commit_failed`; test schema/secret redaction.
- Domain Client/APIKey/Job/Attempt, enums, job registry mapping `(job_type, payload_version)`; client identity dibuat server-side.
- MySQL migrations `clients`, `api_keys`, `jobs(client_id,payload_version)`, attempts, outbox; GORM adapters, UoW.
- Admin CLI create/list client dan generate/list/rotate/revoke key (plaintext once); Fiber `X-API-Key` middleware + scope checks + owner scope pada setiap query.
- `POST /jobs`, `GET /jobs/:id`, `GET /jobs` dengan `client_id` ownership dan idempotency scoped per client.
- Idempotency create + transactional outbox record.
- Swaggo annotations, generated docs, Swagger UI.
- Unit dan HTTP handler tests termasuk auth, scope, client isolation, key expiry/revocation, redaction.

**Output:** create job durable, query job, API terdokumentasi, belum mengeksekusi job.

### Milestone 2 — Redis + Worker Processing

- Redis Streams adapter + consumer group bootstrap.
- Outbox dispatcher + dispatch retry, dengan event publish/mark-failure yang terpisah.
- Worker runtime, handler registry `(type, payload_version)`, claim CAS + attempt tracking, **bounded slot-based read/backpressure**.
- Mock webhook dan mock mailer.
- Success/permanent/transient error classification.
- Instrumentasi `job.claimed`, `job.attempt_failed`, `job.completed`, `job.dead`, `redis.ack_failed` beserta unit test event.
- Integration tests DB/Redis dan E2E happy path.

**Output:** immediate job diproses via worker, memiliki histori, success/dead state.

### Milestone 3 — Scheduling, Retry, Crash Recovery

- Scheduler polling `SKIP LOCKED`.
- Exponential backoff + jitter.
- Heartbeat, lease expiry, reaper memakai **`XPENDING` → DB lease check → guarded `XCLAIM` → DB recheck**; `XAUTOCLAIM` hanya optimasi opsional.
- DB queued-job/outbox reconciler.
- Duplicate publish/redelivery tests.
- Worker graceful shutdown.
- Instrumentasi event retry/scheduler/reaper/reconciliation, korelasi job_id, dan pengujian no-false-success pada DB commit failure.
- Mock payment untuk idempotency double-capture test.

**Output:** transient failure dan worker crash dipulihkan tanpa kehilangan durability semantics.

### Milestone 4 — Report + Hardening + Portfolio Demo

- Report handler + MinIO + result metadata.
- Prometheus metrics, dashboard/query contoh untuk structured logs, tuning noise/retention (logger dan request ID sudah dimulai pada Milestone 1).
- Auth sudah selesai Milestone 1; lakukan security hardening, load testing key lookup dan cache/rotation strategy. Payload limit/secret handling divalidasi ulang.
- Load generator (700 emails, 200 webhooks, 100 reports).
- E2E failure injection + recovery assertions.
- README, API examples, diagrams, benchmarking notes.

**Output:** demo lengkap yang memperlihatkan retry, recovery, scaling, observability, client authentication/isolation, dan idempotency.

Uji report dengan dua client yang meminta nama laporan sama dan attempt pertama yang selesai setelah kehilangan lease. Hanya hasil attempt yang commit boleh dibaca sebagai hasil job.

**Backlog pasca-MVP (P1/P2):** manual dead-job requeue dengan audit, safe data retention cleanup + runbooks, kemudian durable `job_events` timeline. Keempatnya bukan blocker untuk membuka MVP internal bila safeguard P0 sudah lolos.

Perkiraan waktu bergantung pengalaman dan kedalaman test; milestone adalah urutan kerja, bukan komitmen durasi.

---

## 22. Definition of Done

### Functional

- [ ] Admin dapat membuat client unik, menerbitkan API key scoped, merotasi dan mencabutnya (secret ditampilkan sekali).
- [ ] Fiber menolak key invalid/expired/revoked, disabled client, dan insufficient scopes dengan response yang benar.
- [ ] Semua job dimiliki `client_id` sah dan seluruh query/idempotency diisolasi per client.
- [ ] Setiap job mempunyai immutable `payload_version`; handler compatibility diuji.
- [ ] Worker tidak mengambil Redis message melebihi slot eksekusi tersedia.
- [ ] Setiap job type statis tervalidasi dan handler terdaftar.
- [ ] `POST /jobs` persist job dan outbox atomik.
- [ ] API dapat menampilkan job status dan attempt history.
- [ ] Multiple worker berjalan pada satu consumer group per queue.
- [ ] Transient errors retry dengan bounded exponential backoff + jitter.
- [ ] Permanent errors dan exhausted attempts berakhir sebagai `dead`.
- [ ] Scheduled jobs dijalankan setelah due.
- [ ] Crash recovery menggunakan Redis PEL (`XPENDING`/`XCLAIM`) + DB lease/fencing dengan DB recheck setelah ownership change.
- [ ] Duplicate publish atau redelivery tidak menjalankan dua active DB claims pada generation yang sama.
- [ ] Mock payment capture tidak membuat double side effect.
- [ ] Report output disimpan di MinIO, bukan pada broker payload.

### Engineering quality

- [ ] Key dihasilkan dengan `crypto/rand` 32-byte entropy, hanya digest HMAC disimpan di MySQL, pepper dari secret manager.
- [ ] Admin CLI merupakan satu-satunya entrypoint pengelolaan client/key pada MVP; tidak ada route public generate key.
- [ ] Auth, client isolation, key lifecycle, worker backpressure, recovery races, dan payload versions diuji unit/integration/E2E.
- [ ] Clean Architecture dependency direction ditegakkan.
- [ ] Domain/application tidak import Fiber/GORM/go-redis.
- [ ] SQL migrations versioned, dengan indeks untuk polling/recovery.
- [ ] Swaggo spec/generated docs sesuai endpoint.
- [ ] Unit tests domain/application >= 80% line coverage (target awal).
- [ ] Integration tests memakai MySQL/Redis nyata.
- [ ] Concurrency, reaper, dan crash windows diuji.
- [ ] `go test -race ./...` lolos.
- [ ] Docker Compose demo bisa dijalankan dari fresh checkout setelah env/migration setup.
- [ ] Logs JSON terstruktur dengan event catalog §16.3 tersedia di seluruh runtime service; metrics, health/readiness endpoint tersedia.
- [ ] Retry, permanent failure, attempts exhausted, commit failure, Redis ACK/publish failure, worker recovery, dan stale message memiliki event yang berbeda.
- [ ] Setiap job bisa ditelusuri via `job_id`; API log menghubungkan `request_id` dengan `job_id`.
- [ ] Tidak ada false `job.completed`/`job.retry_scheduled` jika transaksi DB gagal.
- [ ] Secret/PII redaction dan structured log tests lulus; `INFO` production tidak spam heartbeat/empty polling.
- [ ] README berisi instruksi start, desain, tradeoff, batasan, dan demo.

### Reliability acceptance (bukan klaim exactly-once)

- [ ] Setelah Redis unavailable sementara, queued job tetap bisa terpublish ulang dari DB intent.
- [ ] Setelah worker mati, job bisa diambil ulang jika eligible dan lease expired.
- [ ] Heartbeat dan semua hasil `FinishJob` ditolak tanpa ACK pada dan setelah expiry, termasuk saat token lama masih tersimpan dan reaper belum berjalan (§29.3).
- [ ] Setelah DB `completed` commit dan XACK gagal, tidak mengulangi handler karena terminal state.
- [ ] Setelah side effect tetapi sebelum DB commit, external mock dedup bekerja dengan idempotency key stabil.
- [ ] Tidak ada unbounded retry untuk invalid payload/unknown job type.
- [ ] Selama key rotation, job yang telah diterima tetap mencapai terminal state dan request baru memakai key yang berlaku.
- [ ] Bila Redis PEL ownership berpindah bersamaan heartbeat, tidak ada false recovery untuk lease DB yang aktif.

---

## 23. Architectural Decision Records (ADR Ringkas)

| ID | Decision | Rationale | Tradeoff |
|---|---|---|---|
| ADR-001 | MySQL = source of truth | Durable state dan queryable history | Higher DB write load |
| ADR-002 | Redis Streams = broker MVP | Consumer groups dan PEL mendukung worker distribution | Perlu recovery/reconciliation sendiri |
| ADR-003 | Transactional outbox | Mengatasi gap DB commit vs XADD | Tambahan tabel dan dispatcher |
| ADR-004 | At-least-once delivery | Realistis untuk distributed system | Downstream idempotency wajib |
| ADR-005 | Clean Architecture | Use cases bisa diuji tanpa infra | Lebih banyak interfaces/boilerplate |
| ADR-006 | Fiber v3 | HTTP adapter Go yang ringan, terpisah dari core | Dependency framework tetap dipin |
| ADR-007 | Swaggo | Dokumentasi API dari annotations | Output Swagger 2.0, perlu proses regenerate |
| ADR-008 | Static job types | Aman dan scope MVP terkendali | Tidak support arbitrary custom jobs |
| ADR-009 | DB lease dengan expiry ketat, token, dan enqueue generation | Waktu MySQL setelah lock membatasi heartbeat/finalisasi. Token dan generation menolak worker/pesan lama | Perlu transaksi singkat dan tes pada batas waktu |
| ADR-010 | SQL migrations terpisah GORM | Schema changes auditable | Perlu disiplin migrasi |
| ADR-011 | `log/slog` JSON lifecycle events + `job_id` correlation | Investigasi retry/recovery end-to-end tanpa framework logging tambahan | Perlu event naming, redaction, dan pengendalian volume log |
| ADR-012 | Admin-generated API keys per client | Isolasi identitas, scoped access, rotasi/revoke tanpa mengganti client | Perlu key lifecycle, HMAC pepper, DB read saat auth |
| ADR-013 | Bounded worker slots + API admission controls | Tidak over-prefetch Redis atau backlog tanpa batas | Rate limits/capacity perlu tuning |
| ADR-014 | XPENDING + guarded XCLAIM + DB recheck | Menghindari active-lease recovery race | Lebih banyak round-trip dibanding XAUTOCLAIM |
| ADR-015 | Business payload_version | Kompatibilitas backlog selama rolling deployment | Handler versi lama perlu masa dukungan |
| ADR-016 | Job retention & operator requeue terpisah MVP | Kontrol biaya/insiden tanpa menambah risiko MVP | Perlu maintenance P1 |
| ADR-017 | Durable job_events pada P2 | Queryable transactional timeline | Write amplification dan kebutuhan retention |

---

## 24. Risiko, Trade-off, dan Batasan

### 24.1 Exactly-once tidak dijamin

At-least-once Redis message delivery ditambah DB fencing **tidak bisa mengunci external side effect** yang sudah terjadi sebelum crash. Hasil yang lebih kuat hanya mungkin melalui downstream idempotency dan kontrak bisnis yang tepat. **Jangan mengiklankan exactly-once execution**.

### 24.2 MySQL bisa menjadi bottleneck

Job state, attempts, outbox, scheduler, dan heartbeat semuanya menghasilkan DB writes. Gunakan index, batching, pooling, dan data retention/archival. Jangan membuat heartbeat untuk setiap goroutine jauh lebih sering dari kebutuhan.

### 24.3 Redis Streams tidak otomatis menyediakan priority queue

MVP memakai FIFO-ish stream order per queue, dengan concurrency/polling dan redelivery yang tidak menjamin urutan ketat eksekusi/penyelesaian. Field priority tidak berarti pesan pasti diproses lebih dulu.

### 24.4 Operasi network tidak atomik dengan DB

Transactional outbox memindahkan masalah dari *lost work* menjadi *possible duplicate delivery*. Ini tradeoff yang disengaja. Perlindungan duplicate terletak pada generation fencing dan downstream idempotency.

### 24.5 Lease expiry tidak otomatis menghentikan side effect

Timeout handler dan DB fencing tidak dapat membatalkan seluruh request yang telah diterima provider eksternal. Terapkan idempotency provider dan batasan concurrency/latency, terutama untuk payment-like workflow.

### 24.6 Scheduler bukan hard real-time

Under load, job bisa dieksekusi setelah `scheduled_at`, bukan dijamin tepat pada detik tersebut. Ukur scheduling lag dan berikan SLO setelah benchmark.

### 24.7 Redis AOF tidak menggantikan MySQL

Persistensi AOF mengurangi kehilangan stream saat restart tetapi tidak menghilangkan semua failure windows. MySQL/outbox/reconciler tetap menjadi jalur durable recovery.

### 24.8 API key leakage dan rotation failure

API key adalah bearer secret. Tidak ada penggantian identitas manusia yang kuat di MVP, sehingga akses operator CLI harus dibatasi dan diaudit. Jika key bocor, revoke segera, rotate via secret manager, review accepted jobs, dan pantau auth anomaly.

### 24.9 Backpressure bukan jaminan no-overload

Limiter lokal berlaku per replica dan bisa berubah saat API restart. Kuota job aktif per client diserialisasi di MySQL. Ambang global merupakan batas operasional, sehingga insert serentak lintas client bisa melampauinya sementara. Uji Redis outage dengan job langsung dan terjadwal, dan ukur biaya query admisi sebelum menambah counter.

### 24.10 Payload compatibility dan retention

Hapus handler lama hanya setelah seluruh backlog/scheduled jobs yang memakai `payload_version` lama telah selesai/ditangani. Cleanup history tidak boleh merusak source of truth/reconciliation.

---

## 25. Demo Narrative untuk Portfolio

1. Buat dua client (`payment-service`, `notification-service`) melalui admin CLI, generate API key berbeda, dan tunjukkan client B tidak dapat membaca job client A.
2. Submit 1.000 jobs melalui load generator dengan scoped client API key: 700 email, 200 webhook, 100 report.
3. Monitor status job, queue depth, attempts, dan latency.
4. Mulai satu worker; catat processing rate.
5. Scale worker menjadi lima; tunjukkan throughput dan backlog berubah.
6. Simulasikan `503` dan `429`; amati log `job.attempt_failed` → `job.retry_scheduled` → `scheduler.job_released` bersama `retry_delay_ms` dan `next_run_at`.
7. Simulasikan `400` invalid payload; tunjukkan `dead` tanpa retry berulang.
8. Matikan worker ketika job sedang berjalan; tunjukkan PEL + DB lease recovery dengan log `reaper.job_recovered` dan `job_id` yang sama.
9. Simulasikan payment berhasil tetapi response putus; tunjukkan idempotent mock capture.
10. Simulasikan Redis outage setelah API create; tunjukkan outbox catch-up.
11. Jalankan report lebih lama dari satu heartbeat interval; cek hasil di MinIO.

**Kriteria penilaian:** jelaskan bukan hanya *apa yang berjalan*, tetapi *apa yang terjadi pada setiap crash window* dan bagaimana transaksinya membatasi duplikasi/kehilangan pekerjaan.

---

## 26. Referensi Teknis Resmi

- [Fiber v3 documentation](https://docs.gofiber.io/)
- [Fiber v3 Swaggo middleware](https://github.com/gofiber/contrib/tree/main/v3/swaggo)
- [Swaggo/swag README](https://github.com/swaggo/swag)
- [Redis Streams overview](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis XREADGROUP](https://redis.io/docs/latest/commands/xreadgroup/)
- [Redis XACK](https://redis.io/docs/latest/commands/xack/)
- [Redis XPENDING](https://redis.io/docs/latest/commands/xpending/)
- [Redis XCLAIM](https://redis.io/docs/latest/commands/xclaim/)
- [Redis XAUTOCLAIM (optimasi dengan peringatan ownership)](https://redis.io/docs/latest/commands/xautoclaim/)
- [Redis XTRIM (opsi ACKED mulai Redis 8.2)](https://redis.io/docs/latest/commands/xtrim/)
- [OWASP Key Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html)
- [go-redis documentation](https://redis.io/docs/latest/develop/clients/go/)
- [GORM documentation](https://gorm.io/docs/)
- [Testcontainers for Go](https://golang.testcontainers.org/)
- [golang-migrate](https://github.com/golang-migrate/migrate)
- [Go structured logging (`log/slog`)](https://pkg.go.dev/log/slog)

---

## 27. Client-Based API Key Authentication (MVP / P0)

### 27.1 Model identitas: client ≠ API key

Antriaga adalah **service-to-service API**, bukan aplikasi login user. `client` adalah **aplikasi/service pemanggil** (misalnya `payment-service`) dengan identitas permanen; `api_key` adalah credential yang bisa dibuat, dirotasi, dan dicabut tanpa mengganti identitas client. Satu client boleh memiliki beberapa key aktif agar rotasi tanpa downtime berjalan aman. Setiap key tepat milik **satu** client. Ini bukan mekanisme login manusia, OAuth2, atau user-session management.

| Konsep | Contoh | Aturan |
|---|---|---|
| `client_id` | UUIDv7 | Primary identifier server-generated, tidak dipilih user |
| `client_code` | `payment-service` | Slug unik yang mudah diingat, immutable setelah dibuat |
| `client.status` | `active` / `disabled` | Client disabled ditolak untuk request API baru |
| `key_id` | UUIDv7 | Identifier publik credential, berbeda dari `client_id` |
| API key plaintext | `ant_live_<key_id>_<random_secret>` | Secret minimal **32 random bytes** dari `crypto/rand`; hanya ditampilkan **sekali** |
| Stored credential | `HMAC-SHA256(pepper, full_api_key)` | Disimpan sebagai `BINARY(32)`; plaintext dan pepper tidak disimpan di DB |
| `api_keys.scopes` | `jobs:create`, `jobs:read` | Izin minimum per key, tidak otomatis operator/admin |
| `api_keys.expires_at` | UTC timestamp | Kedaluwarsa sesuai policy; `revoked_at` untuk pencabutan segera |
| `jobs.client_id` | Foreign key `clients.id` | Job terikat pada client pemilik, bukan key yang dapat dirotasi |

**Bentuk key (format kontrak, bukan key sungguhan):**

```text
ant_live_<key-id-uuid>_<base64url-32-random-bytes>
ant_test_<key-id-uuid>_<base64url-32-random-bytes>
```

`ant_live` dan `ant_test` adalah label lingkungan, **bukan** pengganti isolasi environment. Gunakan DB dan `AUTH_KEY_PEPPER_B64` terpisah per lingkungan. Saat mem-parse, pecah header dengan batas empat bagian (`SplitN(key, "_", 4)` menghasilkan `ant`, env, `key_id`, secret). `key_id` UUID tidak memuat `_`, tetapi secret base64url boleh memuat `_` dan `-`, jadi jangan memecah tanpa batas. `key_id` berguna untuk lookup dengan indeks, bukan untuk autentikasi. Jangan menerima `client_id` dari request body/header sebagai sumber identitas: **server menurunkannya dari API key terverifikasi**.

### 27.2 Database: `clients` dan `api_keys`

Migrations greenfield dijalankan **berurutan**: `clients` → `api_keys` → `jobs` → `job_attempts` → `outbox_events`. Jika suatu saat migrasi dari DB v2 yang sudah memiliki rows, siapkan *backfill client* sebelum `jobs.client_id` dibuat `NOT NULL`; tidak boleh langsung menerapkan FK/NOT NULL pada existing rows tanpa rencana migrasi.

```sql
-- 000001_create_clients.up.sql
CREATE TABLE clients (
    id CHAR(36) NOT NULL PRIMARY KEY,
    code VARCHAR(80) NOT NULL,
    name VARCHAR(160) NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'active',
    allowed_job_types JSON NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    disabled_at DATETIME(3) NULL,
    UNIQUE KEY uq_clients_code (code),
    KEY idx_clients_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 000002_create_api_keys.up.sql
CREATE TABLE api_keys (
    id CHAR(36) NOT NULL PRIMARY KEY,
    client_id CHAR(36) NOT NULL,
    label VARCHAR(100) NOT NULL,
    environment VARCHAR(12) NOT NULL,
    secret_digest BINARY(32) NOT NULL,
    pepper_version SMALLINT UNSIGNED NOT NULL DEFAULT 1,
    scopes JSON NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'active',
    expires_at DATETIME(3) NULL,
    last_used_at DATETIME(3) NULL,
    revoked_at DATETIME(3) NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uq_api_keys_digest (secret_digest),
    KEY idx_api_keys_client_status (client_id, status),
    KEY idx_api_keys_expiration (status, expires_at),
    CONSTRAINT fk_api_keys_client FOREIGN KEY (client_id) REFERENCES clients(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`jobs` dalam migration awal wajib mempunyai:

```sql
client_id CHAR(36) NOT NULL,
payload_version SMALLINT UNSIGNED NOT NULL DEFAULT 1,
UNIQUE KEY uq_jobs_client_idempotency (client_id, idempotency_key),
KEY idx_jobs_client_created (client_id, created_at),
KEY idx_jobs_client_status (client_id, status, created_at),
CONSTRAINT fk_jobs_client FOREIGN KEY (client_id) REFERENCES clients(id)
```

**Penting:** indeks unik `idempotency_key` global dari v2 **diganti** dengan indeks komposit `(client_id, idempotency_key)`. Dua client boleh menggunakan idempotency key sama tanpa membaca atau memengaruhi job milik pihak lain. Unique index MySQL memungkinkan banyak nilai `NULL` sehingga request tanpa idempotency key masih bisa membuat job baru. `request_hash` dihitung sesuai kontrak canonical request di §27.7.

Kredensial operasional untuk `admin` harus terisolasi: privilege menulis `clients`/`api_keys` **tidak** diberikan ke user DB read-only; proses worker tidak perlu memiliki akses membaca `api_keys`. Dalam deployment dengan satu user DB aplikasi untuk MVP, kompensasi menggunakan network/private secrets dan rencana pemisahan privilege.

### 27.3 Generate API key oleh admin, bukan melalui public route

Admin CLI (binary `cmd/admin`) dijalankan hanya oleh operator yang mempunyai akses OS/deployment terotorisasi:

```bash
# Contoh perintah desain CLI; akan tersedia setelah CLI diimplementasikan.
antriaga-admin clients create --code payment-service --name "Payment Service"
antriaga-admin clients list
antriaga-admin keys create --client payment-service --label primary \
  --scopes jobs:create,jobs:read --expires-in 90d
antriaga-admin keys list --client payment-service  # metadata saja; tanpa secret
antriaga-admin keys rotate --key-id <key-id> --overlap 24h
antriaga-admin keys revoke --key-id <key-id> --reason compromised
antriaga-admin clients disable --code payment-service --reason retired
```

Perintah `keys create` menghasilkan key **hanya sekali** pada output yang aman; pengguna bertanggung jawab memasukkannya ke secret manager. Jangan kirim plaintext ke application log, audit log, database, issue tracker, atau screenshot. Jika key hilang, **buat key baru dan revoke key lama**: plaintext tidak dapat dipulihkan. Untuk mode noninteraktif, gunakan opsi `--out-file` yang aman (permission `0600`) dan berikan peringatan agar file segera dipindah ke secret manager; output terminal juga bisa terekam oleh shell/session audit.

Contoh generator Go (fungsi ilustratif yang bisa diuji tanpa Fiber/GORM):

```go
package credentials

import (
    "crypto/hmac"
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "errors"
    "fmt"
)

// keyID dibuat di application/admin use case; env: "live" atau "test".
// pepper wajib berasal dari secret manager, bukan dari database atau kode.
func Generate(keyID, env string, pepper []byte) (string, [32]byte, error) {
    var digest [32]byte
    if keyID == "" || (env != "live" && env != "test") || len(pepper) < 32 {
        return "", digest, errors.New("invalid key generation configuration")
    }
    random := make([]byte, 32)
    if _, err := rand.Read(random); err != nil {
        return "", digest, err
    }
    secret := base64.RawURLEncoding.EncodeToString(random)
    plaintext := fmt.Sprintf("ant_%s_%s_%s", env, keyID, secret)
    mac := hmac.New(sha256.New, pepper)
    _, _ = mac.Write([]byte(plaintext))
    copy(digest[:], mac.Sum(nil))
    return plaintext, digest, nil
}

func Verify(plaintext string, storedDigest []byte, pepper []byte) bool {
    mac := hmac.New(sha256.New, pepper)
    _, _ = mac.Write([]byte(plaintext))
    return hmac.Equal(storedDigest, mac.Sum(nil))
}
```

Kunci HMAC `AUTH_KEY_PEPPER_B64` wajib bernilai acak minimal 32 bytes (representasi base64, di-decode saat bootstrap). Kolom `api_keys.pepper_version` (default 1) mencatat pepper yang dipakai untuk digest. MVP hanya mendukung versi 1. Jika pepper diputar kelak, gunakan dual-verify per versi selama periode transisi dan rencana penggantian semua key; kehilangan pepper akan menginvalidasi kemampuan memverifikasi key lama. Karena secret sudah 256-bit random, HMAC-SHA256 cukup untuk lookup aman; ini **berbeda** dari password manusia yang perlu password-hashing khusus. Pertahankan penggunaan `hmac.Equal` dan batasi panjang header sebelum mem-parsing.

### 27.4 Alur autentikasi dan authorization di Fiber

```mermaid
sequenceDiagram
    participant C as Client Service
    participant A as Fiber Auth Middleware
    participant DB as MySQL
    participant U as Application Use Case
    C->>A: X-API-Key: ant_live_<key_id>_<secret>
    A->>A: Validate shape, length, environment
    A->>DB: Find api_keys by key_id + clients
    DB-->>A: digest, key status/expiry, scopes, client status
    A->>A: HMAC + constant-time compare
    alt Invalid, revoked, expired, client disabled
        A-->>C: 401 invalid_api_key (generic)
    else Valid but scope insufficient
        A-->>C: 403 forbidden
    else Valid and scope allowed
        A->>U: AuthPrincipal(client_id, key_id, scopes)
        U->>DB: Query/insert with client_id scope
        U-->>C: API response
    end
```

1. Ambil **satu** `X-API-Key` header; reject header ganda/berukuran besar dan **jangan** terima API key di query string atau JSON body.
2. Validasi prefix (`ant_live`/`ant_test`), `key_id` (UUID), panjang, dan karakter tanpa meng-copy secret ke error.
3. Lookup `api_keys.id` dengan join client; hitung digest penuh dan bandingkan konstan waktu. Key tidak ditemukan harus menghasilkan error generik dan jalur dummy compare seperlunya untuk mengurangi informasi timing.
4. Tolak jika `client.status != active`, `key.status != active`, revoked, expired, atau environment tidak cocok.
5. Inject principal ke request-scoped context adapter: `ClientID`, `APIKeyID`, scopes; application menerima principal sebagai command/query input value **bukan** `fiber.Ctx`.
6. Enforce scopes **dan** batas kepemilikan data: `WHERE jobs.id=? AND jobs.client_id=?` untuk Get/Attempts/Events/Requeue; `List` selalu di-filter `client_id` server-side. ID job client lain dibalas `404`, bukan membocorkan eksistensi job.
7. Submit job menyimpan `jobs.client_id` dari principal (server-side), bukan dari payload. `Idempotency-Key` dedup hanya di dalam client yang sama.
8. `last_used_at` di-update secara throttled/asynchronous (misalnya paling sering 5 menit per key) agar setiap request tidak menambah DB write; jangan jadikan log aktivitas sebagai sumber audit otorisasi utama.

| Endpoint/action | Minimum scope | Pembatasan |
|---|---|---|
| `POST /api/v1/jobs` | `jobs:create` | Allowed job type milik client, quotas |
| `GET /api/v1/jobs` | `jobs:read` | Filter `client_id` wajib |
| `GET /api/v1/jobs/:id` | `jobs:read` | Owner saja; cross-client return `404` |
| `GET /api/v1/jobs/:id/attempts` | `jobs:read` | Owner saja; error disanitasi |
| `GET /api/v1/jobs/:id/events` (P2) | `jobs:read` | Owner saja |
| `POST /api/v1/jobs/:id/requeue` (P1) | `jobs:requeue` | Owner saja + audit + explicit reason |
| Admin create/revoke key | **Admin CLI only** | Tidak menerima user API key biasa |
| `/health/live`, `/health/ready` | Tidak memakai client key | Respons ringkas, tidak membocorkan detail credential |
| `/metrics`, `/swagger/*` | Internal access policy | Tidak otomatis publik walau ada UI Swagger |

**Contoh request (hanya placeholder):**

```http
POST /api/v1/jobs HTTP/1.1
Host: antriaga.internal
X-API-Key: <API_KEY_DARI_SECRET_MANAGER>
Idempotency-Key: welcome-user-10021
Content-Type: application/json

{"type":"email.send","payload_version":1,"payload":{"to":"sample@example.com","subject":"Welcome","template":"welcome"}}
```

**Status error auth:** `401` untuk key absent/malformed/invalid/expired/revoked/client disabled dengan pesan generik `INVALID_API_KEY`; `403` untuk key valid tetapi scope kurang dengan `INSUFFICIENT_SCOPE`; `404` untuk job client lain; `429` bila client mencapai limit submit/rate. Gunakan TLS di reverse proxy, security headers, dan rate limit per client/key/IP untuk mengurangi penyalahgunaan. `api_keys` tidak pernah dikirim ke worker, Redis, atau job payload.

### 27.5 Lifecycle, rotasi, audit, dan logging

- **Key creation:** satu client aktif dapat memiliki beberapa key berlabel (`primary`, `rollover`, `ci-staging`) dan expiry. Default expiry disepakati (contoh 90 hari), configurable.
- **Rotation:** buat key baru → deploy ke client → verifikasi request memakai key baru → revoke key lama setelah overlap terbatas. Jika *compromised*, revoke **segera** tanpa overlap.
- **Revocation:** middleware memverifikasi status terhadap DB; jika menggunakan cache auth, TTL pendek dan mekanisme invalidasi wajib. MVP lebih aman melakukan lookup DB dengan pool/timeout yang dibatasi.
- **Client disable:** blok request baru untuk seluruh key; job yang **sudah diterima** tetap diselesaikan berdasarkan reliability contract (kecuali tindakan pembatalan operator yang terpisah). Dokumentasikan pilihan ini agar tidak mengejutkan operator.
- **Audit:** buat tabel `admin_audit_events` (P1) atau audit sink internal dengan event `client.created`, `client.disabled`, `api_key.created`, `api_key.rotated`, `api_key.revoked`, `api_key.auth_failed`. Untuk aksi keamanan, audit durable lebih baik daripada mengandalkan stdout log.
- **Logging:** field aman `client_id`, `api_key_id`, `request_id`, `auth_result`, `reason_code`; **jangan** log secret, header raw, full key prefix/suffix, digest, atau pepper. Metrics auth agregat tanpa `client_id` label untuk menghindari high-cardinality.

### 27.6 Test plan autentikasi (wajib MVP)

| Test | Ekspektasi |
|---|---|
| Generate 1.000 key untuk client yang sama | Seluruh plaintext dan digest unik; memakai `crypto/rand` |
| Key terverifikasi dan scope benar | `202`/`200`, principal `client_id` tepat |
| Invalid, typo, missing, revoked, expired, client disabled | `401` generik tanpa informasi apakah key pernah ada |
| Key valid tetapi kurang scope | `403`, tanpa perubahan state |
| Client A mengakses job client B | `404`; list/attempts/events tidak membocorkan data B |
| Client A dan B menggunakan `Idempotency-Key` sama | Dua job terpisah, tidak konflik |
| API key diputar ketika worker memproses job | Job yang sudah di-commit berjalan sampai terminal; request baru memakai key baru |
| Key generation/rotation gagal DB commit | Jangan mencetak key yang belum tersimpan sebagai berhasil; atomic transaction |
| Logging/auth panic/error | Header API key/secret/pepper tidak bocor |
| MySQL auth lookup unavailable | Fail closed (`503` atau `401` sesuai error policy; rekomendasi `503` agar tidak terlihat key salah) |

---

### 27.7 Canonical request untuk deduplikasi

`Idempotency-Key` opsional. Jika ada, terima 1–150 byte ASCII tercetak tanpa spasi (0x21–0x7E), bandingkan byte secara case-sensitive, dan scope ke client terautentikasi. Header kosong atau invalid menghasilkan `400`. Gunakan `VARBINARY(150)` agar indeks unik menerapkan semantik yang sama. Request tanpa key menyimpan `NULL`.

1. Validasi request secara struktural, termasuk JSON UTF-8 dan nama object yang unik setelah decoding escape pada semua kedalaman. Tolak nama duplikat dengan `400`. Validasi tipe field dan kecocokan `queue` dengan job type. Terapkan default efektif saat submit dan pakai nilai itu juga untuk insert.
2. Bentuk objek canonical dengan field `type`, `queue` hasil server mapping, `payload_version`, `payload`, `max_attempts`, dan `scheduled_at`. Untuk job immediate, gunakan `null` saat field dihilangkan atau bernilai `null`. Untuk job terjadwal, parse RFC3339, konversi ke UTC, potong ke presisi milidetik, dan serialize dengan tiga digit pecahan serta akhiran `Z`. Pakai timestamp hasil normalisasi yang sama saat menyimpan job.
3. Serialize objek canonical menjadi JSON UTF-8 tanpa whitespace tambahan, urutkan nama setiap objek berdasarkan byte UTF-8 dari nama hasil decoding, dan pertahankan urutan array. Serialisasi string memakai UTF-8 literal, dengan `\"` untuk tanda kutip, `\\` untuk backslash, dan `\u00xx` huruf kecil untuk karakter kontrol U+0000–U+001F. Pertahankan token angka payload tanpa konversi floating point. Hash byte canonical dengan SHA-256 dan simpan hex huruf kecil. Pertahankan aturan versi ini selama hash lama masih dipakai.
4. Untuk key yang sudah ada, bandingkan hash dengan row yang sudah commit. Hash sama mengembalikan job existing. Hash berbeda menghasilkan `409`. Untuk insert baru, jadwal yang disediakan harus lebih besar dari waktu MySQL. Terapkan pemeriksaan waktu bergerak setelah lookup sehingga retry identik tetap bisa menemukan job asal sesudah jadwal lewat. Jika unique key bertabrakan saat insert serentak, rollback insert dan baca row pemenang yang sudah commit dari transaksi baru.

| Pasangan request dalam satu client | Hasil |
|---|---|
| Key sama, whitespace atau urutan field objek berbeda | Job sama |
| Key sama, default yang dihilangkan dibandingkan nilai efektif yang sama, atau queue dihilangkan dibandingkan queue yang cocok | Job sama |
| Key sama, timestamp `2030-01-01T07:00:00+07:00` dan `2030-01-01T00:00:00.000Z` | Job sama, termasuk retry setelah jadwal lewat |
| Key sama, jadwal dihilangkan dan jadwal `null` | Job immediate yang sama |
| Key sama, urutan array berbeda atau token angka payload `1` dan `1.0` | `409` |
| Key `Event-A` dan `event-a` | Dua key berbeda |
| Nama JSON berulang, termasuk ketika salah satunya ditulis dengan escape | `400`, tanpa insert |

Gunakan [fixture canonical request](docs/examples/idempotency-cases.json) untuk tes use case dan penyimpanan. Bekukan waktu pada fixture tes agar kasus sebelum dan sesudah due tetap deterministik.

## 28. Backpressure, Worker Concurrency, dan Client Quotas (P0)

### 28.1 Batas concurrency yang nyata

`WORKER_CONCURRENCY=10` berarti maksimal 10 handler aktif **per worker instance**. Consumer harus mempunyai **slot kosong sebelum** memanggil `XREADGROUP`; `COUNT <= available_slots`, bukan mengambil 50 pesan lalu menaruh semuanya di buffer in-memory tanpa lease. Redis PEL menandai pesan sudah delivered meski handler belum mulai, sehingga over-prefetch memperumit recovery.

```text
while running:
    available = concurrency_limit - in_flight
    if available <= 0: wait_for_slot_or_shutdown()
    batch_size = min(available, configured_consume_batch)
    deliveries = XREADGROUP(COUNT=batch_size, BLOCK=...)
    for delivery in deliveries:
        acquire_slot()  # slot telah dihitung; implementasi harus race-safe
        dispatch_with_bounded_goroutine(delivery)
```

Batas awal: `WORKER_CONCURRENCY=10`, `REDIS_CONSUME_BATCH_MAX=10`, `PER_QUEUE_CONCURRENCY_*` dapat mengatur pengalokasian pool per queue. Gunakan bounded semaphore/channel dan `context.Context` untuk graceful shutdown. Timeout dan heartbeat mulai dihitung **setelah** DB claim berhasil, bukan ketika message masih menunggu semaphore.

### 28.2 Pengendalian overload di API dan dispatcher

- **Rate limit per client:** API memakai token bucket lokal per instance setelah auth, berdasarkan `client_id`. Burst awal sama dengan RPS. Batasi jumlah bucket dengan `API_RATE_LIMIT_MAX_CLIENTS` dan hapus bucket idle yang token-nya sudah penuh. Jika bucket baru tidak dapat dialokasikan, kembalikan `429 RATE_LIMITED`. Kebijakan tetap sama ketika Redis putus, dengan kuota laju berlaku per replica.
- **Kuota durable per client:** jalankan transaksi `READ COMMITTED`; lock row client dan periksa kembali status aktif. Periksa receipt dedup dari row job lebih dulu. Key yang sudah tercatat mengikuti kontrak dedup tanpa memakai slot baru. Untuk insert baru, hitung `queued`, `scheduled`, `retrying`, dan `processing` milik client melalui indeks. Pada batas `API_MAX_OUTSTANDING_PER_CLIENT`, kembalikan `429 CLIENT_QUEUE_LIMIT_EXCEEDED`. Semua jalur yang menambah job aktif, termasuk requeue P1 jika dibuat, mengikuti urutan lock client → job. Scheduler/retry mengubah state job yang sudah masuk hitungan.
- **Ambang global:** sebelum insert baru, periksa total job aktif yang sama dan outbox `pending`/`publishing`. Pada `API_OUTSTANDING_HIGH_WATERMARK` atau `API_OUTBOX_HIGH_WATERMARK`, kembalikan `503 SYSTEM_BACKPRESSURE` dengan `Retry-After`. Ukur profil query dan umur antrean. Pemeriksaan global bersifat operasional di bawah concurrency lintas client. Jika Redis putus, API tetap menyimpan job baru selama MySQL dan ambang mengizinkan, lalu dispatcher mengejar backlog setelah Redis pulih.
- **Per-queue worker budget:** cegah satu queue (contoh `report`) memonopoli worker untuk job `email`; pisahkan pool atau terapkan semaphore per queue.
- **Feedback:** metrik queue waiting age, in-flight, slot utilization, Redis PEL, outbox oldest pending. Threshold awal hanya hipotesis, diukur lewat load test.

### 28.3 Contoh matrix overflow

| Kondisi | Respons |
|---|---|
| Client melebihi RPS | `429 RATE_LIMITED`, `Retry-After` |
| Kuota outstanding client terlampaui | `429 CLIENT_QUEUE_LIMIT_EXCEEDED` |
| Total job aktif atau backlog outbox mencapai safety threshold | `503 SYSTEM_BACKPRESSURE`, `Retry-After` |
| Redis outage dan MySQL siap | Terapkan limiter dan ambang yang sama. Terima job baru selama ada kapasitas; dispatch menyusul |
| Worker process kehilangan slot | Stop baca Redis, bukan over-consume |

Tambahkan `antriaga_worker_inflight{queue}`, `antriaga_worker_available_slots{queue}`, `antriaga_api_rejected_total{reason}`, `antriaga_queue_wait_seconds`, `antriaga_outbox_oldest_pending_seconds`. Jangan mengklaim strict FIFO karena banyak worker dan redelivery dapat mengubah urutan penyelesaian.

---

## 29. Recovery Race-Condition Hardening (P0)

### 29.1 Mengapa `XAUTOCLAIM` saja tidak cukup

`XAUTOCLAIM` **mengubah pemilik** Redis PEL segera setelah menemukan message yang idle. Namun job pada MySQL bisa tetap memiliki **lease aktif**, misalnya handler yang lama dan heartbeat terlambat atau ketika pemeriksaan terjadi bersamaan dengan renewal. Karena itu `PEL idle` **bukan** bukti eksekusi gagal. Untuk MVP yang lebih mudah dipahami, gunakan `XPENDING` untuk enumerasi, validasi status/lease di MySQL, lalu `XCLAIM` dengan batas `min-idle` untuk pesan yang perlu takeover. Lakukan validasi ulang DB dalam transaksi sebelum melakukan perubahan job; command Redis/DB tidak bisa atomic lintas sistem.

### 29.2 Algoritme recovery (DB-leased, Redis-owned)

```mermaid
flowchart TD
    A[XPENDING per queue dan consumer group] --> B[Load job by job_id + enqueue_seq]
    B --> C{Safe to ACK per section 8.4?}
    C -->|Yes| D[XACK setelah bukti state durable, tanpa execute]
    C -->|No| E{Status processing dan lease DB aktif?}
    E -->|Yes| F[Skip; monitor, jangan reset / ACK]
    E -->|No| G[XCLAIM dengan min-idle guard]
    G --> H{Claim Redis benar-benar berhasil?}
    H -->|No| I[Retry next scan]
    H -->|Yes| J[DB TX: lock job; recheck lease, token, seq]
    J --> R{Safe to ACK per section 8.4?}
    R -->|Yes| D
    R -->|No| K{Masih eligible untuk recovery?}
    K -->|No| L[No state change; biarkan owner aktif ACK atau scan lagi]
    K -->|Yes| M[Recover queued or expired processing per state rules]
    M --> N[Commit DB]
    N -->|Success| O[XACK old claimed message]
    N -->|Failure| P[No XACK; repeat later]
```

- Pastikan `XCLAIM` mengembalikan message yang **benar-benar** berhasil diambil; `min-idle` dapat gagal bila message baru diproses pihak lain.
- Setelah `XCLAIM`, **selalu recheck DB** di transaksi karena lease/status bisa berubah sesaat setelah preflight check. Jika ternyata lease aktif, jangan `XACK` dan jangan requeue; original worker tetap dapat `XACK` saat sukses (ack berbasis group/message ID), atau message diperiksa ulang setelah lease benar-benar expired.
- Jika `processing`, gunakan lock dan waktu MySQL setelah lock sesuai §9.2. Recovery hanya boleh mengubah baris bila status, generation, dan token yang diperiksa masih cocok, serta `lease_expires_at <= db_now`. Tutup attempt aktif sebagai `lease_expired` dan kosongkan lease. Bila `attempts < max_attempts`, naikkan `enqueue_seq` dan buat outbox baru dalam transaksi yang sama. Bila jatah habis, commit `dead` tanpa outbox baru. `attempts` tetap bernilai sama sampai worker berikutnya berhasil claim.
- Jika job masih `queued` dengan generation yang sama setelah `XCLAIM`, lock job dan naikkan `enqueue_seq` dengan guard status `queued` dan generation asal. Buat outbox `pending` untuk generation baru dengan `available_at=db_now` dalam transaksi yang sama. Status tetap `queued`, tanpa mengubah counter atau baris attempt. ACK pesan lama setelah commit. Jika worker lebih dulu claim dan lease masih aktif, pertahankan pesan di PEL. Jika reaper atau reconciler lebih dulu menaikkan generation, pesan lama aman di-ACK menurut §8.4. Reconciler untuk pesan hilang menggunakan guard generation yang sama.
- Terapkan §8.4 juga setelah recheck `XCLAIM`. Job terminal, generation lama, atau `retrying` dengan generation sama dan jadwal tersimpan dapat di-ACK tanpa perubahan MySQL. Untuk `retrying`, jangan republish. Jika transaksi recovery gagal, tunda ACK sampai DB membuktikan pesan lama aman.
- MySQL menolak heartbeat dan hasil worker setelah expiry meski reaper belum mengganti status atau token. Efek eksternal yang telah dimulai masih mungkin selesai. Gunakan **stable downstream idempotency key**.
- DB sweeper dijalankan **terpisah** dari Redis PEL scan agar recover tetap berlangsung ketika stream/PEL hilang. Sweeper juga harus memakai `SELECT ... FOR UPDATE`/CAS yang sama.

Alternatif performa setelah E2E stabil: `XAUTOCLAIM` scanning dapat dipakai bila state machine secara eksplisit menangani ownership pindah sebelum DB check; jangan menyatakan kedua pendekatan identik tanpa menguji race. **Selalu** gunakan `min-idle`, timestamp UTC, dan conditional update.

### 29.3 Test race injection

1. `XPENDING` menunjukkan idle; heartbeat berhasil tepat sebelum `XCLAIM` → jangan requeue.
2. Dua reaper mengklaim pending message yang sama → paling banyak satu committed recovery generation.
3. Worker menyelesaikan `completed` setelah Redis claim tapi sebelum DB TX recovery → reaper mendeteksi terminal dan hanya ACK.
4. DB commit recovery gagal setelah `XCLAIM` → tidak ACK, dapat dipulihkan di scan berikutnya.
5. Isi job `processing` dan attempt aktif dengan token yang sama serta expiry yang sudah lewat menurut MySQL. Jalankan heartbeat dan setiap varian finalisasi (`completed`, `retrying`, `dead`) sebelum reaper dimulai. Masing-masing menghasilkan `ErrLostLease`. Verifikasi status, attempt, token, expiry, counter, dan outbox tetap sama, pesan masih di PEL, serta event transisi tidak ditulis. Lalu jalankan recovery dan verifikasi bahwa attempt ditutup sekali. Jika masih ada jatah, hanya satu generation dan outbox baru dibuat. Jika jatah habis, job berakhir `dead` tanpa outbox baru. Hasil lama juga ditolak setelah recovery.
6. Uji rekonsiliasi §29.4 dengan backlog normal, kehilangan pesan yang terkonfirmasi, broker error, dan generation yang berubah sebelum lock.
7. Redis PEL dipindah pemilik tetapi DB lease masih aktif → worker asli tetap dapat selesai/ACK, tidak ada false-retry.
8. Uji batas dengan waktu keputusan tetap `T` pada presisi milidetik melalui fixture internal pengujian guard SQL: expiry `T + 1 ms` masih aktif, expiry `T` dan `T - 1 ms` kedaluwarsa. Worker dan recovery harus mengambil keputusan yang saling melengkapi. Jalur runtime selalu membaca waktu MySQL setelah lock.
9. Pegang lock job pada transaksi A. Saat lease masih aktif, mulai heartbeat atau finalisasi pada transaksi B dan tunggu sampai lock wait teramati. Gunakan koneksi terpisah untuk mengamati `UTC_TIMESTAMP(3) >= lease_expires_at`, kemudian lepaskan lock A tanpa mengubah token atau status. B harus menolak operasi sebagai `ErrLostLease`. Sinkronkan lewat lock/barrier dan polling berbatas waktu, tanpa mengandalkan durasi `sleep` untuk mengurutkan transaksi.

10. Hentikan worker setelah `XREADGROUP` dan sebelum claim MySQL. Recovery membuat tepat satu generation dan outbox baru tanpa menaikkan `attempts`, lalu ACK setelah commit. Uji dua reaper serentak, worker yang lebih dulu memperoleh lease aktif, dan rollback insert outbox yang menyisakan pesan pending.
11. Paksa `XACK` gagal setelah commit `retrying`. Sebelum due, reaper harus bisa ACK tanpa mengubah `scheduled_at` atau `attempts`, memanggil handler, atau membuat outbox. Pada saat due, scheduler membuat tepat satu generation baru.

---

### 29.4 Rekonsiliasi job queued di luar PEL

Reaper PEL menangani pesan yang sudah diterima consumer tetapi ditinggalkan. Reconciler di bawah ini memulihkan pesan yang publish-nya sudah tercatat tetapi entrinya hilang. Umur job hanya membantu mengurutkan scan.

1. Scan batch terbatas dengan cursor stabil. Ambil job `queued` bersama row outbox untuk `enqueue_seq` aktif. Jika row masih `pending` atau `publishing`, dispatcher menangani publish atau reclaim lease. Jika row tidak ada atau berstatus `published` tanpa `published_stream_id`, catat pelanggaran invariant untuk diagnosis. Pada jalur normal, dispatcher menyimpan ID dan status `published` dalam satu commit.
2. Untuk outbox `published` dengan ID, lakukan `XRANGE <stream> <published_stream_id> <published_stream_id> COUNT 1` pada Redis primary di luar transaksi MySQL. Hasil kosong yang sukses mengonfirmasi entri hilang. Entri valid dengan identitas event berbeda di ID itu juga mengonfirmasi bahwa publish yang dicatat sudah hilang. Jika entri yang sama masih ada, biarkan job menunggu worker atau reaper PEL. Error koneksi, timeout, atau envelope yang tidak dapat diverifikasi tidak menjadi bukti kehilangan.
3. Setelah kehilangan terkonfirmasi, mulai transaksi MySQL, lock job terlebih dahulu lalu row outbox terkait. Cek ulang bahwa job masih `queued`, generation sama, dan outbox masih `published` dengan ID yang diperiksa. Jika cocok, naikkan generation dan buat tepat satu outbox baru dengan `available_at` dari waktu MySQL. Commit tanpa mengubah attempts. Jika state berubah sebelum lock, batalkan keputusan dan lanjutkan scan.
4. Generation baru yang masih menunggu dispatcher tidak memenuhi syarat pemeriksaan kehilangan. Simpan correlation ID dan bukti probe pada log teredaksi. Emit `reconciler.job_republished` setelah commit. Jalur ini tidak mengirim ACK.

Uji antrean backlog yang masih memuat pesan, pesan hilang setelah reset atau trimming, Redis error, ID yang dipakai ulang dengan event berbeda, dua reconciler serentak, dispatcher belum selesai, dan worker yang sukses claim sebelum recheck. Uji group hilang saat stream dan worker tetap hidup: dua consumer menangani `NOGROUP` serentak, group siap tanpa reset cursor yang sudah dibuat, dan job diproses tanpa restart. Backlog sehat tidak membuat generation baru. Untuk satu kehilangan yang dikonfirmasi, hanya satu transaksi membuat generation baru.

---

## 30. Business Payload Versioning dan Rolling Deployment (P0)

Pisahkan **`schema_version` transport message** (format Redis event) dari **`payload_version` bisnis** (format payload untuk registered job type). `payload_version` disimpan di **MySQL `jobs`** saat submit dan selalu dibawa ke handler dari DB. Jangan menganggap perubahan format JSON aman hanya karena service Go bisa deploy ulang.

```json
{
  "type": "email.send",
  "payload_version": 1,
  "payload": {
    "to": "sample@example.com",
    "template": "welcome",
    "subject": "Welcome"
  }
}
```

Registry sebaiknya memetakan `(job_type, payload_version)` → handler/decoder:

```go
type HandlerKey struct {
    Type    string
    Version uint16
}

// Misal registry.Register(HandlerKey{Type:"email.send", Version:1}, v1Handler)
// dan registry.Register(HandlerKey{Type:"email.send", Version:2}, v2Handler).
```

**Compatibility contract:**

- `POST /jobs` menolak `payload_version` tidak didukung (`422 UNSUPPORTED_PAYLOAD_VERSION`), dan **tidak** menyimpan job rusak.
- Worker yang menemukan job valid yang versinya tidak bisa diproses karena rolling deployment **tidak langsung membuang ke `dead`**. Perlakukan sebagai masalah kompatibilitas deployment dengan alert/quarantine/controlled retry sementara sampai worker versi kompatibel tersedia; jangan membuat loop tanpa batas.
- **Expand–migrate–contract**: rilis worker yang menerima v1+v2 → aktifkan API menghasilkan/menerima v2 → tunggu backlog v1 terminal atau migrasikan secara aman → hapus v1 setelah window yang terdokumentasi.
- `payload_version` **immutable** selama job berjalan. Perubahan payload bisnis untuk requeue manual adalah operasi eksplisit yang terversi dan diaudit, bukan edit tersembunyi.
- Integration test harus mencakup job v1 yang dibuat sebelum deploy, diproses setelah deploy v2; juga job versi unsupported, scheduled job yang menunggu lama, dan hasil rollback versi worker.
- Payload `result` boleh memiliki `result_version` saat struktur output mulai berubah (P1), tetap backward-compatible untuk endpoint query.

---

## 31. Dead Job Management dan Manual Requeue (P1)

MVP tetap menyimpan `dead` + riwayat `job_attempts`; versi P1 menambahkan operasi remediation yang aman. Istilah *dead-letter* di Antriaga merujuk pada **terminal state durable di MySQL** (dan opsional dead-letter stream fase lanjutan), bukan klaim bahwa Redis otomatis memindahkan dead jobs ke queue terpisah.

### 31.1 API dan safeguards

| API | Scope | Semantik |
|---|---|---|
| `GET /api/v1/jobs?status=dead` | `jobs:read` | Daftar dead jobs **client tersebut** |
| `GET /api/v1/jobs/:id/attempts` | `jobs:read` | Riwayat error_code dan attempt, redacted |
| `GET /api/v1/jobs/:id/events` (P2) | `jobs:read` | Timeline durable, jika fitur aktif |
| `POST /api/v1/jobs/:id/requeue` | `jobs:requeue` | Command eksplisit, audit wajib, hanya owner |

Payload operator (contoh kontrak fase P1):

```json
{
  "reason": "downstream recovered; approved by oncall",
  "reset_attempts": true
}
```

`reason` wajib, dibatasi panjang, disanitasi, dan tidak berisi credential. `reset_attempts` hanya diizinkan dengan scope/role khusus atau kebijakan server; untuk meminimalkan kompleksitas, **rekomendasi** buat **job baru dengan `source_job_id`** untuk manual requeue, karena histori job asli tetap immutable dan attempt_number tidak bentrok. Jika memilih reaktivasi ID lama, wajib reset attempts dengan audit trail dan mekanisme fencing yang benar; **jangan** mencampurkan kedua model tanpa ADR.

Implementasi pilihan **job baru** memerlukan migration P1 untuk `jobs.source_job_id CHAR(36) NULL` dengan FK/index yang sesuai, atau link table `job_requeues(source_job_id,new_job_id,operation_id,reason,actor_id,created_at)`; desain ini belum menjadi bagian dari migration MVP. Idempotency manual requeue perlu `Requeue-Request-ID` atau dedicated operation idempotency agar double-click tidak membuat dua job requeue. Periksa side effect eksternal yang mungkin terjadi sebelum gagal, terutama payment, sebelum mengizinkan operator melanjutkan. Rate-limit requeue dan minta alasan operator. `404` tetap digunakan untuk cross-client access.

### 31.2 Unit/integration tests

- `dead` akibat invalid payload tidak otomatis di-requeue oleh system scheduler.
- Requeue berhasil → job baru berlink `source_job_id`, outbox atomik, audit event durable.
- Dua command requeue identik → hanya satu job baru.
- Manual requeue tidak melewati `allowed_job_types`, scope, dan owner check.
- Side effect idempotency tetap sama atau berbeda **sesuai kontrak bisnis**; untuk payment, jangan menghasilkan second capture karena membuat job ID baru.

---

## 32. Data Retention, Cleanup, dan Redis Stream Trimming (P1)

Retensi adalah **kebijakan operasional terukur**, bukan sekadar menjalankan `DELETE` berdasarkan umur. Nilai berikut hanya default awal untuk demo/staging, **bukan rekomendasi regulasi** untuk data pembayaran atau pelanggan:

| Data | Usulan awal | Catatan |
|---|---|---|
| `jobs.completed` | 30 hari | Simpan ringkasan, archive sebelum purge jika dibutuhkan |
| `jobs.dead` | 90 hari | Lebih panjang untuk investigasi; operator dapat memperpanjang hold |
| `job_attempts` | Mengikuti parent job | Tidak delete lebih awal dari job induk |
| `outbox_events.published` | 7 hari setelah confirmed published & safe reconciliation | Pertahankan row outbox generation aktif selama job masih `queued` agar `published_stream_id` tersedia untuk §29.4. Outbox `pending/publishing` tidak dihapus otomatis |
| `job_events` (P2) | 30–90 hari, mengikuti job | Data yang diaudit dapat memerlukan retensi berbeda |
| Structured logs | 14–30 hari | Diatur log collector, bukan oleh aplikasi saja |
| Redis Stream | Trim hanya eligible acked/settled messages | Baseline Redis 7 memiliki batasan API trim |
| MinIO report output | Ikuti retensi job dan kebijakan object orphan | Pertahankan hasil yang masih direferensikan. Bersihkan percobaan yang kalah hanya setelah attempt berhenti dan hasil commit terverifikasi |

**MySQL cleanup worker (`cmd/maintenance` atau cron terkontrol):**

1. Cari terminal jobs yang melewati cutoff dalam batch kecil (misalnya 100–500) dengan index; hanya `completed/dead/cancelled` dan bukan legal/operational hold.
2. Pilih retention class, archive ke storage yang sesuai bila diwajibkan, catat proof/metrics.
3. Dalam transaction pendek, hapus dependents (`job_events` jika ada, `job_attempts`, `outbox_events`) lalu parent `jobs` dengan FK tetap konsisten. Tunda purge parent dan data outbox jika state tersebut masih dibutuhkan untuk membuktikan settlement entry Redis. Pada jalur baseline, selesaikan trim yang aman dan verifikasi referensi stream sudah hilang sebelum menghapus bukti durable terkait.
4. Throttle per batch untuk menekan lock, replication lag, dan I/O; catat rows deleted, oldest age, failures.
5. Untuk data MySQL yang besar, evaluasi partitioning/archiving secara terpisah dengan pengujian FK dan query pattern.

**Redis 7 baseline:** trim hanya prefix berurutan yang terbukti aman, dengan scan batch terbatas. Ambil `last-delivered-id` dari `XINFO GROUPS` dan ID pending tertua dari `XPENDING` untuk setiap consumer group yang diharapkan. Kandidat harus berada pada atau sebelum posisi delivery semua group, sebelum ID pending tertua jika ada, melewati batas umur, dan sudah settled menurut MySQL sesuai §8.4. PEL kosong tetap memerlukan pemeriksaan posisi delivery. Pertahankan entry dengan envelope atau state durable yang belum bisa dibuktikan.

Pilih threshold `MINID` sehingga hanya ID yang lebih kecil dari batas aman terhapus. Sebelum trim, recheck prefix dan identitas entry, himpunan group, posisi delivery, dan batas pending secara atomik di Redis. Tunda jika group hilang, dibentuk ulang, posisi mundur, atau salah satu pemeriksaan gagal. Jalankan trimming dalam maintenance window yang disepakati. Operasi create/reset group ditangguhkan selama perhitungan dan penerapan batas. Jika terjadi failover Redis, hitung ulang dari primary baru. Uji restart sebelum mengaktifkan trim otomatis. Opsi `ACKED` tersedia mulai Redis 8.2. Reconciler tetap menangani kehilangan stream yang terkonfirmasi.

**Acceptance:** setelah cleanup, state nonterminal, outbox pending, dan pesan belum delivered tetap tersedia. Uji worker berhenti dengan PEL kosong dan backlog baru, serta batas pending tertua dan group yang dibentuk ulang. Foreign keys valid; log audit cleanup tidak memuat payload penuh; log pembatasan bisa menunjukkan kemajuan dan error tanpa mengisi storage tanpa batas.

---

## 33. Operational Runbooks, Incident Response, dan SLO (P1)

Sediakan `docs/runbooks/*.md` berisi **symptom → pemeriksaan → tindakan aman → validasi pemulihan → eskalasi**. Jangan menjalankan `XACK`, `XDEL`, `DELETE`, atau perubahan status manual tanpa memastikan invariant MySQL dan mencatat incident/operator action. Perintah berikut read-only untuk diagnosis awal.

### 33.1 Kumpulan runbook

| Insiden | Sinyal | Diagnosa aman | Tindakan awal |
|---|---|---|---|
| Redis tidak tersedia | publish errors, outbox pending naik | Redis health, `antriaga_outbox_oldest_pending_seconds` | Pulihkan Redis; biarkan dispatcher retry outbox, jangan clear DB |
| Worker mati/terjebak | active workers turun, oldest queued naik | worker logs, lease expiry, `XPENDING` | Investigasi koneksi/CPU, restart worker sehat, pantau reaper |
| Banyak `dead` | `dead_rate` tinggi | group by `error_code`, job type, provider status | Perbaiki upstream/konfigurasi; jangan mass-requeue blind |
| MySQL gagal | `db.commit_failed` meningkat, API 503 | DB health, connection saturation, slow query | Pulihkan DB; jangan ACK pesan yang belum commit |
| Outbox backlog | oldest pending tinggi | batch dispatcher, per-job error_code, Redis health | Scale dispatcher dengan claim lock aman, revisi concurrency |
| Lease loss berulang | `worker.lease_lost_total` tinggi | heartbeat RTT, GC pauses, handler timeout, DB contention | Tune timeout/lease dan sembuhkan bottleneck sebelum reclaim |
| Disk tinggi | MySQL/Redis/log volume naik | retention dashboard, PEL, table sizes | Jalankan cleanup aman/archival, jangan trim pending data |
| API key diduga bocor | auth errors aneh/usage spike | `api_key_id`, client, IP/rate (sesuai privasi) | Revoke key, issue new key, evaluasi jobs yang diterima, audit |

### 33.2 Query read-only untuk incident

```sql
SELECT client_id, queue, status, COUNT(*) AS total
FROM jobs
GROUP BY client_id, queue, status
ORDER BY total DESC
LIMIT 50;

SELECT id, job_id, enqueue_seq, status, publish_attempts, available_at
FROM outbox_events
WHERE status IN ('pending', 'publishing')
ORDER BY available_at ASC
LIMIT 50;

SELECT id, client_id, type, attempts, max_attempts,
       lease_owner, lease_expires_at, last_error_code
FROM jobs
WHERE status = 'processing' AND lease_expires_at <= UTC_TIMESTAMP(3)
ORDER BY lease_expires_at
LIMIT 50;
```

```bash
# Read-only diagnosis pada lingkungan lokal; jangan masukkan password ke command history.
redis-cli XPENDING antriaga:queue:email antriaga-workers
redis-cli XINFO GROUPS antriaga:queue:email
```

**Contoh SLO awal** (dibuat sebagai target sebelum diukur, bukan klaim telah tercapai): 99.9% job yang *diterima* mencapai terminal state dalam jendela sesuai tipe job bila downstream sehat; P95 queue waiting age di bawah target per queue; zero accepted-job loss setelah restart yang diuji. Pisahkan SLO internal Antriaga dari availability external mock/provider. Definisikan error budget & alert window berdasarkan beban nyata.

Runbook mencatat owner, revisi, link dashboard, dan langkah verifikasi bahwa `outbox_pending`, queue oldest wait, serta lease expired telah turun setelah perbaikan.

---

## 34. Durable Job Event Timeline (P2 / setelah MVP)

Structured `slog` menjelaskan runtime, tetapi **tidak transaksional** dan retensinya tergantung log collector. Untuk timeline yang dapat di-query secara konsisten, tambahkan `job_events` sebagai **audit histori perubahan state**. Persist event bersamaan dengan transaksi state di MySQL; jangan menghasilkan durable event `completed` hanya dari output log sebelum commit.

### 34.1 Schema event

```sql
-- Migration terpisah fase P2, misalnya 000006_create_job_events.up.sql
CREATE TABLE job_events (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    job_id CHAR(36) NOT NULL,
    client_id CHAR(36) NOT NULL,
    event_type VARCHAR(80) NOT NULL,
    attempt_number INT NULL,
    from_status VARCHAR(24) NULL,
    to_status VARCHAR(24) NULL,
    actor_type VARCHAR(24) NOT NULL,
    actor_id VARCHAR(128) NULL,
    request_id VARCHAR(100) NULL,
    metadata JSON NULL,
    occurred_at DATETIME(3) NOT NULL,
    KEY idx_job_events_job_time (job_id, occurred_at, id),
    KEY idx_job_events_client_time (client_id, occurred_at),
    CONSTRAINT fk_job_events_job FOREIGN KEY (job_id) REFERENCES jobs(id),
    CONSTRAINT fk_job_events_client FOREIGN KEY (client_id) REFERENCES clients(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`metadata` adalah allowlisted ringkasan (`error_code`, `retry_delay_ms`, `stream_id` bila perlu) **tanpa** payload mentah/secret/PII. `event_type` meniru katalog log saat relevan, tetapi event durable adalah *committed transition* bukan setiap debug/heartbeat. `actor_type` misalnya `api`, `worker`, `scheduler`, `reaper`, `operator`. `actor_id` bukan token API key. Gunakan `api_key_id` khusus bila perlu audit tindakan client, dengan kebijakan privasi/retensi yang jelas.

### 34.2 Endpoint dan contoh hasil

```http
GET /api/v1/jobs/{id}/events?page=1&limit=20
X-API-Key: <REDACTED>
```

```json
{
  "data": [
    {"event":"job.accepted","occurred_at":"2026-11-18T05:00:00Z","to_status":"queued"},
    {"event":"job.claimed","occurred_at":"2026-11-18T05:00:01Z","to_status":"processing","attempt_number":1},
    {"event":"job.retry_scheduled","occurred_at":"2026-11-18T05:00:02Z","to_status":"retrying","attempt_number":1,"metadata":{"error_code":"UPSTREAM_UNAVAILABLE","retry_delay_ms":5000}},
    {"event":"job.completed","occurred_at":"2026-11-18T05:00:08Z","to_status":"completed","attempt_number":2}
  ],
  "meta":{"page":1,"limit":20,"total":4}
}
```

Query **wajib** melakukan owner check `jobs.client_id = principal.ClientID` sebelum mengambil event. `GET /events` memakai scope `jobs:read`. Event bisa dicari dengan `job_id` dan time, tetapi jangan expose dashboard lintas client tanpa auth khusus. Simpan event `job.requeued` bila operator membuat job baru dan tautkan `source_job_id` di metadata/reference yang aman.

### 34.3 Implementasi, storage, dan acceptance

- Pada transaksi `ClaimJob`, `FinishJob`, `DispatchDueJobs`, `RecoverStaleJobs`, tulis 1 row event setelah update state, **sebelum TX commit**.
- Jangan membuat event durable untuk Redis `XACK` success jika tidak ada DB state transition; cukup `slog`/metrics kecuali membutuhkan operational audit khusus.
- Paging, retention dan index wajib agar satu job dengan ribuan attempts/events tetap dapat di-query aman.
- E2E: DB commit rollback → **tidak** ada event sukses; duplicate Redis delivery → hanya satu event committed claim; cross-client events → `404`.
- Dokumen Swagger menambah DTO `JobEventResponse` **ketika fitur diimplementasikan**, bukan mengiklankan endpoint P2 sebagai MVP.

---

## 35. Ringkasan Keputusan Final MVP

**Antriaga** dikembangkan sebagai **distributed background job engine** menggunakan Go Fiber v3 dengan Clean Architecture. HTTP API menggunakan client-based API keys yang dihasilkan melalui admin CLI, diverifikasi HMAC, dan mengikat setiap job/query ke identitas `client_id` serta scopes. HTTP API hanya mengurus auth, validasi, persistence, dan query. MySQL menyimpan job, attempts, dan transactional outbox sebagai source of truth. Redis Streams mendistribusikan ready-job messages ke worker pool. Scheduler melepas scheduled/retrying job, dispatcher mempublish outbox, dan reaper mengurus lease expiry, Redis PEL reclaim yang race-safe (`XPENDING`/guarded `XCLAIM`), serta reconciliation. Worker menggunakan bounded concurrency dan handler registry berversi `(job_type, payload_version)` untuk empat workload statis; setiap attempt tercatat dan seluruh eksekusi berpegang pada at-least-once + idempotency. Semua komponen menghasilkan `slog` JSON terstruktur dengan event catalog yang membedakan job sukses, handler failure, retry, dead, outbox failure, Redis ACK failure, dan recovery; `job_id` menghubungkan seluruh proses dan `request_id` menghubungkan request HTTP ke job awal.

**Prioritas implementasi:** autentikasi client & isolation, correctness/recovery races, bounded concurrency, payload compatibility, dan observability adalah P0. Dead-job management, safe cleanup/retention, runbooks adalah P1; queryable durable event timeline adalah P2. Fitur seperti priority queue, dashboard, atau adapter broker lain ditunda.

> **Catatan status:** Dokumen ini adalah *system design + implementation blueprint*, bukan klaim bahwa source code, test suite, Docker image, atau Swagger spec telah dibuat/dijalankan. Snippet kode dan Compose ditujukan sebagai panduan implementasi yang harus disatukan, diuji, dan disesuaikan saat coding. **Bagian 36–44 mengatur AI-assisted implementation serta skills eksternal (v4).**

---

## 36. AI-Assisted Development / Vibe Coding: Operating Model (P0 untuk proses kerja)

Bagian 36–44 adalah **notes operasional untuk AI coding agent** (ChatGPT/Codex, Claude Code, Cursor, Copilot, atau agent lain). *Vibe coding* pada sistem antrean terdistribusi harus tetap **spec-driven, reviewable, dan test-driven**. Kemudahan membuat kode tidak boleh mengorbankan invariants pada Bagian 6–9, 16, 27–30.

### 36.1 Urutan sumber kebenaran

Saat agent mendapat instruksi yang bertentangan, gunakan urutan ini:

1. **Keamanan, data nyata, dan persetujuan operator** (jangan menjalankan aksi destruktif tanpa approval).
2. **Keputusan arsitektur yang disetujui** dalam dokumen v4 + ADR aktif.
3. **Keadaan nyata repository saat itu**: file, modul, migration yang sudah diterapkan, kontrak test yang sudah dipertahankan. Bila berbeda dengan desain, **laporkan discrepancy**; jangan diam-diam menimpa.
4. **`AGENTS.md` repository** dan checklist milestone/issue.
5. **Community Agent Skills**: saran proses/idiom, bukan izin mengganti teknologi atau semantik job.
6. Asumsi agent. Jika asumsi berdampak pada correctness/security, tulis dulu dalam plan dan minta keputusan sebelum perubahan besar.

### 36.2 Batas implementasi yang tidak boleh dilompati

| Jangan lakukan otomatis | Kenapa | Tindakan pengganti |
|---|---|---|
| Mengganti Fiber ↔ framework lain atau MySQL ↔ database lain | Mengubah kontrak desain | ADR + review eksplisit |
| Mengganti Redis Streams menjadi Redis Lists/ PubSub | Hilang consumer-group dan PEL recovery | Ikuti §8 dan §9 |
| Meng-ACK Redis sebelum commit DB yang memastikan state aman | Berpotensi kehilangan job | Unit/integration test urutan commit → ACK |
| Mengklaim exactly-once | Bertentangan dengan model delivery | Dokumentasikan at-least-once + idempotency |
| Membuat custom arbitrary executor / URL bebas | Scope dan permukaan serangan meningkat | Hanya 4 job type dan target mock server-configured |
| Menyimpan plaintext API key | Credential leakage | HMAC/key digest + secret shown once (§27) |
| Membuat giant diff lintas 15 package untuk perubahan kecil | Sulit dibuktikan benar | Vertical slice dengan acceptance test |
| Menambahkan library tanpa alasan/versi | Dependency & security drift | Jelaskan tradeoff, pin version, update `go.mod` |
| Menyatakan “tests pass” tanpa menjalankan command | False confidence | Laporkan `not run`, alasan, dan next verification |

### 36.3 Prinsip sesi dan context hygiene

- **Satu pekerjaan = satu target perilaku** dengan tes dan acceptance criteria; jangan minta agent membangun seluruh Antriaga dalam satu prompt.
- Maksimal **satu boundary arsitektur utama** per tugas (misalnya domain+usecase, adapter+integration test, atau HTTP+DTO+Swagger). Untuk perubahan lintas komponen, pecah fase.
- Setiap prompt menyertakan **scope**, **file yang relevan**, **referensi § v4**, **perilaku yang diinginkan**, **test**, dan **non-goals**.
- Saat konteks panjang/agent berganti, gunakan `docs/ai/worklog.md` dan `docs/ai/decisions.md`, bukan mengandalkan ingatan chat.
- Agent tidak boleh membuat hasil benchmarking, coverage, atau deployment status palsu. Angka baru wajib berasal dari output tool/test sungguhan.
- Review manual khusus untuk **migrations, auth, ownership query, idempotency, leases, recovery, side effects, dan cleanup**.

---

## 37. Repository Agent Rules: `AGENTS.md` dan Folder AI

`AGENTS.md` di root adalah **kontrak kerja pendek** yang selalu dapat dibaca agent, bukan pengganti blueprint 3.000+ baris. Blueprint lengkap ada di `SYSTEM_DESIGN.md`, yang dirujuk oleh `AGENTS.md`.

### 37.1 File pendukung yang disarankan

| Path | Fungsi | Kapan diperbarui |
|---|---|---|
| `AGENTS.md` | Guardrails arsitektur, DoD, security, workflow agent | Saat kebijakan development berubah |
| `SYSTEM_DESIGN.md` | System design rujukan | Hanya setelah review keputusan desain |
| `design/adr/ADR-XXXX-*.md` | Alasan keputusan/alternatif/risiko | Saat ada tradeoff/breaking change |
| `docs/ai/task-template.md` | Template ticket/prompt per vertical slice | Saat format tugas berubah |
| `docs/ai/worklog.md` | Progress dan verifikasi aktual, bukan klaim | Di akhir setiap task |
| `docs/ai/decisions.md` | Ringkasan keputusan yang belum jadi ADR | Saat keputusan dibuat |
| `.agents/skills/<name>/SKILL.md` | Skill custom yang sudah direview | Ketika prosedur repetitif stabil |
| `.github/workflows/ci.yml` | Gates otomatis untuk membuktikan perubahan | Bersamaan dengan perubahan commands |

### 37.2 Isi `AGENTS.md` yang disarankan

Gunakan file pendamping `AGENTS.md` untuk isi lengkap. Minimal wajib mencakup:

1. Stack dan **4 job types**, non-goals, dependency direction.
2. Source of truth, outbox, DB-commit-before-XACK, lease fencing, at-least-once/idempotency.
3. Auth client-based API key, scope dan ownership; secret redaction.
4. Structured logs dan Swagger yang wajib sinkron dengan perubahan kode.
5. Working protocol: **inspect → plan → red test → minimal implementation → verification → report**.
6. Larangan perubahan destruktif dan instalasi skill/script sembarangan.

**Penting:** agent harus mengikuti `AGENTS.md` sekaligus melihat keadaan nyata repository. Jangan membuat `CLAUDE.md`, `.cursor/rules`, atau `.github/copilot-instructions.md` berisi instruksi yang saling bertentangan; jika perlu, buat file adapter singkat yang mengarah ke satu aturan kanonik `AGENTS.md`.

### 37.3 Template task singkat (`docs/ai/task-template.md`)

```markdown
# Task: <nama spesifik>
- Milestone / issue: <M1/M2/M3/M4, ticket>
- Goal: <satu perilaku yang harus bekerja>
- Referensi desain: <§X.Y, §Z>
- In scope: <file/package>
- Out of scope: <jelas>
- Invariants: <DB commit, idempotency, auth, dll>
- Acceptance tests: <nama test, edge cases>
- Migration/API/Swagger impact: <none / rincian>
- Verification commands: <perintah konkret>
- Risk and rollback: <singkat>
```

---

## 38. Workflow Perubahan Kecil: Plan → Test → Implement → Verify

### 38.1 Prosedur standar untuk agent

1. **Preflight:** baca `AGENTS.md`, bagian desain terkait, `go.mod`, package yang berubah, migrasi, test existing, dan Makefile. Jangan mengarang struktur package.
2. **Task plan:** jawab singkat “tujuan, file yang diubah, risiko correctness, test yang ditambah, non-goals”. Jika ada design mismatch, catat dan hentikan perubahan arsitektur sampai diputuskan.
3. **RED:** tulis test yang mereproduksi behavior/bug. Untuk concurrency gunakan barrier/channel/controlled fake clock; **hindari `time.Sleep()` yang membuat test flaky**.
4. **GREEN:** implementasi minimum hingga test lolos; pertahankan domain/application bebas dari Fiber/GORM/go-redis.
5. **REFACTOR:** rapikan error wrapping, penamaan Go, transaction boundaries, context propagation. Jangan mengubah behavior tanpa test.
6. **Verifikasi aktual:** jalankan pemeriksaan relevan di §42, simpan hasil perintah dan error yang terjadi; jika environment tidak punya Docker, tulis integration test **not run**, bukan *passed*.
7. **Documentation sync:** update migrasi, DTO/Swaggo, event logs, README, ADR, dan worklog jika terpengaruh.
8. **Handoff:** tulis apa yang berhasil, perubahan file, test output, risiko tersisa, dan task berikutnya yang paling kecil.

### 38.2 Pembagian task yang efektif

| Task size | Contoh yang baik | Jangan dijadikan satu task |
|---|---|---|
| S (1–3 file) | Implementasi pure backoff + table-driven tests | Seluruh retry scheduler + reaper + worker |
| M (3–8 file) | `CreateJob` use case + fake UoW + test outbox atomicity | Create API + auth + Redis + recovery + dashboard |
| M | MySQL `ClaimJob` adapter + transactional integration tests | Semua layer dari HTTP sampai mock payment |
| M | Auth middleware + principal extraction + HTTP tests | Seluruh management CLI dan key rotation dalam satu diff |
| L (pecah jadi 2–5 PR) | Crash recovery beserta lease/PEL races | Satu PR menyentuh seluruh runtime tanpa checkpoint |

Ukuran di atas pedoman review, bukan batas matematis. Jika satu behavior memerlukan file lebih banyak, tetap tulis plan dan pecah testable commits.

### 38.3 Kontrak laporan setelah task

```text
Task: <goal>
Changed files: <daftar file nyata>
Implemented: <perilaku dan batasan>
Tests added: <test dan failure scenarios>
Commands run: <perintah> -> PASS / FAIL / NOT RUN + alasan
Security check: <secrets, owner scope, privilege>
Docs changes: <Swagger, README, ADR, runbook>
Open issues: <risiko/assumptions>
Next task: <satu langkah konkret>
```

---

## 39. Repo Agent Skills yang Relevan untuk Antriaga

Daftar ini merupakan **referensi repository publik yang telah diperiksa pada 19 November 2026**. Isi/branch dan cara instalasi bisa berubah. Semua skills **opsional**: pasang yang menunjang task, bukan semuanya. Cek `SKILL.md`, `allowed-tools`, scripts, lisensi, dan izin sebelum dipakai; konten pihak ketiga tidak memiliki otoritas untuk mengesampingkan `AGENTS.md`/system design.

### 39.1 Prioritas repository

| Repo sumber | Skills yang dipilih | Kapan digunakan | Status |
|---|---|---|---|
| [eduardo-sl/go-agent-skills](https://github.com/eduardo-sl/go-agent-skills) | `go-architecture-review`, `go-error-handling`, `go-database`, `go-concurrency-review`, `go-test-quality`, `go-security-audit`, `go-observability`, `go-api-design`, `go-ci`, `go-openapi` | **Utama**: pola idiomatik Go, Clean Architecture, MySQL, tests, logger, security, dokumentasi API | **Mulai dari sini** |
| [obra/superpowers](https://github.com/obra/superpowers) | `writing-plans`, `test-driven-development`, `systematic-debugging`, `verification-before-completion`, `requesting-code-review` | Mengendalikan siklus kerja agent agar tidak langsung coding besar tanpa test dan verifikasi | **Sangat disarankan** |
| [ardanlabs/skills](https://github.com/ardanlabs/skills) | `use-modern-go`, `ardan-service-architecture`, `ardan-pr-review` | Review idiom Go/arsitektur/PR. Gunakan sebagai second opinion, bukan mengganti layering v4 | Tambahan |
| [trailofbits/skills](https://github.com/trailofbits/skills) | `insecure-defaults`, `differential-review`, `supply-chain-risk-auditor`, `audit-context-building` | Security review ketika auth, credentials, GitHub Actions, dependency, dan release dibuat | Review keamanan |
| [anthropics/skills](https://github.com/anthropics/skills) | `skill-creator`, `doc-coauthoring` | Opsional untuk membuat skill lokal khusus Antriaga atau memelihara dokumen desain | Opsional |

**Catatan integrasi:** `go-agent-skills` adalah repository Agent Skills kompatibel banyak agent; `obra/superpowers`/`ardanlabs`/`trailofbits` punya metode instalasi berbeda per agent. Jangan menganggap semua bisa diaktifkan dengan perintah sama. Nama-nama di atas adalah skill/fitur **yang dicantumkan oleh upstream**, bukan skill yang sudah diinstal di repository Antriaga.

### 39.2 Skill-to-task mapping (gunakan 1–3 per task)

| Pekerjaan Antriaga | Skill pertama | Skill pelengkap | Gate manual wajib |
|---|---|---|---|
| Desain package/domain/usecase/ports | `go-architecture-review` | `writing-plans` | Review dependency rule §4.3 |
| MySQL migrations + transactions + outbox | `go-database` | `go-test-quality` | Integrasi `TX rollback`, duplicate publish |
| REST Fiber v3 dan Swaggo | `go-api-design` | `go-openapi` | Validasi di docs Fiber/Swaggo yang dipin; UI/DTO test |
| Client API keys / HMAC / scopes | `go-security-audit` | `insecure-defaults` | Threat review, isolation tests, no plaintext/log secret |
| Worker concurrency dan shutdown | `go-concurrency-review` | `go-test-quality` | Race detector, leak/shutdown integration test |
| Retry, scheduler, reaper, lease fencing | `systematic-debugging` | `go-concurrency-review` | Fault-injection + crash-window matrix §9.6 |
| slog, metrics, structured events | `go-observability` | `go-test-quality` | No false success before commit, redaction tests |
| Refactor atau bug fix | `test-driven-development` | `verification-before-completion` | Reproduce bug, `go test`, check regression |
| PR security dan dependencies | `differential-review` | `supply-chain-risk-auditor` | Human review, `govulncheck`, scan secrets |
| Pipeline/CI and quality gates | `go-ci` | `ardan-pr-review` | Actual CI green, pinned versions |

**Batas rekomendasi:** `go-openapi` membahas praktik OpenAPI, sedangkan v4 tetap menggunakan **Swaggo Swagger 2.0** sesuai §13. Skill **tidak boleh memigrasikan tooling ke OpenAPI 3** tanpa ADR dan persetujuan.

### 39.3 Instalasi selektif (contoh berdasarkan instruksi upstream)

Contoh untuk agent yang mendukung CLI Agent Skills dan sudah memiliki Node/npm:

```bash
# Tinjau skills yang tersedia; belum memasang apa pun.
npx skills add eduardo-sl/go-agent-skills --list

# Contoh pemasangan subset untuk project ini; pilih agent target di prompt interaktif.
npx skills add eduardo-sl/go-agent-skills \
  --skill go-architecture-review \
  --skill go-error-handling \
  --skill go-database \
  --skill go-test-quality \
  --skill go-concurrency-review
```

Untuk skills proses dari Superpowers, ikuti cara instalasi **per agent** di [README upstream](https://github.com/obra/superpowers#installation). Untuk security skills dari Trail of Bits, rujuk [README upstream](https://github.com/trailofbits/skills#installation). Tinjau scripts dan permissions sebelum memberi akses tulis/terminal. Versi/tag atau commit skills yang akhirnya dipakai sebaiknya **dicatat dan dipin** dalam dokumentasi project agar perilaku agent dapat direproduksi.

---

## 40. Skill Lokal yang Sebaiknya Dibuat untuk Antriaga

Skill umum membantu idiom Go, tetapi tidak mengetahui correctness rules Antriaga. Setelah dua atau tiga task berulang, pertimbangkan **satu custom skill** yang *membaca spesifikasi lokal*, misalnya `.agents/skills/antriaga-reliability/SKILL.md`:

````markdown
---
name: antriaga-reliability
description: Review or implement Antriaga job creation, publish, claim, ACK, retry, lease recovery, and idempotency changes. Use whenever editing application execution/recovery, mysql transaction adapter, Redis Streams adapter, or worker runtime.
---

# Antriaga reliability guard

Before touching code, read `AGENTS.md` and system design §§6–9, 27–30.

## Checklist

1. Job + outbox transaction atomic? For immediate jobs, no publish before commit.
2. MySQL is source of truth; Redis message only carries ID/generation.
3. Every worker claim is conditional on status+generation+attempts, with a lease token.
4. Heartbeat and every finalize require processing status, matching lease token, and strict expiry checked with MySQL time after the job row lock. Lost lease means no mutation or XACK (§9.2).
5. Handler retries vs redelivery vs lease recovery vs outbox retry are distinct.
6. Downstream side effects use stable idempotency keys; do not claim exactly-once.
7. Recovery validates DB lease before and after claiming Redis PEL ownership.
8. Bounded concurrency: do not consume more than available execution slots.
9. Tests cover duplicate messages, crash windows, races, failed commit and XACK.
10. Emit structured slog events without secrets, and report actual test output.

If a rule is violated, report the exact invariant and file/line, not a generic suggestion.
````

**Saran:** jangan membuat 20 skill custom sejak hari pertama. Mulai dari `antriaga-reliability`, kemudian `antriaga-api-contract` (DTO/Swaggo/auth) dan `antriaga-release-check` hanya jika kebutuhan berulang terbukti. Evaluasi dengan task konkret: apakah skill mengurangi regresi, bukan sekadar menambah instruksi.

---

## 41. Prompt Pack — Siap Copy untuk AI Coding Agent

Semua prompt berikut berasumsi agent dapat membaca repository dan blueprint di `SYSTEM_DESIGN.md`.

### 41.1 Prompt sesi baru / repo onboarding

```text
You are implementing Antriaga, a Go Fiber v3 distributed background job engine.
Read AGENTS.md and SYSTEM_DESIGN.md before editing.
Inspect the real repository first; identify current milestone and what exists.
Return: current status, missing P0 components, package dependency violations,
and the smallest next task with precise files and acceptance tests.
Do not write code yet. Do not claim any tests were executed.
```

### 41.2 Prompt implementasi vertical slice

```text
Implement only: <ONE BEHAVIOR>. Milestone: <M1/M2/M3/M4>.
Refer to design sections <X.Y, Z>. In scope: <packages/files>.
Out of scope: <features to avoid>. Preserve MySQL source of truth,
at-least-once semantics, Clean Architecture boundaries, and client ownership.
First inspect existing code and propose plan, tests (RED), and risk.
Then implement the minimum code, run relevant tests, and report actual outputs.
Update Swagger/migration/logging only where this change requires it.
```

### 41.3 Prompt atomic CreateJob + scoped idempotency

```text
Implement CreateJob use case and its unit tests only.
Read §§7, 9.1, 11, 27. One transaction must persist jobs + outbox
for immediate jobs. client_id comes from authenticated principal, never body.
Dedup is scoped to client_id; same key with different request hash returns 409.
Test transaction rollback, duplicate key, cross-client key reuse,
unsupported payload_version, and correct slog events after commit.
Do not implement Redis publisher or mock external clients in this task.
```

### 41.4 Prompt Redis recovery / race review

```text
Review (do not rewrite) the current recovery implementation against §§8–9 and 29.
Inspect XPENDING/XCLAIM ownership changes, DB lease validity,
post-claim DB recheck, lease expiry with the token still current, lock-wait timing,
stale generation, and XACK timing. Apply the deterministic cases in §29.3.
Produce a numbered list of actual bugs with file:line, reproduction tests,
and the smallest safe patch. No theoretical bug claims without code evidence.
```

### 41.5 Prompt auth security review

```text
Audit API key generation/verification and client isolation against §27.
Check crypto/rand entropy, stored HMAC digest, pepper config, secret shown once,
rotation/revocation, scope authorization, 404 masking for cross-client jobs,
unique idempotency by client, rate limiting, and log redaction.
Add/identify negative tests. Do not print API keys, pepper, or live tokens.
```

### 41.6 Prompt bug fixing dan penutupan task

```text
Reproduce this bug with a deterministic failing test first: <bug>.
Trace the cause; do not make speculative edits. Fix with minimal code.
Run gofmt, go vet ./..., and relevant go test commands, including -race
for concurrency changes where supported. If integration tests need Docker
and Docker is unavailable, explicitly mark NOT RUN.
Report changed files, verified commands, remaining risks, and next step.
```

### 41.7 Prompt review pull request

```text
Review this diff for Antriaga correctness, not style alone.
Prioritize: DB transactions, duplicate execution, Redis ACK order,
lease races, per-client ownership and API key security, long-running
handler shutdown, and test gaps. Output findings by severity with exact
file:line references and reproduction guidance. If none found, state
remaining coverage limitations. Do not modify files during review.
```

---

## 42. Verification Gates untuk AI-Assisted Development

**Perintah di bawah adalah target CI/Makefile yang harus diimplementasikan**, bukan klaim sudah tersedia atau lulus. Jangan memberi test status `PASS` sebelum menjalankannya pada repository nyata.

### 42.1 Fast loop (setiap perubahan kecil)

```bash
gofmt -w <modified-go-files>  # hanya file Go yang disentuh
go vet ./...
go test ./... -count=1
```

Untuk memeriksa formatting di CI secara non-mutating, gunakan `gofmt -l` dan fail bila output tidak kosong. Untuk perubahan concurrency:

```bash
go test -race ./... -count=1
```

**Prasyarat race detector:** platform Go/CGO/compiler mendukung `-race`. Bila tidak tersedia, catat keterbatasannya dan pastikan gate dijalankan pada CI yang cocok sebelum merge.

### 42.2 Extended loop (setiap PR yang menyentuh adapter/runtime)

| Perubahan | Bukti wajib sebelum merge |
|---|---|
| Domain/use case | Unit tests + table-driven negative paths + no forbidden imports |
| MySQL adapter/migration | Testcontainers MySQL, TX rollback, `SKIP LOCKED`, migration up/down policy |
| Redis adapter/worker | Consumer group integration, duplicate message, ACK failure, slot-bound concurrency |
| Scheduler/reaper | Fault injection, lease races, idempotency, stale generation, failed DB commit |
| Auth/admin CLI | Key lifecycle tests, cross-client 404, scopes, secret redaction and expiry |
| HTTP/DTO | Fiber `app.Test`, validation, Swagger regenerated + diff check |
| Logging | Required events present, no secret/PII in events, no false success before commit |
| Release/dependency | Pin versions, `govulncheck`, secret scan, review image/deploy config |

### 42.3 Makefile contract yang sebaiknya dibuat

```makefile
# Target names are proposed; commands depend on actual repository setup.
.PHONY: fmt-check vet test test-race test-integration swag-gen swag-check build verify
fmt-check:
	@test -z "$$(gofmt -l $$(find cmd internal examples -name '*.go' -type f))"
vet:
	go vet ./...
test:
	go test ./... -count=1
test-race:
	go test -race ./... -count=1
test-integration:
	go test -tags=integration ./tests/integration/... -count=1
swag-gen:
	swag init -g cmd/api/main.go -d ./ --parseInternal -o ./docs
swag-check: swag-gen
	git diff --exit-code -- docs/
build:
	go build ./cmd/...
verify: fmt-check vet test build
```

Perhatikan: target `fmt-check` memakai `find` POSIX, dan `make` biasanya tidak terpasang di Windows. Sediakan `scripts/check.ps1` yang menjalankan `gofmt -l`, `go vet`, `go test`, dan `go build`. CI Linux memakai target Makefile di atas. `go test -race` di Windows membutuhkan `CGO_ENABLED=1` dan kompiler C (mis. MinGW-w64). Jika tidak tersedia, catat gerbang race sebagai NOT RUN lokal dan jalankan di CI. `test-integration` butuh tag/path nyata, Docker/Testcontainers, dan migrasi test. `swag-check` hanya valid jika CLI Swaggo sudah **dipin** dan generated docs dikomit. Atur error handling Makefile ketika folder `examples` belum tersedia saat Milestone 1.

### 42.4 Gate penutupan milestone

| Milestone | Blocker untuk dinyatakan selesai |
|---|---|
| M1 | Auth + owner scope; migrations & UoW; create/list/get; Swagger sinkron; rollback test |
| M2 | Actual consumer groups; bounded worker concurrency; outbox publish; success/dead E2E |
| M3 | Retry & scheduler; worker lease & recovery races; duplicate/ACK/crash E2E |
| M4 | Report output, dashboards/metrics, runbooks minimum, demo reproducible, release/security review |

**Stop condition:** jika test failure membuktikan job loss, double capture, cross-client data leak, plaintext secret, atau stale lease menimpa active job, jangan lanjut ke milestone berikutnya sebelum dibuat reproduksi dan perbaikan diverifikasi.

---

## 43. Implementation Backlog yang Cocok untuk Agent (Contoh Awal)

Untuk menjaga momentum tanpa membangun terlalu banyak sekaligus, mulailah dari task berikut:

| Urutan | Task | Acceptance minimum | Referensi |
|---:|---|---|---|
| 1 | Scaffold Go module, `cmd/*`, config, manual DI, `slog` JSON, CI skeleton | `go build ./cmd/...`, unit logger/redaction | §3–5, §16 |
| 2 | Domain Client, ApiKey, Job, JobStatus, payload versioning | State + validation tests, no infrastructure imports | §6, §27, §30 |
| 3 | Versioned SQL migrations `clients`, `api_keys`, `jobs`, `attempts`, `outbox` | MySQL Testcontainers migration smoke/rollback policy | §7, §27 |
| 4 | Admin create client & issue key (crypto/rand + HMAC) | Key shown only once, digest persisted; negative tests | §27 |
| 5 | Principal extraction, scope middleware, ownership | Invalid/revoked key denied; cross-client job `404` | §12, §27 |
| 6 | `CreateJob` application service + MySQL UoW | Job+outbox atomic, scoped idempotency, rollback test | §9.1, §11 |
| 7 | Fiber HTTP POST/GET/LIST + Swaggo | Validation, response contract, regenerated docs, HTTP tests | §12–14 |
| 8 | Redis Streams adapter + dispatcher only | `XADD` after commit, idempotent republish test | §8, §9.1 |
| 9 | Worker claim/finalize + registry + mock webhook | conditional claim, ack ordering, negative tests | §9.2, §10 |
| 10 | Scheduler, retry, crash recovery in separate PRs | fault injection and lease race matrix | §9.3–9.6, §29 |

**Catatan dependency:** urutan di atas adalah backlog awal. Bila `go build ./cmd/...` belum bisa lewat karena entrypoints belum dibuat, tulis skeleton minimal dulu, jangan menandai gate PASS secara palsu. Pengujian payment dan MinIO mengikuti milestone dalam §21, bukan blocker untuk ticket scaffolding.

---

## 44. Session Handoff dan Review Akhir (V4)

Simpan ringkasan singkat sebagai konteks untuk agent berikutnya. Contoh *format*, bukan status implementasi nyata:

```markdown
# AI Worklog — Antriaga
Updated: <UTC timestamp>
Milestone: <1–4>
Task: <satu target>

## Implemented
- <verified behavior>

## Files changed
- <real path>: <purpose>

## Verification evidence
- `go test ./... -count=1`: <PASS/FAIL/NOT RUN>
- `go test -race ./...`: <PASS/FAIL/NOT RUN>
- Integration/Testcontainers: <PASS/FAIL/NOT RUN and reason>

## Decisions and risks
- <ADR or unresolved issue>

## Next smallest task
- <exact behavior + expected tests>
```

**Quality statement:** Skill repository eksternal dapat membantu menulis kode lebih idiomatik, tetapi **tidak membuktikan** Antriaga fault-tolerant. Klaim reliability baru sah setelah fault-injection E2E, concurrent races, log/metrics, dan idempotency diukur dengan hasil nyata. V4 menambahkan proses untuk mencapai bukti tersebut tanpa memperluas scope teknis v3.
---

## 45. Public Markdown Documentation — Untuk User yang Menggunakan Antriaga (v4.1)

**Tujuan:** Pengguna Antriaga tidak semestinya perlu membaca 3.000+ baris system design sebelum dapat memanggil API. Sediakan dokumentasi dua lapis: **public/user docs** yang ringkas dan actionable, dan **maintainer/design docs** untuk arsitektur serta implementasi. Perubahan ini *tidak* memperluas scope MVP atau menjanjikan bahwa runtime sudah dibangun.

### 45.1 Persona dan user journey

| Pembaca | Kebutuhan | Dokumentasi |
|---|---|---|
| Pengembang layanan pemanggil | Mendapat key, submit job, lihat status, tangani error | `docs/getting-started/quickstart.md`, `docs/guides/*` |
| Integrator API/SDK | Request/response, payload version, idempotency, contoh Go | `docs/reference/api.md`, `docs/examples/go-client.md` |
| Operator/deployer | Install, config, rotate key, deploy, monitor, debug | `docs/operations/*`, `docs/reference/configuration.md` |
| Evaluator open source | Mengerti fitur/limitasi tanpa setup kompleks | `README.md`, `docs/README.md`, `docs/faq.md` |
| Kontributor/AI coding agent | Perubahan kontrak API harus menyesuaikan docs publik | `AGENTS.md`, Bagian 36–44, §46–47 |

### 45.2 Struktur wajib dokumentasi Markdown

Buat file **nyata**, bukan hanya link placeholder. Paket pendamping `antriaga-v4-docs-package.zip` menyertakan semuanya.

```text
antriaga/
├── README.md                                  # Entry point publik
├── AGENTS.md                                  # Panduan AI agent / maintainer
├── CONTRIBUTING.md                            # Opsional saat alur kontribusi tersedia
├── SECURITY.md                                # Responsible vulnerability disclosure
├── CHANGELOG.md                               # Rilis dan breaking changes (verified only)
├── SYSTEM_DESIGN.md                            # Blueprint, §§1–47
└── docs/
    ├── README.md                              # Documentation portal / navigation
    ├── getting-started/
    │   ├── installation.md                    # Prerequisites, startup, readiness
    │   └── quickstart.md                      # First job via curl
    ├── guides/
    │   ├── authentication.md                  # Unique clients, X-API-Key, scopes, rotation
    │   ├── submitting-jobs.md                 # POST, idempotency, scheduling, version
    │   ├── job-types.md                       # 4 payloads MVP + constraints
    │   └── tracking-jobs.md                   # Lifecycle, attempts, retry, dead
    ├── reference/
    │   ├── api.md                             # Endpoints, response envelope, MVP vs P1/P2
    │   ├── configuration.md                   # Environment variables & safety
    │   └── errors.md                          # HTTP status, retry policy for API users
    ├── examples/
    │   └── go-client.md                       # Standard library Go integration
    ├── operations/
    │   ├── deployment.md                      # First deploy, scaling, rolling upgrade
    │   ├── monitoring.md                      # Logs and metrics / job correlation
    │   ├── troubleshooting.md                 # Symptom → diagnosis → action
    │   └── security.md                        # Secret/key/transport/PII guidance
    └── faq.md                                 # Common questions and caveats
```

**Tambahan open-source root docs:** `SECURITY.md` dan `CHANGELOG.md` memuat kebijakan dan catatan awal. Panduan `CONTRIBUTING.md` dapat ditambahkan saat alur kontribusi tersedia. Verifikasi saluran pelaporan privat dan versi rilis sebelum publikasi.

**Catatan Swaggo:** Karena v4 menghasilkan `docs/docs.go`, `docs/swagger.json`, `docs/swagger.yaml`, Markdown disimpan dalam subfolder agar output Go package dan doc website dapat hidup berdampingan. **Jangan menghapus seluruh `docs/` saat regenerate Swagger.** Bila kelak generated spec dipindah, perbarui `swag init -o`, import Go `docs`, Swagger UI, CI, dan tautan dokumentasi secara konsisten.

### 45.3 Kontrak pengalaman pengguna (MVP)

Sebelum menganggap dokumentasi "user-ready", pastikan user bisa menyelesaikan urutan berikut dari README tanpa harus membaca source code internal:

1. Memahami scope MVP dan keterbatasan (at-least-once, 4 job types, mock payment only).
2. Menemukan requirements, cara deploy, dan mengecek `/health/ready`.
3. Meminta operator menerbitkan key berdasarkan unique `client_id`, kemudian menggunakannya via `X-API-Key`.
4. Mengirim immediate job melalui cURL dengan `Idempotency-Key` yang stabil.
5. Melihat job status `scheduled/queued/processing/retrying/completed/dead` dan attempts.
6. Memahami retry worker versus retry HTTP client, termasuk `429`, `503`, timeout, dan `409`.
7. Mengetahui kapan harus menghubungi operator, informasi aman apa yang dibagikan, dan informasi rahasia yang tidak boleh dibagikan.

Dokumen publik harus menyebut bahwa `202 Accepted` **bukan** sukses side effect; API key tidak bisa dihasilkan sendiri dari public HTTP API; `client_id` tidak boleh dikirim untuk mengambil ownership client lain; dan Redis **bukan** source of truth.

### 45.4 Contoh API key dan onboarding client

Gunakan **placeholder**, bukan credential asli:

```bash
# Admin-side only, CLI planned (§27):
antriaga-admin clients create --code notification-service --name "Notification Service"
antriaga-admin keys create --client notification-service --label primary \
  --scopes jobs:create,jobs:read --expires-in 90d

# Client-side, dengan secret injected dari secret manager:
export ANTRIAGA_URL=http://localhost:8080
# ANTRIAGA_API_KEY telah diset oleh secret manager
curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "${ANTRIAGA_URL}/api/v1/jobs?status=queued&page=1&limit=20"
```

Admin CLI di atas adalah **rancangan command contract**, tidak otomatis tersedia sebelum `cmd/admin` diimplementasikan. Seluruh contoh dan README wajib menunjukkan status implementasi dengan jelas; jangan membuat user mengira deployment/tag atau image sudah dirilis.

### 45.5 Anti-pattern dokumentasi publik

- Jangan menyalin source-level pseudocode `GORM`/Redis/lease ke Quickstart jika pemakai hanya memerlukan HTTP contract.
- Jangan menjanjikan `exactly-once`, priority queue, public key issuance, arbitrary webhooks, custom job scripts, manual requeue atau event timeline pada MVP.
- Jangan menyarankan mengedit status MySQL secara manual, `XACK`/`XTRIM` paksa, atau mem-post API keys di tiket bantuan.
- Jangan memberikan contoh penggunaan mock payment seolah-olah payment gateway sungguhan.
- Jangan menyembunyikan status **belum diimplementasikan** atau menyatakan sample response telah diverifikasi jika belum diuji.

---

## 46. Dokumentasi sebagai Release Gate (v4.1)

### 46.1 Mapping perubahan kode → dokumentasi

| Perubahan kode | File dokumentasi yang wajib diperbarui | Bukti minimal |
|---|---|---|
| Endpoint, DTO, response/error | `docs/reference/api.md`, `docs/reference/errors.md`, Swaggo annotation | Swagger regenerated dan integration/API contract test |
| API key/authorization/roles | `docs/guides/authentication.md`, `docs/operations/security.md` | Invalid/revoked/expired/scope/ownership tests |
| Job payload/type/version | `docs/guides/job-types.md`, `docs/guides/submitting-jobs.md`, contoh | Version compatibility tests, payload schema tests |
| Retry, lease, recovery | `docs/guides/tracking-jobs.md`, `docs/operations/troubleshooting.md` | Fault injection / worker race tests |
| Env/config/Compose | `docs/reference/configuration.md`, `docs/operations/deployment.md` | Clean-install smoke test, readiness checks |
| Observability/log events | `docs/operations/monitoring.md`, `docs/operations/troubleshooting.md` | Log and metrics assertions |
| Fitur baru / limitasi berubah | `README.md`, `docs/README.md`, `docs/faq.md` | Scope/feature maturity audit |

### 46.2 Acceptance checklist dokumentasi

- [ ] Semua Markdown link relatif resolve dan tidak ada path placeholder yang diklaim sudah ada.
- [ ] Status fitur jelas: **MVP**, **P1**, atau **P2**; tidak mengklaim implementasi/release yang belum ada.
- [ ] cURL quickstart hanya memakai endpoint dan status MVP, memiliki `X-API-Key`, dan tidak mengekspos secret.
- [ ] Contoh `Idempotency-Key` memperlihatkan cara retry submit yang aman, bukan exactly-once side effect.
- [ ] Contoh payload 4 job type sinkron dengan DTO dan validation nyata.
- [ ] `401`, `403`, `404` cross-client, `409`, `422`, `429`, `503` didokumentasikan dan diuji.
- [ ] Swaggo docs dan Markdown reference tidak saling bertentangan.
- [ ] Instruksi instalasi mengikuti binary/Compose/migrations yang benar-benar ada.
- [ ] Ada panduan troubleshooting, keamanan kredensial, serta proses pelaporan error dengan `request_id`/`job_id` saja.
- [ ] Jika contoh belum dijalankan, dinyatakan **example / unverified**. Jangan tandai pass tanpa hasil.

### 46.3 Quality gates saat repository siap

```bash
# Contoh target pemeriksaan; buat scripts/CI nyata sebelum mengklaim telah dijalankan.
# 1) Pemeriksaan link Markdown internal.
# 2) Lint Markdown (mis. markdownlint-cli2), versi dipin.
# 3) Regenerate Swaggo, pastikan git diff tidak memuat kontrak yang usang.
# 4) Jalankan smoke test quickstart terhadap Docker Compose di test environment.
# 5) Jalankan security negative tests dan permission isolation.
```

Documentation check harus menjadi bagian PR template/CI jika tersedia. Public docs ditulis ramah pengguna (Bahasa Indonesia + istilah teknis yang konsisten), contoh perintah ditempatkan di code fence dengan bahasa yang tepat, dan setiap fitur harus dapat ditemukan dari `README.md` atau `docs/README.md`.

---

## 47. Vibe Coding — Agent Rules untuk Public Documentation (v4.1)

Selain aturan arsitektur dalam `AGENTS.md`, AI coding agent mengikuti kebijakan berikut ketika menyentuh API, CLI, konfigurasi, atau behavior runtime:

1. **Read → Compare:** baca dokumen publik terdampak **dan** aktual DTO/router/handler/CLI sebelum membuat contoh. Jangan mengarang endpoint, flag, sample response, atau image tag.
2. **Diff minimal:** jika fitur belum selesai, buat draft dokumentasi dengan label "planned/unverified" dan jangan menandainya released.
3. **Sync changes:** setiap perubahan contract juga mengubah API reference, guide terkait, sample cURL/Go, dan Swaggo annotations (lalu regenerate).
4. **Security first:** API key hanya placeholder; tidak boleh tercetak plaintext atau muncul di test snapshot/log. Cross-client owner isolation dan scopes harus dijelaskan.
5. **Test documentation:** parse Markdown links, compile Go example bila memungkinkan, jalankan cURL smoke test terhadap test deployment bila tersedia, dan catat yang tidak dijalankan.
6. **Report evidence:** laporkan file Markdown yang diperbarui, hasil link/syntax checks, hasil API contract tests, serta caveat "belum diuji" secara transparan.

**Prompt yang bisa dipakai pada AI coding agent:**

```text
Anda bekerja pada Antriaga. Baca AGENTS.md dan desain §12–14, §27,
§30, §45–47. Tugas: sesuaikan dokumentasi publik setelah perubahan
[endpoint/payload/auth/config] tertentu. Periksa kode nyata terlebih dulu.
Update README, docs/ terkait, Swaggo annotations, dan contoh usage.
Jangan mengklaim command atau endpoint tersedia jika belum diimplementasi.
Jaga API key redaction dan cross-client isolation. Jalankan link check,
unit/API tests, dan docs smoke tests yang tersedia; laporkan PASS/FAIL/NOT RUN
beserta command aktual dan alasan.
```

**Hasil v4.1:** Dokumen ini tetap menjadi technical blueprint. `README.md` dan `docs/` di paket pendamping adalah **artefak dokumentasi user yang siap di-review dan di-integrasikan ke repository**, bukan bukti aplikasi sudah dijalankan.
