---
description: One-time onboarding for stockwiz. Creates data directories, optionally stores a FRED API key, and shows the disclaimer. Run this once after installing the plugin.
argument-hint: (no arguments)
allowed-tools: Bash, Read, Write, AskUserQuestion
---

# /stockwiz-setup

You are running the one-time onboarding flow for the stockwiz plugin. This command is idempotent — running it again is safe and will not destroy existing config or sessions. It is intentionally dependency-free so it works on a completely fresh install.

## Step 1 — Introduce stockwiz

Print a brief, plain-language introduction. Three sentences, no marketing language:

> stockwiz is a research copilot for US equity analysis. It aggregates public information from ~11 free web sources, forces disciplined bull/base/bear theses with measurable kill switches, runs an adversarial devil's-advocate pass on every thesis, and produces self-contained HTML reports you can open offline or share. It is a tool for thinking, not a source of advice.

## Step 2 — Show the disclaimer

Read `${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/disclaimer.html`. Strip the HTML tags and show the content as prose to the user. This is the same disclaimer that will appear on every report artifact. It's important the user sees it at least once before using stockwiz.

Format it with a clear heading:

```
──────── Compliance notice ────────

Analytical tool, not investment advice. stockwiz is a research assistant that
synthesizes publicly available information. Nothing in this document is an
offer, solicitation, or recommendation to buy, sell, or hold any security.
Figures are drawn from public sources at the fetch time shown and may be
stale, incomplete, or in error. Forward-looking statements are analytical
framings, not predictions. You are responsible for your own investment
decisions and should consult a licensed financial professional before acting
on any information this tool generates.

───────────────────────────────────
```

## Step 3 — Create data directories

Run these shell commands via Bash:

```bash
mkdir -p ~/.claude/stockwiz/sessions
```

Verify the directory now exists. If the `mkdir` fails (permission issue, disk full), surface the error clearly and abort — don't continue with a broken setup.

## Step 4 — Write or preserve the config file

Check if `~/.claude/stockwiz/config.json` already exists:

- **If it does not exist**, write this default:
  ```json
  {
    "version": 1,
    "fredApiKey": null,
    "defaultHorizon": "long",
    "ratelimit_ms": 1500
  }
  ```
- **If it does exist**, read it and leave it alone. Setup is idempotent — we do not clobber user config on re-run.

## Step 5 — Offer to store a FRED API key

Ask the user via `AskUserQuestion`:

**Question:** "stockwiz can optionally use the FRED (Federal Reserve Economic Data) API for macro context on rates-sensitive and cyclical sectors. FRED is free but needs an API key. Would you like to set one up now?"

**Options:**
1. "Yes, I have a FRED key" — proceed to next step
2. "No, skip for now" — leave `fredApiKey` as null; stockwiz will fall back to scraping the FRED public pages
3. "What is FRED?" — explain in one paragraph and re-ask

**If the user picks "What is FRED?"**, explain:
> FRED is the Federal Reserve Bank of St. Louis's free economic data service. It publishes thousands of time series covering interest rates, inflation, employment, commodity prices, and more. stockwiz uses it when a thesis depends on macro variables — for example, a REIT thesis that hinges on the 10-year Treasury yield, or a copper miner that tracks commodity prices. You can get a free API key at https://fred.stlouisfed.org/docs/api/api_key.html. It's optional — stockwiz will fall back to scraping FRED's public pages if you don't provide one, just slower and less clean.

Then re-ask the original question.

**If the user picks "Yes, I have a FRED key"**, ask for the key as a plain text follow-up (not via AskUserQuestion — just a direct prompt). When they provide it, read the existing `config.json`, update `fredApiKey`, write it back. Confirm: "Stored. stockwiz will use your FRED key for macro context."

**If the user picks "No, skip for now"**, acknowledge and move on. You can always run `/stockwiz-setup` again later to add it.

## Step 6 — Confirm understanding of scope

Ask via `AskUserQuestion`:

**Question:** "stockwiz is for analytical research, not for generating trading advice. You'll see disclaimers on every report. Are you clear on that?"

**Options:**
1. "Yes, understood"
2. "Wait — explain what's different about this vs a stock-picking tool"

**If the user picks option 2**, explain:
> stockwiz does not tell you what to buy or sell. It helps you build disciplined theses — forcing every view to have an explicit bull case, base case, bear case, specific disconfirmers, and measurable kill switches (thresholds you can check later to see if you were wrong). It runs an adversarial pass on every thesis that tries to destroy it, so you see the strongest counter-arguments before you commit. It tracks your theses over time via sessions, so you can revisit and see where you were right or wrong. The goal is to think better about stocks — not to get a ticker list to follow.

Then re-ask the original question.

Do not persist the consent. This is a gentle gate on intent, not a legal waiver. Every report has a disclaimer footer regardless.

## Step 7 — Print the quickstart

Show the user how to actually use stockwiz. Keep it to the two commands that actually exist — do not promise features that aren't wired.

```
──────── Quick start ────────

/stockwiz NVDA                         Full deep-dive on NVIDIA (long horizon)
/stockwiz NVDA --horizon=swing         Same deep-dive weighted for swing trades
/stockwiz AAPL                         Try any US ticker

Sessions live in ~/.claude/stockwiz/sessions/<TICKER>-<timestamp>/
Each run produces a self-contained report.html you can open offline.

After a run completes, ask follow-up questions in this same chat and
stockwiz will answer from the session workspace without re-fetching.

─────────────────────────────
```

If the user asks about other commands (`/stockwiz-compare`, `/stockwiz-bear`, etc.) — they are planned but not yet built. Point them at the Roadmap section of the README. Do NOT pretend they work.

## Step 8 — Done

Print: "Setup complete. Try `/stockwiz NVDA` to run your first deep-dive." and exit.

## Error handling

- If `mkdir` fails → surface the OS error, abort.
- If `~/.claude/stockwiz/config.json` exists but is invalid JSON → back it up as `config.json.bak-<timestamp>`, write a fresh default, inform the user.
- If the user cancels any AskUserQuestion → treat as "skip for now" and continue.
- If the disclaimer asset is missing → surface the path, recommend reinstalling the plugin, abort.

## Hard rules

- Never store anything persistent about the user other than the config file.
- Never ask for any secret other than the FRED key (and only if the user opts in).
- Never send the config file or any user data over the network.
- Never skip Step 2 (the disclaimer). It must be shown at least once.
