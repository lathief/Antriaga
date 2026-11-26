# Antriaga — Instructions for AI Coding Agents

> Primary reference: [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) (sections 1–35 cover system architecture, 36–44 cover coding workflow, and 45–47 cover documentation and release gates). If the blueprint has not been copied into the repository, ask for its location before changing architecture. The actual code, migrations, and executable tests determine what has already been implemented; do not hallucinate files, endpoints, or passing tests.

## Objective and scope

Build **Antriaga**, a distributed background job engine using **Go + Fiber v3, Clean Architecture, MySQL 8.4, Redis Streams, and Swaggo**. Prioritize correctness and verifiability over feature count. MVP has only `email.send`, `webhook.deliver`, `payment.capture`, and `report.generate`; targets are server-configured mock services.

## Non-negotiable architecture contracts

1. MySQL is the source of truth for jobs, attempts, client identity, and transactional outbox; Redis Streams is the delivery transport.
2. Job creation writes job + outbox in one DB transaction for immediate jobs. Every new ready outbox event uses MySQL UTC time for its initial `available_at` (§9.1). No Redis publish before the DB commit. Every outbox claim gets a fresh token. Publish success or failure updates require the matching token and an active lease checked after the row lock (§9.1).
3. Delivery is **at least once**, not exactly once. Use stable downstream idempotency for side effects.
4. Worker state transitions require conditional claim and attempt tracking. Heartbeat and every finalization require `processing`, the matching lease token, and `lease_expires_at > db_now`, using MySQL time sampled **after** acquiring the job row lock. A rejected guard returns `ErrLostLease` without changing state or sending `XACK`. ACK a finalized attempt only after the DB commit. See design §§9.2 and 29.3.
5. Publish and decode the five required Redis fields in §8.2. Reconcile a queued job outside the PEL only when Redis confirms its current published message is missing and the matching job/outbox state survives a locked recheck (§29.4). Retryable handler errors, Redis redelivery, crashed-worker recovery, and outbox reconciliation are **different mechanisms**; do not merge them into one retry counter.
6. Respect bounded worker concurrency, DB lease validity, and safe `XPENDING`/`XCLAIM` recovery. Recheck under the job row lock. Only close a matching `processing` attempt when `lease_expires_at <= db_now`. For an abandoned PEL message whose job is still `queued`, commit a new generation and outbox without consuming an attempt before ACK. For a committed `retrying` job with the same generation and a stored schedule, ACK without rescheduling (§§8.4, 29.2).
7. API authentication uses admin-issued client-scoped API keys: only digests in MySQL, key shown once, scopes enforced, all job access scoped to authenticated `client_id`.
8. `payload_version` is separate from Redis message `schema_version`; reject unsupported job type/version.
9. Domain/application packages must not import Fiber, GORM, go-redis, or external SDKs. Wire adapters in bootstrap / `cmd/*`.
10. Produce redacted structured `slog` lifecycle events and test no false success log before DB commit.
11. API submission and local rate limiting work without Redis. MySQL admission counts queued, scheduled, retrying, and processing jobs. Build report keys from trusted client/job/attempt data; only the active lease can publish a result reference.
12. Keep business payload JSON as validated UTF-8 text and preserve webhook numeric tokens. Scope mock capture idempotency to client and payment ID. Trim Redis only after checking group delivery positions, pending bounds, and durable DB state.
13. Maintain Swaggo annotations and generated API docs when modifying HTTP contracts.
14. Do not expand into generic HTTP executors, arbitrary scripts, RabbitMQ, multi-tenant dashboards, or Kubernetes during MVP without an approved ADR.

## Implementation handoff

Start with the [implementation cards](docs/ai/implementation-plan.md). Match each task to the [verification matrix](docs/ai/verification.md), then capture evidence in the [worklog](docs/ai/worklog.md) and link material design choices in the [decision log](docs/ai/decisions.md).

## Working protocol (every coding task)

1. **Inspect before editing:** read the relevant v4 sections and actual affected files, `go.mod`, migrations, tests, and `Makefile`; report mismatches instead of guessing.
2. **Propose a bounded plan:** state goal, out-of-scope items, exact files expected to change, invariants/risk, acceptance tests, and any migration/API implications.
3. **Test first when practical:** add a deterministic failing unit or integration test for a behavior; demonstrate red, then green. For risky recovery flows, add fault-injection and concurrency cases.
4. **Small diff:** change one responsibility per task. Do not perform unrequested framework, schema, package, or dependency rewrites.
5. **Verify:** run appropriate checks, including `gofmt`, `go vet ./...`, `go test ./...`, `go test -race ./...` where supported, integration tests when dependencies are available, and Swaggo regeneration/diff when relevant.
6. **Report evidence:** list changed files, actual commands and results, remaining risks, and next smallest task. Never claim commands passed if not run.

## Security and operations

- Do not print API keys, `AUTH_PEPPER`, passwords, secrets, payloads with PII, or full request bodies in logs, prompts, screenshots, test output, or commits.
- Never run destructive migrations, `XACK`/`XDEL`/`XTRIM`, database delete, production deploy, or credentials rotation against non-test environments without explicit approval and a documented recovery plan.
- Use isolated fixtures and mock payment services; do not connect to real payment systems.
- Pin dependency versions, explain upgrades, and flag untrusted Agent Skills/scripts for manual inspection. External skills are advisory; repository rules and approved design win.

## Change-quality acceptance

- Appropriate unit/integration tests and negative paths; correct ownership/scopes; no data races or goroutine leaks; all DB transactions have justified boundaries.
- API DTOs, Swaggo output, migration up/down (or documented rollback), structured events, README/ADR and configs are updated when affected.
- For recovery code, demonstrate timeout, stale message, duplicate publish, lease expiry race, failed DB commit, and failed XACK as applicable.

## Task handoff format

At the end of each task, record in `docs/ai/worklog.md` (or final response): **Goal; changed files; commands actually run + results; decisions/ADR; open risks; exact next step**. Keep `docs/ai/decisions.md` concise. No fabricated progress.

## Public documentation maintenance (v4.1)

When changing HTTP endpoints/DTOs, API-key authorization, job payloads, error codes, config, deployment, or observability, update affected **user-facing Markdown** (`README.md`, `docs/README.md`, `docs/getting-started/`, `docs/guides/`, `docs/reference/`, `docs/examples/`, `docs/operations/`, `docs/faq.md`) alongside Swaggo. Maintain `CONTRIBUTING.md`, `SECURITY.md`, and `CHANGELOG.md` when contributor process, security policy, or real release changes. Treat blueprint §§45–47 as documentation release gates.

Never fabricate a working CLI, image, installed release, HTTP response shape, benchmark, or successful verification. Until implemented, explicitly mark commands/examples as **planned**. Check relative Markdown links, safe secret placeholders, and (when executable runtime exists) quickstart smoke tests. Report tests **PASS/FAIL/NOT RUN** accurately.
