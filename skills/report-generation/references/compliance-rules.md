# Compliance rules — banned phrases and rewrites

stockwiz is a research tool, not a licensed investment advisor. Every HTML artifact passes this compliance check in the `report-writer` agent **before** it is written to disk. The goal is consistent, neutral, analytical language — not legal hedging for its own sake.

## The core principle

Any phrase that reads as advice, a promise, or a guarantee is rewritten to descriptive, analytical language. "Consider" replaces "buy". "May see" replaces "will return". Nothing is "safe" or "guaranteed" or "risk-free".

Quoted source material is exempt from rewriting **but must be wrapped in `<q>...</q>` tags** and clearly attributed. The compliance scanner skips matches inside `<q>` tags.

## Banned phrase table

All matches are **case-insensitive with word boundaries** (regex `\b...\b`).

| # | Banned phrase | Rewrite | Notes |
|---|---|---|---|
| 1 | `buy` (imperative, not "buyback" / "buy-side" / "buyer") | `consider` | Do not rewrite "buyback", "share buyback", "M&A buy", "buy-side" (industry term for investment managers), "buyer" |
| 2 | `sell` (imperative, not "sell-through", "sell-side", "sales", "seller") | `re-evaluate` | Do not rewrite "sell-through", "sell-side" (industry term for research desks / analysts), "sales", "seller" |
| 3 | `recommend` / `recommendation` | `analysis suggests` / `analytical view` | |
| 4 | `should buy` | `may consider` | |
| 5 | `should sell` | `may re-evaluate` | |
| 6 | `will return` / `will gain` | `may see` / `has potential for` | Forward-looking certainty is disallowed |
| 7 | `guaranteed` / `guarantee` (as an investment claim) | `likely` / `indicated` | Do not rewrite "dividend guarantee" if it's a quoted term |
| 8 | `risk-free` | `lower-risk` | |
| 9 | `undervalued` (as a claim) | `trading below framing` | |
| 10 | `overvalued` (as a claim) | `trading above framing` | |
| 11 | `moonshot` / `to the moon` | `asymmetric upside scenario` | |
| 12 | `N-bagger` / `bagger` | `multi-fold upside` | |
| 13 | `safe` (applied to an investment) | `lower-volatility` | Do not rewrite "safe harbor", "safe haven asset" in quotes |
| 14 | `no risk` | `limited disclosed risk` | |
| 15 | `can't lose` | **STRIP SENTENCE, LOG** | |
| 16 | `pump` / `dump` (in investment context) | **STRIP, LOG** | |
| 17 | `hot stock` / `hot pick` | `high-attention name` | |
| 18 | `next Amazon` / `next Nvidia` (or similar "next X" pattern) | `comparable category to X` | Name-based templates |
| 19 | `free money` | `asymmetric return scenario` | |
| 20 | `easy money` | `low-friction scenario` | |

## Pass procedure

The `report-writer` agent runs this pass after composing the HTML string and before writing to disk:

1. Load this file.
2. Load the full HTML string.
3. For each rule in the table:
   a. Search the HTML with a case-insensitive, word-boundary regex.
   b. **Skip any match that is inside a `<q>...</q>` tag.** Quoted source text is exempt (the quote tags make the attribution clear).
   c. Skip matches inside URL strings (`https://` through the next space or quote).
   d. If the rule is **STRIP SENTENCE**: remove the entire enclosing sentence and append the stripped sentence to a log list.
   e. Otherwise: apply the rewrite, preserving surrounding punctuation.
4. After all rules have been applied, **re-scan** the HTML for any remaining banned phrases outside `<q>` tags.
5. Loop (apply → re-scan) up to **3 times** total.
6. If after 3 iterations any banned phrase still remains outside `<q>`, **ABORT** the report generation and return an error. This indicates the report content is systematically out of compliance and the issue needs to be addressed upstream (in the `thesis-discipline` agent or one of the four analysis agents).
7. Log every rewrite applied and every stripped sentence to `meta.json.stages` under the `report-writer` stage as an `adjustments` array.

