# PLAN_CUACA.md — Tab Cuaca & Polusi (Tab ke-3/4)

> Branch `dev` — 2026-08-27
> Status: **Done** — Phase C1/C2 selesai, 7 issues closed, `check 0` `build pass`. Tidak ada task cuaca tertunda di roadmap aktif.
> Referensi: `../AGENTS.md`, `../ARCHITECTURE.md`, `DOC_FITUR_MARKET_TENTANG.md` (§9 Roadmap)
> BottomNav saat ini: `Berita /` · `Cuaca /cuaca` · `Harian /harian` · `Tentang /tentang` (4 tab, Market hidden). Cuaca sudah jadi **tab ke-2** `Cuaca /cuaca` + `/cuaca/cari`.

## 1. Ringkasan

**1 tab baru `Cuaca`** yang cover 2 kebutuhan user sekaligus:
1. **Suhu kota lokasi saat ini** + forecast 7 hari + per jam
2. **Polusi (AQI/PM2.5/PM10)** lokasi saat ini

Plus **halaman search & filter kota lain** (1 halaman `/cuaca/cari` dipakai untuk kedua data — pilih kota → balik ke `/cuaca?lat=&lon=` tampil suhu+polusi kota itu).

**Prinsip:** Gratis tanpa API key berbayar, server-side fetch + `cached()` TTL 10m + CDN `s-maxage=600`, fallback Jakarta jika user tolak lokasi.

---

## 2. API Gratis (Stack Rekomendasi — Tanpa Key)

| Kebutuhan | API | Endpoint | Key | Limit |
|-----------|-----|----------|-----|-------|
| **Geocoding search kota** | **Open-Meteo Geocoding** | `https://geocoding-api.open-meteo.com/v1/search?name={q}&count=5&language=id&format=json` | Tanpa | Unlimited |
| **Reverse geocode lat/lon → nama kota** | Open-Meteo Reverse | `https://geocoding-api.open-meteo.com/v1/reverse?latitude={lat}&longitude={lon}&language=id` | Tanpa | Unlimited |
| **Suhu current + forecast 7 hari + hourly** | **Open-Meteo Weather** | `https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m&daily=temperature_2m_max,temperature_2m_min,weather_code,precipitation_sum&hourly=temperature_2m,weather_code&timezone=Asia/Jakarta&forecast_days=7` | Tanpa | Unlimited |
| **Polusi AQI/PM2.5/PM10** | **Open-Meteo Air Quality** | `https://air-quality-api.open-meteo.com/v1/air-quality?latitude={lat}&longitude={lon}&current=us_aqi,pm2_5,pm10,ozone,nitrogen_dioxide&timezone=Asia/Jakarta` | Tanpa | Unlimited |

**Kenapa trio Open-Meteo:**
- 1 provider, 0 daftar key, 0 cost — sama seperti CoinGecko+Yahoo di Market
- Unlimited + cache 10m + CDN = aman traffic tinggi
- `language=id`, `timezone=Asia/Jakarta` native, `weather_code` ada mapping WMO
- Alternatif fallback (jika butuh): OpenWeatherMap (1000/hari butuh key), Nominatim OSM (1 req/s)

**Mapping `weather_code` WMO → label Indonesia:**
`0 Cerah, 1 Cerah Berawan, 2 Berawan, 3 Mendung, 45 Kabut, 51 Gerimis ... 95 Petir` — buat `src/lib/weatherCode.ts`.

---

## 3. Arsitektur & Alur Data

```
Browser (navigator.geolocation) ──► lat/lon ──► +page.server.ts ?lat=&lon=
                                                     │
                                                     ├─► lib/server/weather.ts
                                                     │      ├─ cached('weather:current:{lat},{lon}') → Open-Meteo Weather (7)
                                                     │      ├─ cached('weather:air:{lat},{lon}')     → Open-Meteo Air Quality (7s)
                                                     │      └─ cached('weather:geo:{q}')             → Open-Meteo Geocoding (7s)
                                                     │      + cached('weather:reverse:{lat},{lon}')   → Reverse geocode
                                                     │
                                                     └─► +page.svelte props { weather, airQuality, cityName, fetchedAt }
                                                          │
                                                          ├─ WeatherCard (suhu, feels like, H/L, humidity, wind, code)
                                                          ├─ AirQualityCard (AQI 0-300 + kategori + PM2.5/PM10)
                                                          ├─ Forecast 7 hari (daily max/min + code)
                                                          └─ Hourly strip (24 jam)

Search: /cuaca/cari?q=Bandung → geocoding 5 hasil → klik → navigate /cuaca?lat=-6.9&lon=107.6
```

