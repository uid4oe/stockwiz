# Changelog

All notable changes to stockwiz are documented in this file.

## [0.1.0] — 2026-04-11 (in progress)

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