## What about quoted source material?

Many sources (Zacks, Seeking Alpha, TradingView) label stocks with "Strong Buy", "Sell", etc. When the report-writer needs to reproduce a source's label verbatim for accuracy, it **must**:

1. Wrap the quoted text in `<q>` tags: `<q>Strong Buy (Seeking Alpha Quant)</q>`
2. Attribute the source inline or via footnote
3. Describe the label in neutral language in the surrounding prose: "Seeking Alpha's quant model assigns this ticker its <q>Strong Buy</q> label (a 5 on their 1–5 scale)"

The surrounding prose is compliance-checked, but the quoted text inside `<q>` is not rewritten. This preserves source accuracy without making stockwiz appear to issue recommendations.

## Edge cases the scanner must handle

- **Word boundaries matter.** `\bbuy\b` matches "buy" but not "buyback" or "buyer".
- **Hyphenated compounds create word boundaries.** The `-` character is a non-word character, so `\bsell\b` will match inside `sell-side`, and `\bbuy\b` will match inside `buy-side`. These are industry terms, not imperatives — **explicitly exempt them**. Before applying rule 1 or 2, check if the match is immediately followed by `-side` or preceded by `-side`; if so, skip it.
- **Case-insensitive match, case-preserved rewrite.** If the source had "Buy", the rewrite should be "Consider" (capital C). Simpler to just lowercase rewrites and let context sort it.
- **Compound phrases rewritten first.** Rule 4 (`should buy`) is matched before rule 1 (`buy`) so that "should buy" becomes "may consider" rather than "should consider".
- **Inside HTML attributes.** Don't rewrite banned phrases that appear inside `alt=""`, `title=""`, `href=""`, or `<!-- comments -->` — these are structural, not content.
- **Inside `<code>` or `<pre>`.** Exempt — likely a snippet of a URL or source marker.
- **Inside footnote numbers.** The `<sup>` citations should not contain prose, so this is rare, but skip them defensively.
- **Disclaimer footer boilerplate.** The `stockwiz-disclaimer` block contains the phrase "recommendation to buy, sell, or hold" as part of legally-required disclaimer language. This text is pre-approved and exempt — skip rewriting inside `class="stockwiz-disclaimer"` blocks.

### Explicit exemption list (do NOT rewrite)

These tokens look like banned phrases under a naive regex but are industry terminology or legally-mandated text:

| Token | Why it's exempt |
|---|---|
| `sell-side` | Industry term for research desks and analysts at investment banks |
| `buy-side` | Industry term for investment managers and asset allocators |
| `buyback` / `share buyback` | Corporate action, not an imperative |
| `buyer` / `seller` | Descriptive noun, not an imperative |
| `sell-through` | Retail metric |
| `sales` / `sales growth` | Revenue terminology |
| `sold-out` / `sold out` | Inventory status |
| Disclaimer footer contents | Legally-required boilerplate |

## Ordering

The rules are applied in **table order** (most-specific first), which matters because "should buy" must be caught before "buy" alone. When extending this table, place longer/compound phrases before shorter ones that are substrings of them.

## Extension

When the HTML spot-check during verification catches a new problematic phrase, add it here with a neutral rewrite and re-run the reports. The compliance list is explicitly a living document.

---

## Pre-filter guidance for skill authors

This section is the **canonical reference** for skills and subagents that write content which will later pass through the compliance pass. If you are writing a new skill or agent that produces prose destined for the HTML report (analysis files, thesis, adversarial output), read this section and link to it from your SKILL.md rather than duplicating the banned-phrase list in your own file.

### The principle

The compliance pass (run by `report-writer` Step 8 before writing `report.html`) rewrites banned imperative language and wraps quoted source text in `<q>` tags. You, the skill author, should **pre-filter** your output so the compliance pass has little to do. Pre-filtering is not a safety net — the compliance pass is the safety net. Pre-filtering is an efficiency and clarity concern: every phrase that gets rewritten by the compliance pass is a phrase whose original wording was lost, and every iteration of the pass burns tokens.

