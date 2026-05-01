---
description: Run a full deep-dive research pipeline on a US equity ticker and produce a self-contained HTML report. Produces bull/base/bear thesis with measurable kill switches, runs an adversarial devil's-advocate pass, and stores everything in a durable session workspace for follow-up questions.
argument-hint: <TICKER> [--horizon=long|swing]
allowed-tools: Bash, Read, Write, Glob, Grep, Task, WebFetch, WebSearch, TodoWrite
---

# /stockwiz

You are orchestrating a full equity deep-dive. This command coordinates subagents and skills to produce a complete research session and a self-contained HTML artifact.

The user typed something like `/stockwiz NVDA` or `/stockwiz NVDA --horizon=swing`. Parse the argument, validate, run the pipeline, and return a workspace pointer plus short summary so follow-up questions can be answered from the session workspace.

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
  "commandVersion": "0.4.2",
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

## Step 5 — Stage the todo list

Use `TodoWrite` to create the pipeline stages. This gives the user visibility into progress:

```
1. Gather data from public sources (deep-researcher agent)
2. Run four analysis agents IN PARALLEL (fundamental, sentiment, peer-comp, risk)
3. Synthesize thesis (thesis-discipline agent, full mode)
4. Adversarial stress test (devils-advocate agent)
5. Reconcile thesis with adversarial pass (thesis-discipline agent, reconcile mode)
6. Generate HTML report (report-writer agent)
7. Finalize session metadata
```

**v0.4.0 architectural note.** Every non-trivial unit of work in the pipeline is now a subagent dispatched via the `Task` tool — no Skills are loaded into the orchestrator's main context during a deep-dive. Stage 2's four analysis agents run **concurrently** (dispatched in a single message with four Task calls). Stages 1, 3, 4, 5, 6 dispatch one subagent each, sequentially, because each has a data dependency on the prior stage's output. `thesis-discipline` is invoked twice — once in `full` mode (Stage 3) to synthesize, once in `reconcile` mode (Stage 5) to merge devils-advocate feedback — each invocation is an independent Task with an isolated context.

Mark stage 1 as in_progress.

## Step 6 — Stage 1: Delegate to deep-researcher

Use the `Task` tool to invoke the `deep-researcher` subagent. Pass it this prompt:

> Fetch equity research data for a stockwiz deep-dive.
>
> TICKER: <TICKER>
> HORIZON: <HORIZON>
> SESSION_DIR: <absolute path to SESSION_DIR>
> MODE: full
>
> This is a deep-dive run. Fetch the 12 current sources per the source-extraction skill, in this order:
>   1. SEC EDGAR (curl + SEC JSON APIs) — fatal if this fails
>   2. Finviz (WebFetch)
>   3. Stockanalysis.com (curl + lynx) — 5Y financials
>   4. Macrotrends (WebSearch + curl + lynx) — 10-20Y deep cycle history
>   5. Yahoo Finance (curl + quoteSummary JSON API)
>   6. Google Finance + Google News RSS (curl + RSS + lynx) — news layer
>   7. Simply Wall Street (WebSearch + curl + lynx)
>   8. Seeking Alpha (curl + lynx) — public sections only
>   9. Zacks (curl + lynx) — best-effort, fail fast on Cloudflare
>   10. Reddit (curl + .json endpoints for r/stocks, r/wallstreetbets, r/investing)
>   11. Barchart insider trades (WebFetch) — best-effort; 3/6/12-month buy/sell directional time series
>   12. X.com institutional commentary (WebSearch site:x.com, whitelist-gated) — best-effort; verified handles only
>
> Each source's reference file under skills/source-extraction/references/ prescribes the exact tool, URLs, extraction targets, and failure mode detection — follow them. Do NOT substitute WebFetch for curl on gated sources (SEC, Yahoo, Stockanalysis, Macrotrends, Google, SWS, SA, Zacks, Reddit). Finviz and Barchart are WebFetch sources; X.com is the only WebSearch-only source.
>
> SEC EDGAR is the only fatal source. If SEC fails, stop fetching and set meta.json.status to "failed". All other source failures (Zacks Cloudflare, Yahoo rate limit, Reddit 429, Barchart 403, etc.) are non-fatal — continue to the next source. Zacks, Reddit, Barchart, and X.com are explicitly best-effort; do not retry challenges. For X.com, an empty whitelist-hit set (`quotes: []`) is `status: ok`, not failure.
>
> Write each fetched source to SESSION_DIR/raw/ per the file output convention and update SESSION_DIR/meta.json.sources after each fetch. Use the slug conventions from source-extraction/SKILL.md (sec-edgar-10k.md, finviz-snapshot.md, stockanalysis.md, macrotrends.md, yahoo-fundamentals.md, google-finance.md, simply-wall-street.md, seeking-alpha.md, zacks-snapshot.md, reddit.md, barchart-insider-trades.md, x-com-commentary.md).
>
> Return a compact summary (≤300 words) with succeeded sources, failed sources + reasons, absolute file paths, and any cross-source sanity flags. Do NOT include raw content in your return.