**Reuse pola Market:**
- `Promise.allSettled` weather + air quality paralel — 1 gagal ≠ halaman crash
- `fetchWithTimeout` 7s + `cached()` TTL 10m (weather/air) & 1 jam (geocoding)
- `s-maxage=600` CDN
- Fallback: jika `lat/lon` tidak ada & geolocation ditolak → default `Jakarta -6.2,106.8`

**Cache keys:**

| Key | TTL | Isi |
|-----|-----|-----|
| `weather:current:{lat},{lon}` | 10m | current + daily + hourly |
| `weather:air:{lat},{lon}` | 10m | us_aqi, pm2_5, pm10 |
| `weather:geo:{q}` | 1 jam | search 5 kota |
| `weather:reverse:{lat},{lon}` | 1 hari | nama kota |

---

## 4. Peta Route & Navigasi

| Route | File | Fungsi |
|-------|------|--------|
| `/cuaca` | `routes/cuaca/+page.server.ts` + `+page.svelte` | Halaman utama: lokasi saat ini (query `?lat=&lon=` atau default Jakarta), 2 card suhu+polusi + forecast 7 hari + hourly |
| `/cuaca/cari` | `routes/cuaca/cari/+page.server.ts` + `+page.svelte` | Search kota: input `?q=`, geocoding 5 hasil, klik → `/cuaca?lat=&lon=&name=` |

**BottomNav update:**

```
[📰 Berita /] [📈 Market /market] [🌤️ Cuaca /cuaca] [ℹ️ Tentang /tentang]  → 4 tab
```

`BottomNav.svelte` tambah entry `cuaca` icon `sun/cloud`, `match: p.startsWith('/cuaca')`. Urutan: Cuaca sebelum Tentang (Tentang paling kanan).

**Geolocation flow (client):**
1. `onMount` cek `navigator.geolocation`
2. Jika ada & permission granted → `getCurrentPosition` → `goto('/cuaca?lat=&lon=')` (replaceState)
3. Jika ditolak → tetap di default Jakarta + tombol `Gunakan Lokasi Saya` + `Cari Kota Lain`

---

## 5. UI/UX Detail

### 5.1 `/cuaca` — Halaman Utama

```
Header: Cuaca • Diperbarui 2 menit lalu (timeAgo)
[Card Lokasi Saat Ini — gradient sky/blue]
  Jakarta, DKI Jakarta • -6.2,106.8
  31°C Cerah Berawan • Feels like 34°C
  H 33° L 26° • Humidity 78% • Wind 12 km/h
  [📍 Gunakan Lokasi Saya] [🔍 Cari Kota Lain → /cuaca/cari]

[Card Polusi — warna by AQI]
  AQI 85 Sedang (kuning) • PM2.5 18 µg/m³ • PM10 32 • O3 45
  Progress bar 0-300: Baik 0-50 hijau, Sedang 51-100 kuning, Tidak Sehat 101-150 oranye...

[Forecast 7 Hari — horizontal scroll]
  Sen 31°/26° 🌤️ • Sel 32°/27° 🌧️ ...

[Hourly 24 jam — strip]
  12:00 31° • 13:00 32° • ...

Footer: Sumber Open-Meteo (CAMS) • Bukan data BMKG resmi
```

**Kategori AQI (US AQI):**
`0-50 Baik hijau, 51-100 Sedang kuning, 101-150 Tidak Sehat oranye, 151-200 Sangat Tidak Sehat merah, 201-300 Berbahaya ungu, 300+ Berbahaya maroon`

**Dark mode:** card `dark:bg-neutral-900`, text `dark:text-neutral-100`, progress bar tetap warna AQI.

### 5.2 `/cuaca/cari` — Search & Filter

```
Header: Cari Kota • Back → /cuaca
[Input: placeholder "Cari kota, mis. Bandung" + debounce 300ms]
Hasil 5:
  • Bandung, Jawa Barat, Indonesia -6.9,107.6 → klik → /cuaca?lat=-6.9&lon=107.6&name=Bandung
  • ...
Empty: "Tidak ada kota ditemukan"
Filter (opsional Phase 2): chip Provinsi / Pulau (client filter dari hasil geocoding `admin1`)
```

