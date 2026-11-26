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

### Added (agent planning)

- [Kartu tugas awal](docs/ai/implementation-plan.md), [matriks verifikasi](docs/ai/verification.md), [catatan keputusan](docs/ai/decisions.md), dan [format handoff](docs/ai/worklog.md).
- [Backlog setelah MVP](docs/post-mvp.md) dengan prasyarat, pemicu, dan kriteria selesai per kandidat.

Status runtime masih menunggu implementasi. Lihat [desain sistem](SYSTEM_DESIGN.md) untuk kontrak lengkap.
