# X.com (formerly Twitter) — Institutional Commentary

**Status:** active. Best-effort source — surfaces near-real-time commentary from *verified institutional handles only*. A whitelist gate enforces this; non-whitelisted handles are discarded without reading content.
**Keyless.** Accessed via Google `WebSearch` with `site:x.com`, not via the X.com API (which requires paid auth).
**Rate policy:** 1 WebSearch call per ticker. Counts against the global ≤3 WebSearch budget — see `source-extraction/SKILL.md`.

<!-- review-by 2026-11-01 — handles deprecate, accounts go private, journalists change jobs; review the whitelist every 6 months. -->

## Why X.com is in the data set

For liquid US equities, several signals appear on X.com hours-to-days before they make it into Reuters/Bloomberg headlines or Seeking Alpha articles:

- Sell-side analysts post target changes immediately after their report drops
- Wire-service reporters tease scoops before the formal article publishes
- Issuer-official accounts post guidance or material events (often mirroring a same-time 8-K)
- Aggregators (`@unusual_whales`, `@firstsquawkcom`) repackage same-day institutional flow data

The signal is high *only* when it comes from a known, accountable source. Anonymous "fintwit" accounts, cashtag-spam handles, WSB-orbit accounts, and "guru" personalities have no edge on average and large negative edge during euphoria phases. The whitelist below is the entire point of this source — without it, X.com is noise indistinguishable from Reddit.

## Method

Single `WebSearch` per ticker:

```
WebSearch: site:x.com "{TICKER}" {company_short_name} {current_year}
```

Where `{company_short_name}` is the most-natural informal name for the company (`Boston Scientific` → `Boston Scientific`; `Alphabet Inc Class A` → `Alphabet` or `Google`).

The search returns a list of result snippets with the format:

```
{handle} on X: "{snippet}" — x.com/{handle}/status/{id}
```

## Whitelist gate (mandatory)

A result may be quoted in the raw file **only** if its handle (the `@username` part) appears in one of these tiers. Discard unrecognized handles silently.

**Tier 1 — financial wires / news desks:**
`@Reuters`, `@WSJ`, `@BloombergNews`, `@business`, `@ft`, `@CNBC`, `@SquawkCNBC`,
`@YahooFinance`, `@MarketWatch`, `@IBDinvestors`, `@SeekingAlpha`, `@FactSet`

**Tier 2 — recognized journalists (verified profiles only):**
`@michaelsantoli`, `@CarlQuintanilla`, `@hbsbull`, `@JoeSquawk`, `@SaraEisen`,
`@leslie_picker`, `@deenyfletcher`, `@DeItaone`, `@LizClaman`

**Tier 3 — sell-side firm verified accounts:**
`@MorganStanley`, `@GoldmanSachs`, `@JPMorgan`, `@BofASecurities`, `@Citi`,
`@WellsFargo`, `@DeutscheBank`, `@UBS`, `@BarclaysIB`

**Tier 4 — analyst-note aggregators (use carefully):**
`@wallstengine`, `@firstsquawkcom`, `@unusual_whales` (institutional flow only —
ignore retail-opinion posts from this account)

**Tier 5 — issuer official accounts** (for guidance / announcements only, never sentiment):
the verified `@<companyname>` handle (e.g. `@bostonsci`, `@Apple`, `@PayPalNews`).

### Excluded — drop without reading

- Anonymous / pseudonymous handles (no real name, no institutional affiliation)
- WSB-orbit accounts (`@wallstreetbets`, anything visibly in that orbit)
- Cashtag-spam accounts (handles that post hundreds of `$TICKER` mentions/day)
- "Guru" accounts, even at high follower counts
- Anything with 🚀 / 💎 emoji frequency > 1 per post
- Any account posting "buy NOW" / "moon" / price-target-without-source content

## What to extract

Up to **5 quotes per ticker**. For each surviving (whitelisted) result, capture:

```json
{
  "handle": "@SeekingAlpha",
  "tier": 1,
  "snippet": "...up to 240 characters of the post body, verbatim...",
  "url": "https://x.com/SeekingAlpha/status/...",
  "fetched_at": "2026-05-01T12:00:00Z"
}
```

`snippet` should be the post text as returned in the search-result snippet — do not paraphrase, do not interpret. If the snippet is truncated by Google, mark it with `…` at the truncation point. Do not follow the URL to retrieve the full tweet (the actual `x.com` URLs are gated behind login walls and will not return useful content via WebFetch).

## Extraction prompt template

> These are Google search results scoped to `site:x.com` for a US equity ticker. For each result, identify the X.com handle (the `@username`). **Discard any result whose handle is not in the whitelist provided in `references/x-com.md`.** For surviving results, return up to 5 entries as a JSON array. Each entry: `{ handle, tier, snippet, url, fetched_at }`. Tier is the integer 1–5 corresponding to the whitelist section. Snippet ≤ 240 chars, verbatim from the search result. Do not editorialize. JSON only, no commentary. If zero whitelisted results, return `[]`.

## Failure signatures

WebSearch is generally reliable. Modes that count as failure:

- WebSearch returns zero results for the query → reason `no-results`. **Not failure** — this is a normal outcome for low-attention tickers. Write the raw file with `status: ok` and an empty `quotes: []` array, not `status: failed`.
- WebSearch rate-limited (429-equivalent in the WebSearch tool wrapper) → reason `429`. One retry after 10s, then mark failed.
- Tool error (network, invalid query) → reason `tool-error`. Mark failed; do not retry.

A `quotes: []` outcome is **expected** for ~30–50% of tickers and should not propagate as a quality flag.

## Fallback

None. If X.com fails, the institutional commentary lens is simply absent. Sentiment-synthesis will note the gap in Unknowns and continue without re-weighting.

## Slug

Write to `raw/x-com-commentary.md`.

## Calls

1 WebSearch call (counted against the global ≤3 WebSearch budget that source-extraction shares with Macrotrends and SWS slug resolution). Zero WebFetch calls.

## A note on compliance

X.com posts often contain advisory language ("BUY this", "sell rating cut to $X", etc.). The raw file captures these inside `<q>`-style quoting via the `snippet` field — at the report-writer stage, snippets are wrapped in `<q>` tags so the imperative language is unambiguously attributed to its source. **Never paraphrase a tweet without the `<q>` wrap.** This is a compliance hazard point: tweets are tempting to summarize with imperative verbs and that breaks the no-recommendations rule. Quote, attribute, do not synthesize.
