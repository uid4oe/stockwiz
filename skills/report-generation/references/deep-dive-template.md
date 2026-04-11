# Deep-dive template (v0.3 — insights-first)

The primary stockwiz HTML artifact. Used by `/stockwiz <ticker>` and output to `SESSION_DIR/report.html`.

**v0.3 redesign rationale.** The earlier template was a long-form research brief — hero at the top, then a sequence of substantive sections, with the reader expected to scroll through ~100KB of HTML to understand the thesis. Field use revealed that most readers want a 20-second scan first, then details on demand. This template puts a dense TL;DR above the fold and progressively discloses everything else via native `<details>`/`<summary>` — zero JavaScript, print-friendly, scannable.

## Overall shape

```
┌────────────────────────────────────────────┐
│ Hero              [1 viewport height]      │
│ Key metrics strip                          │
│                                            │
│ THE TL;DR                                  │
│   ▲ Bull · ◆ Base · ▼ Bear (1 line each)  │
│   ● Key insight                            │
│   ⟐ Closest kill switch                    │
│   ? Biggest unknown                        │
├────────────────────────────────────────────┤ [fold]
│                                            │
│ ▾ Three Cases (full, open by default)     │
│ ▸ Kill Switches                           │
│ ▸ Fundamentals                            │
│ ▸ Sentiment                               │
│ ▸ Peers                                   │
│ ▸ Risk                                    │
│ ▸ Assumption Ledger                       │
│ ▸ Unknowns                                │
│ ▸ Sources                                 │
│ ▸ Adversarial Pass                        │
├────────────────────────────────────────────┤
│ Disclaimer (always visible, not collapsible)│
└────────────────────────────────────────────┘
```

Target size for the above-the-fold block: ~700-800px of visible content on a 1440px-wide 1024px-tall laptop display, so the entire TL;DR is visible without scroll on first load.

## Document skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>stockwiz — {TICKER} — Research Brief</title>
  <style>
    /* inline base-styles.css here */
  </style>
</head>
<body>
  <div class="stockwiz-container">
    {hero}
    {metrics-strip}
    {tldr}
    {details-three-cases}       <!-- open by default -->
    {details-kill-switches}
    {details-fundamentals}
    {details-sentiment}
    {details-peers}
    {details-risk}
    {details-assumption-ledger}
    {details-unknowns}
    {details-sources}
    {details-adversarial}
    {disclaimer}                <!-- always visible, loaded verbatim -->
  </div>
</body>
</html>
```

## Section specs

### 1. Hero

```html
<header class="stockwiz-hero">
  <p class="eyebrow">stockwiz research brief · {horizon} horizon · generated {human date}</p>
  <h1>
    <span class="ticker">NVDA</span>
    <span class="sep">—</span>
    <span class="company-name">NVIDIA Corporation</span>
  </h1>
  <div class="price-row">
    <span class="price">$188.63</span>
    <span class="meta">as of {fetch timestamp}</span>
    <span class="meta">·</span>
    <span class="meta">{sector} / {industry}</span>
  </div>
</header>
```

Keep it spare. No sparkline in the hero (sparklines move into the Fundamentals details section). No tagline (the TL;DR below carries the summary). Just ticker, name, price, as-of, sector.

### 2. Key metrics strip

Six metrics in a single row. Each is a label + value card, divided by thin rules. Monospace for tabular numerics.

```html
<section class="stockwiz-metrics">
  <div class="metric">
    <div class="label">Market cap</div>
    <div class="value">$4.58T<sup><a href="#src-2">2</a></sup></div>
  </div>
  <div class="metric">
    <div class="label">P/E (trailing)</div>
    <div class="value">38.5</div>
  </div>
  <div class="metric">
    <div class="label">P/E (forward)</div>
    <div class="value">22.7</div>
  </div>
  <div class="metric">
    <div class="label">FCF yield</div>
    <div class="value">2.1%</div>
  </div>
  <div class="metric">
    <div class="label">Revenue YoY</div>
    <div class="value">+65.5%</div>
  </div>
  <div class="metric">
    <div class="label">ROIC</div>
    <div class="value">126.3%</div>
  </div>
