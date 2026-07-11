# Changelog

All notable changes to stockwiz are documented in this file.

## [0.5.0] — 2026-07-12

### Stage 1 fetch sharding + report-writer efficiency — speed and token-usage release

A static analysis of the pipeline post-0.4.0 found the remaining costs concentrated in Stage 1 (the single sequential deep-researcher, ~20 of the ~45 total minutes) and in the report-writer's context habits (reading assets only to re-emit them; re-scanning the full HTML in-context for compliance). Projected net effect: **~45 → ~25-30 min wall-clock per deep-dive, per-run token usage down ~25-40%.**

**Stage 1 parallel fetch shards** (`commands/stockwiz.md` Step 6, `agents/deep-researcher.md`):

- The 12 sources are split into four shards with **disjoint origins** — A: SEC + Finviz + Barchart; B: Stockanalysis + Macrotrends; C: Google News/Finance + SWS + Seeking Alpha + Zacks; D: Yahoo + Reddit + X.com — dispatched as four concurrent `deep-researcher` instances in one message, mirroring the Stage 2 pattern. The 1500ms politeness delay is preserved *within* each shard; since each origin lives in exactly one shard, per-origin politeness is unchanged. Stage 1 wall-clock becomes `max(A..D)` ≈ 6-8 min instead of the ~20-min sum.
- `deep-researcher` is now a **parameterized shard agent**: the caller passes an explicit `SOURCES` list, and the agent reads only its own sources' reference files (~4× smaller context per instance). The duplicated "source-specific mechanics" section is gone — reference files are the single source of truth. Per-shard budgets (≤12 fetches; WebSearch A:0 / B:≤1 / C:≤2 / D:≤1) replace the global 35-fetch/3-WebSearch caps, which also resolves the 0.4.2 drift where SKILL.md said ≤4 WebSearch while deep-researcher said ≤3.
- **Orchestrator pre-flight** (new Step 4.5): prerequisites (curl/jq-or-python3/lynx) and the ticker→CIK lookup now run locally against the SEC tickers cache before any dispatch — `ticker-not-found-at-sec` fails fast without spending fetch budget.
- **meta.json now has exactly one writer: the orchestrator.** Shards return compact per-source statuses + key facts; the orchestrator merges them in one write (previously deep-researcher read-modify-wrote meta.json after every fetch, 12×/run) and composes `raw/_sanity.md` from the returned key facts (the cross-source sanity check spans shards). The report-writer's compliance log likewise travels via its return summary instead of a direct meta.json write. `meta.json.sources` entries no longer duplicate the fetched URL — it lives in the raw file frontmatter.

**report-writer efficiency** (`agents/report-writer.md`, `skills/report-generation/*`):

- **Asset splice:** `base-styles.css` (15KB) and `disclaimer.html` are no longer Read into context and re-emitted by hand. The agent composes with `/*@@BASE_STYLES@@*/` and `<!--@@DISCLAIMER@@-->` markers; a single Bash python3 splice injects both files byte-for-byte and substitutes `{version}`/`{timestamp}`/`{session-id}`. Kills ~8k duplicated tokens per run and the CSS transcription-error risk.
- **Grep-first compliance pass:** the agent composes to `report-draft.html`, locates candidate violations with a canonical grep (now specified in `compliance-rules.md` § Pass procedure), judges only the matching lines against the exemptions, applies targeted Edits, and promotes the clean draft via `mv`. Same 3-iteration abort semantics; a non-compliant or partial file can never sit at `report.html`. Replaces up to 3 full in-context re-scans of 60-150KB HTML.

**Prompt diet + drift fixes:**

- `commands/stockwiz.md` trimmed ~478 → ~330 lines; changelog-style narration ("v0.4.0 architectural note", "Why NOT to sequentialize", the long "What success looks like") moved to `docs/architecture.md`'s historical note or deleted. Agent files lose their "previously took N minutes" history.
- Dormant MODE→source-set tables (thesis/compare/revisit/bear) moved from `deep-researcher.md` to `docs/architecture.md` § Reduced fetch plans — the agent has no mode logic anymore; callers pass explicit source lists.
- Drift fixes: `source-extraction/SKILL.md`'s fictional "90-150 seconds wall-clock" envelope replaced with per-shard budgets; `report-generation/SKILL.md` § Scope no longer describes the retired walking skeleton; `architecture.md` agent count (3 → 8) and already-shipped "pending change" note; `session-workspace.md` stale "stages are sequential (not parallel)" claim and `commandVersion` example; `x-com.md` global-WebSearch-budget mentions.

## [0.4.2] — 2026-05-01

### Two new Stage 1 sources — Barchart insider directional + X.com whitelisted institutional commentary

Absorbed from a now-retired personal screener skill (`stock-leads`). Both feed into the existing `sentiment-synthesis` agent through the standard raw-file convention; no architectural change.

**New sources:**

