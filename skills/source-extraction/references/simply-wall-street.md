# Simply Wall St

**Status:** active. High signal-to-noise source for a compact quality/risk view, via their public "snowflake" summary and written narratives on each stock page.
**Access method:** **`Bash` + `curl`** with browser User-Agent. SWS is not Cloudflare-gated but serves a JavaScript-lite SSR that requires a realistic UA to return useful HTML.
**Rate policy:** 1500ms delay; one retry on transient failure.

## What it gives us

Simply Wall St's value for stockwiz is:

- **Snowflake scores** (0–6 on each axis): Value, Future, Past, Health, Dividends — a compact five-dimensional read on the stock's character
- **Analyst price target range** (if shown in the public view): low, average, high
- **SWS-flagged risks and rewards** — 3–8 short bullets each, captured verbatim. These are often the best single source of "what the consensus is actually watching"
- **Short SWS narrative / verdict** — a 2–4 sentence summary

The snowflake scores alone are worth the fetch: they're a compact alternative-view signal that complements Finviz's technicals and SEC's numbers.

## URL resolution — search first

SWS uses a slug-based URL pattern that requires the sector and company slug:

```
https://simplywall.st/stocks/us/{sector-slug}/{exchange}-{ticker}/{company-slug}
```

We don't know the sector or company slug a priori. Two resolution strategies:

### Strategy A — Use WebSearch to find the canonical URL (recommended)

Call the `WebSearch` tool with query `site:simplywall.st {TICKER}` and grab the first result URL. This costs 1 `WebSearch` call against your 2-call budget but is the most reliable way to get the right page.

### Strategy B — Use SWS search endpoint (fallback)

SWS has a search endpoint at `https://simplywall.st/api/search?query={ticker}` but it's undocumented and returns JSON that may or may not include the canonical URL directly. Use this only if WebSearch is unavailable. If it fails, skip Simply Wall St.

## Fetch

Once you have the canonical URL (e.g. `https://simplywall.st/stocks/us/semiconductors/nasdaq-nvda/nvidia`):

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'

curl -sS -L \
  -A "$BROWSER_UA" \
  --max-time 30 \
  -H "Accept: text/html,application/xhtml+xml" \
  "$SWS_URL" \
  > /tmp/stockwiz-sws-${TICKER}.html
```

Then either:

**A) Use WebFetch on the file URL** — not supported; WebFetch only takes HTTP URLs.

**B) Read the file and parse with an LLM pass.**

Since we saved the HTML to a tempfile, we `Read` the file with a line/byte limit to capture the useful section. SWS pages are typically 100–400KB. Read the first ~50KB and ask the skill-running model to extract the target fields.

Or, the cleanest approach: use `curl` piped through a text extractor like `lynx -dump -stdin` or `w3m -dump -T text/html` to get a text-only view, saving tokens:

```bash
curl -sS -L -A "$BROWSER_UA" --max-time 30 "$SWS_URL" | \
  lynx -stdin -dump -width=120 -nolist > /tmp/stockwiz-sws-${TICKER}.txt
```

`lynx` is preinstalled on macOS and most Linux distros. If it's not available, fall back to saving raw HTML and letting the skill model skim it.

## What to extract

Ask the parsing model (whether via Read + prompt or via lynx + Read) to extract exactly these fields:

- **Company / ticker / exchange** — confirm we're on the right page
- **Snowflake scores**: Value, Future, Past, Health, Dividends (each 0–6 integer; if the dividend axis is absent the company doesn't pay, mark it 0)
- **Current price + SWS "fair value estimate"** (if shown)
- **Analyst target range**: low / consensus / high, plus number of analysts
- **"Rewards" list** — 3–8 bullets, captured verbatim. These are SWS's distilled bull points.
- **"Risks" list** — 3–8 bullets, captured verbatim. These are the most valuable part for thesis-discipline.
- **SWS verdict / narrative** — the 2–4 sentence summary that SWS shows above the snowflake. Capture verbatim in a `<q>...</q>`-safe way (no imperative language from SWS gets rewritten).
- **Last update date** of the SWS page — if visible.

Preserve all snowflake scores as the raw integers SWS uses (0/1/2/3/4/5/6). Do NOT translate them to labels like "strong" or "weak" — those introduce interpretation.

## Writing the raw file

```markdown
---
source: Simply Wall St
url: https://simplywall.st/stocks/us/semiconductors/nasdaq-nvda/nvidia
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
---

# Simply Wall St — NVDA

## Identity

- Company: NVIDIA Corporation
- Ticker: NASDAQ:NVDA
- SWS page last updated: 2026-04-09

## Snowflake scores (0–6 scale)

- Value: 1
- Future: 6
- Past: 6
- Health: 6
- Dividends: 1

## Pricing view

- Current price: US$188.63
- SWS fair value estimate (if shown): US$215.40
- Analyst target range (N analysts): low US$195.00 / consensus US$269.16 / high US$340.00

## SWS rewards (verbatim)

1. Trading at {X}% above our estimate of fair value
2. Earnings are forecast to grow 25.8% per year
3. Became profitable this year
4. ...

## SWS risks (verbatim)

1. Significant insider selling over the past 3 months
2. Shareholders have been diluted in the past year
3. Large one-off items impacting financial results
4. ...

## SWS narrative (verbatim)

> {full text of SWS summary, preserved for <q>-wrapping in the report}
```

Preserve the verbatim lists. The report-writer will display SWS risks directly in the sentiment section — they're a high-signal input that doesn't fit neatly into Finviz's numeric fields.

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| WebSearch returns no SWS result | `ticker-not-on-sws` | Skip source |
| curl HTTP 403/429 | `sws-blocked` | Retry once with 10s backoff; then skip |
| curl times out | `timeout` | Retry once; then skip |
| Page content shows "This stock is not covered" | `sws-not-covered` | Mark as ok but with empty extract |
| lynx unavailable and HTML too large | `extractor-missing` | Fall back to Read raw HTML with limit |

Skipping SWS is not fatal — it's a supplementary source.

## Slug

Write to `raw/simply-wall-street.md`.

## Calls

- 1 `WebSearch` call (URL resolution)
- 1 `curl` call (page fetch)
- Total: **2 fetches** per deep-dive

## A compliance note

SWS's rewards and risks lists often contain words like "undervalued", "overvalued", "expensive", "buy opportunity", etc. **Capture them verbatim in the raw file.** The report-writer wraps them in `<q>` tags in the final HTML, which exempts them from the compliance rewrite pass. Never rewrite source labels at the fetch stage — the audit trail requires verbatim capture.

## On the sentiment value

Simply Wall St is one of the three "sentiment-rich" sources (SWS, Seeking Alpha, TradingView) that give stockwiz its character beyond pure numbers. The SWS snowflake is a compact alternative view; the risks list is often where stockwiz's thesis-discipline agent pulls half its bear-case inputs from (via the sentiment-synthesis agent's capture of the SWS risks list). Treat this source as important even though it's "soft" data.
