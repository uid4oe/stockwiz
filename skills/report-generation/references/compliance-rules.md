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
6. If after 3 iterations any banned phrase still remains outside `<q>`, **ABORT** the report generation and return an error. This indicates the report content is systematically out of compliance and the issue needs to be addressed upstream (in the `thesis-discipline` skill or the analysis skills).
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
