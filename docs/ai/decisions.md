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

| Insert baru dengan `scheduled_at` yang tidak lebih besar dari waktu MySQL ditolak (`422`) | Retry identik tetap menemukan job asal setelah jadwal lewat | Blueprint §§12.4, 27.7 |
| Pepper bernama `AUTH_KEY_PEPPER_B64`, digest mencatat `api_keys.pepper_version` (MVP hanya versi 1) | Rotasi pepper ditunda ke P1. Kolom sudah tersedia agar tidak butuh migrasi ulang | Blueprint §27.2, §27.3, [backlog](../post-mvp.md) |
| Outbox pertama job terjadwal memakai `enqueue_seq = 2` | Job dibuat dengan seq 1 tanpa outbox, lalu seq naik saat release | Blueprint §9.3 |
| Parser API key memakai `SplitN(key, "_", 4)` | Secret base64url boleh memuat `_` | Blueprint §27.1 |
| Bentuk respons job dan attempts ditetapkan di Task 9, sebelum handler ditulis | Keputusan field dicatat di sini lalu `api.md` diperbarui | [Rencana Task 9](implementation-plan.md) |

| Cakupan awal dipangkas: `email.send` dan `webhook.deliver` dulu, `payment.capture` opsional, `report.generate` dan MinIO ditunda | Proyek portofolio harus selesai dan dapat dibuktikan. Kontrak job type yang ditunda tetap di blueprint | [Prioritas portofolio](implementation-plan.md#prioritas-portofolio), [backlog](../post-mvp.md) |
| Demo crash satu perintah menjadi deliverable M3 | Bukti recovery yang bisa dipamerkan, bukan hanya dokumen | [Deliverable portofolio](implementation-plan.md#deliverable-portofolio-direncanakan) |
| CI nyata sejak Task 2 (bukan skeleton) | Badge CI hijau sebagai sinyal kualitas. Badge baru dipasang setelah workflow pernah lulus | [Task 2](implementation-plan.md) |
| Metrik Prometheus dasar dibangun bersama worker dan dispatcher (M2 dan M3) | Bahan untuk pengukuran dan demo, tidak menunggu M4 | [Metrik dasar](implementation-plan.md#metrik-dasar-dibangun-bersama-m2-dan-m3) |
| README portofolio dan versi Inggris dijadwalkan setelah demo dan pengukuran | Tidak ada klaim demo atau angka sebelum ada keluaran nyata | [Deliverable c dan d](implementation-plan.md#deliverable-portofolio-direncanakan) |
| Bagian proses AI (§36–§44) dipindah ke `docs/ai/` setelah M1 | Mengurangi penekanan pada proses AI di blueprint tanpa memutus rujukan sekarang | [Deliverable e](implementation-plan.md#deliverable-portofolio-direncanakan) |
| Requeue dead job sederhana dipertimbangkan naik ke MVP, diputuskan di akhir M3 | Kelengkapan operasi dead job, bergantung pada waktu | [Opsi requeue](implementation-plan.md#opsi-requeue-dead-job-sederhana-ke-mvp) |

Untuk entri baru, catat keputusan, alasan, konsekuensi untuk API/schema/tes, dan rujukan ke perubahan. Bukti eksekusi per tugas masuk ke [worklog](worklog.md).
