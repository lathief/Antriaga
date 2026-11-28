# Referensi API HTTP

**Base URL lokal:** `http://localhost:8080`  
**Prefix:** `/api/v1`  
**Headers:** `X-API-Key` (wajib), `Content-Type: application/json` untuk POST; `Idempotency-Key` opsional.

> Ini adalah kontrak API dari blueprint v4, **belum generated API specification dari server yang berjalan**. Saat implementasi siap, *source of truth* machine-readable adalah output Swaggo yang dibangun dari kode. Swagger UI opsional pada `/swagger/index.html` dan harus dibatasi untuk akses operator.

## Endpoint MVP

| Method | Endpoint | Required scope | Response |
|---|---|---|---|
| `POST` | `/api/v1/jobs` | `jobs:create` | `202 Accepted` |
| `GET` | `/api/v1/jobs/:id` | `jobs:read` | `200 OK` atau `404` |
| `GET` | `/api/v1/jobs` | `jobs:read` | `200 OK` |
| `GET` | `/api/v1/jobs/:id/attempts` | `jobs:read` | `200 OK` atau `404` |
| `GET` | `/health/live` | Tidak perlu client key | `200 OK` (jika proses hidup) |
| `GET` | `/health/ready` | Tidak perlu client key | `200` / `503` |
| `GET` | `/metrics` | Internal network policy | `200 OK` |
| `GET` | `/swagger/*` | Internal network policy | Swagger UI jika enabled |

`GET /api/v1/jobs/:id/events` (P2), `POST /api/v1/jobs/:id/requeue` (P1), cancel dan queue control **tidak termasuk MVP**.

**Catatan rencana (belum diimplementasikan):**
- Kolom `priority` ada di database tetapi **tidak dihormati** dan **tidak diekspos** pada MVP.
- Respons `202` direncanakan memuat header `Location` ke job yang dibuat, dan replay idempoten diberi penanda yang membedakannya dari job baru. Bentuk pastinya ditetapkan di Task 9 ([rencana](../ai/implementation-plan.md)) sebelum handler ditulis.

## `POST /api/v1/jobs`

**Headers:** `X-API-Key`, `Content-Type: application/json`; `Idempotency-Key` opsional, 1–150 byte ASCII tercetak tanpa spasi, dibandingkan case-sensitive dalam satu client. Lihat [kontrak deduplikasi](../guides/submitting-jobs.md#idempotency-key-untuk-submit).

**Body umum:**

```json
{
  "type": "email.send",
  "payload_version": 1,
  "payload": { "to":"sample@example.com", "subject":"Welcome", "template":"welcome" },
  "max_attempts": 5
}
```

Field yang didukung kontrak: `type` (required), `payload` (required), `payload_version` (default 1), `max_attempts` (default 5), `scheduled_at` (opsional, RFC3339), dan `queue` (opsional tetapi harus cocok dengan type; lebih baik omit). Payload `report.generate` menyerahkan penentuan object key kepada server. Token angka `webhook.deliver.data` dipertahankan melalui penyimpanan dan pengiriman. Detail masing-masing payload pada [Job Types](../guides/job-types.md).

**Respons ilustratif** `202 Accepted`:

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

HTTP `202` menandakan job dan outbox intent (untuk job langsung) sudah tersimpan secara durable. Saat Redis putus, API tetap memproses submit lewat MySQL sesuai kebijakan kapasitas. Job `scheduled`, `queued`, `retrying`, dan `processing` ikut dihitung. Kuota client menghasilkan `429`; ambang global menghasilkan `503`. Submit ulang dengan key yang sudah commit mengikuti kontrak dedup jika limiter lokal masih mengizinkan.

## `GET /api/v1/jobs/:id`

Mengembalikan satu job **milik client terautentikasi**. Status dapat berupa `scheduled`, `queued`, `processing`, `retrying`, `completed`, atau `dead`. Detail response shape selain field inti di atas masih harus dikunci lewat DTO dan Swaggo implementasi. Unknown ID dan ID milik client lain **sama-sama `404`**.

## `GET /api/v1/jobs`

Filter yang direncanakan: `queue`, `status`, `page`, `limit`. `page>=1`; `limit` dari 1 sampai 100. Server **selalu** menyaring berdasarkan `client_id` dari API key.

```http
GET /api/v1/jobs?queue=email&status=dead&page=1&limit=20
```

**Respons ilustratif:**

```json
{"data": [], "meta": {"page":1, "limit":20, "total":124}}
```

## `GET /api/v1/jobs/:id/attempts`

Membaca histori percobaan eksekusi milik job client tersebut; `attempt_number`, `status`, timestamp, dan redacted error metadata diambil dari `job_attempts`. **Envelope/field respons detail belum difinalisasi di blueprint v4**; definisikan serta tambahkan contoh valid setelah DTO implemented. Client jangan menebak struktur sebelum versi rilis terbit.

## Error response

**Contoh envelope:**

```json
{
  "error": {
    "code": "UNSUPPORTED_JOB_TYPE",
    "message": "job type is not supported",
    "request_id": "req-019a9e11"
  }
}
```

Lihat [Errors](errors.md) untuk status code dan kebijakan retry.

## Keterbatasan dan kompatibilitas

- API versi pertama adalah `/api/v1`; breaking changes sebaiknya memerlukan versi baru atau kebijakan transisi eksplisit.
- Business `payload_version` disimpan di job; berbeda dari `schema_version` pada pesan Redis internal.
- Job side effects **at-least-once**; downstream tetap harus idempotent.
- Headers/request/response tidak boleh mengekspos secret atau payload sensitif dalam log.
- Swagger security scheme memakai header `X-API-Key`; contoh error dan scopes harus konsisten dengan middleware runtime.

[Kembali ke indeks dokumentasi](../README.md)