Wait for the subagent to return. Read its summary.

**After deep-researcher returns:**

1. Re-read `$SESSION_DIR/meta.json` to see which sources succeeded.
2. Append a stage entry:
   ```json
   { "stage": "deep-researcher", "startedAt": "...", "endedAt": "...", "status": "ok", "succeeded": N, "failed": M }
   ```
3. **If SEC EDGAR failed** (check `meta.json.sources["sec-edgar"].status == "failed"`), jump to Step 11 (fatal error). SEC is the ground-truth source — without it, no thesis.
4. **If fewer than 3 sources succeeded in total**, also jump to Step 13 — there's not enough data to build a credible thesis even with the non-fatal sources intact.
5. Otherwise, mark stage 1 complete in TodoWrite and proceed. It is fine (and expected) that some of SWS, SA, Yahoo may have failed — the thesis can still be built from the remaining sources.

## Step 7 — Stage 2: Dispatch four analysis agents IN PARALLEL

**Previously sequential, now concurrent.** v0.3.x ran the four analysis skills sequentially in the main context, averaging ~32 minutes of wall-clock time (fundamental 10min + sentiment 9min + peer 5min + risk 8min). Measured Phase 2.5 sessions showed this was the single biggest-time artificial bottleneck in the pipeline. In v0.4.0 the four are subagents dispatched concurrently via the `Task` tool, so wall-clock time on this stage becomes `max(fundamental, sentiment, peer, risk)` ≈ 10 minutes, saving ~22 minutes per deep-dive.

The four analysis agents are mutually independent:
- They read from `raw/` (immutable after Stage 1 completes)
- They write to independent `analysis/<name>.md` files
- They do NOT read each other's in-flight output
- They do NOT touch `meta.json` — the orchestrator updates it after all four return

**Dispatch pattern.** Issue **a single message with four `Task` tool calls**. Claude Code's Task tool runs them concurrently when launched in one message. Each subagent runs in its own isolated context.

For each subagent dispatch, the Task prompt should include:

> Analyze the current session and write your output file.
>
> SESSION_DIR: `<absolute path to SESSION_DIR>`
> TICKER: `<TICKER>`
> HORIZON: `<HORIZON>`
>
> Read `SESSION_DIR/meta.json` to discover which `raw/` sources have `status: ok`. Skip failed sources. Follow your agent's hard rules for input discovery, output structure, and return format. Produce your output file at `SESSION_DIR/analysis/<your-slug>.md`. Do NOT touch `meta.json` — the orchestrator will update it after you return. Return a compact summary per your agent's return-summary spec.

Dispatch four `Task` calls **in one message** (not sequentially):

1. **Agent `fundamental-analysis`** → produces `analysis/fundamental.md` (reads SEC, Stockanalysis, Macrotrends, Finviz, optionally Yahoo / SWS / Google Finance)
2. **Agent `sentiment-synthesis`** → produces `analysis/sentiment.md` (reads Google News, SWS, SA, Zacks, Finviz, SEC 8-K, Reddit, optionally Yahoo / Stockanalysis)
3. **Agent `peer-comparison`** → produces `analysis/peer-comp.md` (reads SWS competitor snowflakes, Finviz sector tags, Stockanalysis target metrics)
4. **Agent `risk-screen`** → produces `analysis/risk.md` (reads Finviz, Stockanalysis, Macrotrends, SEC, SWS, SA, Google News)

