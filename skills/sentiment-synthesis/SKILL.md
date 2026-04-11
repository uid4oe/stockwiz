---
name: sentiment-synthesis
description: This skill should be used when synthesizing sentiment signals (SWS risks/rewards, Seeking Alpha factor grades, Finviz insider transactions and short interest, analyst consensus, news) for a US equity. Applies explicit weighting with 30-day recency half-life and produces a structured sentiment summary with conflicting signals preserved. Read from SESSION_DIR/raw/ and write to SESSION_DIR/analysis/sentiment.md.
version: 0.1.0
---

# sentiment-synthesis

The "soft signals" skill. Where `fundamental-analysis` deals in SEC numbers and multi-year trends, this skill deals in what the crowd, the insiders, the analysts, and the alternative-data providers are collectively saying. It is explicitly NOT a consensus-translator — its job is to preserve disagreements and weight recency, not to pick a side.

## When to use

Loaded by `/stockwiz` after `deep-researcher` and alongside `fundamental-analysis`. Input is `SESSION_DIR/raw/`, output is `SESSION_DIR/analysis/sentiment.md`.

## Core principle

**Conflicts are signal, not noise.** If Seeking Alpha's quant is positive and SWS's risks list is negative, that disagreement IS the data. You preserve both, label their weights, and flag the conflict explicitly. You do NOT compute a net "sentiment score" and call it a day — that erases information.

## Inputs to read

Check `status: ok` in each file's frontmatter. Skip failed sources.

**Primary sources — news layer (Phase 2.5):**
- `raw/google-finance.md` — **the news layer**. Google News RSS produces up to 20 recent headlines with publishers and publication dates. This is the single most important input for the news timeline and the publisher-distribution shape signal. Read the "Recent news" section and the "Publisher distribution" metric.

**Primary sources — alternative view & proprietary ratings:**
- `raw/simply-wall-street.md` — SWS risks list (high signal), rewards list, narrative verdict. The risks list is the single most valuable qualitative input; it typically surfaces 3-5 specific concerns analysts are watching.
- `raw/seeking-alpha.md` — SA Quant Rating + Factor Grades (when SSR-available — note gap), Wall Street analyst distribution, article teaser titles as a secondary news proxy
- `raw/zacks-snapshot.md` — **Zacks Rank (1-5) + Style Scores (VGM)** — unique proprietary ratings not covered by any other source. When Zacks succeeds, include its rank and style scores in the Analyst Positioning section. When it fails (Cloudflare), note in Unknowns.

**Primary sources — insider / short / quantitative positioning:**
- `raw/finviz-snapshot.md` — insider transactions (3-month % change), institutional transactions, short float, analyst recom numeric (1–5)
- `raw/sec-edgar-10k.md` — most recent 8-K filings (material events) from the submissions API section

**Primary sources — retail contrarian:**
- `raw/reddit.md` — **retail sentiment from r/stocks, r/wallstreetbets, r/investing**. Weight 0.2 (lowest in the rubric). Use for the contrarian-flag check: if r/wallstreetbets activity is `heavy` AND title lexicon is uniformly bullish (🚀, YOLO, to the moon), that's a historical near-term-consolidation signal and belongs in Conflicting Signals, NOT as a confirmation of a bull thesis.

**Secondary sources:**
- `raw/yahoo-fundamentals.md` — `recommendationKey` string and 5-tier analyst distribution (if Yahoo succeeded)
- `raw/stockanalysis.md` — analyst distribution, next earnings date, average price target

## Weighting rubric

Each source type gets a weight from 0.0 to 1.0, and a recency half-life of 30 days (older information decays by 50% per 30 days).

| Source type | Weight | Rationale |
|---|---|---|
| SEC 8-K material events | 1.0 | Ground truth — legally-mandated disclosures |
| SEC Form 4 insider transactions | 0.9 | Dollar-weighted insider activity with dates |
| Google News RSS from wire services (Reuters, Bloomberg, WSJ, FT, AP) | 0.85 | Reported, edited, fact-checked; `<source>` element in RSS identifies publisher |
| Finviz insider transactions | 0.8 | Hard $ signal with dates |
| Tier-1 news (identified as Tier-1 via publisher in Google News RSS) | 0.8 | Same as above for non-wire Tier-1 |
| Zacks Rank + Style Scores | 0.7 | Proprietary quant composite, widely referenced |
| Analyst research reports (named firms) | 0.6 | Incentive-laden but well-sourced |
| Simply Wall St risks list | 0.6 | Analytical composite, not raw sentiment |
| Google News RSS from opinion publishers (SA, Motley Fool, Benzinga, Zacks op-ed) | 0.5 | Lower weight reflects mixed-quality editorial |
| Seeking Alpha Quant Rating + Factor Grades | 0.5 | Quantitative composite, not editorial |
| Finviz analyst recom numeric | 0.5 | Composite of many sources |
| Simply Wall St rewards list | 0.4 | Often marketing-tinted |
| Seeking Alpha article teasers (titles only) | 0.3 | Headline-level only; no body access |
| Reddit (r/stocks, r/wsb, r/investing) | 0.2 | High noise; useful ONLY as a contrarian signal at extremes |

