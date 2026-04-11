# Changelog

All notable changes to stockwiz are documented in this file.

## [0.1.1] — 2026-04-11

### Phase 1.5 — Scrape fixes and sentiment-rich sources

**Fixes**
- **SEC EDGAR** was returning HTTP 403 on all endpoints because WebFetch cannot set a custom User-Agent, and SEC's fair-access policy blocks generic UAs. Switched to `Bash + curl` with a descriptive User-Agent, and replaced 10-K HTML scraping with SEC's structured XBRL JSON APIs (`company_tickers.json` → `data.sec.gov/submissions/...` → `data.sec.gov/api/xbrl/companyconcept/...`). Ticker → CIK mapping is now cached at `~/.claude/stockwiz/cache/company_tickers.json` for 30 days. First run fetches 7 SEC calls; subsequent runs fetch 6 (tickers cached).
- **Yahoo Finance** was returning HTTP 503 because its HTML pages are client-rendered and reject scraper UAs, and the first attempt was getting an empty JavaScript shell. Switched to `Bash + curl` using a browser User-Agent with a consent-cookie dance, targeting Yahoo's `query1.finance.yahoo.com/v10/finance/quoteSummary/...` JSON API with `summaryDetail`, `defaultKeyStatistics`, `financialData`, `price`, `assetProfile`, and the three history modules. Includes crumb-token retry path for occasional auth errors.

**New sources brought forward from Phase 2**
- **Simply Wall Street** — snowflake scores, SWS-flagged risks and rewards, narrative verdict. WebSearch to resolve canonical URL, curl + lynx dump to extract.
- **Seeking Alpha (public sections only)** — Quant Rating, Factor Grades (A+–F on Valuation/Growth/Profitability/Momentum/Revisions), Wall Street analyst tier counts. curl + lynx dump. Paywalled article bodies are not fetched.
- **Stockanalysis.com** — 5-year income statement, statistics snapshot, overview. curl + lynx dump. Serves as a structural fallback for Yahoo Finance when the Yahoo JSON API is unavailable.

**Infrastructure**
- `deep-researcher` now uses **mixed tooling**: WebFetch for Finviz (the one source where it works), Bash+curl for the 5 gated/JS-heavy sources. Prerequisites checked at start: curl (hard dependency), jq-or-python3 (for SEC JSON parsing), lynx (soft dependency — falls back to raw HTML Read). Cookie jars cleaned up after Yahoo fetches.
- Rate-limit envelope widened from 20 to **25 total fetches** (reflecting SEC's multi-API-call structure) + 2 WebSearch. Wall-clock budget 60–90s.
- Updated failure-mode detection table with `503`, `api-error: ...`, curl exit codes.
- Updated fallback chains to reflect Phase 1.5 source set.
- Orchestrator's Step 6 tightened: SEC EDGAR failure is fatal (jumps to error path); <3 total successes is also fatal; other individual source failures are non-fatal.

**Known Phase 1.5 limitations**
- TradingView deferred to Phase 2 — needs headless-Chrome fallback for JS-rendered content.
- Zacks, Macrotrends, Google Finance, Reddit also deferred to Phase 2.
- SEC 10-K prose (business description, risk factors, MD&A narrative) not extracted yet — only structured XBRL facts. Phase 2 will add 10-K HTML fetch with targeted truncation.

## [0.1.0] — 2026-04-11

### Phase 1 — Walking skeleton
- Plugin scaffolding: `plugin.json`, `README.md`, `LICENSE`, `CHANGELOG.md`, `.gitignore`
- `commands/stockwiz-setup.md` — one-time onboarding
- `commands/stockwiz.md` — orchestrator for deep-dive pipeline
- `agents/deep-researcher.md` — multi-source WebFetch gatherer (SEC EDGAR, Yahoo Finance, Finviz in Phase 1)
- `agents/report-writer.md` — HTML artifact producer with compliance pass
- `skills/source-extraction/SKILL.md` + references for sec-edgar, yahoo-finance, finviz
- `skills/thesis-discipline/SKILL.md` — forces bull/base/bear + disconfirmers + kill switches
- `skills/report-generation/SKILL.md` + stripped deep-dive template
- `skills/report-generation/assets/{disclaimer.html, base-styles.css}`
- `skills/report-generation/references/compliance-rules.md` — banned phrase table

### Upcoming
- Phase 2: real analyses (fundamental, sentiment, peer, risk) + devils-advocate + remaining 8 sources
- Phase 3: thesis, compare, bear, pivot commands + peer-scout agent
- Phase 4: revisit, monitor commands + macro-context skill + docs + visual polish
