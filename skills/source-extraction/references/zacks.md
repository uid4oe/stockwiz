# Zacks

**Status:** active. Unique data points (Zacks Rank, Style Scores) that are not replicated by any other free source. Historically Cloudflare-challenged — we fail fast on challenge pages and skip rather than fighting.
**Access method:** **`Bash` + `curl`** with browser User-Agent. Aggressive failure detection.
**Rate policy:** 1500ms delay; **zero retries** on Cloudflare challenge (they don't clear in seconds).

## What it gives us

Zacks publishes a quote page with:

- **Zacks Rank** — 1 to 5 integer, where 1 = "Strong Buy" and 5 = "Strong Sell" (in their language; we capture the integer, wrap the label in `<q>` at render). Their rank is based on earnings-estimate revisions and is widely cited in financial research.
- **Zacks Style Scores** — A through F letter grades on four axes: Value, Growth, Momentum, VGM (composite). Proprietary quantitative scoring.
- **Earnings ESP** (Expected Surprise Percentage) — how much Zacks thinks earnings will beat or miss consensus on the next report
- **Estimate revision counts** — number of analysts revising up vs down over last 7d / 30d / 60d / 90d
- **Industry rank** — Zacks's ranking of the stock's industry against all industries (1-250 scale)

The Zacks Rank and Style Scores are the most unique value. The estimate revisions are duplicated elsewhere (Finviz shows some, Yahoo's quoteSummary has some), but Zacks normalizes them consistently.

## The Cloudflare situation

Zacks aggressively uses Cloudflare Bot Management. Symptoms:
- Sometimes the direct page fetch returns the real HTML — you see Zacks Rank, scores, etc.
- Sometimes the direct fetch returns a Cloudflare challenge page (`challenges.cloudflare.com`, "Attention Required", "Checking your browser")
- Behavior can change week to week as Cloudflare updates its detection

**stockwiz's policy:** fail fast. If a challenge is detected, mark the source failed with reason `cloudflare-challenge` and move on. Do NOT retry — challenges don't clear in 1.5s. Do NOT try clever header variations — they usually fail too and burn the fetch budget.

A future phase can add headless Chrome fallback for Zacks, but today stockwiz accepts Zacks as a best-effort source that may or may not yield data on a given run.

## URL patterns

Primary quote page:
```
https://www.zacks.com/stock/quote/{TICKER}
```

Secondary (sometimes has additional data):
```
https://www.zacks.com/stock/research/{TICKER}/key-stock-data
```

fetch the primary only. If it succeeds and has all the key fields, you're done. If it's missing the Style Scores, optionally try the secondary page.

## Fetch

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'

curl -sS -L \
  -A "$BROWSER_UA" \
  --max-time 25 \
  -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9" \
  -H "Accept-Language: en-US,en;q=0.9" \
  -H "Referer: https://www.google.com/" \
  "https://www.zacks.com/stock/quote/${TICKER}" \
  > /tmp/stockwiz-zacks-${TICKER}.html
```

Note the extra headers: `Accept-Language`, `Referer: https://www.google.com/`. These increase the chance of looking like a real browser click from search. They help a little but don't guarantee success.

After fetch, **immediately check for Cloudflare challenge** before doing any extraction:

```bash
if grep -q "challenges.cloudflare.com\|Attention Required\|Checking your browser\|cf-browser-verification" /tmp/stockwiz-zacks-${TICKER}.html; then
  echo "cloudflare-challenge" > /tmp/stockwiz-zacks-${TICKER}.status
  # Write failure stub and move on; do NOT retry
fi
```

If no challenge detected, proceed to lynx extraction:

```bash
lynx -dump -stdin -width=120 -nolist < /tmp/stockwiz-zacks-${TICKER}.html > /tmp/stockwiz-zacks-${TICKER}.txt
```

## What to extract

From the lynx dump or raw HTML, extract:

- **Zacks Rank** — integer 1-5, sometimes shown as "Zacks Rank #2" or as an image with alt text
- **Zacks Rank text label** — "Strong Buy", "Buy", "Hold", "Sell", "Strong Sell" — capture verbatim for `<q>` wrapping at render
- **Style Scores** — four letters (e.g. "Value A, Growth A, Momentum B, VGM A")
- **Industry** — the industry Zacks assigns (sometimes more granular than Finviz)
- **Industry Rank** — e.g. "Top 8% (17 out of 250)"
- **Zacks's 1-year target** — if shown
- **Forward P/E, PEG, P/S** — Zacks sometimes shows these on the quote page (cross-check with other sources)
- **Estimate revisions** — count of up vs down revisions over the last 7 / 30 / 60 days
- **Earnings ESP** — percentage number (positive or negative)
- **Most recent earnings surprise** — actual vs consensus
- **Average volume** and **52-week range** — redundancy cross-check

## Writing the raw file

```markdown
---
source: Zacks
url: https://www.zacks.com/stock/quote/NVDA
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Zacks — NVDA

## Identity

- Company: NVIDIA Corporation
- Ticker: NVDA
- Exchange: NASDAQ
- Industry: Semiconductor - General (Zacks classification)
- Industry Rank: Top 11% (28 / 250)

## Zacks Rank

- Rank: 2
- Label (verbatim): "Buy"
- Rank as of: {date if shown}

## Style Scores

- Value: C
- Growth: A
- Momentum: B
- VGM (composite): B

## Earnings

- Earnings ESP: +2.34%
- Last reported surprise: +5.54% (EPS) / +3.03% (Sales)
- Next earnings date: May 27, 2026

## Estimate revisions (consensus analyst EPS)

- Current year EPS estimate: $11.07
- 7-day revision: +0.08
- 30-day revision: +0.24
- 60-day revision: +0.51
- 90-day revision: +0.95
- Direction: strongly positive (all windows positive)
- Revision counts (up / down): 7d: 8/0, 30d: 22/1, 60d: 31/2, 90d: 38/3

## Valuation (cross-check)

- Forward P/E: 17.04
- PEG: 0.44
- P/S: 21.23
```

If Zacks Rank's label contains "Buy" or "Sell", capture verbatim — the report-writer wraps it in `<q>` tags at render time so the compliance pass doesn't rewrite it.

## Failure modes (detailed)

| Condition | Reason | Action |
|---|---|---|
| `challenges.cloudflare.com` or "Attention Required" in response body | `cloudflare-challenge` | Mark failed, **no retry**, move on |
| HTTP 403 | `403` | Mark failed, no retry |
| HTTP 429 | `429` | 10s backoff, one retry, then mark failed |
| Empty response or <500 bytes | `empty-response` | Mark failed |
| Page loads but Zacks Rank is not present in content | `zacks-data-missing` | Mark source ok but note partial yield (maybe a restructured page) |
| Page loads and has data for wrong ticker (Zacks occasionally redirects) | `redirect-mismatch` | Mark failed |

**Maximum fetch attempts per session:** 2 (primary URL + one retry on 429 only). No additional attempts. No clever workarounds. If Zacks isn't cooperating on this run, accept it and move on — Zacks is a "nice to have" source, not a critical one.

## Slug

Write to `raw/zacks-snapshot.md`.

## Calls

- Happy path: **1 curl call** (primary quote page)
- With retry on 429: 2 curl calls
- With secondary page for style-score backfill: 2-3 calls (usually stops at 1-2)

## Strategic note

Because Zacks is unreliable, stockwiz does not depend on it for any critical path. Its value is additive: when Zacks works, we get a unique proprietary rating to compare against SA Quant and Finviz Analyst Recom. When Zacks doesn't work, the thesis can still be built from the other 8 sources with no material loss.

The deep-researcher agent should log Zacks success/failure rate in `_sanity.md` so users can tell over time whether Zacks is consistently unreachable in their environment.
