# Pemantauan, log, dan peringatan

Antriaga dirancang mencetak JSON structured logs dengan `log/slog`, sehingga `job_id` bisa ditelusuri lintas API, dispatcher, scheduler, worker, dan reaper. Logs membantu diagnosis; **MySQL tetap source of truth** untuk job status dan attempts.

## Job-level logging

Field utama yang direkomendasikan: `time`, `level`, `event`, `component`, `request_id`, `client_id` (aman), `job_id`, `attempt_number`, `worker_id`, `error_code`, `retry_delay_ms`, `next_run_at`, `duration_ms`.

**Contoh ilustratif:**

```json
{
  "level": "WARN",
  "event": "job.retry_scheduled",
  "component": "worker",
  "job_id": "019a9e11-330b-7e1e-bb8a-119511778801",
  "attempt_number": 2,
  "error_code": "UPSTREAM_UNAVAILABLE",
  "retry_delay_ms": 10000
}
```

Jangan menulis `X-API-Key`, authorization headers, HMAC digest, pepper, payload sensitif, atau full external response ke logs.

## Follow one job di Docker Compose (local demo)

```bash
docker compose -f deployments/docker-compose.yml logs --no-log-prefix \
  api dispatcher worker scheduler reaper \
  | jq -c 'select(.job_id == "<JOB_ID>") | {time,component,event,job_id,attempt_number,error_code}'
```

Perintah mengasumsikan `jq` tersedia dan semua service menulis JSONL tanpa non-JSON; bila tidak, filter per-service. Pada production gunakan log backend terpusat.

## Metrics yang direkomendasikan

| Metric | Makna |
|---|---|
| `antriaga_jobs_created_total` | Job diterima |
| `antriaga_jobs_completed_total`, `antriaga_jobs_dead_total` | Outcome pekerjaan |
| `antriaga_jobs_retried_total` | Retry handler |
| `antriaga_queue_depth` | Antrean per queue |
| `antriaga_oldest_queued_job_seconds` | Job queued paling lama |
| `antriaga_outbox_oldest_pending_seconds` | Umur pending outbox |
| `antriaga_worker_inflight` | Job sedang dieksekusi |
| `antriaga_worker_lease_lost_total` | Worker kehilangan lease |
| `antriaga_redis_ack_errors_total` | Masalah ACK Redis |
| `antriaga_api_auth_failed_total` | Autentikasi gagal agregat |

Metric name adalah bagian dari **desain** yang perlu diuji setelah implementasi. Hindari `job_id`, `client_id` atau API key sebagai metric label untuk menghindari high cardinality/leakage.

## Alert baseline (tune setelah load test)

- Pending outbox berusia lebih dari 60 detik dan membesar.
- Queue wait tertua lebih dari 10 menit.
- Tidak ada worker untuk queue yang berisi job.
- Banyak dead jobs, Redis publish/ACK error, atau lease lost.
- Readiness MySQL atau Redis kritis gagal.

## Lihat job history

Pemakai dapat memakai `GET /api/v1/jobs/:id` dan `GET /api/v1/jobs/:id/attempts`. Endpoint durable event timeline `/events` **baru P2**; gunakan logs per job sebagai sumber investigasi operasional sementara, bukan audit durable.

Lihat [Tracking Jobs](../guides/tracking-jobs.md) dan [Troubleshooting](troubleshooting.md).

[Kembali ke indeks dokumentasi](../README.md)
