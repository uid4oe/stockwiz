# Google Finance + Google News RSS

**Status:** active. Two-part source — Google Finance for quote data (limited SSR value) and **Google News RSS** for high-quality news aggregation. The RSS feed is the primary value here; the Finance page is secondary and often returns an empty JS shell.
**Access method:** **`Bash` + `curl`** with browser User-Agent (Finance) and plain `curl` (News RSS — no UA gating).
**Rate policy:** 1500ms delay between fetches; one retry on transient failure.

## Why this source is in the set

stockwiz previously had no dedicated news source — Simply Wall St and Seeking Alpha gave us some narrative, Reddit gave us retail chatter, but nothing was pulling reported, editorially-filtered news. Google News RSS fills that gap:

- **Keyless.** No API signup, no rate limit challenges, no Cloudflare
- **Well-formed XML.** RSS 2.0 with standard fields — title, link, pubDate, description, source
- **Broad coverage.** Aggregates Bloomberg, Reuters, WSJ, CNBC, Yahoo Finance, Benzinga, Seeking Alpha public articles, and hundreds of other financial publishers
- **Timestamped.** Every item has a pubDate — perfect for recency-weighted sentiment synthesis
- **Titles + snippets are free.** You don't get article bodies (those are behind individual publishers' paywalls) but headlines plus short descriptions are a large fraction of the signal

Google Finance's quote page adds: current price / day range / 52w range / market cap / basic multiples (where visible in SSR). This is redundant with Finviz and Stockanalysis, so it's a cross-check, not a primary. Skip if Google Finance SSR is empty.

## URL patterns

### Google News RSS (primary — always fetch first)

```
https://news.google.com/rss/search?q=%22{TICKER}%22+stock&hl=en-US&gl=US&ceid=US:en
```