- **Barchart insider trades** ([`skills/source-extraction/references/barchart.md`](skills/source-extraction/references/barchart.md)) — single-page WebFetch returning a 12-field JSON of buy/sell counts and shares across trailing 3-, 6-, and 12-month windows. Sentiment-synthesis derives `insider_signal_3m = (buy_shares_3m − sell_shares_3m) / (buy_shares_3m + sell_shares_3m + 1)` clamped to [-1, +1], skipping the signal for sectors {Real Estate, Financial Services} or option-exercise-dominated tickers (option grants are compensation noise, not directional). Fills a gap Finviz left open: Finviz only summarizes a single 3-month % delta, with no shape or longer windows. Best-effort; sentiment-synthesis falls back to the Finviz `insider_trans` summary alone on Barchart failure.

- **X.com institutional commentary** ([`skills/source-extraction/references/x-com.md`](skills/source-extraction/references/x-com.md)) — single `WebSearch site:x.com "{ticker}"` per ticker, gated by a strict 5-tier whitelist of ~30 verified handles (financial wires Tier 1, recognized journalists Tier 2, sell-side firms Tier 3, analyst-note aggregators Tier 4, issuer official accounts Tier 5). Up to 5 surviving snippets are written to `raw/x-com-commentary.md` with `{handle, tier, snippet ≤240 chars, url}`. Anonymous handles, WSB-orbit accounts, cashtag-spam accounts, and "guru" accounts are discarded without reading. Empty `quotes: []` is `status: ok`, not failure — many low-coverage tickers will return zero whitelist hits, and that's a normal outcome. Whitelist carries a `<!-- review-by 2026-11-01 -->` comment because handles deprecate.

**Sentiment-synthesis weighting rubric** (`agents/sentiment-synthesis.md`) — extended with Barchart at 0.85 (hard share-count signal across three windows) and X.com tier-derived weights (Tier 1 wires 0.85, Tier 2 journalists 0.7, Tier 3 sell-side 0.6, Tier 4 aggregators 0.5, Tier 5 issuer 0.4 sentiment-only). The Insider Activity output section now shows the 3/6/12-month buy/sell breakdown and the computed scalar (or an explicit "skipped" reason). A new Institutional Commentary (X.com) subsection lives under Recency-Weighted News. The "Unknowns" line about X/Twitter being unscraped is gone.

**Updated source counts** — "10 total" → "12 total" across `skills/source-extraction/SKILL.md`, `agents/deep-researcher.md`, `commands/stockwiz.md`. Per-source call budget: Barchart 1 WebFetch, X.com 1 WebSearch — totals now ~29-33 fetches + 3 WebSearch (was ~28-32 + 2).

**Why absorb instead of keep both:**

The retired `stock-leads` skill was a multi-ticker *screener* that surfaced 10–20 candidates near 52-week lows; `stockwiz` is a single-ticker *deep-dive*. They have different jobs but both pulled from public web sources, and stock-leads's two genuinely-novel enrichments (directional insider time series, whitelisted institutional X.com commentary) improve any single-ticker analysis. Folding them into stockwiz's existing sentiment pipeline adds value without doubling the plugin's surface area; the screening logic (Trefis seed discovery, proximity filter, quality gate, run-to-run diffs) was deliberately not migrated because it has no role in single-ticker work.

## [0.4.1] — 2026-04-11

### Language migration finish — completes the 0.4.0 skill→agent refactor

A self-review of 0.4.0 surfaced ~25 stale "analysis skill" / "thesis-discipline skill" / "your SKILL.md" phrases scattered across 11 files that the structural refactor missed. Three of them were inside Task prompts the orchestrator passes to subagents at runtime — meaning subagents would see incorrect descriptions of how their inputs were produced. The rest were documentation drift in commands, agents, docs, README, and reference files. None of it is structurally wrong (the actual agents, file paths, and dispatch patterns are correct in 0.4.0), but the prose lies about it everywhere.

This release is **language-migration only**. No behavior changes. No new features. No version of the structural refactor is undone.

**Runtime-visible Task prompts fixed** (orchestrator → subagent dispatch):
- `commands/stockwiz.md:317` — Stage 6 report-writer prompt: "from the four analysis skills" → "produced by the four analysis agents dispatched in parallel at Stage 2"
- `commands/stockwiz.md:321,331` — Two references to "Step 5 of your SKILL.md" / "Step 6.5 of your SKILL.md" → "your agent instructions". report-writer is `agents/report-writer.md`, not a SKILL.md.

**Orchestrator narrative fixed**:
- `commands/stockwiz.md:442,458` — Edge case descriptions for analysis failure paths now say "agent" not "skill"
- `commands/stockwiz.md:472` — "What success looks like" paragraph (the one that contradicted the just-committed Stage 2 rewrite) — now describes the parallel dispatch pattern, the per-stage timing, and the ~45-minute end-to-end target

