# Seeking Alpha

**Status:** active. Valuable for the Quant Rating and Factor Grades on each symbol page. Article bodies and "Wall Street Ratings" are paywalled — we only capture what's free and public on the symbol landing page.
**Access method:** **`Bash` + `curl`** with browser User-Agent. SA is not Cloudflare-gated on the symbol page but throttles scraper-looking UAs.
**Rate policy:** 1500ms delay; one retry on transient failure.

## What we capture (public only)

Seeking Alpha's symbol landing page (`seekingalpha.com/symbol/{ticker}`) has a few things visible without login:

- **SA Quant Rating** — a single label ("Strong Buy", "Buy", "Hold", "Sell", "Strong Sell") and a numeric score (1.0–5.0). Capture verbatim. Downstream compliance will wrap it in `<q>` tags.
- **SA Factor Grades** — A+ through F letter grades on five axes: Valuation, Growth, Profitability, Momentum, Revisions. These are SA's proprietary scoring and one of the cleanest alternative views in the free web.
- **Wall Street consensus** — count of analysts at each rating tier (Strong Buy N, Buy M, Hold K, Sell L, Strong Sell P). Usually visible as a small widget.
- **Next earnings date** — usually shown in the top summary.
- **Short article teasers** — the first 1–2 sentences of recent articles. Do NOT fetch article bodies (paywalled + legally questionable). Only the teaser text that's visible on the symbol page is in-bounds.

We do NOT capture:
- Full article bodies (paywalled)
- SA Premium features
- Comment sections
- Historical rating changes beyond what's shown

## URL patterns

### Primary: symbol landing page

```
https://seekingalpha.com/symbol/{ticker}
```

### Secondary (optional): ratings page

```
https://seekingalpha.com/symbol/{ticker}/ratings
```

Has a more detailed quant rating breakdown. Often has the same data as the symbol page in slightly different form. Fetch only if the primary is incomplete.

## Fetch

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'

curl -sS -L \
  -A "$BROWSER_UA" \
  --max-time 30 \
  -H "Accept: text/html,application/xhtml+xml" \
  -H "Accept-Language: en-US,en;q=0.9" \
  "https://seekingalpha.com/symbol/${TICKER}" \
  | lynx -stdin -dump -width=120 -nolist \
  > /tmp/stockwiz-sa-${TICKER}.txt
```

As with Simply Wall St, pipe through `lynx -stdin -dump` to collapse the HTML to text. If lynx is missing, save raw HTML and Read it with a byte limit.

## What to extract

Parse the text-dumped file (via Read + prompt) and extract:

- **Company name + ticker** — confirmation
- **Current price** (if shown)
- **SA Quant Rating** — both the label string (e.g. `"Strong Buy"`) and the numeric score (e.g. `4.75`). Capture both verbatim.
- **SA Factor Grades** — the letter grade for each of: Valuation, Growth, Profitability, Momentum, Revisions. Store as strings (`"A+"`, `"B-"`, etc.)
- **Wall Street analyst distribution** — counts at each tier
- **Number of analysts total**
- **Next earnings date** (if shown on the symbol page)
- **Short description** — the 1–2 sentence description SA shows above the chart

## Writing the raw file

```markdown
---
source: Seeking Alpha
url: https://seekingalpha.com/symbol/NVDA
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Seeking Alpha — NVDA

## Identity

- Company: NVIDIA Corporation
- Ticker: NVDA
- Exchange: NASDAQ
- Current price shown on page: 188.63

## SA Quant Rating

- Label (verbatim): "Strong Buy"
- Score: 4.75 (on 1.0–5.0 scale, higher = stronger rating)

## SA Factor Grades

- Valuation: C-
- Growth: A+
- Profitability: A+
- Momentum: A
- Revisions: A

## Wall Street analyst distribution

- Total analysts: 48
- Strong Buy: 32
- Buy: 12
- Hold: 4
- Sell: 0
- Strong Sell: 0

## Next earnings

- Date: 2026-05-21 (after market close)

## Short description

> {1–2 sentence company blurb SA shows above the chart, captured verbatim}
```

Again: capture rating labels verbatim. The report-writer handles wrapping in `<q>` tags and framing neutrally.

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| HTTP 403 / 429 | `sa-blocked` | Retry once with 10s backoff; then skip |
| HTTP 5xx | `sa-5xx` | Retry once; then skip |
| Page redirects to login | `sa-login-wall` | Skip — we don't log in |
| "This page does not exist" in body | `ticker-not-on-sa` | Skip |
| Empty text dump | `empty-response` | Skip |

Skipping Seeking Alpha is never fatal. It's a supplementary source, though a high-value one.

## Slug

Write to `raw/seeking-alpha.md`.

## Calls

- 1 curl call (primary page)
- Optional 1 more (ratings page) if primary was incomplete
- Total: **1–2 fetches** per deep-dive

## On rating labels and compliance

This is the single most compliance-hazardous source in the set. Every SA Quant Rating and every analyst tier label contains the words "Buy" or "Sell", which stockwiz's compliance pass rewrites aggressively. **The solution is verbatim capture + `<q>` wrapping**, as documented in `compliance-rules.md`:

In the raw file, just write the label as-is: `"Strong Buy"`, `"4 Strong Sell"`, etc. No quote tags at the fetch stage.

In the final HTML report, the report-writer agent wraps these in `<q>` tags:
```html
<p>Seeking Alpha's Quant model assigns this ticker its <q>Strong Buy</q> label
(4.75 on the 1–5 scale)<sup>...</sup>. The SA methodology is a quantitative
composite of 100+ fundamental and technical factors.</p>
```

Inside `<q>`, the compliance pass does not rewrite. The surrounding prose is stockwiz's own framing, which must use neutral language ("SA's highest category", "top of the 1–5 scale", etc.).

This pattern applies to every rating label from every source — Yahoo's `recommendationKey`, Zacks Rank, TradingView technical rating — but SA is the most common place it matters because the ratings are front-and-center on the page.
