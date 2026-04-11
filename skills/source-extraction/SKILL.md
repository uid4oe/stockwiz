---
name: source-extraction
description: This skill should be used when fetching raw equity data from public web sources. Provides URL patterns, per-source extraction targets, access methods (WebFetch vs Bash+curl), fallback behavior, rate-limiting guidance, and failure-mode detection for the stockwiz source set. Do not fetch equity data without consulting this skill first.
version: 0.2.0
---

# source-extraction

The data layer for stockwiz. This skill tells the `deep-researcher` agent (and any caller) exactly how to fetch raw equity research from public sources — what URLs to construct, what tool to use (WebFetch vs Bash+curl), what data points to extract, when to fall back, and how to detect failure.

## Mixed tooling: WebFetch and Bash+curl

stockwiz uses **two** fetch paths depending on what the source's gatekeeping looks like:

**`WebFetch`** — used when a source accepts default User-Agents and returns meaningful HTML directly. Finviz is the canonical example. WebFetch has a built-in model that extracts requested fields from the HTML, which is clean and token-efficient.

**`Bash` + `curl`** — used when a source requires a specific User-Agent header (SEC EDGAR), needs a cookie from a prior page visit (Yahoo Finance), or simply behaves better when accessed as a browser. `curl` gives us full header control, `-c`/`-b` for cookies, and the ability to pipe through `lynx -dump` for clean text extraction from HTML-heavy pages. WebFetch cannot do any of this.

The rule: **use whichever the source's reference file prescribes.** Do not second-guess it. If the reference file says `curl with browser UA`, using WebFetch will produce the exact failures we already saw on the first `/stockwiz NVDA` run (SEC 403, Yahoo 503).

## Core principles

1. **LLM-based extraction, not DOM selectors.** Whether via WebFetch or via curl+`lynx -dump` + a Read pass, extraction is always prompted in plain language asking for specific fields. No CSS selectors, no regexes over raw HTML — ask for what you want and let the model read the page like a human. Layout-resilient.

2. **Politeness is not optional.** Wait at least **1500ms** between any two fetches (WebFetch OR curl; they share the budget). Configurable via `~/.claude/stockwiz/config.json`'s `ratelimit_ms`. Use `Bash sleep 1.5` when in doubt. Do not spin-wait or poll.

3. **Fail fast, move on.** A single source failing must never block a deep-dive (with the single exception of SEC EDGAR — see below). Mark the source as `failed` in `meta.json.sources`, record the reason, and continue. Maximum one retry per source per run.

4. **Preserve numbers verbatim.** Never round, never reformat, never synthesize. If the page says "$50.12B", write "$50.12B". If the page does not disclose a figure, write "not disclosed in source".

5. **Date every figure.** "Revenue $X (FY2025)" not "Revenue $X". If the date is ambiguous, write "as of fetch time" and record the fetch timestamp in the file's frontmatter.

6. **Never editorialize.** You are a librarian. "This is a great stock" never appears in any file you write. Your job is to capture what the source said, with provenance.

7. **SEC EDGAR is the only fatal source.** If SEC fails, the orchestrator aborts the whole deep-dive (no ground truth = no thesis). If Yahoo, Finviz, SWS, SA, or Stockanalysis fail individually, the deep-dive proceeds with what it has.

## Rate-limit envelope

A full deep-dive (MODE=full) should fit within this envelope:
- **≤25 total fetches** (WebFetch + curl, combined hard cap — abort with error if exceeded)
- **≤2 total WebSearch calls** (used only to resolve canonical URLs for sources like Simply Wall St)
- **60–90 seconds** of wall-clock time
- **1500ms** minimum delay between fetches

The Phase 1.5 budget is wider than Phase 1 because SEC's structured-JSON approach uses 6–7 small API calls instead of 2 HTML fetches.

A reduced mode (MODE=thesis, MODE=compare, MODE=revisit) uses a smaller source subset — see each source's reference file for which modes include it.

