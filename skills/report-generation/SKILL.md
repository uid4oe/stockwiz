---
name: report-generation
description: This skill should be used when producing HTML research artifacts from a stockwiz session workspace. Provides section templates, inline styling rules, citation structure, disclaimer injection, and the mandatory compliance pass. Defers aesthetic direction to the frontend-design skill.
version: 0.5.0
---

# report-generation

The skill that turns a completed stockwiz session workspace into a single self-contained HTML file. Every stockwiz report goes through this skill. The skill provides structure and compliance rules; it defers to `frontend-design` for aesthetic choices (typography, color nuance, hero treatment, distinctive visual language).

## When to use

You are called by the `report-writer` agent after a session has written `thesis.md`, `analysis/*.md`, and `meta.json`. Your output is a single HTML file written to `SESSION_DIR/report.html`.

## The three templates

- **`deep-dive`** — the main research artifact, used by `/stockwiz`. See [`references/deep-dive-template.md`](references/deep-dive-template.md).
- **`compare`** — multi-ticker comparison (roadmap). (`references/compare-template.md`)
- **`pivot`** — thesis pivot menu (roadmap). (`references/pivot-template.md`)

only `deep-dive` is implemented. Other templates fall back to an error message if invoked before (roadmap).

## Composition rules

### Self-contained above all else

The HTML file must open in any browser with no network access and render correctly. This means:

- **Inline `<style>`** — no `<link rel="stylesheet">` to local or remote CSS
- **Inline `<script>`** — preferably none at all; if needed, inline only, never `src=`
- **No `<img src="https://..."/>`** — use inline SVG or `data:` URLs for any visual
- **Fonts** — at most one Google Fonts `<link>` for a display face, with `font-display:swap` so the document renders immediately even if the font fails to load. System stack + Google Font is the only external dependency allowed.
- **Base CSS** — spliced from `${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/base-styles.css` into the `/*@@BASE_STYLES@@*/` marker inside the `<style>` block by the report-writer's post-composition Bash splice. The CSS file is never loaded into model context — the splice injects it byte-for-byte.

### Citation structure

Every numerical claim, every factual statement about the company, every quote from a source gets a citation. Citations use `<sup>` footnote numbers that link to a numbered Sources section at the bottom of the document.

```html
<p>Revenue grew 22% YoY in FY2025<sup><a href="#src-3">3</a></sup>, driven by
data center demand.</p>
...
<section class="stockwiz-sources">
  <h2>Sources</h2>
  <ol>
    <li id="src-1">SEC EDGAR 10-K, filed 2025-02-20. Retrieved 2026-04-11.</li>
    <li id="src-2">Yahoo Finance key statistics. Retrieved 2026-04-11.</li>
    <li id="src-3">SEC EDGAR 10-K, Item 7 MD&amp;A. Retrieved 2026-04-11.</li>
  </ol>
</section>
```

Rule: **if you cannot find a source for a number, do not include the number.** Write "undisclosed" or "not available in fetched sources" instead. Never synthesize.

### Quoted source text

If a source's own text must be reproduced (e.g. a rating label, a management quote from a 10-K, a snippet from a news article), wrap it in `<q>...</q>` tags. The compliance pass exempts `<q>` content from rewriting. Example:

```html
<p>Management describes the supply chain situation as
<q>tight but improving</q><sup><a href="#src-1">1</a></sup>.</p>
```

### No external JS, no interactivity (beyond CSS)

The HTML is a document, not an app. Tabs, collapsibles, charts — all of these come from CSS `<details>` / `:target` / grid tricks, not JavaScript. Keep the bundle small (target <300KB), the dependencies zero, and the behavior predictable.

## Compliance pass — mandatory

**Before** promoting the draft to `report.html`, run the grep-first compliance pass as specified in [`references/compliance-rules.md`](references/compliance-rules.md). In summary:

