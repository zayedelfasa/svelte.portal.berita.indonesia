# Dokumentasi Fitur — Market Ticker + Bottom Navigation + Tentang Aplikasi

> Branch `dev` — update 2026-09-03
> Status: **Market legacy hidden; migrasi TradingView planned**. Phase 0/1 legacy terdokumentasi historis; implementasi baru wajib mengikuti `docs/PLAN_MARKET_TRADINGVIEW.md`.
> Status validasi legacy terakhir: `svelte-check 0 error`, `vite build pass`
> Legenda: ✅ Done · ⏳ Planned · 🔄 In Progress — lihat §11 Status Tracker

## 1. Ringkasan

Tiga fitur utama ditambahkan untuk memperluas portal dari **agregator berita** menjadi **portal berita + market**:

| Fitur | Route | Deskripsi |
|-------|-------|-----------|
| **Market Ticker** | Global (di `+layout`) | Bar scrolling di bawah Header menampilkan IHSG, LQ45, USD/IDR, BTC, ETH, SOL, BNB, USDT |
| **Bottom Navigation** | Global (di `+layout`) | Navigasi 3 tab fixed bottom: `Berita` `/` · `Market` `/market` · `Tentang` `/tentang` |
| **Halaman Market** | `/market` | Detail tabel Saham/Forex + Crypto Top 5 dengan harga & perubahan 24j |
| **Halaman Tentang** | `/tentang` | Info aplikasi, 3 fitur utama, teknologi, sumber data, versi |

Market legacy masih terdokumentasi untuk histori, tetapi hidden dari BottomNav. Target baru: TradingView Widgets read-only, client-only, tanpa server fetch/cache/API key. **Dilarang memakai dummy harga.** Detail migrasi ada di `docs/PLAN_MARKET_TRADINGVIEW.md`.

---

## 2. Latar Belakang & Keputusan Desain

### Kenapa Bottom Tab, bukan Drawer kiri?

| Kriteria | Drawer Kiri | **Bottom Tab (dipilih)** |
|----------|-------------|--------------------------|
| Discoverability | Tersembunyi, butuh 2 tap | Selalu terlihat, 1 tap |
| Jumlah menu | Cocok untuk >5 item | **Ideal untuk 3 item** (pattern Stockbit/Bibit/Indodax) |
| Thumb reach | Jauh dari jempol | Thumb-friendly di HP |
| Layout 420px | Sesak, tutup konten | Tidak bentrok dengan chip kategori horizontal |

Portal didesain mobile-first `max-w-[420px]` centered — Bottom Tab adalah standar native app untuk 2-5 menu utama.

### Urutan Tab

```
[ 📰 Berita ]  [ 📈 Market ]  [ ℹ️ Tentang ]
  / (aktif)      /market        /tentang
  + /baca        (tengah)       (kanan)
  + /cari
```

Market di tengah sebagai spotlight fitur baru.

---

## 3. Arsitektur

```
Browser
  │
  ├─ Vercel CDN (s-maxage=600)
  │
  └─ SvelteKit Server
       │
       ├─ +layout.server.ts ──► fetchMarketData() ──► cached('market:ticker')
       │                                              ├─ CoinGecko API ──► BTC/ETH/SOL/BNB/USDT
       │                                              └─ Yahoo Finance ──► ^JKSE (IHSG), LQ45, IDR=X
       │
       ├─ +page.server.ts (berita)  ──► Promise.allSettled 11 sumber (tetap)
       └─ /market/+page.server.ts  ──► fetchMarketData() reuse cache
```

**Prinsip reuse:**
- Cache memori `lib/server/cache.ts` TTL 10 menit — sama dengan berita
- `Promise.allSettled` — market gagal ≠ layout gagal
- Header `cache-control: public, s-maxage=600, stale-while-revalidate=1800` — hemat quota API publik
- Fetch server-side only — bebas CORS & aman dari rate-limit client

---

## 4. API & Data

