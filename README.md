# Antriaga

> Distributed Background Job Engine — *Every Job Matters. Every Failure Has a Recovery Path.*

Antriaga dirancang sebagai layanan background-job terdistribusi berbasis **Go/Fiber, MySQL, dan Redis Streams**, dengan persistent job state, retry terukur, scheduling, worker recovery, dan client-based API key.

> **Status proyek:** paket ini berisi **dokumentasi kontrak dan contoh penggunaan berdasarkan system design v4**, **bukan klaim bahwa aplikasi/CLI sudah tersedia atau semua perintah telah berhasil dijalankan**. Periksa release/README implementasi sebelum mengikuti prosedur instalasi. Dokumentasi API final harus diselaraskan dengan generated Swaggo dari kode sebenarnya.

## Pilih jalur Anda

| Anda ingin... | Mulai di |
|---|---|
| Memahami apa yang didukung | [Dokumentasi pengguna](docs/README.md) |
| Menginstal atau menjalankan lokal | [Installation](docs/getting-started/installation.md) |
| Mengirim job pertama | [Quickstart](docs/getting-started/quickstart.md) |
| Memperoleh dan memakai API key | [Authentication](docs/guides/authentication.md) |
| Mengetahui payload 4 job types | [Job Types](docs/guides/job-types.md) |
| Integrasi melalui API/Go | [API Reference](docs/reference/api.md) dan [Go example](docs/examples/go-client.md) |
| Menjalankan dan memonitor sistem | [Deployment](docs/operations/deployment.md), [Monitoring](docs/operations/monitoring.md) |
| Menyelesaikan kesalahan | [Troubleshooting](docs/operations/troubleshooting.md) |
| Mempelajari internals / berkontribusi | [System Design v4](SYSTEM_DESIGN.md) dan [AGENTS.md](AGENTS.md) |
| Memulai implementasi | [Paket kerja agent](docs/ai/implementation-plan.md) |
| Menilai pekerjaan setelah MVP | [Backlog setelah MVP](docs/post-mvp.md) |

## Cakupan MVP

- `email.send`, `webhook.deliver`, `payment.capture`, `report.generate`
- `POST /api/v1/jobs`, `GET /api/v1/jobs`, `GET /api/v1/jobs/:id`, `GET /api/v1/jobs/:id/attempts`
- `X-API-Key` per client, scopes `jobs:create` dan `jobs:read`
- MySQL sebagai source of truth; Redis Streams untuk delivery; **at-least-once**, bukan exactly-once
- `payment.capture` **hanya ke mock payment gateway** pada MVP, bukan pembayaran sungguhan

**Belum termasuk MVP:** public API key issuance, generic URL/script executor, manual requeue (P1), durable event timeline (P2), web dashboard, cancel API, dan priority scheduling nyata.

## Open-source project files

- [SECURITY.md](SECURITY.md): pelaporan keamanan privat (kontak perlu dikonfigurasi sebelum rilis)
- [CHANGELOG.md](CHANGELOG.md): perubahan dokumentasi dan rilis yang terverifikasi

## Dokumentasi dan kontribusi

Baca [konvensi dokumentasi](docs/README.md#status-dan-konvensi) sebelum mengadopsi contoh. Jika ada ketidaksesuaian, kontrak hasil implementasi dan generated Swagger yang diuji memiliki prioritas untuk penggunaan rilis nyata; perbarui dokumentasi ketika mengubah kode.
