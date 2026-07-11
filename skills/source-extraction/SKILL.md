---
name: source-extraction
description: This skill should be used when fetching raw equity data from public web sources. Provides URL patterns, per-source extraction targets, access methods (WebFetch vs Bash+curl), fallback behavior, rate-limiting guidance, and failure-mode detection for the stockwiz source set. Do not fetch equity data without consulting this skill first.
version: 0.5.0
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

2. **Politeness is not optional.** Within a fetch shard, wait at least **1500ms** between any two fetches (WebFetch OR curl; they share the budget). Configurable via `~/.claude/stockwiz/config.json`'s `ratelimit_ms`. Use `Bash sleep 1.5` when in doubt. Do not spin-wait or poll. Shards may run concurrently because their sources hit **disjoint origins** — the delay protects each origin, and each origin lives in exactly one shard.

3. **Fail fast, move on.** A single source failing must never block a deep-dive (with the single exception of SEC EDGAR — see below). Mark the source as `failed` in `meta.json.sources`, record the reason, and continue. Maximum one retry per source per run.

4. **Preserve numbers verbatim.** Never round, never reformat, never synthesize. If the page says "$50.12B", write "$50.12B". If the page does not disclose a figure, write "not disclosed in source".

5. **Date every figure.** "Revenue $X (FY2025)" not "Revenue $X". If the date is ambiguous, write "as of fetch time" and record the fetch timestamp in the file's frontmatter.

6. **Never editorialize.** You are a librarian. "This is a great stock" never appears in any file you write. Your job is to capture what the source said, with provenance.

7. **SEC EDGAR is the only fatal source.** If SEC fails, the orchestrator aborts the whole deep-dive (no ground truth = no thesis). If Yahoo, Finviz, SWS, SA, or Stockanalysis fail individually, the deep-dive proceeds with what it has.

## Rate-limit envelope

A full deep-dive fetches 12 sources via **four concurrent fetch shards** (see `commands/stockwiz.md` Step 6 for the shard layout and per-shard budgets). Each shard fits within this envelope:

- **≤12 fetches per shard** (WebFetch + curl combined; the orchestrator's shard prompt may set a lower cap)
- **WebSearch per the shard's stated budget** — shard A: 0, shard B: ≤1 (Macrotrends slug, cache-miss only), shard C: ≤2 (SWS URL resolution + optional Google News company-name fallback), shard D: ≤1 (X.com)
- **1500ms** minimum delay between fetches within the shard

Happy-path total across all shards: ~29–33 fetches + ≤4 WebSearch. Stage 1 wall-clock is `max(shard A..D)`, not the sum. The budget covers multi-page sources (SEC 6 API calls with the tickers cache pre-resolved, Macrotrends 4 pages, Stockanalysis 3 pages).

**Per-source call budget (happy path):**
| Source | Calls |
|---|---|
| SEC EDGAR | 6-7 (tickers cache + submissions + 5 concepts) |
| Finviz | 1 |
| Stockanalysis | 3 |
| Macrotrends | 1 WebSearch + 4 curl = 5 |
| Yahoo Finance | 2-3 (cookie + API + optional crumb) |
| Google Finance + News RSS | 2-3 (1 RSS + 1-2 Finance) |
| Simply Wall Street | 1 WebSearch + 1 curl = 2 |
| Seeking Alpha | 1-2 |
| Zacks | 1 (best-effort) |
| Reddit | 3 (one per sub) |
| Barchart insider trades | 1 WebFetch |
| X.com institutional commentary | 1 WebSearch |
| **Total happy path** | **~29-33 fetches + 3 WebSearch** |

Reduced fetch plans for future commands (thesis / compare / revisit / bear) use smaller source subsets — see `docs/architecture.md` § Reduced fetch plans. The orchestrating command passes an explicit `SOURCES` list to each shard dispatch.

Degraded run: the orchestrator applies the fewer-than-3-successful-sources gate after all shards return and aborts the run with reason `insufficient-sources`. Within a shard, fail fast per source and never spend retries on Cloudflare challenges or 403s — don't waste budget on a clearly blocked environment.

## Source index

The source set currently wires **12 sources**. TradingView remains deferred because it needs a headless browser (JS-rendered). FRED stays on the roadmap it's only called by the macro-context skill, not deep-researcher.

| # | Source | Reference file | Phase | Tool | Keyless? |
|---|---|---|---|---|---|
| 1 | SEC EDGAR | [`references/sec-edgar.md`](references/sec-edgar.md) | | **curl** + SEC JSON APIs | yes |
| 2 | Yahoo Finance | [`references/yahoo-finance.md`](references/yahoo-finance.md) | best-effort | **curl** + quoteSummary JSON API | yes |
| 3 | Google Finance + Google News RSS | [`references/google-finance.md`](references/google-finance.md) | | **curl** + RSS/lynx | yes |
| 4 | Simply Wall Street | [`references/simply-wall-street.md`](references/simply-wall-street.md) | | **curl** + lynx | yes |
| 5 | Zacks | [`references/zacks.md`](references/zacks.md) | (best-effort) | **curl** + lynx | yes |
| 6 | Seeking Alpha | [`references/seeking-alpha.md`](references/seeking-alpha.md) | (public only) | **curl** + lynx | yes |
| 7 | TradingView | `references/tradingview.md` | (roadmap) | likely headless Chrome | yes |
| 8 | Finviz | [`references/finviz.md`](references/finviz.md) | | **WebFetch** | yes |
| 9 | Stockanalysis.com | [`references/stockanalysis.md`](references/stockanalysis.md) | | **curl** + lynx | yes |
| 10 | Macrotrends | [`references/macrotrends.md`](references/macrotrends.md) | | **curl** + lynx | yes |
| 11 | FRED | `references/fred.md` | (roadmap) | curl | optional key |
| 12 | Reddit | [`references/reddit.md`](references/reddit.md) | | **curl** + .json endpoints | yes (rate-limited) |
| 13 | Barchart insider trades | [`references/barchart.md`](references/barchart.md) | best-effort | **WebFetch** | yes |
| 14 | X.com institutional commentary | [`references/x-com.md`](references/x-com.md) | best-effort, whitelist-gated | **WebSearch** (`site:x.com`) | yes |

## Shard assignment (MODE=full)

The orchestrator splits the 12 sources into four shards with **disjoint origins** and dispatches them in parallel. Within a shard, fetch in the listed order — more reliable sources first:

| Shard | Sources, in order | Notes |
|---|---|---|
| **A — ground truth + snapshot** | SEC EDGAR (curl + JSON APIs), Finviz (WebFetch), Barchart insider trades (WebFetch) | SEC failure is fatal — skip the rest of the shard and return immediately. Barchart is best-effort. |
| **B — financial history** | Stockanalysis.com (curl + lynx, 3 pages), Macrotrends (slug cache + curl + lynx, 4 pages) | Macrotrends uses WebSearch only on slug-cache miss. |
| **C — news + vendor views** | Google Finance + News RSS (curl + RSS + lynx), Simply Wall Street (WebSearch + curl + lynx), Seeking Alpha (curl + lynx, public only), Zacks (curl + lynx) | Zacks is best-effort — fail fast on Cloudflare, zero retries. |
| **D — market chatter** | Yahoo Finance (curl + cookies + JSON API), Reddit (curl + .json, 3 subs), X.com institutional commentary (WebSearch, whitelist-gated) | Yahoo and X.com are best-effort; empty X.com `quotes: []` is `status: ok`. |

**FRED is never fetched by `deep-researcher`.** It is only called by the `macro-context` skill when a specific series is needed to contextualize a thesis.

**TradingView is deferred (roadmap)** because it requires headless Chrome to render its client-side data. Its technical rating, analyst ideas, and community comments would be valuable but are unreachable via curl alone.

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

## Fallback chains

Some data points are available from multiple sources. Use fallback chains to get the data from wherever works first.

**Price / basic quote:**
Finviz → Yahoo API → Stockanalysis

**Fundamental ratios (P/E, margins, ROE):**
Finviz → Yahoo API → Stockanalysis

**Financial statements (revenue, FCF, debt):**
SEC EDGAR XBRL facts (authoritative) → Stockanalysis financials → Yahoo API history modules

**Insider & institutional ownership (static % held):**
Finviz → Yahoo API defaultKeyStatistics (heldPercentInsiders/Institutions)

**Insider directional time series (3/6/12-month buy vs sell):**
Barchart insider trades (primary) → Finviz `insider_trans` summary (fallback — single 3m % only, no shape)

**Institutional commentary (wires / sell-side / journalists):**
X.com whitelist (primary, verified handles only) → Google News RSS wire-service headlines (fallback)

**Short interest:**
Finviz → Yahoo API

**Alternative-view / sentiment:**
Simply Wall St (snowflake + risks) → Seeking Alpha (factor grades + quant rating) → Zacks (best-effort rank + style scores)

**News / catalysts:**
Google News RSS (primary, broad aggregation) → Seeking Alpha article teasers → SEC 8-K material events

**Retail sentiment (contrarian signal, weight 0.2):**
Reddit r/stocks + r/wallstreetbets + r/investing

**Long-run cycle context (10Y+ history):**
Macrotrends (primary, 10-20Y) → Stockanalysis (5Y fallback)

Use the fallback chain only when the primary source's reference file tells you to. Don't try every source in the chain as a routine — that blows the fetch budget. The shards fetch all 12 sources; the chains matter for downstream analyses when they need a specific field and the primary source was marked failed.

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
| Barchart insider trades | `barchart-insider-trades.md` |
| X.com institutional commentary | `x-com-commentary.md` |

On revisit, prefix with `revisit-YYYYMMDD-`:
- `revisit-20260418-yahoo-fundamentals.md`
- `revisit-20260418-finviz-snapshot.md`