</section>
```

Always six metrics. Pick from: market cap, P/E T, P/E F, FCF yield, revenue growth YoY, ROIC, ROE, net debt/cash, gross margin, operating margin. For a bank, swap in net interest margin / efficiency ratio. For a REIT, swap in FFO yield / cap rate. Default picks above.

Each metric has a `<sup>` citation to the Sources section. Values that are "—" if unavailable; never fabricated.

### 3. THE TL;DR

The centerpiece. Above the fold. Five atoms.

```html
<section class="stockwiz-tldr">
  <h2>The TL;DR</h2>

  <!-- Compact three cases: one-line headlines only -->
  <div class="stockwiz-cases-compact">
    <div class="case">
      <div class="case-label">
        <span class="case-mark">▲</span>
        <span>Bull</span>
      </div>
      <p class="case-headline">{Bull headline from thesis.md, one sentence, ends with period.}</p>
    </div>
    <div class="case">
      <div class="case-label">
        <span class="case-mark">◆</span>
        <span>Base</span>
      </div>
      <p class="case-headline">{Base headline}</p>
    </div>
    <div class="case">
      <div class="case-label">
        <span class="case-mark">▼</span>
        <span>Bear</span>
      </div>
      <p class="case-headline">{Bear headline}</p>
    </div>
  </div>

  <!-- Three callouts: insight, kill switch, unknown -->
  <div class="stockwiz-callouts">

    <div class="callout insight">
      <div class="callout-label">
        <span class="callout-mark">●</span>
        <span>Key insight</span>
      </div>
      <p class="callout-body">{The ONE most striking thing — see "How to pick the key insight" below.}<sup>...</sup></p>
    </div>

    <div class="callout kill-switch">
      <div class="callout-label">
        <span class="callout-mark">⟐</span>
        <span>Closest kill switch</span>
      </div>
      <p class="callout-body">{The kill switch with the smallest margin of safety, from thesis.md § Kill Switches. Written as a single sentence with metric + threshold + time window.}</p>
      <p class="margin-bar">Current: <span class="current">{current value}</span> · Trigger: {threshold} · Margin: {pct or pp} to go</p>
    </div>

    <div class="callout">
      <div class="callout-label">
        <span class="callout-mark">?</span>
        <span>Biggest unknown</span>
      </div>
      <p class="callout-body">{The most material item from thesis.md § Unknowns, as a single sentence.}</p>
    </div>

  </div>
