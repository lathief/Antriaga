# Verifikasi implementasi Antriaga

> Perintah berikut adalah target blueprint. Jalankan setelah Go module, package, Compose, dan Makefile terkait tersedia. Tandai pemeriksaan yang belum bisa dilakukan di [worklog](worklog.md).

## Jalur verifikasi

| Perubahan | Pemeriksaan minimum |
|---|---|
| Go lokal | `gofmt` pada berkas yang berubah, `go vet ./...`, `go test ./... -count=1`, `go build ./cmd/...` |
| Transaksi atau Redis | `go test -tags=integration ./tests/integration/... -count=1` dengan MySQL/Redis terisolasi sesuai kasus dan migrasi pada database test |
| Worker dan concurrency | `go test -race ./... -count=1` pada platform yang mendukung, ditambah tes race integrasi memakai lock/barrier dan waktu MySQL |
| Endpoint dan rilis | Regenerasi Swaggo dengan CLI yang dipin; periksa diff kontrak. Jalankan [panduan cepat](../getting-started/quickstart.md) dari lingkungan bersih dan cocokkan hasil dengan §22 blueprint |

Target Makefile dan CI mengikuti [blueprint §§19–20 dan §42](../../SYSTEM_DESIGN.md).

## Matriks kegagalan

| Kasus | Cara memicu | Bukti lulus |
|---|---|---|
| Bootstrap MySQL dan migrasi | Start dari volume kosong dengan satu migrasi valid; ulangi dengan migrasi gagal. Uji startup API saat Redis belum ada | MySQL sehat sebelum migrasi. API baru berjalan setelah `migrate` sukses, tanpa bergantung pada Redis. Kegagalan terlihat pada status dan log `migrate` |
| Outbox baru langsung due | Buat job immediate, release scheduled/retrying, dan jalankan recovery pada fixture terpisah | Outbox baru `pending`, `available_at` berasal dari waktu transaksi MySQL, dan claim tersedia setelah commit |
| Commit job/outbox (ANT-03) | Gagalkan insert outbox pada transaksi create | Use case gagal, job dan outbox tidak tersimpan, dan tidak ada log sukses |
| Kontrak HTTP create (setelah ANT-03) | Ulangi kegagalan transaksi lewat route setelah auth dan HTTP tersedia | Respons error sesuai kontrak, tanpa `202` |
| Dedup serentak | Kirim request sama dari satu client, lalu ulang key dengan request canonical berbeda | Satu job/outbox; key dengan request berbeda menghasilkan conflict; client lain tetap terisolasi. Jalankan [fixture canonical §27.7](../examples/idempotency-cases.json) termasuk byte key yang beda kapital dan retry terjadwal setelah due |
| Redis sementara tidak tersedia | Matikan Redis, kirim job langsung dan terjadwal, ulangi key yang sudah commit, lalu pulihkan Redis | Limiter lokal tetap berlaku. Saat kapasitas tersedia, job tersimpan dan dispatcher akhirnya publish. Outage sendiri tidak menolak submit |
| Kuota job aktif | Isi job `scheduled` dan `retrying` tanpa menambah outbox, lalu kirim request serentak pada batas client atau global | Kuota client memberi `429`, ambang global memberi `503`. Replay dengan key sama tetap membaca job asal jika rate limit mengizinkan |
| Idempotensi payment mock | Dua client memakai `payment_id` sama. Ulangi tuple identik dan ubah order, amount, atau currency dengan key yang sama | Masing-masing client punya hasil tersendiri. Retry identik menghasilkan satu capture per client. Tuple berbeda memberi conflict permanen |
| Presisi webhook | Submit `9007199254740993` dan `0.123456789012345678901` dalam `data`, baca dari MySQL, lalu capture body outbound pada mock | Token angka tetap identik, termasuk pada retry. Bandingkan token JSON secara langsung tanpa `float64` |
| Trim Redis P1 | Stop worker dengan backlog belum delivered dan PEL kosong. Uji juga entri pending serta group yang dibentuk ulang | Hanya prefix settled yang terhapus. Backlog belum delivered dan PEL tetap dapat diproses setelah worker aktif |
| Isolasi report | Dua client meminta laporan periode dan tipe yang sama. Tahan attempt lama sampai lease hilang, lalu biarkan ia menulis setelah attempt pemenang selesai | Key berbeda per client, job, dan attempt. Result memperlihatkan hanya file attempt pemenang. Cleanup tidak menghapus file yang masih mungkin direferensikan |
| Dispatcher kehilangan lease | A tertahan hingga expiry; B claim dengan token baru; A mencoba finalisasi sukses dan gagal | A tidak mengubah row. Jika terjadi dua XADD, hanya satu worker claim pada generation itu |
| Pesan Redis v1 | Publish melalui dispatcher lalu baca dengan decoder worker; ulangi dengan field hilang dan versi unsupported | Semua field v1 terbaca; pesan invalid tetap pending tanpa claim atau ACK |
| Rekonsiliasi broker | Uji backlog dengan pesan masih ada, pesan hilang, Redis error, dan recheck generation setelah lock | Hanya kehilangan terkonfirmasi pada state yang masih cocok membuat satu generation/outbox baru. Tidak menaikkan attempt |
| Group redis hilang saat runtime | Hilangkan group pada Redis test dengan stream, worker, dan job `queued` tetap hidup | Consumer membentuk ulang group mulai dari `0` tanpa restart, menangani `BUSYGROUP` tanpa reset, dan job diproses |
| Perbaikan envelope rusak | Gunakan envelope rusak dengan identitas publish yang cocok, lalu paksa rollback atau ulangi sesudah commit | Satu replacement generation tanpa attempt baru, ACK setelah commit, rollback tanpa ACK. Versi unsupported menunggu consumer kompatibel |
| Worker berhenti sebelum claim | Hentikan worker setelah pesan masuk PEL | Satu generation/outbox baru, jumlah attempt tetap, dan pesan lama di-ACK setelah commit |
| Finalisasi worker terlambat | Biarkan lease habis sebelum reaper, dengan token lama masih cocok | Heartbeat/finalisasi ditolak; state dan riwayat attempt tetap; tanpa ACK |
| ACK gagal setelah retry commit | Gagalkan ACK setelah `retrying` tersimpan, lalu jalankan reaper sebelum due | Pesan lama dibersihkan, jadwal dan counter tetap, scheduler membuat satu generation saat due |
| Efek eksternal sebelum crash | Mock menerima efek sebelum worker commit hasil, lalu recovery | Pengiriman ulang membawa key downstream yang sama. Pada mock idempotent, efek tercatat satu kali |

Aturan waktu, batas expiry, dan recheck setelah lock mengikuti blueprint §§9 dan 29. Rekam hasil database, Redis, dan mock sebagai bukti. Log terstruktur membantu diagnosis dan korelasi.
