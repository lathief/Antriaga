# Mengirim dan menjadwalkan job

## Submit immediate job

`POST /api/v1/jobs` membutuhkan `jobs:create`. Queue ditentukan server berdasarkan `type`, bukan arbitrarily oleh client. Set `payload_version` agar perubahan format dapat dipantau.

```bash
curl --fail-with-body -X POST "$ANTRIAGA_URL/api/v1/jobs" \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  -H "Content-Type: application/json" \
  -H 'Idempotency-Key: order-10021-webhook' \
  --data '{
    "type": "webhook.deliver",
    "payload_version": 1,
    "payload": {
      "event": "order.completed",
      "reference_id": "ORD-10021",
      "data": {"order_id":"ORD-10021", "total":250000}
    }
  }'
```

**Hasil `202 Accepted`**: job disimpan durable di MySQL, dan dispatcher akan mempublikasikannya melalui transactional outbox. `202` tidak berarti handler sukses.

## Scheduled job

Kirim `scheduled_at` sebagai RFC3339/UTC. Contoh waktu berikut **harus diganti ke waktu di masa depan** pada saat digunakan:

```json
{
  "type": "report.generate",
  "payload_version": 1,
  "scheduled_at": "2030-01-01T01:00:00Z",
  "payload": {
    "report_type": "monthly_sales",
    "period": "2026-10",
    "format": "csv"
  }
}
```

Job berstatus `scheduled` sampai scheduler menandai due dan mengantrekan pekerjaan. Scheduling bersifat **best effort**, bergantung pada polling/backlog; bukan real-time hard guarantee.

## Idempotency-Key untuk submit

Gunakan `Idempotency-Key` berisi 1–150 byte ASCII tercetak tanpa spasi. Key dibandingkan secara case-sensitive dalam satu client. Request dengan key yang sama mengembalikan job sebelumnya jika representasi canonical sama. Urutan field objek, whitespace, nilai default eksplisit, dan offset timestamp yang menunjuk milidetik UTC yang sama tidak membuat conflict. Urutan array dan token angka payload tetap signifikan. Perbedaan isi canonical menghasilkan `409`. Retry identik dengan jadwal yang sudah lewat tetap dapat menemukan job asal. Lihat [aturan canonical lengkap](../../SYSTEM_DESIGN.md#277-canonical-request-untuk-deduplikasi).

- Ruang dedup adalah `(client_id, idempotency_key)`.
- Key antar client boleh sama tanpa membuka job client lain.
- Jangan membuat key baru ketika sekadar mengulangi request setelah network timeout.
- Idempotency untuk **penerimaan job** berbeda dari idempotency **side effect external** pada handler; Antriaga tetap memiliki at-least-once execution.

## `payload_version`

MVP menggunakan versi `1` sebagai default. Versi tidak dikenal ditolak dengan `422 UNSUPPORTED_PAYLOAD_VERSION`. Nilai versi tersimpan dengan job dan tidak berubah selama lifecycle. Untuk integrasi jangka panjang, tampilkan versi secara eksplisit dan uji compatibility sebelum upgrade.

## Validasi dasar

- Job type hanya salah satu dari 4 type MVP.
- Payload JSON harus valid untuk type tersebut; body maksimum 256 KiB. Worker mempertahankan token angka pada `webhook.deliver.data` dari submit sampai POST outbound.
- `max_attempts` mengikuti policy deployment (contoh rentang 1–10).
- `scheduled_at` wajib valid dan diizinkan oleh policy scheduling.
- Field `queue` opsional hanya jika cocok dengan mapping type; sebaiknya **tidak dikirim**.
- Tidak ada generic URL/script executor. Target webhook/provider diatur operator server-side.

Lanjut ke [Job Types](job-types.md) dan [Errors](../reference/errors.md).

[Kembali ke indeks dokumentasi](../README.md)
