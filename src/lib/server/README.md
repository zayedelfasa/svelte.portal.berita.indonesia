# server/ — Logika Server-Side Only

**Jangan import folder ini dari komponen client.** Semua kode di sini hanya boleh
dieksekusi dari `+page.server.ts`, `+server.ts`, atau file lain di dalam `server/`.

## File

| File | Fungsi |
|---|---|
| `http.ts` | `fetchWithTimeout(url, opts, ms=7000-8000)` AbortController + UA browser; `stripHtml()`; `firstImgSrc()` |
| `rss.ts` | `parseRss(xml, sourceId)` — fast-xml-parser, `removeNSPrefix`, ekstrak enclosure → image |
| `aggregator.ts` | `fetchAggregator(path, id)` — JSON berita-indo-api → `Article[]`, fallback RSS bila mati |
| `cache.ts` | Map memori: `cached(key, fn, ttl)`, `peekCache(key)`, `invalidateCache(prefix)` + `TTL` split berita 10m, crypto 2m, idx 15m, forex 10m, trending 1j, weather 10m, geo 1j, reverse 1d, Harian: gempa/bola 5m, harga 6j, hijri 12j, libur 24j |
| `market.ts` | **Legacy Market** `fetchMarketData(): Promise<MarketData>` — hidden; jangan tambah provider. Target migrasi: TradingView client-only sesuai `docs/PLAN_MARKET_TRADINGVIEW.md` |
| `weather.ts` | **Cuaca** `fetchWeather(lat,lon): WeatherData`, `fetchAirQuality(lat,lon): AirQualityData`, `searchCity(q): GeoCity[]`, `reverseGeocode(lat,lon): string` — Open-Meteo gratis unlimited, `cached('weather:*')` + `fetchWithTimeout(7000)` |
| `parsers.ts` | Pure parser BMKG/ESPN/TheSportsDB/PAXG/Pangan — unit-tested tanpa fetch live; ESPN `team.logo` → `BolaMatch.homeLogo/awayLogo` |
| `bola.ts` | **Bola** `fetchBola({week}): BolaData` — 7 liga ESPN (UA `axios/1.7.0`) + TheSportsDB 4790 fallback Liga 1, `?dates=YYYYMMDD` week Mon-Sun, dedup Map, `cached('bola:scoreboard'+':week')` TTL 5m |
| `sources/` | Adapter 11 media — [README](sources/README.md) |
| `../weatherCode.ts` | Mapping WMO `weather_code → {label, icon}` ID (dipakai `WeatherCard`/`ForecastStrip`) |

## Alur request

```
load berita (+page.server.ts)
  → cached('rss:detik', fn) → fetchWithTimeout → parseRss/fetchAggregator → Article[]

load market legacy (+layout.server.ts & /market), saat ini hidden
  → cached('market:ticker') → legacy fetch; target berikutnya ganti TradingView Widgets client-only tanpa server cache

load cuaca (/cuaca/+page.server.ts)
  → Promise.allSettled([fetchWeather(lat,lon), fetchAirQuality(lat,lon), reverseGeocode(lat,lon)])
      → cached('weather:current:{lat,lon}') → api.open-meteo.com/v1/forecast
      → cached('weather:air:{lat,lon}')     → air-quality-api.open-meteo.com
      → cached('weather:geo:{q}')           → geocoding-api.open-meteo.com/v1/search
      → cached('weather:reverse:{lat,lon}') → geocoding-api.open-meteo.com/v1/reverse
      → tanpa setHeaders (hindari double dengan layout, s-maxage via +layout.server.ts)
```

## Cache

- TTL split: berita 10m, crypto 2m, idx 15m, forex 10m, trending 1h, weather 10m, geo 1h, reverse 1d; prune >200 entri
- Keys: `rss:{id}`, `agg:{id}`, `market:ticker`, `market:trending`, `weather:current:{lat,lon}`, `weather:air:{lat,lon}`, `weather:geo:{q}`, `weather:reverse:{lat,lon}`
- `peekCache` baca stale tanpa reset TTL (dipakai market fallback 24j)
- Per-instance serverless + CDN `s-maxage=600` (`+layout.server.ts` set header, child pages reuse tanpa set ulang); `?force=1` = `no-store` + invalidasi cache fitur

## Menambah helper server

1. Pure function → `http.ts`
2. Butuh cache → `cached(key, fn, TTL.xxx)` dari `cache.ts`
3. Return tipe ternormalisasi (`Article[]` atau `WeatherData`) — jangan bocorkan mentah ke route
