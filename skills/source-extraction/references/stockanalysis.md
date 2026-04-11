# Stockanalysis.com

**Status:** active. Excellent source for multi-year financial statements. Scrape-friendly HTML. Primary fallback for Yahoo Finance when the Yahoo JSON API is gated or returns incomplete modules.
**Access method:** **`Bash` + `curl`** with browser User-Agent. Stockanalysis serves clean HTML with modest JS enhancement; lynx or direct HTML parsing both work.
**Rate policy:** 1500ms delay; one retry on transient failure.

## Why it's in the source set

Brought forward because:

1. **Yahoo Finance is unreliable.** Even with the JSON API fix, Yahoo 503s and crumb failures are frequent enough to need a structured-data fallback that doesn't depend on Yahoo.
2. **5–10Y financial history is the backbone of trend analysis.** Finviz gives one year of numbers; Yahoo gives 4 years if you're lucky; Stockanalysis gives 5–10 years of income statement, balance sheet, and cash flow with consistent columns.
3. **Scrape-friendly.** Plain HTML tables, no aggressive JS rendering, no consent walls, no Cloudflare in my experience.

## URL patterns

Stockanalysis uses lowercase tickers in slugs:

```
https://stockanalysis.com/stocks/{ticker-lower}/
https://stockanalysis.com/stocks/{ticker-lower}/financials/
https://stockanalysis.com/stocks/{ticker-lower}/financials/balance-sheet/
https://stockanalysis.com/stocks/{ticker-lower}/financials/cash-flow-statement/
https://stockanalysis.com/stocks/{ticker-lower}/statistics/
https://stockanalysis.com/stocks/{ticker-lower}/forecast/
```

fetch **only the most valuable pages** to stay under the fetch budget:

1. `/stocks/{ticker}/` — overview, current metrics
2. `/stocks/{ticker}/financials/` — income statement (5Y default)
3. `/stocks/{ticker}/statistics/` — valuation metrics + ownership

Skip balance sheet, cash flow, and forecast pages unless SEC EDGAR failed and we need Stockanalysis to fully backfill. In that emergency fallback mode, fetch all 6 pages.

## Fetch

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'
TICKER_LOWER=$(echo "$TICKER" | tr '[:upper:]' '[:lower:]')

for path in "" "/financials/" "/statistics/"; do
  URL="https://stockanalysis.com/stocks/${TICKER_LOWER}${path}"
  SLUG=$(echo "${path:-overview}" | tr '/' '-' | sed 's/^-//;s/-$//')
  curl -sS -L \
    -A "$BROWSER_UA" \
    --max-time 30 \
    -H "Accept: text/html,application/xhtml+xml" \
    "$URL" \
    | lynx -stdin -dump -width=140 -nolist \
    > "/tmp/stockwiz-sa-${TICKER}-${SLUG:-overview}.txt"
  sleep 1.5
done
```

Use `lynx -dump -width=140` — wider than SWS/SA pages because Stockanalysis uses wider tables. 140 columns captures most tables without wrapping.

If lynx is not available, save raw HTML and parse with Read. Tables render cleanly as text in lynx dumps, so lynx is strongly preferred here.

## What to extract

### From the overview page (`/stocks/{ticker}/`)

- Company name, ticker, exchange, sector, industry
- Current price, day change, market cap
- A small summary block with: P/E, EPS, dividend yield, 52w range
- Description / business summary (1–2 paragraphs)

### From the financials page (`/stocks/{ticker}/financials/`)

This is the main table: **5 years of annual income statement** with one column per year. Extract every row:

- Revenue
- Cost of revenue
- Gross profit
- Operating expenses (SG&A, R&D, other)
- Operating income
- Interest expense
- Pre-tax income
- Income tax
- Net income
- EPS basic / diluted
- Shares outstanding (basic / diluted)

Preserve all numbers verbatim. Date each column with the fiscal year end.

Stockanalysis also shows growth rates (YoY) in a small sub-row under each absolute row. Capture those too — they save computation downstream.

### From the statistics page (`/stocks/{ticker}/statistics/`)

A dense dashboard of ~40 metrics organized into sections:

- **Valuation**: market cap, EV, P/E (T+F), PEG, P/S, P/B, EV/EBITDA, EV/Sales
- **Financial position**: total cash, debt, D/E, current ratio, quick ratio
- **Margins & returns**: gross / operating / profit margins, ROE, ROA, ROIC
- **Growth**: revenue 1Y / 3Y / 5Y CAGR, EPS same
- **Trading**: 52w range, beta, average volume, short interest
- **Ownership**: shares outstanding, float, insider %, institutional %
- **Dividend**: yield, rate, payout, 5Y CAGR
- **Analyst**: average price target, analyst count, most common rating (if shown)

Capture every metric. Stockanalysis is our dedicated full-stats source.

## Writing the raw file

Organize into a single markdown file:

```markdown
---
source: Stockanalysis.com
url: https://stockanalysis.com/stocks/nvda/ (+ /financials/ + /statistics/)
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Stockanalysis.com — NVDA

