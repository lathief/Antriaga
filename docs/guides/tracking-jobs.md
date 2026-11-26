# Melacak status dan riwayat job

## Siklus hidup job

```text
created -> scheduled -> queued -> processing -> completed
             |             |          |
             |             |          +-> retrying -> queued
             |             |          +-> dead
             |             +-> [worker pending, recoverable]
             +-> [waiting for due time]
```

| Status | Keterangan |
|---|---|
| `scheduled` | Menunggu waktu pelaksanaan awal |
| `queued` | Siap diproses; mungkin masih menunggu outbox/Redis/worker |
| `processing` | Sedang dijalankan satu worker dengan DB lease |
| `retrying` | Gagal sementara, menunggu exponential backoff |
| `completed` | Hasil eksekusi telah berhasil di-commit |
| `dead` | Permanent failure atau attempts habis |
| `cancelled` | Disiapkan di state machine, tetapi cancel API **bukan MVP** |

Tidak ada status `failed` terpisah pada MVP; kegagalan setiap percobaan dicatat pada attempts, sedangkan job keseluruhan menjadi `retrying` atau `dead`.

## Periksa job dan riwayat percobaan

```bash
curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "$ANTRIAGA_URL/api/v1/jobs/<JOB_ID>"

curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "$ANTRIAGA_URL/api/v1/jobs/<JOB_ID>/attempts"

curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "$ANTRIAGA_URL/api/v1/jobs?status=dead&page=1&limit=20"
```

`jobs:read` wajib; hasil hanya milik client tersebut. Error message disanitasi dan tidak mengandung credential atau raw external payload.

## Memahami retry

Secara default max attempts 5, base backoff 5 detik, cap 15 menit, jitter 0–20% (semua **konfigurasi awal**). Saat handler gagal secara retryable dan attempts masih tersisa, job masuk `retrying` dengan jadwal berikutnya; scheduler mengembalikannya ke queue ketika due.

Jika worker kehilangan lease atau gagal menyimpan hasil, pesan tetap menunggu pemulihan. Reaper menutup percobaan dengan lease kedaluwarsa dan menjadwalkan eksekusi ulang jika jatah percobaan masih ada. Gunakan job ID yang sama untuk memantau pemulihan setelah worker restart.

## At-least-once dan external side effects

Worker bisa menyelesaikan side effect lalu crash sebelum commit DB; eksekusi ulang dapat terjadi. Untuk email/webhook/payment gunakan idempotency downstream, bukan mengandalkan job status untuk exactly-once.

## Fitur yang belum tersedia di MVP

- `POST /api/v1/jobs/:id/requeue`: **P1**, membutuhkan scope `jobs:requeue`, owner-check, reason, audit.
- `GET /api/v1/jobs/:id/events`: **P2** durable event timeline.
- Cancel/pause/resume queue: fase lanjutan.

Operator dapat melihat structured logs berdasarkan `job_id` untuk diagnosis. Lihat [Monitoring](../operations/monitoring.md) dan [Troubleshooting](../operations/troubleshooting.md).

[Kembali ke indeks dokumentasi](../README.md)
