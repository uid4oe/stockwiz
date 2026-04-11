# Reddit

**Status:** active. Retail sentiment as a **contrarian signal** — not a confirmatory one. Three subreddits covered: r/stocks, r/wallstreetbets, r/investing. Fetched via `.json` API endpoints which are keyless but heavily rate-limited.
**Access method:** **`Bash` + `curl`** with a descriptive User-Agent (Reddit's API TOS requires one). **JSON endpoints**, not HTML scraping.
**Rate policy:** Reddit throttles unauthenticated requests aggressively. 1500ms between fetches + graceful failure on 429.

## Why retail sentiment is in the data set

Retail discussion on Reddit is **high-noise, low-accuracy** on any given post, but a useful **aggregate** signal in two ways:

1. **Contrarian indicator at extremes.** When r/wallstreetbets mentions a ticker 500+ times in a week with uniformly bullish framing, that historically correlates with a near-term top. When retail loses interest entirely, that sometimes correlates with a bottom.
2. **Early narrative detection.** Retail discussion often surfaces material concerns 1–2 weeks before mainstream coverage — short-seller reports, FDA rumors, management tweets, supply-chain disruptions. The signal-to-noise is terrible but the latency is sometimes ahead of Bloomberg.

stockwiz captures Reddit at **weight 0.2** in the sentiment-synthesis rubric. It's never the primary driver of a thesis claim. It's context.

## What we capture

For each of the three subreddits:

- **Top 5-10 posts** from the last 30 days that mention the ticker
- **Per post**: title, upvotes, comment count, post date, permalink
- **NO comment bodies** — comments on Reddit are 99% noise and 1% insight; we intentionally do not capture them . What we don't capture:
- Post body text (frequently walls of memes, emoji, or lorem-ipsum-grade analysis)
- Individual user names (privacy + they don't predict anything)
- Live streaming data (r/wallstreetbets posts come and go in minutes during market hours)

## URL patterns

Reddit exposes a JSON API by appending `.json` to any listing URL:

```
https://www.reddit.com/r/stocks/search.json?q={TICKER}&sort=new&t=month&restrict_sr=1
https://www.reddit.com/r/wallstreetbets/search.json?q={TICKER}&sort=new&t=month&restrict_sr=1
https://www.reddit.com/r/investing/search.json?q={TICKER}&sort=new&t=month&restrict_sr=1
```

Parameters:
- `q={TICKER}` — search query (uppercase ticker)
- `sort=new` — newest first (alternatively `sort=hot` or `sort=top` for different signal shapes)
- `t=month` — time window (last 30 days)
- `restrict_sr=1` — restrict to the current subreddit (otherwise Reddit searches across all subreddits)

For sort, **`new`** is the most informative for signal detection. Use `top` (with `t=month`) as a fallback if `new` returns too much noise (e.g. off-topic posts in r/stocks where the ticker is mentioned incidentally).

## Fetch

```bash
# Reddit's API TOS requires a descriptive User-Agent
REDDIT_UA="stockwiz-research/0.3 by anonymous (contact: noreply@stockwiz.local)"

for sub in stocks wallstreetbets investing; do
  URL="https://www.reddit.com/r/${sub}/search.json?q=${TICKER}&sort=new&t=month&restrict_sr=1&limit=15"
  curl -sS -L \
    -A "$REDDIT_UA" \
    -H "Accept: application/json" \
    --max-time 20 \
    "$URL" \
    > "/tmp/stockwiz-reddit-${sub}-${TICKER}.json"
  sleep 1.5
done
```

Reddit's JSON endpoint usually returns HTTP 200 with a JSON body on success. **Rate limiting is tricky** — Reddit sometimes returns HTTP 429, but sometimes returns HTTP 200 with a JSON body containing `{"message": "Too Many Requests", "error": 429}` instead of the expected listing structure. **You must check both the HTTP status AND the parsed JSON body** before assuming the response is usable:

```bash
# Check HTTP status (curl -w prints it after body)
curl -sS -A "$REDDIT_UA" -H "Accept: application/json" \
  -w "\n__HTTP_STATUS__:%{http_code}\n" \
  --max-time 20 "$URL" > /tmp/stockwiz-reddit-${sub}-${TICKER}.json
```

Then, in your JSON parser:

```python
import json
with open(f'/tmp/stockwiz-reddit-{sub}-{TICKER}.json') as f:
    raw = f.read()
# Strip the appended status line if present
if '__HTTP_STATUS__' in raw:
    raw = raw.split('__HTTP_STATUS__')[0].strip()
d = json.loads(raw)

# Check for body-encoded rate limit (HTTP 200 but error payload)
if isinstance(d, dict) and d.get('error') == 429:
    # Reason: rate-limited (body)
    ...
    continue

# Check for expected listing structure
if 'data' not in d or 'children' not in d.get('data', {}):
    # Reason: unexpected-shape (redirect, error, or format change)
    ...
    continue

# Happy path
posts = d['data']['children']
```

On private/quarantined/deleted subreddits, you get a redirect or 403. On ticker with zero posts, you get an empty `children` array (still HTTP 200, not an error — just mark as `no-posts`).

### Parsing

The JSON structure is:

```json
{
  "kind": "Listing",
  "data": {
    "after": "...",
    "before": null,
    "children": [
      {
        "kind": "t3",
        "data": {
          "title": "NVDA earnings preview: Data Center Revenue Expected to Top $35B",
          "score": 452,
          "num_comments": 187,
          "created_utc": 1712793600,
          "permalink": "/r/stocks/comments/xyz/nvda_earnings_preview/",
          "url": "https://www.reddit.com/r/stocks/comments/xyz/...",
          "author": "[deleted]",
          "subreddit": "stocks",
          "selftext": "...",  // post body — IGNORE
          "flair": null
        }
      },
      ...
    ]
  }
}
```

Use `jq` to extract:

```bash
jq '.data.children[] | .data | {title, score, num_comments, created_utc, permalink}' \
  /tmp/stockwiz-reddit-stocks-${TICKER}.json
```

Or Python if jq unavailable:

```python
import json
with open(f'/tmp/stockwiz-reddit-{sub}-{TICKER}.json') as f:
    d = json.load(f)
for child in d['data']['children'][:10]:
    p = child['data']
    print(p['title'], p['score'], p['num_comments'])
```

## What to extract per subreddit

Top 5-10 posts by score (or by date if score-sort returns stale posts). For each:

- **Title** (verbatim)
- **Score** (upvotes minus downvotes, can be negative)
- **Comment count**
- **Post date** (convert `created_utc` unix timestamp to ISO 8601)
- **Permalink** (record for attribution, not fetched)

Plus aggregate metrics per subreddit:
- **Total post count in last 30 days** matching the ticker query (from `children.length` — note Reddit's limit defaults to 25; we set `limit=15`)
- **Average score** of the captured posts
- **Activity level**: `quiet` (0-3 posts), `normal` (4-10), `elevated` (11-15), `heavy` (15+). Subjective threshold; records the shape of the conversation.

## Writing the raw file

```markdown
---
source: Reddit
url: https://www.reddit.com/r/stocks/search.json?q=NVDA (+ r/wallstreetbets, r/investing)
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
notes: low-weight source (0.2 in sentiment synthesis rubric); contrarian signal, not confirmatory
---

# Reddit — NVDA

## r/stocks (activity: elevated)

- Posts in last 30 days matching "NVDA": 12
- Average score: 84
- Capture: top 10 by date

### Recent posts

1. **[2026-04-10] "NVDA earnings preview: Data Center Revenue Expected to Top $35B"**
   - Score: 452 · Comments: 187
   - Permalink: /r/stocks/comments/xyz/

2. **[2026-04-09] "Is NVDA still a buy at $188?"**
   - Score: 28 · Comments: 94

3. **[2026-04-08] "NVDA vs AMD for long-term hold — 2026 edition"**
   - Score: 171 · Comments: 203

[... up to 10 ...]

## r/wallstreetbets (activity: heavy)

- Posts in last 30 days matching "NVDA": 15+ (at API limit)
- Average score: 312
- Capture: top 10 by date

### Recent posts

1. **[2026-04-10] "🚀🚀🚀 NVDA YOLO $50K calls"**
   - Score: 2,847 · Comments: 412
   - Permalink: ...

2. **[2026-04-10] "NVDA bagholders assemble"**
   - Score: -17 · Comments: 38
   - Permalink: ...

[... up to 10 ...]

## r/investing (activity: normal)

- Posts in last 30 days matching "NVDA": 6
- Average score: 41
- Capture: top 6 by date

### Recent posts

[...]

## Aggregate read

- **Total posts captured:** 31 across three subreddits
- **Overall activity:** elevated-to-heavy (consistent with a mega-cap in an attention cycle)
- **Sentiment shape** (title lexicon only — not computed as a score): predominantly bullish titles in r/wallstreetbets ("YOLO", "🚀", "to the moon" variants), mixed in r/stocks, analytical in r/investing
- **Contrarian signal:** heavy bullish r/wallstreetbets activity historically correlates with near-term consolidation more often than continued uptrends — note this in sentiment synthesis as a contrarian flag, weight 0.2
```

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| HTTP 429 | `rate-limited` | Skip remaining subreddits this run, mark partial |
| HTTP 200 with body `{"error": 429}` or `{"message": "Too Many Requests"}` | `rate-limited` (same reason code as HTTP 429) | Skip remaining subreddits this run, mark partial |
| HTTP 403 on a sub | `sub-private-or-banned` | Skip that sub, continue others |
| Empty `children` array | `no-posts` | Still mark ok but note low activity |
| JSON parse error | `invalid-json` | Mark failed for that sub, continue others |
| Reddit responds with HTML challenge (rare) | `reddit-html-response` | Skip |

Skipping Reddit entirely is never fatal. Skipping 1-2 subreddits within Reddit is acceptable — just note the partial yield.

## Slug

Write to `raw/reddit.md`. The three per-sub JSON responses go to `raw/reddit-stocks.json`, `raw/reddit-wallstreetbets.json`, `raw/reddit-investing.json` as the audit trail.

## Calls

- 3 curl calls (one per subreddit) in the happy path
- With rate-limit 429 on sub 1: stop there, 1 call
- Total: **1-3 curl calls** per deep-dive

## Compliance note

Reddit post titles frequently contain strong language — "to the moon", "YOLO", "bagholder", "buy the dip", "sell your house", emoji, etc. **Capture titles verbatim.** The report-writer wraps them in `<q>` tags when reproduced, which exempts them from the compliance rewrite pass. Do NOT paraphrase or sanitize at the fetch stage — the audit trail requires verbatim capture, even (especially) when the language is colorful.

The only language-rewrite rule that applies to Reddit capture is **never strip or modify the source**. "🚀🚀🚀 NVDA YOLO $50K calls" is legitimate source data for a contrarian sentiment signal; removing the emoji or rewording the title destroys the signal.

## A note on privacy

Reddit's JSON endpoint returns usernames alongside post metadata. **Do NOT capture usernames** in the stockwiz raw file. Individual users don't predict anything (we're aggregating for signal, not doxxing), and including them would be both unnecessary and inappropriate. The parsing pipeline should extract title/score/comments/date/permalink and discard every other field.