### CoinGecko (Crypto — tanpa key)

```
GET https://api.coingecko.com/api/v3/coins/markets
  ?vs_currency=usd
  &ids=bitcoin,ethereum,solana,binancecoin,tether
  &order=market_cap_desc&per_page=5&page=1
  &sparkline=true&price_change_percentage=24h
```

| Field | Mapping |
|-------|---------|
| `id` | `bitcoin` → `BTC` / `ethereum` → `ETH` dst (`CRYPTO_MAP`) |
| `current_price` | `price` (USD) |
| `price_change_percentage_24h` | `change24h` (%) |

Limit gratis: ~10-50 req/menit. Dengan cache 10 menit + CDN = aman untuk traffic tinggi.

### Yahoo Finance (IDX & Forex — tanpa key)

```
GET https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1d&range=7d
Fallback host: query2.finance.yahoo.com; USD/IDR fallback: exchangerate.host
Symbols: ^JKSE (IHSG), IDR=X (USD/IDR), LQ45 (derived)
```

| Field | Mapping |
|-------|---------|
| `chart.result[0].meta.regularMarketPrice` | `price` |
| `chart.result[0].meta.previousClose` | hitung `change24h = (price-prev)/prev*100` |
| `meta.currency` | `currency` (IDR) |

**Fallback:** Jika Yahoo block (403/timeout), coba `query2`. USD/IDR mencoba `exchangerate.host`. Jika tetap gagal, gunakan stale cache maksimal 24 jam. Jika tidak ada stale cache, tampilkan pesan sopan dan jangan tampilkan angka perkiraan. LQ45 jika tidak ada di Yahoo → derived `~ IHSG * 0.135`, wajib diberi label `estimasi`.

### Interface

```ts
interface MarketItem {
  symbol: string;      // IHSG, LQ45, USD/IDR, BTC...
  name: string;        // IHSG, LQ45, USD/IDR, Bitcoin...
  price: number;
  change24h: number | null;
  currency: string;    // IDR | USD
  type: 'idx' | 'forex' | 'crypto';
  sparkline?: number[]; // close 7 hari, jika tersedia
}
interface MarketData {
  items: MarketItem[];
  fetchedAt: string;   // ISO
}
```

---

## 5. Komponen & File Changes

### File Baru (6)

| File | Fungsi |
|------|--------|
| `src/lib/server/market.ts` | Fetch & normalisasi CoinGecko + Yahoo, `fetchMarketData()` |
| `src/lib/components/MarketTicker.svelte` | Bar marquee gelap `bg-slate-900`, duplikasi `[...items, ...items]` untuk loop seamless, `hover:pause`, `prefers-reduced-motion` |
| `src/lib/components/BottomNav.svelte` | 3 tab fixed `bottom-0 left-1/2 -translate-x-1/2 max-w-[420px]`, icon SVG stroke dinamis `active ? 2.2 : 1.8`, `safe-area-inset-bottom` |
| `src/routes/+layout.server.ts` | Load `market` global, try/catch agar tidak crash |
| `src/routes/market/+page.server.ts` | Load `marketDetail` reuse cache |
| `src/routes/market/+page.svelte` | Tabel Saham/Forex + Crypto, `fmtPrice` IDR/USD, badge `bg-emerald-50`/`bg-red-50` |

| File | Fungsi |
|------|--------|
| `src/routes/tentang/+page.svelte` | Static page 5 card: Apa ini? / 3 Fitur / Teknologi / Sumber Data / Versi |

### File Dimodifikasi (1)

| File | Perubahan |
|------|-----------|
| `src/routes/+layout.svelte` | Import `MarketTicker` + `BottomNav`, `let { data }` props, render `<MarketTicker data={data?.market}>` di bawah `<Header />`, `main` tambah `pb-[calc(56px+env(safe-area-inset-bottom))]`, render `<BottomNav />` sebelum `</div>` |

### File Tidak Diubah (sengaja)