**Agent body refs fixed**:
- `agents/report-writer.md:97,113` — Two "the thesis-discipline skill" references in the TL;DR atom curation steps → "the thesis-discipline agent"
- `agents/devils-advocate.md:29,44` — Two "thesis-discipline skill" references → "agent"
- `agents/deep-researcher.md:196` — Hard rule about not writing to analysis/ now correctly references "the four analysis agents" instead of "analysis skills"

**docs/architecture.md fixed (6 stale refs)**:
- Line 24 (the core principle statement) — "The analyst (four analysis skills + thesis-discipline + devils-advocate + report-writer)" → "The analyst (four analysis agents dispatched in parallel at Stage 2 — fundamental-analysis, sentiment-synthesis, peer-comparison, risk-screen — plus the thesis-discipline agent, the devils-advocate agent, and the report-writer agent)". This is the most important single fix in 0.4.1 because it's the textual statement of the architectural invariant.
- Line 31 — "Don't let analysis skills re-fetch" → "Don't let analysis agents re-fetch (enforced by their tool lists — no WebFetch/WebSearch)"
- Line 160 — failure mode explanation
- Line 233 — pre-filter compliance reference, with note explaining the section is named "skill authors" for historical reasons
- Lines 242, 245 — "How to add a new source" guidance corrected
- Historical note at line 202 (which describes v0.3.x correctly) intentionally preserved.

**docs/session-workspace.md fixed (8 stale refs)**:
- Directory tree annotations (lines 43-45) — "skill output" → "agent output" for all four analysis files
- File lifecycle table (lines 66, 68-71) — "Created by ... skill" → "agent" for all five
- Multi-writer rules section (lines 83, 87) — "owning skill" → "owning agent"

**README.md fixed**:
- Main description paragraph rewritten to mention "dispatch four analysis agents in parallel" instead of "run four analysis skills"
- Roadmap "Parallel analysis skills" item struck through and marked **✓ delivered in 0.4.0** with a note that the projected 42% speedup is pending end-to-end validation

**Reference doc refs fixed (4 stale refs)**:
- `compliance-rules.md:52` — abort error description now references the agents that would need to be fixed upstream
- `deep-dive-template.md:226` — "thesis-discipline skill" → "agent"
- `simply-wall-street.md:164` — note about SWS feeding bear-case inputs now correctly traces the data flow through sentiment-synthesis agent → thesis-discipline agent
- `macrotrends.md:194` — "fundamental-analysis skill will compute" → "agent will compute"

**Internal consistency fixes**:

- **Return-summary word limits aligned.** Two of the four parallel analysis agents (`peer-comparison`, `risk-screen`) said "Return ≤150 words" while the other two and the orchestrator's Stage 2 prompt said "≤200". All four now say ≤200, matching the orchestrator's expectation.

- **thesis-discipline MODE error handling added.** The agent now has an explicit "MODE dispatch and error handling" section that branches on the four documented modes (`full`, `reconcile`, `drift`, `implicit`) and returns a clear error string for unknown/missing/dormant modes rather than silently guessing. Hardening for a class of bug that was theoretical-only — the orchestrator always passes a valid MODE — but worth defending against.

**Token-cost honesty note** (added to the 0.4.0 entry retroactively):

The 0.4.0 CHANGELOG framed the refactor as a wall-clock win (~78 min → ~45 min, ~42% reduction) but did not discuss token cost. The full picture: running four agents in parallel means four isolated contexts each re-loading raw file content, rather than reading it once into the orchestrator's main context and sharing it across four sequential skill invocations. **Token cost is therefore similar or slightly higher than the v0.3.x sequential pattern, not lower.** The win is wall-clock latency, not API spend. For users running many analyses per week, the per-run cost is roughly unchanged; for users running a single analysis and waiting for it, the latency improvement is the entire benefit. This was an honesty gap in 0.4.0's release notes.

**Verification**: post-fix grep of `analysis skill[s]` and `thesis-discipline skill` across all non-CHANGELOG files returns only the legitimate `skills/source-extraction/SKILL.md` and `skills/report-generation/SKILL.md` references (those two are still skills — pure reference libraries) and the historical-note paragraph in `docs/architecture.md` line 202 that describes v0.3.x correctly.

`plugin.json` and `meta.json.commandVersion` bumped to **0.4.1**.

## [0.4.0] — 2026-04-11

### Parallelization refactor — skills → agents for the analysis layer

**Motivation:** timing data from five recorded sessions showed that deep-dive runs took 60–78 minutes of wall-clock time, and the single biggest-time artificial bottleneck was Stage 2 — the four analysis skills (`fundamental-analysis`, `sentiment-synthesis`, `peer-comparison`, `risk-screen`) running **sequentially in the main context**, totaling ~32 minutes on SNOW. On top of that, `thesis-discipline` was also a Skill running in the main context; its reconcile mode specifically was slowed by accumulated context bloat (8 min on SNOW vs 3 min on MBLY).

**Change:** converted all five work units from Skills to agents. The four analysis agents are now dispatched **concurrently** in a single message with four `Task` calls at Stage 2, running in parallel in isolated contexts. `thesis-discipline` became a single agent supporting `full` and `reconcile` modes via a MODE parameter, dispatched independently at Stage 3 and Stage 5.