Recency half-life: every signal is time-stamped. A 60-day-old signal has half the weight of a 1-day-old signal. If a signal isn't dated in the raw source, treat it as "as-of fetch time" and note that assumption.

**Publisher distribution shape signal** (from Google News RSS): compute the share of wire-service headlines (Reuters/Bloomberg/WSJ/FT/AP) vs opinion-publisher headlines (SA/Benzinga/Fool) in the 20-item capture. A wire-heavy distribution (>60% wire) signals "something is happening that wire services are reporting" — earnings, M&A, regulatory, macro. An opinion-heavy distribution signals "retail narrative cycle" — momentum, meme, speculation. Note the shape in the Net Sentiment Read paragraph; do NOT assign it a numeric score.

## Output structure

Write `SESSION_DIR/analysis/sentiment.md`:

```markdown
# Sentiment Synthesis — <TICKER>

*Generated by sentiment-synthesis skill from <N> raw sources. Source files: {list}.*

## Net Sentiment Read

A single paragraph framing the overall sentiment picture — NOT a single numeric score. Describe the shape: "Analytical and quantitative sources lean positive, insider activity is uniformly negative, and there's one material recent catalyst to flag." Identify the two or three most important signals.

## Recency-Weighted News

Vertical dated list of material news items. **Primary source in Phase 2.5 is Google News RSS** from `raw/google-finance.md`, typically 15-20 items. Supplement with SEC 8-K filings (from `raw/sec-edgar-10k.md` submissions metadata) and SA article teaser titles (from `raw/seeking-alpha.md`).

Format:
- **[YYYY-MM-DD]** (publisher, weight) — headline wrapped in `<q>` for compliance. Citation.

Most recent items at the top. Items older than 45 days get trimmed unless materially unique (e.g. an 8-K or a rating change). Aim for 8-12 items in the final list — not the full 20 from Google News, just the meaningful ones.

**Tag each item's weight inline** per the rubric — readers need to see that a Reuters headline weighs more than a Seeking Alpha op-ed. Example:
- **[2026-04-10]** (*Reuters*, weight 0.85) — <q>Nvidia stock hits fresh high on AI chip demand</q>
- **[2026-04-10]** (*Seeking Alpha*, weight 0.5) — <q>NVIDIA: The Rerating Is Over, The Growth Story Isn't</q>

## Publisher Distribution

One-paragraph description of the wire-service vs opinion-publisher mix from Google News RSS. Example: "16 of 20 recent headlines are from opinion publishers (Seeking Alpha, Motley Fool, Benzinga) with only 4 from wire services (Reuters, Bloomberg) — this is a retail-narrative-cycle distribution, not a material-events distribution. Interpret subsequent signals accordingly." Do NOT reduce this to a single score; the shape is the signal.

## Social Signal

Two parts: **analyst narrative** (from SWS verdict + SA teaser titles) and **retail sentiment** (from Reddit).

**Analyst narrative:** SWS narrative verdict (verbatim inside `<q>`), SA article teaser frequency and title-lexicon observations. Describe the shape of the conversation, not your opinion of it.

**Retail sentiment (Reddit, weight 0.2):** from `raw/reddit.md`, report for each subreddit:
- Activity level: `quiet` / `normal` / `elevated` / `heavy` (per reddit.md's convention)
- Total post count matching the ticker in last 30 days
- Average score
- Title-lexicon shape: `bullish` (🚀, YOLO, "to the moon", "calls"), `bearish` ("puts", "bagholders", "crash"), `mixed`, `analytical` — assess from titles only, never read post bodies

**Contrarian-flag check:** if r/wallstreetbets activity is `heavy` AND title lexicon is uniformly bullish, note this as a contrarian flag in Conflicting Signals. Historically this correlates with near-term consolidation more often than continued uptrends. **Do NOT treat r/wsb enthusiasm as confirmation of a bull thesis** — it's specifically the opposite signal at extremes.

## Insider Activity

- Insider ownership % (latest)
- Net insider transactions over trailing 3 months (from Finviz and/or SWS qualitative flag)
- Notable individual transactions (if disclosed with $ amounts)
- 6-month or 12-month trend if available
- Qualitative flag from SWS (e.g. "SWS flags 'insiders have only sold in the past 3 months'")

Cite every figure.

## Short Interest

- Short as % of float
- Short ratio (days to cover)
- Trend over prior month (if stockanalysis shows it)
- Note: contextual only. Short interest spikes can mean short thesis or can mean squeeze setup — do not interpret.

## Analyst Positioning

Six distinct composite ratings when all sources succeed. Present each with its source and neutral framing; wrap raw labels in `<q>` tags.

- **Number of analysts covering** (from SA, stockanalysis, or Yahoo)
- **Average price target** and % from current price
- **Wall Street distribution**: <q>Strong Buy</q> N, <q>Buy</q> N, Hold N, <q>Sell</q> N, <q>Strong Sell</q> N
- **Finviz analyst recom numeric** (1.0–5.0, lower = more positive)
- **SA Quant Rating** numeric (if SSR-available — note if client-side-only and thus missing)
- **SA Factor Grades** (if SSR-available — Valuation/Growth/Profitability/Momentum/Revisions)
- **Zacks Rank** (1-5 integer from `raw/zacks-snapshot.md`, with label in `<q>` tags). Note: Zacks is best-effort; if the source was Cloudflare-blocked, note in Unknowns.
- **Zacks Style Scores** (Value/Growth/Momentum/VGM — each A-F)
- **Yahoo recommendationKey** raw string (wrapped in `<q>`, if Yahoo succeeded)

Frame in neutral analytical language. Never generate advisory text of your own — only reproduce source labels inside `<q>`.

**Cross-source agreement check:** of the composite ratings available, do they agree or disagree? Example: "Finviz analyst recom 1.3 (~Strong Buy), SA Quant 4.75 (~Strong Buy), Zacks Rank 2 (<q>Buy</q>), Wall Street 32/48 at <q>Strong Buy</q> — four sources align on the positive end. Alignment is itself a signal (consensus is well-informed OR consensus is herded; either is useful context)."

## Conflicting Signals

**The most important section of this skill.** List every place where sources disagree:

1. SA Quant Rating says X but SWS risks list says Y
2. Insider transactions are net-selling but SWS rewards list flags improving ROCE
3. Finviz analyst recom is 1.3 (~Strong Buy) but SWS snowflake Value score is 1/6 (~Overvalued)
4. (etc)

For each conflict: state both signals with their weights, the source files, and do NOT resolve. The thesis-discipline skill downstream may use these conflicts as explicit disconfirmers.

## Unknowns

- Article body content (paywalled behind Seeking Alpha Premium / WSJ / Bloomberg)
- Reddit sentiment (Phase 2+)
- Social media activity (X/Twitter, Discord, etc — not scraped)
- Specific insider transaction $ amounts per person (only net % available from Finviz)
- Analyst report content beyond ratings and price targets
- Any SA Factor Grade / Quant Rating that was client-side-loaded and missing from SSR HTML
```