- `src/routes/+page.server.ts` — tetap fokus ke 11 sumber berita (market diambil via layout, tidak duplikasi)
- `src/routes/+page.svelte` — `Ticker` berita tetap di dalam page, tidak dipindah ke layout

---

## 6. UI/UX Detail

### MarketTicker

- Posisi: `sticky` tidak — diletak di layout sebagai `div` biasa di bawah `Header` (ikut scroll, tidak dobel-sticky dengan chip kategori)
- Style: `border-b border-slate-800 bg-slate-900 py-2` (dark mode `dark:bg-neutral-900`), kontras dengan News Ticker `bg-gray-50`
- Animasi: `@keyframes marquee-market 80s linear infinite`, `width: max-content`, `translateX(-50%)` untuk loop, `hover:[animation-play-state:paused]`
- Format: `IHSG 7.234 ▲1.23%` — IDR tanpa desimal ribuan, USD 2 desimal, crypto <1 → 4 desimal
- Aksesibilitas: `role="status" aria-label="Ticker saham dan crypto"`, `prefers-reduced-motion: none`

### BottomNav

- Layout: `fixed bottom-0 left-1/2 -translate-x-1/2 w-full max-w-[420px] z-20 border-t bg-white/95 backdrop-blur`
- Item: `flex-1 flex-col items-center py-2.5 text-[10px]`, active `text-red-500 font-bold`, inactive `text-gray-400`
- Icon: Berita (calendar+lines), Market (chart `3 3v18h18 + 7 16l4-4`), Tentang (info `circle+!`)
- Spacer: `<div h-[56px]>` + `main pb-[calc(56px+env(safe-area-inset-bottom))]` agar konten tidak ketutup

### Halaman /market

- Header: Judul + subtitle + `Diperbarui {timeAgo}`
- Section Saham: card `rounded-xl border bg-white px-4 py-3` per item, kanan badge `+2.34% 24j`
- Section Crypto: tambah avatar `h-8 w-8 rounded-full bg-slate-900` dengan 3 huruf symbol
- Footer note: `Sumber: Yahoo Finance & CoinGecko. Bukan rekomendasi investasi.`

### Halaman /tentang

- 5 card `rounded-xl border p-4` konsisten dengan design system portal
- List 3 fitur dengan emoji + bold, teknologi mention `s-maxage=600`, sumber data list disc, versi `v0.1.0 — branch dev`

---

## 7. Verifikasi

```bash
npm run check   # svelte-check found 0 errors and 0 warnings
npm run build   # 191 modules, 7.47s, PWA precache 31 entries
npm run dev     # cek manual:
                # /         → MarketTicker legacy muncul conditional, BottomNav 4 tab, Berita aktif
                # /market   → tabel market legacy; migrasi TradingView belum aktif
                # /tentang  → 5 card info
                # /baca?source=detik&... → BottomNav Berita tetap aktif
npm run preview # cek build produksi + safe-area di mobile
```

**Edge cases tested:**
- CoinGecko/Yahoo timeout → `Promise.allSettled` + fallback stale/empty jujur → tidak ada angka dummy
- Cache hit → ticker instan tanpa fetch ulang dalam 10 menit
- `prefers-reduced-motion` → animasi mati

---

## 8. Risiko & Mitigasi

| Risiko | Mitigasi |
|--------|----------|
| CoinGecko rate limit 429 | Cache 10m + CDN `s-maxage=600` → hit API < 6x/jam/instance |
| Yahoo 403 block | `query2` → Forex `exchangerate.host` → stale cache max 24j → empty state sopan |
| Layout double fetch (layout + /market) | Reuse key `market:ticker` — kedua route share cache yang sama |
| BottomNav tutup konten | `pb-[calc(56px+safe-area)]` di main + spacer div |

---

## 9. Roadmap Market — Phase 0-3 (dari PLAN_MARKET.md)

