# Runbook insiden (P1, kerangka)

> **Status:** direncanakan untuk fase P1 setelah MVP. Isi runbook diturunkan dari [blueprint §33](../../SYSTEM_DESIGN.md). Folder ini baru berupa kerangka, belum ada prosedur yang diverifikasi.

Setiap runbook memakai urutan: gejala, pemeriksaan, tindakan aman, validasi pemulihan, eskalasi. Jangan menjalankan `XACK`, `XDEL`, `DELETE`, atau mengubah status job secara manual tanpa membuktikan invarian MySQL dan mencatat tindakan operator.

## Daftar yang direncanakan

| Insiden | Rujukan |
|---|---|
| Redis tidak tersedia | §33.1 |
| Worker mati atau terjebak | §33.1 |
| Banyak job `dead` | §33.1 |
| MySQL gagal | §33.1 |
| Backlog outbox | §33.1 |
| Lease hilang berulang | §33.1 |
| Disk penuh | §33.1 |
| API key diduga bocor | §33.1 dan [keamanan operasional](../operations/security.md) |

Untuk diagnosis awal gunakan [pemecahan masalah](../operations/troubleshooting.md).

[Kembali ke indeks dokumentasi](../README.md)
