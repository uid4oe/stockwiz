# Yahoo Finance

**Status:** Phase 1. High-value source for price, fundamentals, and statistics. More brittle than SEC EDGAR — can be consent-walled in some regions.
**Keyless.** No account needed, but Yahoo sometimes injects a regional consent interstitial.
**Rate policy:** 1500ms delay; fail after one retry.

## Why it's here

Yahoo Finance is the most widely-used source for quick fundamentals. It provides:
- Current quote data (price, day range, volume, market cap)
- Key statistics (P/E trailing and forward, EPS, beta, margins, ROA/ROE, FCF, etc.)
- Financial statements (income, balance sheet, cash flow — usually 4 years)
- Holders (insider %, institutional %, short %)

It's not the most reliable source layout-wise, but when it works the coverage is broad and single-page.

## URL patterns

### Quote page
```
https://finance.yahoo.com/quote/{ticker}
```

### Key statistics
```
https://finance.yahoo.com/quote/{ticker}/key-statistics
```

### Financials (income statement)
```
https://finance.yahoo.com/quote/{ticker}/financials
```

### Profile page (less-gated fallback for basic info)
```
https://finance.yahoo.com/quote/{ticker}/profile
```

## What to extract

**From quote page:**
- Current price
- Previous close
- Day range
- 52-week range
- Market cap
- Avg volume (3m)
- P/E (trailing)
- EPS (trailing)
- Dividend & yield (if any)
- Earnings date (next)
- 1-year target estimate

**From key-statistics:**
- Enterprise value
- Trailing P/E, Forward P/E, PEG
- Price/Sales, Price/Book
- Enterprise Value / Revenue, Enterprise Value / EBITDA
- Profit margin, Operating margin
- Return on assets, Return on equity
- Revenue (TTM), Revenue per share
- Quarterly revenue growth (YoY)
- Gross profit
- EBITDA
- Diluted EPS (TTM)
- Quarterly earnings growth (YoY)
- Total cash, Total debt
- Debt / Equity
- Current ratio
- Book value per share
- Operating cash flow (TTM)
- Levered free cash flow (TTM)
- Beta (5y monthly)
- 52-week change, S&P 500 52-week change
- 52-week high, 52-week low
- 50-day and 200-day moving averages
- Avg vol (3m and 10-day)
- Shares outstanding, float
- % held by insiders, % held by institutions
- Shares short, short ratio, short % of float, short % of shares outstanding
- Forward annual dividend rate, yield, payout ratio, 5-year avg div yield

That's a lot of fields. The extraction prompt should ask for all of them at once — Yahoo puts them on a single page.

## Extraction prompt template

For the key-statistics page:

> This is a Yahoo Finance key statistics page. Extract the following values in a markdown key-value list, preserving numbers exactly as shown and noting "N/A" for any field marked as such or missing:
>
> Valuation: Market cap, Enterprise value, P/E (T), P/E (F), PEG, P/S, P/B, EV/Revenue, EV/EBITDA
> Profitability: Profit margin, Operating margin, ROA, ROE
> Income: Revenue (TTM), Revenue per share, Qtr revenue growth (YoY), Gross profit, EBITDA, Diluted EPS (TTM), Qtr earnings growth (YoY)
> Balance sheet: Total cash, Total debt, D/E, Current ratio, Book value / share
> Cash flow: Op cash flow (TTM), Levered FCF (TTM)
> Trading: Beta, 52w change, 52w high, 52w low, 50-day MA, 200-day MA, Avg vol (3m), Avg vol (10d)
> Shares: Outstanding, Float, Insider %, Institutional %, Shares short, Short %, Short ratio
> Dividends: Forward rate, Forward yield, Payout ratio, 5y avg yield
>
> Do not editorialize. Do not compute new values. Return exactly what the page shows.

## Consent-wall detection

Yahoo sometimes serves `consent.yahoo.com` or `guce.yahoo.com` pages instead of the actual quote. Signs:
- URL contains `consent.yahoo.com` or `guce.yahoo.com`
- Body contains "We value your privacy" prominently
- No financial data in the first 2000 characters of response

On consent-wall detection: mark `failed` with reason `consent-wall`, do NOT retry (retry will hit the same wall). Fall through to Stockanalysis or Finviz for the data.

## Fallback chain

If quote page fails:
1. Try `/profile` — less-gated, gives basic info (sector, industry, company description, officers)
2. If profile also fails, skip Yahoo entirely and rely on Stockanalysis (Phase 2) + Finviz

If key-statistics page fails specifically:
1. Try the financials page — some statistics are available there
2. If both fail, skip and rely on Finviz for ratios

If financials page fails:
1. Try the quote page's summary
2. Fall through to SEC 10-K (already fetched) for the authoritative statements
3. In Phase 2, Stockanalysis.com is the primary redundancy

## Slug

Write output to `raw/yahoo-fundamentals.md`. If you split by page, append the page name: `raw/yahoo-profile.md`, `raw/yahoo-financials.md`. Only `yahoo-fundamentals.md` is considered canonical for downstream analyses.

## Calls

2–3 WebFetch calls in the happy path:
1. Quote page
2. Key-statistics
3. Financials (optional — can skip in MODE=thesis)