**Measured impact** (projected from Phase 2.5 session data):

| Stage | v0.3.x (sequential) | v0.4.0 (parallel) | Savings |
|---|---|---|---|
| Stage 2 (four analyses) | ~32 min | ~10 min (max of four) | ~22 min |
| Stage 3 (thesis full) | ~12 min (main-context bloat) | ~4-6 min (isolated) | ~6 min |
| Stage 5 (reconcile) | ~8 min (main-context bloat) | ~2-3 min (isolated) | ~5 min |
| **Total wall-clock** | **~78 min** | **~45 min** | **~42%** |

**New agents** (`agents/`):

- `fundamental-analysis.md` — isolated-context analyst that reads SEC + Stockanalysis + Macrotrends + Finviz (+ optional Yahoo/SWS/Google Finance), writes `analysis/fundamental.md`
- `sentiment-synthesis.md` — isolated-context analyst that reads Google News + SWS + SA + Zacks + Finviz + SEC 8-K + Reddit, applies the weighting rubric and publisher-distribution signal, writes `analysis/sentiment.md`
- `peer-comparison.md` — isolated-context analyst that reads SWS competitor snowflakes, writes `analysis/peer-comp.md`
- `risk-screen.md` — isolated-context analyst that reads Finviz + Stockanalysis + Macrotrends + SEC + SWS + SA + Google News, writes `analysis/risk.md`
- `thesis-discipline.md` — isolated-context synthesizer supporting `full` / `reconcile` / `drift` (dormant) / `implicit` (dormant) modes via a MODE parameter; writes or appends to `thesis.md`

Each analysis agent has `tools: Read, Write, Grep, Bash` (no WebFetch/WebSearch — they work from cached raw data only). Each pins `model: sonnet`. Each agent's body includes a "you run in isolation, read only specific raw files, write exactly one file, never touch meta.json" section.

**Deleted skills** (`skills/`):

- `skills/fundamental-analysis/` (directory removed)
- `skills/sentiment-synthesis/`
- `skills/peer-comparison/`
- `skills/risk-screen/`
- `skills/thesis-discipline/`

**Only two skills remain**, both pure reference libraries:

- `skills/source-extraction/` — per-source extraction specs consulted by deep-researcher
- `skills/report-generation/` — HTML template + compliance rules + assets consulted by report-writer

**Orchestrator changes** (`commands/stockwiz.md`):

- **Stage 2 rewritten** for parallel Task dispatch. A single message issues four `Task` calls concurrently. Each Task prompt passes `SESSION_DIR`, `TICKER`, `HORIZON`. The orchestrator waits for all four to return, then appends stage entries with `parallel: true` to meta.json.
- **Stage 3 rewritten** to dispatch `thesis-discipline` (MODE=full) via `Task`. Same pattern as devils-advocate — isolated context, compact return summary.
- **Stage 5 rewritten** to dispatch `thesis-discipline` (MODE=reconcile) via `Task`. Reads only thesis.md + analysis/devils-advocate.md. Append-only to thesis.md.
- **Step 5 todo list updated** to reflect "IN PARALLEL" for Stage 2 and "agent" labels for each stage.
- **Architectural note added** — no skills are loaded into the orchestrator's main context during a deep-dive. Every work unit is an agent.
- **Race condition notes added** — the four parallel agents write to four independent files and do NOT touch meta.json. The orchestrator is the single writer of meta.json. No shared mutable state.

**Yahoo Finance and Zacks demoted to "best-effort"**

Both have failed in 100% of recorded sessions (Yahoo → rate-limited, Zacks → Cloudflare/Imperva). They remain in the fetch plan for cross-check value when they happen to succeed, but:

- Their status badges in source-extraction/SKILL.md and deep-researcher.md now read "best-effort"
- Their reference files explicitly acknowledge the ~0% observed success rate
- deep-researcher's source-specific mechanics section notes "accept them when they work, move on silently when they don't, do NOT retry"

This doesn't save wall-clock time directly (they still fail fast) but it sets accurate user expectations.

**Architecture doc updated** (`docs/architecture.md`):

- State-flow diagram rewritten to show the Stage 2 parallel dispatch block
- "Current assignments" table updated: 8 agents (was 3), 2 skills (was 8)
- Historical note added explaining the v0.4.0 refactor and the measured savings
- "How to add a new analysis agent" section rewritten (was "add a new analysis skill")

**What's intentionally NOT changed in this release**

