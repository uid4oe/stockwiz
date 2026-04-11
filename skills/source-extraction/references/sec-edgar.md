# SEC EDGAR

**Status:** active. 5, always try first. Most reliable source in the set. Authoritative for US filings.
**Access method:** **`Bash` + `curl`** (not WebFetch). SEC requires a descriptive User-Agent header per their [fair-access policy](https://www.sec.gov/os/accessing-edgar-data), and WebFetch does not expose header control. curl does.
**Rate policy:** SEC asks for ≤10 requests/second. stockwiz uses 1500ms delays, well under the limit.

## Why first

SEC EDGAR is the one source that:
- Is legally the primary record — no source disagrees with SEC filings, they're the ground truth
- Does not Cloudflare-challenge, does not consent-wall, does not paywall
- Provides structured XBRL facts via a free JSON API
- Determines whether a ticker is a US filer at all (10-K) or a foreign private issuer (20-F)

If SEC EDGAR fails with curl + descriptive UA, something is very wrong — abort the deep-dive with an error.

## Why curl, not WebFetch

WebFetch in Claude Code sends a default User-Agent that SEC's CDN blocks. SEC's fair-access policy requires requests to identify themselves with a descriptive name and contact (any reasonable string works — SEC does not verify the contact). curl lets us set `-A "stockwiz research plugin noreply@stockwiz.local"` and the 403s disappear.

**This is the pattern for any source that requires header control.** The `deep-researcher` agent uses curl for SEC EDGAR and Yahoo Finance JSON API; WebFetch is still used for Finviz and any source that accepts default UAs.

## approach: structured JSON APIs

Instead of scraping the 10-K HTML page (which is ~10MB and fragile), use SEC's structured data APIs. These return small, versioned JSON and give us the numbers we need for a thesis without parsing 10MB of HTML. The narrative (business description, risk factors, MD&A prose) is deferred to when we'll add proper 10-K HTML fetching with targeted truncation.

## Step-by-step fetch plan

### Step 1 — Resolve ticker → CIK

SEC needs a 10-digit zero-padded CIK (Central Index Key) to identify a company. We map ticker → CIK from SEC's published index:

```bash
curl -sS \
  -A "stockwiz research plugin noreply@stockwiz.local" \
  -H "Accept: application/json" \
  --max-time 30 \
  "https://www.sec.gov/files/company_tickers.json" \
  > ~/.claude/stockwiz/cache/company_tickers.json
```

**Cache this response.** The file is ~12KB and changes slowly (days, not minutes). On subsequent `/stockwiz` runs, reuse the cache if it's less than 30 days old. Cache location: `~/.claude/stockwiz/cache/company_tickers.json`.

The JSON structure is a dict keyed by index:
```json
{
  "0": {"cik_str": 320193, "ticker": "AAPL", "title": "Apple Inc."},
  "1": {"cik_str": 789019, "ticker": "MSFT", "title": "Microsoft Corp"},
  "...": "..."
}
```

Find the entry matching `ticker == TICKER` (case-insensitive), extract `cik_str`, and zero-pad to 10 digits: `1045810` → `0001045810`.

Parse with `jq` if available:
```bash
CIK=$(jq -r --arg t "$TICKER" '
  [.[] | select(.ticker == ($t | ascii_upcase))][0].cik_str
' ~/.claude/stockwiz/cache/company_tickers.json)
CIK_PADDED=$(printf "%010d" "$CIK")
```

If `jq` is not installed, use Python:
```bash
CIK=$(python3 -c "
import json, sys
d = json.load(open('/Users/$(whoami)/.claude/stockwiz/cache/company_tickers.json'))
t = '$TICKER'.upper()
for v in d.values():
    if v['ticker'] == t:
        print(v['cik_str']); break
")
CIK_PADDED=$(printf "%010d" "$CIK")
```

If no match found, the ticker is not a US filer (or is delisted, or typo). Mark SEC EDGAR as `failed` with reason `ticker-not-found-at-sec`. The orchestrator's Step 6 will treat this as fatal — correctly.

### Step 2 — Fetch filing history (submissions API)

```bash
curl -sS \
  -A "stockwiz research plugin noreply@stockwiz.local" \
  -H "Accept: application/json" \
  --max-time 30 \
  "https://data.sec.gov/submissions/CIK${CIK_PADDED}.json" \
  > "$SESSION_DIR/raw/sec-submissions.json"
```

This returns company metadata + a `filings.recent` object with columns of arrays (accession numbers, dates, forms, primary documents). Extract:

- **Company name** (field: `name`)
- **SIC / SIC description** (field: `sic`, `sicDescription`)
- **State of incorporation** (field: `addresses.business.stateOrCountry`)
- **Fiscal year end** (field: `fiscalYearEnd`)
- **Exchanges** (field: `exchanges` array)
- **Most recent 10-K**: scan `filings.recent.form` for `"10-K"`, find the index, grab the corresponding `filingDate`, `accessionNumber`, and `primaryDocument`
- **Most recent 10-Q**: same pattern with `"10-Q"`
- **Most recent 8-K**: same (for drift checks / material events)

If the only annual filing form is `20-F` (foreign private issuer), mark SEC as `ok` but append a note: `non-us-filer-20f-only`. The orchestrator decides how to handle it; we proceed and let the user know via `_sanity.md`.

Construct the 10-K document URL (do NOT fetch it — 10-K HTML prose parsing is on the roadmap):
```
https://www.sec.gov/Archives/edgar/data/{cik_unpadded}/{accession_no_dashes}/{primaryDocument}
```
Record this URL in the raw file for future deep-narrative passes.

### Step 3 — Fetch structured financial facts (companyconcept API)

For each concept below, fetch the XBRL concept timeseries. Each returns ~10–100KB.

```bash
curl -sS \
  -A "stockwiz research plugin noreply@stockwiz.local" \
  -H "Accept: application/json" \
  --max-time 30 \
  "https://data.sec.gov/api/xbrl/companyconcept/CIK${CIK_PADDED}/us-gaap/${CONCEPT}.json"
```

**concept list (5 core concepts, one fetch each):**

| Concept | What it is | Fallback concepts if primary missing |
|---|---|---|
| `Revenues` | Top-line revenue | `RevenueFromContractWithCustomerExcludingAssessedTax` (ASC 606, newer filings) |
| `NetIncomeLoss` | Bottom-line net income | — |
| `NetCashProvidedByUsedInOperatingActivities` | Cash from operations (CFO) | — |
| `PaymentsToAcquirePropertyPlantAndEquipment` | Capital expenditures | `PaymentsToAcquireProductiveAssets` |
| `LongTermDebtNoncurrent` | Long-term debt | `LongTermDebt` |

For each concept, the response includes a `units` object keyed by unit (usually `USD`). Inside, `values` is an array of observations with `start`, `end`, `val`, `fy`, `fp`, `form`, `accn`, `filed`. Extract:

- Most recent **annual** observation (where `fp` is `FY`)
- Most recent **quarterly** observation (where `fp` in `Q1`, `Q2`, `Q3`, `Q4`)
- Preceding **annual** observation (for YoY comparison)

Store all of these verbatim with dates.

**Free Cash Flow** is derived in the analysis layer, not fetched: FCF = CFO − CapEx. Record both raw values; let `fundamental-analysis` compute it with explicit assumption notation.

### Step 4 — Write the raw file

Combine the above into a single markdown file at `$SESSION_DIR/raw/sec-edgar-10k.md`:

```markdown
---
source: SEC EDGAR
url: https://data.sec.gov/submissions/CIK0001045810.json (+ 5 companyconcept calls)
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
cik: 0001045810
---

# SEC EDGAR — NVDA

## Company

- Name: NVIDIA CORP
- CIK: 0001045810
- SIC: 3674 (Semiconductors & Related Devices)
- State of incorporation: DE
- Fiscal year end: 01-28
- Exchanges: NASDAQ

## Most recent filings

- 10-K: filed 2025-02-21, accession 0001045810-25-000023, primary document nvda-20250126.htm
  - URL: https://www.sec.gov/Archives/edgar/data/1045810/000104581025000023/nvda-20250126.htm
- 10-Q: filed 2025-08-28, accession 0001045810-25-000060, primary document nvda-20250727.htm
- 8-K (most recent): filed 2026-02-15

## XBRL facts

### Revenues (us-gaap:Revenues)

- FY2025 (ending 2025-01-26): $130,497,000,000 — filed 2025-02-21 — 10-K
- FY2024 (ending 2024-01-28): $60,922,000,000 — filed 2024-02-21 — 10-K
- Q2 FY2026 (ending 2025-07-27): $46,743,000,000 — filed 2025-08-28 — 10-Q
- YoY (FY2025 vs FY2024): +114.2%

### NetIncomeLoss

- FY2025: $72,880,000,000
- FY2024: $29,760,000,000
- YoY: +144.9%

### NetCashProvidedByUsedInOperatingActivities (CFO)

- FY2025: $64,089,000,000
- FY2024: $28,090,000,000

### PaymentsToAcquirePropertyPlantAndEquipment (CapEx)

- FY2025: $3,236,000,000
- FY2024: $1,069,000,000

### LongTermDebtNoncurrent

- Most recent 10-K: $8,463,000,000

## Derived (for downstream analyses)

- Free Cash Flow (CFO − CapEx), FY2025: $60,853,000,000

## Notes

- 10-K prose (business description, risk factors, MD&A) not extracted; URL recorded above for future deep-narrative passes (roadmap).
```

All numbers preserved verbatim from SEC's JSON, dated by fiscal period. `fundamental-analysis` can consume this directly.

## Failure modes

| curl exit code / HTTP | Reason | What to do |
|---|---|---|
| 22 (HTTP 4xx/5xx with `--fail`) | Non-2xx response | Capture status, record as failure reason |
| 28 | Timeout | Retry once after 5s |
| 6 | DNS failure | Record as `dns-failure`, do not retry |
| HTTP 403 | User-Agent blocked | **Should not happen** with descriptive UA — if it does, escalate: write diagnostic to `_sanity.md` noting the UA in use |
| HTTP 404 | Ticker not in tickers file, or CIK not in submissions API | Mark ticker-not-found-at-sec |
| HTTP 429 | Rate-limited | Retry once with a 10s backoff; if still 429, abort this source and note in _sanity.md |

On any failure that prevents the submissions API from returning data, mark SEC EDGAR as `failed` and let the orchestrator decide (it will treat this as fatal — correctly, since we cannot build a thesis without ground-truth filings).

## Calls

- First run on a given machine: 1 (tickers) + 1 (submissions) + 5 (concepts) = **7 curl calls**
- Subsequent runs (tickers cached): 1 (submissions) + 5 (concepts) = **6 curl calls**

Both fit comfortably within the 35-fetch hard cap.

## Slug

Write structured output to `raw/sec-edgar-10k.md`. Also cache the raw JSON responses to `raw/sec-submissions.json` and `raw/sec-concept-*.json` for debugging — these are the audit trail if downstream analyses question a figure.

## Non-US filer handling

If a ticker is in the tickers file but the submissions API only shows `20-F` (not `10-K`), the company is a foreign private issuer. Some ADRs are in this category (TSM, SAP). We extract whatever structured data is available (many FPIs file some XBRL facts) and flag the source file with a `non-us-filer-20f-only` note. The orchestrator may or may not treat this as fatal depending on how much data was recovered. Be permissive here — it's better to try and partially succeed than to reject valid tickers.

## UA configurability (future)

For marketplace distribution, it would be polite to let users configure the contact in their User-Agent via `~/.claude/stockwiz/config.json`. TODO. For now, the hardcoded string works because SEC doesn't verify contacts — they just need something descriptive.
