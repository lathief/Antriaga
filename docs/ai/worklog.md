# Handoff implementasi Antriaga

Catat satu entri per pekerjaan. Gunakan ID kartu atau issue, dan sertakan commit jika tersedia. Tandai hasil sebagai `LULUS`, `GAGAL`, atau `BELUM DIJALANKAN` sesuai bukti. Sertakan alasan dan prasyarat untuk pemeriksaan yang tertunda.

## Entri awal — Paket kerja agent

- **Status:** dokumen perencanaan disiapkan. Implementasi runtime belum dimulai di snapshot ini.
- **Perubahan:** kartu tugas awal, skenario verifikasi, catatan keputusan, changelog blueprint, dan backlog setelah MVP.
- **Verifikasi dokumentasi:** pemeriksaan PowerShell lulus untuk 130 tautan relatif, struktur Markdown, dan tiga kartu tugas awal. *(Klaim historis, belum terverifikasi: skripnya tidak ada di repositori. Verifikasi ulang pada Task 1 memakai skrip sementara: 161 tautan, 0 rusak.)*
- **Verifikasi runtime:** belum dijalankan. Server dan suite pengujian belum tersedia.
- **Tugas terdekat:** [Task 2 (ANT-01) — Fondasi Go dan proses API](implementation-plan.md), setelah Task 1 (perbaikan dokumen).

## Review lanjutan — Kontrak worker dan tes penerimaan

- **Status:** pembaruan blueprint selesai.
- **Verifikasi dokumentasi:** Format XADD dibandingkan dengan payload v1. Kontrak rekonsiliasi, fencing state saat recheck, retensi Stream ID, dan kriteria ANT-03 diperiksa secara statis. Pemeriksaan tautan dan keberadaan fixture cURL lulus.
- **Verifikasi runtime:** belum dijalankan karena suite dan runtime belum tersedia.
- **Tugas berikutnya:** kartu fondasi yang tercantum pada entri awal.

## Review tiga proyek — isolasi report dan penerimaan job

- **Kontrak** — worker menulis report dengan key per client, job, dan attempt. Attempt dengan finalisasi sah menerbitkan referensi hasil. API memakai limiter lokal dan admisi MySQL, termasuk untuk job scheduled dan retrying, selama Redis putus. Blueprint, referensi API/konfigurasi, contoh payload, serta matriks tes sudah diselaraskan.
- **Bukti** *(klaim historis, belum terverifikasi; skrip tidak ada di repositori)* — pemeriksaan Python dan `bash -n` lulus untuk 138 tautan, contoh Bash/JSON, fixture dedup, nilai konfigurasi admission, dan payload report tanpa key pilihan client.
- **Tahap berikutnya** — ANT-01 untuk fondasi, lalu ANT-03 untuk satu alur email dengan transactional outbox serta admisi baru. Tes Redis outage dan isolasi report menunggu runtime terkait.

## Review lanjutan — precision, payment mock, dan trimming

- **Kontrak** — payload JSON tervalidasi disimpan sebagai teks UTF-8. Key mock payment scoped ke client dan payment ID, dengan konflik permanen jika tuple berbeda. Trimming Redis 7 menjaga batas posisi delivery, PEL, dan bukti settlement MySQL.
- **Bukti** *(klaim historis, belum terverifikasi)* — pemeriksaan Python dan `bash -n` mencakup 138 tautan lokal, contoh Bash/JSON, dan kecocokan kontrak kolom, error mock, serta aturan cutoff trimming.
- **Berikutnya** — ANT-01, lalu skema dan submit. Uji round-trip angka webhook dan payment mock ketika handler tersedia. Uji cleanup Redis pada milestone P1.

## Task 1 — Perbaikan dokumen hasil review (branch `docs/review-fixes`)

- **Tujuan dan status:** menutup temuan review 26 November 2026. Selesai. Commit `1e57d5b`, di-merge ke `master` lewat PR #1 (`d166d0e`).
- **Berkas:** `SYSTEM_DESIGN.md`, `AGENTS.md`, `CHANGELOG.md`, `README.md`, `docs/README.md`, `docs/reference/configuration.md`, `docs/getting-started/installation.md`, `docs/ai/implementation-plan.md`. Baru: `CONTRIBUTING.md`, `docs/ai/task-template.md`, `docs/runbooks/README.md`, `design/adr/ADR-0000-template.md`.
- **Perubahan kontrak:** aturan `scheduled_at` masa lalu (ditolak 422), nama `AUTH_KEY_PEPPER_B64`, kolom `api_keys.pepper_version`, seq outbox pertama job terjadwal (2), parsing key `SplitN(…, "_", 4)`, ERD diselaraskan dengan DDL.
- **Pemeriksaan:** skrip Python sementara memeriksa 161 tautan relatif Markdown: 0 rusak (LULUS). Klaim 130 dan 138 tautan pada entri lama tidak diverifikasi ulang dan tidak ada skripnya di repositori.
- **Belum dijalankan:** lint Markdown dan seluruh tes runtime (belum ada kode).
- **Berikutnya:** Task 2 (fondasi Go dan `scripts/check.ps1`).

