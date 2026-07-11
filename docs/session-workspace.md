# Session workspace — filesystem contract and meta.json schema

This is the canonical reference for **where stockwiz stores things** and **what the state files look like**. Every skill, agent, and command in the repo is expected to honor the conventions documented here. If you are writing a new skill or agent, read this before touching a file path.

## Filesystem layout

stockwiz writes to two root directories: `~/Projects/stockwiz/` (the plugin source) and `~/.claude/stockwiz/` (the runtime data). The plugin source is read-only at runtime — every path that is written during a session is under `~/.claude/stockwiz/`.

```
~/.claude/stockwiz/
├── config.json                              # user config: FRED key, rate-limit, defaults
├── cache/
│   ├── company_tickers.json                 # SEC ticker→CIK map (refreshed every 30 days)
│   └── macrotrends-slugs.json               # ticker→company-slug map (self-populating)
└── sessions/
    └── <TICKER>-<ISO8601>/                  # one directory per /stockwiz run
        ├── meta.json                        # session state, schema below
        ├── thesis.md                        # synthesized bull/base/bear thesis
        ├── report.html                      # self-contained HTML artifact
        ├── raw/                             # librarian outputs — never edited after write
        │   ├── sec-edgar-10k.md             # structured XBRL facts
        │   ├── sec-submissions.json         # SEC submissions API raw (audit trail)
        │   ├── sec-concept-Revenues.json    # per-concept XBRL raw (audit trail)
        │   ├── sec-concept-NetIncomeLoss.json
        │   ├── ...                          # (other SEC concept JSON files)
        │   ├── yahoo-fundamentals.md        # parsed Yahoo quoteSummary
        │   ├── yahoo-quotesummary.json      # Yahoo API raw (audit trail)
        │   ├── finviz-snapshot.md           # Finviz quote page extract
        │   ├── stockanalysis.md             # 5Y financials + statistics
        │   ├── macrotrends.md               # 10-20Y deep history
        │   ├── google-finance.md            # Google Finance SSR + Google News RSS
        │   ├── google-news-feed.xml         # Google News RSS raw (audit trail)
        │   ├── simply-wall-street.md        # SWS snowflake + risks + rewards
        │   ├── seeking-alpha.md             # SA factor grades + article teasers
        │   ├── zacks-snapshot.md            # Zacks Rank + Style Scores (best-effort)
        │   ├── reddit.md                    # retail sentiment across 3 subs
        │   ├── reddit-stocks.json           # per-sub JSON raw (audit trail)
        │   ├── reddit-wallstreetbets.json
        │   ├── reddit-investing.json
        │   └── _sanity.md                   # optional, cross-source disagreements
        └── analysis/                        # analyst outputs — structured markdown
            ├── fundamental.md               # fundamental-analysis agent output
            ├── sentiment.md                 # sentiment-synthesis agent output
            ├── peer-comp.md                 # peer-comparison agent output
            ├── risk.md                      # risk-screen agent output
            └── devils-advocate.md           # devils-advocate agent output
```

### Also used (scratch, not persistent)

```
/tmp/stockwiz-yahoo-cookies.XXXXXX            # Yahoo consent cookie jar (cleaned after Yahoo fetch)
/tmp/stockwiz-<source>-<TICKER>.{html,txt,xml,json}  # curl download intermediate (cleaned after parse)
```

These are cleaned up by the fetching source's reference file at end-of-source. If a run crashes mid-way, leftover files are harmless but should be swept periodically.

## File lifecycle

