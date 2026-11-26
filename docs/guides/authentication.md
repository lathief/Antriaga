# Autentikasi dan pengelolaan API key

Antriaga memakai **service-to-service API key**, bukan login user atau OAuth. Setiap aplikasi mempunyai `client_id` unik (server-generated), `client_code` yang bisa dibaca manusia, dan satu atau beberapa API key dengan scopes yang dibatasi.

## Siapa melakukan apa?

| Aktor | Kewenangan |
|---|---|
| Administrator/infrastructure operator | Create/disable client; issue, rotate, revoke keys via admin CLI |
| Client application | Mengirim `X-API-Key` saat request, hanya memperoleh scope yang diberikan |
| Worker/Redis | **Tidak** menerima API key client di payload maupun stream |

## Memberikan API key ke aplikasi

Contoh perintah admin **yang direncanakan** (belum tentu tersedia hingga `cmd/admin` diimplementasikan):

```bash
antriaga-admin clients create --code notification-service --name "Notification Service"
antriaga-admin keys create --client notification-service --label primary \
  --scopes jobs:create,jobs:read --expires-in 90d
```

Key berpola `ant_test_<key-id-uuid>_<random-secret>` untuk environment test atau `ant_live_...` untuk live. Teks tersebut ilustratif; jangan menyalin literal `<...>` sebagai credential. Secret random dibuat kriptografis, disimpan hanya sebagai keyed digest di DB, dan **ditampilkan sekali** ketika dibuat.

**Gunakan header:**

```http
X-API-Key: <API_KEY_DARI_SECRET_MANAGER>
```

Hindari query string, JSON body, logging header, screenshot, atau menyimpan plaintext API key ke source control.

## Scopes MVP

| Scope | Aksi |
|---|---|
| `jobs:create` | `POST /api/v1/jobs` |
| `jobs:read` | `GET /api/v1/jobs`, `GET /api/v1/jobs/:id`, `GET /api/v1/jobs/:id/attempts` |
| `jobs:requeue` | **P1 saja** untuk manual requeue yang diaudit |

Semua data job dibatasi pada `client_id` yang diturunkan dari key; client tidak bisa mengirim `client_id` lain untuk melewati isolasi. Job milik client lain harus tampak sebagai `404` generik.

## Rotasi dan pencabutan

1. Operator membuat key baru untuk **client yang sama** (atau menjalankan `keys rotate`).
2. Deploy key baru lewat secret manager, verifikasi pemanggilan API.
3. Cabut key lama setelah masa overlap yang ditetapkan.
4. Jika key bocor, **revoke segera tanpa overlap**, lalu kirim key baru.

Contoh CLI rencana:

```bash
antriaga-admin keys list --client notification-service
antriaga-admin keys rotate --key-id <KEY_ID> --overlap 24h
antriaga-admin keys revoke --key-id <KEY_ID> --reason compromised
```

`keys list` hanya mengembalikan metadata; **plaintext tidak dapat ditampilkan lagi**. Jika hilang, buat key baru.

## Failure behavior

| Status | Makna |
|---|---|
| `401 INVALID_API_KEY` | Tidak ada key, key salah/revoked/expired, atau client disabled (pesan generik) |
| `403 INSUFFICIENT_SCOPE` | Credential valid, tetapi action tidak diizinkan |
| `404` | ID job tidak ada **atau bukan milik client** |
| `429` | Terkena per-client rate/admission limit |
| `503` | Auth DB tidak tersedia / service degraded, bukan bukti key invalid |

Semua request ke `/api/v1/*` membutuhkan client key. Health endpoints terbuka secara terkontrol; `/metrics` dan `/swagger/*` harus dibatasi pada network operator.

## Keamanan operasional

Hanya gunakan HTTPS pada jaringan non-lokal. Gunakan key terpisah antar environment/client, lakukan rotation terencana, batasi scopes, dan audit kegagalan autentikasi **tanpa** mencatat material key. Lihat [Security](../operations/security.md).

[Kembali ke indeks dokumentasi](../README.md)
