# Deep-dive template

The primary stockwiz HTML artifact. Used by `/stockwiz <ticker>` and output to `SESSION_DIR/report.html`.

This template is the spec. The `report-writer` agent follows it section by section, fills it with content from the session workspace, runs the compliance pass, and writes the file.

## Overall structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>stockwiz — {TICKER} — Research Brief</title>
  <!-- Optional single Google Fonts link for display face, font-display:swap -->
  <link href="https://fonts.googleapis.com/css2?family=..." rel="stylesheet">
  <style>
    /* inline base-styles.css here */
    /* plus any frontend-design overrides */
  </style>
</head>
<body>
  <div class="stockwiz-container">
    {hero}
    {snapshot-strip}
    {three-cases}
    {kill-switches}
    {fundamentals}         <!-- Phase 2+ -->
    {sentiment}            <!-- Phase 2+ -->
    {peers}                <!-- Phase 2+ -->
    {risk}                 <!-- Phase 2+ -->
    {assumption-ledger}    <!-- Phase 2+ -->
    {unknowns}
    {sources}
    {adversarial-appendix} <!-- Phase 2+ -->
    {disclaimer}           <!-- loaded verbatim from assets/disclaimer.html -->
  </div>
</body>
</html>
```

## Section specs

### 1. Hero

```html
<header class="stockwiz-hero">
  <p class="stockwiz-meta">Research brief · {horizon} horizon · generated {human date}</p>
  <h1>{TICKER} — {Company Name}</h1>
  <p class="tagline">{Base case headline from thesis.md — one sentence}</p>
  <div class="hero-row">
    <span class="price">{current price}</span>
    <span class="price-meta">as of {fetch timestamp}</span>
    <!-- inline SVG sparkline, ~250d if available, otherwise labeled "coarse" -->
    <svg class="stockwiz-sparkline" viewBox="0 0 300 60">...</svg>
  </div>
</header>
```

The tagline should be the **Base Case headline** extracted from `thesis.md`, not bull or bear. It represents the central scenario.

The sparkline is inline SVG. For Phase 1, if only end-of-period values are available, draw a 2–5 point sparkline and label it "coarse — based on available data points". Do not fabricate daily data.

### 2. Snapshot strip

```html
<section class="stockwiz-snapshot">
  <div class="card">
    <div class="label">Market cap</div>
    <div class="value">{value}<sup><a href="#src-2">2</a></sup></div>
  </div>
  <div class="card">
    <div class="label">P/E (trailing)</div>
    <div class="value">{value}<sup><a href="#src-2">2</a></sup></div>
  </div>
  <!-- Forward P/E, FCF yield, revenue growth YoY, beta -->
</section>
```

Cards: market cap, P/E (T), P/E (F), FCF yield, revenue growth YoY, beta. Each card has a `<sup>` citation to the Sources section. If a value is unavailable from the fetched sources, write "—" in the value slot, not a fabricated number.

### 3. The three cases — the centerpiece

This is the most important section. Three equal-width columns for Bull / Base / Bear cases. **The order is deterministically shuffled per ticker** (using a stable hash of the ticker) to avoid anchoring bias — for the same ticker the order is always the same, but different tickers differ.

```html
<section class="stockwiz-cases-section">
  <h2>Three cases</h2>
  <div class="stockwiz-cases">
    <article class="case">
      <h3>{case name — Bull / Base / Bear}</h3>
      <p class="headline">{one-sentence headline}</p>
      <ul>
        <li>{claim 1}<sup><a href="#src-N">N</a></sup></li>
        <li>{claim 2}<sup><a href="#src-N">N</a></sup></li>
        <li>{claim 3}<sup><a href="#src-N">N</a></sup></li>
      </ul>
      <h4>Disconfirmers</h4>
      <ul>
        <li>{disconfirmer 1}</li>
        <li>{disconfirmer 2}</li>
      </ul>
      <p class="timeline"><strong>6m:</strong> ... · <strong>12m:</strong> ... · <strong>24m:</strong> ...</p>
    </article>
    <article class="case">...</article>
    <article class="case">...</article>
  </div>
