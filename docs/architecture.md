# stockwiz architecture

This document is the entry point for anyone trying to understand how stockwiz works internally. If you are fixing a bug, adding a source, or wondering why a skill is a skill instead of an agent, read this first.

## What stockwiz is (and isn't)

stockwiz is a **Claude Code plugin** whose entire "code" is markdown instructions to an LLM. There is no runnable program outside a Claude Code session. Everything in this repo is either:

- **Runtime instructions** read by Claude when a command is invoked (`commands/`, `agents/`, `skills/`)
- **Source-extraction specs** that document how to fetch and parse public web data (`skills/source-extraction/references/`)
- **Assets** inlined into generated HTML reports (`skills/report-generation/assets/`)
- **Meta-documentation** about the architecture and conventions (`docs/`, `README.md`, `CHANGELOG.md`)

There is no Python, no JavaScript, no tests, no build step, no CI. Running stockwiz means invoking `/stockwiz <TICKER>` inside a Claude Code session; Claude reads the orchestrator markdown, follows the instructions, delegates to subagents via the `Task` tool, loads skills via the `Skill` tool, and writes files to the user's session workspace.

## Core architectural principle: librarian / analyst separation

**This is the one design decision that makes everything else coherent.** Preserve it.

stockwiz splits every operation into two roles:

- **The librarian** (`deep-researcher` agent, source-extraction skill + reference files) captures data from public sources **verbatim, without interpretation**. It produces `raw/*.md` files that are the audit trail. Raw files are immutable once written — no downstream stage ever edits them. If numbers change, it's because a newer run fetched fresh data into a new session directory.

- **The analyst** (four analysis agents dispatched in parallel at Stage 2 — `fundamental-analysis`, `sentiment-synthesis`, `peer-comparison`, `risk-screen` — plus the `thesis-discipline` agent, the `devils-advocate` agent, and the `report-writer` agent) reads the librarian's output and interprets it. Every analyst component writes its own file under `analysis/` (or `thesis.md`, `report.html`). The analyst may disagree with the raw data, frame it differently, or flag it — but the analyst never rewrites it.

The benefit: when a downstream output is questioned, you can always grep `raw/` to see what the source actually said. The audit trail is immutable and the interpretation layer is reversible.

Violations to watch for:

- Don't let deep-researcher editorialize ("this looks like a strong company"). It's a librarian.
- Don't let analysis agents re-fetch from the web. They read from `raw/` only (enforced by their tool lists — no WebFetch/WebSearch).
- Don't let thesis-discipline modify raw data, only analysis outputs.
- Don't let the report-writer invent numbers; if it can't find a citation in `raw/` or `analysis/`, the number doesn't appear in the report.

## State-flow diagram: `/stockwiz <TICKER>` pipeline (v0.5.0)

