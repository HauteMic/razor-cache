# razor-cache

Historical intraday market data cache for the Razor stock trading simulator. Hosted at [github.com/HauteMic/razor-cache](https://github.com/HauteMic/razor-cache). Each file contains one trading day of 1-minute OHLCV bars for a single ticker, stored as JSON.

---

## Repository Layout

This repo contains `cache_snp500.zip` (802 MB). The remaining datasets exceed GitHub's file size limits and are hosted on Google Drive -- see [Downloads](#downloads) below.

```
razor-cache/
├── cache_snp500.zip   # 503 tickers · 2024-05-10 → 2026-04-30 · 493 trading days · schema v2 · 802 MB
├── README.md
└── .gitignore
```

| Dataset | Index / Universe | Schema | Size | Location |
|---|---|---|---|---|
| `cache_snp500.zip` | S&P 500 (503 tickers) | v2 | 802 MB | This repo |
| `cache_og_298.zip` | Original 298-ticker universe | v2 | 537 MB | Google Drive |
| `cache_ndx.zip` | NASDAQ-100 (101 tickers) | v2 | 194 MB | Google Drive |
| `cache_apex.zip` | Hand-picked legacy (13 tickers) | v2 | 95 MB | Google Drive |
| `cache_djia.zip` | Dow Jones Industrial Average (30) | v2 | 68 MB | Google Drive |
| `cache.zip` | Mixed / recent (352 tickers) | v1 | ~100 MB | Google Drive |

---

## Downloads

**`cache_snp500.zip`** is available directly from this repository.

All other datasets were moved to Google Drive due to GitHub file size constraints:

> [Google Drive -- Razor Cache Files](https://drive.google.com/drive/folders/1dfzLv8uZKAxCpWsHo57wPveT9RfawPGj?usp=sharing)

Each zip extracts to a flat folder of `{TICKER}_{YYYY-MM-DD}.json` files matching the schema described below.

---

## File Naming Convention

```
{TICKER}_{YYYY-MM-DD}.json
```

Examples:
- `AAPL_2025-06-02.json` -- Apple, June 2 2025
- `MSFT_2024-11-15.json` -- Microsoft, Nov 15 2024

Files are only present for **market trading days** (NYSE calendar). Weekends, federal holidays, and early-close half-days have no file.

---

## Data Schemas

There are two schema variants. The only difference is the presence of the `meta` block.

### Schema v2 (standard -- all directories except `cache`)

```json
{
  "symbol": "AAPL",
  "date": "2025-06-02",
  "fetchedAt": "2026-04-30T05:06:49.584Z",
  "meta": {
    "currency": "USD",
    "exchangeName": "NasdaqNM",
    "regularMarketPrice": 201.45
  },
  "bars": [
    {
      "time": 1748872200000,
      "open": 201.10,
      "high": 201.75,
      "low": 200.98,
      "close": 201.45,
      "volume": 284310
    }
  ]
}
```

### Schema v1 (legacy -- `cache` directory only)

Identical to v2 but without the `meta` block:

```json
{
  "symbol": "AAL",
  "date": "2026-03-16",
  "fetchedAt": "2026-04-13T22:51:19.545Z",
  "bars": [
    {
      "time": 1773667800000,
      "open": 10.41,
      "high": 10.51,
      "low": 10.41,
      "close": 10.49,
      "volume": 1423554
    }
  ]
}
```

---

## Field Reference

### Top-level fields

| Field | Type | Description |
|---|---|---|
| `symbol` | `string` | Ticker symbol (uppercase, e.g. `"AAPL"`) |
| `date` | `string` | Trading date in `YYYY-MM-DD` format (local market date, US Eastern) |
| `fetchedAt` | `string` | ISO 8601 UTC timestamp of when the data was fetched from the upstream source |
| `meta` | `object` | *(v2 only)* Security metadata at fetch time -- see below |
| `bars` | `Bar[]` | Ordered array of 1-minute OHLCV candles, earliest first |

### `meta` object (v2 only)

| Field | Type | Description |
|---|---|---|
| `currency` | `string` | ISO 4217 currency code (always `"USD"` for US equities) |
| `exchangeName` | `string` | Exchange name string from the upstream provider (may be empty) |
| `regularMarketPrice` | `number` | Last regular-market close price at the time of fetch |

### `Bar` object

| Field | Type | Description |
|---|---|---|
| `time` | `number` | Bar open time as **Unix milliseconds** (UTC). Divide by 1000 for Unix seconds. |
| `open` | `number` | Opening price of the 1-minute bar (float, USD) |
| `high` | `number` | Highest traded price within the bar (float, USD) |
| `low` | `number` | Lowest traded price within the bar (float, USD) |
| `close` | `number` | Closing price of the 1-minute bar (float, USD) |
| `volume` | `number` | Share volume traded during the bar (integer) |

### Bar timing

- Bars are **1-minute candles** (60 000 ms apart).
- The first bar of a regular session opens at **09:30 ET** (14:30 UTC in winter / 13:30 UTC in summer).
- A full regular session contains **390 bars** (09:30–16:00 ET inclusive). Some files have fewer bars due to trading halts, early closes, or data gaps.
- Pre-market and after-hours bars are **not included**.

---

## Usage in Razor

The simulator resolves cache files by constructing the path:

```
{cacheDir}/{SYMBOL}_{YYYY-MM-DD}.json
```

To replay a trading day, load the file, iterate `bars` in order, and feed each bar's OHLCV values into the strategy engine. The `time` field can be used to reconstruct intraday timestamps for display or event scheduling.

When `meta` is present, `regularMarketPrice` is used as the prior-close reference for gap calculations at session open.

---

## Coverage Notes

- Ticker symbols reflect the symbol **at the time of the trading day**. Renamed, merged, or delisted tickers may appear under their historical symbol (e.g. `YHOO` in `cache_apex`).
- `cache_og_298` and `cache` overlap some tickers with `cache_snp500` and `cache_ndx`; the simulator prefers the most specific/recent cache directory for a given (ticker, date) pair.
- All price values are **unadjusted** (raw market prices, not split- or dividend-adjusted).
