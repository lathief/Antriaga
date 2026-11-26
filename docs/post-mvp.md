# Backlog Antriaga setelah MVP

> Semua item masih berupa kandidat. Prioritas mengikuti [blueprint §§31–34](../SYSTEM_DESIGN.md) dan bukti saat runtime sudah digunakan.

## Kapan backlog bisa dimulai

- Seluruh M1–M4 memenuhi [kriteria blueprint](../SYSTEM_DESIGN.md#22-definition-of-done). Panduan cepat berhasil dari lingkungan bersih dan jalur recovery kritis lulus tes integrasi.
- Catat baseline dengan profil beban dan jumlah worker yang jelas. Ukur p95 waktu tunggu job, usia outbox tertua, throughput, dan hasil recovery saat Redis atau worker berhenti. Sepakati target yang sesuai beban, lalu ulangi pengukuran untuk setiap optimasi.
- Temuan yang masih mengganggu durabilitas, keamanan, atau pemulihan kembali ke backlog MVP. Catat bukti dan blocker di [worklog](ai/worklog.md).

## P1 — Operasional dan pemulihan

| Urutan | Kandidat | Ambil saat | Kriteria selesai |
|---|---|---|---|
| 1 | Retensi history dan outbox, serta trimming Redis yang aman (§32) | Pertumbuhan data mulai membebani disk, backup, atau query | Cleanup berbatch. Pending/unacked dan state yang masih dibutuhkan recovery dilindungi. Simulasi restart dan recovery tetap lulus setelah cleanup. |
| 2 | Requeue manual untuk job `dead` (§31) | Operator membutuhkan cara terkontrol untuk mengulang kegagalan yang sudah diperbaiki | Job baru tertaut ke asal, scope dan owner diperiksa, audit tercatat, request ulang tidak membuat dua job, dan kontrak idempotensi downstream diuji. |
| 3 | Runbook insiden, SLO, dan latihan restore (§33) | Sudah ada baseline dan operator dapat mengukur backlog serta waktu pulih | Alert terhubung ke runbook. Latihan gangguan Redis dan restore MySQL menghasilkan waktu pemulihan serta hasil rekonsiliasi yang terdokumentasi. |

## P2 — Riwayat yang dapat ditanyakan

| Kandidat | Ambil saat | Kriteria selesai |
|---|---|---|
| Durable job event timeline (§34) | Riwayat attempt dan log belum cukup untuk menjawab urutan transisi setelah insiden | Event transisi commit bersama perubahan state. Endpoint berpaginasi, terisolasi per client, dan mengikuti kebijakan retensi. Ukur tambahan volume tulis. |

Dashboard operator, cancel API, prioritas, dan perubahan broker memerlukan kasus penggunaan yang terukur serta keputusan desain terpisah. Prioritaskan tuning query, batch, dan jumlah worker ketika pengukuran menunjukkan bottleneck pada jalur yang ada.
