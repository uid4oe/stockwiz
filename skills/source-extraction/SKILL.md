---
name: source-extraction
description: This skill should be used when fetching raw equity data from public web sources via WebFetch. Provides URL patterns, per-source extraction targets, access methods, fallback behavior, rate-limiting guidance, and failure-mode detection for the stockwiz source set. Do not call WebFetch for equity data without consulting this skill first.
version: 0.1.0
---

# source-extraction

The data layer for stockwiz. This skill tells the `deep-researcher` agent (and any caller) exactly how to fetch raw equity research from public sources — what URLs to construct, what data points to extract, when to fall back, and how to detect failure.

## Core principles

1. **LLM-based extraction, not DOM selectors.** Every source is fetched via `WebFetch` with a prompt that specifies the data points to extract. No CSS selectors, no regexes over HTML — ask the fetcher's inner model what you want in plain language, and let it read the page like a human. This is resilient to layout changes in a way that selectors are not.

2. **Politeness is not optional.** Wait at least **1500ms** between any two WebFetch calls (adjustable via `~/.claude/stockwiz/config.json`'s `ratelimit_ms`). If you are uncertain whether enough time has passed, run `Bash sleep 1.5`. Do not spin-wait or poll.

3. **Fail fast, move on.** A single source failing must never block a deep-dive. Mark the source as `failed` in `meta.json.sources`, record the reason, and continue. Maximum one retry per source per run.

4. **Preserve numbers verbatim.** Never round, never reformat, never synthesize. If the page says "$50.12B", write "$50.12B". If the page does not disclose a figure, write "not disclosed in source".

5. **Date every figure.** "Revenue $X (FY2025)" not "Revenue $X". If the date is ambiguous, write "as of fetch time" and record the fetch timestamp in the file's frontmatter.

6. **Never editorialize.** You are a librarian. "This is a great stock" never appears in any file you write. Your job is to capture what the source said, with provenance.

## Rate-limit envelope

A full deep-dive (MODE=full) should fit within this envelope:
- **≤20 total WebFetch calls** (hard cap — abort with error if exceeded)
- **≤2 total WebSearch calls** (used only to resolve canonical URLs for sources 4 and 10)
- **45–75 seconds** of wall-clock time
- **1500ms** minimum delay between WebFetch calls

A reduced mode (MODE=thesis, MODE=compare, MODE=revisit) uses a smaller source subset — see each source's reference file for which modes include it.

Degraded run: if **fewer than 3 sources succeed** in the first **5 attempts**, bail early. Set `meta.json.status = "degraded"` or `"failed"` and return to the caller. Don't waste the remaining budget on what is clearly a blocked/broken environment.

## Source index

For Phase 1, only sources 1, 2, and 8 are wired. Remaining sources get reference files in Phase 2.

| # | Source | Reference file | Phase | Keyless? |
|---|---|---|---|---|
| 1 | SEC EDGAR | [`references/sec-edgar.md`](references/sec-edgar.md) | Phase 1 | yes |
| 2 | Yahoo Finance | [`references/yahoo-finance.md`](references/yahoo-finance.md) | Phase 1 | yes (scraped) |
| 3 | Google Finance | `references/google-finance.md` | Phase 2 | yes (scraped) |
| 4 | Simply Wall Street | `references/simply-wall-street.md` | Phase 2 | yes (scraped) |
| 5 | Zacks | `references/zacks.md` | Phase 2 | yes (scraped) |
| 6 | Seeking Alpha | `references/seeking-alpha.md` | Phase 2 | yes (public sections only) |
| 7 | TradingView | `references/tradingview.md` | Phase 2 | yes (scraped) |
| 8 | Finviz | [`references/finviz.md`](references/finviz.md) | Phase 1 | yes (scraped) |
| 9 | Stockanalysis.com | `references/stockanalysis.md` | Phase 2 | yes (scraped) |
| 10 | Macrotrends | `references/macrotrends.md` | Phase 2 | yes (scraped) |
| 11 | FRED | `references/fred.md` | Phase 4 | optional key |
| 12 | Reddit | `references/reddit.md` | Phase 2 | yes (rate-limited JSON) |

## Fetch order for MODE=full

Fetch in this order. Earlier sources are more reliable and give you enough to finish a minimum analysis even if later sources fail:

1. SEC EDGAR — always try first, most reliable
2. Yahoo Finance (quote → key-statistics → financials)
3. Stockanalysis.com (Phase 2)
4. Finviz
5. Macrotrends (Phase 2)
6. Simply Wall Street (Phase 2)
7. Zacks (Phase 2)
8. Seeking Alpha (Phase 2)
9. TradingView (Phase 2)
10. Google Finance (Phase 2)
11. Reddit subs (Phase 2)

**FRED is never fetched by `deep-researcher`.** It is only called by the `macro-context` skill when a specific FRED series is needed to contextualize a thesis.

## Failure-mode detection

Before declaring a fetch successful, check the returned content for these patterns. Any match → mark the source as `failed` with the appropriate reason.

| Pattern in response | Reason code | What it means |
|---|---|---|
| `challenges.cloudflare.com`, `Cloudflare Ray ID`, `Attention Required` | `cloudflare-challenge` | Cloudflare is blocking anonymous access |
| `consent.yahoo.com`, `guce.yahoo.com`, `We value your privacy` | `consent-wall` | A regional consent/GDPR wall is intercepting |
| HTTP 403 / `Forbidden` | `403` | Origin is rejecting the request |
| HTTP 429 / `Too Many Requests` | `429` | Rate-limited |
| Empty body or <200 bytes of content | `empty-response` | Nothing useful returned |
| 3+ redirects or redirect loop | `redirect-loop` | Probably a login wall |
| `Sign in` / `Log in` wall in first 500 bytes, with no substantive data below | `auth-wall` | Hit a paywall or login |
| `Page not found` / `404` | `404` | Ticker doesn't exist on that source |

## Fallback chains

Some data points are available from multiple sources. Use fallback chains to get the data from wherever works first.

**Price / basic quote:**
Yahoo Finance → Stockanalysis → Finviz → Google Finance

**Fundamental ratios (P/E, margins, ROE):**
Yahoo Finance key-statistics → Finviz → Stockanalysis

**Financial statements (revenue, FCF, debt):**
SEC EDGAR 10-K (authoritative) → Yahoo Finance financials → Stockanalysis → Macrotrends

**Insider & institutional ownership:**
Finviz → SEC EDGAR proxy → Yahoo Finance holders page

**Short interest:**
Finviz → Stockanalysis → Yahoo Finance

Use the fallback chain only when the primary source's reference file tells you to. Don't try every source in the chain as a routine — that blows the fetch budget.

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
