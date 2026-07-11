---
description: Run a full deep-dive research pipeline on a US equity ticker and produce a self-contained HTML report. Produces bull/base/bear thesis with measurable kill switches, runs an adversarial devil's-advocate pass, and stores everything in a durable session workspace for follow-up questions.
argument-hint: <TICKER> [--horizon=long|swing]
allowed-tools: Bash, Read, Write, Glob, Grep, Task, WebFetch, WebSearch, TodoWrite
---

# /stockwiz

You are orchestrating a full equity deep-dive. This command coordinates subagents to produce a complete research session and a self-contained HTML artifact.

The user typed something like `/stockwiz NVDA` or `/stockwiz NVDA --horizon=swing`. Parse the argument, validate, run the pipeline, and return a workspace pointer plus short summary so follow-up questions can be answered from the session workspace.

Every non-trivial unit of work is a subagent dispatched via the `Task` tool — no Skills are loaded into your main context during a deep-dive. Stage 1 dispatches **four fetch shards concurrently**; Stage 2 dispatches **four analysis agents concurrently**; Stages 3–6 dispatch one subagent each, sequentially, because each has a data dependency on the prior stage's output. See `docs/architecture.md` for the full pipeline rationale.

## Step 1 — Parse arguments

The arguments you received are in `$ARGUMENTS`. Parse them:

1. **Split on whitespace.** The first non-flag token is the ticker. Flags start with `--`.
2. **Uppercase the ticker.** Validate it against the regex `^[A-Z][A-Z.\-]{0,6}$`. US tickers are 1–7 characters, alphanumeric, optionally containing `.` or `-` (e.g. `BRK.B`, `BF.B`). If the ticker fails validation or is missing, print:
   > Usage: /stockwiz <TICKER> [--horizon=long|swing]
   > Example: /stockwiz NVDA --horizon=long
   > The ticker must be a valid US equity symbol (1–7 chars, letters, optionally `.` or `-`).
   and exit.
3. **Parse `--horizon=<value>`.** Accept `long` or `swing`. Default to `long` if absent. If another value, print an error and exit.

Store the parsed values in local variables: `TICKER`, `HORIZON`.

## Step 2 — Setup check

Read `~/.claude/stockwiz/config.json`. If the file doesn't exist or isn't valid JSON, print:

> stockwiz is not set up yet. Please run `/stockwiz-setup` first.

and exit. Do NOT attempt to create the config yourself — `/stockwiz-setup` is the canonical place for that, and skipping it would also skip the disclaimer.

## Step 3 — Create the session workspace

Compute the session directory:

```
ISO_TIMESTAMP=$(date -u +%Y%m%dT%H%M%S)   # e.g. 20260411T213200
SESSION_DIR=~/.claude/stockwiz/sessions/${TICKER}-${ISO_TIMESTAMP}
```

Use `Bash` to create the directory structure:

```bash
mkdir -p "$SESSION_DIR/raw" "$SESSION_DIR/analysis"
```

Verify the directories exist. If `mkdir` fails, surface the error and exit.

## Step 4 — Initialize meta.json

Write the initial `$SESSION_DIR/meta.json`:

```json
{
  "schemaVersion": 1,
  "ticker": "<TICKER>",
  "mode": "full",
  "horizon": "<HORIZON>",
  "commandVersion": "0.5.0",
  "createdAt": "<ISO 8601 with offset>",
  "status": "started",
  "stages": [],
  "sources": {},
  "reportPath": null,
  "thesisPath": "thesis.md",
  "followUps": []
}
```

Use the local timezone offset in `createdAt` — e.g. `2026-04-11T14:32:00-07:00`. Use `date -Iseconds` (GNU) or `date "+%Y-%m-%dT%H:%M:%S%z"` (BSD/macOS) to generate it.

## Step 4.5 — Pre-flight: prerequisites and ticker→CIK resolution

Run these checks in your own context via `Bash` **before dispatching any subagent** — they are cheap, and each failure here aborts the run before any fetch budget is spent.

