# Changelog

Catatan ini memuat perubahan dokumen dan kontrak blueprint yang sudah ditulis. Entri rilis runtime dicatat setelah implementasi dan verifikasi selesai.

## Unreleased — Documentation Blueprint v4.1

### Added (documentation only)

- Dokumentasi Markdown untuk installation, quickstart, authentication, job types, submit, tracking, API reference, errors, configuration, Go example, operations, security, FAQ.
- Aturan sinkronisasi public documentation dan Swaggo untuk coding agents.
- Kebijakan pelaporan keamanan awal, termasuk kebutuhan verifikasi kanal pelaporan sebelum publikasi.

### Changed (documentation only)

- Menata ulang indeks dokumentasi pengguna, menambahkan navigasi kembali ke indeks, dan memperbarui rujukan ke `SYSTEM_DESIGN.md`.
- Memperjelas bahwa heartbeat dan finalisasi worker memakai lease aktif berdasarkan waktu MySQL setelah row lock. Menambahkan skenario expiry dan lock wait.
- Menetapkan token unik per claim outbox serta guard untuk finalisasi publish sukses maupun gagal.
- Menetapkan recovery sebelum claim MySQL untuk membuat generation/outbox baru tanpa menaikkan attempt. Memperjelas ACK ulang setelah jadwal `retrying` tersimpan.

- Menyamakan format pesan Redis v1 dan contoh `XADD`, serta menentukan penanganan envelope invalid atau versi yang belum didukung.
- Menetapkan rekonsiliasi berbasis bukti kehilangan Stream ID, termasuk tes yang membedakan pesan hilang dari backlog normal.
- Memisahkan verifikasi rollback ANT-03 dari assertion respons HTTP pada tugas berikutnya.

- Menetapkan pemulihan consumer group saat `NOGROUP` dan runbook untuk envelope Redis rusak dengan generation pengganti sebelum ACK.
- Membakukan canonical request dan key case-sensitive untuk ANT-03, termasuk fixture ekuivalensi dan konflik.

- Menetapkan bootstrap Compose dengan MySQL sehat dan migrasi sukses sebelum proses aplikasi dimulai. API dapat mulai saat Redis belum siap.
- Menyeragamkan waktu awal `available_at` pada setiap insert outbox dan menambahkan skenario verifikasi.

- Menetapkan limiter API yang independen dari Redis, kuota job aktif sejak status scheduled, dan ambang operasional untuk penerimaan job baru.
- Menetapkan object key report privat per client, job, dan attempt, dengan publikasi referensi hasil hanya melalui finalisasi lease yang sah.

- Mempertahankan token angka webhook dengan kolom payload teks JSON tervalidasi, mengisolasi idempotensi payment mock per client, dan melindungi pesan belum delivered saat trimming Redis.

### Fixed (documentation only, review 26 November 2026)

- Memilih satu aturan jadwal: insert baru dengan `scheduled_at` yang tidak lebih besar dari waktu MySQL ditolak (`422`). Retry identik tetap menemukan job asal.
- Menyeragamkan nama variabel pepper menjadi `AUTH_KEY_PEPPER_B64` dan menambah kolom `api_keys.pepper_version`.
- Menyelaraskan ERD dengan DDL (`payload` teks, `idempotency_key` biner, kolom `priority`, `started_at`, `request_hash`).
- Menetapkan seq outbox pertama job terjadwal (2) dan aturan parsing key `SplitN(…, "_", 4)`.
- Mengganti path modul placeholder dengan `github.com/lathief/Antriaga`, mem-pin `swag` v1.16.6, dan mencatat Go 1.26 untuk Fiber v3.7.0.
- Menambah catatan Windows (tanpa `make`, `-race` butuh CGO) dan catatan limiter per replica.

### Changed (documentation only, review portofolio)

- Menambah bagian [Prioritas portofolio](docs/ai/implementation-plan.md#prioritas-portofolio): cakupan inti `email.send` dan `webhook.deliver`, `payment.capture` opsional, `report.generate` dan MinIO ditunda ([backlog](docs/post-mvp.md)).
- Menambah deliverable portofolio yang direncanakan (demo crash, ukur, README portofolio, versi Inggris, pemindahan bagian proses AI, dokumen pembanding), metrik dasar bersama M2/M3, opsi requeue sederhana, dan risiko baru. Task 2 mewajibkan CI yang benar-benar berjalan.
- Tidak ada klaim demo, status CI, atau angka ukur. Semuanya berstatus direncanakan.

### Changed (documentation only, tindak lanjut review kedua)

- Memperbarui status Task 1 dan menandai klaim pemeriksaan tautan lama (130 dan 138) sebagai belum terverifikasi.
- Menambah prasyarat lingkungan pada Task 2 dan keputusan bentuk respons sebelum handler pada Task 9.
- Mencatat keputusan review di `docs/ai/decisions.md` dan memasukkan rotasi pepper ke backlog P1.

### Added (documentation only, review 26 November 2026)

- `CONTRIBUTING.md`, [template task](docs/ai/task-template.md), [kerangka runbook](docs/runbooks/README.md), dan [template ADR](design/adr/ADR-0000-template.md).

### Added (agent planning)

- [Rencana implementasi M1](docs/ai/implementation-plan.md), [matriks verifikasi](docs/ai/verification.md), [catatan keputusan](docs/ai/decisions.md), dan [format handoff](docs/ai/worklog.md).
- [Backlog setelah MVP](docs/post-mvp.md) dengan prasyarat, pemicu, dan kriteria selesai per kandidat.

Status runtime masih menunggu implementasi. Lihat [desain sistem](SYSTEM_DESIGN.md) untuk kontrak lengkap.
