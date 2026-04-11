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
  "commandVersion": "0.3.2",
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
1. Gather data from public sources (deep-researcher)
2. Run four analysis skills (fundamental, sentiment, peer-comp, risk)
3. Synthesize thesis (thesis-discipline full mode)
4. Adversarial stress test (devils-advocate)
5. Reconcile thesis with adversarial pass (thesis-discipline reconcile mode)
6. Generate HTML report (report-writer)
7. Finalize session metadata
```

all four analysis skills run sequentially in the main context before thesis synthesis. The devils-advocate subagent runs in an isolated Task context after thesis synthesis. The thesis-discipline skill is invoked twice — once in `full` mode to synthesize, once in `reconcile` mode to merge devils-advocate feedback.

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
> This is a deep-dive run. Fetch the 10 current sources per the source-extraction skill, in this order:
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
>
> Each source's reference file under skills/source-extraction/references/ prescribes the exact tool, URLs, extraction targets, and failure mode detection — follow them. Do NOT substitute WebFetch for curl on gated sources (SEC, Yahoo, Stockanalysis, Macrotrends, Google, SWS, SA, Zacks, Reddit). Finviz is the only WebFetch source.
>
> SEC EDGAR is the only fatal source. If SEC fails, stop fetching and set meta.json.status to "failed". All other source failures (Zacks Cloudflare, Yahoo rate limit, Reddit 429, etc.) are non-fatal — continue to the next source. Zacks and Reddit are explicitly best-effort; do not retry challenges.
>
> Write each fetched source to SESSION_DIR/raw/ per the file output convention and update SESSION_DIR/meta.json.sources after each fetch. Use the slug conventions from source-extraction/SKILL.md (sec-edgar-10k.md, finviz-snapshot.md, stockanalysis.md, macrotrends.md, yahoo-fundamentals.md, google-finance.md, simply-wall-street.md, seeking-alpha.md, zacks-snapshot.md, reddit.md).
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

## Step 7 — Stage 2: Run four analysis skills

Load each analysis skill in sequence and invoke it per its SKILL.md instructions. Each skill reads specific files from `SESSION_DIR/raw/` and writes its output to `SESSION_DIR/analysis/<name>.md`. They run sequentially because they're skills in the main context (not subagents) — parallelism would require wrapping each in a Task.

Before starting each analysis, check which raw sources succeeded by reading `meta.json.sources`. Skip any file with `status: failed`.

### Stage 2a — fundamental-analysis

Load `${CLAUDE_PLUGIN_ROOT}/skills/fundamental-analysis/SKILL.md`. Follow its instructions to read from:
- `raw/sec-edgar-10k.md` (XBRL facts)
- `raw/stockanalysis.md` (5Y history + statistics)
- `raw/finviz-snapshot.md` (current multiples)
- `raw/yahoo-fundamentals.md` (cross-check, if status ok)

Write `analysis/fundamental.md` per the skill's output structure (Snapshot / Valuation Framing / Quality / Growth / Capital Structure / Ownership / Assumption Ledger / Unknowns).

Append a stage entry to `meta.json.stages`.

### Stage 2b — sentiment-synthesis

Load `${CLAUDE_PLUGIN_ROOT}/skills/sentiment-synthesis/SKILL.md`. Follow its instructions to read from:
- `raw/simply-wall-street.md` (risks list, rewards, narrative — primary signal)
- `raw/seeking-alpha.md` (factor grades if present; article teasers)
- `raw/finviz-snapshot.md` (insider txns, short, analyst recom)
- `raw/sec-edgar-10k.md` (recent 8-K for material events)
- `raw/yahoo-fundamentals.md` (recommendationKey if present)

Write `analysis/sentiment.md` per the skill's output structure. Preserve source labels verbatim for later `<q>` wrapping in the report.

Append a stage entry.

### Stage 2c — peer-comparison

Load `${CLAUDE_PLUGIN_ROOT}/skills/peer-comparison/SKILL.md`. Follow its instructions. peers come from `raw/simply-wall-street.md` competitor snowflakes (free context — SWS typically includes 2–3 peers with their own scores).

Write `analysis/peer-comp.md`. If no peer data is available (SWS failed or didn't provide competitors), write a minimal file noting the limitation and move on.

Append a stage entry.

### Stage 2d — risk-screen

Load `${CLAUDE_PLUGIN_ROOT}/skills/risk-screen/SKILL.md`. Follow its instructions to read from:
- `raw/finviz-snapshot.md` (beta, ATR, volatility, 52w range, short)
- `raw/stockanalysis.md` (5Y beta, interest coverage, debt ratios)
- `raw/sec-edgar-10k.md` (debt position)
- `raw/simply-wall-street.md` (SWS risks as tail risks)

Write `analysis/risk.md` per the skill's output structure.

Append a stage entry. Mark stage 2 complete in TodoWrite.

**If any analysis skill fails** (e.g. its primary raw inputs are all `status: failed`), write a minimal stub `analysis/<name>.md` noting the failure and continue. Downstream stages will render thin fallbacks. A single analysis skill failure is not fatal.

## Step 8 — Stage 3: Synthesize thesis with thesis-discipline (full mode)

Load the `thesis-discipline` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/thesis-discipline/SKILL.md`) and follow its `full` mode instructions.

