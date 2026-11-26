# Berkontribusi ke Antriaga

> **Status:** proyek masih tahap dokumentasi dan perencanaan. Belum ada kode runtime, jadi perintah build dan tes di bawah adalah target yang direncanakan.

## Sebelum mulai

1. Baca [README](README.md), [desain sistem](SYSTEM_DESIGN.md), dan [AGENTS.md](AGENTS.md). `AGENTS.md` berlaku untuk kontributor manusia maupun AI agent.
2. Pilih satu task dari [rencana implementasi](docs/ai/implementation-plan.md). Gunakan [template task](docs/ai/task-template.md).
3. Perubahan arsitektur memerlukan ADR di [`design/adr/`](design/adr/ADR-0000-template.md) dan persetujuan sebelum kode.

## Alur kerja

1. Buat branch dari `master` dengan nama deskriptif, misalnya `feat/create-job` atau `docs/fix-links`.
2. Tulis tes yang gagal dulu bila praktis, lalu implementasi minimum.
3. Satu perubahan menyentuh satu tanggung jawab. Hindari refactor, dependensi, atau skema di luar task.
4. Jalankan pemeriksaan (target, setelah Makefile atau `scripts/check.ps1` tersedia): `gofmt`, `go vet ./...`, `go test ./... -count=1`, `go test -race ./...`, tes integrasi, dan regenerasi Swaggo.
5. Perbarui dokumentasi pengguna, Swaggo, dan `CHANGELOG.md` bila kontrak berubah (lihat bagian pemeliharaan di `AGENTS.md`).
6. Di PR, laporkan perintah yang benar-benar dijalankan dengan hasil `LULUS`, `GAGAL`, atau `BELUM DIJALANKAN`.

## Keamanan

- Jangan commit API key, pepper, password, atau `.env`.
- Laporkan kerentanan lewat [SECURITY.md](SECURITY.md), bukan issue publik.