1. **curl** — `which curl`. Missing → fatal, reason `curl-not-installed`.
2. **JSON parser** — `which jq || which python3`. Neither → fatal, reason `json-parser-missing`.
3. **lynx** — `which lynx`. Missing is NOT fatal; record `LYNX=missing` and pass it to the fetch shards (lynx-dependent sources fall back to raw-HTML Read).
4. **SEC tickers cache** — check `~/.claude/stockwiz/cache/company_tickers.json` exists and is less than 30 days old. If not, fetch it (one curl call, per `skills/source-extraction/references/sec-edgar.md` Step 1 — descriptive User-Agent required). If this fetch fails → fatal, reason `sec-unreachable`.
5. **Resolve TICKER → CIK** — look up the ticker in the cache with `jq` or `python3` (see the sec-edgar reference for the snippet). No match → fatal, reason `ticker-not-found-at-sec` ("No SEC filings found for <TICKER>. Verify it is a US-listed equity."). On match, zero-pad to 10 digits: `CIK_PADDED`.

On any fatal condition, jump to Step 13.

## Step 5 — Stage the todo list

Use `TodoWrite` to create the pipeline stages. This gives the user visibility into progress:

```
1. Fetch sources via four fetch shards IN PARALLEL (deep-researcher agent ×4)
2. Run four analysis agents IN PARALLEL (fundamental, sentiment, peer-comp, risk)
3. Synthesize thesis (thesis-discipline agent, full mode)
4. Adversarial stress test (devils-advocate agent)
5. Reconcile thesis with adversarial pass (thesis-discipline agent, reconcile mode)
6. Generate HTML report (report-writer agent)
7. Finalize session metadata
```

Mark stage 1 as in_progress.

## Step 6 — Stage 1: Dispatch four fetch shards IN PARALLEL

The 12 sources are split into four shards with **disjoint origins**, so the per-origin 1500ms politeness delay is preserved inside each shard while the shards run concurrently. Wall-clock for Stage 1 is `max(shard A..D)` instead of the sum.

**Dispatch pattern.** Issue **a single message with four `Task` tool calls**, each invoking the `deep-researcher` agent. Shards write disjoint `raw/` files and never touch `meta.json` — you merge their results after all four return.

Common preamble for every shard prompt:

> Fetch one shard of equity research data for a stockwiz deep-dive.
>
> TICKER: <TICKER>
> HORIZON: <HORIZON>
> SESSION_DIR: <absolute path>
> LYNX: <available|missing>
>
> Fetch ONLY the sources listed below, in order, per their reference files under `skills/source-extraction/references/`. Write each source to SESSION_DIR/raw/ per the file output convention and slugs in source-extraction/SKILL.md. Do NOT touch meta.json — the orchestrator merges shard results. Return your compact shard summary (per-source status lines with fetched_at, key facts, notes) per your agent spec. Do NOT include raw content in your return.

Per-shard specifics:

1. **Shard A — ground truth + snapshot.** `SHARD: A`, `CIK_PADDED: <value>` (already resolved — skip ticker→CIK resolution).
   Sources: SEC EDGAR (curl + JSON APIs; submissions + 5 concepts — the tickers cache is already fresh), Finviz (WebFetch), Barchart insider trades (WebFetch, best-effort).
   Budget: ≤10 fetches, 0 WebSearch. **SEC failure is fatal — if SEC fails, skip the rest of the shard and return immediately with the failure flagged.**

2. **Shard B — financial history.** `SHARD: B`.
   Sources: Stockanalysis.com (curl + lynx, 3 pages), Macrotrends (slug cache → WebSearch only on cache miss; 4 curl pages + lynx).
   Budget: ≤9 fetches, ≤1 WebSearch.

3. **Shard C — news + vendor views.** `SHARD: C`.
   Sources: Google Finance + Google News RSS (curl + RSS + lynx), Simply Wall Street (WebSearch + curl + lynx), Seeking Alpha (curl + lynx, public sections only), Zacks (curl + lynx, best-effort, fail fast on Cloudflare, zero retries).
   Budget: ≤9 fetches, ≤2 WebSearch (SWS URL resolution + optional Google News company-name fallback).

