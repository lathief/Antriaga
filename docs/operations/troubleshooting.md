# Pemecahan masalah

> Untuk API user, mulai dari HTTP status/error envelope. Untuk operator, gunakan status job di MySQL, metrics, dan structured logs. **Jangan** mengubah status job langsung via SQL untuk "memperbaiki" antrean.

| Gejala | Periksa | Tindakan aman |
|---|---|---|
| `401 INVALID_API_KEY` | Key, expiry, client disabled, environment | Minta admin verifikasi metadata/rotate key; jangan log secret |
| `403 INSUFFICIENT_SCOPE` | Scope dari key | Admin buat key dengan scope benar; jangan hardcode super-admin |
| `404` pada job milik client lain | Client identity dan job ID | Pastikan client benar; respons sengaja tidak membocorkan ownership |
| `409` | Request identik vs idempotency key | Gunakan key yang sama **hanya** untuk logical request sama |
| `422 UNSUPPORTED_PAYLOAD_VERSION` | Versi handler/API | Gunakan versi yang didukung; jangan edit job existing |
| `429` | Quota/rate limit | Honor Retry-After bila ada; bounded backoff dan jitter |
| `503` | MySQL tidak tersedia atau ambang job aktif/outbox tercapai | Periksa readiness, job aktif termasuk scheduled, dan backlog outbox. Retry submit dengan key yang sama |
| `queued` lama | Pending outbox, dispatcher, Redis, worker slots | Periksa age outbox, log dispatcher termasuk `outbox.lease_lost`, consumer group, dan backpressure. Jika worker berhenti sebelum claim MySQL, reaper menerbitkan generation baru tanpa menaikkan attempt |
| `retrying` lama | `scheduled_at`, scheduler, clock, queue health | Periksa log scheduler dan due state. Reaper dapat membersihkan pesan lama setelah jadwal retry tersimpan tanpa mengubah jadwal itu |
| `processing` lama | Lease, heartbeat, worker, external service | Cari `worker.lease_lost` dan `reaper.job_recovered` untuk `job_id` yang sama. Periksa heartbeat, waktu expiry di MySQL, dan keterlambatan reaper. Jangan XACK manual tanpa validasi DB |
| Banyak `dead` | Attempts error_code, provider status, config | Perbaiki penyebab; P1 requeue butuh audit dan owner authorization |
| Duplikasi side effect | External idempotency key dan crash windows | Periksa dedup di provider; jangan menjanjikan exactly-once |
| Redis restart atau `NOGROUP` | Ketersediaan Redis, consumer group, PEL, dan outbox | Worker dan reaper mencoba menyiapkan ulang group mulai dari `0`. Periksa readiness, hak akses Redis, dan bahwa backlog dikonsumsi kembali |

## Diagnosis read-only

```bash
curl --fail http://localhost:8080/health/live
curl -i http://localhost:8080/health/ready

docker compose -f deployments/docker-compose.yml logs --tail=200 \
  api dispatcher worker scheduler reaper
```

Operator dapat memakai SQL read-only (dengan DB account read-only yang dibatasi) untuk memeriksa jumlah status, misalnya:

```sql
SELECT status, COUNT(*) AS total FROM jobs GROUP BY status;
SELECT status, COUNT(*) AS total FROM outbox_events GROUP BY status;
```

Hindari query yang mencetak payload dan API secret. Query di atas untuk diagnosis; tidak memperbaiki state.

## Pemulihan envelope Redis rusak

Gunakan jalur ini untuk pesan yang dipastikan rusak permanen. Untuk versi transport yang valid tetapi belum didukung, pulihkan kompatibilitas consumer lalu jalankan scan PEL biasa. Lihat [kontrak transport](../../SYSTEM_DESIGN.md#82-message-payload).

1. Catat insiden, stream, group, Stream ID, versi yang terlihat, dan kesalahan decoder dalam catatan operasi yang disimpan. Lakukan diagnosis pada Redis primary dan MySQL tanpa mengekspos payload atau kredensial.
2. Temukan outbox `published` berdasarkan stream dan `published_stream_id`. Cocokkan identitas event dengan bukti independen, misalnya `event_id` yang masih terbaca dan cocok dengan row outbox. Stream ID saja tidak cukup karena dapat dipakai ulang setelah reset. Jika asalnya tidak dapat dipastikan, hentikan perbaikan dan eskalasikan.
3. Untuk job `queued` pada generation aktif, jalankan perbaikan melalui prosedur maintenance yang diotorisasi. Di satu transaksi, lock job lalu outbox lama, periksa ulang status, generation, dan identitas publish, naikkan generation, lalu insert satu outbox baru. Jangan menambah attempt. Simpan hasil dan ID outbox pengganti pada catatan insiden setelah commit. Jika job sudah terminal atau generation lama sudah diganti, catat bukti durable tersebut tanpa membuat generation baru. Jangan mengganggu attempt `processing`; biarkan recovery lease yang berlaku menanganinya.
4. ACK pesan lama hanya setelah state durable terbukti dan catatan insiden tersimpan. Periksa identitas entri dan ACK secara atomik pada Redis agar ID yang dipakai ulang tidak ikut tersentuh. Jika verifikasi gagal, hentikan ACK dan diagnosis ulang. Ulang prosedur dengan insiden yang sama harus melihat generation pengganti yang sudah tersimpan dan tidak membuat outbox tambahan.

Uji alur pada Redis/MySQL terisolasi dengan envelope v1 yang rusak tetapi asalnya dapat diverifikasi, commit yang gagal, dan versi valid yang belum didukung. Untuk commit gagal, pesan tetap pending dan tidak ada outbox pengganti.

## Kapan mengeskalasi?

- Outbox age terus membesar walau dispatcher aktif.
- Lease expired terus-menerus atau handler melakukan side effect ganda.
- API tetap menerima request, tetapi job tidak pernah menjadi `completed` atau `dead`.
- Migrations gagal atau pepper API key hilang/terkompromi.

Catat `request_id`, `job_id`, component, event, timestamp UTC, dan `error_code` yang sudah teredaksi. Jangan kirim API key ke channel support.

Lihat [Monitoring](monitoring.md), [Deployment](deployment.md), dan [Security](security.md).

[Kembali ke indeks dokumentasi](../README.md)
