# Produk: pengguna, masalah, dan metrik sukses

> **Status:** satu halaman ringkas. Semua metrik adalah **target, belum diukur**. Tidak ada implementasi yang dijalankan di snapshot ini.

## Pengguna

Developer layanan pemanggil (misalnya layanan notifikasi atau layanan pembayaran) yang butuh menjalankan pekerjaan latar belakang secara andal tanpa mengoperasikan antrean sendiri. Pengguna lain adalah operator yang menerbitkan API key dan memantau sistem.

## Masalah yang diselesaikan

- Request API tidak menunggu pengiriman email atau webhook.
- Job yang sudah diterima tidak hilang saat proses atau Redis restart.
- Kegagalan sementara dicoba ulang dengan batas, dan job yang gagal permanen tetap terlihat.
- Setiap client hanya melihat job miliknya.

Rincian ada di [blueprint §1.1](../SYSTEM_DESIGN.md).

## Metrik sukses (target, belum diukur)

| Metrik | Target | Cara mengukur |
|---|---|---|
| Waktu sampai job pertama berhasil | Hitungan menit dari clone, tiga perintah | Jalankan [quickstart](getting-started/quickstart.md) dari lingkungan bersih dan catat waktunya |
| Job diterima yang hilang setelah restart atau crash | Nol | Demo crash (deliverable a di [rencana](ai/implementation-plan.md#deliverable-portofolio-direncanakan)) dan tes pemulihan |
| p95 waktu tunggu antrean | Ditetapkan setelah baseline | Load generator (deliverable b) dengan profil beban dan jumlah worker yang dicatat |

Angka target numerik ditentukan setelah ada pengukuran. Jangan mengisinya dengan perkiraan.

## Yang sengaja belum ada

Fitur berikut belum ada pada MVP dan masuk [backlog](post-mvp.md):

- Callback saat job selesai (pengguna saat ini hanya bisa polling).
- Webhook terdaftar per client (target webhook hanya dari konfigurasi server).
- Prioritas job (kolom `priority` ada, tetapi tidak dihormati dan tidak diekspos).
- Cancel job dan pause queue.

[Kembali ke indeks dokumentasi](README.md)