Wait for all four to return. Each returns a compact summary (≤200 words) per its agent's return-summary spec.

### After all four return

1. **Verify outputs exist.** For each of the four expected files under `SESSION_DIR/analysis/`, check that it was created and is non-empty. If a file is missing, mark that stage entry as failed but continue — downstream stages handle missing analysis files via fallbacks.

2. **Append stage entries to `meta.json.stages`** — one per agent, from the compact summaries:
   ```json
   { "stage": "fundamental-analysis", "startedAt": "...", "endedAt": "...", "status": "ok", "output": "analysis/fundamental.md", "parallel": true }
   { "stage": "sentiment-synthesis",  "startedAt": "...", "endedAt": "...", "status": "ok", "output": "analysis/sentiment.md",  "parallel": true }
   { "stage": "peer-comparison",      "startedAt": "...", "endedAt": "...", "status": "ok", "output": "analysis/peer-comp.md",  "parallel": true }
   { "stage": "risk-screen",          "startedAt": "...", "endedAt": "...", "status": "ok", "output": "analysis/risk.md",       "parallel": true }
   ```
   The `parallel: true` field is informational — it marks that these four ran concurrently so post-run timing analysis can tell.

3. **Failure handling:**
   - If **any single agent** returned an error or produced no output file, log its stage entry as `status: "failed"` with the error reason and continue. One or two analysis failures are non-fatal; thesis-discipline's fallback mode reads from raw files directly when an analysis file is missing.
   - If **all four agents failed**, log four failed stage entries and set `meta.json.status = "degraded"`. Still not fatal — thesis-discipline will fall back entirely to raw files. But flag this prominently in the final summary so the user knows quality is reduced.
   - If the **Task dispatch itself errors** (not an agent failure — an orchestration bug), that's fatal — jump to Step 13.

Mark Stage 2 complete in TodoWrite.

### Why this isn't a race condition

The four agents write to four independent files (`analysis/fundamental.md`, `analysis/sentiment.md`, `analysis/peer-comp.md`, `analysis/risk.md`) — no file is written by more than one agent. None of them touches `meta.json` — only the orchestrator updates it, and only after all four return. The `raw/` files are read-only at this point in the pipeline (deep-researcher finished writing them in Stage 1, nothing modifies them afterward). There is no shared mutable state.

### Why NOT to sequentialize as a fallback

Do NOT fall back to sequential dispatch "for safety". The parallel pattern is the whole point of this refactor — sequential would reintroduce the ~22 minutes of artificial latency we just eliminated. The `Task` tool documentation explicitly supports concurrent subagent dispatch via multiple tool uses in one message. Trust it.

## Step 8 — Stage 3: Synthesize thesis with thesis-discipline (full mode)

**v0.4.0 change:** `thesis-discipline` is now a subagent (not a Skill), dispatched via `Task`. This runs it in an isolated context that reads only the four analysis files (or raw/ fallbacks), not the orchestrator's accumulated state. Expected speedup: reduces Stage 3 from ~12 min to ~4-6 min by eliminating main-context bloat.

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
{
  "stage": "thesis-discipline-full",
  "startedAt": "...",
  "endedAt": "...",
  "status": "ok",
  "output": "thesis.md",
  "fallbackMode": false
}
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
{
  "stage": "devils-advocate",
  "startedAt": "...",
  "endedAt": "...",
  "status": "ok",
  "weakestClaimsFound": N,
  "killSwitchesInadequate": M,
  "freshFetches": K
}
```

If the devils-advocate subagent fails (returns an error or the output file is missing), log it as failed with reason and continue — the report-writer will render a placeholder for the adversarial appendix. Non-fatal; the user can re-run `/stockwiz <TICKER>` to retry. (A future `/stockwiz-bear` command is on the roadmap to allow targeted adversarial re-runs without re-fetching.)

Mark stage 4 complete.

## Step 10 — Stage 5: Reconcile thesis with adversarial pass

**v0.4.0 change:** `thesis-discipline` reconcile is now a subagent dispatched via `Task`, running in an isolated context that reads only `thesis.md` + `analysis/devils-advocate.md`. Expected speedup: Phase 2.5 measurements showed this stage taking 8 min on SNOW due to main-context bloat; isolated it should drop to 2-3 min.

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
{
  "stage": "thesis-reconcile",
  "startedAt": "...",
  "endedAt": "...",
  "status": "ok",
  "adjustmentsApplied": N,
  "killSwitchesTightened": M
}
```

