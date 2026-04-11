---
name: thesis-discipline
description: This skill should be used when synthesizing equity analyses into a disciplined investment thesis with mandatory bull/base/bear cases, explicit disconfirmers, and measurable kill switches. It forces structure that prevents vague framings and supports honest revisit and drift checking. Supports four modes — full synthesis from analyses, drift check against prior thesis, implicit-thesis extraction for pivot, and reconcile (merging devils-advocate feedback into a prior thesis).
version: 0.2.0
---

# thesis-discipline

This skill exists because most investment theses are vague in exactly the ways that matter: bull cases without specific triggers, bear cases treated as disclaimers, base cases that are "somewhere between bull and bear", kill switches that can never actually be triggered, and unknowns that are never named. This skill makes those failures impossible to commit.

## When to use

You are called by:
- `/stockwiz` (main command) in **full** mode after the four analysis skills have written their outputs, and again in **reconcile** mode after devils-advocate has written its pass
- `/stockwiz-thesis` in **full** mode after a reduced analysis pass
- `/stockwiz-compare` in **relative** mode (3-bullet bull/bear/base per ticker)
- `/stockwiz-revisit` in **drift** mode to compare a prior thesis to current state
- `/stockwiz-pivot` in **implicit** mode to extract the latent thesis from an analysis

## Four modes

### Mode: `full`
Synthesize a complete thesis from analysis files. Write `SESSION_DIR/thesis.md`.

**Phase 2+ inputs (preferred):**
- `SESSION_DIR/analysis/fundamental.md` — from fundamental-analysis skill
- `SESSION_DIR/analysis/sentiment.md` — from sentiment-synthesis skill
- `SESSION_DIR/analysis/peer-comp.md` — from peer-comparison skill
- `SESSION_DIR/analysis/risk.md` — from risk-screen skill
- Optionally `SESSION_DIR/analysis/macro.md` if present (Phase 4+)
- Reference `SESSION_DIR/raw/*.md` by path for citations, but do not re-read them unless you need a specific figure that isn't summarized in the analyses

**Fallback input mode (Phase 1.5 or if analyses are absent):**
If the analysis files are missing (e.g. a Phase 1.5 run that skipped the analysis skills), read directly from `SESSION_DIR/raw/` files. This is a degraded mode — note it in the thesis preamble: `*Generated in Phase 1.5 fallback mode — analyses skipped, synthesis direct from raw files.*`

When reading from analyses (Phase 2+), **prefer their content over re-deriving from raw**. The analyses already did the hard work of dating figures, computing derived metrics, and noting source disagreements. Your job is synthesis across the four analytical lenses, not re-computation.

**Output structure (mandatory).** `thesis.md` must contain exactly these sections in this order:

```markdown
# {TICKER} — {Company Name} — Investment Thesis

*Generated {ISO timestamp}. Horizon: {long|swing}. Sources: see [meta.json].*

## Bull Case

**Headline.** {One sentence that states the bull thesis as a claim, not a hope. TARGET: ≤180 characters including punctuation — if longer, produce a Compact Headline below.}

**Compact headline.** {Optional, ≤100 characters. Used by the TL;DR panel in the HTML report. Required only if the main Headline exceeds 140 characters. The compact form should preserve the core claim; it's a tighter paraphrase, not a truncation.}

**Supporting claims.**
1. {Claim with a specific, citeable figure} [raw/source-file.md#anchor]
2. {Claim with a specific, citeable figure} [raw/source-file.md#anchor]
3. {Claim with a specific, citeable figure} [raw/source-file.md#anchor]

**Disconfirmers.** (Things that would break the bull case if they appear.)
1. {Disconfirmer 1 — measurable, dated}
2. {Disconfirmer 2 — measurable, dated}

**Timeline.** 6m checkpoint: {what to watch}. 12m: {what to watch}. 24m: {what to watch}.

## Base Case

**Headline.** {One sentence. MUST be its own scenario with its own driving assumption, NOT "something between bull and bear".}

**Supporting claims.**
1. ...
2. ...
3. ...

**Disconfirmers.**
1. ...
2. ...

**Timeline.** ...

## Bear Case

**Headline.** ...

**Supporting claims.** ... (3, with citations)

**Disconfirmers.** ... (2 — things that would break your OWN bear case)

**Timeline.** ...

## Explicit Disconfirmers

A consolidated list of every disconfirmer from the three cases. Numbered, crossreferenced to the case they belong to. The reader should be able to build a watchlist from this section alone.

## Kill Switches

Measurable triggers — each one a specific threshold that can be checked on a future Finviz refresh. Every kill switch must have:
- A metric with a source (e.g. "gross margin from `raw/yahoo-fundamentals.md`")
- A threshold ("below 55%")
- A time window ("in any single quarter" or "for two consecutive quarters")

Bad kill switches that MUST be rewritten:
- "if margins collapse" — unmeasurable
- "if the story breaks" — unmeasurable
- "if sentiment shifts" — unmeasurable

Good kill switches:
- "if gross margin drops below 55% for two consecutive quarters (source: `raw/yahoo-fundamentals.md`)"
- "if insider selling exceeds $50M in a quarter (source: `raw/finviz-snapshot.md`)"
- "if short interest rises above 12% of float and stays there for 60 days (source: `raw/finviz-snapshot.md`)"

## Unknowns

**Ordered list (most material first) of things your analysis could not determine.** Ordering is mandatory: item 1 is the Unknown with the largest downstream impact on the thesis, item 2 the second-largest, etc. The report-writer picks item 1 for the "Biggest unknown" callout in the TL;DR panel, so getting this ordering right is load-bearing.

**Materiality ranking criteria** (apply in order):
1. **Impact on the dominant case.** An Unknown that would flip the Base case if resolved against the current framing is more material than one that would only adjust the Bear case.
2. **Impact on the Assumption Ledger.** An Unknown that maps to a high-sensitivity row in `analysis/fundamental.md`'s Assumption Ledger is more material than one that maps to a low-sensitivity row.
3. **Time horizon to resolution.** An Unknown that would be resolved by the next 10-Q is more actionable (and therefore more material) than one that resolves over 3 years.
4. **Cross-source information asymmetry.** An Unknown where multiple sources explicitly flag "not disclosed" is more material than one where a single source didn't mention it.

Be specific. "I don't know what the management team is planning for the next 18 months" is worse than "I don't know whether the company's Q3 guidance includes the pending Apple contract — the 10-K does not disclose; Seeking Alpha and news sources did not clarify".

Unknowns are a first-class output. A thesis with zero unknowns is a thesis that is lying about its confidence.

Format each item as a numbered bullet with the rank implied by position:

1. **[MOST MATERIAL]** {one-sentence unknown, with which case it affects and the resolution horizon}
2. {next most material}
3. {...}
```

### Mode: `drift`
Compare a prior thesis to the current state. Write `SESSION_DIR/analysis/revisit-YYYYMMDD.md`.

**Inputs:**
- Prior `SESSION_DIR/thesis.md`
- Fresh `SESSION_DIR/raw/revisit-YYYYMMDD-*.md` files

**Output structure:**

```markdown
# Drift Check — {TICKER} — {YYYY-MM-DD}

*Prior thesis generated {prior-timestamp}. Days elapsed: {N}.*

## Price & Metric Delta

- Prior price: {$X} on {date}
- Current price: {$Y} on {date} (source: `raw/revisit-{date}-yahoo.md`)
- Delta: {pct}%
- Prior key metrics vs current (table):
  | Metric | Prior | Current | Delta |

## Narrative Delta

3–5 bullet points summarizing material news since the prior thesis. Date each one.

## Kill Switch Verdicts

For each kill switch from the prior thesis, output:

**Kill switch:** "{original text}"
**Verdict:** Triggered / Not triggered / Inconclusive (no data)
**Evidence:** {citation with current value from revisit raw files}
**Margin:** {how close was the trigger? e.g. "gross margin is 58.2%, threshold was 55%, margin of safety 3.2pp"}

## Recommended Next Step

One of:
- "No action — thesis intact, all kill switches safe" (if none triggered and margin is comfortable)
- "Watch — {specific kill switch} is within {X}pp of threshold; revisit in {M} weeks"
- "Re-evaluate — kill switch {name} has triggered; consider re-running /stockwiz-bear for a fresh adversarial view"
- "Stale — prior thesis is {N} days old; consider a fresh /stockwiz run"
```

### Mode: `reconcile`
After `devils-advocate` writes its stress-test pass, merge its feedback into the existing `thesis.md` **without overwriting or softening the original claims**. Append, don't rewrite.

**Inputs:**
- `SESSION_DIR/thesis.md` — the existing thesis (from `full` mode)
- `SESSION_DIR/analysis/devils-advocate.md` — the adversarial pass

**What you do:**

1. Read both files.