- deep-researcher remains a single agent (its internal source fetches are inherently sequential due to 1500ms politeness delays; parallelizing across hosts adds complexity without much wall-clock savings since each source's latency is dominated by LLM extraction, not network)
- devils-advocate unchanged (already an agent)
- report-writer unchanged (already an agent)
- compliance-rules.md and the pre-filter guidance unchanged
- No visible changes in the user-facing UX (same command, same output format, same session workspace layout)

**Version metadata:** `plugin.json` and `meta.json.commandVersion` bumped to **0.4.0**.

## [0.3.3] — 2026-04-11

### Fix invalid plugin manifest schema

0.3.2 shipped an invalid `plugin.json` that Claude Code refused to load with `"Plugin stockwiz has an invalid manifest file"`. Root cause: when adding author metadata I included npm-style fields (`homepage`, `repository`, and a `url` inside `author`) that are not part of the Claude Code plugin manifest schema. The accepted schema is strict: `name`, `version`, `description`, `author: {name, email}` — nothing else.

**Fix.** Reverted `plugin.json` to the minimal valid shape. Author name and email are preserved. The repo URL moved into the `description` field as plain text so it's still surfaced in plugin listings.

Verified against five reference plugin.json files under `~/.claude/plugins/` (frontend-design, claude-opus-4-5-migration, feature-dev, learning-output-style, code-review) — all use the same four-field shape.

`commandVersion` bumped to 0.3.3 in the orchestrator's meta.json skeleton.

## [0.3.2] — 2026-04-11

### Architectural cleanup from self-review — no new features

A comprehensive critical review of 0.3.1 surfaced plan-vs-reality drift (6 commands promised but not built), schema ownership gaps, duplicated compliance pre-filter lists, empty scaffolding directories, and widespread "Phase N" build-history language leaking into runtime docs. This release fixes all 15+ items from the review, with zero new features. The goal is to raise the quality baseline of everything that already exists before any more features land.

**Plan-vs-reality cleanup — credibility restored**

- **README commands table rewritten** to list only `/stockwiz-setup` and `/stockwiz` (the two commands that actually exist). Everything else moved to a new Roadmap section marked as "planned but not yet built".
- **stockwiz-setup quickstart rewritten** to show only the two live commands. Previously it directed users to run `/stockwiz-compare`, `/stockwiz-bear`, `/stockwiz-monitor` as first-run examples — all of which would have produced command-not-found errors.
- **Agent description fields no longer lie.** `deep-researcher` previously claimed to be "Invoked by /stockwiz, /stockwiz-thesis, /stockwiz-compare, /stockwiz-bear, and /stockwiz-revisit" — four of those five don't exist. Updated to "Currently invoked only by /stockwiz; supports reduced fetch plans reserved for future commands." Same fix applied to `devils-advocate`.
- **Skill "Loaded by" sections** in fundamental-analysis, peer-comparison, and thesis-discipline previously referenced unbuilt commands as active callers. Corrected to reflect current reality with a note about roadmap commands.
- **Orchestrator edge cases and final message** cleaned of references to `/stockwiz-bear`, `/stockwiz-revisit`, `/stockwiz-thesis`, `/stockwiz-compare`. Replaced with "re-run `/stockwiz <TICKER>`" where the suggestion was to retry, or with "a future /stockwiz-\* command on the roadmap" where the suggestion was genuinely aspirational.

**Empty scaffolding deleted**

Six empty reference directories and the entire empty `macro-context/` skill tree removed. Created in Phase 1 scaffolding for files planned in later phases, never populated:

- `skills/macro-context/` (and its empty `references/` subdir) — was a ghost skill with no SKILL.md
- `skills/fundamental-analysis/references/`
- `skills/peer-comparison/references/`
- `skills/sentiment-synthesis/references/`
- `skills/risk-screen/references/`
- `skills/thesis-discipline/references/`

`docs/` is no longer empty — it now contains two substantive architecture documents (below).

**New documentation: `docs/`**

- **`docs/architecture.md`** — the entry point for any contributor. Documents the librarian/analyst separation principle as the core architectural invariant, a 7-stage ASCII state-flow diagram of the `/stockwiz` pipeline with explicit fatal-vs-non-fatal error paths, the agent-vs-skill decision rule (context pollution risk / anchoring hazard / composition isolation → agent; otherwise skill), the current assignment of each work unit with rationale and open questions, the file layout contract (summary, with full detail in session-workspace.md), the single-entry-point compliance pass design, and step-by-step "how to add a new source / analysis skill / command" guides.
- **`docs/session-workspace.md`** — the authoritative filesystem contract. Complete directory tree with file purposes, file lifecycle table (created by / read by / mutable? / persistent?), multi-writer rules (immutable raw/, append-only thesis.md, read-modify-write meta.json), the **full meta.json schema** with field-level documentation (previously defined only in commands/stockwiz.md Step 4 as a JSON snippet), the reason-code taxonomy for `sources[slug].reason`, config.json schema, cache schemas (`company_tickers.json`, `macrotrends-slugs.json`), retention policy, and the checklist for "how to add a new source to the filesystem contract".

Together these two files close the biggest contract gap identified in the review: meta.json previously had 6 writers and 1 schema definition inside a JSON snippet; it now has a proper documented contract.

**Pre-filter compliance rules consolidated**

Previously the banned-phrase pre-filter guidance was duplicated across `fundamental-analysis`, `sentiment-synthesis`, `peer-comparison`, `risk-screen`, and `thesis-discipline` hard-rules sections. Adding a new banned phrase required updating five files.

- New **"Pre-filter guidance for skill authors"** section added to `compliance-rules.md` as the single source of truth. Covers: the principle, phrases to avoid, neutral alternatives, how to quote source text with `<q>` tags, the industry-terminology exemption list, why pre-filtering matters, and a one-line reference template for skills to link instead of duplicating.
- **Each analysis skill's Hard Rules** now has a single line pointing at `compliance-rules.md § Pre-filter guidance for skill authors` instead of the duplicated list. Skill-specific additions (e.g. fundamental-analysis's "undervalued/overvalued", risk-screen's "risky/safe", peer-comparison's "cheap/expensive") are kept inline but as brief callouts rather than full lists.