### Phrases to avoid generating

You should never generate these in your own prose (i.e. not quoted from a source):

- `buy` / `sell` / `recommend` / `recommendation` / `should buy` / `should sell` as imperatives or advice
- `guaranteed` / `risk-free` as claims about an investment
- `undervalued` / `overvalued` as standalone claims
- `moonshot` / `to the moon` / `N-bagger` / `bagger`
- `safe` as applied to an investment ("a safe stock")
- `no risk` / `can't lose` / `pump` / `dump`
- `hot stock` / `hot pick` / `free money` / `easy money`
- `next Amazon` / `next Nvidia` pattern

### Phrases to use instead

- Instead of "buy X" → **"consider X"** or "X is analytically interesting"
- Instead of "sell X" → **"re-evaluate X"** or "X warrants re-evaluation"
- Instead of "recommend X" → **"analysis suggests X"** or "the analytical view is X"
- Instead of "should buy" → **"may consider"**
- Instead of "will return 20%" → **"may see 20% returns"** or "has the potential for 20% returns"
- Instead of "guaranteed" → **"likely"** or "indicated by the data"
- Instead of "risk-free" → **"lower-risk"**
- Instead of "undervalued" → **"trading below framing"** or "trading below peer median"
- Instead of "overvalued" → **"trading above framing"**
- Instead of "safe investment" → **"lower-volatility position"**

### Quoting source text

When you need to reproduce a source label verbatim — e.g. "Seeking Alpha Quant Rating: Strong Buy", "Yahoo recommendationKey: strong_buy", "Zacks Rank 1 (Strong Buy)", "SWS flags 'high non-cash earnings'" — capture the label verbatim in your markdown WITHOUT `<q>` tags. The report-writer wraps source labels in `<q>` tags at render time based on the source slug, which makes the compliance pass skip them. Your job is to preserve attribution, not to sanitize the source.

Example: in your analysis file, write

```
Seeking Alpha's Quant Rating is "Strong Buy" (4.75 on a 1-5 scale, higher = more positive).
```

The report-writer will render this as:

```html
<p>Seeking Alpha's Quant Rating is <q>Strong Buy</q> (4.75 on a 1-5 scale, higher = more positive).</p>
```

The compliance pass sees `<q>Strong Buy</q>` and leaves it alone.

### Industry terminology is exempt

These words look banned under a naive regex but are legitimate industry terminology. The compliance pass explicitly exempts them, and you may use them freely in prose:

- `sell-side` / `buy-side` (research desks and asset managers)
- `buyback` / `share buyback` (corporate action)
- `sell-through` (retail metric)
- `sales` / `sales growth` (revenue terminology)
- `seller` / `buyer` (descriptive nouns)
- `safe harbor` / `safe haven` (financial terms)

### Why pre-filter?

1. **Preserves author voice.** A thesis paragraph that you wrote with specific wording is more coherent than the same paragraph after the compliance pass has rewritten three words.
2. **Reduces compliance-pass churn.** The pass loops up to 3 times applying rewrites. If you pre-filter, it usually completes in 1 iteration with zero rewrites — faster and cleaner.
3. **Improves reader clarity.** "Analysis suggests this stock is trading below framing" reads deliberately. "This is a buy — you should definitely consider it" (auto-rewritten to "This is a consider — you should definitely consider it") reads like a machine mangled it.
4. **Aligns mental models.** Pre-filtering encourages skill authors to think analytically rather than advisorially from the start. The compliance pass is a guard against drift, not a substitute for discipline.

### What to link from your SKILL.md

Instead of duplicating the banned list in your skill's Hard Rules section, add a one-line reference:

> **No banned imperative language.** See `compliance-rules.md` § "Pre-filter guidance for skill authors" for the canonical list of phrases to avoid and their neutral alternatives. The compliance pass is a safety net; your output should not trigger it.

This gives every skill a single source of truth for compliance guidance and makes adding new banned phrases a one-place edit.