4. **Shard D — market chatter.** `SHARD: D`.
   Sources: Yahoo Finance (curl + cookie jar + quoteSummary JSON API, best-effort — usually rate-limited, no retry on first failure), Reddit (curl + .json endpoints, r/stocks + r/wallstreetbets + r/investing), X.com institutional commentary (1 WebSearch, whitelist-gated, best-effort — empty `quotes: []` is `status: ok`).
   Budget: ≤8 fetches, ≤1 WebSearch.

Wait for all four to return.

### After all four shards return

1. **Merge into meta.json** (one read-modify-write): for every source line in the shard returns, set `meta.json.sources[<slug>] = { "status": ..., "reason": <if failed>, "file": ..., "fetched_at": ... }`. Append four stage entries:
   ```json
   { "stage": "fetch-shard-A", "startedAt": "...", "endedAt": "...", "status": "ok", "succeeded": N, "failed": M, "parallel": true }
   ```
   (one per shard, A through D).

2. **Cross-source sanity check** — from the shards' returned key facts (no raw-file reads needed), compare where sources overlap:
   - Company name: SEC vs Finviz vs Yahoo — flag only dramatic differences, not formatting.
   - Current price and market cap: Finviz vs Yahoo vs Google — flag differences >~3%.
   - Trailing P/E: flag relative disagreement >20%.
   If anything is flagged, Write `raw/_sanity.md` recording each disagreement with both values and their sources. Do NOT resolve disagreements — just log them.

3. **Gates:**
   - **If SEC EDGAR failed** (`meta.json.sources["sec-edgar"].status == "failed"`) — fatal, jump to Step 13 (reason from shard A: `sec-submissions-failed` or `non-us-filer-no-data`). SEC is the ground-truth source — without it, no thesis.
   - **If fewer than 3 sources succeeded in total** — fatal, jump to Step 13, reason `insufficient-sources`.
   - Otherwise mark stage 1 complete and proceed. It is fine (and expected) that some of Yahoo, Zacks, SWS, SA fail — the thesis can be built from the remaining sources.

## Step 7 — Stage 2: Dispatch four analysis agents IN PARALLEL

The four analysis agents are mutually independent: they read from `raw/` (immutable after Stage 1), write to independent `analysis/<name>.md` files, never read each other's in-flight output, and never touch `meta.json`.

**Dispatch pattern.** Issue **a single message with four `Task` tool calls**. Do NOT fall back to sequential dispatch — the concurrency is the point, and there is no shared mutable state to race on.

For each subagent dispatch, the Task prompt should include:

> Analyze the current session and write your output file.
>
> SESSION_DIR: `<absolute path to SESSION_DIR>`
> TICKER: `<TICKER>`
> HORIZON: `<HORIZON>`
>
> Read `SESSION_DIR/meta.json` to discover which `raw/` sources have `status: ok`. Skip failed sources. Follow your agent's hard rules for input discovery, output structure, and return format. Produce your output file at `SESSION_DIR/analysis/<your-slug>.md`. Do NOT touch `meta.json` — the orchestrator will update it after you return. Return a compact summary per your agent's return-summary spec.

The four Task calls:

1. **Agent `fundamental-analysis`** → produces `analysis/fundamental.md`
2. **Agent `sentiment-synthesis`** → produces `analysis/sentiment.md`
3. **Agent `peer-comparison`** → produces `analysis/peer-comp.md`
4. **Agent `risk-screen`** → produces `analysis/risk.md`

Wait for all four to return. Each returns a compact summary (≤200 words).

### After all four return

1. **Verify outputs exist.** For each expected file under `SESSION_DIR/analysis/`, check it was created and is non-empty. A missing file marks that stage entry failed, but you continue — downstream stages handle missing analysis files via fallbacks.