</section>
```

### Hash-based ordering

Use the following pattern (described in prose — the report-writer agent can implement it in its composition step):

- Compute a stable 32-bit hash of the ticker (simple FNV-1a is fine; `sum of char codes mod 6` also works).
- The hash modulo 6 picks one of six permutations of [Bull, Base, Bear]:
  - 0 → Bull, Base, Bear
  - 1 → Bull, Bear, Base
  - 2 → Base, Bull, Bear
  - 3 → Base, Bear, Bull
  - 4 → Bear, Bull, Base
  - 5 → Bear, Base, Bull
- Render the three `<article>` elements in that order.

The CSS in `base-styles.css` uses `.stockwiz-cases` as a grid with `grid-template-columns: repeat(3, 1fr)`, so equal visual weight is enforced at the layout level regardless of order.

### 4. Kill switches

```html
<section class="stockwiz-kill-switches">
  <h3>Kill switches</h3>
  <ul>
    <li>{kill switch 1 — measurable trigger + source}<sup><a href="#src-N">N</a></sup></li>
    <li>{kill switch 2 — measurable trigger + source}<sup><a href="#src-N">N</a></sup></li>
    <!-- as many as the thesis-discipline skill produced -->
  </ul>
</section>
```

Kill switches span the full width under the three cases because they threaten all three. Each one is a specific threshold + time window + source, straight from `thesis.md`.

### 5. Fundamentals

In Phase 2+, render the full section from `SESSION_DIR/analysis/fundamental.md`. Structure:

```html
<section class="stockwiz-fundamentals">
  <h2>Fundamentals</h2>

  <!-- Valuation Framing -->
  <h3>Valuation Framing</h3>
  <p>{one-to-two paragraph DCF framing from the Valuation Framing section of fundamental.md}. The market appears to embed {X} assumption, which given {Y} implies {Z}.<sup><a href="#src-N">N</a></sup></p>
  <table class="stockwiz-table">
    <tr><th>Multiple</th><th>Trailing</th><th>Forward</th><th>5Y Avg</th></tr>
    <tr><td>P/E</td><td>{n}</td><td>{n}</td><td>{n}</td></tr>
    <tr><td>P/S</td><td>{n}</td><td>—</td><td>{n}</td></tr>
    <tr><td>EV/EBITDA</td><td>{n}</td><td>—</td><td>{n}</td></tr>
  </table>

  <!-- Quality -->
  <h3>Quality</h3>
  <p>{Quality paragraph from fundamental.md}</p>
  <div class="stockwiz-metric-grid">
    <div>
      <div class="label">ROIC trend</div>
      <svg class="stockwiz-sparkline" viewBox="0 0 300 60">...</svg>
      <div class="footnote">{first year} → {last year}</div>
    </div>
    <div>
      <div class="label">Gross margin trend</div>
      <svg class="stockwiz-sparkline" viewBox="0 0 300 60">...</svg>
    </div>
    <div>
      <div class="label">Operating margin trend</div>
      <svg class="stockwiz-sparkline" viewBox="0 0 300 60">...</svg>
    </div>
  </div>

  <!-- Growth -->
  <h3>Growth</h3>
  <p>{Growth paragraph}</p>
  <svg class="stockwiz-sparkline" viewBox="0 0 400 80">
    <!-- revenue trajectory with first/last data points labeled -->
  </svg>
  <table class="stockwiz-table">
    <tr><th>Metric</th><th>FY-4</th><th>FY-3</th><th>FY-2</th><th>FY-1</th><th>TTM</th><th>5Y CAGR</th></tr>
    <tr><td>Revenue ($B)</td>...</tr>
    <tr><td>EPS (diluted)</td>...</tr>
    <tr><td>FCF ($B)</td>...</tr>
  </table>

  <!-- Capital Structure -->
  <h3>Capital Structure</h3>
  <p>{Capital structure paragraph}</p>
  <ul>
    <li>Total debt: $X.XB<sup><a href="#src-N">N</a></sup></li>
    <li>Cash: $X.XB</li>
    <li>Net cash/debt: $X.XB</li>
    <li>Interest coverage: X.X×</li>
    <li>Debt/Equity: X.XX</li>
  </ul>

  <!-- Ownership -->
  <h3>Ownership</h3>
  <p>Insider ownership {x}%, institutional {y}%, short float {z}%.<sup>...</sup></p>
