# KosPay — Fase 0: Rencana

> Status: menunggu persetujuan. Pertanyaan terbuka ada di bagian 5.

## 1. Keputusan teknis

| # | Keputusan | Alasan |
|---|---|---|
| 1 | Next.js App Router versi stabil terbaru yang didukung `@opennextjs/cloudflare`, TypeScript strict, Tailwind, npm | Sesuai stack |
| 2 | Uang disimpan sebagai **integer rupiah**, bukan float/numeric | Pembandingan `amount_match` jadi persis |
| 3 | Semua logika tanggal memakai zona waktu **Asia/Jakarta** (`lib/dates.ts`) | Jatuh tempo, telat, dan rekap tidak bergeser karena UTC |
| 4 | **Membaca** data lewat klien Supabase milik user, jadi RLS yang menyaring. **Menulis** invoice/proof/notifikasi hanya dari server dengan service role, lewat `transition.ts` | Penghuni tidak bisa mengubah status invoice dengan cara apa pun |
| 5 | `transition.ts` = state machine murni (`nextStatus()`) ditambah satu RPC Postgres `apply_invoice_transition()`. RPC ini mengunci baris (`FOR UPDATE`), mengecek status asal, lalu menulis invoice + proof + `invoice_events` (audit) + notifikasi dalam **satu transaksi**. Hanya role `service_role` yang boleh mengeksekusinya | Atomik. Auto-konfirmasi dari mutasi bank nanti cukup memanggil fungsi yang sama dengan `actor_type='integration'` |
| 6 | **OCR tidak memblokir upload**: bukti disimpan dulu (`ocr_status='processing'`, invoice jadi `reported`, notifikasi ke pemilik), lalu OCR berjalan di `after()` (di Workers memakai `waitUntil`). Timeout 15 detik. Bila gagal: `ocr_status='failed'` dan muncul flag "perlu cek manual" | Antrean pemilik ikut ter-update lewat Realtime saat OCR selesai |
| 7 | Model OCR diatur lewat env `OCR_MODEL`. Output diminta JSON saja, lalu `JSON.parse` + zod `safeParse`. Gagal di salah satu langkah dihitung "OCR gagal". Teks di gambar tidak pernah dipakai sebagai instruksi | Teks dalam gambar adalah data tidak dipercaya |
| 8 | Flags bernilai `'ok' \| 'fail' \| 'unknown' \| 'skip'`. Field yang tidak terbaca (null) menjadi `unknown`, bukan `ok`. Tombol "Konfirmasi semua" hanya mengambil proof yang semua flag-nya `ok`/`skip`, `ocr_status='done'`, dan `is_success_screen=true` | Nilai yang tidak terbaca tidak boleh dianggap lolos |
| 9 | Kedua bucket Storage **private** (`proofs`, `qris`) dan tidak punya policy untuk user. Server yang mengecek hak akses, lalu membuat signed URL berumur 120 detik. Path proof: `{property_id}/{invoice_id}/{uuid}.jpg` | Tidak ada akses langsung ke file |
| 10 | Test memakai Vitest. Test DB (RLS dan transisi) memakai **PGlite** (Postgres in-process, tanpa Docker) dengan stub `auth.uid()` dan role `authenticated` | Bisa jalan tanpa Docker |
| 11 | Rate limit disimpan di Postgres (`rate_limits` + `hit_rate_limit`): maksimal 5 upload per 10 menit per user | Penyimpanan in-memory tidak andal di Workers |
| 12 | Validasi upload di server: cek magic bytes (JPEG/PNG/WebP), maksimal 2 MB, SHA-256 dengan WebCrypto. Kompresi di klien memakai canvas (sisi terpanjang 1280px, JPEG q≈0.75, ~300 KB) | Sesuai spesifikasi, kompatibel dengan Workers |
| 13 | Cron: satu endpoint `/api/cron` dengan `Authorization: Bearer CRON_SECRET`. Idempotensi lewat tabel `cron_runs(job, run_key)` + `ON CONFLICT DO NOTHING` | Aman bila cron terpanggil ganda |
| 14 | Invoice periode M dibuat saat `hari ini ≥ due_date − invoice_lead_days` (default 7). `base_amount` diambil dari harga kamar saat itu. Hanya kamar aktif dengan penghuni aktif | |
| 15 | Snapshot dibuat pada `recap_day`. Bila `recap_day ≥ due_day`, yang direkap periode bulan berjalan; bila tidak, periode bulan lalu. Hari 29–31 di-clamp ke akhir bulan | Rekap selalu dibuat setelah jatuh tempo |
| 16 | "Telat" dihitung saat query (`due_date < today AND status <> 'paid'`). Cron hanya mengirim notifikasi "telat" sekali per invoice | |
| 17 | Titik ekstensi `lib/payments/sources/` berisi interface `PaymentSource` (kosong untuk saat ini) | Integrasi mutasi bank nanti tinggal ditambah |