2. Identify which of devils-advocate's "Weakest Claims" are materially correct — meaning the devils-advocate found a specific inverse condition that is already visible in source data, not a rhetorical attack. A "material" finding has:
   - A specific counter-data point (not just a contrary interpretation)
   - Citation to a source file or a fresh fetch URL
   - A measurable implication

3. Identify which of devils-advocate's kill-switch-adequacy verdicts are material. Verdicts of `vague`, `slow`, `missing-trigger`, or `wrong-metric` on a kill switch, WITH a concrete rewrite proposed, are material.

4. Append to `thesis.md` a new section `## Adjustments After Stress Test` (just before `## Unknowns`). Structure:
   ```markdown
   ## Adjustments After Stress Test

   The devils-advocate pass raised N material issues, summarized below. The original claims above are preserved verbatim; adjustments are listed here separately so the reader can see the delta.

   ### Material weakening of claims

   - **Bull claim #2** (gross margin defensibility): devils-advocate notes that the FY2025 → FY2026 gross margin step-down of 3.92pp already exists in stockanalysis.com's own data cited by the claim itself. The bull claim's certainty on "pricing power is converting into durable FCF" is reduced accordingly — a 6.07pp margin of safety to the kill switch (71.07% vs 65% threshold) is narrower than the bull framing implied.

   ### Kill switch tightening

   - **Original:** "Gross margin below 65% for two consecutive quarters"
     **Rewrite:** "Gross margin below 67% in any single quarter OR below 65% for two consecutive quarters"
     **Reason:** devils-advocate noted that a single-quarter sub-67% print would already invalidate the "glide path" framing and should not wait for a second confirming quarter.

   ### Unresolved adversarial points

   - devils-advocate raised {issue X} but it was not adopted because {specific reason — "no counter-data cited" or "attack is a reinterpretation not a contradiction"}
   ```

5. Do NOT modify the bull/base/bear case bodies directly. The reader should be able to see the original thesis and the adjustments side by side — that's the audit trail.

6. If devils-advocate raised no material issues (all verdicts `adequate`, all weakest claims just alternative interpretations), write a short `## Adjustments After Stress Test` noting that the adversarial pass found the thesis internally consistent, and move on.

### Mode: `implicit`
Extract the latent thesis from an existing analysis. Write `SESSION_DIR/analysis/implicit-thesis.md`. Short and sharp — this is used by `peer-scout` to find alternative expressions.

**Inputs:**
- `SESSION_DIR/thesis.md`
- `SESSION_DIR/analysis/fundamental.md`

**Output structure:**

```markdown
# Implicit Thesis — {TICKER}

**Sector / sub-industry.** {one phrase}

**Underlying macro or cycle driver.** {one phrase — e.g. "AI capex acceleration", "rate-cut re-rating of duration", "copper supply deficit"}

**Key quality attribute being expressed.** {one phrase — e.g. "pricing power via near-monopoly", "FCF conversion stability", "margin expansion from scale"}

**Primary fragility.** {one phrase — e.g. "customer concentration in Big Tech", "leverage to single commodity", "management execution dependency"}

**One-sentence distilled thesis.** {the full implicit thesis in a single sentence that a peer-scout could use to find alternative expressions}
```

## Hard rules (apply to all modes)

1. **Every factual claim gets a citation** — `[raw/source-file.md#anchor]` or `[analysis/skill.md]`. No uncited numbers.
2. **Every number is dated.** "Revenue $50.1B (FY2025)" not "Revenue $50.1B".
3. **No banned phrases** — you are a layer above the compliance pass but you should not generate language that will trigger rewrites. Avoid "recommend", "buy", "sell", "guaranteed", "undervalued", "should buy". Use "analysis suggests", "consider", "re-evaluate", "trading below framing", "may consider".
4. **Bull and bear get equal rigor.** If the bull case has 3 cited claims and a timeline, the bear case must too. Asymmetric treatment is itself a bias.
5. **Base case is NOT the average.** It is its own scenario. A good test: if you removed the bull and bear cases, would the base case still make sense as a standalone thesis? If not, rewrite.
6. **Unknowns are mandatory.** A thesis without a real Unknowns section is dishonest about its confidence.
7. **Kill switches are measurable.** If a reader cannot check the switch with one WebFetch on Finviz or Yahoo, it is not a kill switch, it is a vibe.

## Pointers into the reference file

For examples of good and bad kill switches by thesis archetype (growth, value, cyclical, turnaround), see `references/kill-switch-examples.md` — but that file is Phase 2+ content. In Phase 1, keep kill switches grounded by following the "metric + threshold + time window" structure above.