</section>
```

**IMPORTANT: ordering of the three cases.** Even in the compact TL;DR view, the three cases render in **hash-shuffled order** (same as the full cases section below) to prevent anchoring. Use the same FNV-1a hash of the ticker mod 6 permutation as the full section — **the compact TL;DR and the full details section must use the SAME order** for visual consistency.

Compact case bodies are **one sentence each** — typically the Headline from thesis.md's corresponding case section, sometimes truncated or gently rewritten for length. If the thesis uses "NVIDIA is the only vertically-integrated vendor...", the compact TL;DR uses the same sentence or a tight paraphrase.

### How to pick the "key insight"

> **Non-determinism note.** The key-insight selection is explicitly the most **non-deterministic** output in the report. Two runs on the same ticker may legitimately pick different insights if multiple heuristics apply. This is a deliberate trade-off: an LLM-curated insight is more useful than a deterministic one ("pick the largest metric by magnitude") because the LLM can factor in context the heuristic can't. The cost is reduced reproducibility — if the user cares about stable insights across runs, they should save the session workspace and reference it directly rather than expecting subsequent runs to match. Every other output in stockwiz (numbers, citations, sections) IS reproducible; the insight picking is the one exception.

The report-writer reads `analysis/fundamental.md` and `analysis/sentiment.md` and picks **one** most striking observation that a reader who is unfamiliar with the stock would find surprising or defining. Heuristics in priority order:

1. **Anomalous quality metric**: if ROIC, ROE, or FCF margin is in the top 1% of all US equities (usually means >50% ROIC, >50% ROE, >40% FCF margin), say so with specifics
2. **Step-change in growth**: if revenue or EPS multiplied by more than 3x in the most recent 5 years
3. **Capital structure surprise**: if the company has net cash equal to 10%+ of market cap (rare and material)
4. **Margin trajectory inflection**: if gross margin has compressed or expanded by more than 3pp in the most recent year
5. **Historical outlier**: if a 5Y CAGR on any line is more than 40%
6. **Alternative-view standout**: if SWS Snowflake Future score is 6/6 or Value is 1/6 (extremes)

The insight should be one sentence, cite a source file, and avoid advisory language. Example: "ROIC of 126% is among the highest in US equities, reflecting capacity-constrained demand more than competitive moat — FY2027 readings will show whether this is durable."

### How to pick the "closest kill switch"

Read `thesis.md` § Kill Switches. Each kill switch has a current reading and a trigger threshold. Compute the percentage margin (or percentage-point margin for ratios) between current and trigger. Pick the one with the smallest margin.

Format the one-liner as: "{Metric} {below/above} {threshold} {time window}". Example: "Gross margin below 65% for two consecutive quarters (currently 71.07%, 6.07pp of margin before trigger)."

If no kill switch has a computed margin (e.g. all are structural like "short float > 5%" without a current value), pick the most specific and measurable one.

### How to pick the "biggest unknown"

Read `thesis.md` § Unknowns. The list is already prioritized by the thesis-discipline skill (most material first). Take the first item, rewrite it as a single sentence if it's longer. Example: "Customer concentration of data-center revenue (top 4 hyperscalers) is not disclosed in current sources and materially affects the bear case."

## Below the fold — progressive disclosure

Everything below the TL;DR uses `<details>`/`<summary>` with the `stockwiz-section` class. The pattern:

```html
<details class="stockwiz-section" open>
  <summary>
    <span class="section-title">Three Cases</span>
    <span class="section-abstract">Full bull / base / bear with supporting claims, disconfirmers, and timelines.</span>
  </summary>
  <div class="details-body">
    <!-- section content -->
  </div>
</details>
```

The first `<details>` (Three Cases) is `open` by default. All others are collapsed by default.

Each summary has a **one-line abstract** that helps the reader decide whether to expand. Compute these at report-writer time from the section content:

- Three Cases: "Full bull / base / bear with supporting claims, disconfirmers, and timelines."
- Kill Switches: "{N} measurable triggers, tightest is {metric} with {margin} to go."
- Fundamentals: "Revenue {growth}, FCF margin {margin}, ROIC {roic}, net {cash|debt} {amount}."
- Sentiment: "{N} SWS risks flagged, {M} analysts covering, insider activity {net buy|net sell|flat}."
- Peers: "{N} peers via SWS competitor snowflakes, target scores {value}/6 value and {future}/6 future."
- Risk: "Beta {beta}, {pct} off 52w high, {n} concentration risks flagged."
- Assumption Ledger: "{N} forward-looking assumptions with source and sensitivity."
- Unknowns: "{N} items the analysis could not determine from available sources."
- Sources: "{ok}/{total} succeeded — {list of successful source slugs}."
- Adversarial Pass: "{N} weakest claims ranked, {M} kill switches tightened."

The abstracts are one-line summaries, not teasers. They should give the reader enough to decide "yes I care about this section" or "no I don't".

## Collapsible section 1 — Three Cases (open by default)

```html
<details class="stockwiz-section" open>
  <summary>
    <span class="section-title">Three Cases</span>
    <span class="section-abstract">Full bull / base / bear with supporting claims, disconfirmers, and timelines.</span>
  </summary>
  <div class="details-body">
    <div class="stockwiz-cases-full">
      <article class="case">
        <h4>{Bull / Base / Bear}</h4>
        <p class="headline">{one-sentence headline}</p>
        <ul>
          <li>{claim 1 with citation}</li>
          <li>{claim 2 with citation}</li>
          <li>{claim 3 with citation}</li>
        </ul>
        <h4>Disconfirmers</h4>
        <ul>
          <li>{disconfirmer 1}</li>
          <li>{disconfirmer 2}</li>
        </ul>
        <p class="timeline"><strong>6m:</strong> … · <strong>12m:</strong> … · <strong>24m:</strong> …</p>
      </article>
      <article class="case">…</article>
      <article class="case">…</article>
    </div>
  </div>
