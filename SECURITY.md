# Security Policy — Antriaga

> **Status:** kebijakan awal untuk repository open-source yang direncanakan. Sebelum rilis publik, maintainer harus mengonfirmasi saluran pelaporan privat yang benar dan menambahkan versi yang benar-benar didukung.

## Supported versions

Belum ada versi rilis resmi yang diverifikasi oleh paket dokumentasi ini. Jangan menganggap snapshot blueprint ini didukung sebagai layanan production.

## Reporting a vulnerability

**Jangan** publikasikan API key, exploit dengan data nyata, credential, atau PoC berisi informasi sensitif dalam issue publik.

Setelah GitHub repository dan fitur **Private Vulnerability Reporting** diaktifkan, gunakan kanal tersebut untuk laporan keamanan. Jika kanal privat belum tersedia, maintainer harus menyediakan alamat/kontak privat yang terverifikasi sebelum meminta laporan dari publik. Paket ini tidak mengarang alamat security contact maupun menjanjikan SLA respons.

Sertakan langkah reproduksi minimal, dampak, affected version/commit, dan logs yang telah diredaksi. Jangan menjalankan pengujian destruktif terhadap deployment pihak lain.

## Security boundaries

Antriaga mengandalkan client-scoped API keys, TLS, MySQL ownership checks, secret redaction, server-configured external targets, dan bounded retries. Baca [Security Operations](docs/operations/security.md) untuk mitigasi penggunaan. Tidak ada janji exactly-once atau keamanan payment production pada MVP.
