# Barchart Insider Trades

**Status:** active. Best-effort source — adds *directional* time series for insider activity (3/6/12-month buy/sell shares), which Finviz only summarizes as a single 3-month % delta.
**Keyless.** No account needed.
**Rate policy:** 1500ms delay; one retry on 429/503; no retry on 403/Cloudflare.

## Why it's in the data set

Finviz already gives an `Insider Trans` field — a single percent change over the trailing 3 months. That's enough to flag direction at a glance, but it loses the *shape* of insider activity:

- Was it one large transaction or many small ones?
- Has buying accelerated or decelerated over the past 12 months?
- Are sells stamped with "(Option Exercise)" — i.e. neutral compensation noise — or open-market dispositions?

Barchart's insider-trades page aggregates filings into a 3/6/12-month grid of buys vs sells with share counts, which lets sentiment-synthesis compute a simple directional signal:

```
insider_signal_3m = (buy_shares_3m − sell_shares_3m) / (buy_shares_3m + sell_shares_3m + 1)
```

clamped to [-1, +1]. This is captured as raw aggregates in the source file; the agent computes the signal at read time.

## URL pattern

```
https://www.barchart.com/stocks/quotes/{TICKER}/insider-trades
```

Single page, single fetch. Ticker uppercase.

## What to extract

The aggregated transaction summary table at the top of the page, exposing 12 fields:

| Field | Meaning |
|---|---|
| `buys_3m` | Count of buy transactions in last 3 months |
| `buy_shares_3m` | Total shares bought in last 3 months |
| `sells_3m` | Count of sell transactions in last 3 months |
| `sell_shares_3m` | Total shares sold in last 3 months |
| `buys_6m` / `buy_shares_6m` / `sells_6m` / `sell_shares_6m` | Same for 6-month window |
| `buys_12m` / `buy_shares_12m` / `sells_12m` / `sell_shares_12m` | Same for 12-month window |

If the page exposes a "net shares" or "% of float" derived field, capture it as a free-text note but do not rely on it — the four count/share pairs are the primary signal.

## Extraction prompt template

> This is the Barchart insider-trades page for a US equity. Extract the aggregated transaction summary at the top of the page. Return a single JSON object with keys: `buys_3m`, `buy_shares_3m`, `sells_3m`, `sell_shares_3m`, `buys_6m`, `buy_shares_6m`, `sells_6m`, `sell_shares_6m`, `buys_12m`, `buy_shares_12m`, `sells_12m`, `sell_shares_12m`. All values as integers. Use null for any field that is missing or not disclosed. Also return a `notes` field as a string capturing any inline qualifier the page applies (e.g. "majority option exercises", "no insider activity in last 3 months"). JSON only, no commentary.

## Skip-conditions (not failures — known noise)

For some sectors, insider transactions are dominated by routine option-exercise activity and the directional signal is meaningless. The sentiment-synthesis agent will set `insider_signal_3m` to null and downweight the field when the underlying ticker is in:

- **Real Estate** (REITs use heavy unit-grant compensation)
- **Financial Services** (option-grant velocity dominates open-market signal)

The raw file still captures the numbers verbatim — the *interpretation* layer applies the skip rule, not the fetch layer.

## Failure signatures

- HTTP 403 / `Access Denied` → reason `403`. No retry. Mark failed.
- `challenges.cloudflare.com` / `Attention Required` → reason `cloudflare-challenge`. No retry. Mark failed.
- HTTP 429 → reason `429`. Sleep 10s, one retry, then mark failed if still 429.
- HTTP 503 → reason `503`. Sleep 5s, one retry.
- Empty body or no transaction grid present → reason `empty-response`. Mark failed.
- Page redirects to login / paywall → reason `auth-wall`. Mark failed.

On terminal failure, sentiment-synthesis falls back to Finviz's `Insider Trans` field alone (no directional time series).

## Fallback

None at the source layer. If Barchart fails, the directional signal is simply unavailable for this ticker; downstream sentiment weighting compensates by re-weighting the remaining insider signals (Finviz `insider_trans`, SEC Form 4 if present in 8-Ks, SWS qualitative flag).

## Slug

Write to `raw/barchart-insider-trades.md`.

## Calls

1 WebFetch call. That's the whole source.

## A note on directional interpretation

Barchart presents transaction data factually — "X shares bought, Y shares sold" — without editorial framing. **Capture as raw numbers; let sentiment-synthesis compute the signal.** Do not pre-interpret with phrases like "insiders are bullish" in the raw file. The agent later writes a neutral observation like "trailing-3m insider signal +0.42 (cluster buying)" with the source figures cited inline.