**Debounce:** client `setTimeout 300ms` → `fetch('/cuaca/cari?q=')` via server load, tidak hit API tiap ketik.

---

## 6. File Plan

### Baru (7)

| File | Fungsi |
|------|--------|
| `src/lib/server/weather.ts` | `fetchWeather(lat,lon)`, `fetchAirQuality(lat,lon)`, `searchCity(q)`, `reverseGeocode(lat,lon)` — cached + fetchWithTimeout |
| `src/lib/weatherCode.ts` | Mapping WMO `weather_code → {label, icon}` id |
| `src/lib/components/WeatherCard.svelte` | Card suhu current + H/L + humidity + wind, props `weather` |
| `src/lib/components/AirQualityCard.svelte` | Card AQI + kategori warna + PM2.5/PM10, props `air` |
| `src/lib/components/ForecastStrip.svelte` | 7 hari + hourly strip |
| `src/routes/cuaca/+page.server.ts` | Load `?lat=&lon=&name=` → weather+air parallel, fallback Jakarta |
| `src/routes/cuaca/+page.svelte` | UI utama + geolocation client + link cari |
| `src/routes/cuaca/cari/+page.server.ts` | Load `?q=` → geocoding 5 hasil |
| `src/routes/cuaca/cari/+page.svelte` | Input search + list hasil + filter |

### Ubah (2)

| File | Perubahan |
|------|-----------|
| `src/lib/components/BottomNav.svelte` | Tambah tab Cuaca `🌤️` `href="/cuaca"` |
| `src/routes/+layout.svelte` | Tidak perlu ticker tambahan (cuaca tidak pakai ticker) |

**Total:** ~9 file.

---

## 7. Tahapan Eksekusi (2 Phase)

### Phase C1 — MVP (1-2 hari)

1. `weather.ts` + `weatherCode.ts` — 3 fetch + cache
2. `cuaca/+page.*` — current suhu + polusi 2 card + Jakarta default
3. `BottomNav` tambah tab Cuaca
4. Geolocation client `Gunakan Lokasi Saya`

**Deliverable C1:** `/cuaca` tampil suhu+polusi Jakarta & lokasi saat ini, 0 search dulu.

### Phase C2 — Search (1 hari)

1. `cuaca/cari/+page.*` — geocoding search 5 hasil + debounce
2. `reverseGeocode` untuk nama kota dari lat/lon
3. Forecast 7 hari + hourly strip
4. AQI kategori warna + progress bar

**Deliverable C2:** Search kota lain + forecast lengkap — tab Cuaca komplit.

---

## 8. Verifikasi

```bash
npm run check   # 0 error
npm run build   # pass, PWA ok
npm run dev     # cek:
# /cuaca → suhu Jakarta 31°C + AQI 85 + 7 hari
# klik Gunakan Lokasi Saya → allow → lat/lon update
# /cuaca/cari?q=Surabaya → 5 hasil → klik → /cuaca?lat=-7.2&lon=112.7 → suhu Surabaya
# tolak lokasi → tetap Jakarta + tombol cari
# dark mode card tetap kontras
# BottomNav 4 tab, Cuaca aktif di /cuaca & /cuaca/cari
```

---

## 9. Risiko & Mitigasi

| Risiko | Mitigasi |
|--------|----------|
| Geolocation ditolak | Fallback Jakarta + tombol Cari Kota Lain, tidak crash |
| Open-Meteo 5xx | `Promise.allSettled` — suhu gagal → tampil polusi saja & sebaliknya, card `Tidak tersedia` |
| Geocoding 0 hasil | Tampil `Tidak ada kota ditemukan` + saran 3 kota populer (Jakarta, Surabaya, Medan) |
| AQI model vs real station | Footer `Sumber CAMS (model), bukan BMKG resmi` + link info |
| BottomNav 4 tab sesak di 320px | Icon 5×5 + label `text-[10px]`, tetap 4 tab max (jangan 5) |

---

## 10. Referensi

- Open-Meteo Weather: `https://open-meteo.com/en/docs#current=temperature_2m&forecast_days=7`
- Open-Meteo Air Quality: `https://open-meteo.com/en/docs/air-quality-api`
- Open-Meteo Geocoding: `https://open-meteo.com/en/docs/geocoding-api`
- WMO Weather Code: `https://open-meteo.com/en/docs#weathervariables`
- Fallback: OpenWeatherMap `api.openweathermap.org/data/2.5/weather` (butuh key), Nominatim `nominatim.openstreetmap.org`
