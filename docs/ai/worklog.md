# Handoff implementasi Antriaga

Catat satu entri per pekerjaan. Gunakan ID kartu atau issue, dan sertakan commit jika tersedia. Tandai hasil sebagai `LULUS`, `GAGAL`, atau `BELUM DIJALANKAN` sesuai bukti. Sertakan alasan dan prasyarat untuk pemeriksaan yang tertunda.

## Entri awal — Paket kerja agent

- **Status:** dokumen perencanaan disiapkan. Implementasi runtime belum dimulai di snapshot ini.
- **Perubahan:** kartu tugas awal, skenario verifikasi, catatan keputusan, changelog blueprint, dan backlog setelah MVP.
- **Verifikasi dokumentasi:** pemeriksaan PowerShell lulus untuk 130 tautan relatif, struktur Markdown, dan tiga kartu tugas awal.
- **Verifikasi runtime:** belum dijalankan. Server dan suite pengujian belum tersedia.
- **Tugas terdekat:** [Task 2 (ANT-01) — Fondasi Go dan proses API](implementation-plan.md), setelah Task 1 (perbaikan dokumen).

## Review lanjutan — Kontrak worker dan tes penerimaan

- **Status:** pembaruan blueprint selesai.
- **Verifikasi dokumentasi:** Format XADD dibandingkan dengan payload v1. Kontrak rekonsiliasi, fencing state saat recheck, retensi Stream ID, dan kriteria ANT-03 diperiksa secara statis. Pemeriksaan tautan dan keberadaan fixture cURL lulus.
- **Verifikasi runtime:** belum dijalankan karena suite dan runtime belum tersedia.
- **Tugas berikutnya:** kartu fondasi yang tercantum pada entri awal.

## Review tiga proyek — isolasi report dan penerimaan job

- **Kontrak** — worker menulis report dengan key per client, job, dan attempt. Attempt dengan finalisasi sah menerbitkan referensi hasil. API memakai limiter lokal dan admisi MySQL, termasuk untuk job scheduled dan retrying, selama Redis putus. Blueprint, referensi API/konfigurasi, contoh payload, serta matriks tes sudah diselaraskan.
- **Bukti** — pemeriksaan Python dan `bash -n` lulus untuk 138 tautan, contoh Bash/JSON, fixture dedup, nilai konfigurasi admission, dan payload report tanpa key pilihan client.
- **Tahap berikutnya** — ANT-01 untuk fondasi, lalu ANT-03 untuk satu alur email dengan transactional outbox serta admisi baru. Tes Redis outage dan isolasi report menunggu runtime terkait.

## Review lanjutan — precision, payment mock, dan trimming

- **Kontrak** — payload JSON tervalidasi disimpan sebagai teks UTF-8. Key mock payment scoped ke client dan payment ID, dengan konflik permanen jika tuple berbeda. Trimming Redis 7 menjaga batas posisi delivery, PEL, dan bukti settlement MySQL.
- **Bukti** — pemeriksaan Python dan `bash -n` mencakup 138 tautan lokal, contoh Bash/JSON, dan kecocokan kontrak kolom, error mock, serta aturan cutoff trimming.
- **Berikutnya** — ANT-01, lalu skema dan submit. Uji round-trip angka webhook dan payment mock ketika handler tersedia. Uji cleanup Redis pada milestone P1.

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