2. **Append stage entries to `meta.json.stages`** — one per agent:
   ```json
   { "stage": "fundamental-analysis", "startedAt": "...", "endedAt": "...", "status": "ok", "output": "analysis/fundamental.md", "parallel": true }
   ```

3. **Failure handling:**
   - Any single agent failing is non-fatal — log its stage entry as `status: "failed"` and continue; thesis-discipline falls back to raw files for the missing lens.
   - All four failing sets `meta.json.status = "degraded"` — still non-fatal, but flag it prominently in the final summary.
   - The Task dispatch itself erroring (an orchestration bug, not an agent failure) is fatal — jump to Step 13.

Mark Stage 2 complete in TodoWrite.

## Step 8 — Stage 3: Synthesize thesis with thesis-discipline (full mode)

Use the `Task` tool to invoke the `thesis-discipline` subagent. Pass it this prompt:

> Synthesize an investment thesis from completed analysis files.
>
> MODE: full
> SESSION_DIR: `<absolute path>`
> TICKER: `<TICKER>`
> HORIZON: `<HORIZON>`
>
> Read the four analysis files (`analysis/fundamental.md`, `analysis/sentiment.md`, `analysis/peer-comp.md`, `analysis/risk.md`). Check each for `status: ok` content; for any that are missing or are failure stubs, fall back to reading the corresponding `raw/` files directly per your agent's fallback mode.
>
> Write `SESSION_DIR/thesis.md` with the mandatory sections: Bull Case / Base Case / Bear Case / Explicit Disconfirmers / Kill Switches / Unknowns. Each case must have a one-sentence Headline, and if the Headline exceeds 140 characters, produce a Compact headline (≤100 characters) alongside for the TL;DR panel. Every claim cited. Base case must be its own scenario (not an average). Kill switches must be measurable (metric + threshold + time window). Unknowns ordered by materiality (most material first).
>
> Return a ≤200-word summary per your agent's return-summary spec for `full` mode.

Wait for the subagent to return. Read its summary.

Append a stage entry to `meta.json.stages`:
```json
{ "stage": "thesis-discipline-full", "startedAt": "...", "endedAt": "...", "status": "ok", "output": "thesis.md", "fallbackMode": false }
```

Set `fallbackMode: true` if the agent reported reading from `raw/` directly (because analysis files were missing).

If the thesis-discipline agent fails entirely (returns an error or no output file), jump to Step 13 (fatal). A thesis is not optional — if we can't build one, there's nothing to run devils-advocate against.

Mark Stage 3 complete.

## Step 9 — Stage 4: Devils-advocate adversarial pass

Use the `Task` tool to invoke the `devils-advocate` subagent. Pass it this prompt:

> Stress-test a stockwiz investment thesis as an isolated adversarial pass.
>
> Input mode: A (thesis.md)
> Thesis file: <absolute path to SESSION_DIR/thesis.md>
> SESSION_DIR: <absolute path>
>
> Read ONLY thesis.md. Do NOT read raw/ files — your anchoring protection depends on seeing only the conclusion, not the bull-framed source data. You may run up to 3 fresh WebFetch/WebSearch calls to find specific disconfirming evidence for your weakest-claim attacks, but do not use them for general research.
>
> Produce SESSION_DIR/analysis/devils-advocate.md with the mandatory sections per agents/devils-advocate.md: Weakest Claims (ranked), The Opposing Narrative, Missing Counter-Evidence, Alternative Interpretations, Kill Switch Adequacy, Fresh Evidence Fetched.
>
> Return a ≤300-word summary: how many weakest claims you identified, what rank-1 was, how many kill switches you rated as inadequate, how many fresh fetches you used.

Wait for the subagent to return. Read its summary.

Append a stage entry:
```json
{ "stage": "devils-advocate", "startedAt": "...", "endedAt": "...", "status": "ok", "weakestClaimsFound": N, "killSwitchesInadequate": M, "freshFetches": K }
```

If the devils-advocate subagent fails (returns an error or the output file is missing), log it as failed with reason and continue — the report-writer will render a placeholder for the adversarial appendix. Non-fatal; the user can re-run `/stockwiz <TICKER>` to retry.