Degraded run: if **fewer than 3 sources succeed** in the first **8 attempts**, bail early. Set `meta.json.status = "degraded"` or `"failed"` and return to the caller. Don't waste the remaining budget on what is clearly a blocked/broken environment.

## Source index

Phase 1.5 wires **6 sources**: SEC EDGAR, Yahoo Finance (JSON API), Finviz, Simply Wall St, Seeking Alpha, Stockanalysis.com. Remaining sources stay in Phase 2.

| # | Source | Reference file | Phase | Tool | Keyless? |
|---|---|---|---|---|---|
| 1 | SEC EDGAR | [`references/sec-edgar.md`](references/sec-edgar.md) | Phase 1.5 | **curl** + SEC JSON APIs | yes |
| 2 | Yahoo Finance | [`references/yahoo-finance.md`](references/yahoo-finance.md) | Phase 1.5 | **curl** + quoteSummary JSON API | yes |
| 3 | Google Finance | `references/google-finance.md` | Phase 2 | TBD | yes |
| 4 | Simply Wall Street | [`references/simply-wall-street.md`](references/simply-wall-street.md) | Phase 1.5 | **curl** + lynx | yes |
| 5 | Zacks | `references/zacks.md` | Phase 2 | TBD (likely curl) | yes |
| 6 | Seeking Alpha | [`references/seeking-alpha.md`](references/seeking-alpha.md) | Phase 1.5 (public only) | **curl** + lynx | yes |
| 7 | TradingView | `references/tradingview.md` | Phase 2 | likely headless Chrome | yes |
| 8 | Finviz | [`references/finviz.md`](references/finviz.md) | Phase 1 | **WebFetch** | yes |
| 9 | Stockanalysis.com | [`references/stockanalysis.md`](references/stockanalysis.md) | Phase 1.5 | **curl** + lynx | yes |
| 10 | Macrotrends | `references/macrotrends.md` | Phase 2 | TBD | yes |
| 11 | FRED | `references/fred.md` | Phase 4 | curl | optional key |
| 12 | Reddit | `references/reddit.md` | Phase 2 | curl + .json endpoints | yes (rate-limited) |

## Fetch order for MODE=full (Phase 1.5)

Fetch in this order. Earlier sources are more reliable and give you enough to finish a minimum analysis even if later sources fail:

1. **SEC EDGAR** (curl + JSON) — always try first. Fatal if it fails.
2. **Finviz** (WebFetch) — single page, most reliable scraped source, cheap to try early so we have quick confirmation the pipeline works.
3. **Stockanalysis.com** (curl + lynx) — 5Y financials, structural backfill for Yahoo.
4. **Yahoo Finance** (curl + JSON API) — fundamentals and statistics.
5. **Simply Wall Street** (WebSearch + curl + lynx) — snowflake + risks + narrative.
6. **Seeking Alpha** (curl + lynx) — quant rating + factor grades.

**FRED is never fetched by `deep-researcher`.** It is only called by the `macro-context` skill when a specific FRED series is needed to contextualize a thesis (Phase 4).

The order puts SEC and Finviz first — they are our two most reliable sources. Stockanalysis comes before Yahoo because Yahoo's JSON API is flakier. SWS and SA are last because they're the most likely to be blocked.

## Failure-mode detection

Before declaring a fetch successful, check the returned content for these patterns. Any match → mark the source as `failed` with the appropriate reason.

With curl, check `curl -w "%{http_code}"` and the response body. With WebFetch, inspect the returned content for the patterns.