Peran ditentukan dari data, tanpa kolom role. Pemilik: `properties.owner_id = auth.uid()`. Penghuni: `tenants.auth_user_id = auth.uid() AND is_active`.

## 2. Struktur folder

```
Kos/
├─ supabase/
│  ├─ config.toml
│  ├─ migrations/
│  │  ├─ 0001_schema.sql        # enum, tabel, index, constraint
│  │  ├─ 0002_functions.sql     # helper RLS, apply_invoice_transition, hit_rate_limit
│  │  ├─ 0003_rls.sql           # policy + grant
│  │  ├─ 0004_storage.sql       # bucket private
│  │  └─ 0005_realtime.sql      # publication notifications, payment_proofs
│  └─ seed.sql                  # 1 kosan, 10 kamar, invoice beragam status
├─ src/
│  ├─ app/
│  │  ├─ (auth)/masuk/
│  │  ├─ (penghuni)/tagihan/
│  │  ├─ (pemilik)/pemilik/{antrean,rekap/[period],pengaturan}/
│  │  ├─ api/proofs/route.ts
│  │  ├─ api/cron/route.ts
│  │  ├─ api/rekap/[period]/csv/route.ts
│  │  ├─ manifest.ts
│  │  └─ layout.tsx
│  ├─ components/{ui,tenant,owner}/
│  ├─ lib/
│  │  ├─ env.ts
│  │  ├─ supabase/{server,browser,admin,middleware}.ts
│  │  ├─ payments/{state-machine,transition,flags,invoice-generator}.ts
│  │  ├─ payments/sources/
│  │  ├─ ocr/{schema,prompt,parse,client}.ts
│  │  ├─ proofs/{validate-image,hash,submit}.ts
│  │  ├─ recap/{compute,csv,snapshot}.ts
│  │  ├─ notifications/{create,templates}.ts
│  │  ├─ cron/{jobs,run}.ts
│  │  ├─ whatsapp/link.ts
│  │  └─ rate-limit.ts, dates.ts, money.ts
│  ├─ client/image-compress.ts
│  ├─ types/database.ts
│  └─ middleware.ts
├─ tests/
│  ├─ unit/   (flags, state-machine, ocr-parse, recap, dates, wa-link)
│  └─ db/     (rls.test.ts, transition.test.ts — PGlite)
└─ .env.example  wrangler.jsonc  open-next.config.ts  vitest.config.ts
```

## 3. DDL final

