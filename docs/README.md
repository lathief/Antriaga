# Dokumentasi Antriaga

Antriaga dirancang untuk menerima dan menjalankan job di latar belakang. Gunakan panduan ini untuk menyiapkan layanan, mengintegrasikan API, dan memantau eksekusi job.

> **Status:** Dokumentasi ini menjelaskan spesifikasi MVP berdasarkan [desain sistem v4](../SYSTEM_DESIGN.md). Perintah, konfigurasi, dan respons API adalah contoh kontrak yang direncanakan. Verifikasi terhadap runtime masih diperlukan.

## Mulai menggunakan Antriaga

1. [Instalasi dan persyaratan](getting-started/installation.md) untuk menyiapkan layanan lokal.
2. [Autentikasi dan API key](guides/authentication.md) untuk meminta kredensial dengan scope yang sesuai.
3. [Panduan cepat](getting-started/quickstart.md) untuk mengirim job pertama dan memeriksa hasilnya.

## Integrasi API

- [Mengirim dan menjadwalkan job](guides/submitting-jobs.md) membahas jadwal, idempotensi, dan versi payload.
- [Jenis job MVP](guides/job-types.md) memuat contoh payload email, webhook, simulasi pembayaran, dan laporan.
- [Melacak status dan riwayat job](guides/tracking-jobs.md) menjelaskan status, percobaan eksekusi, dan retry.
- [Contoh integrasi Go](examples/go-client.md) menggunakan `net/http`.

## Referensi

- [API HTTP](reference/api.md)
- [Konfigurasi](reference/configuration.md)
- [Error HTTP dan retry dari client](reference/errors.md)

## Operasional

- [Deployment dan pembaruan versi](operations/deployment.md)
- [Pemantauan, log, dan peringatan](operations/monitoring.md)
- [Pemecahan masalah](operations/troubleshooting.md)
- [Keamanan operasional](operations/security.md)
- [Pertanyaan umum](faq.md)

## Pengembangan

- [Paket kerja awal](ai/implementation-plan.md), [matriks verifikasi](ai/verification.md), dan [handoff](ai/worklog.md).
- [Catatan keputusan](ai/decisions.md), [template task](ai/task-template.md), [changelog](../CHANGELOG.md), dan [backlog setelah MVP](post-mvp.md).
- [Panduan kontribusi](../CONTRIBUTING.md), [template ADR](../design/adr/ADR-0000-template.md), dan [kerangka runbook P1](runbooks/README.md).

## Status dan konvensi

- Gunakan `http://localhost:8080` untuk pengembangan lokal dan HTTPS untuk akses melalui jaringan.
- Ganti placeholder seperti `<JOB_ID>` dengan nilai dari lingkungan Anda. Masukkan `ANTRIAGA_API_KEY` melalui pengelola rahasia.
- Waktu pada contoh menggunakan UTC. Ketepatan jadwal bergantung pada scheduler dan antrean.
- Label **MVP**, **P1**, dan **P2** menunjukkan tahap rencana. P1 dan P2 dijadwalkan setelah MVP.
- Periksa status HTTP dan gunakan field respons yang didokumentasikan. Server dapat menambahkan field baru.
- Swagger UI direncanakan tersedia di `/swagger/index.html` ketika diaktifkan. Lindungi UI sebelum memberi akses melalui jaringan.

## Pemeliharaan dokumentasi

| Perubahan | Dokumen yang perlu diperbarui |
|---|---|
| Endpoint, status HTTP, atau DTO | [API HTTP](reference/api.md), [error HTTP](reference/errors.md), dan anotasi Swaggo |
| Scope atau autentikasi | [Autentikasi dan API key](guides/authentication.md), [keamanan operasional](operations/security.md) |
| Payload atau versinya | [Jenis job](guides/job-types.md), [pengiriman job](guides/submitting-jobs.md), dan contoh terkait |
| Konfigurasi atau deployment | [Konfigurasi](reference/configuration.md), [deployment](operations/deployment.md) |
| Retry atau pemulihan worker | [Pelacakan job](guides/tracking-jobs.md), [pemecahan masalah](operations/troubleshooting.md) |

Sebelum rilis, uji panduan cepat serta skenario API key tidak valid, akses job lintas client yang menghasilkan `404`, dan versi payload yang tidak didukung.