| Pattern in response | Reason code | What it means |
|---|---|---|
| `challenges.cloudflare.com`, `Cloudflare Ray ID`, `Attention Required` | `cloudflare-challenge` | Cloudflare is blocking anonymous access |
| `consent.yahoo.com`, `guce.yahoo.com`, `We value your privacy` | `consent-wall` | A regional consent/GDPR wall is intercepting |
| HTTP 403 / `Forbidden` | `403` | Origin rejecting the request — for SEC this means the User-Agent is wrong; for others it usually means a WAF/CDN block |
| HTTP 429 / `Too Many Requests` | `429` | Rate-limited; 10s backoff then one retry, then fail |
| HTTP 503 / `Service Unavailable` | `503` | Temporary backend failure; one retry at 5s backoff, then fail |
| Empty body or <200 bytes of content | `empty-response` | Nothing useful returned (common with JS-only pages like Yahoo HTML) |
| 3+ redirects or redirect loop | `redirect-loop` | Probably a login wall |
| `Sign in` / `Log in` wall in first 500 bytes, with no substantive data below | `auth-wall` | Hit a paywall or login |
| `Page not found` / `404` | `404` | Ticker doesn't exist on that source |
| JSON with `error` field non-null (Yahoo API) | `api-error: <message>` | Yahoo-specific; may need crumb retry |
| curl exit code 28 | `timeout` | Network timeout; one retry, then fail |
| curl exit code 6 | `dns-failure` | DNS didn't resolve; do not retry |

## Fallback chains (Phase 1.5)

Some data points are available from multiple sources. Use fallback chains to get the data from wherever works first.

**Price / basic quote:**
Finviz → Yahoo API → Stockanalysis

**Fundamental ratios (P/E, margins, ROE):**
Finviz → Yahoo API → Stockanalysis

**Financial statements (revenue, FCF, debt):**
SEC EDGAR XBRL facts (authoritative) → Stockanalysis financials → Yahoo API history modules

**Insider & institutional ownership:**
Finviz → Yahoo API defaultKeyStatistics (heldPercentInsiders/Institutions)

**Short interest:**
Finviz → Yahoo API

**Alternative-view / sentiment:**
Simply Wall St (snowflake + risks) → Seeking Alpha (factor grades + quant rating)

Use the fallback chain only when the primary source's reference file tells you to. Don't try every source in the chain as a routine — that blows the fetch budget. The orchestrator fetches all 6 sources in order; the chains matter for downstream analyses when they need a specific field and the primary source was marked failed.

## File output convention

Every fetched source gets written to `SESSION_DIR/raw/<source-slug>.md` with this frontmatter format:

```markdown
---
source: <human-readable source name>
url: <the actual URL you fetched>
fetched_at: <ISO 8601 timestamp with offset>
status: ok
---

# {Source name} — {TICKER}

{Extracted content, structured as the source's reference file specifies.
Preserve all numbers verbatim. Date every figure. Do not editorialize.}
```

On failure, still write the file with `status: failed` and a `reason:` field, leaving the body as:

```markdown
---
source: <name>
url: <attempted URL>
fetched_at: <timestamp>
status: failed
reason: cloudflare-challenge
---

# {Source name} — {TICKER}

_Source unavailable at fetch time. No data extracted._
```

This lets later consumers (analyses, report-writer) grep for `status: ok` in frontmatter and know which files are real.

## Slugs (filename conventions)

| Source | Slug filename |
|---|---|
| SEC EDGAR 10-K | `sec-edgar-10k.md` |
| SEC EDGAR 10-Q | `sec-edgar-10q.md` |
| Yahoo Finance | `yahoo-fundamentals.md` |
| Google Finance | `google-finance.md` |
| Simply Wall St | `simply-wall-street.md` |
| Zacks | `zacks-snapshot.md` |
| Seeking Alpha | `seeking-alpha.md` |
| TradingView | `tradingview.md` |
| Finviz | `finviz-snapshot.md` |
| Stockanalysis | `stockanalysis.md` |
| Macrotrends | `macrotrends.md` |
| FRED (per series) | `fred-{series-id}.md` |
| Reddit (per sub) | `reddit-{sub}.md` |

On revisit, prefix with `revisit-YYYYMMDD-`:
- `revisit-20260418-yahoo-fundamentals.md`
- `revisit-20260418-finviz-snapshot.md`