| Path | Created by | Read by | Mutated after creation? | Persists? |
|---|---|---|---|---|
| `config.json` | `/stockwiz-setup` | all commands | yes (user updates, setup re-runs) | yes |
| `cache/company_tickers.json` | `/stockwiz` pre-flight (Step 4.5) | `/stockwiz` pre-flight (CIK lookup) | refreshed every 30 days | yes |
| `cache/macrotrends-slugs.json` | `deep-researcher` shard B (first Macrotrends fetch on new ticker) | `deep-researcher` shard B | append-only | yes |
| `sessions/<T>-<t>/meta.json` | `/stockwiz` Step 4 | every stage | **yes — orchestrator only, between stages** | yes |
| `sessions/<T>-<t>/raw/*.md` | `deep-researcher` fetch shards (`_sanity.md`: orchestrator) | analysis agents, thesis-discipline, report-writer | **no — immutable audit trail** | yes |
| `sessions/<T>-<t>/raw/*.json` | `deep-researcher` fetch shards | debugging only | no | yes |
| `sessions/<T>-<t>/analysis/fundamental.md` | `fundamental-analysis` agent | `thesis-discipline`, `report-writer` | no | yes |
| `sessions/<T>-<t>/analysis/sentiment.md` | `sentiment-synthesis` agent | `thesis-discipline`, `report-writer` | no | yes |
| `sessions/<T>-<t>/analysis/peer-comp.md` | `peer-comparison` agent | `thesis-discipline`, `report-writer` | no | yes |
| `sessions/<T>-<t>/analysis/risk.md` | `risk-screen` agent | `thesis-discipline`, `report-writer` | no | yes |
| `sessions/<T>-<t>/analysis/devils-advocate.md` | `devils-advocate` agent | `thesis-discipline` reconcile, `report-writer` | no | yes |
| `sessions/<T>-<t>/thesis.md` | `thesis-discipline` full mode | `devils-advocate`, `thesis-discipline` reconcile, `report-writer` | **yes — reconcile appends `## Adjustments After Stress Test`, never overwrites existing sections** | yes |
| `sessions/<T>-<t>/report-draft.html` | `report-writer` agent | `report-writer` (splice + compliance) | yes, until promoted | no — promoted to `report.html` or deleted on abort |
| `sessions/<T>-<t>/report.html` | `report-writer` agent (promotion of clean draft) | end user | no (re-runs create new session dir) | yes |
| `/tmp/stockwiz-*` | individual sources | parsing | yes | no — cleaned after source completes |

## Multi-writer rules

### Immutable once written

`raw/*.md` files are **never modified** after they are first written by the `deep-researcher` fetch shards. They are the audit trail. Downstream stages read them; if they need derived information they write it to `analysis/`, not back to `raw/`. This is a hard rule — violations destroy auditability.

`analysis/*.md` files are written once by their owning agent and then only read by downstream stages. The owning agent is the sole writer of its file.

### Append-only mutation

`thesis.md` is written by the `thesis-discipline` agent in `full` mode, then the same agent in `reconcile` mode appends a `## Adjustments After Stress Test` section. The bull/base/bear case bodies are **never rewritten** — reconcile adds a new section that records the delta, preserving the original claims verbatim for audit.

### Read-modify-write — orchestrator only

`meta.json` is the single mutable state file, and **the orchestrator is its only writer**. Subagents — the four fetch shards, the four analysis agents, thesis-discipline, devils-advocate, report-writer — return compact summaries, and the orchestrator merges them into `meta.json` between stages. This is what makes the parallel dispatch at Stages 1 and 2 race-free: concurrent subagents write disjoint files and never touch shared state.

When the orchestrator updates meta.json, the convention is **always Read first, mutate, Write**, never Write without reading the current state:

```python
# idiomatic meta.json update (orchestrator, between stages)
import json
with open(meta_path) as f:
    m = json.load(f)
m['sources']['finviz'] = {'status': 'ok', 'file': 'raw/finviz-snapshot.md', 'fetched_at': ...}
m['stages'].append({'stage': 'fetch-shard-A', 'startedAt': ..., 'status': 'ok', 'parallel': True})
with open(meta_path, 'w') as f:
    json.dump(m, f, indent=2)
```

One special case: `raw/_sanity.md` is composed by the **orchestrator** (not a shard) after Stage 1, from the key facts the shards return — cross-source disagreements span shards, so no single shard can see them. Like every other `raw/` file, it is immutable once written.

## meta.json schema (authoritative)

Every session directory contains exactly one `meta.json` with this shape. This is the single source of truth; every writer must honor it.