</section>
```

If the `fundamental-analysis` skill did not produce a file (Phase 1.5 fallback), render a thin version with just the snapshot strip data and note "Detailed fundamental analysis not produced in this session — see thesis.md for core figures."

### 6. Sentiment

In Phase 2+, render from `SESSION_DIR/analysis/sentiment.md`. Structure:

```html
<section class="stockwiz-sentiment">
  <h2>Sentiment</h2>

  <!-- Net sentiment read -->
  <p class="lead">{Net Sentiment Read paragraph from sentiment.md}</p>

  <div class="stockwiz-sentiment-grid">
    <!-- Insider activity -->
    <div>
      <h4>Insider activity</h4>
      <ul>
        <li>Insider ownership: {x}%<sup>...</sup></li>
        <li>Net 3m insider transactions: {y}</li>
        <li>{SWS insider flag if present, wrapped in <q>}</li>
      </ul>
    </div>

    <!-- Short interest -->
    <div>
      <h4>Short interest</h4>
      <ul>
        <li>Short % of float: {x}%</li>
        <li>Short ratio: {y} days</li>
        <li>Trend vs prior month: {delta}</li>
      </ul>
    </div>

    <!-- Analyst positioning -->
    <div>
      <h4>Analyst positioning</h4>
      <ul>
        <li>{N} analysts covering</li>
        <li>Average target: ${x} ({pct}% from current)</li>
        <li>Finviz analyst recom numeric: {n}/5 (lower = more positive)</li>
        <li>Distribution: <q>Strong Buy</q> {n}, <q>Buy</q> {n}, Hold {n}, <q>Sell</q> {n}, <q>Strong Sell</q> {n}</li>
      </ul>
    </div>
  </div>

  <!-- SWS risks (if present — high signal) -->
  <h4>Simply Wall St flagged risks</h4>
  <ul>
    <li><q>{SWS risk 1 verbatim}</q><sup>...</sup></li>
    <li><q>{SWS risk 2 verbatim}</q></li>
    <li><q>{SWS risk 3 verbatim}</q></li>
  </ul>

  <!-- Recent news timeline (teaser titles only if no news sources wired) -->
  <h4>Recent headlines (titles only)</h4>
  <ul class="news-timeline">
    <li><strong>{date}</strong> — <q>{headline}</q><sup>...</sup></li>
    <li>...</li>
  </ul>

  <!-- Conflicting signals -->
  <h4>Conflicting signals</h4>
  <p>{From the Conflicting Signals section of sentiment.md — preserved disagreements between sources}</p>
</section>
```

### 7. Peers

Render from `SESSION_DIR/analysis/peer-comp.md`. Structure:

```html
<section class="stockwiz-peers">
  <h2>Peers</h2>
  <p>Peers sourced from {SWS competitor snowflakes / stockanalysis.com peer tagging}.</p>

  <table class="stockwiz-table stockwiz-comp-table">
    <tr><th>Metric</th><th>{TARGET}</th><th>{PEER1}</th><th>{PEER2}</th><th>Direction</th></tr>
    <!-- rows as defined in peer-comp.md comp table, cells with .good/.watch classes
         for direction-of-goodness coloring -->
  </table>

  <p class="stockwiz-caveat"><strong>Caveats.</strong> {caveats paragraph from peer-comp.md — business mix differences, time-period differences, size differences, etc.}</p>