</details>
```

Same hash-shuffled order as the compact TL;DR above. Visual weight equal (grid does this).

## Collapsible section 2 — Kill Switches

```html
<details class="stockwiz-section">
  <summary>
    <span class="section-title">Kill Switches</span>
    <span class="section-abstract">{N} measurable triggers, tightest is {metric} with {margin} to go.</span>
  </summary>
  <div class="details-body">
    <div class="stockwiz-kill-switches">
      <div class="switch">
        <div class="trigger">{Kill switch 1 as a sentence: metric + threshold + time window}<sup>…</sup></div>
        <div class="margin">Current <span class="current">{value}</span> · Trigger {threshold} · {margin} to go</div>
      </div>
      <!-- more switches, most-fragile-first -->
    </div>
  </div>
</details>
```

Sort by margin of safety ascending (closest to trigger first).

## Collapsible section 3 — Fundamentals

```html
<details class="stockwiz-section">
  <summary>
    <span class="section-title">Fundamentals</span>
    <span class="section-abstract">Revenue {growth}, FCF margin {fcf}, ROIC {roic}, net {cash|debt} {amount}.</span>
  </summary>
  <div class="details-body">
    <h3>Valuation framing</h3>
    <p>{one paragraph from analysis/fundamental.md § Valuation Framing}</p>
    <table class="stockwiz-table">…multiples table…</table>

    <h3>Quality</h3>
    <p>{quality paragraph}</p>
    <svg class="stockwiz-sparkline" viewBox="0 0 300 40">…ROIC trend…</svg>
    <div class="stockwiz-sparkline-label"><span>FY-4</span><span>TTM</span></div>

    <svg class="stockwiz-sparkline" viewBox="0 0 300 40">…gross margin trend…</svg>
    <div class="stockwiz-sparkline-label"><span>FY-4</span><span>TTM</span></div>

    <h3>Growth</h3>
    <p>{growth paragraph}</p>
    <table class="stockwiz-table">…5Y revenue/EPS/FCF table…</table>

    <h3>Capital structure</h3>
    <ul>
      <li>Total debt: $X.XB<sup>…</sup></li>
      <li>Cash: $X.XB</li>
      <li>Net {cash|debt}: $X.XB</li>
      <li>Interest coverage: {x}×</li>
      <li>D/E: {x}</li>
    </ul>

    <h3>Ownership</h3>
    <p>Insider {x}%, institutional {y}%, short float {z}%.</p>
  </div>
</details>
```

## Collapsible section 4 — Sentiment

```html
<details class="stockwiz-section">
  <summary>
    <span class="section-title">Sentiment</span>
    <span class="section-abstract">{N} SWS risks flagged, {M} analysts covering, insider activity {shape}.</span>
  </summary>
  <div class="details-body">
    <p>{net sentiment read paragraph from analysis/sentiment.md}</p>

    <h3>Simply Wall St flagged risks</h3>
    <ul>
      <li><q>{SWS risk 1 verbatim}</q><sup>…</sup></li>
      <li><q>{SWS risk 2 verbatim}</q></li>
      <li><q>{SWS risk 3 verbatim}</q></li>
    </ul>

    <h3>Insider & short</h3>
    <ul>
      <li>Insider ownership: {x}% · Net 3m transactions: {y}</li>
      <li>Short float: {z}% · Short ratio: {r} days</li>
    </ul>

    <h3>Analyst positioning</h3>
    <ul>
      <li>{N} analysts covering · target mean ${x} ({pct}% from current)</li>
      <li>Distribution: <q>Strong Buy</q> {n}, <q>Buy</q> {n}, Hold {n}, <q>Sell</q> {n}, <q>Strong Sell</q> {n}</li>
    </ul>

    <h3>Recent headlines</h3>
    <ul>
      <li><strong>{date}</strong> — <q>{headline from Google News or SA}</q><sup>…</sup></li>
      <!-- up to 8 items, most recent first -->
    </ul>

    <h3>Conflicting signals</h3>
    <p>{conflicts section from sentiment.md, preserved not resolved}</p>
  </div>
