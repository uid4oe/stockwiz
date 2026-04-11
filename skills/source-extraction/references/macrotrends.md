# Macrotrends

**Status:** Phase 2.5. Deep historical financials — typically 10Y to 20Y of annual data per company, with consistent columns across years. Complements Stockanalysis (5Y) with longer cycle context.
**Access method:** **`Bash` + `curl`** with browser User-Agent + `lynx -dump`. Scrape-friendly, not Cloudflare-gated.
**Rate policy:** 1500ms delay; one retry on transient failure.

## Why it's here

Five years of history isn't enough to see a business cycle. Macrotrends is the only free source that consistently publishes 10-20 years of annual financial data in a clean tabular format. For cyclical industries (semis, energy, materials, autos), banking, and any mean-reversion thesis, the extra 5-15 years of history is the difference between "growth is decelerating" and "growth is reverting to the long-run mean."

Macrotrends's value for stockwiz:

- **10Y-20Y revenue history** with YoY growth rates
- **10Y-20Y EPS history**
- **10Y-20Y margin history** (gross, operating, net)
- **10Y-20Y FCF history**
- **10Y-20Y debt history**
- **10Y-20Y ROE / ROA / ROIC history**
- **Chart URLs** showing visual trends (we don't fetch images, but the URL structure can be recorded)

Phase 2.5 fetches only the revenue, earnings, margin, and FCF pages to stay within the fetch budget. Phase 3+ can add balance sheet and return metrics if needed.

## URL resolution

Macrotrends uses a slug-based URL pattern:

```
https://www.macrotrends.net/stocks/charts/{TICKER}/{company-slug}/{metric-page}
```

Example:
```
https://www.macrotrends.net/stocks/charts/NVDA/nvidia/revenue
https://www.macrotrends.net/stocks/charts/AAPL/apple/revenue
https://www.macrotrends.net/stocks/charts/TSLA/tesla/revenue
```

The company slug is a kebab-case version of the company name. For well-known tickers you can guess (nvidia, apple, microsoft, tesla), but the safest approach is **WebSearch** to resolve:

```
WebSearch: "site:macrotrends.net {TICKER} revenue"
```

The first result should be the canonical `/stocks/charts/{TICKER}/{slug}/revenue` URL. Extract the `{slug}` component from it and reuse it for all subsequent page fetches.

**Slug cache (Phase 2.5 — self-populating).** stockwiz maintains a persistent slug cache at `~/.claude/stockwiz/cache/macrotrends-slugs.json`. The deep-researcher checks the cache before issuing the WebSearch:

1. Read `~/.claude/stockwiz/cache/macrotrends-slugs.json`. If it doesn't exist, treat as `{}`.
2. If `cache[TICKER]` is set, use that slug directly — **skip WebSearch entirely**.
3. If not, run the WebSearch (`site:macrotrends.net {TICKER} revenue`), extract the slug from the top result URL (it's the segment after `/charts/{TICKER}/`), and **write it back**:
   ```python
   cache = json.load(open(path)) if exists(path) else {}
   cache[TICKER] = resolved_slug
   json.dump(cache, open(path, 'w'), indent=2)
   ```
4. Then proceed to the 4 curl fetches.

Over many sessions the cache fills up and the WebSearch cost drops to zero for previously-seen tickers. The cache file is tiny (one line per ticker) and doesn't need pruning.

Handle write-races defensively: read-modify-write in a single turn. If another session is writing simultaneously, the worst case is one extra WebSearch — acceptable.

## Pages to fetch

Once you have the slug, fetch these pages (in this order):

1. `/{TICKER}/{slug}/revenue` — 10Y+ annual revenue with growth rates
2. `/{TICKER}/{slug}/net-income` — 10Y+ annual net income
3. `/{TICKER}/{slug}/gross-profit` — 10Y+ gross profit and margin
4. `/{TICKER}/{slug}/free-cash-flow` — 10Y+ FCF

Stop there for Phase 2.5. Each page returns a tabular display that `lynx -dump` converts cleanly. Total: 4 curl calls.

Skip in Phase 2.5 (deferred to Phase 3+):
- Balance sheet pages (total-assets, total-liabilities, long-term-debt)
- Return metric pages (roe, roa, roic)
- Share-count pages
- Dividend history pages

## Fetch

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'
SLUG="nvidia"  # from WebSearch resolution

for page in revenue net-income gross-profit free-cash-flow; do
  URL="https://www.macrotrends.net/stocks/charts/${TICKER}/${SLUG}/${page}"
  curl -sS -L \
    -A "$BROWSER_UA" \
    --max-time 30 \
    -H "Accept: text/html,application/xhtml+xml" \
    "$URL" \
    | lynx -stdin -dump -width=120 -nolist \
    > "/tmp/stockwiz-macrotrends-${TICKER}-${page}.txt"
  sleep 1.5
done
```

If lynx is missing, save raw HTML and Read it with offset/limit. Macrotrends tables are relatively simple (data in `<table>` elements) so raw HTML parsing works OK.

## What to extract

### Revenue page

Macrotrends's revenue page has two main sections: a historical annual table and a TTM quarterly table. Extract the annual table only:

```
Year    Revenue       YoY Growth
2026    $215,938 M    +65.47%
2025    $130,497 M    +114.20%
2024    $60,922 M     +125.85%
2023    $26,974 M     +0.22%
2022    $26,914 M     +61.40%
2021    $16,675 M     +52.74%
2020    $10,918 M     +6.81%
2019    $10,221 M     -6.81%
2018    $10,968 M     +40.58%
2017    $7,802 M      +17.87%
2016    $6,621 M      +32.64%
2015    $4,990 M      ...
```

Capture all years available (typically 10-20). Preserve numbers verbatim.

### Net income page

Same format: Year / Net Income / YoY Growth. 10-20 years annual.

### Gross profit page

Macrotrends often combines gross profit with gross margin on the same page:

```
Year    Gross Profit    Gross Margin
2026    $153,463 M      71.07%
2025    $97,858 M       74.99%
...
```

Capture both columns.

### FCF page

Macrotrends's free cash flow page typically shows:

```
Year    Free Cash Flow   FCF Margin (as % of revenue)
2026    $96,676 M        44.77%
2025    $60,853 M        46.63%
...
```

Capture both columns.

## Writing the raw file

```markdown
---
source: Macrotrends
url: https://www.macrotrends.net/stocks/charts/NVDA/nvidia/ (+ 4 metric pages)
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Macrotrends — NVDA

## Identity

- Ticker: NVDA
- Slug: nvidia
- Data coverage: FY2012 through FY2026 (15 years annual)

## Revenue history (annual, $ millions)

| FY | Revenue | YoY Growth |
|---|---|---|
| 2026 | 215,938 | +65.47% |
| 2025 | 130,497 | +114.20% |
| 2024 | 60,922 | +125.85% |
| 2023 | 26,974 | +0.22% |
| 2022 | 26,914 | +61.40% |
| 2021 | 16,675 | +52.74% |
| 2020 | 10,918 | +6.81% |
| 2019 | 10,221 | -6.81% |
| 2018 | 10,968 | +40.58% |
| 2017 | 7,802 | +17.87% |
| 2016 | 6,621 | +32.64% |
| 2015 | 4,990 | +14.02% |
| 2014 | 4,377 | +0.87% |
| 2013 | 4,339 | +4.11% |
| 2012 | 4,168 | +12.64% |

10-year revenue CAGR: ~30.0% (computed: 215938/4990 = 43.3x, 43.3^(1/10) ≈ 1.46 → 45.9%) — verify manually

Actually, DO NOT compute CAGRs in the raw file. The fundamental-analysis skill will compute them downstream. Just capture the raw table verbatim.

## Net income history (annual, $ millions)

| FY | Net Income | YoY Growth |
|---|---|---|
| 2026 | 120,067 | +64.75% |
| 2025 | 72,880 | +144.89% |
| ... | ... | ... |

## Gross profit and margin history (annual)

| FY | Gross Profit ($M) | Gross Margin |
|---|---|---|
| 2026 | 153,463 | 71.07% |
| 2025 | 97,858 | 74.99% |
| ... | ... | ... |

## Free cash flow history (annual)

| FY | FCF ($M) | FCF Margin |
|---|---|---|
| 2026 | 96,676 | 44.77% |
| 2025 | 60,853 | 46.63% |
| ... | ... | ... |
```

Preserve every number verbatim. 10-15 years is typical; some tickers have 20 years, some have fewer if the company is young or IPO'd recently.

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| WebSearch doesn't return a macrotrends URL | `macrotrends-slug-unresolved` | Try with known slug heuristic (lowercase company name, kebab-case); if that also 404s, skip |
| HTTP 404 on revenue page | `ticker-not-on-macrotrends` | Skip all subsequent pages (company not covered) |
| HTTP 429 | `rate-limited` | 10s backoff, one retry, then skip |
| lynx dump is mostly navigation with no data rows | `extraction-empty` | Check the first page; if empty, skip subsequent pages |

Skipping Macrotrends is never fatal. Stockanalysis's 5Y history is the primary historical source; Macrotrends is the deep-history enhancement.

## Slug

Write to `raw/macrotrends.md`.

## Calls

- 1 WebSearch call (slug resolution, unless cached)
- 4 curl calls (revenue, net-income, gross-profit, FCF)
- Total: **5 fetches** per deep-dive

## What analyses use this

- `fundamental-analysis` uses the long history for cycle framing and long-run CAGR calculation
- `thesis-discipline` can reference multi-cycle context for "growth reverting to long-run mean" framings in the bear case
- Any analyst doing "is this a cyclical peak?" question relies on the 10+Y view — which is exactly what Macrotrends provides