Mark stage 4 complete.

## Step 10 — Stage 5: Reconcile thesis with adversarial pass

**If devils-advocate was skipped or failed in Stage 4**, skip this reconcile step entirely. `thesis.md` remains as Stage 3 wrote it. Record a stage entry with `status: "skipped"` and `reason: "devils-advocate-unavailable"`.

Otherwise, use the `Task` tool to invoke the `thesis-discipline` subagent in `reconcile` mode:

> Reconcile adversarial feedback into an existing investment thesis.
>
> MODE: reconcile
> SESSION_DIR: `<absolute path>`
> TICKER: `<TICKER>`
>
> Read `SESSION_DIR/thesis.md` (the existing thesis from Stage 3) and `SESSION_DIR/analysis/devils-advocate.md` (the adversarial pass from Stage 4). Identify which of devils-advocate's Weakest Claims are materially correct (specific counter-data with citations, not just rhetorical attacks) and which Kill Switch Adequacy verdicts include concrete rewrites.
>
> Append a new section `## Adjustments After Stress Test` to `thesis.md` (just before `## Unknowns`). The original bull/base/bear claims MUST be preserved verbatim — adjustments live in their own section as an audit trail. If devils-advocate raised no material issues, write a short section noting the thesis was found internally consistent.
>
> Return a ≤200-word summary per your agent's return-summary spec for `reconcile` mode.

Wait for the subagent to return.

Append a stage entry to `meta.json.stages`:
```json
{ "stage": "thesis-reconcile", "startedAt": "...", "endedAt": "...", "status": "ok", "adjustmentsApplied": N, "killSwitchesTightened": M }
```

