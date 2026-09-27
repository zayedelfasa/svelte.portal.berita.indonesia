# src/lib — Kode Reusable

Semua file di sini bisa di-import via alias `$lib/...` (dikonfigurasi SvelteKit).

## Struktur

| Path | Fungsi |
|---|---|
| `components/` | Komponen UI Svelte — [README](components/README.md) |
| `config/` | Registry konfigurasi — [README](config/README.md) |
| `server/` | Kode **server-side only** — [README](server/README.md) |
| `utils/` | State global runes + helper client — [README](utils/README.md) |
| `types.ts` | Interface inti: `Article`, `SourceDef`, `SourceAdapter`, `SourceResult`, `CategoryId` |
| `server/market.ts` | Legacy market: `MarketItem`, `MarketData`, `fetchMarketData()` — hidden; jangan dikembangkan sebelum migrasi TradingView |
| `server/weather.ts` | Cuaca: `WeatherData`, `AirQualityData`, `GeoCity`, `fetchWeather()`, `fetchAirQuality()`, `searchCity()`, `reverseGeocode()` — Open-Meteo gratis unlimited |
| `weatherCode.ts` | Mapping WMO `weather_code → {label, icon}` bahasa Indonesia |
| `time.ts` | `timeAgo(iso, now)`, `isNew(iso, now)` — pure function, aman SSR |
| `categories.ts` | 6 kategori kanonik + guard `isCategoryId()` |

## ⚠️ Aturan Penting

**Jangan import `$lib/server/*` dari komponen client atau file non-server.**
Folder `server/` berisi logika fetch upstream, parser XML, dan cache yang hanya boleh
dieksekusi di server (`+page.server.ts`, `+server.ts`). Import dari komponen akan
membocorkan kode/kredensial ke bundle browser.

Alur data yang benar:
```
komponen → props (data dari load function) ✅
komponen → $lib/server/... ❌
```

## Konvensi

- File state reaktif global bernama `*.svelte.ts` (runes `$state` bisa dipakai di luar komponen)
- Semua helper waktu menerima parameter `now` agar bisa mengikuti ticker global (`utils/clock.svelte.ts`)
- Tipe `Article` adalah kontrak tunggal antara server (adapter) dan UI
