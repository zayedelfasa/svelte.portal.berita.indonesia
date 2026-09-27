# PLAN Market — TradingView Embed (READ ONLY)

> Status: **NEXT / belum dikerjakan** — plan diperbarui 2026-09-03. Market legacy masih hidden; jangan implement provider custom baru.
> Sumber market baru: **TradingView Widget (embed) — gratis, tanpa API key, READ ONLY**
> BottomNav: kembali 4 tab `Berita | Cuaca | Market | Tentang` — Market aktif lagi (sebelumnya hidden)

## 1. Ringkasan Sederhana

Market kemarin mati karena Yahoo diblokir. Solusi baru: jangan ambil data sendiri, tapi tampilkan widget dari TradingView.

- Tidak perlu server fetch
- Tidak perlu cache 10 menit
- Tidak perlu API key
- User cuma lihat harga, tidak bisa trading (READ ONLY)

Isi awal: **IHSG, USD/IDR, BTC, ETH, SOL** — 5 simbol utama. Ada tombol **"Lihat lainnya →"** untuk buka daftar lengkap di `/market`.

## 2. Kenapa Hapus Yahoo?

| Masalah Yahoo | Solusi TradingView |
|---|---|
| Sering 403 / diblokir | Widget TradingView tidak pernah 403 |
| Butuh `query1` + `query2` + fallback `exchangerate.host` | Tidak butuh fallback |
| Perlu cache 10m + stale 24j | Tidak perlu cache, widget update otomatis |
| `LQ45` harus estimasi `IHSG*0.135` | TradingView ada data asli `IDX:COMPOSITE`, `IDX:LQ45` (jika ada) |

Keputusan: **Hapus semua kode Yahoo** di `src/lib/server/market.ts` (fungsi `fetchTwelveItem`, `fetchIdxForex`, `YAHOO_SYMBOLS`, `query1/2`). CoinGecko boleh keep atau hapus juga — untuk Phase TradingView, semua via TV saja biar simpel.

## 3. Fitur Market Baru (READ ONLY)

### A. Ticker Tape — di semua halaman (global)
Posisi: di `+layout.svelte` bawah `Header`
- Widget: `embed-widget-ticker-tape.js`
- Simbol: `IDX:COMPOSITE` (IHSG), `FX_IDC:USDIDR`, `BINANCE:BTCUSDT`, `BINANCE:ETHUSDT`, `BINANCE:SOLUSDT`
- Jalan otomatis kiri ke kanan
- Transparan, support dark mode `colorTheme: light | dark`

### B. Halaman /market — daftar lengkap
Posisi: `src/routes/market/+page.svelte`
- Widget: `embed-widget-market-overview.js` atau `embed-widget-screener.js` (pilih 1)
- Tab di dalam widget: IDX | Forex | Crypto
- Isi tab IDX: IHSG, LQ45, BBCA, BBRI, BMRI, TLKM
- Isi tab Forex: USD/IDR, EUR/IDR, JPY/IDR, SGD/IDR
- Isi tab Crypto: BTC, ETH, SOL, BNB, USDT
- Ada tombol **"Lihat lainnya"** di bawah widget → link ke TradingView screener eksternal (opsional)

### C. Halaman Detail /market/[symbol] — grafik kecil
Posisi: `src/routes/market/[symbol]/+page.svelte`
- Widget: `embed-widget-mini-chart.js` atau `embed-widget-symbol-overview.js`
- Param: `btc` → `BINANCE:BTCUSDT`, `ihsg` → `IDX:COMPOSITE`
- Tampilkan harga + chart 1D/7D + perubahan 24j
- Tombol kembali ke `/market`

### D. Tombol "Lihat lainnya"
- Di Ticker Tape: tombol kecil di kanan `Lihat lainnya →` link `/market`
- Di /market: link di bawah widget `Buka di TradingView →` (buka tab baru, opsional)
- Tidak perlu load semua simbol di ticker, cukup 5 utama biar HP tidak berat

## 4. Arsitektur Baru (Tanpa Server Fetch)

```
Sebelum (Yahoo):
Browser → +layout.server.ts → fetchMarketData() → CoinGecko + Yahoo (403) → cache 10m → MarketTicker

Sesudah (TradingView):
Browser → +layout.svelte → <TradingViewTicker /> → script TradingView langsung (client only)
         → /market/+page.svelte → <TradingViewMarket /> → widget market-overview
         → /market/[symbol] → <TradingViewMiniChart symbol="BINANCE:BTCUSDT" />
```

- Tidak ada `cached('market:ticker')` lagi untuk market (tetap ada untuk berita & cuaca)
- Tidak ada `setHeaders s-maxage` untuk market
- `+layout.server.ts` tidak perlu load market lagi (hapus try/catch fetchMarketData)
- Widget update realtime dari TradingView, bukan dari server kita

## 5. File yang Diubah / Dibuat

