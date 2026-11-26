# Panduan cepat: jalankan job pertama

> Alur berikut adalah **contoh untuk kontrak MVP yang direncanakan**. Memerlukan Antriaga runtime dan mock mailer yang benar-benar telah diimplementasikan.

## 1. Periksa kesiapan layanan

```bash
export ANTRIAGA_URL="http://localhost:8080"
curl --fail "$ANTRIAGA_URL/health/ready"
```

## 2. Peroleh API key dari administrator

Mintalah operator membuat client unik (misalnya `notification-service`) dan API key dengan scopes `jobs:create,jobs:read`. **Client tidak dapat membuat API key lewat endpoint publik.** Kunci hanya tampil sekali saat diterbitkan. Simpan melalui secret manager dan injeksikan ke proses sebagai environment variable:

```bash
# Environment variable ANTRIAGA_API_KEY harus sudah disuplai dari secret manager.
test -n "${ANTRIAGA_API_KEY:-}" || echo "API key belum tersedia"
```

Hindari menulis API key langsung dalam command shell / history atau log. Lihat [Authentication](../guides/authentication.md).

## 3. Kirim job email demo

```bash
curl --fail-with-body -sS -X POST "$ANTRIAGA_URL/api/v1/jobs" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  -H 'Idempotency-Key: demo-welcome-001' \
  --data '{
    "type": "email.send",
    "payload_version": 1,
    "payload": {
      "to": "sample@example.com",
      "subject": "Welcome",
      "template": "welcome"
    },
    "max_attempts": 5
  }'
```

**Contoh respons ilustratif `202 Accepted`:**

```json
{
  "data": {
    "id": "019a9e11-330b-7e1e-bb8a-119511778801",
    "queue": "email",
    "type": "email.send",
    "payload_version": 1,
    "status": "queued",
    "attempts": 0,
    "max_attempts": 5,
    "created_at": "2026-11-18T05:00:00Z"
  }
}
```

`202` berarti job **diterima dan dipersist** (termasuk outbox untuk immediate job), **bukan** email sudah dikirim.

## 4. Lihat status dan riwayat

```bash
export JOB_ID="019a9e11-330b-7e1e-bb8a-119511778801" # Ganti dari respons POST
curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "$ANTRIAGA_URL/api/v1/jobs/$JOB_ID"

curl --fail-with-body -sS \
  -H "X-API-Key: ${ANTRIAGA_API_KEY}" \
  "$ANTRIAGA_URL/api/v1/jobs/$JOB_ID/attempts"
```

Job dapat berada di status `retrying` saat menunggu percobaan ulang. Status akhir eksekusi adalah `completed` jika berhasil atau `dead` jika gagal permanen atau batas percobaan tercapai. Lihat [Tracking Jobs](../guides/tracking-jobs.md).

## 5. Langkah berikutnya

- [Job Types](../guides/job-types.md): contoh 4 payload MVP
- [Submit & Schedule](../guides/submitting-jobs.md): jadwal, idempotency, versioning
- [Errors](../reference/errors.md): cara menangani `401`, `403`, `409`, `429`, `503`
- [Go Client](../examples/go-client.md): integrasi dari aplikasi Go

[Kembali ke indeks dokumentasi](../README.md)