1. Compose to `SESSION_DIR/report-draft.html`, then run the canonical candidate grep from the compliance rules via Bash — never re-scan the full 60–150KB HTML in model context.
2. Judge each matching line against the exemptions (`<q>...</q>`, HTML attributes, URLs, comments, `<code>`/`<pre>`, `-side` compounds, disclaimer block).
3. Apply targeted rewrites. For "strip sentence" rules, remove the enclosing sentence and keep a local log for the return summary.
4. Re-grep; loop up to 3 times.
5. If banned phrases remain outside the exemptions after 3 iterations, **abort**: delete the draft and return an error to the calling command. Only a clean draft is promoted to `report.html`.

Report every rewrite and every stripped sentence in the return summary — the orchestrator records them as `adjustments: [...]` / `strippedSentences: [...]` on the report-writer stage entry in meta.json (subagents never write meta.json).

## Disclaimer injection

The report-writer emits a `<!--@@DISCLAIMER@@-->` marker as the final element before `</body>`; its post-composition Bash splice replaces the marker with `${CLAUDE_PLUGIN_ROOT}/skills/report-generation/assets/disclaimer.html` verbatim and substitutes the placeholders:

- `{version}` → plugin version from `.claude-plugin/plugin.json`
- `{timestamp}` → ISO timestamp of report generation
- `{session-id}` → basename of the session directory (e.g. `NVDA-20260411T143200`)

The disclaimer CSS is in `base-styles.css` under `.stockwiz-disclaimer`. Your `<style>` block will already include it.

## Aesthetic direction — defer to frontend-design

For typography, color, hero treatment, and anything visual beyond the grid/layout primitives in `base-styles.css`, invoke the `frontend-design` skill. Key principles to carry over:

- **Distinctive, not generic.** Avoid the "Tailwind starter aesthetic" — no purple gradients, no pastel soft shadows, no generic Inter, no glassmorphism.
- **Considered typography.** Pick one display face (via Google Fonts link) and rely on the serif body stack in `base-styles.css`. Two fonts max.
- **Cohesive color.** The base styles use a warm neutral palette (#fbfaf7 background, #1a1a1a ink, #2c5f7e accent). You can override the accent to suit the ticker's sector, but keep the palette single-scheme (not light+dark toggle).
- **Restrained motion.** No scroll-triggered animations, no parallax. This is a research document, not a portfolio piece.
- **Equal visual weight for bull/base/bear.** This is non-negotiable. Same column width, same font, same background, same color intensity. No arrows. No emoji. No traffic-light coloring. The sparkline and accent treatments for the three cases must be identical.

## Output convention

Compose to `SESSION_DIR/report-draft.html`, splice assets, run the compliance pass, then promote the clean draft to `SESSION_DIR/report.html`. After promotion, verify:

- File size is between **10KB and 500KB** (via Bash `wc -c`). Under 10KB means something's missing; over 500KB means binary bloat leaked in.
- The file starts with `<!DOCTYPE html>` and ends with `</html>`.
- The file contains `stockwiz-disclaimer` (class on the disclaimer footer).
- The file contains the string `## Bull Case` was not accidentally emitted (a sign that markdown wasn't converted to HTML).

Return to the caller a summary:
- File path
- Byte count
- Sections present (from a fixed list)
- Compliance rewrites applied (count)
- Sentences stripped (count)
- Any sections skipped due to missing data

## Scope

Only the deep-dive template is wired, with the full insights-first section set described in `references/deep-dive-template.md`: hero → key metrics strip → TL;DR panel above the fold, then collapsible Three Cases / Kill Switches / Fundamentals / Sentiment / Peers / Risk / Assumption Ledger / Unknowns / Sources / Adversarial Pass, and the always-visible disclaimer footer.

Sections degrade gracefully: if an analysis file is missing or is a failure stub, the report-writer renders a thin fallback from `raw/` files (or a placeholder note) and records per-section fidelity in its return summary. A report built from few sources looks thin — that's acceptable and honest; a padded-out report is not.