| File | Aksi | Keterangan |
|---|---|---|
| `src/lib/components/TradingViewTicker.svelte` | **BARU** | Ticker tape global, props `theme`, inject script via `onMount` + `$effect` |
| `src/lib/components/TradingViewMarket.svelte` | **BARU** | Market overview untuk /market, props `theme` |
| `src/lib/components/TradingViewMiniChart.svelte` | **BARU** | Mini chart untuk /market/[symbol], props `symbol`, `theme` |
| `src/lib/server/market.ts` | **HAPUS / KOSONGKAN** | Hapus Yahoo, TwelveData, exchangerate.host. Sisa CoinGecko opsional atau hapus total |
| `src/routes/+layout.svelte` | **EDIT** | Ganti `<MarketTicker>` → `<TradingViewTicker>` |
| `src/routes/+layout.server.ts` | **EDIT** | Hapus `fetchMarketData()` |
| `src/routes/market/+page.svelte` | **EDIT** | Ganti tabel custom → `<TradingViewMarket>` + tombol Lihat lainnya |
| `src/routes/market/+page.server.ts` | **EDIT** | Hapus load market, jadi static atau kosong |
| `src/lib/components/BottomNav.svelte` | **EDIT** | Tambah tab Market kembali → 4 tab: Berita, Cuaca, Market, Tentang |
| `src/lib/components/MarketTicker.svelte` | **HAPUS** | Tidak dipakai lagi |
| `src/lib/components/MarketCalculator.svelte` | **KEEP** | Opsional, bisa dipakai lagi nanti |

## 6. Contoh Component (Svelte 5 Runes)

`TradingViewTicker.svelte` — simpel, client only:
```svelte
<script lang="ts">
  import { browser } from '$app/environment';
  let { theme = 'light' }: { theme: 'light' | 'dark' } = $props();
  let container: HTMLDivElement | undefined = $state(undefined);
  let loaded = $state(false);
  const cfg = $derived({
    symbols: [
      { proName: "IDX:COMPOSITE", title: "IHSG" },
      { proName: "FX_IDC:USDIDR", title: "USD/IDR" },
      { proName: "BINANCE:BTCUSDT", title: "BTC" },
      { proName: "BINANCE:ETHUSDT", title: "ETH" },
      { proName: "BINANCE:SOLUSDT", title: "SOL" }
    ],
    colorTheme: theme, isTransparent: false, displayMode: "adaptive", locale: "id"
  });
  $effect(() => {
    void cfg.colorTheme;
    if (!browser || !container || loaded) return;
    container.innerHTML = '';
    const s = document.createElement('script');
    s.src = 'https://s3.tradingview.com/external-embedding/embed-widget-ticker-tape.js';
    s.async = true;
    s.innerHTML = JSON.stringify(cfg);
    container.appendChild(s);
    loaded = true;
  });
</script>
<div class="tradingview-widget-container">
  <div bind:this={container}></div>
</div>
<a href="/market" class="mt-2 block text-center text-xs font-bold text-blue-600">Lihat lainnya →</a>
```

## 7. BottomNav 4 Tab (Aktif Lagi)

```
[ Berita ] [ Cuaca ] [ Market ] [ Tentang ]
   /         /cuaca    /market    /tentang
```

- Market icon: chart `M3 3v18h18 + 7 16l4-4`
- Active: `text-red-500 font-bold`, inactive `text-gray-400`
- `max-w-[420px]` tetap, tidak diubah

## 8. Urutan Kerja (Step by Step)

1. Buat 3 component TradingView baru (Ticker, Market, MiniChart)
2. Edit `BottomNav.svelte` → tambah Market tab
3. Edit `+layout.svelte` → ganti MarketTicker
4. Edit `+layout.server.ts` → hapus fetchMarketData
5. Edit `/market` page → ganti tabel → widget + tombol Lihat lainnya
6. Hapus `market.ts` Yahoo & `MarketTicker.svelte` lama
7. Test: `npm run check` 0 error, `npm run build` pass, cek dark mode

## 9. Catatan Penting

- TradingView gratis, READ ONLY, wajib tampilkan logo/attribution (jangan di-hide)
- Tidak perlu API key, tidak perlu cache
- Widget load via script eksternal → perlu internet, lazy load biar tidak berat
- Untuk kepatuhan: tambahkan footer note `Data dari TradingView. Bukan rekomendasi investasi.`
- Jika TradingView lambat/error → tampilkan fallback `Data sementara tidak tersedia — Muat ulang`

## 10. Referensi Widget

- Ticker Tape: https://www.tradingview.com/widget/ticker-tape/
- Market Overview: https://www.tradingview.com/widget/market-overview/
- Mini Chart: https://www.tradingview.com/widget/mini-chart/
- Symbol Overview: https://www.tradingview.com/widget/symbol-overview/

> Yahoo Finance, TwelveData, exchangerate.host → HAPUS total. Tidak dipakai lagi di plan ini.
