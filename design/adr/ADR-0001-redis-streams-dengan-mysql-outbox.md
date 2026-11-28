# ADR-0001: Redis Streams dengan transactional outbox MySQL

- **Status:** diusulkan
- **Tanggal:** 28 November 2026
- **Rujukan desain:** [blueprint §23 (ADR-001 sampai ADR-003)](../../SYSTEM_DESIGN.md) dan [§9](../../SYSTEM_DESIGN.md)

## Konteks

Antriaga harus menerima job lewat HTTP tanpa kehilangan job yang sudah diterima, mendistribusikannya ke banyak worker, dan memulihkan diri saat worker atau Redis gagal. Proyek ini juga berfungsi sebagai portofolio untuk menunjukkan pola keandalan. Pilihan broker memengaruhi kompleksitas recovery dan jumlah dependensi.

## Keputusan

- **MySQL adalah sumber kebenaran** untuk job, attempt, dan outbox.
- **Redis Streams membawa notifikasi kesiapan job** ke worker lewat consumer group.
- **Transactional outbox** menjembatani commit MySQL dan `XADD` Redis, karena keduanya tidak bisa atomik.
- Delivery **at-least-once**. Duplikasi ditangani lewat fencing `enqueue_seq`, lease DB, dan idempotensi downstream.

## Konsekuensi

- Ada dua sistem yang harus dipulihkan setelah gangguan (MySQL dan Redis), sehingga recovery lebih rumit daripada satu sistem.
- Outbox menambah tabel dan proses dispatcher. Duplikat publish adalah hal normal dan harus ditahan oleh claim bersyarat.
- MySQL tetap titik tunggal kegagalan. Redis yang mati hanya menunda pengiriman.
- Setiap claim, heartbeat, dan finalisasi menulis ke MySQL. Batas throughput belum diukur.

## Alternatif

Kolom angka sengaja kosong sampai ada pengukuran. Jangan mengisinya dengan perkiraan.

| Opsi | Apa yang lebih sederhana | Apa yang hilang | Kapan lebih tepat | Angka |
|---|---|---|---|---|
| Antrean MySQL saja dengan `SELECT ... FOR UPDATE SKIP LOCKED` | Satu dependensi, satu sumber kebenaran, tanpa outbox dan dispatcher | Tidak ada consumer group atau PEL bawaan. Beban polling dan tulis semua di MySQL. Fan-out ke banyak worker terbatas oleh polling | Beban sedang, tim kecil, dan satu database sudah cukup | Diisi setelah pengukuran |
| Redis saja (Streams atau list) tanpa MySQL sebagai sumber kebenaran | Tanpa outbox dan tanpa dua sumber kebenaran | Durabilitas dan query riwayat bergantung pada persistensi Redis. Idempotensi per client dan histori attempt jadi sulit | Job boleh hilang saat Redis gagal dan riwayat tidak penting | Diisi setelah pengukuran |
| Kafka atau NATS JetStream | Replikasi, retensi, dan consumer group matang | Operasi lebih berat. Retry bertahap, scheduling, dan status job per client tetap harus dibangun | Throughput tinggi, banyak konsumen berbeda, atau event streaming antar tim | Diisi setelah pengukuran |
| Library siap pakai (Asynq, River) | Retry, scheduling, dan UI sudah ada dan teruji | Tidak mendemonstrasikan pola keandalan yang menjadi tujuan proyek ini | Produk nyata yang butuh job queue tanpa membangun sendiri | Tidak berlaku |

## Rencana pembuktian

Ukur varian antrean MySQL saja secara singkat (job per detik dan p95 waktu tunggu pada 1 dan beberapa worker) dan bandingkan dengan jalur Redis Streams. Tujuannya menunjukkan di mana varian sederhana mulai tidak cukup. Hasil dan profil bebannya dicatat di [worklog](../../docs/ai/worklog.md), lalu kolom angka di atas diisi. Rencana ini belum dijalankan.

Keputusan terkait: [catatan keputusan](../../docs/ai/decisions.md).