## Hard rules

1. **No numeric sentiment score.** Resist the temptation to compute a single "sentiment = 0.73" output. You lose too much information, and the thesis-discipline skill downstream is smarter than a scalar.
2. **Conflicts are preserved.** Never pick sides in a source disagreement. List both, weight them, let downstream decide.
3. **Date every signal.** If a signal isn't dated, note that it's "as-of fetch time" and discount its weight accordingly.
4. **Source labels go in `<q>` at report time.** In this file, capture "Strong Buy" verbatim without `<q>` tags — that wrapping happens later in report-writer. But do NOT generate your own advisory prose. Use neutral framing everywhere outside direct quotes.
5. **Never editorialize.** "Investors are worried" is fabrication unless a specific source said it. "SWS's risks list includes 'profit margin decreased' and 'insiders have only sold in the past 3 months'" is a factual restatement.
6. **Recency decays.** A stale news item from 6 months ago has a quarter of the weight of a current one. State the recency in your framing: "6-month-old analyst report (weight 0.15 after half-life decay)".
7. **Match signal weight to claim strength.** A weight-0.2 signal should not anchor a strong claim. If your strongest insider-activity signal is from Reddit (weight 0.2), your framing should be "there is some online commentary suggesting" not "investors widely note that".

## A note on Seeking Alpha in Phase 1.5

Phase 1.5 learned that SA's Quant Rating and Factor Grades are client-side-loaded and not in the SSR HTML that `curl + lynx` retrieves. The current raw file often has business description + article teasers but no grades. When this happens:

- Use the article teaser titles as a news proxy (weight 0.3)
- Note the gap in Unknowns
- Do NOT fabricate grades from the teasers — they contain opinion words like "Rerating" and "Game Changer" that are emphatically NOT stockwiz-sourced signals

Phase 2+ will either find SA's direct JSON API for factor grades or use headless Chrome to render the page.