</details>
```

## Collapsible sections 5–10 — Peers, Risk, Assumption Ledger, Unknowns, Sources, Adversarial Pass

Each follows the same pattern: a `<summary>` with title + abstract, a `<div class="details-body">` with the section content. Detailed content structure matches the earlier version of this template (comp tables, risk bullets, assumption-ledger table, unknowns list, sources ordered list, adversarial appendix with weakest claims / opposing narrative / kill switch adequacy).

The Adversarial Pass section uses `class="stockwiz-adversarial"` on the `<div class="details-body">` and includes a small italic intro paragraph explaining the adversarial protocol.

## Disclaimer (always visible, not collapsible)

```html
<!-- Loaded verbatim from ${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/disclaimer.html -->
<!-- Substitute {version}, {timestamp}, {session-id} -->
<footer class="stockwiz-disclaimer">
  <p><strong>Analytical tool, not investment advice.</strong> stockwiz is a
  research assistant that synthesizes publicly available information. Nothing
  in this document is an offer, solicitation, or recommendation to buy, sell,
  or hold any security. Figures are drawn from public sources at the fetch
  time shown and may be stale, incomplete, or in error. Forward-looking
  statements are analytical framings, not predictions. You are responsible
  for your own investment decisions and should consult a licensed financial
  professional before acting on any information in this document.</p>
  <p class="footnote">Generated by stockwiz {version} on {timestamp}. Session: {session-id}.</p>
</footer>
```

**Note:** the disclaimer sits OUTSIDE any `<details>` — it is not collapsible. It is always the last element before `</body>`. The compliance pass explicitly exempts the contents of `class="stockwiz-disclaimer"` so the legal boilerplate `"recommendation to buy, sell, or hold"` passes through unchanged.

## Report size targets

- **full (10 sources, all analyses, adversarial pass)**: 60–150KB of HTML, inclusive of inline CSS and SVG sparklines
- **raw-only fallback (analyses missing)**: 30–50KB with thin sections
- **Hard max**: 500KB — anything larger indicates binary content leaking in

## Graceful degradation rule

If any analysis file is missing, render a thin fallback for that section from raw files directly. The report must always be emittable as long as `thesis.md` exists.

For the TL;DR's three callouts:
- **Key insight**: if `analysis/fundamental.md` is missing, pick from raw data directly (Finviz ROIC, Stockanalysis 5Y CAGR, etc.)
- **Closest kill switch**: always available from `thesis.md` § Kill Switches
- **Biggest unknown**: always available from `thesis.md` § Unknowns

The TL;DR is always fully populated. The fold-below sections may be thin.

## What makes this "insights-first minimal"

1. **Above the fold is a 5-atom summary.** Hero, 6 metrics, 3 one-sentence cases, 3 callouts. Nothing else. A 20-second scan gets the complete high-level picture.
2. **No decorative flourishes.** No purple gradients. No emoji. No traffic-light colors. No hero sparklines. Just typography and whitespace.
3. **Progressive disclosure.** Readers who want details click to expand. Readers who don't can print the page and get everything inline.
4. **Equal visual weight for the three cases**, always — both in the compact TL;DR and in the full details section. Hash-shuffled to prevent anchoring.
5. **Key insight is curated, not extracted.** The report-writer picks ONE striking fact from the analyses using the heuristic above. Quality over quantity.
6. **Closest kill switch shows margin of safety explicitly.** The reader knows not just what could break the thesis but how close it is to breaking.
7. **Biggest unknown is front-loaded.** The thing the analysis can't tell you is as important as the things it can. It goes above the fold, not hidden in a unknowns section.
