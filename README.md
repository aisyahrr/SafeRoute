# Safe Route by MATA 🌙

Aplikasi anti-begal buat mahasiswa Universitas Pamulang. Tujuannya satu: bikin pulang malem lebih aman. Caranya: rute yang lewat jalan besar & rame, peringatan kalo lo keluar jalur, dan auto-SOS kalau hp lo dirampok / mati / hilang sinyal.

Dibangun pas hackathon, jadi banyak keputusan desain dioptimalisasi buat **demo-able + works under demo conditions**, bukan buat scale.

---

## Tim MATA

| Nama | Role |
|---|---|
| **Malika Shakila** | Hipster (design & product) |
| **Muchtar Ali Anwar** | Hacker (engineering) |
| **M Tohari Maolana** | Hustler (business & strategy) |
| **Aisyah Rahmawati** | Hacker (engineering) |

---

## Daftar Isi

- [Fitur Utama](#fitur-utama)
- [Tech Stack](#tech-stack)
- [Algoritma Routing Anti-Begal](#algoritma-routing-anti-begal)
- [Dead-Man's Switch (Auto-SOS)](#dead-mans-switch-auto-sos)
- [Safety PIN Anti-Bypass](#safety-pin-anti-bypass)
- [Skema Database](#skema-database)
- [Arsitektur Realtime](#arsitektur-realtime)
- [Struktur Project](#struktur-project)
- [Setup Local](#setup-local)
- [Setup Supabase](#setup-supabase)
- [Setup Telegram Bot](#setup-telegram-bot)
- [Setup pg_cron Auto-SOS](#setup-pg_cron-auto-sos)
- [Build APK](#build-apk)
- [Demo Flow](#demo-flow)
- [Trade-off & Limitasi](#trade-off--limitasi)
- [Catatan Lisensi & Aset](#catatan-lisensi--aset)

---

## Fitur Utama

1. **Rute Aman** — Pilih 3 rute (Teraman / Waspada / Risiko) yang dihitung dari data berita kriminal real-time. Sistem otomatis nge-avoid zona begal sambil tetep lewat jalan besar yang rame & terang.
2. **SOS Manual** — Tap satu tombol → semua kontak darurat dapet notif Telegram instant + bisa blast WhatsApp ke tiap kontak.
3. **Dead-Man's Switch** — Heartbeat tiap 20 detik selama perjalanan. Hp dirampok / mati / hilang sinyal → auto-SOS Telegram ke kontak dalam ≤2 menit, tanpa user perlu pencet apapun.
4. **Route Deviation Alert** — Modal warning + countdown 28 detik kalau user keluar dari rute aman > 150 m (skenario: dipaksa belok begal).
5. **Safety PIN 4 digit** — "Saya Aman" minta PIN. Begal yang ngerampok HP gak tau PIN → auto-SOS jalan walau dia mencet aman.
6. **Notif Berita Realtime** — Pas pipeline scraping news Groq nambah berita kriminal baru, semua user yang lagi buka app langsung dapet banner + masuk ke tab Notifikasi.

---

## Tech Stack

| Layer | Stack | Kenapa |
|---|---|---|
| Mobile | Expo SDK 56 + React Native 0.85 + TypeScript strict | Dev build, native module aman (battery, location, maps) |
| Backend | Supabase (Postgres + RLS + Realtime + Edge Functions + pg_cron) | Free tier dapet semua: DB + auth + realtime + cron + serverless |
| Routing | OpenRouteService (ORS) `driving-car` profil | Free, support `avoid_polygons` buat nge-skip zona begal |
| Geocoding | Google Places + ORS Pelias fallback | Google buat akurasi nama jalan, ORS buat free tier limit |
| LLM Pipeline | Groq (Llama 3.3 70B) | Klasifikasi berita kriminal → level bahaya, ekstrak koordinat dari teks |
| Notif Push | Telegram Bot API + Supabase Realtime | Bot Telegram = gratis + masuk meski app close (offline-friendly) |
| Crypto | `js-sha256` pure JS | Hash PIN tanpa native module → tidak butuh rebuild dev client |

---

## Algoritma Routing Anti-Begal

Bagian paling jantung dari aplikasi. Tujuannya: dari titik A ke B, pilih rute yang **lewat jalan gede + ngindarin lokasi yang ada laporan begal**.

### Pertanyaan inti

> "Kalau gua naik motor dari Pamulang ke Lebak Bulus jam 11 malem, jalan mana yang paling aman?"

Asumsi:
- Target user = mahasiswa Unpam naik motor matic
- Jam rawan = malem (penerangan minim, jalan sepi)
- Definisi "aman" = (jalan ramai + terang) ∧ (jauh dari lokasi laporan begal)

### Konsep dasar: 3 lapisan rute

| Tipe | Strategi ORS | Yang dihindari | Detour budget |
|---|---|---|---|
| **Teraman** | `recommended` (optimal waktu via jalan kelas tinggi) | Semua zona begal level ≥3 | ≤1.4× rute terpendek |
| **Waspada** | `shortest` (jarak terpendek) | Zona disertai kekerasan/fatal level ≥4 | ≤1.3× rute terpendek |
| **Risiko** | `shortest` tanpa avoid | (Tidak ada) | — (baseline) |

User boleh pilih sendiri. Default ke Teraman, tapi waktu/jarak ada di kartu biar trade-off jelas.

### Kenapa "Teraman" pake `recommended` instead of `shortest`

ORS punya 2 preference utama:
- `shortest` = jarak terpendek (gak peduli kelas jalan)
- `recommended` = optimal waktu (otomatis bias ke jalan kelas tinggi/arteri besar)

Buat malem, **jalan besar = lebih aman** karena:
- Penerangan lampu jalan
- Saksi (warga / driver / ojol lewat)
- Gak ada gang sepi
- CCTV biasanya ada di arteri besar

Jadi `recommended` (bias jalan besar) > `shortest` (bisa lewat gang potong) buat keamanan malem. Trade-off-nya jarak bisa lebih jauh sedikit — itu **fitur**, bukan bug.

### Bagaimana zona begal dihindari (GeoJSON MultiPolygon)

Untuk tiap laporan berita kriminal yang `trusted = true` dan `level >= threshold`:

1. Bikin **lingkaran 12-sisi** mengelilingi koordinat kejadian, dengan radius berdasarkan level:
   ```
   radius(level) = 90 + level × 50  meter
   ```
   - Level 3 (begal kendaraan) → radius 240 m
   - Level 4 (begal + kekerasan) → radius 290 m
   - Level 5 (fatal) → radius 340 m

   Kenapa segini? Setara satu blok jalan kota. Cukup lebar buat geser ke jalan tetangga, gak terlalu lebar sampai memutari seluruh klaster.

2. Vertex lingkaran di-scale `1 / cos(π/12)` (≈ 1.035) supaya **circumscribed polygon** (mengelilingi penuh radius), bukan inscribed (motong ke dalam). Penting karena ORS treat polygon edge sebagai wall — kalau inscribed, ada celah di antara sisi yang bisa di-cut.

3. Kumpulkan semua lingkaran → MultiPolygon GeoJSON → kirim ke ORS sebagai `avoid_polygons`.

4. ORS otomatis hitung shortest path yang **TIDAK menyentuh** polygon manapun. Ini operasi geometri di ORS yang udah teroptimisasi (gak perlu kita reinvent).

### Safe filter: jangan hindari zona yang menutup titik asal/tujuan

Kalau rumah lo kebetulan di radius 240 m dari titik laporan begal, sistem **harus** rute lewat sana — gak bisa dihindari. Logika:

```typescript
.filter((r) =>
  haversine(r, origin) > 500 && haversine(r, dest) > 500
)
```

500m = sedikit lebih besar dari radius polygon terbesar (340m), supaya ada slack ruang ORS buat masuk/keluar.

### Detour budget: rem keselamatan dari over-engineering

Kadang zona begal di kota begitu rapat sampai bentuk "tembok" yang nyumpel jalan utama. Tanpa rem, ORS bisa muter via Jakarta buat balik ke Pamulang → 45 km untuk perjalanan yang seharusnya 8 km.

User normal gak akan ikutin rute kek gitu — mending hentian ojek aja.

Solusi: bandingin tiap rute dengan baseline (rute Risiko = shortest tanpa avoid):

```typescript
const baseKm = risikoRaw.distanceKm;

const waspadaPick = waspadaRaw.distanceKm <= baseKm * 1.3
  ? waspadaRaw
  : risikoRaw;  // fall-back ke Risiko

const teramanPick = teramanRaw.distanceKm <= baseKm * 1.4
  ? teramanRaw
  : waspadaPick; // fall-back ke Waspada
```

Berlaku berantai. Output garansi: rute yang ditampilin selalu **realistic untuk dipakai**, bukan rute aman teoritis yang useless.

### Safety score (0..100)

Setelah ORS balikin geometri rute, kita re-evaluate berapa "aman" rute itu:

```typescript
function routeSafetyScore(coords: LatLng[], reports: Report[]): number {
  let penalty = 0;
  for (const r of reports) {
    if (!r.trusted) continue;
    const nearestPoint = min(haversine(r, c) for c in coords);
    const radius = dangerRadius(r.level);
    const w = r.level × recencyWeight(r.occurredAt);

    if (nearestPoint <= radius)       penalty += 2.0 × w;  // menembus zona
    else if (nearestPoint <= radius + 250) penalty += 0.8 × w;  // serempet
    else if (nearestPoint <= radius + 600) penalty += 0.3 × w;  // dekat
  }
  return clamp(0, 100, round(100 - penalty));
}
```

Penalti dibobot oleh:
- **Severity** (level 1-5): kejadian level 5 menarik penalti lebih besar daripada level 3
- **Recency** (kapan terjadi): kejadian ≤14 hari → bobot penuh. >180 hari → bobot 0.35 (tetap dihitung — lokasi yang fatal 6 bulan lalu tetep patut diwaspadai).

### Monotonic score guard

Edge case: kadang Teraman score-nya keluar lebih rendah dari Waspada karena random recency weighting → bingungin user. Solusi: paksa monotonic di display:

```typescript
waspada.safetyScore = Math.min(waspada.safetyScore, teraman.safetyScore);
risiko.safetyScore  = Math.min(risiko.safetyScore,  waspada.safetyScore);
```

Garansi: Teraman ≥ Waspada ≥ Risiko di kartu. Badge skor tidak akan kontradiktif sama label rute.

### Graceful degradation

Kalau ORS error untuk preference tertentu (mis. avoid polygon bikin route impossible):

```
Teraman gagal → pakai geometri Waspada
Waspada juga gagal → pakai geometri Risiko
Risiko gagal → return null (tampil error state)
```

Tidak ada fallback ke garis statis — itu bisa nyesatin di malem hari (rute lo gak ada di jalan aslinya). Mending kasih error daripada salah info.

### Visualisasi di peta

Sumber kebenaran satu: konstanta `dangerRadiusM(level)` di [src/lib/routing.ts](src/lib/routing.ts) dipake oleh:
1. Router buat bikin avoid polygon
2. Peta buat gambar lingkaran zona rawan

Otomatis sinkron. Kalau lo ubah ukuran radius, peta langsung update ikut ngegambar. Gak akan pernah ada situasi "peta nunjukin zona kecil tapi router ngindarin lebih lebar".

---

## Dead-Man's Switch (Auto-SOS)

Tombol SOS manual cuma kerja kalau user **bisa** mencet. Kalau hp dirampok / pingsan / mati baterai / hilang sinyal panjang → tombol useless.

Solusi: **kebalikan**. Sistem trigger kalau user **tidak ngirim sinyal** dalam waktu tertentu.

### Flow

```
Detik 00:00  ─[💓]─ heartbeat: posisi A, batt 85%
Detik 00:20  ─[💓]─ heartbeat: posisi B, batt 84%
Detik 00:40  ─[💓]─ heartbeat: posisi C, batt 84%
Detik 01:00  ─[💓]─ heartbeat: posisi D, batt 83%
Detik 01:20  ─[💓]─ heartbeat: posisi D, batt 83%   ← hp dirampok di sini
Detik 01:40  ─[ ! ]─ heartbeat hilang
Detik 02:00  ─[ ! ]─ heartbeat hilang
...
Detik 03:20  ─[🚨]─ pg_cron fire dead-man-check
                    → kirim Telegram ke kontak darurat
                    → mark sos_sent_at = now()
```

### Komponen

| Bagian | File | Tanggung jawab |
|---|---|---|
| Pengirim heartbeat | [src/screens/NavigasiAktifScreen.tsx](src/screens/NavigasiAktifScreen.tsx) | `setInterval(sendHeartbeat, 20_000)` selama tripId valid |
| Service heartbeat | [src/services/trips.ts](src/services/trips.ts) | PATCH `last_heartbeat_at` ke DB |
| Edge function | [supabase/functions/dead-man-check/index.ts](supabase/functions/dead-man-check/index.ts) | Cari trip stale → kirim Telegram → mark sent |
| Scheduler | pg_cron + pg_net (Supabase) | Tiap menit invoke edge function |

### Trade-off threshold

- **Kirim tiap**: 20 detik. Lebih sering = boros baterai+data. Lebih jarang = miss detail posisi.
- **Anggap "hilang"**: 2 menit. Lebih cepat = false alarm pas lewat tunnel. Lebih lambat = nyawa hilang.

### Kenapa heartbeat cuma jalan di Navigasi Aktif

Tujuannya bukan "track user 24/7" — itu invasif. Tapi "selama lo nyatain lagi jalan (mulai perjalanan), kita backup lo". Sekali user tap Selesai / SOS / back → heartbeat off. Privacy by design.

---

## Safety PIN Anti-Bypass

Skenario: begal liat HP korban bunyi modal "Peringatan Keluar Rute". Instinct begal = pencet tombol paling menonjol = "Saya Aman" → bypass auto-SOS.

Solusi: tombol "Saya Aman" minta **PIN 4 digit** yang cuma user yang tau.

### Behavior

- Tap "Saya Aman" → keypad muncul
- Input PIN benar → modal close, lanjut navigasi normal
- Input PIN salah → label merah "PIN salah — SOS dikirim" → 600ms kemudian SOS Telegram blast ke semua kontak yang punya `telegram_chat_id`
- Diem 28 detik (countdown habis tanpa input) → SOS jalan otomatis

### Implementasi

- PIN di-hash pake `js-sha256` (pure JS, gak butuh native module) dengan salt UID:
  ```ts
  hashPin(pin, uid) = SHA256(`${uid}:${pin}`)
  ```
- Disimpen di `profiles.safety_pin_hash`. Plain PIN gak pernah dikirim ke DB / log.
- Verify in-memory cached (refresh tiap app reload), supaya UI keypad gak nahan nunggu round-trip network.

### Threat model

Ya, 4-digit PIN cuma 10.000 kombinasi, brute force trivial. Tapi:
- Threat utama = **physical** (begal liat hp, gak punya akses DB)
- DB protect by RLS — user lain gak bisa baca hash user kita
- Service role only — gak ada anon access ke `safety_pin_hash`

Jadi hash ini bukan main security. Yang ngelindungin user adalah RLS + per-attempt UI design (kalau salah → SOS langsung jalan, gak ada "coba lagi 3x").

---

## Skema Database

Tabel utama (lihat [supabase/schema.sql](supabase/schema.sql)):

```
profiles
  ├── id (uuid, FK auth.users)
  ├── full_name, email, phone
  ├── level, safe_trip_days, report_count
  ├── preferences (jsonb)
  └── safety_pin_hash       ← PIN buat anti-bypass

emergency_contacts
  ├── id, user_id (FK)
  ├── name, phone, relationship, active
  └── telegram_chat_id      ← buat auto-SOS Telegram

reports
  ├── id, title, description, category, level
  ├── latitude, longitude, location_name
  ├── source ('community' | 'news')
  ├── source_url            ← unique buat upsert pipeline
  ├── reporter_name, trusted
  └── created_at, occurred_at

trips
  ├── id, user_id (FK), token (32-hex)
  ├── origin/dest (label + lat + lon)
  ├── status ('active' | 'arrived' | 'sos' | 'cancelled')
  ├── last_lat, last_lon, last_battery
  ├── last_heartbeat_at     ← dead-man's switch trigger
  └── sos_sent_at           ← idempotency, gak spam

notifications
  ├── id, user_id (nullable buat broadcast)
  ├── type ('critical' | 'trip' | 'education' | 'system')
  ├── title, body, section
  └── link_url              ← URL artikel berita
```

### Trigger fan-out berita → notifications

```sql
create function fanout_news_to_notifications()
returns trigger as $$
begin
  if new.source = 'news' then
    insert into notifications (user_id, type, title, body, section, link_url)
    values (
      null,  -- broadcast (semua user dapet)
      case when new.level >= 2 then 'critical' else 'system' end,
      'Berita: ' || new.title,
      coalesce(new.description, ''),
      'HARI INI',
      new.source_url
    );
  end if;
  return new;
end; $$;
```

Tiap kali pipeline Groq UPSERT berita baru → trigger jalan → notif kebuat → Supabase Realtime push ke semua user yang lagi buka app.

### RLS

- `profiles`: read public, insert/update own
- `emergency_contacts`: full CRUD own
- `reports`: read public, no insert from client (cuma pipeline pake service key)
- `trips`: full CRUD own + RPC publik `get_trip_by_token` buat halaman track
- `notifications`: read own + broadcast (`user_id is null`)

---

## Arsitektur Realtime

```
┌─────────────────────────────────────────────────┐
│  Pipeline scraping Groq (cron 00:00 WIB)        │
│      │                                          │
│      ▼ UPSERT into reports (source='news')      │
├─────────────────────────────────────────────────┤
│  Postgres trigger fanout_news_to_notifications  │
│      │                                          │
│      ▼ INSERT broadcast into notifications      │
├─────────────────────────────────────────────────┤
│  Supabase Realtime publication                  │
│      │                                          │
│      ▼ WebSocket push                           │
├─────────────────────────────────────────────────┤
│  Mobile clients (semua user yang lagi online)   │
│      │                                          │
│      ├── HomeScreen: NewsBanner slide down      │
│      └── NotifikasiScreen: prepend ke list      │
└─────────────────────────────────────────────────┘
```

Setup mensyaratkan:

```sql
alter publication supabase_realtime add table public.reports;
alter publication supabase_realtime add table public.notifications;
```

---

## Struktur Project

```
saferoute/
├── App.tsx                  # entry, render MapScreen
├── src/
│   ├── lib/supabase.ts      # koneksi Supabase
│   ├── lib/routing.ts       # algoritma routing + dangerRadiusM
│   ├── types/index.ts       # tipe data (Report, DangerLevel, dll)
│   ├── constants/config.ts  # level bahaya, region peta, ambang SOS
│   ├── services/trips.ts    # heartbeat + trip service
│   ├── data/dummyReports.ts # data seed buat demo
│   └── screens/
│       ├── MapScreen.tsx          # peta utama + marker + legenda
│       └── NavigasiAktifScreen.tsx# navigasi aktif + heartbeat
├── supabase/
│   ├── schema.sql
│   └── functions/
│       ├── dead-man-check/index.ts
│       └── manual-sos/index.ts
└── .env.example
```

---

## Setup Local

```bash
git clone <repo-url>
cd saferoute
npm install

# .env (copy dari .env.example)
EXPO_PUBLIC_SUPABASE_URL=https://<project>.supabase.co
EXPO_PUBLIC_SUPABASE_KEY=sb_publishable_xxx
EXPO_PUBLIC_ORS_API_KEY=xxx
EXPO_PUBLIC_GOOGLE_MAPS_KEY=xxx
EXPO_PUBLIC_GROQ_API_KEY=xxx  # untuk pipeline scraping

npx expo start --dev-client
```

Butuh dev client (bukan Expo Go) karena `react-native-maps`, `expo-battery`, `expo-location` semua native module.

> App tetap jalan tanpa Supabase (pakai data dummy di `src/data/dummyReports.ts`), tapi fitur backend belum aktif sampai env diisi.

---

## Setup Supabase

1. **Schema**: SQL Editor → New Query → paste seluruh isi [supabase/schema.sql](supabase/schema.sql) → Run.

2. **Patch schema lama** (kalau lo udah jalanin schema di awal sebelum fitur baru):
   ```sql
   alter table public.profiles add column if not exists safety_pin_hash text;
   alter table public.emergency_contacts add column if not exists telegram_chat_id text;
   alter table public.notifications add column if not exists link_url text;
   ```

3. **Trigger fan-out berita**:
   ```sql
   drop policy if exists "notif own" on public.notifications;
   create policy "notif read" on public.notifications
     for select using (auth.uid() = user_id or user_id is null);

   create or replace function public.fanout_news_to_notifications()
   returns trigger language plpgsql security definer set search_path = public as $$
   begin
     if new.source = 'news' then
       insert into public.notifications (user_id, type, title, body, section, link_url)
       values (
         null,
         case when new.level >= 2 then 'critical' else 'system' end,
         'Berita: ' || new.title,
         coalesce(new.description, ''),
         'HARI INI',
         new.source_url
       );
     end if;
     return new;
   end; $$;

   drop trigger if exists trg_fanout_news on public.reports;
   create trigger trg_fanout_news
     after insert on public.reports
     for each row execute function public.fanout_news_to_notifications();
   ```

4. **Enable Realtime**:
   ```sql
   alter publication supabase_realtime add table public.reports;
   alter publication supabase_realtime add table public.notifications;
   ```

5. **Deploy edge functions**:
   ```bash
   npx supabase functions deploy dead-man-check
   npx supabase functions deploy manual-sos
   ```

---

## Setup Telegram Bot

1. Di Telegram, chat **@BotFather** → `/newbot` → kasih nama → copy token (format `1234567890:AAH-xxxxx`).
2. Set secret di Supabase:
   ```bash
   npx supabase secrets set TELEGRAM_BOT_TOKEN=<token-bot>
   ```
3. Kontak darurat lo install Telegram → search bot lo → tap Start.
4. Buka `https://api.telegram.org/bot<TOKEN>/getUpdates` → cari `chat.id` (angka panjang).
5. Tambahin chat_id ke kontak via UI (Kontak Darurat → field Telegram Chat ID).

---

## Setup pg_cron Auto-SOS

```sql
-- Enable extensions
create extension if not exists pg_cron with schema extensions;
create extension if not exists pg_net with schema extensions;

-- Simpen service role key di Vault
select vault.create_secret(
  '<service-role-key>',
  'service_role_key',
  'Buat pg_cron invoke edge function dead-man-check'
);

-- Jadwalin tiap menit
select cron.schedule(
  'saferoute-dead-man-check',
  '* * * * *',
  $$
  select net.http_post(
    url := 'https://<project>.supabase.co/functions/v1/dead-man-check',
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || (select decrypted_secret from vault.decrypted_secrets where name = 'service_role_key'),
      'Content-Type', 'application/json'
    )
  );
  $$
);

-- Verify
select * from cron.job;
select * from cron.job_run_details order by start_time desc limit 5;
```

Cek `status = succeeded` setelah 1-2 menit → cron aktif.

---

## Build APK

```bash
npx eas-cli build -p android --profile preview
```

Selesai (~15 menit), dapet URL download APK. Kirim ke channel `link-repo` di hackathon.

> ⚠️ **Peta butuh dev build untuk fitur penuh.** `react-native-maps` jalan di Expo Go dengan provider default. Kalau mau Google Maps + Heatmap, perlu dev build (`npx expo run:android`) dan API key Google Maps.

---

## Demo Flow

Demo end-to-end (~3 menit):

1. **HomeScreen** — tunjukin stats (Trip Aman / Kontak Darurat / Berita 24 jam), badge level. Insert berita via SQL Editor di laptop → audience liat NewsBanner slide down di hp + notif di tab Notifikasi auto-muncul.

2. **Perjalanan** — pilih asal (Unpam) dan tujuan dekat. Tap Cari Rute. Tunjukin 3 kartu rute dengan skor + jarak + waktu. Cerita: "Teraman 92/100 tapi 12 menit lebih lama dari Risiko — user pilih trade-off-nya sendiri."

3. **Navigasi Aktif** — Mulai Perjalanan dari kartu Teraman. Banner navigasi muncul. Tunjukin posisi user di peta + heartbeat (silent — cek di SQL: `select * from trips order by created_at desc limit 1;` last_heartbeat_at update).

4. **Demo SOS manual** — tap tombol SOS merah → blast Telegram ke semua kontak. Tunjukin notif di HP kontak (atau lo sendiri kalau pake akun demo).

5. **Demo route deviation** — long-press "Monitoring Aktif" di bottom sheet. Modal oranye muncul + countdown. Tap "Saya Aman (PIN)" → keypad → input PIN salah → instan SOS Telegram jalan. Cerita: "Begal mencet aman tapi gak tau PIN — auto-SOS tetep masuk."

6. **Demo dead-man's switch** — kembali ke navigasi normal. Long-press banner atas. Banner merah → "Auto-SOS terkirim ke N kontak" dalam 2 detik. Cerita: "Skenario: hp dirampok, user gak bisa pencet apapun. Sistem detect heartbeat hilang → auto-SOS jalan."

---

## Trade-off & Limitasi

| Yang dilakuin | Trade-off |
|---|---|
| Heartbeat tiap 20 detik | Boros baterai +data sedikit. Kalau lo mau hemat, bisa naikin ke 30 detik dengan ngubah threshold dead-man ke 3 menit |
| PIN cuma 4 digit | Brute force trivial — tapi threat-nya physical (begal), bukan offline DB dump. RLS yang utama melindungi |
| ORS free tier | Limit 6000 km per request, rate-limited per menit. Cukup buat user MVP, butuh self-host atau paid tier untuk scale |
| pg_cron tiap 1 menit | Worst case 60 detik delay buat detect heartbeat hilang. Bisa nge-tiap 30 detik dengan ngubah `'* * * * *'` ke `'*/30 * * * * *'` (butuh pg_cron versi mendukung second) |
| Telegram bot per project | Satu bot serve semua user. Kalau user lo ribuan + sering blast, bot bisa rate-limited Telegram (~30 msg/sec). Solusi: pakai bot pool atau switch ke Expo Push Notification |
| Notif berita broadcast ke semua user | Tiap berita = 1 notif buat tiap user. Kalau lo punya 10k user + 50 berita/hari = 500k notif/hari. Solusi: geo-filter (cuma user dalam radius 5km dari kejadian), simpan sebagai broadcast row tunggal (user_id=null) sudah dipake — gak perlu fan-out per user |
| Service role key di pg_cron via Vault | Kalau key bocor (DB dump leaked), attacker dapet full DB access. Mitigation: rotate service role key periodik, pakai dedicated service account dengan least-privilege RLS bypass |
| Track page (sempat dibuat) di-drop | Supabase Edge Function impose CSP `default-src 'none'; sandbox` yang block halaman web interaktif. Pivot ke Telegram-only — pesan udah lengkap dengan Google Maps link, gak butuh halaman track |

---

## Catatan Lisensi & Aset

- ORS API key: free tier untuk OpenRouteService — registrasi di [openrouteservice.org](https://openrouteservice.org)
- Google Maps API key: butuh billing account, restrict ke Android package + SHA-1 di GCP console
- Groq API key: free tier untuk Llama 3.3 — registrasi di [console.groq.com](https://console.groq.com)
- Telegram Bot: gratis tanpa batas
- Supabase: free tier (500MB DB, 5GB bandwidth/bulan) cukup untuk demo MVP

Semua API key client-safe sudah pake prefix `EXPO_PUBLIC_*` — itu dibundel ke APK. Kalau lo mau melindungi, restrict di console masing-masing (rate limit, allowed origins).

Service role key Supabase **JANGAN PERNAH** di-`EXPO_PUBLIC_*` — bakal bundle ke APK dan bypass RLS untuk semua user. Server-side only (edge function, GitHub Actions, pg_cron Vault).

---

Dibuat untuk hackathon oleh **Tim MATA**. Hubungi maintainer kalau lo nemu bug atau mau kontribusi.
