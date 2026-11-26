# Keamanan operasional

## Prinsip identitas

- API client memakai `X-API-Key`; ID client diperoleh **dari key yang terverifikasi**, bukan dari parameter request.
- Admin membuat client/key hanya via CLI/operator plane yang dilindungi. **Tidak ada endpoint publik untuk menerbitkan key.**
- Scope least-privilege: `jobs:create`, `jobs:read`, dan `jobs:requeue` **baru P1**.
- Server harus selalu melakukan query owner-scoped (`jobs.client_id`). Job client lain mengembalikan `404` generik.

## Penyimpanan dan transmisi secret

- Key plaintext hanya tampil saat penerbitan; MySQL menyimpan **HMAC-SHA256 digest** dengan `AUTH_KEY_PEPPER_B64` rahasia terpisah.
- Gunakan secret manager/env injection, bukan commit ke Git, issue, atau client log. **Catatan contoh cURL:** interpolasi key pada argumen `-H` dapat terlihat di daftar proses pada host bersama; itu hanya contoh development. Untuk automation/production, pilih klien yang menginjeksi header dari secret manager tanpa memaparkan nilainya sebagai process arguments.
- Pisahkan test dan live credentials, DB, dan pepper.
- TLS wajib untuk jaringan non-lokal. Restrict akses DB/Redis/MinIO/metrics/Swagger.
- Rotasi key dengan overlap terbatas; compromise → revoke segera.

## Batasan yang aman

- Job payload maksimum awal 256 KiB; file besar disimpan di object storage.
- Client **tidak** mengirim arbitrary endpoint URL, SQL, script, atau executable code pada MVP.
- Provider external dipilih dari konfigurasi server dan dipanggil dengan timeout, error classification, serta idempotency jika side effect penting.
- `payment.capture` hanyalah **mock** dan tidak boleh dipakai untuk menangani dana nyata pada MVP.

## Logging dan data protection

Jangan menyimpan raw API key, pepper, digest, password, external auth header, payload PII, stack trace rahasia, atau raw SQL exception dalam log. Gunakan field seperti `client_id`, `api_key_id`, `request_id`, `job_id`, `error_code`. Data retention automated adalah **P1**; hingga tersedia, operator harus mengatur backup/akses/retention secara manual.

## Tindakan insiden credential bocor

1. Revoke key lewat admin CLI atau prosedur operator yang aman.
2. Terbitkan key baru untuk client sama; deploy lewat secret manager.
3. Audit request log yang sudah teredaksi berdasarkan `api_key_id`/client/time window.
4. Evaluasi apakah user tersebut bisa membaca/membuat jobs yang tidak seharusnya.
5. Jika pepper bocor, desain proses rotate pepper dan reissue keys dengan migrasi terencana.

## Security review sebelum release

Test missing/revoked/expired key, insufficient scope, cross-client `404`, client-scoped idempotency, body size limits, redaction, dan SSRF protection via server-defined targets. Non-MVP admin UI/RBAC/multi-tenancy lanjutan tidak boleh diasumsikan sudah tersedia.

[Kembali ke indeks dokumentasi](../README.md)
