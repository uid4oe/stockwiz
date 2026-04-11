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

- **The analyst** (four analysis skills + thesis-discipline + devils-advocate + report-writer) reads the librarian's output and interprets it. Every analyst component writes its own file under `analysis/` (or `thesis.md`, `report.html`). The analyst may disagree with the raw data, frame it differently, or flag it — but the analyst never rewrites it.

The benefit: when a downstream output is questioned, you can always grep `raw/` to see what the source actually said. The audit trail is immutable and the interpretation layer is reversible.

Violations to watch for:

- Don't let deep-researcher editorialize ("this looks like a strong company"). It's a librarian.
- Don't let analysis skills re-fetch from the web. They read from `raw/` only.
- Don't let thesis-discipline modify raw data, only analysis outputs.
- Don't let the report-writer invent numbers; if it can't find a citation in `raw/` or `analysis/`, the number doesn't appear in the report.

## State-flow diagram: `/stockwiz <TICKER>` pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                         /stockwiz <TICKER>                           │
│                     (commands/stockwiz.md — 416 lines)               │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Steps 1-5: pre-flight                                               │
│  ├─ parse args (TICKER, --horizon)                                   │
│  ├─ check config.json exists (else exit: need /stockwiz-setup)       │
│  ├─ create SESSION_DIR/raw/, SESSION_DIR/analysis/                   │
│  ├─ write initial meta.json (schemaVersion=1, status=started)        │
│  └─ TodoWrite the 7 pipeline stages                                  │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STAGE 1 — deep-researcher (agent, Task subagent, isolated context)  │
│  Fetches 10 sources in priority order:                               │
│     SEC EDGAR → Finviz → Stockanalysis → Macrotrends → Yahoo →       │
│     Google News RSS → SWS → Seeking Alpha → Zacks → Reddit           │
│  Writes raw/*.md files, updates meta.json.sources[]                  │
│  Returns a compact summary (≤300 words, no raw content)              │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                     ┌───────────┴──────────┐
                     │                      │
               SEC ok?                SEC failed?
                     │                      │
                     ▼                      ▼
┌───────────────────────────┐   ┌──────────────────────────────┐
│  STAGE 2 — 4 analysis     │   │  Step 13: FATAL ERROR        │
│  skills (Skill tool,      │   │  meta.json.status = failed   │
│  sequential, main context)│   │  No thesis, no report        │
│  ├─ fundamental-analysis  │   │  Return diagnostic to user   │
│  ├─ sentiment-synthesis   │   └──────────────────────────────┘
│  ├─ peer-comparison       │
│  └─ risk-screen           │
│  Each reads raw/*, writes │
│    analysis/*.md          │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────┐
│  STAGE 3 — thesis-discipline (full mode, Skill tool)      │
│  Reads analysis/* files, writes thesis.md with:           │
│    ## Bull Case ## Base Case ## Bear Case                 │
│    ## Explicit Disconfirmers ## Kill Switches ## Unknowns │
│  Every claim cited, kill switches measurable              │
└────────────┬──────────────────────────────────────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────┐
│  STAGE 4 — devils-advocate (agent, Task, isolated)        │
│  Input: ONLY thesis.md (no raw/ access, anchoring proof)  │
│  Output: analysis/devils-advocate.md                      │
│    ## Weakest Claims (ranked)                             │
│    ## Opposing Narrative                                  │
│    ## Missing Counter-Evidence                            │
│    ## Alternative Interpretations                         │
│    ## Kill Switch Adequacy                                │
│  Up to 3 fresh WebFetch/WebSearch for disconfirming data  │
│  NON-FATAL on failure (report gets a placeholder)         │
└────────────┬──────────────────────────────────────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────┐
│  STAGE 5 — thesis-discipline (reconcile mode, Skill tool) │
│  Reads thesis.md + analysis/devils-advocate.md            │
│  APPENDS ## Adjustments After Stress Test to thesis.md    │
│  NEVER rewrites original bull/base/bear claims            │
│  (Skipped if Stage 4 failed)                              │
└────────────┬──────────────────────────────────────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────┐
│  STAGE 6 — report-writer (agent, Task, isolated)          │
│  Step 5: curate TL;DR atoms (key insight / closest        │
│          kill switch / biggest unknown)                   │
│  Step 6: compose v0.3 insights-first HTML:                │
│    hero → metrics → TL;DR → <details> sections → footer   │
│  Step 8: compliance pass (≤3 iterations, abort if fails)  │
│  Step 9: write report.html                                │
│  Step 10: verify (size, markers, compliance grep)         │
└────────────┬──────────────────────────────────────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────┐
│  STAGE 7 — finalize                                       │
│  Update meta.json: status=complete, reportPath, times     │
│  Print workspace pointer + TL;DR summary to user          │
│  Leave session hot for conversational follow-ups          │
└───────────────────────────────────────────────────────────┘
```

### Fatal vs non-fatal failures

Only the **SEC EDGAR fetch** is fatal in the normal pipeline. The reasoning: SEC filings are the ground truth for US equities; without them, every downstream claim is ungrounded. Everything else is non-fatal:

- **Any individual non-SEC source can fail** (Zacks Cloudflare, Yahoo 429, Reddit JSON-body 429, etc.) — deep-researcher logs the failure and continues to the next source. The analysis layer gracefully degrades and flags the gap in the Unknowns section.
- **Fewer than 3 total sources succeeding** is fatal — not enough data to build a credible thesis.
- **An individual analysis skill failing** is non-fatal — the skill writes a stub, thesis-discipline falls back to reading raw files directly, and report-writer renders a thin section with a placeholder note.
- **devils-advocate failing** is non-fatal — Stage 5 is skipped, Stage 6 renders the adversarial appendix as a placeholder.
- **report-writer compliance pass failing after 3 iterations** is fatal — we refuse to emit a non-compliant report rather than silently pass.

These are all enforced in `commands/stockwiz.md` Step 6 / Step 9 / Step 13. The complete fatal-error list lives in Step 13.

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

### Current assignments

| Component | Type | Primary reason |
|---|---|---|
| `deep-researcher` | agent | Context pollution — reads raw HTML, JSON, lynx dumps |
| `devils-advocate` | agent | Anchoring protection — must read only thesis.md, not raw/ |
| `report-writer` | agent | Composition isolation — produces 60-150KB HTML output |
| `fundamental-analysis` | skill | Currently a skill; **candidate for agent-ification** (reads many raw files) |
| `sentiment-synthesis` | skill | Currently a skill; **candidate for agent-ification** |
| `peer-comparison` | skill | Skill is fine (small inputs, short output) |
| `risk-screen` | skill | Currently a skill; candidate for agent-ification |
| `thesis-discipline` | skill | Skill — runs in main context by design (needs visibility of analysis outputs) |
| `source-extraction` | skill | Pure reference — never invoked on its own, only consulted |
| `report-generation` | skill | Pure reference — consulted by report-writer agent |

**Known open question:** the four analysis skills run sequentially in the orchestrator's main context. This accumulates ~200KB of raw and analysis content in the main context before the orchestrator even hands off to devils-advocate and report-writer. Wrapping each analysis skill as a subagent would unlock context isolation and parallelism, at the cost of five more Task-tool calls per run. This is a roadmap item, not a current bug, but it's the single most impactful architectural change available.

## Where things live

See [`session-workspace.md`](session-workspace.md) for the full filesystem contract and meta.json schema. The short version:

- **Repo source** (`~/Projects/stockwiz/`) — read-only at runtime
  - `commands/` — user-invocable slash commands (2 currently, 6 planned)
  - `agents/` — Task-delegated subagents (3)
  - `skills/` — Skill-tool-loaded instructions, each with a `SKILL.md` and optional `references/` + `assets/`
  - `docs/` — human-facing architecture and contracts (this file + session-workspace.md)

- **Runtime state** (`~/.claude/stockwiz/`) — read-write per user
  - `config.json` — user configuration
  - `cache/` — long-lived caches (SEC ticker map, Macrotrends slugs)
  - `sessions/<TICKER>-<ISO8601>/` — one directory per `/stockwiz` run, immutable after completion

- **Scratch** (`/tmp/stockwiz-*`) — ephemeral, cleaned per-source

## Compliance pass — single enforcement point

Every HTML report runs through a compliance pass before being written to disk. The pass is **centralized in exactly one place**: `agents/report-writer.md` Step 8, which loads `skills/report-generation/references/compliance-rules.md` and applies it.

The rule set has three classes:

1. **Banned imperative phrases** ("buy", "sell", "recommend", "guaranteed", etc.) → rewritten to neutral analytical language
2. **Exemptions** — words/phrases that LOOK banned but are industry terminology (`sell-side`, `buy-side`) or legally-mandated boilerplate (disclaimer footer contents)
3. **Quote-wrapped source text** — anything inside `<q>...</q>` tags is exempt from rewriting

No other stage runs the compliance pass. Other stages are expected to **pre-filter** their outputs (avoid generating banned language in the first place) to reduce compliance-pass churn, but this is a softer expectation than the report-writer's hard enforcement.

The pre-filter guidance lives in `compliance-rules.md` under a "Pre-filter guidance for skill authors" section. Every analysis skill references that section rather than duplicating the banned list.

## How to add a new source

1. Write `skills/source-extraction/references/<source>.md` following the template set by existing references (`sec-edgar.md`, `finviz.md`, etc.). Document URL pattern, access method (curl or WebFetch), extraction targets, failure modes, and per-source rate limits.
2. Add the source to the index table in `skills/source-extraction/SKILL.md` with its phase, tool, and keyless status.
3. Add the source to the fetch plan in `agents/deep-researcher.md` Step 2 at the appropriate priority position. Update any per-source mechanics if the source needs special handling (cookies, cache, etc.).
4. If the source needs a persistent cache, document the schema in `docs/session-workspace.md`.
5. If the source produces a new raw file slug, add it to the filesystem contract in `docs/session-workspace.md`.
6. Update the analysis skill(s) that should consume the new source. The source exists in `raw/` but analysis skills won't read it unless their Inputs section says so explicitly.
7. Update the orchestrator's deep-researcher Task prompt in `commands/stockwiz.md` if the total source count changes.

**Common trap**: adding a source to the reference files and index but forgetting to update the orchestrator prompt OR the analysis skills. The source gets fetched but nothing reads it. See the Phase 2.5 self-review for examples of this exact failure mode.

## How to add a new analysis skill

1. Create `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`, `version`) and a body that documents: when to use, inputs, output structure, hard rules, limitations.
2. Follow the librarian/analyst separation: read from `raw/` or `analysis/<other-skill>.md`, write exactly one file to `analysis/<name>.md`.
3. Follow the pre-filter compliance guidance in `compliance-rules.md` — don't generate banned imperative language.
4. Add the skill to `commands/stockwiz.md` Step 7 as a new Stage 2X substage.
5. Update the orchestrator's report-writer Task prompt in `commands/stockwiz.md` to mention the new `analysis/<name>.md` file.
6. Update `deep-dive-template.md` with a rendering spec for the new section.
7. Update `report-writer.md` to handle the new section in its composition list.

## How to add a new command

1. Create `commands/<name>.md` with YAML frontmatter (`description`, `argument-hint`, `allowed-tools`) and a body that documents the pipeline step-by-step as a numbered procedure.
2. If the command delegates to existing agents/skills, document the invocation prompts explicitly.
3. Update `README.md` Commands table to include the new command.
4. Remove the command from `README.md` Roadmap section.
5. Update `stockwiz-setup.md` quickstart if the command is user-facing.
6. If the command uses any dormant modes (thesis-discipline `drift` / `implicit` / `relative`, devils-advocate mode B), update those modes' documentation to say they are now live.

## Open questions and known risks

- **Analysis skills in main context**: see decision rule section above. This is the biggest pending architectural change.
- **No tests.** The entire verification strategy is manual smoke-testing via actual Claude Code runs. A lightweight `scripts/audit.sh` that checks for stale command references, YAML frontmatter validity, schema consistency, and dangling file references would catch many drift bugs before they ship.
- **Commit-vs-runtime drift.** Every phase has added surface. A quarterly "cleanup commit" that deletes dead scaffolding and updates stale documentation is probably healthy hygiene.
- **Non-deterministic key insight selection.** The TL;DR key insight is deliberately curated by the LLM rather than picked by a deterministic formula. This trades reproducibility for context-aware insight quality. Documented in `deep-dive-template.md`.