```sql
-- ===== 0001_schema.sql =====
create extension if not exists pgcrypto;

create type invoice_status as enum ('unpaid','reported','paid');
create type proof_status   as enum ('pending','confirmed','rejected');
create type ocr_status     as enum ('processing','done','failed');
create type actor_type     as enum ('tenant','owner','system','integration');

create table properties (
  id                 uuid primary key default gen_random_uuid(),
  owner_id           uuid not null references auth.users(id) on delete restrict,
  name               text not null check (length(trim(name)) between 1 and 100),
  due_day            smallint not null default 5  check (due_day between 1 and 31),
  recap_day          smallint not null default 28 check (recap_day between 1 and 31),
  invoice_lead_days  smallint not null default 7  check (invoice_lead_days between 0 and 28),
  qris_image_path    text,
  bank_name          text check (length(bank_name) <= 50),
  account_last4      char(4) check (account_last4 ~ '^[0-9]{4}$'),
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now()
);
create index on properties (owner_id);

create table rooms (
  id             uuid primary key default gen_random_uuid(),
  property_id    uuid not null references properties(id) on delete cascade,
  name           text not null check (length(trim(name)) between 1 and 50),
  monthly_price  integer not null check (monthly_price between 1 and 100000000),
  unique_code    smallint not null check (unique_code between 1 and 999),
  is_active      boolean not null default true,
  created_at     timestamptz not null default now(),
  unique (property_id, unique_code),
  unique (property_id, name),
  unique (id, property_id)
);

create table tenants (
  id            uuid primary key default gen_random_uuid(),
  room_id       uuid not null references rooms(id) on delete restrict,
  auth_user_id  uuid references auth.users(id) on delete set null,
  name          text not null check (length(trim(name)) between 1 and 100),
  phone         text not null check (phone ~ '^62[0-9]{8,13}$'),
  email         text check (email = lower(email)),
  is_active     boolean not null default true,
  created_at    timestamptz not null default now()
);
create unique index tenants_one_active_per_room on tenants(room_id) where is_active;
create unique index tenants_one_active_per_user on tenants(auth_user_id) where is_active and auth_user_id is not null;
create index on tenants (lower(email));

create table invoices (
  id            uuid primary key default gen_random_uuid(),
  room_id       uuid not null,
  property_id   uuid not null,
  tenant_id     uuid references tenants(id) on delete set null,
  period        char(7) not null check (period ~ '^[0-9]{4}-(0[1-9]|1[0-2])$'),
  base_amount   integer not null check (base_amount > 0),
  unique_code   smallint not null check (unique_code between 1 and 999),
  amount_due    integer generated always as (base_amount + unique_code) stored,
  due_date      date not null,
  issued_at     timestamptz not null default now(),
  status        invoice_status not null default 'unpaid',
  confirmed_at  timestamptz,
  confirmed_by  uuid references auth.users(id),
  foreign key (room_id, property_id) references rooms(id, property_id) on delete restrict,
  unique (room_id, period),
  check ((status = 'paid') = (confirmed_at is not null))
);
create index on invoices (property_id, period);
create index on invoices (status, due_date);

create table payment_proofs (
  id             uuid primary key default gen_random_uuid(),
  invoice_id     uuid not null references invoices(id) on delete restrict,
  uploaded_by    uuid not null references auth.users(id),
  image_path     text not null unique,
  image_sha256   char(64) not null check (image_sha256 ~ '^[0-9a-f]{64}$'),
  status         proof_status not null default 'pending',
  reject_reason  text check (length(reject_reason) <= 500),
  ocr_status     ocr_status not null default 'processing',
  ocr_json       jsonb,
  flags          jsonb not null default '{}',
  reference_no   text,
  created_at     timestamptz not null default now(),
  reviewed_at    timestamptz,
  reviewed_by    uuid references auth.users(id),
  check (status <> 'rejected' or length(trim(coalesce(reject_reason,''))) > 0),
  check ((status = 'pending') = (reviewed_at is null))
);
create unique index proofs_one_pending_per_invoice on payment_proofs(invoice_id) where status = 'pending';
create index on payment_proofs (image_sha256);
create index on payment_proofs (reference_no) where reference_no is not null;
create index on payment_proofs (invoice_id, created_at desc);

create table invoice_events (
  id           bigint generated always as identity primary key,
  invoice_id   uuid not null references invoices(id) on delete cascade,
  proof_id     uuid references payment_proofs(id),
  from_status  invoice_status not null,
  to_status    invoice_status not null,
  actor_type   actor_type not null,
  actor_id     uuid,
  reason       text,
  created_at   timestamptz not null default now()
);
create index on invoice_events (invoice_id, created_at);

create table notifications (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users(id) on delete cascade,
  type        text not null check (type in ('proof_submitted','proof_rejected','proof_confirmed','invoice_issued','invoice_late')),
  title       text not null,
  body        text not null,
  data        jsonb not null default '{}',
  read_at     timestamptz,
  created_at  timestamptz not null default now()
);
create index on notifications (user_id, created_at desc);
create index notifications_unread on notifications (user_id) where read_at is null;

create table monthly_snapshots (
  id            uuid primary key default gen_random_uuid(),
  property_id   uuid not null references properties(id) on delete cascade,
  period        char(7) not null check (period ~ '^[0-9]{4}-(0[1-9]|1[0-2])$'),
  generated_at  timestamptz not null default now(),
  data          jsonb not null,
  unique (property_id, period)
);

create table cron_runs (job text not null, run_key text not null, ran_at timestamptz not null default now(), primary key (job, run_key));
create table rate_limits (key text not null, window_start timestamptz not null, hits int not null default 0, primary key (key, window_start));
```

### RLS

Fungsi helper `security definer`, `stable`, `search_path=''`: `is_property_owner(pid)` dan `my_room_ids()`.

| Tabel | Pemilik | Penghuni | Tulis |
|---|---|---|---|
| properties | select/update/insert (`owner_id = auth.uid()`) | select kosan kamarnya | pemilik |
| rooms, tenants | semua aksi di kosannya | select baris sendiri | pemilik |
| invoices, payment_proofs, invoice_events | select | select kamar sendiri | **service role saja** |
| notifications | select + `update(read_at)` milik sendiri | sama | kolom lain via service role |
| monthly_snapshots | select | — | service role |
| cron_runs, rate_limits | — | — | service role |

`EXECUTE` pada `apply_invoice_transition` dan `hit_rate_limit` dicabut dari `public`, `anon`, dan `authenticated`.

### Transisi yang diizinkan

- `unpaid → reported`: penghuni submit bukti
- `reported → paid`: pemilik konfirmasi (atau integration nanti)
- `reported → unpaid`: pemilik tolak, wajib alasan
- `unpaid → paid`: pemilik tandai lunas manual (mis. tunai)
- Semua transisi dari `paid` ditolak

## 4. Env

`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`, `OCR_MODEL`, `CRON_SECRET`, `APP_URL`. Hanya `NEXT_PUBLIC_*` yang boleh sampai ke klien; `lib/env.ts` memakai `import 'server-only'`.

## 5. Pertanyaan terbuka

1. **Login penghuni**: usulan OTP/magic link email (gratis); pemilik login email + password.
2. **ID model OCR**: `claude-haiku-5-5` belum terverifikasi; usulan `OCR_MODEL` default `claude-haiku-4-5`, diverifikasi di Fase 3.
3. **Supabase dev**: Docker + Supabase CLI lokal, atau project Supabase cloud (Free) khusus dev.
4. **Waktu terbit invoice**: H-7 sebelum jatuh tempo (bisa diatur), atau selalu tanggal 1.
