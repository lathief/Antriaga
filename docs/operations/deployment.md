# Deployment dan pembaruan versi

> **Status:** runbook rancangan, bukan bukti deployment berhasil. Nama binary/Compose/Makefile mengikuti system design v4 dan harus disesuaikan dengan repository implementasi saat rilis.

## Deployment minimum

- Pisahkan proses `api`, `dispatcher`, `worker`, `scheduler`, `reaper` supaya bisa direstart dan diskalakan terpisah.
- MySQL 8.4 dengan persistent volume, migration versioned, serta backup/recovery teruji.
- Redis 7+ dengan persistence (mis. AOF) serta network internal; Redis adalah transport dan **bukan source of truth**.
- MinIO untuk output report dan mock services untuk demo MVP.
- Reverse proxy TLS, secret manager, internal metrics, access logging teredaksi, healthchecks.

## Checklist sebelum first deploy

1. Pin image/tag dependency dan version migration; review secrets di secret manager.
2. Tunggu MySQL sehat. Jalankan migration `clients` → `api_keys` → `jobs` → `job_attempts` → `outbox_events`. Pada Compose, service `migrate` harus keluar sukses sebelum aplikasi dimulai.
3. Pastikan `AUTH_KEY_PEPPER_B64` valid dan konsisten antara API dan admin CLI di environment tersebut.
4. Jalankan API dan scheduler setelah migrasi selesai. Periksa readiness MySQL dan outbox untuk API.
5. Setelah Redis siap, jalankan dispatcher, reaper, dan worker. Pastikan consumer group di-bootstrap secara idempoten dan periksa readiness tiap proses. API memakai MySQL untuk readiness dan tetap menjalankan limiter lokal saat Redis putus. Job baru tetap tunduk pada kuota client serta ambang job aktif dan outbox.
6. Buat client dan key dari operator/admin CLI dengan scope paling kecil.
7. Jalankan quickstart smoke test + test isolasi antar client sebelum mengekspos API.
8. Pasang monitoring queue/outbox/dead/lease/ACK error dan atur alert.

## Scale workers

```bash
# Perintah rencana untuk Docker Compose berbasis service 'worker':
docker compose -f deployments/docker-compose.yml up -d --scale worker=3
```

Worker concurrency perlu disesuaikan dengan pool DB dan kapasitas upstream. Jangan over-prefetch Redis Streams; jumlah pesan yang diambil harus tidak melebihi slot worker tersedia.

## Rolling upgrade dan payload version

1. Deploy worker baru yang mendukung versi payload lama dan baru.
2. Pastikan delayed jobs lama masih dapat diproses.
3. Setelah ready, API menerima format versi baru.
4. Tunggu backlog lama selesai atau migrasikan dengan proses eksplisit yang diuji.
5. Baru hentikan versi handler lama setelah compatibility window dan rollback plan ada.

Payload version dan Redis transport schema version adalah dua hal berbeda. **Jangan** ubah langsung job payload yang sudah diterima.

## Database migrations dan rollback

Jangan mengandalkan `AutoMigrate()` di production. Gunakan migration yang direview dan uji migrasi dua arah ketika memungkinkan. Operasi destruktif memerlukan backup dan migration plan tersendiri; beberapa schema changes tidak bisa dibalik otomatis tanpa kehilangan data.

## Shutdown dan recovery

Worker berhenti mengambil job baru, drain in-flight dalam grace period (awal 30 detik). Jika belum selesai, lease/PEL recovery melakukan pemulihan. Side effect bisa terjadi ulang; downstream idempotency wajib.

## Backup dan data retention

Kebijakan retention/cleanup automated adalah **P1**, tetapi backup MySQL dan pembatasan disk perlu dilakukan sejak deployment pertama. Pada Redis 7, trim hanya prefix dengan bukti settled dari MySQL. Periksa posisi delivery dan batas PEL setiap group. Saat PEL kosong, pesan di atas `last-delivered-id` masih perlu dikirim. Tunda trim bila group dibentuk ulang atau bukti belum lengkap.

Referensi: [Configuration](../reference/configuration.md), [Monitoring](monitoring.md), [Troubleshooting](troubleshooting.md), [Security](security.md).

[Kembali ke indeks dokumentasi](../README.md)