## Identity

- Company: NVIDIA Corp
- Exchange: NASDAQ
- Sector: Technology
- Industry: Semiconductors
- Current price: 188.63 (as of fetch)

## 5-year income statement (annual)

| Metric | FY2025 | FY2024 | FY2023 | FY2022 | FY2021 |
|---|---|---|---|---|---|
| Revenue ($M) | 130,497 | 60,922 | 26,974 | 26,914 | 16,675 |
| Revenue growth YoY | +114.2% | +125.9% | +0.2% | +61.4% | ... |
| Gross profit ($M) | ... | ... | ... | ... | ... |
| Gross margin | ... | ... | ... | ... | ... |
| Operating income ($M) | ... | ... | ... | ... | ... |
| Operating margin | ... | ... | ... | ... | ... |
| Net income ($M) | 72,880 | 29,760 | 4,368 | 9,752 | 4,332 |
| EPS (diluted) | ... | ... | ... | ... | ... |
| Diluted shares outstanding | ... | ... | ... | ... | ... |

## Statistics snapshot

### Valuation
- Market cap: ...
- Enterprise value: ...
- P/E (trailing): ...
- P/E (forward): ...
- PEG: ...
- P/S: ...
- EV/EBITDA: ...

### Financial position
- Total cash: ...
- Total debt: ...
- Debt/Equity: ...
- Current ratio: ...

### Margins & returns
- Gross margin: ...
- Operating margin: ...
- Profit margin: ...
- ROIC: ...
- ROE: ...
- ROA: ...

### Growth (CAGR)
- Revenue 3Y: ...
- Revenue 5Y: ...
- EPS 3Y: ...
- EPS 5Y: ...

### Ownership
- Shares outstanding: ...
- Float: ...
- Insider %: ...
- Institutional %: ...

### Dividend
- Yield: ...
- Rate: ...
- Payout ratio: ...
- 5Y dividend growth: ...

### Analyst view
- Average price target: ...
- Analyst count: ...
- Most common rating (raw): ...
```

Preserve every number verbatim. Stockanalysis uses consistent formatting ($M, %, decimals) so extraction is straightforward.

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| HTTP 404 on overview page | `ticker-not-on-stockanalysis` | Skip all Stockanalysis pages |
| HTTP 429 | `rate-limited` | Retry once after 10s; then skip |
| HTTP 5xx | `stockanalysis-5xx` | Retry once; then skip |
| lynx dump is empty or <500 chars | `extraction-empty` | Fall back to raw HTML Read |
| Table rows missing (e.g. `/financials/` shows "Data not available") | `incomplete-data` | Mark ok but note in _sanity.md |

Skipping Stockanalysis is never fatal — it's a fallback/complement, not a ground-truth source. The only situation where losing Stockanalysis hurts materially is when Yahoo Finance also failed; in that case, we lose multi-year trend data, and `fundamental-analysis` will flag many Unknowns in its output.

## Slug

Write to `raw/stockanalysis.md`. If using lynx dumps, also save the raw text dumps to `raw/stockanalysis-overview.txt`, `raw/stockanalysis-financials.txt`, `raw/stockanalysis-statistics.txt` for debugging.

## Calls

- normal: **3 curl calls** (overview, financials, statistics)
- Emergency mode (Yahoo failed, need balance sheet + cash flow): **6 curl calls**

Normal mode fits comfortably in the budget.

## A note on why we use lynx

HTML parsing of data tables via LLM prompts works, but it's wasteful — the LLM sees all the markup, navigation, scripts, and styling noise. Piping through `lynx -dump` converts the page to a clean text representation where tables become aligned column text. The LLM then reads that text and extracts fields with much better signal-to-noise and token efficiency.

`lynx` is preinstalled on macOS and on most Linux distros. If a user's system is missing it, the agent should install it via homebrew (`brew install lynx`) or apt (`apt-get install lynx`) — we currently assume it is present. A future phase can add a detection step in `/stockwiz-setup` that warns if lynx is missing.