```jsonc
{
  // Schema version for forward compatibility; bump when adding required fields.
  "schemaVersion": 1,

  // Identifying fields set at session creation and never modified.
  "ticker": "NVDA",                      // uppercase US equity symbol
  "mode": "full",                        // "full" (only mode currently built)
                                         // reserved: "thesis", "compare", "bear", "revisit"
  "horizon": "long",                     // "long" | "swing"
  "commandVersion": "0.5.0",             // version of /stockwiz that created this session
  "createdAt": "2026-04-11T14:32:00-07:00",  // ISO 8601 with local tz offset

  // Status of the session. Transitions: started -> (complete | degraded | failed).
  "status": "started",                   // "started" | "complete" | "degraded" | "failed"

  // Pipeline stages, appended in order as they run.
  // Each stage entry has { stage, startedAt, endedAt, status, ...stage-specific-fields }.
  "stages": [
    {
      "stage": "fetch-shard-A",          // one entry per fetch shard, A through D
      "startedAt": "2026-04-11T14:32:01-07:00",
      "endedAt":   "2026-04-11T14:33:45-07:00",
      "status": "ok",                    // "ok" | "degraded" | "failed"
      "succeeded": 3,                    // count of this shard's sources that returned usable data
      "failed": 0,                       // count of this shard's sources marked failed
      "parallel": true,                  // informational — ran concurrently with its siblings
      "notes": ""
    },
    // ...fetch-shard-B, fetch-shard-C, fetch-shard-D
    {
      "stage": "fundamental-analysis",
      "startedAt": "...",
      "endedAt": "...",
      "status": "ok",
      "output": "analysis/fundamental.md",
      "parallel": true
    },
    // ...sentiment-synthesis, peer-comparison, risk-screen, thesis-discipline (full),
    //    devils-advocate, thesis-discipline (reconcile), report-writer
    {
      "stage": "report-writer",
      "startedAt": "...",
      "endedAt": "...",
      "status": "ok",
      "template": "deep-dive",
      "outputFile": "report.html",
      "outputBytes": 98432,
      "sectionsRendered": {              // per-section fidelity
        "hero": "full",
        "metrics": "full",
        "tldr": "full",
        "three-cases": "full",
        "kill-switches": "full",
        "fundamentals": "full",
        "sentiment": "full",
        "peers": "thin",                 // "full" | "thin" | "placeholder"
        "risk": "full",
        "assumption-ledger": "full",
        "unknowns": "full",
        "sources": "full",
        "adversarial": "full"
      },
      "tldrAtoms": {                     // curated TL;DR content (non-deterministic)
        "keyInsight": "ROIC of 126% is among the highest in US equities...",
        "closestKillSwitch": "Gross margin below 65% for two consecutive quarters",
        "biggestUnknown": "Customer concentration of data-center revenue..."
      },
      "adjustments": [],                 // compliance pass rewrites applied, if any
      "strippedSentences": [],           // sentences removed by "strip" compliance rules
      "complianceIterations": 1,         // 1-3; 3+ means abort
      "disclaimerNote": "Disclaimer boilerplate exempted from compliance pass"
    }
  ],

  // Per-source fetch results, merged in by the orchestrator from the shard returns.
  // Keys are source slugs from source-extraction/SKILL.md. The fetched URL is not
  // duplicated here — it lives in the raw file's frontmatter (the audit trail).
  "sources": {
    "sec-edgar": {
      "status": "ok",                    // "ok" | "failed" | "degraded"
      "fetched_at": "2026-04-11T14:32:15-07:00",
      "file": "raw/sec-edgar-10k.md"     // path relative to session dir
    },
    "yahoo-finance": {
      "status": "failed",
      "reason": "rate-limited",          // see failure-mode detection table in source-extraction/SKILL.md
      "fetched_at": "2026-04-11T14:33:20-07:00",
      "file": "raw/yahoo-fundamentals.md"  // written as a failure stub
    }
    // ...finviz, stockanalysis, macrotrends, google-finance, simply-wall-street,
    //    seeking-alpha, zacks, reddit, barchart, x-com
  },

  // Terminal state fields set by the finalize stage.
  "completedAt": null,                   // set when status transitions to "complete"
  "failedAt": null,                      // set when status transitions to "failed"
  "error": null,                         // human-readable description on failure
  "reportPath": null,                    // "report.html" once written
  "thesisPath": "thesis.md",             // set at session creation

  // Follow-up operations appended by future /stockwiz-revisit runs (roadmap).
  "followUps": []
}
```

### Valid field transitions

- `status`: starts as `"started"`, transitions forward only — `started → complete` / `started → degraded → complete` / `started → failed`. Never goes back to `started`.
- `sources[slug].status`: set once by the orchestrator when merging the shard returns, never modified after. If a source fails and a later retry succeeds, write a new source slug (e.g. `yahoo-finance-retry`), don't mutate the original.
- `stages[]`: append-only. Never remove or reorder entries.
- `reportPath`, `completedAt`, `failedAt`, `error`: set by the finalize step of `/stockwiz`, not by subagents.