> Semua Phase tetap **gratis tanpa API key berbayar**. Cache split (`crypto 2m`, `IDX 15m`, `Forex 10m`), `Promise.allSettled`, fallback stale/empty jujur tanpa dummy.

### Phase 0 — Done ✅ (2026-01-04)

| Item | File | Status |
|------|------|--------|
| `fetchMarketData()` CoinGecko + Yahoo, `cached('market:ticker')`, fallback dummy | `src/lib/server/market.ts` | ✅ Done |
| `MarketTicker.svelte` marquee `bg-slate-900` 80s | `src/lib/components/MarketTicker.svelte` | ✅ Done |
| `BottomNav.svelte` 4 tab fixed `max-w-[420px]` safe-area | `src/lib/components/BottomNav.svelte` | ✅ Done |
| `+layout.server.ts` load market global | `src/routes/+layout.server.ts` | ✅ Done |
| `+layout.svelte` inject ticker + bottom nav | `src/routes/+layout.svelte` | ✅ Done |
| `/market` tabel saham/crypto + `/tentang` static | `src/routes/market/`, `src/routes/tentang/` | ✅ Done |
| Build `check 0 error`, `vite build` pass | — | ✅ Done |

### Phase 1 — Quick Win (1-2 hari) ✅ Done — Prioritas Tertinggi

> Goal: bikin user klik Market → betah, **tanpa tambah API baru** (reuse CoinGecko+Yahoo+berita pool).

#### 1.1 Detail Symbol `/market/[symbol]` ⏳

> **Tanpa TradingView** — tidak embed widget chart (diputuskan). Chart di sini pakai **Sparkline 7d** yang sudah ada (`Sparkline.svelte`), cukup tanpa API/embed eksternal.

- Route: `src/routes/market/[symbol]/+page.server.ts` + `+page.svelte`
- Param: `btc`, `eth`, `ihsg`, `usd-idr` (lowercase dari `MarketItem.symbol`)
- Server: `find` dari `fetchMarketData()` reuse cache, `404` jika tidak ketemu
- UI: Header symbol + price + badge 24h + **Sparkline 7d** (reuse `Sparkline.svelte`), tombol balik, `Diperbarui {timeAgo}`
- Acceptance: `/market/btc` & `/market/ihsg` render detail + sparkline, dark mode ikut theme

#### 1.2 Top Gainer / Loser + Trending Strip ⏳

- Data: sort `items` by `change24h` desc/asc (CoinGecko sudah ada), trending via CoinGecko `/search/trending` cache `market:trending` TTL 1 jam
- UI `/market`: 2 card horizontal scroll di atas tabel — `🔥 Top Gainer` (3 hijau) + `💧 Top Loser` (3 merah) + `⭐ Trending`
- File: tambah `fetchTrending()` di `market.ts`

#### 1.3 Kalkulator Lot & Converter ⏳

- Lokasi: Card di `/market` bawah tabel + di `/market/[symbol]`
- Logika client only: saham `1 lot=100 lembar → Rp price*100`, crypto `IDR = USD * kurs USD/IDR`
- Komponen: `src/lib/components/MarketCalculator.svelte` (props `item`, `usdIdr`)

#### 1.4 Auto-Tag Berita Market ⏳

- Logika: regex judul+summary (`BBCA→BTC`, `Bitcoin→BTC`, `IHSG→IHSG` ...) di home & `/market/[symbol]`
- File: `src/lib/marketTag.ts` pure `tagArticle(article, marketItems)`
- UI: badge `rounded-full bg-slate-900 text-[10px]` di `NewsItem`/`ArticleView`, klik → `/market/[symbol]`

**Deliverable Phase 1:** 4 fitur, 0 API baru — ✅ selesai (2026-01-05). TradingView tidak dipakai (keputusan) — chart pakai Sparkline 7d yang sudah ada.

### Phase 2 — Medium (3-5 hari) ⏳ Planned — Retention

#### 2.1 Watchlist ⭐ ⏳

