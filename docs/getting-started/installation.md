# Instalasi dan persyaratan

> **Status:** panduan instalasi **yang ditargetkan** setelah binary dan Docker Compose dibuat. Perintah dan nama service mengacu pada blueprint v4; berkas compose serta image perlu tersedia terlebih dahulu dari repository implementasi.

## Kebutuhan

- Docker Engine dan Docker Compose v2, atau runtime setara
- Docker host dengan ruang untuk volume MySQL, Redis, dan MinIO
- Akses shell untuk `make` dan admin CLI. Compose menyertakan migrator dalam service `migrate`
- `curl`; opsional `jq` untuk membaca JSON
- **Untuk deployment non-lokal:** reverse proxy TLS, secret manager, backup MySQL, dan pembatasan akses network

## Komponen

| Service | Peran |
|---|---|
| `api` | Fiber HTTP API, health endpoints, Swagger (opsional) |
| `mysql` | Data durable jobs, attempts, clients, keys, outbox |
| `migrate` | Menjalankan migrasi satu kali setelah MySQL sehat, lalu keluar |
| `redis` | Stream event dan consumer groups |
| `dispatcher` | Publikasi transactional outbox ke Redis |
| `worker` | Eksekusi handler untuk job types |
| `scheduler` | Mengaktifkan scheduled/retrying jobs |
| `reaper` | Memulihkan pending jobs / stale lease |
| `minio`, `mock-mailer`, `mock-webhook`, `mock-payment` | Storage dan provider demo |

## Menjalankan lingkungan lokal (setelah implementasi tersedia)

```bash
cp .env.example .env
# Isi variabel yang diwajibkan di .env. Jangan commit .env.
# Set AUTH_KEY_PEPPER_B64 dari sumber rahasia yang aman, bukan nilai contoh permanen.
make up
curl --fail http://localhost:8080/health/live
curl --fail http://localhost:8080/health/ready
```

`make up` menunggu MySQL sehat dan service `migrate` keluar sukses sebelum aplikasi berjalan. API dan scheduler dapat mulai meski Redis belum siap. Worker, dispatcher, dan reaper menunggu Redis. Set `MYSQL_MIGRATION_URL` untuk service `migrate` di `.env`. Jika startup tertahan, periksa `docker compose -f deployments/docker-compose.yml logs migrate`. Target ini masih direncanakan dalam blueprint.

Contoh pembuatan pepper (untuk local development saja):

```bash
openssl rand -base64 32
```

Alternatif PowerShell tanpa `openssl`:

```powershell
[Convert]::ToBase64String([Security.Cryptography.RandomNumberGenerator]::GetBytes(32))
```

Simpan hasilnya ke secret manager / env runtime dengan akses terbatas. **Jangan** memasukkan secret ke dokumentasi atau issue tracker.

## Menyiapkan kredensial

Hanya operator berwenang dapat membuat `client` dan `api_key` lewat **admin CLI** yang direncanakan. Lanjutkan ke [Authentication](../guides/authentication.md), kemudian [Quickstart](quickstart.md).

## Verifikasi awal

- `/health/live` mengukur kesehatan proses; **tidak** menjamin dependency sehat
- `/health/ready` menandakan dependency yang diperlukan untuk API siap; Redis dapat dilaporkan degraded bila MySQL/outbox masih dapat menerima job
- UI Swagger (jika aktif) tersedia pada `http://localhost:8080/swagger/index.html`; nonaktifkan atau lindungi dalam deployment publik
- API menerima `X-API-Key` untuk `/api/v1/*`; permintaan tanpa key harus menghasilkan `401`

## Masalah umum

Lihat [Troubleshooting](../operations/troubleshooting.md) dan [Configuration](../reference/configuration.md). Jika nama target Makefile atau image berbeda pada rilis final, ikuti README yang dibundel bersama rilis tersebut.

[Kembali ke indeks dokumentasi](../README.md)
