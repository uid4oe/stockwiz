# Changelog

All notable changes to stockwiz are documented in this file.

## [0.2.0] — 2026-04-11

### Phase 2 — Four analysis skills + adversarial pass

The signal uplift from Phase 1.5 was that we got six sources of raw data. Phase 2 turns that raw data into disciplined analytical outputs, runs an isolated adversarial pass against the synthesized thesis, and renders full HTML sections instead of placeholders. This is the quality jump: the report is no longer "thesis + raw data references", it's a complete research brief.

**New analysis skills** (`skills/<name>/SKILL.md`)
- **`fundamental-analysis`** — reads SEC XBRL facts, stockanalysis.com 5Y history, Finviz ratios, and Yahoo cross-check. Writes `analysis/fundamental.md` with: Snapshot, Valuation Framing (explicit DCF discussion, no single-number output), Quality (multi-year ROIC/margin trends), Growth, Capital Structure, Ownership, Assumption Ledger (mandatory table of forward-looking claims with sensitivities), Unknowns. Hard rule: no single-point intrinsic value, always framing ranges.
- **`sentiment-synthesis`** — reads SWS risks/rewards/narrative, Seeking Alpha factor grades (when SSR-available), Finviz insider/short/analyst, SEC 8-K recent. Writes `analysis/sentiment.md` with explicit weighting rubric (SEC 1.0 through Reddit 0.2, 30-day half-life), Recency-Weighted News, Insider Activity, Short Interest, Analyst Positioning, Conflicting Signals. Hard rule: no numeric sentiment score — conflicts are preserved, not resolved.
- **`peer-comparison`** — reads SWS competitor snowflakes (free Phase 2 peer context) + target metrics. Writes `analysis/peer-comp.md` with Selected Peers + rationale, Comp Table (same columns across rows, direction-of-goodness annotated), Relative Positioning paragraph, mandatory Caveats section on business-mix differences.
- **`risk-screen`** — reads Finviz volatility/beta/technicals, stockanalysis interest coverage, SEC debt, SWS risks as tail signals. Writes `analysis/risk.md` with Volatility & Beta, Drawdown Profile, Correlation (Phase 4 limitation noted), Concentration Risks, Capital-Structure Risks, Tail Risks. Hard rule: descriptive not predictive.

**New subagent** (`agents/devils-advocate.md`)
- Red-colored adversarial subagent that reads ONLY `thesis.md` (anchoring protection — no raw file access by default). Runs in an isolated Task context after thesis synthesis.
- Produces `analysis/devils-advocate.md` with: Weakest Claims (ranked by fragility), The Opposing Narrative (structured like a bull case with its own disconfirmers), Missing Counter-Evidence, Alternative Interpretations, Kill Switch Adequacy (each original kill switch gets a verdict: adequate/vague/slow/missing-trigger/wrong-metric with a proposed rewrite).
- Up to 3 fresh WebFetch/WebSearch calls allowed for finding specific disconfirming evidence.
- Hard rule: never balances, never softens. Explicitly hostile tone as operational protection against collegial equivocation.

**thesis-discipline gains a `reconcile` mode**
- After devils-advocate finishes, the orchestrator invokes thesis-discipline again in `reconcile` mode.
- Reads both `thesis.md` and `analysis/devils-advocate.md`.
- Identifies material devils-advocate findings (specific counter-data with citations, not rhetorical attacks) and appends a `## Adjustments After Stress Test` section to `thesis.md`.
- **Original bull/base/bear claims are preserved verbatim** — adjustments live separately as an audit trail of what the stress test changed.
- If devils-advocate raised no material issues, writes a short section noting the thesis was found internally consistent.

**Deep-dive HTML template fills in the real sections**
- Fundamentals, Sentiment, Peers, Risk, Assumption Ledger are no longer placeholders — each renders the full content from its corresponding `analysis/*.md` file with inline sparklines, tables, citations.
- **Adversarial Appendix** is a new section rendering the full text of `analysis/devils-advocate.md` with a muted-red treatment, clearly labeled as stress test not conclusion.
- **Graceful degradation rule**: if any analysis file is missing (Phase 1.5 fallback or a skill failure), the report renders a thin version of that section from raw data directly, or a placeholder note. The report is always emittable as long as `thesis.md` exists.

**Orchestrator wires Phase 2 stages**
- Pipeline grows from 5 stages to 7:
  1. deep-researcher (unchanged from Phase 1.5)
  2. Four analysis skills sequentially (new)
  3. thesis-discipline `full` mode (now reads from analysis/ instead of raw/)
  4. devils-advocate subagent (was skipped in Phase 1.5)
  5. thesis-discipline `reconcile` mode (new)
  6. report-writer (now renders full sections)
  7. finalize
- Individual analysis failures are non-fatal — thesis-discipline and report-writer both have fallback paths.
- devils-advocate failure is non-fatal — user can retry with `/stockwiz-bear`.
- SEC EDGAR failure remains the only fatal source.
- `commandVersion` bumped to 0.2.0 in new session meta.json files.

**Compliance papercut fixes**
- **"sell-side"** and **"buy-side"** now explicitly exempt from the banned-phrase rewrite pass. These are industry terms for research desks / asset managers, not imperatives. Previously `\bsell\b` matched inside "sell-side" due to the hyphen word boundary, causing false positive rewrites like the one observed in the second test run (`"which sell-side number it is using"` → `"which consensus analyst estimate it is using"` — preserved meaning but lost nuance).
- **Disclaimer footer contents** explicitly exempt — the legal boilerplate "recommendation to buy, sell, or hold" in the stockwiz-disclaimer block is pre-approved and not rewritten.
- New "Explicit exemption list" section in `compliance-rules.md` with a table of industry terms and legal-boilerplate tokens that look like banned phrases under naive regex but should not be rewritten.

**Known Phase 2 limitations (deferred to Phase 3)**
- Analysis skills run sequentially in the main context. Parallelizing them would require wrapping each in a Task subagent, which is a scope expansion; for now the ~30–60s sequential run is acceptable.
- Peers are limited to SWS competitor snowflakes (2–3 peers typically). Phase 3 will add proper peer fetching.
- Correlation analysis needs S&P 500 / sector ETF price series — deferred to Phase 4 macro-context skill.
- SEC 10-K prose (business description, Item 1A risk factors, Item 7 MD&A) still not parsed — only XBRL facts. Customer concentration, segment splits, and management discussion remain Unknown in Phase 2.

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