- Pola: ikut `bookmarks.svelte.ts` — `src/lib/utils/watchlist.svelte.ts` `$state<string[]>`, `isWatched()`, `toggleWatch()`, `storage` sync, max 20
- UI: tombol `☆/★` di tiap row `/market` + header `/market/[symbol]`, section `Watchlist Saya` di atas `/market`, badge count di BottomNav

#### 2.2 Fear & Greed + Dominance + Market Cap ⏳

- API gratis tanpa key: `api.alternative.me/fng/?limit=1` + `api.coingecko.com/api/v3/global`
- Server: `fetchMarketInsight()` cached `market:insight` TTL 1 jam
- UI `/market`: 3 kolom `[FNG 72 Greed] [BTC Dom 52%] [MCap $2.4T]` + gauge, fallback hidden jika gagal

#### 2.3 Kurs Lengkap + Emas Antam ⏳

- Kurs Yahoo `JPY=X,EURIDR=X,SGDIDR=X,CNYIDR=X` tambah ke `YAHOO_SYMBOLS`, section `Kurs & Emas` 2 kolom
- Emas: CoinGecko `ids=gold` atau fallback static `Rp 1.45jt/gram`

#### 2.4 Heatmap LQ45 ⏳

- Data: Yahoo batch 10 sample dulu (BBCA,BBRI,BMRI,TLKM,ASII,UNVR,GOTO...) `Promise.allSettled`, scale ke 45 nanti
- UI: `src/lib/components/MarketHeatmap.svelte` grid 5×9 kotak hijau/merah by `change24h`, tooltip `BBCA +1.2%`

#### 2.5 Filter Tab Market ✅

- UI chip `[Semua] [IDX] [Crypto] [Forex]` filter client `items.filter(type)`, `$state<'all'|'idx'|'crypto'|'forex'>`
- Badge tipe, empty state per filter, sort nama/harga/perubahan 24j dengan panah `↑↓`

**Deliverable Phase 2:** watchlist persist + insight + kurs + heatmap — ⏳ belum dikerjakan

### Phase 3 — Advanced (1-2 minggu) ⏳ Planned — Power User

#### 3.1 Sparkline 7 Hari ✅

- Yahoo dan CoinGecko `sparkline=true` → `sparkline` 7 titik close/harga, render mini SVG tanpa library di `Sparkline.svelte`

#### 3.2 Economic Calendar ⏳

- **Pilih:** TradingView Economic Calendar widget (free embed, 0 quota) dulu, Finnhub `calendar/economic` nanti jika custom
- UI: section bawah heatmap 5 event terdekat

**🗑 Dihapus (diputuskan)** — tidak dikerjakan. TradingView embed tidak dipakai di project ini.

#### 3.3 Berita Terkait per Symbol ⏳

- Filter pool berita 100/media (reuse `cari` logic) by keyword symbol → `related: Article[]` max 5 di `/market/[symbol]` bawah chart

#### 3.4 Portfolio Simulasi (Paper) ⏳

- Storage localStorage `portfolio: {symbol,qty,avgPrice}[]` — `src/lib/utils/portfolio.svelte.ts` + `src/routes/market/portfolio/+page.svelte`, hitung PnL client dari `MarketData`

#### 3.5 Alert Harga (PWA Push - Opsional) ⏳

- `src/lib/utils/alerts.svelte.ts` `setInterval` 60s cek `MarketData` → `Notification`, max 5 alert, permission eksplisit

**Deliverable Phase 3:** sparkline + calendar + berita terkait + portfolio — ⏳ belum dikerjakan

### Arsitektur Baru (Phase 1-3)

```
+layout.server.ts (market:ticker) ──┐
/market/+page.server.ts (ticker + insight + trending) ──┤
/market/[symbol]/+page.server.ts (ticker + related)     ├─► market.ts
                                                         ├─ market:trending (1 jam)
                                                         ├─ market:insight (1 jam)
                                                         └─ market:sparkline (10m)
Client: watchlist.svelte.ts, portfolio.svelte.ts, alerts.svelte.ts (localStorage)
```

