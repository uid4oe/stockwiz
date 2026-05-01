# stockwiz

A Claude Code plugin for deep-dive US equity research. Produces disciplined bull/bear/base theses, adversarial devil's-advocate critique, peer comparison, and self-contained HTML artifacts you can open offline and share.

## What it is

A research copilot for discretionary investors. stockwiz aggregates public information from ~11 free web sources, synthesizes it through domain skills (fundamentals, sentiment, peers, risk, macro), forces honest thesis discipline (every case has explicit disconfirmers and measurable kill switches), runs an isolated adversarial pass to stress-test conclusions, and emits a single self-contained HTML report per analysis. Session workspaces are durable on disk, so you can ask follow-up questions after an analysis completes without re-fetching anything.

## What it is not

- **Not investment advice.** stockwiz is an analytical tool. Nothing it produces is a recommendation to buy, sell, or hold any security.
- **Not real-time.** Data is fetched on demand from public pages and is inherently delayed.
- **Not an alpha-generation machine.** Value comes from faster synthesis and disciplined framing — not from beating the market.
- **Not a trading system.** No brokerage integration, no auto-execution, no position sizing.

## Install

```
/plugin install stockwiz
```

Then run the one-time setup:

```
/stockwiz-setup
```

## First run

```
/stockwiz NVDA
```

Open the resulting `~/.claude/stockwiz/sessions/NVDA-*/report.html` in any browser. Ask follow-up questions in the same Claude Code session — they'll be answered from the workspace without re-fetching.

## Commands

| Command | Purpose |
|---|---|
| `/stockwiz-setup` | One-time onboarding; creates data dirs, shows disclaimer |
| `/stockwiz <ticker> [--horizon=long\|swing]` | Full deep-dive pipeline, produces HTML report |

That's the current command surface. After setup, run `/stockwiz NVDA` (or any US ticker) and stockwiz will fetch 12 sources, dispatch four analysis agents in parallel (fundamental, sentiment, peer-comparison, risk), synthesize a disciplined bull/base/bear thesis, run an isolated adversarial stress test via the devils-advocate agent, reconcile the findings, and emit a self-contained HTML research brief into `~/.claude/stockwiz/sessions/<TICKER>-<timestamp>/report.html`. You can then ask follow-up questions in the same chat — stockwiz will answer from the durable session workspace without re-fetching.

## Roadmap

These commands are planned but not yet built. References to them elsewhere in the codebase are aspirational — they do not exist at `/stockwiz-*` yet:

- `/stockwiz-thesis <ticker>` — short-form bull/bear/base only, no HTML artifact
- `/stockwiz-compare <t1> <t2> [...]` — multi-ticker comparison report with a single comp table
- `/stockwiz-bear <ticker>` — standalone adversarial pass (today, adversarial runs as Stage 4 of `/stockwiz`)
- `/stockwiz-pivot <ticker>` — extract the implicit investment thesis and surface alternative expressions
- `/stockwiz-revisit <ticker>` — compare a prior session's thesis to current state, flag drift and triggered kill switches
- `/stockwiz-monitor <ticker>` — schedule weekly revisit via the scheduled-tasks MCP

Other roadmap items:

- **TradingView source** — requires headless Chrome for JS-rendered content
- **FRED macro-context skill** — adds rates / commodity / macro series from the St. Louis Fed API
- **SEC 10-K prose parsing** — currently we extract only XBRL structured facts, not the MD&A / Item 1A narrative
- ~~**Parallel analysis skills**~~ — **✓ delivered in 0.4.0**: the four analysis units were converted from Skills (loaded into main context, sequential) to agents (dispatched concurrently via Task in isolated contexts). Observed impact is deferred until a full run on 0.4.0 confirms the projected ~42% wall-clock speedup.

## Data sources

All sources are public and accessed via WebFetch. No paid APIs, no brokerage accounts. The plugin fetches politely (1.5s between calls) and handles per-source failure gracefully — an analysis continues even if some sources are blocked.

- SEC EDGAR (10-K, 10-Q, 8-K, proxy filings)
- Yahoo Finance (price, fundamentals, statistics)
- Google Finance (redundancy source for quotes)
- Simply Wall Street (snowflake scores, analyst target range)
- Zacks (rank, style scores, estimate revisions)
- Seeking Alpha (quant grades — public sections only)
- TradingView (technical rating, oscillator summary)
- Finviz (full quote page, ratios, technicals)
- Stockanalysis.com (5Y financials, estimates)
- Macrotrends (10Y historical financials)
- FRED (macro series — optional, needs free API key)
- Reddit (retail sentiment, r/stocks / r/wallstreetbets / r/investing)

## Scope

**Included in v1:** US equities only (ADRs OK — they file with the SEC). Long-horizon and swing analysis. HTML artifacts. File-backed session workspaces. Scheduled monitoring.

**Not included in v1:** Options, crypto, forex, futures, non-US listings. Intraday or real-time quotes. Portfolio construction or position sizing. Backtesting. XLSX/PDF/DOCX exports (HTML only). Brokerage integration.

## Compliance notice

> **Analytical tool, not investment advice.** stockwiz is a research assistant that synthesizes publicly available information. Nothing it produces is an offer, solicitation, or recommendation to buy, sell, or hold any security. Figures are drawn from public sources at the fetch time shown and may be stale, incomplete, or in error. Forward-looking statements are analytical framings, not predictions. You are responsible for your own investment decisions and should consult a licensed financial professional before acting on any information this tool generates.

Every HTML report embeds a disclaimer footer. Every report also passes a compliance language pass before being written to disk — banned imperative phrases ("buy", "sell", "recommend", "guaranteed") are rewritten to neutral analytical language. See `skills/report-generation/references/compliance-rules.md` for the full list.

## Known limitations

- Source pages occasionally change layout; the LLM-based extractor is resilient but not infallible. If a source fails repeatedly, `meta.json` will record it and the report will note "source unavailable".
- Free data providers rate-limit and sometimes Cloudflare-challenge unauthenticated requests. stockwiz handles this by failing fast on challenge pages and moving on.
- `/stockwiz-monitor` uses the scheduled-tasks MCP. Scheduled runs only fire while Claude Code is idle.
- Numbers are preserved verbatim from sources. When sources disagree, stockwiz records the disagreement rather than silently picking one.

## Author

**uid4oe** — [oggy@uid4oe.dev](mailto:oggy@uid4oe.dev)

Repo: [github.com/uid4oe/stockwiz](https://github.com/uid4oe/stockwiz)

## License

MIT. Copyright (c) 2026 uid4oe. See [`LICENSE`](LICENSE).
