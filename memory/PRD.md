# PRD — Rahaza Travel ERP (lanjutan repo aklakskeehe/travel)

## Problem Statement Asli (sesi Jun 2026)
Lanjutkan development repo https://github.com/aklakskeehe/travel (ERP travel FARM stack, bahasa Indonesia).
Permintaan: integrasi WhatsApp NYATA menggunakan platform **OpenWA (open-wa/wa-automate)** —
analisis kelayakan dulu, lingkup full (kirim + terima → Inbox CRM, auto-reply, lead otomatis).
User paham risiko unofficial (banned). Nomor WA belum siap; scan QR menyusul.

## Status Workspace
- /app masih TEMPLATE KOSONG. Repo asli di-clone sementara ke /tmp/travel untuk analisis (belum di-setup ke /app).
- Langkah berikutnya bila user konfirmasi: clone repo ke /app, install deps, seed, gate HIJAU, lalu implement provider openwa.

## Hasil Analisis Kelayakan OpenWA (sesi ini, TERVERIFIKASI dengan uji nyata di container)
- ❌ v4.76.0 (stable): GAGAL total di WA Web versi sekarang — hang di cek `window.Debug` (issue GitHub #3346), sudah diuji 4x variasi flag (use-chrome, custom UA, patch timeout 180s). JANGAN pakai v4.
- ✅ v5.0.0-alpha (`npx @open-wa/wa-automate@alpha`): BERHASIL boot di container ini.
  - Chrome/Chromium 151 tersedia (/usr/bin/google-chrome), Node v20.20.2 OK (warning engine non-fatal).
  - Easy API jalan di port pilihan (-p 8002), `--api-key` untuk proteksi.
  - `GET /health` → state sesi; `GET /qr` → JSON berisi data QR (render QR di UI admin).
  - `qr_code_generated` terkonfirmasi di log (QR siap discan).
  - Kirim: `POST /api/messages/sendText` (+ sendImage, sendFile, dsb — 123 endpoint, swagger di /meta/swagger.json).
  - Inbound: flag `--webhook <url>` (POST event ke backend) + SSE `GET /api/events` (auth x-api-key/api_key).
  - Session persist via userDataDir `_IGNORE_<session>` → HARUS disimpan di /app agar tahan restart.

## Pemetaan Fitur WA di Repo → OpenWA (semua lewat funnel tunggal services/whatsapp.py)
Arsitektur repo sudah provider-agnostic (mock | meta_cloud) → tinggal tambah provider `openwa`:
1. Test-send (/api/wa/test-send) → sendText ✅
2. Automation Engine (~20 template rule: booking confirm/cancel/reschedule, driver assigned, reminder, payment, invoice due, win-back, subcharter partner) → sendText ✅
3. Balas dari Inbox CRM → sendText ✅
4. Campaigns (per-recipient) → sendText ✅ (perlu throttle anti-ban)
5. Sequences/drip → sendText ✅
6. Review request pasca-trip (scheduler) → sendText ✅
7. Auto-reply & away-reply → inbound webhook + sendText ✅
8. Broadcasts → masih simulasi murni di repo; perlu di-wire ke send_wa loop + throttle ✅
9. Inbound → lead otomatis + conversation + emit event: adapter payload openwa → handle_inbound ✅
10. ❌ TIDAK didukung OpenWA: atribusi iklan CTWA (`ctwa_clid`/referral) — itu fitur khusus Meta Cloud API; tetap tersedia jika kelak pindah provider meta_cloud.
11. Template Meta (approval WABA) tidak diperlukan di OpenWA — semua pesan teks bebas, tanpa batasan session-window 24 jam.

## Risiko & Catatan
- v5 masih ALPHA → pin versi persis saat implementasi.
- Unofficial → risiko banned; pakai nomor khusus bisnis; throttle blast.
- Restart pod → sesi Chrome bisa perlu scan ulang bila data dir hilang (mitigasi: simpan di /app).
- License key open-wa opsional (fitur premium tertentu); teks dasar gratis.
- WhatsApp real-send, Meta/Google Ads, GA4 di repo masih MOCKED.

## Rencana Implementasi (belum dikerjakan, menunggu konfirmasi user)
1. Setup repo /app (clone, deps, seed, gate HIJAU 46/46).
2. Jalankan OpenWA v5 alpha sebagai proses supervisor (port internal, api-key di .env, session dir /app/openwa-session).
3. `OpenWaProvider` di services/whatsapp.py (send_text via HTTP) + provider `openwa` di config/router.
4. Route webhook internal `/api/wa/openwa-webhook` → adapter → handle_inbound (lead otomatis, auto-reply, Inbox).
5. UI Integrasi API: pilih provider openwa, tampilkan status koneksi + QR code untuk discan, tombol test-send.
6. Wire Broadcasts ke pengiriman nyata + throttling.

## Kredensial
- memory/test_credentials.md repo asli: semua akun demo password `demo12345` (akan berlaku setelah repo di-setup di /app).
