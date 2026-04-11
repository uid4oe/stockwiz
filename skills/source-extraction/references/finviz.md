# Finviz

**Status:** active. First-class source — dense quote page, scrape-friendly, low failure rate.
**Keyless.** No account needed.
**Rate policy:** 1500ms delay; fail after one retry.

## Why it's a first-class source

Finviz publishes a single page per ticker that contains roughly 60 data points in a compact tabular layout. It's the densest free page in the set and has no consent wall, no Cloudflare gating, and no paywall. If Finviz fails, something unusual is happening.

Finviz also provides useful derived fields that Yahoo does not:
- Direct "analyst recom" numeric (average analyst rating 1–5 scale)
- ATR (average true range) and volatility %
- RSI (14)
- Simple moving averages (20, 50, 200) as % from price
- Insider transactions in dollar terms
- Institutional transactions in dollar terms
- Industry and sector tags with direct links

## URL pattern

```
https://finviz.com/quote.ashx?t={ticker}
```

Single page, single fetch. The ticker should be uppercase. Finviz is case-insensitive but the canonical form is upper.

## What to extract

Everything on the snapshot table. The fields map roughly into these categories:

**Identity & classification**
- Company name
- Sector
- Industry
- Country
- Index membership (S&P 500, Nasdaq 100, etc. if shown)

**Valuation**
- Market cap
- P/E (trailing)
- Forward P/E
- PEG
- P/S
- P/B
- P/Cash
- P/FCF
- EPS (TTM)
- EPS next Y
- EPS next Q
- EPS this Y (%)
- EPS next Y (%)
- EPS next 5Y (%)
- EPS past 5Y (%)
- Sales past 5Y (%)
- Quarterly sales growth (%)
- Quarterly earnings growth (%)

**Quality & capital structure**
- ROA
- ROE
- ROI
- Gross margin
- Operating margin
- Profit margin
- Debt/Equity
- LT Debt/Equity
- Current ratio
- Quick ratio

**Ownership & transactions**
- Insider ownership (%)
- Insider transactions (value, net, last 3m)
- Institutional ownership (%)
- Institutional transactions (value, net)
- Shares outstanding
- Float
- Float short (%)
- Short ratio

**Analyst view**
- Analyst recom (1.0–5.0 numeric, 1 = strong buy — **raw value only, NEVER interpret as advice**)
- Target price
- Dividend (yield + rate)
- Payout ratio

**Trading / technical**
- Price
- Prev close
- Change (%)
- Volume
- Avg volume
- Day's range
- 52w range (low / high)
- 52w high (%)
- 52w low (%)
- Beta
- ATR
- Volatility (W / M)
- RSI (14)
- SMA 20, 50, 200 (% from price)
- Rel volume

**Earnings**
- Earnings date (next, with estimate)

## Extraction prompt template

> This is a Finviz quote page for a US equity. Extract every data point from the main snapshot table as a markdown key-value list. Preserve numbers exactly as shown (including % signs, B/M suffixes, and negative values). Organize into these sections: Identity, Valuation, Quality, Ownership, Analyst, Trading, Earnings. Include the "Analyst recom" numeric value verbatim (e.g. "2.1") but do NOT include any text interpretation of it. Do not editorialize. If a field is blank or shows "-", write "not disclosed".

## Failure signatures

Finviz is reliable but check for:
- `Access Denied` or `blocked` in body → reason `403`
- Empty or tiny response → reason `empty-response`
- Redirect to `finviz.com/login` → reason `auth-wall`

On any of these, mark failed. No retry. No Finviz fallback — if Finviz is gone, we lose Finviz-specific fields (analyst recom, RSI, SMAs) and other sources must carry the load for the rest.

## Fallback

None. Finviz is a primary source. Its fields have partial redundancy with Yahoo (P/E, margins, shares) but the technicals and analyst recom are unique.

## Slug

Write to `raw/finviz-snapshot.md`.

## Calls

1 WebFetch call. That's the whole source.

## A note on the analyst recom

Finviz's "Analyst recom" is a 1.0–5.0 numeric average of analyst ratings where 1 = strong buy and 5 = strong sell. **Capture this number verbatim in the raw file.** Downstream in the report, the report-writer agent will present it as "analyst consensus is 2.1 on a 1–5 scale (lower = more positive)" — the raw label "Strong Buy" does NOT need to appear anywhere, and if any quoted text from Finviz says "Strong Buy", wrap it in `<q>` tags at the report stage.

This is a compliance hazard point because the source itself uses advisory language. Keep it as a pure number as long as possible in the pipeline.
