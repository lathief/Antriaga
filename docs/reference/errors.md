# Error HTTP dan retry dari client

## Error envelope

```json
{
  "error": {
    "code": "UNSUPPORTED_JOB_TYPE",
    "message": "job type is not supported",
    "request_id": "req-019a9e11"
  }
}
```

`request_id` adalah correlation identifier untuk dukungan/debugging; tidak boleh menampilkan API key.

## Status code

| HTTP | Contoh kode / kondisi | Tindakan client |
|---:|---|---|
| `400` | Malformed JSON, nama object berulang, atau format field/header invalid | Perbaiki request |
| `401` | `INVALID_API_KEY` | Periksa key, expiry, client status; **jangan** retry tanpa memperbaiki akses |
| `403` | `INSUFFICIENT_SCOPE` | Minta operator memberikan scope; jangan eskalasi otomatis |
| `404` | Job tidak ada atau bukan milik client | Verifikasi `job_id`/client; hindari enumeration |
| `409` | Idempotency conflict / invalid state transition | Pastikan request logical identik untuk key yang sama |
| `413` | Body melebihi batas | Simpan file besar di object storage, kirim metadata |
| `422` | Domain validation, jadwal baru yang tidak di masa depan, atau `UNSUPPORTED_PAYLOAD_VERSION` | Perbaiki payload/version |
| `429` | `RATE_LIMITED` atau `CLIENT_QUEUE_LIMIT_EXCEEDED` | Hormati `Retry-After` bila ada; retry dengan jitter |
| `500` | Internal error | Retry *submission* secara bounded dengan key idempotency sama |
| `503` | Dependency outage atau `SYSTEM_BACKPRESSURE` saat job aktif atau backlog outbox mencapai ambang | Retry *submission* secara bounded, jangan buat key baru |

**Catatan:** beberapa error code yang spesifik baru didesain (misalnya `INVALID_API_KEY`, `UNSUPPORTED_PAYLOAD_VERSION`) dan harus dikonfirmasi di generated Swagger setelah implementasi.

## Retry pada *client submission* berbeda dari retry pada *worker execution*

- Jika POST tidak memberi respons karena timeout, status job **belum tentu gagal**; gunakan `Idempotency-Key` sama saat mengulangi request yang **identik**.
- Retry hanya `429`, `500`, `503`, atau network timeout yang dianggap transient sesuai policy client; cap attempt/delay dan gunakan jitter.
- Jangan mengulangi `401`, `403`, `400`, `409`, `413`, atau `422` tanpa memperbaiki sebabnya.
- HTTP `202` artinya diterima; **jangan** terus mengirim POST untuk mengejar `completed`. Poll `GET /jobs/:id` dengan interval wajar.
- Jika 429 memiliki `Retry-After`, hormati. Klien harus memiliki timeout dan circuit breaker sesuai kebutuhan.

## Kegagalan job worker

Kegagalan handler disimpan di `job_attempts`, lalu job menjadi `retrying` atau `dead`. Redelivery Redis, worker crash recovery, dan transaksi MySQL gagal **bukan** sama dengan error bisnis dari handler. Lihat [Tracking Jobs](../guides/tracking-jobs.md).

## Keamanan error

Jangan tampilkan raw provider response, SQL, Redis credentials, stacktrace, atau secret dalam error HTTP. Simpan error class dan request ID yang aman; investigasi via structured logs.

[Kembali ke indeks dokumentasi](../README.md)
