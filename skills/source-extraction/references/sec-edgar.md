# SEC EDGAR

**Status:** Phase 1, always try first. Most reliable source in the set. Authoritative for US filings.
**Keyless.** No account or API key required.
**Rate policy:** SEC asks for ≤10 requests/second. stockwiz uses 1500ms delays, well under the limit.

## Why first

SEC EDGAR is the one source that:
- Is legally the primary record — no source disagrees with SEC filings, they're the ground truth
- Does not Cloudflare-challenge, does not consent-wall, does not paywall
- Provides revenue, cash flow, debt, and risk factors in structured prose
- Determines whether a ticker is a US filer at all (10-K) or a foreign private issuer (20-F)

If SEC EDGAR fails, something is very wrong. Abort the deep-dive with an error — don't trust the rest.

## URL patterns

### Step 1: Company filing index
```
https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK={ticker}&type=10-K&dateb=&owner=include&count=10
```

Substitute `{ticker}` with the uppercase symbol. The CIK parameter accepts tickers directly — SEC EDGAR resolves them internally.

### Step 2: Latest 10-K primary document
From the filing index page, find the most recent 10-K row. Click its "Documents" link (this is where the WebFetch prompt should ask the inner model to find the row by date and return the primary document URL). The primary `.htm` document is typically named something like:
- `aapl-20240928.htm` (Apple)
- `nvda-20240128.htm` (NVIDIA)
- `<ticker>-<fiscal-end>.htm`

WebFetch that URL.

### Optional: latest 10-Q
Same pattern with `type=10-Q`. Only needed for recent quarterly data if the 10-K is more than 4 months old.

## What to extract

From the 10-K, ask WebFetch to extract — specifically, asking for sections by their Item number:

**Item 1 — Business**
- Business overview (2–4 paragraphs)
- Segment breakdown (revenue by segment if disclosed)
- Geographic breakdown (revenue by region if disclosed)
- Customer concentration (any single customer >10% of revenue)
- Competitive positioning described by management

**Item 1A — Risk Factors**
- Summary of the top 5 risk factors as management describes them
- Don't reproduce the full text — stockwiz sentiment skills will read the raw file if needed

**Item 7 — Management's Discussion & Analysis (MD&A)**
- Revenue growth narrative (YoY changes + stated reasons)
- Margin narrative (what's expanding or compressing and why)
- Forward-looking guidance if present
- Material events or changes during the year

**Item 7A — Market Risk**
- Interest rate sensitivity
- Currency exposure
- Commodity exposure

**Cash flow statement (from Item 8)**
- Cash from operations (TTM)
- Capital expenditures (TTM)
- Free cash flow (CFO − CapEx)
- Stock-based compensation (as a % of CFO)

**Balance sheet (from Item 8)**
- Total debt (current + long-term)
- Cash + short-term investments
- Net debt
- Lease obligations (operating + finance)
- Total equity

**Item 10–14 (insider data, often from proxy)**
- Approximate insider ownership %
- Number of beneficial owners
- CEO / CFO tenure if mentioned
- Notable related-party transactions

## Extraction prompt template

When calling WebFetch on the 10-K, use a prompt like:

> From this SEC 10-K filing, extract the following in markdown:
>
> 1. Company name, ticker, fiscal year end
> 2. A 3-paragraph business overview from Item 1
> 3. Revenue by segment (if disclosed) with dollar amounts and YoY change
> 4. Revenue by geography (if disclosed) with dollar amounts
> 5. Any customer representing >10% of revenue, by name if disclosed
> 6. The 5 risk factors from Item 1A that management emphasizes most
> 7. MD&A narrative on revenue growth and margin changes
> 8. Cash flow: CFO, CapEx, FCF, SBC (all TTM or most recent FY)
> 9. Balance sheet: total debt, cash, net debt, operating lease obligations, total equity
> 10. Any insider ownership % mentioned
>
> Preserve all numbers exactly. Date every figure with the fiscal year or quarter. Do not editorialize or summarize beyond what the filing says. If any item is not disclosed, write "not disclosed in source".

## Non-US filer detection

If the company files a **20-F** instead of a 10-K, it is a foreign private issuer (FPI). The stockwiz scope is US equities only, but **ADRs (American Depositary Receipts) that file a 20-F are borderline**:
- TSM (Taiwan Semiconductor) — 20-F, ADR, sometimes analyzed in US equity research
- SAP (SAP SE) — 20-F, ADR, more clearly non-US

**Decision rule for Phase 1:** if the ticker has only 20-F filings and no 10-K, mark the source file with `status: ok` but `reason: non-us-filer`, proceed with the rest of the deep-dive, and include a warning in the final output that this is an FPI and some US-style metrics may not apply.

## Fallback

None. SEC EDGAR is the ground truth. If it fails:
- Try once more after 5 seconds
- If still failing, abort the deep-dive with error `sec-edgar-unreachable`
- Do not trust the remaining sources without the filing context

## Calls

2 total WebFetch calls in the happy path:
1. Filing index page
2. Primary 10-K document

Optional 3rd call for latest 10-Q if needed.
