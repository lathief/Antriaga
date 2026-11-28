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

### Grace period shutdown

> **Catatan rencana, belum diverifikasi.**

`WORKER_SHUTDOWN_GRACE` (30 detik) hanya berguna jika platform memberi waktu lebih lama dari itu sebelum SIGKILL. Bawaan Docker Compose `stop_grace_period` adalah 10 detik, sehingga worker dimatikan paksa sebelum drain selesai. Bawaan Kubernetes `terminationGracePeriodSeconds` adalah 30 detik, sama dengan grace aplikasi, jadi tidak ada sisa waktu untuk keluar bersih.

Set grace platform sedikit di atas grace aplikasi (mis. 40 detik) untuk service worker. Pemulihan lease tetap menutup job yang terputus, tetapi shutdown bersih mengurangi lease yang harus kedaluwarsa dulu.

## Kapasitas koneksi MySQL

> **Catatan rencana, belum diverifikasi.**

Total koneksi yang mungkin dibuka sama dengan jumlah proses dikali `MYSQL_MAX_OPEN_CONNS`. Nilai itu harus berada di bawah `max_connections` MySQL (bawaan 151), dengan sisa untuk admin dan migrasi. Contoh: 5 worker dengan pool 30 sudah 150 koneksi, terlalu dekat dengan batas bawaan.

Setel pool per binary, bukan satu nilai untuk semua proses, lalu hitung totalnya setiap menambah replica. Worker dengan `WORKER_CONCURRENCY=10` yang memakai transaksi pendek biasanya tidak butuh pool sebesar API. Tentukan angkanya dari pengukuran.

## Ketersediaan dan titik tunggal

> **Catatan rencana, belum diverifikasi.**

MySQL adalah sumber kebenaran sekaligus titik tunggal kegagalan. Ketersediaan sistem sama dengan ketersediaan MySQL, karena API, scheduler, dispatcher, worker, dan reaper semuanya bergantung padanya. Redis yang mati hanya menunda pengiriman, sedangkan MySQL yang mati menghentikan penerimaan dan eksekusi job.

Target RPO dan RTO **belum ditetapkan**. Tentukan setelah backup dan latihan restore dijalankan dan waktunya terukur (lihat [backlog](../post-mvp.md) untuk latihan restore).

## Backup dan data retention

Kebijakan retention/cleanup automated adalah **P1**, tetapi backup MySQL dan pembatasan disk perlu dilakukan sejak deployment pertama. Pada Redis 7, trim hanya prefix dengan bukti settled dari MySQL. Periksa posisi delivery dan batas PEL setiap group. Saat PEL kosong, pesan di atas `last-delivered-id` masih perlu dikirim. Tunda trim bila group dibentuk ulang atau bukti belum lengkap.

Referensi: [Configuration](../reference/configuration.md), [Monitoring](monitoring.md), [Troubleshooting](troubleshooting.md), [Security](security.md).

[Kembali ke indeks dokumentasi](../README.md)