If reconcile fails (returns error or thesis.md wasn't modified when it should have been), log as `status: "failed"` and continue. Non-fatal — the report-writer will render the original thesis without adjustments. The adversarial appendix still shows devils-advocate's findings; the user can see the delta manually even without the reconcile merge.

Mark Stage 5 complete.

## Step 11 — Stage 6: Generate HTML report

Use the `Task` tool to invoke the `report-writer` subagent. Pass:

> Generate a stockwiz HTML artifact for session <SESSION_DIR>.
>
> SESSION_DIR: <absolute path>
> TEMPLATE: deep-dive (v0.3 insights-first)
>
> This is a deep-dive run. The session has raw/ files for up to 12 sources (SEC EDGAR, Finviz, Stockanalysis, Macrotrends, Yahoo, Google Finance + Google News RSS, Simply Wall Street, Seeking Alpha, Zacks, Reddit, Barchart insider trades, X.com institutional commentary), analysis/ files (fundamental, sentiment, peer-comp, risk, devils-advocate) produced by the four analysis agents dispatched in parallel at Stage 2 plus the devils-advocate agent at Stage 4, and a thesis.md (possibly with an Adjustments After Stress Test section from the reconcile step).
>
> **Use the v0.3 insights-first template.** This means:
>
> 1. **First, run Step 5 of your agent instructions — curate the TL;DR atoms** BEFORE composing HTML:
>    - Key insight: ONE most striking observation picked from analysis/fundamental.md and analysis/sentiment.md using the heuristic list in deep-dive-template.md (anomalous quality metric / step-change growth / capital-structure surprise / margin inflection / historical outlier / SWS extreme)
>    - Closest kill switch: the kill switch from thesis.md with the smallest margin of safety, with current/trigger/margin stats
>    - Biggest unknown: the most material item from thesis.md § Unknowns, one sentence
>
> 2. Compose the HTML in the v0.3 section order per deep-dive-template.md:
>    - **Above the fold** (always visible): hero → key metrics strip → TL;DR panel (compact three cases hash-shuffled + three callouts for insight/kill-switch/unknown)
>    - **Below the fold** (native <details>/<summary>, no JavaScript): Three Cases (full, open by default) → Kill Switches → Fundamentals → Sentiment → Peers → Risk → Assumption Ledger → Unknowns → Sources → Adversarial Pass
>    - **Always visible at bottom**: Disclaimer footer, NOT inside a <details>
>
> 3. For each <details> section, produce a one-line abstract in the <summary> so readers can decide what to expand. Use the abstract formulas from Step 6.5 of your agent instructions.
>
> 4. The compact TL;DR cases and the full Three Cases details section MUST use the SAME hash-shuffled order (FNV-1a hash of ticker mod 6).
>
> Before rendering each below-the-fold section, check whether the corresponding analysis file exists and is non-empty. If any file is missing or is a failure stub, render a thin fallback from raw/ files directly per the deep-dive-template graceful-degradation rule. Record per-section fidelity (full / thin / placeholder) in your return summary.
>
> Run the compliance pass per skills/report-generation/references/compliance-rules.md before writing. Exemptions: "sell-side", "buy-side", and any text inside `stockwiz-disclaimer` class are NOT rewritten. Analyst distribution labels ("Strong Buy" etc), Yahoo `recommendationKey` strings, Zacks Rank text labels, SA Quant Rating labels, SWS risks list bullets, Reddit post titles, and Google News headlines MUST be wrapped in `<q>` tags when reproduced verbatim.
>
> Return a compact summary with file path, byte count, curated TL;DR atoms (the three atoms you picked), sections present (full/thin/placeholder per below-the-fold section), compliance rewrites applied, stripped sentences, and sanity check results.

Wait for the subagent to return.

**If report-writer returned an error** (compliance failed, sanity check failed, write failed):
1. Append a stage entry with `status: "failed"` and the error reason.
2. Retry **once** with a refined prompt noting the specific issue.
3. If still failing, jump to Step 13 (fatal error).

**On success:**
1. Append a stage entry for report-writer with the adjustments it reported.
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
   - Source coverage ("3 of 3 sources succeeded" or "2 of 3, Zacks failed")
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
4. **Ticker not found at SEC** (Step 6) — deep-researcher's SEC stage looked up the ticker in `company_tickers.json` and found no match. Reason `ticker-not-found-at-sec`. This is fatal because SEC is the ground-truth source; without it, no thesis.
5. **SEC EDGAR unreachable on first run** (Step 6) — the one-time `company_tickers.json` fetch failed (403 UA issue, DNS, network error). Reason `sec-unreachable`. Fatal.
6. **SEC EDGAR submissions API failed** (Step 6) — ticker resolved but the submissions or companyconcept APIs returned 403/503/timeout even after one retry. Reason `sec-submissions-failed`. Fatal.
7. **Non-US filer (20-F only) with zero recoverable structured data** (Step 6) — the filer has only 20-F filings AND no XBRL concept data could be extracted. Reason `non-us-filer-no-data`. Fatal.
8. **Fewer than 3 sources succeeded in total** (Step 6) — deep-researcher returned but sources['ok'].count < 3. Reason `insufficient-sources`. Fatal because no thesis can be built from 2 or fewer vendor snapshots.
9. **curl not installed** (deep-researcher Step 2.5) — hard dependency missing. Reason `curl-not-installed`. Fatal.
10. **JSON parser missing** (deep-researcher Step 2.5) — neither `jq` nor `python3` available for SEC JSON parsing. Reason `json-parser-missing`. Fatal.
11. **report-writer failed twice** (Step 11) — two consecutive attempts at HTML composition returned errors. Reason carries the underlying error. Fatal.
12. **Compliance pass unresolvable** (Step 11) — report-writer's compliance pass ran 3 iterations and still had banned phrases outside `<q>` tags. Reason `compliance-stuck`. Fatal — we refuse to emit a non-compliant report.

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
2. Do NOT emit `report.html`. If a partial `report.html` exists from a failed report-writer attempt, delete it.
3. Print to the user:
   > stockwiz deep-dive on `<TICKER>` failed.
   > Stage: `<failed-stage>`
   > Reason: `<reason code>` — `<human-readable explanation>`
   > Session workspace saved at `<SESSION_DIR>` for inspection.
   > `<one or two sentences of diagnostic guidance from the table above>`
4. Return.

### Non-fatal failures (for reference — these do NOT jump to Step 13)

The following are explicitly non-fatal and must not call Step 13:

- Any individual non-SEC source failing (Zacks Cloudflare, Yahoo rate limit, Reddit 429, SWS paywall, SA SSR gap) — deep-researcher marks the source `failed` and continues.
- `lynx` not installed — deep-researcher falls back to raw-HTML Read for lynx-dependent sources.
- Any individual analysis agent writing a stub (missing raw inputs) — downstream agents and report-writer handle stubs via thin fallbacks.
- devils-advocate subagent failing — Stage 5 (reconcile) is skipped, report-writer renders the Adversarial Appendix as a placeholder.
- Cookie jar leaks in `/tmp/stockwiz-*` — cleanup bug, not a fatal condition.

## Edge cases

- **Ticker that doesn't exist.** SEC EDGAR (via curl + `company_tickers.json`) will not find the ticker in the mapping file. deep-researcher marks the source failed with reason `ticker-not-found-at-sec`. This is fatal — jump to Step 13 with message: "No SEC filings found for <TICKER>. Verify it is a US-listed equity."

- **Non-US filer (20-F only).** We try to extract whatever structured XBRL data is available from SEC (many FPIs file some facts) and flag the source with a note. The orchestrator proceeds if any data was recovered, else treats it as a fatal SEC failure.

- **lynx not installed.** Stockanalysis, SWS, and SA reference files all prefer lynx for HTML→text conversion. If lynx is missing, deep-researcher falls back to Read-with-limit on the raw HTML file. This is slower and less clean but still functional. The deep-researcher's Step 2.5 prerequisite check notes the missing tool; the agent's summary should mention it so the user knows why certain extractions are thinner than they could be.

- **curl not installed.** Hard dependency. deep-researcher's Step 2.5 aborts with `curl-not-installed`. User needs to install curl (should be present on every macOS and Linux system by default).

- **SEC company_tickers.json fetch fails on first run.** The cache doesn't exist yet. If the fetch fails with a 403 (UA issue) or network error, treat as fatal SEC failure and jump to Step 13 — no CIK lookup means no SEC at all.

- **Analysis agent failure.** Any of the four analysis agents may write a stub file (or no file) if its raw inputs are all failed. The thesis-discipline agent's `full` mode handles this via its fallback mode (reads raw files directly). The report-writer agent renders a thin fallback for that section. Non-fatal at every stage.

- **Devils-advocate failure.** If the Task call times out or the subagent returns an error, log it and skip the reconcile stage (Stage 5). The report-writer will render a placeholder for the Adversarial Appendix section noting the pass was skipped. Non-fatal.

- **Cookie jar leaks.** Yahoo's fetch creates a cookie jar tempfile. It should be deleted after Yahoo finishes (success or failure). If you notice `/tmp/stockwiz-yahoo-cookies.*` files accumulating over many runs, that's a cleanup bug worth fixing.

- **Same-ticker same-second.** The timestamp includes seconds, so collisions are extremely unlikely. If one does occur (mkdir fails with EEXIST), append a short suffix and retry once.

- **Disk full / permission error during session dir creation.** Surface the OS error, abort, do not attempt to continue.

- **User interrupts mid-run.** Claude Code will handle this — the session dir remains on disk with `status: "started"` and whatever partial files were written. The user can manually inspect the directory, or re-run `/stockwiz <TICKER>` to start fresh. A future `/stockwiz-revisit` command on the roadmap would detect interrupted sessions and offer to resume them.

## What success looks like

The user runs `/stockwiz NVDA`. They see a todo list with seven stages. Stage 1 gathers 6–12 sources via the deep-researcher agent in ~20 minutes. Stage 2 dispatches **four analysis agents concurrently** in a single message — fundamental extracts multi-year numbers and builds an assumption ledger, sentiment weighs SWS risks vs analyst distribution and computes the publisher-distribution shape, peer-comparison builds a comp table from SWS competitor snowflakes, risk-screen enumerates beta/drawdown/cycle/concentration/tail — and the orchestrator waits for all four to return (~10 minutes, max of the four, not the sum). Stage 3 dispatches the thesis-discipline agent in `full` mode to synthesize the thesis from the four analysis files (falls back to raw files if any are missing). Stage 4 spawns the red-colored devils-advocate agent in an isolated context; it reads only thesis.md, ranks the weakest claims, builds a coherent opposing narrative, and rewrites weak kill switches. Stage 5 dispatches the thesis-discipline agent again in `reconcile` mode to append an "Adjustments After Stress Test" section to thesis.md while preserving the original claims verbatim. Stage 6 runs the report-writer agent which curates three TL;DR atoms (key insight, closest kill switch, biggest unknown) and composes a 60–150KB insights-first HTML report with collapsible `<details>` sections for the full analyses and adversarial appendix. Stage 7 finalizes meta.json. Total wall-clock ~45 minutes (down from ~78 minutes in v0.3.x) thanks to the Stage 2 parallelization and the isolated-context speedups in Stages 3 and 5.

The user opens `report.html` and sees a clean research brief with three equal-weight columns, a full fundamentals section with multi-year sparklines, an insider-activity grid, a peer comp table, a drawdown profile, an assumption ledger you can interrogate row-by-row, a full-width kill-switches row calibrated with margin-of-safety numbers, a red-highlighted adversarial appendix with ranked weakest claims and kill-switch rewrites, and a disclaimer footer that can't be removed. They come back to the chat and ask "what did devils-advocate say about the margin thesis?" — the main context Greps `analysis/devils-advocate.md` and answers with citation, no re-fetch.

That's the current bar.
