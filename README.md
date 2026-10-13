# KosPay

Web app (mobile-first, PWA) untuk pembayaran kos: penghuni membayar lewat QRIS statis/transfer, upload bukti, OCR mengisi data otomatis, pemilik mengonfirmasi manual, dan ada rekap bulanan.

**Status:** Fase 0 — rencana menunggu persetujuan. Lihat [docs/PLAN.md](docs/PLAN.md).

## Stack

Next.js (App Router) + TypeScript strict + Tailwind · Supabase (Postgres, Auth, Storage private, Realtime) · zod · Anthropic Messages API (OCR, server-only) · Cloudflare Workers via `@opennextjs/cloudflare`.

## Fase

0. Rencana + struktur folder + DDL final
1. Skema DB, migrasi, RLS, seed, setup Supabase client
2. Auth + halaman penghuni
3. Pipeline OCR + flags + notifikasi ke pemilik
4. Dashboard pemilik: antrean konfirmasi, lonceng realtime
5. Rekap, snapshot, export CSV, tombol wa.me, cron
6. PWA, hardening, deploy ke Cloudflare