**Phase-label language purged from runtime docs**

Previously runtime files were littered with "Phase 1.5 limitation", "Phase 2+ feature", "Phase 2.5 source", "(Phase 3)" qualifiers. Phase numbering is internal build history and belongs in CHANGELOG, not in runtime instructions. A contributor reading `sentiment-synthesis/SKILL.md` shouldn't need to know which phase added the Reddit source.

- **~160 phase-label substitutions** across commands, agents, skills, and source reference files, replacing "Phase X" qualifiers with either nothing, "current", or "(roadmap)" as appropriate
- Grammar cleanup pass on the substitution aftermath — several sentences that lost their phase qualifier became ungrammatical; all repaired
- "Status: Phase N" banners in reference files → "Status: active"
- "In Phase 2+ " / "For Phase 1.5 " prefixes → removed
- Step header " (unchanged from Phase 2)" qualifiers → removed
- `/stockwiz-setup`'s "Phase 3 commands coming soon" → proper Roadmap language

**Step 13 fatal-error handler expanded**

Previously Step 13's fatal-case list named four conditions. Earlier steps actually jumped to Step 13 from more than four places (ticker-not-found, mkdir failure, SEC pre-flight failure, <3 sources succeeded, curl missing, JSON parser missing, etc). The list is now **exhaustive**: 12 numbered fatal conditions with reason codes, a clear distinction from the non-fatal failure list, a procedural handling block that sets `failedAtStage` in meta.json, and a final "non-fatal failures" section for contrast.

**Version metadata**

`plugin.json` and session `meta.json.commandVersion` bumped to **0.3.2**.

**No new features. No new sources. No new commands. No new skills.**

This is a cleanup-only release. The three runtime-critical Phase 2.5 fix items from 0.3.1 (orchestrator prompts, analysis-skill inputs, etc) remain in place; nothing was regressed. The next feature work (Phase 3 — actually building the missing commands, or sub-agent-ifying the four analysis skills for context isolation) can proceed against a clean baseline.

## [0.3.1] — 2026-04-11

### Phase 2.5 fix pass — critical bugs found in self-review

A critical review of 0.3.0 surfaced three bugs that would have broken Phase 2.5 at runtime, plus several quality issues worth fixing before a test run. All items from the self-review are addressed.

**Critical bugs fixed**

- **Orchestrator's deep-researcher Task prompt was still saying "Phase 1.5, 6 sources"** (commands/stockwiz.md line ~104). The commandVersion had been bumped but the prompt text was never updated. Would have caused the deep-researcher to fetch only the original 6 Phase 1.5 sources — the four new Phase 2.5 references (Google News RSS, Macrotrends, Zacks, Reddit) would never have been consulted. **Fixed**: prompt now explicitly enumerates all 10 Phase 2.5 sources with their tools and ordering, plus the Phase 2.5 slug conventions.

- **Four analysis skills had zero references to the new Phase 2.5 sources.** fundamental-analysis, sentiment-synthesis, peer-comparison, and risk-screen were all written for the 6 Phase 1.5 sources and nothing knew to consume the new raw/ files. Even if deep-researcher fetched them, the analysis layer would have ignored them. **Fixed**:
  - `sentiment-synthesis` now reads `raw/google-finance.md` (Google News RSS) as the primary news layer, `raw/reddit.md` for retail contrarian signal at weight 0.2 with an explicit anti-confirmation rule at extremes, and `raw/zacks-snapshot.md` for Zacks Rank + Style Scores. Updated weighting rubric with wire-service vs opinion-publisher split. Added a Publisher Distribution section and an explicit cross-source agreement check in Analyst Positioning.
  - `fundamental-analysis` now reads `raw/macrotrends.md` for 10-20Y cycle context, with an explicit preference for Macrotrends over Stockanalysis in the Growth section. Added cycle-framing instruction: compare current growth to long-run mean (e.g. "2.9σ above mean → cyclical peak not structural new normal").
  - `risk-screen` now reads `raw/macrotrends.md` for long-run drawdown profile. New subsection: worst historical annual FCF and net income declines over the 10+Y window, plus a cycle characterization (secular growth / cyclical / turnaround).
  - `peer-comparison` unchanged — SWS competitor snowflakes remain the Phase 2.5 peer source.

