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

### 5. Fundamentals (Phase 2)

In Phase 1, render a placeholder:

```html
<section>
  <h2>Fundamentals</h2>
  <p class="footnote">Detailed fundamental analysis will be available in a future stockwiz version. See <code>analysis/fundamental.md</code> for the current session's fundamental notes if present.</p>
</section>
```

### 6. Sentiment (Phase 2)

Phase 1 placeholder:

```html
<section>
  <h2>Sentiment</h2>
  <p class="footnote">Detailed sentiment synthesis will be available in a future stockwiz version.</p>
</section>
```

### 7. Peers (Phase 2)

Phase 1 placeholder.

### 8. Risk (Phase 2)

Phase 1 placeholder.

### 9. Assumption ledger (Phase 2)

Phase 1 placeholder.

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

### 12. Adversarial appendix (Phase 2)

Loads the full text of `analysis/devils-advocate.md`, converts markdown to HTML, wraps in `<section class="stockwiz-adversarial">`. Clearly labeled as "Adversarial pass — written to stress-test the thesis". Phase 1 omits this (no devils-advocate yet).

### 13. Disclaimer

Load `${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/disclaimer.html` verbatim. Substitute `{version}`, `{timestamp}`, `{session-id}`. Insert as the final element before `</body>`. Do NOT modify its structure or CSS.

## Phase 1 minimum viable report

In Phase 1 the deep-dive report contains:
- Hero
- Snapshot strip (4–6 cards from Yahoo + Finviz)
- Three cases (content from thesis.md; placeholder where analyses are thin)
- Kill switches (from thesis.md)
- Fundamentals placeholder
- Sentiment placeholder
- Peers placeholder
- Risk placeholder
- Assumption ledger placeholder
- Unknowns (from thesis.md)
- Sources (from meta.json.sources)
- Disclaimer

This yields a report that's typically 20–50KB in Phase 1 and passes the compliance spot-check. It's thin, but it's a valid walking skeleton.
