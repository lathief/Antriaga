# Template task Antriaga

Salin blok di bawah untuk setiap task. Satu task memuat satu perilaku yang dapat dites. Rujukan: [blueprint §37.3](../../SYSTEM_DESIGN.md) dan [rencana implementasi](implementation-plan.md).

```markdown
# Task: <nama spesifik>
- Milestone / issue: <M1/M2/M3/M4, ticket>
- Goal: <satu perilaku yang harus bekerja>
- Referensi desain: <§X.Y, §Z>
- In scope: <file/package>
- Out of scope: <jelas>
- Invariants: <DB commit, idempotency, auth, dll>
- Acceptance tests: <nama test, edge cases>
- Migration/API/Swagger impact: <none / rincian>
- Verification commands: <perintah konkret>
- Risk and rollback: <singkat>
```

Setelah selesai, catat hasil di [worklog](worklog.md) dengan format di sana. Tandai setiap pemeriksaan sebagai `LULUS`, `GAGAL`, atau `BELUM DIJALANKAN` sesuai bukti.