### Reason codes for `sources[slug].reason`

When `status == "failed"`, the `reason` field must be one of:

| Reason code | Meaning |
|---|---|
| `cloudflare-challenge` | Cloudflare bot-management page intercepted |
| `consent-wall` | Regional consent/GDPR wall |
| `403` | Origin rejected request (e.g. SEC UA block, Zacks) |
| `429` | HTTP 429 rate limit |
| `503` | HTTP 503 server overload / temporary |
| `rate-limited` | Either HTTP 429 OR body-encoded 429 (Reddit) |
| `empty-response` | Body was <200 bytes or empty JSON |
| `redirect-loop` | 3+ redirects or loop detected |
| `auth-wall` | Login/signup page intercepted |
| `404` | Source does not have this ticker |
| `api-error: <msg>` | Source returned a structured error (Yahoo quoteSummary errors) |
| `timeout` | curl exit 28 |
| `dns-failure` | curl exit 6 |
| `invalid-json` | JSON parse failed |
| `extraction-empty` | lynx dump produced <500 chars of useful content |
| `ticker-not-found-at-sec` | Ticker not in SEC `company_tickers.json` |
| `non-us-filer-20f-only` | Foreign private issuer; no 10-K |
| `sub-private-or-banned` | Reddit subreddit inaccessible |
| `no-posts` | Reddit search returned zero matches |
| `sws-not-covered` | Simply Wall Street does not cover this ticker |

This list is not exhaustive — per-source reference files may introduce additional codes. New codes should be added to this table when introduced.

## Config file schema (~/.claude/stockwiz/config.json)

```jsonc
{
  "version": 1,
  "fredApiKey": null,                    // string or null (optional, for future macro-context skill)
  "defaultHorizon": "long",              // "long" | "swing"
  "ratelimit_ms": 1500                   // minimum ms between fetches (WebFetch or curl)
}
```

`/stockwiz-setup` creates this file with defaults. `/stockwiz` reads it at start and refuses to run if missing. No other stage writes to it.

## Cache schemas

### `cache/company_tickers.json`

Mirror of SEC's `https://www.sec.gov/files/company_tickers.json` (their format):

```jsonc
{
  "0": {"cik_str": 320193, "ticker": "AAPL", "title": "Apple Inc."},
  "1": {"cik_str": 789019, "ticker": "MSFT", "title": "Microsoft Corp"},
  // ...
}
```

Refreshed by the `/stockwiz` orchestrator's pre-flight (Step 4.5) when: (a) the file doesn't exist, or (b) it is more than 30 days old. The pre-flight also does the ticker→CIK lookup locally, so `ticker-not-found-at-sec` aborts before any shard is dispatched.

### `cache/macrotrends-slugs.json`

Self-populating mapping from ticker to Macrotrends company slug:

```jsonc
{
  "NVDA": "nvidia",
  "AAPL": "apple",
  "MSFT": "microsoft"
  // append-only as new tickers are seen
}
```

`deep-researcher` reads this before calling WebSearch for slug resolution; on cache miss, it resolves via WebSearch and writes the new `(ticker, slug)` pair back. Read-modify-write pattern.

## Cleanup and retention

stockwiz does **not** auto-delete session directories. They accumulate under `~/.claude/stockwiz/sessions/` until the user manually prunes them. This is deliberate — a session is the audit trail of a research decision and should outlive its moment.

A future `/stockwiz-prune` or manual rotation may be added if session directories become unwieldy. Until then, users who want to clean up can `rm -rf ~/.claude/stockwiz/sessions/<OLD_TICKER>-*` at their own risk.

## Adding a new source to the filesystem contract

When adding a new source reference file under `skills/source-extraction/references/`:

1. Add the source slug to the source index table in `skills/source-extraction/SKILL.md`
2. Pick a raw filename slug (e.g. `raw/new-source.md`) and document it in the reference file
3. Add any audit-trail JSON/XML/HTML files the source fetches to the list above (e.g. `raw/new-source-data.json`)
4. If the source needs a cache file, add it under `~/.claude/stockwiz/cache/` and document its schema here
5. If the source uses a tempfile pattern, document it under the scratch section

The `deep-researcher` agent's fetch plan should be updated in the same commit so the source is actually invoked.
