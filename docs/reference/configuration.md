# Referensi konfigurasi

> Variabel berikut berasal dari `.env.example` pada design v4. **Default ini adalah starting values untuk diuji**, bukan benchmark atau konfigurasi production siap-pakai. Beberapa service memerlukan subset variabelnya masing-masing.

| Area | Variable | Sample / purpose |
|---|---|---|
| Runtime | `APP_ENV` | `local`, pisahkan environment |
| HTTP | `HTTP_ADDR` | `:8080` |
| Swagger | `ENABLE_SWAGGER` | `true` untuk local; batasi di production |
| Logging | `LOG_LEVEL`, `LOG_FORMAT` | `debug` / `json` lokal, `info` / `json` produksi |
| Database | `MYSQL_DSN` | Go MySQL DSN untuk proses aplikasi |
| Migrasi | `MYSQL_MIGRATION_URL` | URL `golang-migrate` menuju DB MySQL yang sama. Gunakan dari service `migrate` |
| DB pool | `MYSQL_MAX_OPEN_CONNS`, `MYSQL_MAX_IDLE_CONNS` | 30 / 10 sebagai awal |
| Redis | `REDIS_ADDR`, `REDIS_PASSWORD`, `REDIS_DB` | Koneksi broker |
| Redis | `REDIS_CONSUMER_GROUP` | `antriaga-workers` |
| Redis | `REDIS_STREAM_PREFIX` | `antriaga:queue:` |
| Worker | `WORKER_CONCURRENCY` | 10 slot per instance |
| Worker | `REDIS_CONSUME_BATCH_MAX` | 10, tidak melebihi slot tersedia |
| Worker | `PER_QUEUE_CONCURRENCY_<QUEUE>` | Opsional, dari [blueprint §28.1](../../SYSTEM_DESIGN.md). Membatasi slot per queue (mis. `PER_QUEUE_CONCURRENCY_EMAIL`) agar satu queue tidak memonopoli worker. Nama dan nilai bawaan ditetapkan saat worker diimplementasikan |
| Observability | `METRICS_ADDR` | **Direncanakan.** Alamat listener `/metrics` dan liveness untuk proses non-API (worker, dispatcher, scheduler, reaper), mis. `:9100`. Bind ke jaringan internal saja |
| Retry | `JOB_DEFAULT_MAX_ATTEMPTS` | 5 |
| Retry | `RETRY_BASE_DELAY`, `RETRY_MAX_DELAY` | `5s`, `15m` |
| Retry | `RETRY_JITTER_PERCENT` | 20 |
| Lease | `JOB_LEASE_TTL`, `JOB_HEARTBEAT_INTERVAL` | `90s`, `30s` |
| Timeout | `JOB_EXECUTION_TIMEOUT` | `5m` |
| Scheduling | `SCHEDULER_INTERVAL`, `DISPATCHER_INTERVAL`, `REAPER_INTERVAL` | `1s`, `500ms`, `15s` |
| Outbox | `OUTBOX_LEASE_TTL`, `OUTBOX_PUBLISH_TIMEOUT` | `30s`, `5s`. Timeout publish harus positif dan lebih pendek dari TTL |
| Recovery | `REAPER_MIN_IDLE` | `120s` initial |
| Admission | `API_RATE_LIMIT_PER_CLIENT_RPS`, `API_RATE_LIMIT_MAX_CLIENTS` | 20 RPS per API replica dan paling banyak 10000 bucket lokal |
| Admission | `API_MAX_OUTSTANDING_PER_CLIENT` | 1000 job aktif per client, termasuk scheduled dan retrying |
| Admission | `API_OUTSTANDING_HIGH_WATERMARK`, `API_OUTBOX_HIGH_WATERMARK` | 20000 job aktif global atau 10000 outbox pending/publishing sebelum `503` |
| Payload | `JOB_MAX_PAYLOAD_BYTES` | 262144 (256 KiB) |
| Storage | `MINIO_ENDPOINT`, `MINIO_BUCKET` | Bucket privat untuk object report dengan key buatan server |
| Mock | `MOCK_MAILER_URL`, `MOCK_WEBHOOK_URL`, `MOCK_PAYMENT_URL` | Configured server-side target |
| Auth | `AUTH_KEY_PEPPER_B64` | **Wajib dan rahasia**, base64 32+ random bytes |
| Auth | `API_KEY_ENVIRONMENT` | `test` atau `live` sesuai environment |
| Auth | `API_KEY_DEFAULT_EXPIRY_DAYS` | 90 initial |
| Auth | `AUTH_MAX_API_KEY_LENGTH` | 256 (max header length policy) |

## Catatan koneksi MySQL

`MYSQL_MAX_OPEN_CONNS` berlaku **per proses**. Total koneksi semua proses harus di bawah `max_connections` MySQL. Hitungan dan contohnya ada di [deployment](../operations/deployment.md#kapasitas-koneksi-mysql).

## Catatan limiter

`API_RATE_LIMIT_PER_CLIENT_RPS` berlaku **per replica API**. Dengan N replica, laju efektif per client dapat mencapai N kali nilai itu. Kuota job aktif (`API_MAX_OUTSTANDING_PER_CLIENT`) dihitung di MySQL dan tidak bergantung pada jumlah replica.

## Konfigurasi aman

- Jangan menaruh `AUTH_KEY_PEPPER_B64`, password DB/Redis, atau MinIO secret di Git.
- Pepper salah/hilang membuat key lama tidak dapat diverifikasi; rencanakan proses reissue/rotation sebelum rotasi pepper.
- Gunakan HTTPS, restricted MySQL/Redis network, dan internal-only metrics/Swagger.
- Redis AOF/persistence dan backup MySQL perlu strategi operasional; **Redis tidak menggantikan MySQL**.
- Jangan memakai parameter dokumentasi sebagai jaminan throughput. Lakukan load test nyata sebelum mengubah slot worker dan pool DB.

## Hubungan antar timeout

`JOB_HEARTBEAT_INTERVAL` harus lebih kecil dari `JOB_LEASE_TTL`. Setelah MySQL menyatakan lease kedaluwarsa, worker kehilangan hak memperpanjang lease dan menyimpan hasil. `REAPER_MIN_IDLE` menyaring pesan Redis yang perlu diperiksa, lalu reaper menggunakan waktu dan state MySQL untuk memutuskan pemulihan. Lihat [desain worker dan recovery](../../SYSTEM_DESIGN.md#92-claim-dan-eksekusi-job).

## Konfigurasi staging/production

Minimal siapkan env berbeda, secret manager, reverse proxy TLS, DB backups dan migration playbook. Gunakan `LOG_FORMAT=json`; `LOG_LEVEL=info` biasanya lebih aman untuk volume produksi; larang body payload/credential dalam logs. Review [Deployment](../operations/deployment.md) dan [Security](../operations/security.md).

[Kembali ke indeks dokumentasi](../README.md)