## Tindak lanjut review kedua (branch `docs/review-followup`)

- **Tujuan dan status:** menutup sisa catatan review sebelum Task 2. Hanya dokumen, tidak ada kode.
- **Perubahan:** status Task 1 diperbarui, klaim pemeriksaan tautan lama ditandai belum terverifikasi, prasyarat lingkungan ditambah ke Task 2, keputusan bentuk respons ditambah ke Task 9, keputusan review dicatat di `decisions.md`, dan rotasi pepper masuk backlog P1.
- **Pemeriksaan:** pemeriksa tautan sementara (di luar repositori) dijalankan setelah perubahan. Hasilnya ada di entri commit PR.
- **Belum dijalankan:** `go version`, `docker version`, `gcc --version`. Pengecekan itu menjadi langkah pertama Task 2.
- **Berikutnya:** Task 2 setelah branch ini di-merge.

## Review portofolio (branch `docs/portfolio-reprioritize`)

- **Tujuan dan status:** menyesuaikan rencana untuk tujuan portofolio backend middle/senior. Rencana diperbarui, belum ada kode.
- **Berkas:** `docs/ai/implementation-plan.md`, `docs/ai/decisions.md`, `docs/post-mvp.md`, `docs/ai/worklog.md`, `CHANGELOG.md`, `README.md`.
- **Perubahan:** bagian Prioritas portofolio (inti `email.send` dan `webhook.deliver`), catatan cakupan di Task 3, 7, dan 9, CI nyata di Task 2, deliverable portofolio (demo crash, ukur, README, versi Inggris, pemindahan §36–§44, pembanding alternatif), metrik dasar bersama M2/M3, opsi requeue, dan risiko baru. Nomor Task 1 sampai 10 tidak berubah.
- **Koreksi:** pada Task 2 ada duplikasi teks `go mod init` hasil edit sebelumnya, sudah dirapikan.
- **Pemeriksaan:** pemeriksa tautan sementara (di luar repositori) dijalankan, hasil di bawah entri CHANGELOG dan laporan commit.
- **Belum dijalankan:** lint Markdown. Tidak ada tes runtime karena belum ada kode. Tidak ada demo, benchmark, atau status CI yang diklaim.
- **Berikutnya:** Task 2 setelah branch ini di-merge. Pemindahan §36–§44 dijadwalkan setelah M1.

## Review multi-peran (branch `docs/multirole-review`)

- **Tujuan dan status:** mencatat hasil review dari lima peran ke dokumen. Hanya dokumen, tidak ada kode.
- **Temuan per peran:**
  - **Backend:** hitungan admisi global berbatas, SQL eksplisit di jalur panas.
  - **DevOps/SRE:** grace period platform di atas 30 detik, kapasitas koneksi MySQL, `METRICS_ADDR` per binary, dashboard dan alert sebagai berkas, `govulncheck`, image GHCR, MySQL sebagai titik tunggal.
  - **Principal:** positioning proyek di README dan ADR-0001 dengan pembanding alternatif.
  - **Product owner:** halaman produk dengan pengguna dan tiga metrik sukses, catatan `priority` tidak diekspos.
  - **Pengguna:** seed lokal untuk quickstart, `Location` dan penanda replay, field respons, error validasi per field. Callback saat selesai dan webhook terdaftar per client masuk backlog.
- **Berkas:** `docs/ai/implementation-plan.md`, `docs/ai/decisions.md`, `docs/operations/deployment.md`, `docs/operations/monitoring.md`, `docs/reference/configuration.md`, `docs/reference/api.md`, `docs/getting-started/quickstart.md`, `docs/post-mvp.md`, `docs/README.md`, `README.md`, `CHANGELOG.md`. Baru: `docs/product.md`, `design/adr/ADR-0001-redis-streams-dengan-mysql-outbox.md`.
- **Pemeriksaan:** pemeriksa tautan relatif dijalankan setelah perubahan, hasilnya dicatat di laporan commit.
- **Belum dijalankan:** lint Markdown. Tidak ada tes runtime karena belum ada kode. Tidak ada klaim ukur, demo, atau status CI. Semua catatan baru berstatus rencana dan belum diverifikasi.
- **Branch:** `docs/portfolio-reprioritize` sudah di-merge lewat PR #3, jadi pekerjaan ini memakai branch baru dari `master`.
- **Berikutnya:** Task 2 setelah branch ini di-merge.

## Format entri

```text
Tanggal dan ID tugas:
Tujuan dan status:
Berkas atau commit:
Perintah yang dijalankan, hasil, dan lokasi bukti:
Pemeriksaan tertunda dan alasannya:
Keputusan yang berubah (tautan ke decisions.md):
Risiko atau blocker yang masih terbuka:
Satu tugas berikutnya dan prasyaratnya:
```

Lihat [kriteria verifikasi](verification.md), [catatan keputusan](decisions.md), dan [changelog blueprint](../../CHANGELOG.md).