- **Orchestrator's report-writer Task prompt still said "Phase 2, 6 sources"** and didn't mention v0.3 insights-first template or the TL;DR atom curation. **Fixed**: prompt now explicitly requires running Step 5 (curate TL;DR atoms) before composition, enumerates the v0.3 section ordering, specifies the `<details>` abstract formulas, and lists the Phase 2.5 compliance exemptions including new source labels (Zacks Rank, Reddit titles, Google News headlines).

**Quality issues fixed**

- **Duplicated `@media print` block** in base-styles.css (lines 658 and 682). Merged into a single block; the "belt and suspenders" redundant rules are now inside the main print media query.

- **Zero `:focus-visible` styles** for keyboard accessibility. Added focus rings on `details > summary`, `a`, and `button` using the existing `--accent` token. Standard 2px outline with 3px offset for summary elements.

- **Unknowns ordering in thesis-discipline was assumed but not enforced.** deep-dive-template.md said "the list is already prioritized by materiality (most material first)" but thesis-discipline's Unknowns section just said "bulleted list". **Fixed**: thesis-discipline now mandates ordering by materiality with four explicit ranking criteria (impact on dominant case / impact on assumption ledger / time horizon to resolution / cross-source asymmetry). The first item in the list must be the one with the largest downstream impact — this is what the TL;DR "Biggest unknown" callout picks.

- **Key insight selection documented as non-deterministic by design.** Added an explicit note at the top of the "How to pick the key insight" section in deep-dive-template.md acknowledging that two runs may pick different insights when multiple heuristics apply, explaining why this is a deliberate trade-off (LLM context-aware curation vs deterministic magnitude-picking), and noting that every other stockwiz output remains reproducible — the insight is the only exception.

- **Compact case headlines now have a character budget.** thesis-discipline's Bull/Base/Bear case structure now includes an optional `**Compact headline.**` field (≤100 characters) that should be produced when the main Headline exceeds 140 characters. The report-writer's TL;DR compact cards use the Compact form when present, the main Headline otherwise.

**Papercuts fixed**

- **Reddit JSON-body 429 handling.** Reddit sometimes returns HTTP 200 with body `{"error": 429}` or `{"message": "Too Many Requests"}` instead of an HTTP 429 status. reddit.md now documents parsing both the HTTP status AND the JSON body before treating a response as usable, with example Python parsing code and the same `rate-limited` reason code for both cases.

- **Google News RSS with dotted tickers** (BRK.B, BF.B, JW.A). google-finance.md now documents two strategies: (1) preferred, use the company name from SEC EDGAR instead of the ticker; (2) fallback, strip punctuation and accept class-undifferentiated news. Either way, the raw file frontmatter notes which strategy was used.

- **Macrotrends slug cache is now self-populating** in Phase 2.5, not "Phase 3+". deep-researcher and macrotrends.md both document the read-modify-write pattern at `~/.claude/stockwiz/cache/macrotrends-slugs.json`. First run on a ticker pays a WebSearch; subsequent runs read from cache and skip WebSearch entirely.

### Non-issues confirmed during review

The following were checked and found to be OK — worth recording so they don't get re-flagged:

- **CSS/template class consistency.** All 45 classes referenced in the v0.3 template are defined in base-styles.css; zero missing. A handful of CSS classes (`good`, `watch`, `mono`, `num`, `strong`) are utility classes kept intentionally.
- **Above-the-fold vertical budget.** Rough computation: ~752px of content, comfortably under a typical 900px laptop viewport. The TL;DR fits in one screen on standard displays.
- **Step numbering.** Both `commands/stockwiz.md` (1–13) and `agents/report-writer.md` (1–11) have consistent step numbering with no gaps after Phase 2.5.
- **Self-contained CSS.** Zero `url(https://...)` references — confirmed no network dependencies.

## [0.3.0] — 2026-04-11

### Phase 2.5 — Insights-first report + four new sources

Two substantial changes landing together: a ground-up redesign of the HTML report toward "concise insights first, details on demand", and expansion of the source set from 6 to 10 to close real gaps (primarily the missing news layer and long-run historical context).

**Report redesign — insights-first with progressive disclosure**

The previous template was a long-form research brief where the reader scrolled through ~60-120KB of HTML in linear order. Field use showed that readers want a 20-second scan first and details on demand.

New template (v0.3):

1. **Above the fold** (always visible, targets ~1 viewport height on a laptop):
   - **Hero** — ticker, company, price, as-of, sector/industry. Minimal, no tagline, no sparkline.
   - **Key metrics strip** — 6 metrics in one row with tabular-numeric values (market cap, P/E T, P/E F, FCF yield, revenue growth, ROIC)
   - **The TL;DR panel** — compact three cases (one-sentence headlines in hash-shuffled order) + three curated callouts:
     - **Key insight** — ONE most striking observation picked from the analyses via an explicit heuristic (anomalous quality metric, step-change growth, capital-structure surprise, margin inflection, historical outlier, SWS extreme). Curated by the report-writer, not just extracted.
     - **Closest kill switch** — the kill switch with the smallest margin of safety, with a compact stats bar showing current / trigger / margin to go
     - **Biggest unknown** — the most material item from thesis.md Unknowns as one sentence