- `q=%22{TICKER}%22+stock` — the ticker in quotes + "stock" as a disambiguator (prevents ticker-symbol confusion with unrelated words — "F" alone would return Ford + the letter F; `"F" stock` narrows it)
- `hl=en-US` — language
- `gl=US` — geography
- `ceid=US:en` — country:language pair (Google's convention)

**Special-character tickers (BRK.B, BF.B, JW.A, etc).** Tickers containing `.` or `-` don't play well with Google News's tokenizer inside quoted search. Two strategies:

1. **Preferred**: use the company name instead of the ticker. For Berkshire Hathaway Class B, query `"Berkshire Hathaway"` instead of `"BRK.B"`. Get the company name from `raw/sec-edgar-10k.md` after SEC succeeds — SEC always has the legal entity name.
2. **Fallback**: strip the punctuation and quote the base ticker. For `BRK.B` query `"BRK" stock`, understanding that you'll get Class A and Class B news undifferentiated.

Document which strategy you used in the raw file's frontmatter notes so downstream knows the news may be class-undifferentiated.

Alternative search: if you have the company name, you can use the name instead of ticker quote — sometimes gives better results for small-caps where the ticker is ambiguous:

```
https://news.google.com/rss/search?q=%22{Company+Name}%22&hl=en-US&gl=US&ceid=US:en
```

For NVDA, try the ticker-quoted version first; if fewer than 5 items come back, retry with the company name from SEC EDGAR output.

### Google Finance quote page (secondary — optional)

```
https://www.google.com/finance/quote/{TICKER}:NASDAQ
```

Fall back to `:NYSE` on 404. Google Finance is very JS-heavy — most of the real-time data is loaded client-side. What you might get in SSR:
- Company name, logo URL (useless for us), basic description
- Current price (sometimes)
- Day range, 52w range (sometimes)
- Market cap (sometimes)
- P/E (sometimes)
- Primary exchange

Don't expect much; this source is redundant for numerical data. Its value is as a cross-check on the ticker's existence and exchange listing, not as a primary data feed.

## Fetch

### Google News RSS

```bash
TICKER_ENC=$(printf '%s' "$TICKER" | python3 -c "import sys, urllib.parse; print(urllib.parse.quote('\"' + sys.stdin.read().strip() + '\"'))")
RSS_URL="https://news.google.com/rss/search?q=${TICKER_ENC}+stock&hl=en-US&gl=US&ceid=US:en"

curl -sS -L \
  -A "stockwiz research plugin noreply@stockwiz.local" \
  -H "Accept: application/rss+xml, application/xml, text/xml" \
  --max-time 30 \
  "$RSS_URL" \
  > /tmp/stockwiz-gnews-${TICKER}.xml
```

Google News RSS does not gate on User-Agent — a descriptive identifier is polite. It does rate-limit heavily-parallel requests; with 1500ms between fetches you're fine.

### Google Finance page (optional secondary)

```bash
BROWSER_UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'

curl -sS -L \
  -A "$BROWSER_UA" \
  --max-time 20 \
  "https://www.google.com/finance/quote/${TICKER}:NASDAQ" \
  | lynx -stdin -dump -width=120 -nolist \
  > /tmp/stockwiz-gfinance-${TICKER}.txt
```

If lynx is missing, save raw HTML and Read it with an offset/limit. Much of the page will be navigation chrome; only the first ~2000 chars of content are useful.

## What to extract

### From Google News RSS (primary)

Parse the XML with `jq`-for-XML (not available) or Python's `xml.etree.ElementTree`, or just use the LLM to read the file directly. The relevant structure is:

```xml
<rss version="2.0">
  <channel>
    <title>...</title>
    <link>...</link>
    <item>
      <title>Headline text — Publisher</title>
      <link>https://news.google.com/rss/articles/...</link>
      <guid>...</guid>
      <pubDate>Fri, 11 Apr 2026 14:32:00 GMT</pubDate>
      <description>&lt;p&gt;Snippet or headline text&lt;/p&gt;</description>
      <source url="https://www.publisher.com">Publisher</source>
    </item>
    <!-- more items -->
  </channel>
</rss>
```

Extract up to the **20 most recent items**. For each:

- **Title**: strip the " — Publisher" suffix that Google appends
- **Publisher**: from `<source>` element
- **URL**: from `<link>` element (note: these are Google News redirect URLs, not direct publisher URLs; don't fetch them)
- **Published date**: parse pubDate, convert to ISO 8601
- **Description**: strip HTML entities and tags from the description, keep plain text

Sort by published date, most recent first.

### From Google Finance page (secondary)

If the lynx dump contains meaningful content (more than ~500 chars of non-nav text), extract:
- Company name (cross-check against SEC)
- Exchange classification (NASDAQ vs NYSE)
- Current price (cross-check against Finviz / Yahoo)
- Any summary metrics visible (market cap, day range, 52w range, P/E)

If the dump is mostly navigation and error text, mark the Google Finance sub-source as having minimal yield but don't mark the whole source failed — the RSS is the primary.

## Writing the raw file

Write to `$SESSION_DIR/raw/google-finance.md`:

```markdown
---
source: Google Finance + Google News RSS
url: https://news.google.com/rss/search?q=%22NVDA%22+stock (+ Google Finance page)
fetched_at: 2026-04-11T14:32:00+02:00
status: ok
notes: Google News RSS is primary; Google Finance quote page is secondary cross-check only
---

# Google Finance + Google News — NVDA

## Quote cross-check (Google Finance SSR)

- Exchange: NASDAQ (confirmed)
- Current price: $188.63 (cross-checks with Finviz/Stockanalysis)
- Day range: $185.20 – $189.44
- Market cap: $4.58T
- P/E (trailing): 38.48
- Source note: Google Finance SSR yielded limited data; primary multiples should be read from Finviz or Stockanalysis

## Recent news (Google News RSS, last 30 days, most recent first)

_20 items captured. Titles and publishers verbatim; snippets are short editorial summaries, not article bodies (paywalled)._

### 2026-04-10

1. **Nvidia stock hits fresh high on AI chip demand** — *Reuters*
   - Snippet: Shares of Nvidia rose 2.6% in Thursday trading as analysts raised price targets following hyperscaler capex guidance upgrades...
   - Link (Google News): {URL}

2. **NVIDIA: The Rerating Is Over, The Growth Story Isn't** — *Seeking Alpha*
   - Snippet: ...
   - Link: {URL}

### 2026-04-09

3. **Nvidia Q1 Preview: Data Center Revenue Expected to Top $35B** — *Bloomberg*
   - ...

[... up to 20 items total ...]

## Publisher distribution

Of the 20 items captured:
- Reuters: 3
- Bloomberg: 4
- Seeking Alpha: 5
- CNBC: 2
- WSJ: 1
- Other: 5

This distribution is itself useful context — a concentration in opinion publishers (Seeking Alpha) vs wire services (Reuters, Bloomberg) indicates the shape of the conversation.
```

## Failure modes

| Condition | Reason | Action |
|---|---|---|
| Google News RSS returns empty `<channel>` | `no-news-items` | Try fallback query with company name; if still empty, mark ok but note low yield |
| Google News RSS returns non-XML content | `rss-parse-failed` | Save the raw response, mark failed |
| HTTP 429 on RSS endpoint | `rate-limited` | 10s backoff, one retry, then mark failed |
| Google Finance page returns empty lynx dump | `gfinance-ssr-empty` | Note in raw file, mark source partial (not failed — RSS is primary) |
| Google Finance 404 on both NASDAQ and NYSE | `gfinance-not-found` | Note but don't fail — some tickers legitimately aren't on Google Finance (foreign listings, very small caps) |

Skipping Google Finance is never fatal. Losing the RSS headlines hurts sentiment quality but doesn't break the deep-dive.

## Slug

Write to `raw/google-finance.md`. Also save the raw XML to `raw/google-news-feed.xml` as the audit trail.

## Calls

- 1 curl call for Google News RSS
- 1-2 curl calls for Google Finance (NASDAQ try, NYSE fallback on 404)
- Total: **2-3 fetches** per deep-dive

## A note on the publisher distribution metric

The publisher distribution (Reuters X, Bloomberg Y, Seeking Alpha Z...) is a unique signal. A stock where the recent news mix is dominated by wire services (Reuters, Bloomberg, AP) is typically in the "something is happening" state — earnings, M&A, regulatory, macro. A stock where the mix is dominated by opinion publishers (Seeking Alpha, Benzinga, Motley Fool) is typically in the "retail narrative" state — momentum, meme, speculation. The two states are both useful to know about and neither is "better" — but they're different signals. The sentiment-synthesis skill should factor this into its recency-weighted news section.

## Compliance note

News headlines frequently use advisory language ("Buy Now", "Avoid This Stock", "Strong Sell Signal"). **Capture them verbatim** — the report-writer wraps them in `<q>` tags at render time, which exempts them from the compliance rewrite pass. Do NOT paraphrase headlines at the fetch stage; the audit trail requires verbatim capture.
