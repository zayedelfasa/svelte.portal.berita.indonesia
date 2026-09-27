# DOC: JANGAN GUNAKAN DUMMY — Kebijakan Data Market

> **Prinsip:** Lebih baik kosong jujur daripada angka palsu kelihatan live.
> Berlaku untuk legacy market files + `src/lib/server/harga.ts` + `src/lib/server/cache.ts` + `src/routes/market/*` + `MarketTicker.svelte` + `HargaCard`. Market legacy hidden; target migrasi TradingView client-only sesuai `docs/PLAN_MARKET_TRADINGVIEW.md`.

## 1. Masalah Dummy Lama

```ts
// ❌ DILARANG — dihapus 2026-08-26
if (items.length === 0) return [
  { symbol: 'IHSG', price: 7234.5, change24h: 0.85, currency:'IDR', type:'idx' },
  { symbol: 'USD/IDR', price: 16220, change24h: 0.12, currency:'IDR', type:'forex' }
];
```
- Angka hardcode `7234.5` seolah live, padahal statis.
- `fetchedAt = now()` → tidak trigger banner `isStale >30m`, user tertipu.
- Risiko komplain + OJK disclaimer jebol.

## 2. Kebijakan Baru

### Aturan
1. **Jangan hardcode harga** di `market.ts`. Hanya `isEstimated: true` untuk LQ45 derived yang diberi label `est`.
2. **Gagal = kosong atau stale sopan.** Urutan fallback:
   ```
   fetch Yahoo query1 → query2 → USD/IDR fail → exchangerate.host/convert (0-key)
   → Promise.allSettled → items=[] ? cek peekCache('market:ticker') stale <24j
   → ada stale → return stale (fetchedAt lama → banner sopan "Menampilkan data terakhir")
   → tidak ada stale → return { items: [], fetchedAt: now } → UI honest empty sopan
   ```
3. **UI jujur bahasa sopan:** legacy `/market` empty → `Data market sementara tidak tersedia — kami tidak menampilkan angka perkiraan.` + tombol `Muat ulang`. `MarketTicker` hidden kalau `items=[]`. Setelah migrasi, TradingView widget wajib tetap read-only + fallback error-state.
4. **Cache jujur TTL split:** `cache.ts` `TTL.crypto 2m`, `TTL.idx 15m`, `TTL.forex 10m`, `cached(key,fn,ttl)` + `peekCache()` lastGood, `cached()` try/catch fallback ke stale. Jangan delete stale saat TTL habis.

### Yang Masih Boleh
- `LQ45 estimasi` dari `IHSG*0.135` → wajib `isEstimated:true` + badge `est` + name `LQ45 (estimasi)`.
- Legacy CoinGecko fail → `fetchCrypto()` return `[]`, bukan dummy crypto. TradingView widget tidak boleh diberi angka fallback hardcode.

## 3. Implementasi Sekarang

**`cache.ts`**
- `TTL = { crypto:2m, idx:15m, forex:10m }` + `cached(key,fn,ttlMs)`
- `export function peekCache<T>(key)` — ambil stale tanpa cek TTL (max 24j di market.ts)
- `cached()` wrap `fn()` dalam `try/catch`, kalau throw dan ada `hit` → return `hit.value`

**`market.ts`**
- `fetchCrypto()` → `cached('market:crypto', fn, TTL.crypto)` 2m
- `fetchForexFallback()` → `api.exchangerate.host/convert?from=USD&to=IDR` 0-key kalau Yahoo USD/IDR 403 → `change:null`
- `fetchYahooItem()` loop `hosts=[query1, query2]` — layer 2, kalau USD/IDR masih fail → coba `fetchForexFallback()`
- `fetchIdxForex()` → `cached('market:idx', fn, TTL.idx)` 15m, jika `items.length===0` → `return []` (hapus dummy)
- `fetchMarketData()` → `cached('market:ticker', fn, TTL.crypto)` + cek `peekCache` stale <24j

**`+page.svelte`**
- Empty: card slate sopan `Data market IHSG/LQ45 sementara tidak tersedia — Sumber Yahoo Finance sedang sibuk, kami tidak menampilkan angka perkiraan. Silakan muat ulang beberapa saat lagi.`
- Stale: banner amber sopan `Menampilkan data terakhir — diperbarui X — Sumber Yahoo Finance / CoinGecko sedang sibuk. Data adalah cache terakhir.`

**`MarketTicker.svelte`**
- Tetap hidden jika `items.length===0`, tidak paksa dummy

## 4. Curl Test Jujur

```bash
# Cek Yahoo — kalau 403 harus kosong, bukan 7234
curl -s -L -A "Mozilla/5.0 ..." -H "Accept: application/json" \
  "https://query1.finance.yahoo.com/v8/finance/chart/%5EJKSE?interval=1d&range=2d" | jq . 2>&1 | head

# Cek portal — force refresh, harus lihat honest empty jika Yahoo block
curl -s "http://localhost:5173/api/source/idx?force=1" | jq .
```

## 5. Checklist PR

- [ ] `grep -r "7234.5" src/` = 0 hasil
- [ ] `grep -r "dummy" docs/` hanya di doc ini
- [ ] `npm run check` 0 error
- [ ] Test: matikan internet → `/market` tampil card jujur, bukan angka

## 6. Harga Harian — pangan.go.id Dihapus (2026-09-01)

`api.pangan.go.id/api/harga` dihapus selamanya: 95% `null` + timeout 7s → `fetchSembako()` + `SEMBAKO_NAMA` + `parsePanganPrice` usage dihapus. Sembako Rp/kg diganti `🔥 Tren Sembako (7 hari, geo ID)` skor 0-100 via Google Trends embed iframe (B1, no key, no 429) — jujur karena harga sembako beda per daerah (Jakarta vs NTT beda 4rb), tren selalu ada data. Logam: `pax-gold` + `kinesis-silver` (fallback `tether-silver`/`silver-token`) → `parseLogamPrice` per-gram `est`. BBM tambah LPG 3kg/12kg statis. Footer wajib: `Skor 0-100 = minat pencarian, bukan harga Rp. Beda daerah beda harga.`

## 7. Roadmap

- Phase selanjutnya: tambah `Investing`/`kuarsa` sebagai layer 3, tetap tanpa dummy.
- Jika butuh angka fallback, pakai `lastGood` stale + label `Data 45m lalu` + banner, bukan hardcode.
- Harga Phase 2 opsional: `trends.ts` fetch skor JSON `cached('trends:sembako:{date}',6j)` jika embed kurang custom.

---
Ref: `AGENTS.md §5` cache pattern, `ARCHITECTURE.md`, `DOC_FITUR_MARKET_TENTANG.md` Gap A2.
