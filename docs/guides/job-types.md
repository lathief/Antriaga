# Jenis job MVP

> Keempat job type adalah **kontrak payload yang direncanakan**, bukan generic job executor. Server memetakan type ke queue. Pada MVP semua provider adalah mock/local demo, kecuali MinIO sebagai object storage untuk hasil report.

| Job Type | Queue | Purpose | Downstream |
|---|---|---|---|
| `email.send` | `email` | Kirim email asinkron | Mock Mailer |
| `webhook.deliver` | `webhook` | Kirim event HTTP yang sudah disepakati | Mock Webhook Receiver |
| `payment.capture` | `payment` | Simulasi payment capture idempotent | **Mock Payment Gateway saja** |
| `report.generate` | `report` | Buat CSV dan simpan object key | Worker + MinIO |

Semua contoh ditaruh sebagai body untuk `POST /api/v1/jobs` dengan `X-API-Key` dan scope `jobs:create`.

## 1. `email.send`

```json
{
  "type": "email.send", "payload_version": 1,
  "payload": {
    "to": "sample@example.com",
    "subject": "Welcome",
    "template": "welcome"
  },
  "max_attempts": 5
}
```

Contoh perilaku mock: sukses (`202`), permanent (`400`), retryable (`429`, `503`), timeout. Template dan tujuan email disesuaikan dengan mock mailer yang di-deploy operator.

## 2. `webhook.deliver`

```json
{
  "type": "webhook.deliver", "payload_version": 1,
  "payload": {
    "event": "order.completed",
    "reference_id": "ORD-10021",
    "data": { "order_id": "ORD-10021", "total": 250000 }
  }
}
```

Destination webhook **bukan input URL bebas dari client**; target diset di environment/config server. Server menandatangani delivery sesuai kontrak mock; penerima harus melakukan idempotency/dedup untuk mengantisipasi retry.

## 3. `payment.capture` — MOCK ONLY

```json
{
  "type": "payment.capture", "payload_version": 1,
  "payload": {
    "payment_id": "PAY-10092",
    "order_id": "ORD-10021",
    "amount": 250000,
    "currency": "IDR"
  }
}
```

Key downstream mock berasal dari pasangan `client_id` dan `payment_id`. Retry dengan order, nominal, dan mata uang yang sama mendapatkan hasil awal. Perubahan pada salah satu field itu dengan key yang sama memberi `409 PAYMENT_IDEMPOTENCY_CONFLICT` dari mock dan menutup attempt sebagai gagal permanen tanpa capture tambahan. Handler v1 hanya memakai mock gateway.

## 4. `report.generate`

```json
{
  "type": "report.generate", "payload_version": 1,
  "payload": {
    "report_type": "monthly_sales",
    "period": "2026-10",
    "format": "csv"
  }
}
```

Server membuat key privat per client, job, dan attempt. Payload v1 menolak `output_key`. Hasil job menyimpan metadata file dari attempt yang selesai dengan lease sah. Jangan mengirim file bytes besar dalam payload; default maksimum body 256 KiB.

## Error classification ringkas

- Payload tidak valid / unsupported type/version → ditolak saat submit atau permanent fail bila ditemukan di worker.
- HTTP 429 dan 5xx upstream / network timeout → retryable sesuai policy.
- Error 4xx selain 429 biasanya permanent, dengan pengecualian sesuai contract provider.
- Setiap job punya limit attempts; setelah habis masuk `dead`.

Selengkapnya: [Tracking Jobs](tracking-jobs.md), [API Reference](../reference/api.md).

[Kembali ke indeks dokumentasi](../README.md)
