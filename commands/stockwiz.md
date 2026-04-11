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
  "commandVersion": "0.1.1",
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
2. Synthesize thesis (thesis-discipline skill)
3. Adversarial stress test (devils-advocate — Phase 2+)
4. Generate HTML report (report-writer)
5. Finalize session metadata
```

Phase 1.5 note: the thesis-discipline skill runs, but analysis skills (fundamental-analysis, sentiment-synthesis, peer-comparison, risk-screen) are Phase 2. In Phase 1.5, the thesis is built directly from the 6 raw source files with placeholder analysis sections in the HTML.

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
> This is a Phase 1.5 run. Fetch the 6 Phase 1.5 sources per the source-extraction skill, in order: SEC EDGAR (curl + JSON APIs), Finviz (WebFetch), Stockanalysis (curl + lynx), Yahoo Finance (curl + quoteSummary JSON API), Simply Wall Street (WebSearch + curl + lynx), Seeking Alpha (curl + lynx). Each source's reference file prescribes the exact tool and URLs — use those. Do not substitute WebFetch for curl on gated sources (SEC, Yahoo) — that's why Phase 1 failed. Follow the source-extraction/SKILL.md rate limits and failure detection rules. Write each fetched source to SESSION_DIR/raw/ per the file output convention and update SESSION_DIR/meta.json.sources after each fetch.
>
> SEC EDGAR is the only fatal source — if it fails, stop fetching and set meta.json.status to "failed". All other source failures are non-fatal.
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
4. **If fewer than 3 sources succeeded in total**, also jump to Step 11 — there's not enough data to build a credible Phase 1.5 thesis even with the non-fatal sources intact.
5. Otherwise, mark stage 1 complete in TodoWrite and proceed. It is fine (and expected) that some of SWS, SA, Yahoo may have failed — the thesis can still be built from the remaining sources.

## Step 7 — Stage 2: Synthesize thesis with thesis-discipline

In Phase 1.5, we skip the four analysis skills (fundamental, sentiment, peer, risk — those are Phase 2) and go directly from raw data to the thesis. The thesis-discipline skill will read raw files directly and produce the best thesis it can with the available data.

Load the `thesis-discipline` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/thesis-discipline/SKILL.md`) and follow its `full` mode instructions. In Phase 1.5, the inputs are whichever of these succeeded:

- `$SESSION_DIR/raw/sec-edgar-10k.md` (always present — SEC failure is fatal, so if we got here it's `ok`)
- `$SESSION_DIR/raw/finviz-snapshot.md`
- `$SESSION_DIR/raw/stockanalysis.md`
- `$SESSION_DIR/raw/yahoo-fundamentals.md`
- `$SESSION_DIR/raw/simply-wall-street.md`
- `$SESSION_DIR/raw/seeking-alpha.md`

Check each file's frontmatter for `status: ok` before including it in the thesis inputs. Skip files with `status: failed`.

For Phase 1.5, the SWS risks list and SA factor grades are particularly valuable — they're where "alternative view" signal comes from. The thesis-discipline skill should incorporate them into the bear case disconfirmers and the sentiment-related claims in the bull/base cases, respectively.

Produce `$SESSION_DIR/thesis.md` with the mandatory sections from the skill:

```markdown
# <TICKER> — <Company Name> — Investment Thesis

## Bull Case

## Base Case

## Bear Case

## Explicit Disconfirmers

## Kill Switches

## Unknowns
```

Key rules you must enforce:

- Every claim has a citation to a `raw/` file
- Base case is its own scenario, not a midpoint
- Kill switches are measurable (specific threshold + time window + source)
- Unknowns is first-class — in Phase 1 with only 3 sources, there will be many unknowns; that's fine and expected

Append a stage entry for thesis synthesis. Mark stage 2 complete.

## Step 8 — Stage 3: Devils-advocate pass (Phase 2+)

**Phase 1: skip this stage.** The devils-advocate subagent doesn't exist until Phase 2. Write a stage entry noting it was skipped:

```json
{ "stage": "devils-advocate", "status": "skipped", "reason": "phase-2-feature" }
```

Mark stage 3 as completed in the todo list with a note: "Adversarial pass (Phase 2) — skipped in walking skeleton".

**Phase 2+ behavior (for future reference, not to implement now):** delegate to the `devils-advocate` subagent via Task, passing only the path to `thesis.md`. It returns after writing `analysis/devils-advocate.md`. If material issues were raised, append a `## Adjustments After Stress Test` section to `thesis.md` (append-only, don't overwrite claims).

## Step 9 — Stage 4: Generate HTML report

Use the `Task` tool to invoke the `report-writer` subagent. Pass:

> Generate a stockwiz HTML artifact for session <SESSION_DIR>.
>
> SESSION_DIR: <absolute path>
> TEMPLATE: deep-dive
>
> This is a Phase 1.5 run. The session has raw/ files for up to 6 sources (SEC EDGAR, Finviz, Stockanalysis, Yahoo Finance, Simply Wall Street, Seeking Alpha — check each file's frontmatter for status: ok before using it) and a synthesized thesis.md. No analysis/ files yet (Phase 2) and no devils-advocate appendix (Phase 2). Follow the deep-dive template reference at skills/report-generation/references/deep-dive-template.md, using the Phase 1.5 section set: hero, snapshot, three cases, kill switches, Phase 2 placeholders for fundamentals/sentiment/peers/risk/assumption-ledger, unknowns, sources, disclaimer.
>
> The snapshot strip cards should draw from whichever sources succeeded — prefer Finviz for most fields, Yahoo quoteSummary JSON for enterprise value and forward P/E, SEC for the big balance-sheet numbers. The sources section at the bottom must list all successful sources with their fetched_at timestamps.
>
> Pay special attention to SWS risks and SA factor grades if those sources succeeded — they're the Phase 1.5 signal uplift over Phase 1. SWS risks belong in the bear case or the Unknowns section; SA factor grades belong in the sentiment placeholder with a neutral-language framing.
>
> Run the compliance pass per skills/report-generation/references/compliance-rules.md before writing. Abort with error if the pass fails after 3 iterations. Remember that SA's "Strong Buy" label and Yahoo's `recommendationKey` string must be wrapped in <q> tags.
>
> Return a compact summary with file path, byte count, sections present, compliance rewrites applied, and sanity check results.

Wait for the subagent to return.

**If report-writer returned an error** (compliance failed, sanity check failed, write failed):
1. Append a stage entry with `status: "failed"` and the error reason.
2. Retry **once** with a refined prompt noting the specific issue.
3. If still failing, jump to Step 11 (fatal error).

**On success:**
1. Append a stage entry for report-writer with the adjustments it reported.
2. Update `meta.json.reportPath = "report.html"`.
3. Mark stage 4 complete.

## Step 10 — Finalize

Update `$SESSION_DIR/meta.json`:

```json
{
  ...
  "status": "complete",
  "completedAt": "<ISO 8601>",
  "reportPath": "report.html"
}
```

Mark stage 5 complete in TodoWrite.

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
   - Source coverage ("3 of 3 Phase 1 sources succeeded" or "2 of 3, Zacks failed")
   - Path to the HTML report (not opened automatically — tell the user to open it)

3. **A one-line next-step hint**:
   > "Ask a follow-up question to dig into any section — I'll read from this session workspace without re-fetching. Or run `/stockwiz-thesis`, `/stockwiz-compare`, etc. (Phase 3 commands coming soon)."

Do NOT auto-open the HTML file. The user should choose when to view it.

## Step 11 — Fatal error handling

If any fatal error occurs (deep-researcher returned zero successes, SEC EDGAR unreachable, report-writer failed twice, compliance pass unresolvable):

1. Update `$SESSION_DIR/meta.json`:
   ```json
   {
     ...
     "status": "failed",
     "failedAt": "<ISO 8601>",
     "error": "<brief description>"
   }
   ```
2. Do NOT emit `report.html`.
3. Print to the user:
   > stockwiz deep-dive on <TICKER> failed: <reason>.
   > Session workspace saved at <SESSION_DIR> for inspection.
   > Common causes: source rate-limiting, SEC EDGAR unreachable, compliance pass stuck.
4. Return.

## Edge cases

- **Ticker that doesn't exist.** SEC EDGAR (via curl + `company_tickers.json`) will not find the ticker in the mapping file. deep-researcher marks the source failed with reason `ticker-not-found-at-sec`. This is fatal — jump to Step 11 with message: "No SEC filings found for <TICKER>. Verify it is a US-listed equity."

- **Non-US filer (20-F only).** In Phase 1.5 we try to extract whatever structured data is available from SEC (many FPIs file some XBRL facts) and flag the source with a note. The orchestrator proceeds if any data was recovered, else treats it as a fatal SEC failure.

- **lynx not installed.** Stockanalysis, SWS, and SA reference files all prefer lynx for HTML→text conversion. If lynx is missing, deep-researcher falls back to Read-with-limit on the raw HTML file. This is slower and less clean but still functional. The deep-researcher's Step 2.5 prerequisite check notes the missing tool; the agent's summary should mention it so the user knows why certain extractions are thinner than they could be.

- **curl not installed.** Hard dependency for Phase 1.5. deep-researcher's Step 2.5 aborts with `curl-not-installed`. User needs to install curl (should be present on every macOS and Linux system by default).

- **SEC company_tickers.json fetch fails on first run.** The cache doesn't exist yet. If the fetch fails with a 403 (UA issue) or network error, treat as fatal SEC failure and jump to Step 11 — no CIK lookup means no SEC at all.

- **Cookie jar leaks.** Yahoo's fetch creates a cookie jar tempfile. It should be deleted after Yahoo finishes (success or failure). If you notice `/tmp/stockwiz-yahoo-cookies.*` files accumulating over many runs, that's a cleanup bug worth fixing.

- **Same-ticker same-second.** The timestamp includes seconds, so collisions are extremely unlikely. If one does occur (mkdir fails with EEXIST), append a short suffix and retry once.

- **Disk full / permission error during session dir creation.** Surface the OS error, abort, do not attempt to continue.

- **User interrupts mid-run.** Claude Code will handle this — the session dir remains on disk with `status: "started"` and whatever partial files were written. A subsequent `/stockwiz-revisit <TICKER>` (Phase 4) would be able to detect and warn.

## What success looks like

The user runs `/stockwiz NVDA`. They see a todo list updating as stages complete. They see a short progress message from deep-researcher ("SEC EDGAR ok via curl + JSON APIs; Finviz ok; Stockanalysis ok; Yahoo JSON API ok; SWS snowflake + 5 risks ok; Seeking Alpha quant + factor grades ok — 6 of 6"). They see a thesis synthesis note drawing on numeric data from SEC and Stockanalysis, on sentiment from SWS risks and SA factor grades, and on quick-look ratios from Finviz. They see a report-writer completion message noting the compliance pass rewrote two source labels into `<q>` tags. They get a final summary with three case headlines, a kill switch or two, and a pointer to the HTML file. They open `~/.claude/stockwiz/sessions/NVDA-*/report.html` in their browser, see a clean, disclaimer-footed research brief with three equal-weight columns, 6 sources listed in the bottom section, and SWS's risk list fleshing out the bear case. They come back to the chat and ask "what's the insider ownership?" — the main context uses Grep + Read to pull the answer from `raw/finviz-snapshot.md` with a citation, no re-fetch.

That's the Phase 1.5 bar.