**All work units are subagents.** Every non-trivial work unit is a subagent dispatched via Task, running in an isolated context. Stage 1 dispatches four fetch shards in parallel; Stage 2 dispatches the four analysis agents in parallel; all other stages dispatch one agent sequentially because of data dependencies. The orchestrator is the **sole writer of meta.json** — subagents return compact summaries and the orchestrator merges them.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         /stockwiz <TICKER>                           │
│                       commands/stockwiz.md                           │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Steps 1-5: pre-flight (all in the orchestrator's own context)       │
│  ├─ parse args (TICKER, --horizon)                                   │
│  ├─ check config.json exists (else exit: need /stockwiz-setup)       │
│  ├─ create SESSION_DIR/raw/, SESSION_DIR/analysis/                   │
│  ├─ write initial meta.json (schemaVersion=1, status=started)        │
│  ├─ check prerequisites (curl fatal, jq|python3 fatal, lynx noted)   │
│  ├─ resolve TICKER→CIK locally from the SEC tickers cache            │
│  │   └─ no match → FATAL ticker-not-found-at-sec, nothing dispatched │
│  └─ TodoWrite the 7 pipeline stages                                  │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STAGE 1 — FOUR FETCH SHARDS DISPATCHED IN PARALLEL                  │
│  (one message, 4 Task calls, all `deep-researcher` instances)        │
│                                                                      │
│  ┌─ A: SEC EDGAR → Finviz → Barchart ──────────── (fatal gate) ─┐    │
│  ├─ B: Stockanalysis → Macrotrends ──────────────────────────────┤   │
│  ├─ C: Google News/Finance → SWS → Seeking Alpha → Zacks ────────┤   │
│  └─ D: Yahoo → Reddit → X.com ───────────────────────────────────┘   │
│                                                                      │
│  Shards hit disjoint origins (politeness preserved), write disjoint  │
│  raw/*.md files, never touch meta.json. Each returns ≤200-word       │
│  per-source statuses + key facts. Wall-clock ≈ max(A..D), not sum.   │
│  Orchestrator merges statuses into meta.json.sources (one write),    │
│  runs the cross-source sanity check from the returned key facts.     │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                     ┌───────────┴──────────┐
                     │                      │
        SEC ok, ≥3 sources ok?       SEC failed or <3 ok?
                     │                      │
                     ▼                      ▼
┌─────────────────────────────┐  ┌───────────────────────────────┐
│  STAGE 2 — FOUR AGENTS      │  │  Step 13: FATAL ERROR         │
│  DISPATCHED IN PARALLEL     │  │  meta.json.status = failed    │
│  (one message, 4 Task calls)│  │  No thesis, no report         │
│                             │  └───────────────────────────────┘
│  ┌─ fundamental-analysis ─┐ │
│  ├─ sentiment-synthesis ──┤ │  Each runs in isolated context.
│  ├─ peer-comparison ──────┤ │  Each reads specific raw/* files
│  └─ risk-screen ──────────┘ │  and writes exactly one analysis
│                             │  file. None touch meta.json.
│  Orchestrator waits for all │  Wall-clock ≈ max(t1, t2, t3, t4)
│  four returns, then appends │  instead of sum.
│  4 stage entries to         │
│  meta.json.stages.          │
└────────────┬────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 3 — thesis-discipline (Task, isolated, MODE=full)     │
│  Reads analysis/* files (falls back to raw/ if missing).     │
│  Writes thesis.md with mandatory sections:                   │
│    ## Bull Case    ## Base Case    ## Bear Case              │
│    ## Explicit Disconfirmers                                 │
│    ## Kill Switches                                          │
│    ## Unknowns (ordered by materiality)                      │
│  Every case has Headline + optional Compact headline ≤100ch  │
│  Every claim cited. Kill switches measurable.                │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 4 — devils-advocate (Task, isolated)                  │
│  Input: ONLY thesis.md (no raw/ access — anchoring proof)    │
│  Output: analysis/devils-advocate.md                         │
│    ## Weakest Claims (ranked by fragility)                   │
│    ## Opposing Narrative                                     │
│    ## Missing Counter-Evidence                               │
│    ## Alternative Interpretations                            │
│    ## Kill Switch Adequacy                                   │
│  Up to 3 fresh WebFetch/WebSearch for disconfirming data.    │
│  NON-FATAL on failure (report gets a placeholder).           │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 5 — thesis-discipline (Task, isolated, MODE=reconcile)│
│  Reads thesis.md + analysis/devils-advocate.md               │
│  APPENDS ## Adjustments After Stress Test to thesis.md       │
│  NEVER rewrites original bull/base/bear claims — append only │
│  Skipped if Stage 4 failed.                                  │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 6 — report-writer (Task, isolated)                    │
│  Step 5: curate TL;DR atoms (key insight / closest kill      │
│          switch / biggest unknown)                           │
│  Step 6: compose insights-first HTML with @@ asset markers   │
│    hero → metrics → TL;DR → <details> sections → footer     │
│  Step 8: write report-draft.html, Bash-splice CSS+disclaimer │
│  Step 9: grep-first compliance pass (≤3 iters, abort→delete) │
│  Step 10: promote draft → report.html, verify (size, grep)   │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 7 — finalize                                          │
│  Update meta.json: status=complete, reportPath, times        │
│  Print workspace pointer + TL;DR summary to user             │
│  Leave session hot for conversational follow-ups             │
└──────────────────────────────────────────────────────────────┘
```

**Wall-clock impact of the parallel refactors** (v0.4.0 measured from Phase 2.5 session data; v0.5.0 estimated):

| Stage | v0.3.x (sequential) | v0.4.0 | v0.5.0 |
|---|---|---|---|
| Stage 1 (fetch) | ~20 min (12 sources serial) | ~20 min | ~6-8 min (max of 4 shards) |
| Stage 2 total | ~32 min (600+540+300+480s) | ~10 min (max of the four) | ~10 min |
| Stage 3 (thesis full) | ~12 min (main context bloat) | ~4-6 min (isolated) | ~4-6 min |
| Stage 5 (reconcile) | ~8 min | ~2-3 min | ~2-3 min |
| All other stages | ~6 min | ~6 min | ~6 min |
| **TOTAL** | **~78 min** | **~45 min** | **~25-30 min** |

v0.4.0 cut ~42% by parallelizing Stage 2 and isolating Stages 3/5; v0.5.0 cuts another ~35% by sharding Stage 1 across four concurrent fetch agents with disjoint origins.

### Fatal vs non-fatal failures

Only the **SEC EDGAR path** is fatal in the normal pipeline. The reasoning: SEC filings are the ground truth for US equities; without them, every downstream claim is ungrounded. The common case — ticker not found — is caught by the orchestrator's pre-flight CIK lookup before any shard is dispatched; the rare case — SEC's APIs down — surfaces via shard A's return. Everything else is non-fatal:

- **Any individual non-SEC source can fail** (Zacks Cloudflare, Yahoo 429, Reddit JSON-body 429, etc.) — the owning fetch shard logs the failure and continues with its remaining sources. The analysis layer gracefully degrades and flags the gap in the Unknowns section.
- **Fewer than 3 total sources succeeding** is fatal — not enough data to build a credible thesis.
- **An individual analysis agent failing** is non-fatal — the agent writes a stub (or no file), the thesis-discipline agent falls back to reading raw files directly, and the report-writer agent renders a thin section with a placeholder note.
- **devils-advocate failing** is non-fatal — Stage 5 is skipped, Stage 6 renders the adversarial appendix as a placeholder.
- **report-writer compliance pass failing after 3 iterations** is fatal — we refuse to emit a non-compliant report rather than silently pass.

These are all enforced in `commands/stockwiz.md` Steps 4.5, 6, 11, and 13. The complete fatal-error list lives in Step 13.

## Agent vs skill: the decision rule

stockwiz has two ways to package a work unit:

- **Skill** (in `skills/<name>/SKILL.md`) — loaded via the `Skill` tool, runs **in the main context**. The caller's context accumulates whatever the skill reads.
- **Agent** (in `agents/<name>.md`) — delegated via the `Task` tool, runs **in an isolated context**. The caller only sees the agent's compact return summary.

**Decision rule:** use an **agent** if any of these apply:

1. **Context pollution risk** — the work unit reads large files (multi-kilobyte raw data, API responses, HTML dumps) that would bloat the caller's context. `deep-researcher` reads 10+ source files, sometimes hundreds of KB each → agent.
2. **Anchoring hazard** — the work unit must produce a judgment that isn't biased by upstream framing. `devils-advocate` must attack the thesis; if it runs in the main context, it sees all the bull-framed raw data and becomes sycophantic → agent.
3. **Composition isolation** — the work unit produces a large output (HTML, long document) that's better written once and handed back than accumulated line-by-line in the caller. `report-writer` generates 60-150KB of HTML → agent.

**Use a skill** when:

1. The work unit is short-context (reads 1-2 summary files, produces a short output)
2. The caller genuinely wants the skill's content available for subsequent work
3. The work unit doesn't need to protect itself from anchoring bias

### Current assignments (v0.4.0)

| Component | Type | Primary reason |
|---|---|---|
| `deep-researcher` | agent | Context pollution — reads raw HTML, JSON, lynx dumps. Dispatched ×4 in parallel at Stage 1, one instance per fetch shard, each loading only its own sources' reference files |
| `fundamental-analysis` | agent | Context pollution + dispatched in parallel with three siblings at Stage 2 |
| `sentiment-synthesis` | agent | Context pollution + dispatched in parallel |
| `peer-comparison` | agent | Dispatched in parallel at Stage 2 (inputs are small, but parallel execution is the win) |
| `risk-screen` | agent | Context pollution + dispatched in parallel |
| `thesis-discipline` | agent | Composition isolation — produces the thesis, which is the pipeline's central artifact. Supports `full` and `reconcile` modes via a parameter; dormant `drift` and `implicit` modes reserved for future commands |
| `devils-advocate` | agent | Anchoring protection — must read only thesis.md, not raw/ |
| `report-writer` | agent | Composition isolation — produces 60-150KB HTML output |
| `source-extraction` | skill | Pure reference library — consulted by deep-researcher, never invoked on its own |
| `report-generation` | skill | Pure reference library — consulted by report-writer, never invoked on its own |

**Only two Skills remain**, and both are pure reference libraries (`source-extraction/SKILL.md` with per-source reference files, `report-generation/SKILL.md` with template + compliance + assets). No Skill is ever invoked as a work unit in the pipeline — every work unit is an agent dispatched via Task.

**Historical note:** in v0.3.x, the four analysis skills and thesis-discipline were Skills loaded into the orchestrator's main context. Phase 2.5 session timing data showed this was the single biggest-time artificial bottleneck in the pipeline (~32 min sequential for the four analyses; +12 min for thesis-discipline full which was slowed further by main-context bloat; the reconcile pass alone took 8 min on SNOW). v0.4.0 converted all five to agents — the parallel dispatch at Stage 2 saved ~22 minutes per deep-dive, the context-isolation of thesis-discipline another ~5-8 minutes (~42% total). That left Stage 1's single sequential deep-researcher (~20 min for 12 sources) as the dominant bottleneck, so v0.5.0 sharded it into four parallel fetch agents with disjoint origins, moved the ticker→CIK fail-fast into orchestrator pre-flight, and made the orchestrator the sole meta.json writer (shard results are merged once instead of read-modify-written after every fetch). v0.5.0 also stopped the report-writer from reading `base-styles.css`/`disclaimer.html` into context (they're spliced in via Bash post-composition) and made the compliance pass grep-first instead of re-scanning the full 60-150KB HTML in-context up to 3 times.

## Reduced fetch plans (dormant modes)

`/stockwiz` MODE=full fetches all 12 sources via the four shards. Future commands on the roadmap use reduced source sets — the orchestrating command decides the sources and passes an explicit `SOURCES` list to one or more `deep-researcher` shard dispatches (the agent itself has no mode logic):

- **thesis** (5 sources): SEC EDGAR (core concepts only), Finviz, Stockanalysis, Yahoo, Google News RSS.
- **compare** (4 per ticker): Finviz, Yahoo, Stockanalysis, Google News RSS; SEC only if not already fetched for the ticker today.
- **revisit** (3): Finviz, Yahoo, Google News RSS — files prefixed `revisit-YYYYMMDD-`.
- **bear** (6): the thesis set plus SWS (risks list is high-value bear input) and Reddit (retail sentiment as contrarian signal).

## Where things live

See [`session-workspace.md`](session-workspace.md) for the full filesystem contract and meta.json schema. The short version:

- **Repo source** (`~/Projects/stockwiz/`) — read-only at runtime
  - `commands/` — user-invocable slash commands (2 currently, 6 planned)
  - `agents/` — Task-delegated subagents (8)
  - `skills/` — Skill-tool-loaded instructions, each with a `SKILL.md` and optional `references/` + `assets/`
  - `docs/` — human-facing architecture and contracts (this file + session-workspace.md)

- **Runtime state** (`~/.claude/stockwiz/`) — read-write per user
  - `config.json` — user configuration
  - `cache/` — long-lived caches (SEC ticker map, Macrotrends slugs)
  - `sessions/<TICKER>-<ISO8601>/` — one directory per `/stockwiz` run, immutable after completion

- **Scratch** (`/tmp/stockwiz-*`) — ephemeral, cleaned per-source

## Compliance pass — single enforcement point

Every HTML report runs through a compliance pass before the draft is promoted to `report.html`. The pass is **centralized in exactly one place**: `agents/report-writer.md` Step 9, which applies `skills/report-generation/references/compliance-rules.md` grep-first — a canonical Bash grep locates candidate lines in the on-disk draft, the model judges only those lines against the exemptions, and applies targeted edits. The full HTML is never re-scanned in model context.

The rule set has three classes:

1. **Banned imperative phrases** ("buy", "sell", "recommend", "guaranteed", etc.) → rewritten to neutral analytical language
2. **Exemptions** — words/phrases that LOOK banned but are industry terminology (`sell-side`, `buy-side`) or legally-mandated boilerplate (disclaimer footer contents)
3. **Quote-wrapped source text** — anything inside `<q>...</q>` tags is exempt from rewriting

No other stage runs the compliance pass. Other stages are expected to **pre-filter** their outputs (avoid generating banned language in the first place) to reduce compliance-pass churn, but this is a softer expectation than the report-writer's hard enforcement.

The pre-filter guidance lives in `compliance-rules.md` under a "Pre-filter guidance for skill authors" section. Every analysis agent references that section rather than duplicating the banned list. (The section is named "skill authors" for historical reasons — the guidance applies to any work unit that writes prose destined for the HTML report, which post-0.4.0 means agents.)

## How to add a new source

1. Write `skills/source-extraction/references/<source>.md` following the template set by existing references (`sec-edgar.md`, `finviz.md`, etc.). Document URL pattern, access method (curl or WebFetch), extraction targets, failure modes, and per-source rate limits.
2. Add the source to the index table in `skills/source-extraction/SKILL.md` with its phase, tool, and keyless status.
3. Add the source to a fetch shard: pick the shard whose origins and theme fit (or rebalance), update the shard-assignment table in `skills/source-extraction/SKILL.md` AND the shard prompt in `commands/stockwiz.md` Step 6, including that shard's fetch/WebSearch budget. Per-source mechanics (cookies, caches, etc.) live in the reference file only — `deep-researcher.md` deliberately doesn't duplicate them.
4. If the source needs a persistent cache, document the schema in `docs/session-workspace.md`.
5. If the source produces a new raw file slug, add it to the filesystem contract in `docs/session-workspace.md`.
6. Update the analysis agent(s) under `agents/` that should consume the new source. The source exists in `raw/` but analysis agents won't read it unless their Inputs section says so explicitly.
7. Update the total source count wherever it appears (`commands/stockwiz.md`, `skills/source-extraction/SKILL.md`, this file).

**Common trap**: adding a source to the reference files and index but forgetting to update the orchestrator's shard prompt OR the analysis agents. The source gets fetched but nothing reads it (or is listed but never fetched). See the Phase 2.5 self-review for examples of this exact failure mode.

## How to add a new analysis agent

1. Create `agents/<name>.md` with YAML frontmatter (`name`, `description`, `model: sonnet`, `color`, `tools: Read, Write, Grep, Bash`) and a body that documents: input format, workflow, output structure, hard rules, limitations.
2. Follow the librarian/analyst separation: read from `raw/` only, write exactly one file to `analysis/<name>.md`. Do NOT read other agents' in-flight analysis files (they may not be written yet or may be stubs if they failed).
3. Follow the pre-filter compliance guidance in `compliance-rules.md` — don't generate banned imperative language. Link to the canonical section from your agent's hard rules.
4. Add the agent as a fifth (or sixth, etc.) Task call in the parallel dispatch block at Stage 2 of `commands/stockwiz.md`. All analysis agents at Stage 2 are dispatched in a single message with one Task call per agent.
5. Add a stage entry template for the new agent in Stage 2's "After all four return" section — bump "four" to "N".
6. Update `commands/stockwiz.md` thesis-discipline prompts (Stage 3, full mode) to list the new file as an input.
7. Update the `thesis-discipline` agent's `Inputs` section to list the new file.
8. Update the orchestrator's report-writer Task prompt (Stage 6) to mention the new `analysis/<name>.md` file.
9. Update `deep-dive-template.md` with a rendering spec for the new section.
10. Update `agents/report-writer.md` to handle the new section in its composition list.

**Hard rule for new analysis agents**: they must have `tools: Read, Write, Grep, Bash` only. No WebFetch or WebSearch — those are for deep-researcher and devils-advocate. Analysis agents work from cached raw data, never re-fetch.

## How to add a new command

1. Create `commands/<name>.md` with YAML frontmatter (`description`, `argument-hint`, `allowed-tools`) and a body that documents the pipeline step-by-step as a numbered procedure.
2. If the command delegates to existing agents/skills, document the invocation prompts explicitly.
3. Update `README.md` Commands table to include the new command.
4. Remove the command from `README.md` Roadmap section.
5. Update `stockwiz-setup.md` quickstart if the command is user-facing.
6. If the command uses any dormant modes (thesis-discipline `drift` / `implicit` / `relative`, devils-advocate mode B), update those modes' documentation to say they are now live.

## Open questions and known risks

- **No tests.** The entire verification strategy is manual smoke-testing via actual Claude Code runs. A lightweight `scripts/audit.sh` that checks for stale command references, YAML frontmatter validity, schema consistency, and dangling file references would catch many drift bugs before they ship.
- **Commit-vs-runtime drift.** Every phase has added surface. A quarterly "cleanup commit" that deletes dead scaffolding and updates stale documentation is probably healthy hygiene.
- **Non-deterministic key insight selection.** The TL;DR key insight is deliberately curated by the LLM rather than picked by a deterministic formula. This trades reproducibility for context-aware insight quality. Documented in `deep-dive-template.md`.