If reconcile fails (returns error or thesis.md wasn't modified when it should have been), log as `status: "failed"` and continue. Non-fatal — the report-writer renders the original thesis without adjustments; the adversarial appendix still shows devils-advocate's findings.

Mark Stage 5 complete.

## Step 11 — Stage 6: Generate HTML report

Use the `Task` tool to invoke the `report-writer` subagent. Pass:

> Generate a stockwiz HTML artifact for session <SESSION_DIR>.
>
> SESSION_DIR: <absolute path>
> TEMPLATE: deep-dive (insights-first)
>
> This is a deep-dive run. The session has raw/ files for up to 12 sources, analysis/ files (fundamental, sentiment, peer-comp, risk, devils-advocate), and a thesis.md (possibly with an Adjustments After Stress Test section from the reconcile step).
>
> **Use the insights-first deep-dive template.** This means:
>
> 1. **First, curate the TL;DR atoms** BEFORE composing HTML (your agent Step 5):
>    - Key insight: ONE most striking observation picked from analysis/fundamental.md and analysis/sentiment.md using the heuristic list in deep-dive-template.md
>    - Closest kill switch: the kill switch from thesis.md with the smallest margin of safety, with current/trigger/margin stats
>    - Biggest unknown: the most material item from thesis.md § Unknowns, one sentence
>
> 2. Compose the HTML in the section order per deep-dive-template.md:
>    - **Above the fold** (always visible): hero → key metrics strip → TL;DR panel (compact three cases hash-shuffled + three callouts for insight/kill-switch/unknown)
>    - **Below the fold** (native <details>/<summary>, no JavaScript): Three Cases (full, open by default) → Kill Switches → Fundamentals → Sentiment → Peers → Risk → Assumption Ledger → Unknowns → Sources → Adversarial Pass
>    - **Always visible at bottom**: Disclaimer footer, NOT inside a <details>
>
> 3. For each <details> section, produce a one-line abstract in the <summary> per the abstract formulas in your agent Step 6.5.
>
> 4. The compact TL;DR cases and the full Three Cases details section MUST use the SAME hash-shuffled order (FNV-1a hash of ticker mod 6).
>
> Before rendering each below-the-fold section, check whether the corresponding analysis file exists and is non-empty. If any file is missing or is a failure stub, render a thin fallback from raw/ files directly per the deep-dive-template graceful-degradation rule. Record per-section fidelity (full / thin / placeholder) in your return summary.
>
> Run the grep-first compliance pass per skills/report-generation/references/compliance-rules.md before promoting the draft to report.html. Exemptions: "sell-side", "buy-side", and any text inside `stockwiz-disclaimer` class are NOT rewritten. Analyst distribution labels ("Strong Buy" etc), Yahoo `recommendationKey` strings, Zacks Rank text labels, SA Quant Rating labels, SWS risks list bullets, Reddit post titles, and Google News headlines MUST be wrapped in `<q>` tags when reproduced verbatim.
>
> Return a compact summary with file path, byte count, curated TL;DR atoms, sections present (full/thin/placeholder per below-the-fold section), compliance rewrites applied, stripped sentences, and sanity check results.

Wait for the subagent to return.

**If report-writer returned an error** (compliance failed, sanity check failed, write failed):
1. Append a stage entry with `status: "failed"` and the error reason.
2. Retry **once** with a refined prompt noting the specific issue.
3. If still failing, jump to Step 13 (fatal error).

**On success:**
1. Append a stage entry for report-writer, recording the compliance log from its return summary as `adjustments: [...]` and `strippedSentences: [...]`.
2. Update `meta.json.reportPath = "report.html"`.
3. Mark stage 6 complete.

## Step 12 — Finalize

Update `$SESSION_DIR/meta.json`:

```json
{
  ...
  "status": "complete",
  "completedAt": "<ISO 8601>",
  "reportPath": "report.html"
}
```

Mark stage 7 complete in TodoWrite.

Print the final assistant message to the user. It should include:

1. **A workspace pointer** (machine-readable-ish block) so follow-up questions land on the right files:
   ```
   <workspace>
   ticker: <TICKER>
   dir: <absolute SESSION_DIR>
   files: raw/sec-edgar-10k.md, raw/yahoo-fundamentals.md, raw/finviz-snapshot.md, thesis.md, report.html
   thesis: thesis.md
   report: report.html
   </workspace>
   ```

2. **A 10-line summary extracted from `thesis.md`**:
   - Bull case headline
   - Base case headline
   - Bear case headline
   - 1–2 of the most important kill switches
   - Source coverage ("11 of 12 sources succeeded, Zacks failed")
   - Path to the HTML report (not opened automatically — tell the user to open it)

3. **A one-line next-step hint**:
   > "Ask a follow-up question to dig into any section — I'll read from this session workspace without re-fetching, so follow-ups are cheap. Or run `/stockwiz <OTHER_TICKER>` to analyze another name."

Do NOT auto-open the HTML file. The user should choose when to view it.

## Step 13 — Fatal error handling

This step is the destination of every fatal-error jump from earlier steps. **It must handle every case that jumps here**; if you add a new fatal-error path upstream, add it to this list too.

### Fatal conditions (exhaustive list)

Any of these must set `meta.json.status = "failed"`, stop the pipeline, and surface a diagnostic:

1. **Setup not run** (Step 2) — `~/.claude/stockwiz/config.json` missing or invalid. User must run `/stockwiz-setup` first.
2. **Invalid ticker argument** (Step 1) — ticker fails regex `^[A-Z][A-Z.\-]{0,6}$` or is missing entirely. Usage message shown, exit.
3. **mkdir failure** (Step 3) — creating `$SESSION_DIR/raw/` or `$SESSION_DIR/analysis/` failed (permission, disk full). Surface OS error.
4. **curl not installed** (Step 4.5) — hard dependency missing. Reason `curl-not-installed`.
5. **JSON parser missing** (Step 4.5) — neither `jq` nor `python3` available. Reason `json-parser-missing`.
6. **SEC tickers cache unreachable** (Step 4.5) — the `company_tickers.json` fetch failed (403 UA issue, DNS, network error) and no valid cache exists. Reason `sec-unreachable`.
7. **Ticker not found at SEC** (Step 4.5) — no match in `company_tickers.json`. Reason `ticker-not-found-at-sec`. Fatal because SEC is the ground-truth source; without it, no thesis.
8. **SEC EDGAR submissions API failed** (Step 6, shard A) — CIK resolved but the submissions or companyconcept APIs returned 403/503/timeout even after one retry. Reason `sec-submissions-failed`.
9. **Non-US filer (20-F only) with zero recoverable structured data** (Step 6, shard A) — the filer has only 20-F filings AND no XBRL concept data could be extracted. Reason `non-us-filer-no-data`.
10. **Fewer than 3 sources succeeded in total** (Step 6) — reason `insufficient-sources`. No credible thesis can be built from 2 or fewer vendor snapshots.
11. **report-writer failed twice** (Step 11) — two consecutive attempts at HTML composition returned errors. Reason carries the underlying error.
12. **Compliance pass unresolvable** (Step 11) — the compliance pass ran 3 iterations and still had banned phrases outside `<q>` tags. Reason `compliance-stuck` — we refuse to emit a non-compliant report.

### Handling procedure

1. Update `$SESSION_DIR/meta.json`:
   ```json
   {
     ...
     "status": "failed",
     "failedAt": "<ISO 8601>",
     "error": "<reason code + one-line description>",
     "failedAtStage": "<stage-name-that-jumped-here>"
   }
   ```
2. Do NOT emit `report.html`. If a partial draft exists from a failed report-writer attempt, delete it.
3. Print to the user:
   > stockwiz deep-dive on `<TICKER>` failed.
   > Stage: `<failed-stage>`
   > Reason: `<reason code>` — `<human-readable explanation>`
   > Session workspace saved at `<SESSION_DIR>` for inspection.
   > `<one or two sentences of diagnostic guidance>`
4. Return.

### Non-fatal failures (for reference — these do NOT jump to Step 13)

- Any individual non-SEC source failing (Zacks Cloudflare, Yahoo rate limit, Reddit 429, SWS paywall, SA SSR gap) — the shard marks the source `failed` and continues with its remaining sources.
- `lynx` not installed — shards fall back to raw-HTML Read for lynx-dependent sources.
- Any individual analysis agent writing a stub — downstream agents and report-writer handle stubs via thin fallbacks.
- devils-advocate failing — Stage 5 (reconcile) is skipped, report-writer renders the Adversarial Appendix as a placeholder.
- Cookie jar leaks in `/tmp/stockwiz-*` — cleanup bug, not a fatal condition.

## Edge cases

- **Same-ticker same-second.** The timestamp includes seconds, so collisions are extremely unlikely. If one occurs (mkdir fails with EEXIST), append a short suffix and retry once.
- **Non-US filer (20-F only).** Shard A extracts whatever structured XBRL data is available and flags the source. Proceed if any data was recovered, else treat as fatal SEC failure.
- **User interrupts mid-run.** The session dir remains on disk with `status: "started"` and whatever partial files were written. The user can inspect it or re-run `/stockwiz <TICKER>` to start fresh.
- **Disk full / permission error during session dir creation.** Surface the OS error, abort.
- **One shard returns nothing** (Task error, not a source failure). Mark its sources as failed with reason `shard-dispatch-error` and apply the normal gates: fatal only if SEC was in that shard or the total ok-count drops below 3.

## What success looks like

The user runs `/stockwiz NVDA` and sees a seven-stage todo list. Stage 1 fetches 12 sources via four concurrent fetch shards (~6–8 minutes, max of the four instead of the sum). Stage 2 runs the four analysis agents concurrently (~10 minutes). Stages 3–5 synthesize the thesis, attack it with devils-advocate, and reconcile the material findings. Stage 6 composes a 60–150KB insights-first HTML report with a compliance-clean disclaimer footer. Total wall-clock ~25–30 minutes. The user opens `report.html`, sees three equal-weight cases, measurable kill switches with margins of safety, and a red adversarial appendix — then asks follow-up questions that are answered from the session workspace without re-fetching.
