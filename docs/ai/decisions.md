# Catatan keputusan Antriaga

Keputusan teknis lengkap tetap berada di [blueprint §23](../../SYSTEM_DESIGN.md#23-architectural-decision-records-adr-ringkas) dan bagian yang dirujuk di bawah. Catat perubahan penting di sini ketika kontrak desain berubah, dengan tautan ke bagian blueprint yang diperbarui.

| Keputusan | Dampak implementasi | Rujukan |
|---|---|---|
| MySQL menyimpan state; Redis Streams membawa notifikasi kerja | Job immediate dan outbox commit atomik sebelum publish | Blueprint §§7–9 |
| Lease worker dan dispatcher diperiksa dengan waktu MySQL setelah lock | Hasil terlambat ditolak; claim dispatcher memiliki token sendiri | Blueprint §§9.1–9.2 |
| Rekonsiliasi queued memakai identitas publish durable | Dispatcher menyimpan Stream ID. Reconciler hanya republish setelah kehilangan dikonfirmasi Redis dan state tetap cocok | Blueprint §29.4 |
| Recovery sebelum worker claim membuat generation baru | Tidak menambah attempt. ACK setelah state durable | Blueprint §§8.4, 29.2 |
| ACK ulang pada `retrying` menghormati jadwal tersimpan | Scheduler tetap satu-satunya jalur release retry | Blueprint §§8.4, 9.3 |
| API limiter lokal dan admisi MySQL | Redis outage tidak memblokir submit selama ada kapasitas. Job terjadwal langsung masuk hitungan aktif | Blueprint §28 |
| Payload job disimpan sebagai JSON tervalidasi dalam kolom teks | Token angka webhook bertahan dari ingest hingga pengiriman | Blueprint §§7, 10 |
| Idempotensi payment mock terikat client dan payment ID | Tuple berbeda ditolak tanpa capture baru | Blueprint §10 |
| Trimming Redis menjaga batas delivery dan pending | Hanya prefix settled boleh dipotong, termasuk saat reconciler aktif | Blueprint §32 |
| Key report unik per client, job, dan attempt | Hanya attempt dengan finalisasi yang sah menerbitkan referensi hasil | Blueprint §§9.2, 10.2 |

Untuk entri baru, catat keputusan, alasan, konsekuensi untuk API/schema/tes, dan rujukan ke perubahan. Bukti eksekusi per tugas masuk ke [worklog](worklog.md).