**Primary inputs** (preferred — from Stage 2 outputs):
- `$SESSION_DIR/analysis/fundamental.md`
- `$SESSION_DIR/analysis/sentiment.md`
- `$SESSION_DIR/analysis/peer-comp.md`
- `$SESSION_DIR/analysis/risk.md`

Check each file for non-empty content (not just file existence — the analysis skills may have written stub files on failure). For any file that is missing or is a failure stub, fall back to reading the corresponding raw files directly per the skill's fallback mode.

Write `$SESSION_DIR/thesis.md` with the mandatory sections from the skill:

```
## Bull Case
## Base Case
## Bear Case
## Explicit Disconfirmers
## Kill Switches
## Unknowns
```

Key rules the skill enforces:

- Every claim has a citation to an `analysis/` or `raw/` file
- Base case is its own scenario, not a midpoint
- Kill switches are measurable (specific threshold + time window + source)
- Unknowns is first-class — when analyses flagged limitations, those become Unknowns here

Append a stage entry. Mark stage 3 complete.

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

Load the `thesis-discipline` skill again, this time in `reconcile` mode (per the skill's SKILL.md). Pass it both:

- `SESSION_DIR/thesis.md` (the existing thesis from Stage 3)
- `SESSION_DIR/analysis/devils-advocate.md` (the adversarial pass from Stage 4)

The skill will append a `## Adjustments After Stress Test` section to `thesis.md` if devils-advocate raised material issues. The original bull/base/bear claims are preserved verbatim — adjustments live in their own section as an audit trail of what the stress test changed.

**If devils-advocate was skipped or failed in Stage 4**, skip this reconcile step. thesis.md remains as Stage 3 wrote it.

Append a stage entry:
```json
{
  "stage": "thesis-reconcile",
  "status": "ok",
  "adjustmentsApplied": N,
  "killSwitchesTightened": M
}
```

Mark stage 5 complete.

## Step 11 — Stage 6: Generate HTML report

Use the `Task` tool to invoke the `report-writer` subagent. Pass:

> Generate a stockwiz HTML artifact for session <SESSION_DIR>.
>
> SESSION_DIR: <absolute path>
> TEMPLATE: deep-dive (v0.3 insights-first)
>
> This is a deep-dive run. The session has raw/ files for up to 10 sources (SEC EDGAR, Finviz, Stockanalysis, Macrotrends, Yahoo, Google Finance + Google News RSS, Simply Wall Street, Seeking Alpha, Zacks, Reddit), analysis/ files (fundamental, sentiment, peer-comp, risk, devils-advocate) from the four analysis skills and devils-advocate subagent, and a thesis.md (possibly with an Adjustments After Stress Test section from the reconcile step).
>
> **Use the v0.3 insights-first template.** This means:
>
> 1. **First, run Step 5 of your SKILL.md — curate the TL;DR atoms** BEFORE composing HTML:
>    - Key insight: ONE most striking observation picked from analysis/fundamental.md and analysis/sentiment.md using the heuristic list in deep-dive-template.md (anomalous quality metric / step-change growth / capital-structure surprise / margin inflection / historical outlier / SWS extreme)
>    - Closest kill switch: the kill switch from thesis.md with the smallest margin of safety, with current/trigger/margin stats
>    - Biggest unknown: the most material item from thesis.md § Unknowns, one sentence
>
> 2. Compose the HTML in the v0.3 section order per deep-dive-template.md:
>    - **Above the fold** (always visible): hero → key metrics strip → TL;DR panel (compact three cases hash-shuffled + three callouts for insight/kill-switch/unknown)
>    - **Below the fold** (native <details>/<summary>, no JavaScript): Three Cases (full, open by default) → Kill Switches → Fundamentals → Sentiment → Peers → Risk → Assumption Ledger → Unknowns → Sources → Adversarial Pass
>    - **Always visible at bottom**: Disclaimer footer, NOT inside a <details>
>
> 3. For each <details> section, produce a one-line abstract in the <summary> so readers can decide what to expand. Use the abstract formulas from Step 6.5 of your SKILL.md.
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
- Any individual analysis skill writing a stub (missing raw inputs) — downstream skills and report-writer handle stubs via thin fallbacks.
- devils-advocate subagent failing — Stage 5 (reconcile) is skipped, report-writer renders the Adversarial Appendix as a placeholder.
- Cookie jar leaks in `/tmp/stockwiz-*` — cleanup bug, not a fatal condition.

## Edge cases

- **Ticker that doesn't exist.** SEC EDGAR (via curl + `company_tickers.json`) will not find the ticker in the mapping file. deep-researcher marks the source failed with reason `ticker-not-found-at-sec`. This is fatal — jump to Step 13 with message: "No SEC filings found for <TICKER>. Verify it is a US-listed equity."

- **Non-US filer (20-F only).** We try to extract whatever structured XBRL data is available from SEC (many FPIs file some facts) and flag the source with a note. The orchestrator proceeds if any data was recovered, else treats it as a fatal SEC failure.

- **lynx not installed.** Stockanalysis, SWS, and SA reference files all prefer lynx for HTML→text conversion. If lynx is missing, deep-researcher falls back to Read-with-limit on the raw HTML file. This is slower and less clean but still functional. The deep-researcher's Step 2.5 prerequisite check notes the missing tool; the agent's summary should mention it so the user knows why certain extractions are thinner than they could be.

- **curl not installed.** Hard dependency. deep-researcher's Step 2.5 aborts with `curl-not-installed`. User needs to install curl (should be present on every macOS and Linux system by default).

- **SEC company_tickers.json fetch fails on first run.** The cache doesn't exist yet. If the fetch fails with a 403 (UA issue) or network error, treat as fatal SEC failure and jump to Step 13 — no CIK lookup means no SEC at all.

- **Analysis skill failure.** Any of the four analysis skills may write a stub file if its raw inputs are all failed. The thesis-discipline `full` mode handles this via its fallback mode (reads raw files directly). The report-writer renders a thin fallback for that section. Non-fatal at every stage.

- **Devils-advocate failure.** If the Task call times out or the subagent returns an error, log it and skip the reconcile stage (Stage 5). The report-writer will render a placeholder for the Adversarial Appendix section noting the pass was skipped. Non-fatal.

- **Cookie jar leaks.** Yahoo's fetch creates a cookie jar tempfile. It should be deleted after Yahoo finishes (success or failure). If you notice `/tmp/stockwiz-yahoo-cookies.*` files accumulating over many runs, that's a cleanup bug worth fixing.

- **Same-ticker same-second.** The timestamp includes seconds, so collisions are extremely unlikely. If one does occur (mkdir fails with EEXIST), append a short suffix and retry once.

- **Disk full / permission error during session dir creation.** Surface the OS error, abort, do not attempt to continue.

- **User interrupts mid-run.** Claude Code will handle this — the session dir remains on disk with `status: "started"` and whatever partial files were written. The user can manually inspect the directory, or re-run `/stockwiz <TICKER>` to start fresh. A future `/stockwiz-revisit` command on the roadmap would detect interrupted sessions and offer to resume them.

## What success looks like

The user runs `/stockwiz NVDA`. They see a todo list with seven stages. Stage 1 gathers 5–6 sources via deep-researcher. Stage 2 runs four analysis skills sequentially — fundamental extracts multi-year numbers and builds an assumption ledger, sentiment weighs SWS risks vs analyst distribution, peer-comp builds a comp table from SWS competitor snowflakes, risk-screen enumerates beta/drawdown/concentration/tail. Stage 3 synthesizes the thesis reading from the four analyses (not raw files directly). Stage 4 spawns a red-colored devils-advocate subagent in an isolated context; it reads only thesis.md, ranks the weakest claims, builds a coherent opposing narrative, and rewrites weak kill switches. Stage 5 appends an "Adjustments After Stress Test" section to thesis.md preserving the original claims verbatim. Stage 6 generates a 60–120KB HTML report with full Fundamentals, Sentiment, Peers, Risk, Assumption Ledger, and Adversarial Appendix sections. Stage 7 finalizes meta.json.

The user opens `report.html` and sees a clean research brief with three equal-weight columns, a full fundamentals section with multi-year sparklines, an insider-activity grid, a peer comp table, a drawdown profile, an assumption ledger you can interrogate row-by-row, a full-width kill-switches row calibrated with margin-of-safety numbers, a red-highlighted adversarial appendix with ranked weakest claims and kill-switch rewrites, and a disclaimer footer that can't be removed. They come back to the chat and ask "what did devils-advocate say about the margin thesis?" — the main context Greps `analysis/devils-advocate.md` and answers with citation, no re-fetch.

That's the current bar.