</section>
```

### 8. Risk

Render from `SESSION_DIR/analysis/risk.md`. Structure:

```html
<section class="stockwiz-risk">
  <h2>Risk</h2>

  <h4>Volatility & Beta</h4>
  <ul>
    <li>5Y beta: {x}<sup>...</sup></li>
    <li>Weekly volatility: {y}%</li>
    <li>Monthly volatility: {z}%</li>
    <li>ATR (14d): ${a}</li>
  </ul>

  <h4>Drawdown profile</h4>
  <p>{drawdown narrative from risk.md}</p>
  <ul>
    <li>52w high: ${x} ({pct}% from current)</li>
    <li>52w low: ${y} ({pct}% from current)</li>
    <li>50-day MA: ${z}</li>
    <li>200-day MA: ${w}</li>
  </ul>

  <h4>Concentration risks</h4>
  <p>{concentration paragraph from risk.md — or Phase 2 limitation note if 10-K prose wasn't parsed}</p>

  <h4>Capital-structure risks</h4>
  <p>{cap-structure summary}</p>

  <h4>Tail risks</h4>
  <ul>
    <li>{SWS risk 1 verbatim, wrapped in <q>}</li>
    <li>{SWS risk 2}</li>
    <li>{SA article teaser that flags a tail concern, if relevant}</li>
  </ul>
</section>
```

### 9. Assumption ledger

Render the table from `fundamental-analysis`'s `## Assumption Ledger` section:

```html
<section class="stockwiz-assumptions">
  <h2>Assumption Ledger</h2>
  <p>Every forward-looking claim in this analysis, with its source and sensitivity. If the reader disagrees with an assumption, they know exactly which domino to push.</p>
  <table class="stockwiz-table">
    <tr><th>Assumption</th><th>Value</th><th>Source</th><th>Sensitivity</th></tr>
    <!-- rows from fundamental.md -->
  </table>
</section>
```

### 10. Unknowns

```html
<section>
  <h2>Unknowns</h2>
  <ul>
    <li>{unknown 1}</li>
    <li>{unknown 2}</li>
    <!-- from thesis.md's Unknowns section -->
  </ul>
</section>
```

In Phase 1, the thesis-discipline skill still produces an Unknowns list (it's mandatory), so render it.

### 11. Sources

```html
<section class="stockwiz-sources">
  <h2>Sources</h2>
  <ol>
    <li id="src-1">SEC EDGAR 10-K — {URL}. Retrieved {date}.</li>
    <li id="src-2">Yahoo Finance — {URL}. Retrieved {date}.</li>
    <li id="src-3">Finviz snapshot — {URL}. Retrieved {date}.</li>
    <!-- plus any other sources that succeeded, in fetch order -->
  </ol>
  <p class="footnote">{N} sources succeeded, {M} failed. Failed sources: {list with reasons}.</p>
</section>
```

Derive the Sources list from `meta.json.sources` — include only `status: ok` entries. Include the full URL and fetch timestamp. After the list, note any failed sources so the reader knows what's missing.

### 12. Adversarial appendix

Loads the full text of `SESSION_DIR/analysis/devils-advocate.md`, converts markdown to HTML, wraps in `<section class="stockwiz-adversarial">`. Clearly labeled as "Adversarial pass — written to stress-test the thesis, not as a conclusion".

```html
<section class="stockwiz-adversarial">
  <h2>Adversarial Pass</h2>
  <p class="stockwiz-meta">A devil's advocate subagent was run in an isolated context with only the thesis as input. The goal is not balance — the goal is to find the weakest claims and attack them. The orchestrator reconciles this pass against the bull framing; adjustments are listed in the Adjustments After Stress Test section above if any were material.</p>

  <!-- Weakest Claims -->
  <h3>Weakest claims (ranked)</h3>
  <ol>
    <li>
      <p><strong>{claim 1}</strong> — fragility: HIGH</p>
      <p>{why fragile + evidence + when resolved}</p>
    </li>
    <li>...</li>
  </ol>

  <!-- Opposing Narrative -->
  <h3>The opposing narrative</h3>
  <p><strong>{bear headline}</strong></p>
  <p>{supporting claim 1}</p>
  <p>{supporting claim 2}</p>
  <p>{supporting claim 3}</p>
  <p><em>Disconfirmers of this bear case:</em></p>
  <ul><li>{disconfirmer 1}</li><li>{disconfirmer 2}</li></ul>

  <!-- Missing Counter-Evidence -->
  <h3>Missing counter-evidence</h3>
  <ul>{items}</ul>

  <!-- Alternative Interpretations -->
  <h3>Alternative interpretations</h3>
  <ul>{items}</ul>

  <!-- Kill Switch Adequacy -->
  <h3>Kill switch adequacy</h3>
  <table class="stockwiz-table">
    <tr><th>Original</th><th>Verdict</th><th>Why</th><th>Rewrite</th></tr>
    <!-- rows from devils-advocate.md -->
  </table>
</section>
```

**If the devils-advocate pass was skipped** (Phase 1.5 fallback, or `/stockwiz-thesis` reduced mode), render a short placeholder:

```html
<section class="stockwiz-adversarial">
  <h2>Adversarial Pass</h2>
  <p class="stockwiz-meta">Devil's advocate stress test was not run for this session. Run <code>/stockwiz-bear {TICKER}</code> to generate an isolated adversarial pass.</p>
</section>
```

### 13. Disclaimer

Load `${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/disclaimer.html` verbatim. Substitute `{version}`, `{timestamp}`, `{session-id}`. Insert as the final element before `</body>`. Do NOT modify its structure or CSS.

## Phase 2 target report

In Phase 2 the deep-dive report contains:
- Hero with ticker, base-case headline, price, sparkline
- Snapshot strip (4–6 cards)
- Three cases (hash-shuffled order, equal weight)
- Kill switches row
- **Fundamentals**: valuation framing paragraph + multiples table + quality/margin sparklines + growth table + capital structure + ownership (no longer a placeholder)
- **Sentiment**: net sentiment paragraph + insider/short/analyst grid + SWS flagged risks (verbatim in `<q>` tags) + headlines timeline + conflicting signals (no longer a placeholder)
- **Peers**: comp table with direction-of-goodness + caveats paragraph (no longer a placeholder)
- **Risk**: volatility + drawdown + concentration + cap-structure + tail risks (no longer a placeholder)
- **Assumption ledger**: full table from fundamental.md (no longer a placeholder)
- Unknowns (from thesis.md, equal visual weight to bull case)
- Sources (from meta.json.sources)
- **Adversarial appendix**: full devils-advocate.md rendered, clearly labeled as stress test not conclusion
- Disclaimer footer

This yields a report that's typically 60–150KB in Phase 2 depending on how much data came through. The walking skeleton fallback (Phase 1.5 — when analyses and devils-advocate are missing) still works and produces a 25–50KB report with placeholders for the missing sections.

## Graceful degradation rule

If any of the four analysis files or `devils-advocate.md` is missing from the session workspace, render a short placeholder in that section rather than aborting. The report should always be emittable as long as `thesis.md` exists.

The precedence order for each section:
1. **Full section** — if the analysis file exists and has `status: ok` content
2. **Thin section** — if the analysis file is missing but raw data is present, render a minimal version from the raw files directly (e.g. just the snapshot metrics for fundamentals)
3. **Placeholder note** — if neither is available, render a single sentence note explaining what's missing and how to fill it

The goal: a Phase 1.5 run without analysis skills should still produce a valid, compliant, readable report. It will just have fewer panels filled in.
