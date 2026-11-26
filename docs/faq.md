# Pertanyaan umum

## Apakah Antriaga sudah bisa dijalankan?

Berkas dalam bundle ini adalah dokumentasi **berbasis blueprint v4**, bukan bukti implementation/release sudah tersedia. Ikuti rilis repository nyata setelah binary, migrations, Compose, mock services, dan Swagger sudah diimplementasikan dan diuji.

## Apakah `202 Accepted` berarti job sukses?

Tidak. `202` berarti job tersimpan dan akan diproses secara asinkron. Periksa status dengan `GET /api/v1/jobs/:id`.

## Bagaimana saya mendapatkan API key?

Operator membuat client unik dan API key lewat admin CLI yang dilindungi. Client tidak dapat generate key sendiri melalui public API.

## Apakah saya bisa memakai satu key untuk semua aplikasi?

Secara desain sebaiknya **tidak**. Setiap service memakai client unik dan key dengan scope paling kecil. Ini mempermudah isolasi, rotasi, audit, dan revokasi.

## Apakah Antriaga menjamin eksekusi exactly once?

Tidak. Delivery/processing adalah **at-least-once**. External system harus menangani idempotency pada side effect penting.

## Mengapa saya mendapat `404` padahal job ID benar?

Job mungkin milik client lain; Antriaga sengaja tidak membedakan "tidak ditemukan" dengan "bukan milik client" untuk mencegah kebocoran informasi.

## Apakah saya bisa mengirim URL webhook bebas?

Tidak pada MVP. URL target ditetapkan oleh operator melalui konfigurasi server.

## Apakah `payment.capture` aman untuk pembayaran asli?

Tidak. MVP secara eksplisit menggunakan **mock payment gateway** untuk demonstrasi idempotency dan retry.

## Apa perbedaan retry dan requeue manual?

Retry otomatis terjadi ketika handler gagal secara transient dan attempts tersisa. Requeue manual job `dead` adalah fitur **P1** yang harus diaudit dan dibatasi scope.

## Mengapa job `queued` lama?

Bisa karena outbox pending, Redis/dispatcher unavailable, worker slots penuh, atau recovery. Lihat [Troubleshooting](operations/troubleshooting.md).

## Bisakah saya mencari timeline event job dari API?

Endpoint durable `/api/v1/jobs/:id/events` adalah **P2**. Pada MVP periksa `job_attempts` dan structured logs berdasarkan `job_id`.

## Apakah tersedia SDK resmi untuk Go?

Belum didesain sebagai MVP. Ada [contoh integrasi menggunakan net/http](examples/go-client.md) yang dapat dijadikan starting point.

[Kembali ke indeks dokumentasi](README.md)
