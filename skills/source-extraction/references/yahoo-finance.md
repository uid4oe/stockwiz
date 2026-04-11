# Yahoo Finance

**Status:** **best-effort / usually fails.** Yahoo has been rate-limited or 503'd in every recorded stockwiz session (5 of 5) — the quoteSummary JSON API is aggressively throttling unauthenticated requests, and the crumb retry path also fails consistently. Yahoo is retained in the fetch plan as a best-effort cross-check, but stockwiz does NOT depend on it for any critical path. If Yahoo succeeds, the data is a useful third-source verification of P/E, margins, ownership, and short interest. If (more likely) it fails, the deep-researcher marks it failed and moves on — Stockanalysis and Finviz carry the load.

**Historically valuable for** fundamentals and statistics, but the public HTML pages are React-rendered and heavily gated. We fetch the **undocumented JSON quoteSummary API** that yfinance-style tools use, via curl with a browser User-Agent and cookie handling. The infrastructure is correct; Yahoo's rate-limit posture is the limiting factor.
**Access method:** **`Bash` + `curl`** (not WebFetch). Yahoo's HTML pages return either an empty JavaScript skeleton or a 503 to non-browser UAs; the JSON API needs a cookie from a prior visit to `finance.yahoo.com` and sometimes a crumb token.
**Rate policy:** 1500ms delay; one retry on transient failure.

## Why curl, not WebFetch

The same reason as SEC EDGAR: we need header control. Yahoo's CDN routes based on User-Agent — curl with a plain UA gets 503'd, curl with a real browser UA gets served. WebFetch's default UA is not a browser UA, so WebFetch hits the same wall.

Additionally, Yahoo's HTML pages ship a React shell that loads data via client-side JS. WebFetch (and curl) can only see the shell, not the rendered data. The reliable path is the JSON API at `query1.finance.yahoo.com`, which returns structured data directly.

## URL patterns

### Stage 1 — Consent cookie fetch (throwaway)

Yahoo's quoteSummary API checks for an `A1`/`A3` cookie that's set when a browser visits `finance.yahoo.com`. Without it, the API returns a 401 or redirects to a consent page. We simulate the browser visit with curl using `-c` to save the cookies:

```bash
COOKIE_JAR=$(mktemp -t stockwiz-yahoo-cookies.XXXXXX)
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'

curl -sS -L \
  -A "$BROWSER_UA" \
  -c "$COOKIE_JAR" \
  --max-time 20 \
  "https://finance.yahoo.com/quote/${TICKER}" \
  -o /dev/null
```