2. **Below the fold** (native HTML `<details>`/`<summary>`, no JavaScript, print-expands-all):
   - Three Cases (full) — `open` by default
   - Kill Switches (all, sorted by margin ascending)
   - Fundamentals, Sentiment, Peers, Risk, Assumption Ledger, Unknowns, Sources, Adversarial Pass — all collapsed by default
   - Each `<summary>` shows the section title plus a one-line abstract so readers can decide what to expand

3. **Always visible at bottom**: disclaimer footer (never collapsible, compliance-exempt).

The new `base-styles.css` is a ground-up rewrite: minimal sans-serif typography, tabular-numeric monospace for metrics, single warm-blue accent, generous whitespace, no decorative flourishes, print-friendly rules that force-expand all `<details>` when printing.

The `report-writer` agent gains a new **Step 5 — Curate TL;DR atoms** that runs *before* HTML composition. It explicitly picks the key insight, closest kill switch, and biggest unknown using documented heuristics. This is the most important new behavior — the TL;DR is curated, not just extracted.

**Four new sources bringing Phase 2.5 to 10 total**

- **Google Finance + Google News RSS** — the news layer stockwiz has been missing. Google News RSS (`news.google.com/rss/search?q=...`) is keyless, returns well-formed XML, aggregates Bloomberg/Reuters/WSJ/CNBC/Seeking Alpha/many others with publication timestamps. Captures up to 20 recent headlines with publishers + pubDates + snippets. Computes publisher distribution (wire-heavy vs opinion-heavy) as a shape signal. Google Finance quote page is a secondary cross-check (often SSR-empty but occasionally useful).
- **Macrotrends** — 10-20 years of annual historical data for revenue, net income, gross profit/margin, and free cash flow/margin. Complements Stockanalysis's 5Y with deep cycle context. WebSearch resolves the company slug, then 4 curl calls (one per metric page) piped through lynx. Critical for "is this a cyclical peak?" questions.
- **Zacks** — Zacks Rank and Style Scores (VGM) on a best-effort basis. Aggressive Cloudflare-challenge detection: if the response contains challenge page markers, mark failed and move on with zero retries. When it works, provides unique proprietary scoring; when it doesn't, no material loss.
- **Reddit** — r/stocks, r/wallstreetbets, r/investing via keyless `.json` endpoints. Captures top posts' titles / scores / comment counts / dates / permalinks — **never** usernames, never post bodies. Aggregates per-sub post count, average score, activity level (quiet/normal/elevated/heavy). Explicit contrarian-signal weighting (0.2) in sentiment synthesis — heavy retail bullishness historically correlates with near-term consolidation more than continued uptrends.

**Source pipeline updates**

- `deep-researcher` fetch plan expanded from 6 to 10 sources, with a new ordering that front-loads reliable structured sources (SEC → Finviz → Stockanalysis → Macrotrends → Yahoo) before news/narrative/best-effort sources (Google News → SWS → SA → Zacks → Reddit)
- `source-extraction/SKILL.md` source index updated with Phase 2.5 column; 10 sources marked as wired, TradingView and FRED remain deferred
- Fetch budget widened from 25 to **35 total calls + 3 WebSearch** to accommodate Macrotrends' 4-page fetch and the happy path's ~28-32 calls
- New fallback chains documented: news/catalysts (Google News → SA teasers → SEC 8-K), retail sentiment (Reddit r/stocks + r/wallstreetbets + r/investing), long-run cycle context (Macrotrends 10-20Y → Stockanalysis 5Y)
- Reduced modes (thesis, compare, revisit, bear) updated to pick appropriate source subsets

**Report-writer updates for insights-first**

- New `Step 5 — Curate TL;DR atoms` runs before composition; picks key insight / closest kill switch / biggest unknown
- Composition order follows the new v0.3 template section list (hero → metrics → TL;DR → details sections → disclaimer)
- New `Step 6.5 — Compute summary abstracts` produces per-section one-line abstracts so collapsed sections are still scannable
- Step numbering renumbered (old Step 6/7/8/9 became 8/9/10/11)

**Known Phase 2.5 limitations (deferred to Phase 3+)**
- TradingView still requires headless Chrome; deferred
- Peer fetching still relies on SWS competitor snowflakes (2-3 peers typically). A proper peer-fetch capability (Finviz-based per-peer quick fetches) is Phase 3.
- Analysis skills still run sequentially in the main context; parallelization would require wrapping each in a Task subagent
- SEC 10-K prose (business description, Item 1A, Item 7 MD&A) still not parsed — only XBRL facts

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