| Key | TTL | Isi |
|-----|-----|-----|
| `market:ticker` | 10m | IHSG/LQ45/USDIDR + 5 crypto (ada) — ✅ |
| `market:trending` | 1 jam | Trending coins — ✅ |
| `market:insight` | 1 jam | FNG + global — ⏳ |
| `market:sparkline` | ikut cache market | 7d price array — ✅ melalui `MarketItem.sparkline` |

### Urutan Eksekusi Rekomendasi

1. Phase 1.1 + 1.2 (detail + gainer) — unlock navigasi
2. 1.3 + 1.4 (kalkulator + tag) — hubungkan berita
3. 2.1 watchlist — retention
4. 2.2 + 2.3 insight + kurs
5. 2.4 + 2.5 heatmap + filter
6. Phase 3 sesuai kebutuhan user

---

## 10. Referensi

- CoinGecko API: `https://api.coingecko.com/api/v3/coins/markets`
- Yahoo Finance: `https://query1.finance.yahoo.com/v8/finance/chart/^JKSE`
- Alternative.me FNG: `https://api.alternative.me/fng/`
- `ARCHITECTURE.md` (root) — pola cache & adapter factory
- `AGENTS.md` (root) — konteks bisnis+teknis umum
- `docs/PLAN.md` — registry 11 media & resiliensi
- `docs/PLAN_CUACA.md` — Cuaca & Polusi
- `docs/PLAN_FITUR_HARIAN.md` — Daily habit

---

## 11. Status Tracker (Cepat Lihat)

| Phase | Fitur | Status | File Kunci |
|-------|-------|--------|------------|
| **0** | MarketTicker legacy global | ✅ Done historis / hidden | `market.ts`, `MarketTicker.svelte` |
| **0** | BottomNav 4 tab (`Berita|Cuaca|Harian|Tentang`) | ✅ Done | `BottomNav.svelte`, `+layout.svelte` |
| **0** | `/market` tabel legacy | ✅ Done historis / hidden | `market/+page.*` |
| **0** | `/tentang` static | ✅ Done | `tentang/+page.svelte` |
| **1.1** | Detail `[symbol]` + sparkline (tanpa TradingView) | ✅ Done | `market/[symbol]/+page.*` |
| **1.2** | Gainer/Loser + Trending | ✅ Done | `market.ts` `market:trending` |
| **1.3** | Kalkulator Lot/Converter | ✅ Done | `MarketCalculator.svelte` |
| **1.4** | Auto-tag berita | ✅ Done | `marketTag.ts`, `NewsItem` |
| **2.1** | Watchlist ⭐ | ⏳ Planned | `watchlist.svelte.ts` |
| **2.2** | Fear & Greed + Dominance | ⏳ Planned | `market:insight` |
| **2.3** | Kurs lengkap + Emas | ⏳ Planned | `YAHOO_SYMBOLS` |
| **2.4** | Heatmap LQ45 | ⏳ Planned | `MarketHeatmap.svelte` |
| **2.5** | Filter tab Market + sort | ✅ Done | `/market` chip, sort controls |
| **3.1** | Sparkline 7d | ✅ Done | `Sparkline.svelte`, `sparkline` |
| **3.2** | Economic Calendar (TradingView widget) | 🗑 Dihapus | — |
| **3.3** | Berita terkait/symbol | ⏳ Planned | `related: Article[]` |
| **3.4** | Portfolio paper | ⏳ Planned | `portfolio.svelte.ts` |
| **3.5** | Alert harga | ⏳ Planned | `alerts.svelte.ts` |

> Update tracker ini tiap selesai 1 fitur: ganti ⏳ → ✅ (atau 🔄 jika in progress). `PLAN_MARKET.md` sudah di-merge ke sini dan dihapus.