The response body is discarded (it's the React shell). We only care about the cookies saved to `$COOKIE_JAR`. If this fetch fails hard (non-2xx/non-3xx, or DNS error), fail the whole Yahoo source.

### Stage 2 — quoteSummary API call (the real fetch)

```bash
MODULES='summaryDetail,defaultKeyStatistics,financialData,price,assetProfile,summaryProfile,incomeStatementHistory,balanceSheetHistory,cashflowStatementHistory'

curl -sS -L \
  -A "$BROWSER_UA" \
  -b "$COOKIE_JAR" \
  -H "Accept: application/json" \
  --max-time 30 \
  "https://query1.finance.yahoo.com/v10/finance/quoteSummary/${TICKER}?modules=${MODULES}" \
  > "$SESSION_DIR/raw/yahoo-quotesummary.json"
```

This returns a large JSON structure. The key paths you care about are nested inside `quoteSummary.result[0].<module>`.

### Stage 3 (fallback) — Crumb-based retry

If the Stage 2 response is a JSON error like `"Invalid Crumb"` or an HTTP 401, you need a crumb token. Fetch one:

```bash
CRUMB=$(curl -sS \
  -A "$BROWSER_UA" \
  -b "$COOKIE_JAR" \
  --max-time 15 \
  "https://query1.finance.yahoo.com/v1/test/getcrumb")
```

Then retry Stage 2 with `&crumb=$CRUMB` appended. If the crumb call itself fails, mark Yahoo as failed and move on — we have Finviz and Stockanalysis as fallbacks for ratios.

Clean up the cookie jar after all Yahoo fetches:
```bash
rm -f "$COOKIE_JAR"
```

## What to extract from the JSON

The quoteSummary response structure is roughly:

```json
{
  "quoteSummary": {
    "result": [{
      "summaryDetail": { "regularMarketPrice": { "raw": 188.63 }, ... },
      "defaultKeyStatistics": { "forwardPE": { "raw": 17.04 }, ... },
      "financialData": { "profitMargins": { "raw": 0.556 }, ... },
      "price": { "longName": "NVIDIA Corporation", ... },
      "assetProfile": { "sector": "Technology", ... },
      "incomeStatementHistory": { "incomeStatementHistory": [...] },
      "balanceSheetHistory": { "balanceSheetStatements": [...] },
      "cashflowStatementHistory": { "cashflowStatements": [...] }
    }],
    "error": null
  }
}
```

Extract these fields (use `jq` or Python to parse):

**From `summaryDetail`:**
- `regularMarketPrice.raw`, `previousClose.raw`, `open.raw`, `dayLow.raw`, `dayHigh.raw`
- `fiftyTwoWeekLow.raw`, `fiftyTwoWeekHigh.raw`
- `marketCap.raw`, `trailingPE.raw`, `forwardPE.raw`, `priceToSalesTrailing12Months.raw`
- `volume.raw`, `averageVolume.raw`, `averageVolume10days.raw`
- `dividendYield.raw`, `trailingAnnualDividendRate.raw`, `payoutRatio.raw`
- `beta.raw`

**From `defaultKeyStatistics`:**
- `enterpriseValue.raw`, `enterpriseToRevenue.raw`, `enterpriseToEbitda.raw`
- `trailingEps.raw`, `forwardEps.raw`, `pegRatio.raw`
- `priceToBook.raw`, `bookValue.raw`
- `profitMargins.raw`, `sharesOutstanding.raw`, `floatShares.raw`
- `heldPercentInsiders.raw`, `heldPercentInstitutions.raw`
- `shortRatio.raw`, `shortPercentOfFloat.raw`, `sharesShort.raw`
- `52WeekChange.raw`, `SandP52WeekChange.raw`

**From `financialData`:**
- `currentPrice.raw`, `targetMeanPrice.raw`, `targetHighPrice.raw`, `targetLowPrice.raw`
- `numberOfAnalystOpinions.raw`, `recommendationKey` (string: "buy"/"strong_buy"/etc. — **capture raw string for attribution, but never display without `<q>` wrapping**)
- `totalCash.raw`, `totalDebt.raw`, `debtToEquity.raw`
- `currentRatio.raw`, `quickRatio.raw`
- `totalRevenue.raw`, `revenuePerShare.raw`
- `grossMargins.raw`, `operatingMargins.raw`, `ebitdaMargins.raw`
- `revenueGrowth.raw`, `earningsGrowth.raw`
- `returnOnAssets.raw`, `returnOnEquity.raw`
- `freeCashflow.raw`, `operatingCashflow.raw`

**From `price`:**
- `longName`, `shortName`, `symbol`, `exchangeName`

**From `assetProfile`:**
- `sector`, `industry`, `fullTimeEmployees`, `country`, `city`
- `longBusinessSummary` — a 1–2 paragraph company description (useful for the hero tagline and thesis context)

**From history arrays:**
- `incomeStatementHistory.incomeStatementHistory[0..3]`: the last 4 years of income statements (revenue, gross profit, operating income, net income)
- `balanceSheetHistory.balanceSheetStatements[0..3]`: last 4 years of balance sheets (total assets, total liabilities, total equity, cash, debt)
- `cashflowStatementHistory.cashflowStatements[0..3]`: last 4 years of cash flows (CFO, CapEx, FCF implied)

## Writing the raw file

Parse the JSON and write a clean markdown file at `$SESSION_DIR/raw/yahoo-fundamentals.md`:

```markdown
---
source: Yahoo Finance quoteSummary API
url: https://query1.finance.yahoo.com/v10/finance/quoteSummary/NVDA?modules=...
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Yahoo Finance — NVDA

## Identity

- Long name: NVIDIA Corporation
- Symbol: NVDA
- Exchange: NasdaqGS
- Sector: Technology
- Industry: Semiconductors
- Employees: 36,000
- Business summary: {full text from assetProfile.longBusinessSummary}

## Quote

- Price: 188.63 (as of 2026-04-11T14:30:00Z)
- Previous close: 183.91
- Day range: 185.20 – 189.44
- 52-week range: 95.04 – 212.19
- Market cap: 4,583,710,000,000
- Volume: 159,746,110
- Avg volume: 178,920,000

## Valuation

- Trailing P/E: 38.48
- Forward P/E: 17.04
- PEG: 0.44
- P/S: 21.23
- P/B: 29.15
- Enterprise value: 4,532,570,000,000
- EV/Revenue: 20.99
- EV/EBITDA: 34.02

## Quality & margins

- Gross margin: 71.07%
- Operating margin: 60.38%
- Profit margin: 55.60%
- Return on assets: 75.42%
- Return on equity: 101.49%

## Balance sheet

- Total cash: ...
- Total debt: ...
- Debt/Equity: 0.07
- Current ratio: 3.91
- Quick ratio: 3.24

## Cash flow

- Operating cash flow (TTM): ...
- Free cash flow (TTM): ...

## Growth

- Revenue growth YoY: ...
- Earnings growth YoY: ...

## Ownership

- Shares outstanding: 24,300,000,000
- Float: 23,380,000,000
- Insider ownership: 3.79%
- Institutional ownership: 68.39%
- Shares short: 280,870,000
- Short ratio: 1.57
- Short % of float: 1.20%

## Analyst view

- Number of analysts: 42
- Target mean price: 269.16
- Target high: 340.00
- Target low: 195.00
- Raw recommendation key: "strong_buy"  <!-- capture verbatim; report-writer wraps in <q> -->

## Historical financials (last 4 FY)

### Income statement

| FY End | Revenue | Gross profit | Op income | Net income |
|---|---|---|---|---|
| 2025-01-26 | $130,497M | ... | ... | $72,880M |
| 2024-01-28 | $60,922M | ... | ... | $29,760M |
| 2023-01-29 | $26,974M | ... | ... | $4,368M |
| 2022-01-30 | $26,914M | ... | ... | $9,752M |

### Balance sheet

(similar table)

### Cash flow

(similar table)
```

Preserve every number verbatim. Date every figure with the fiscal period.

## Failure modes

| Condition | Reason code | Action |
|---|---|---|
| Stage 1 (consent fetch) returns non-2xx/3xx | `consent-fetch-failed` | Mark Yahoo failed, move on |
| Stage 2 returns JSON with `error` field non-null | `api-error: <error message>` | If error is `"Invalid Crumb"`, go to Stage 3. Otherwise fail. |
| Stage 2 returns HTTP 401/403 | `yahoo-auth` | Try crumb retry once; then fail |
| Stage 2 returns HTTP 429 | `rate-limited` | Retry once after 10s backoff; then fail |
| Stage 2 returns HTTP 5xx | `yahoo-503` | Retry once after 5s backoff; then fail |
| Stage 3 crumb fetch fails | `crumb-unavailable` | Fail Yahoo entirely |
| JSON parse fails | `invalid-json` | Fail Yahoo entirely |
| `quoteSummary.result` is empty array | `ticker-not-found-at-yahoo` | Mark failed |

On any terminal failure, **do not block the deep-dive**. Yahoo is redundant with Stockanalysis and Finviz for most of its fields. The orchestrator's fatal-error rule only triggers if SEC EDGAR fails, not Yahoo.

## Slug

Write structured output to `raw/yahoo-fundamentals.md`. Also save the raw JSON to `raw/yahoo-quotesummary.json` for debugging (it's the audit trail).

## Calls

- Happy path: **2 curl calls** (consent + API)
- With crumb retry: **3 curl calls** (consent + API + crumb + retry)

Fits the budget.

## A note on the recommendationKey field

Yahoo's `financialData.recommendationKey` is a string like `"strong_buy"`, `"buy"`, `"hold"`, `"sell"`, `"strong_sell"`. **Capture it verbatim** in the raw file with no interpretation. Downstream the report-writer will quote it inside `<q>` tags and frame it in neutral language like "Yahoo's aggregate analyst consensus on a 5-level scale is <q>strong_buy</q>", so the compliance pass doesn't rewrite it.

Never generate the word "buy" or "sell" in prose of your own construction. Only ever reproduce it inside quote tags as a direct source attribution.
