# Search Console — reading index status without guessing

*Loaded by `web-launcher` SKILL whenever a live site's indexing is in question, or the user
mentions Search Console, "not indexed", "not appearing in Google", or a coverage reason.*

> **Verified 2026-08-14 · review by 2026-10-13.** Endpoint surface, field names, enum values and
> quotas were read from Google's API reference. Claims marked ⚠️ could not be confirmed against a
> primary source and say why.

## The four walls — read these before promising anything

**1. The "why pages aren't indexed" report cannot be downloaded.** The whole API is four resources:
`sites`, `sitemaps`, `searchanalytics`, `urlInspection`. There is **no coverage endpoint**, and the
BigQuery bulk export contains only performance tables — no indexing-status table. The URL list
behind "Page with redirect ×37" is not obtainable programmatically.

→ So the report is **reconstructed, not fetched**: build a candidate URL set yourself, then inspect
each URL. Your numbers will not match the interface exactly, and you must say so — Google knows
URLs your sitemap and crawl do not contain.

**2. Nothing can be requested, validated, or resubmitted.** "Request indexing" and "Validate fix"
exist only in the interface. The Indexing API covers only `JobPosting` and `BroadcastEvent`, not
general web pages. The single write available is `sitemaps.submit`.

→ Never say "I've told Google to recrawl". Say what you changed, submit the sitemap, and hand the
user the interface link for the rest.

**3. URL Inspection returns the *indexed* version, not the live one.** There is no live-test method
in the API. Right after a fix, it still reports the old state.

→ The honest sentence is: *"live HTTP is now correct; Google has not recrawled yet."* Prove the
first half with `14-diagnostic-checks.md`, and let the second half take its time.

**4. "Crawled – currently not indexed" is not a technical defect.** Google's definition gives no
reason, and their guidance is repeatedly that it is a quality/redundancy judgement rather than a
bug. ⚠️ (guidance is from talks and office hours; not in written documentation)

→ Do not claim you can close it. Name it, say it is a content/authority question, move on.

## Auth — OAuth 2.0, and nothing else

*"Your application must use OAuth 2.0 to authorize requests. No other authorization protocols are
supported."* Scopes: `.../auth/webmasters` (read-write, needed for `sitemaps.submit`) and
`.../auth/webmasters.readonly`.

**A domain property is named `sc-domain:example.com`** in the API, not as a URL. URL-prefix
properties are named by their URL with the trailing slash. `sites.list` returns the exact strings —
read them from there rather than constructing them, and pass one as `siteUrl` on every inspect call.

**Use a desktop OAuth client.** One-time setup, then a refresh token stored locally:

1. Google Cloud console → new project (or reuse one).
2. Enable **Google Search Console API**.
3. OAuth consent screen → External → **Publish it.** Left in "Testing", refresh tokens expire after
   seven days and the integration silently dies a week later. ⚠️ (documented in GCP under consent
   screen, not in the Search Console docs)
4. Credentials → Create OAuth client ID → **Desktop app** → download the JSON.
5. Run the authorization flow once, store the refresh token outside the repository.

Two alternatives, both rejected as the default: a **service account** works — it must be added as a
user on *each* property, and forgetting one returns 403 — but it puts a key file on disk and Google
does not document it as the primary path. **ADC** (`gcloud auth application-default login`) is
undocumented for this API and requires the gcloud SDK, which is a heavier install than one browser
round.

**Dependencies:** the API surface is tiny and the hard part is token refresh, not fetching. An auth
library plus plain `fetch` gives more control than a full client library for far less weight.

## Reconstructing the candidate set

Since the URL list cannot be fetched, build it — and be explicit that each source has a blind spot:

| Source | Catches | Misses |
|---|---|---|
| `sitemap.xml` (follow the index) | Everything the site declares canonical | Every variant Google discovered elsewhere |
| Internal-link crawl | Pages reachable from the site | Orphans and historical URLs |
| **Generated variants** of each canonical URL: trailing-slash flipped, `www`/apex swapped, `http`, `index.html` appended, uppercase | The redirecting twins that dominate "Page with redirect" | Variants you did not think to generate |
| Framework redirect map and `_redirects` | Legacy aliases | — |
| Old sitemaps and previous deploy artefacts | Historical URLs | — |

The variants row is the one that matters. In a healthy site, canonical URLs plus their generated
twins account for nearly all of what Google knows — and that arithmetic is what lets you say a
number is *expected* rather than *broken*.

## Branching — on enums, never on prose

`urlInspection.index.inspect` returns `indexStatusResult`. Use these:

```
verdict          PASS | PARTIAL | FAIL | NEUTRAL | VERDICT_UNSPECIFIED
robotsTxtState   ALLOWED | DISALLOWED
indexingState    INDEXING_ALLOWED | BLOCKED_BY_META_TAG | BLOCKED_BY_HTTP_HEADER
                 | BLOCKED_BY_ROBOTS_TXT
pageFetchState   SUCCESSFUL | SOFT_404 | BLOCKED_ROBOTS_TXT | NOT_FOUND | ACCESS_DENIED
                 | SERVER_ERROR | REDIRECT_ERROR | ACCESS_FORBIDDEN | BLOCKED_4XX
                 | INTERNAL_CRAWL_ERROR | INVALID_URL
googleCanonical  string — the canonical Google picked
userCanonical    string — the canonical you declared
lastCrawlTime, crawledAs, sitemap[], referringUrls[], inspectionResultLink
```

**`coverageState` is a free-text string, not an enum.** Google types it as `string` and publishes no
list of values — and **it is translated according to `languageCode`**. Branch on it and the logic
breaks silently on a non-English account, which is the worst kind of failure: it does not error, it
just decides wrongly.

- **Pin `languageCode: "en-US"` on every request.**
- Branch on `pageFetchState` + `indexingState` + `googleCanonical != inspectionUrl`. Those are real
  enums and language-independent.
- Quote `coverageState` in reports for the human. Never test against it.

`sitemap[]` and `referringUrls[]` may come back empty, and empty means *not available*, not *none*.
Do not conclude "no page links here" from an empty `referringUrls` — crawl for that instead.

## From reason to cause

| What Google reports | Read it as | Where the cause lives |
|---|---|---|
| Page with redirect | `googleCanonical != inspectionUrl`, fetch redirects | Usually **expected** — the redirecting twin of a canonical URL. A defect only when the target is not 200 (`14-diagnostic-checks.md` C1), the redirect is temporary (C3), or it takes more than one hop (C4) |
| Not found (404) | `pageFetchState: NOT_FOUND` | A redirect issued before the target was resolved (C1), a sitemap listing dead URLs (C5), or an old external link. Trace it; do not delete the URL and call it fixed |
| Alternate page with proper canonical tag | `googleCanonical` points elsewhere and you agree | Normal. Only a problem if you did *not* intend that canonical (C6) |
| Google chose different canonical | `googleCanonical != userCanonical` | Real conflict — your declared canonical is being overruled. Check for duplicate content and mismatched signals (C6, C7) |
| Blocked by robots.txt | `indexingState: BLOCKED_BY_ROBOTS_TXT` | Check the **live** file, not the repository — a CDN may be injecting rules (C2) |
| Excluded by noindex tag | `BLOCKED_BY_META_TAG` / `BLOCKED_BY_HTTP_HEADER` | Staging leftovers, or a shell served without the render path (C7) |
| Soft 404 | `pageFetchState: SOFT_404` | A thin or empty page returning 200 (C7) |
| Crawled – currently not indexed | — | Not technical. Say so |

## Deciding "expected" from enums — measured, not inferred

Live `urlInspection` responses from a healthy domain property, 2026-08-14. This is the shape to
branch on:

| URL | `verdict` | `coverageState` (display only) | `googleCanonical` |
|---|---|---|---|
| `https://example.com/docs/` — canonical | **PASS** | "Submitted and indexed" | itself |
| `https://example.com/docs` — slashless twin | **NEUTRAL** | "Page with redirect" | `…/docs/` |
| `https://example.com/old-alias` — never discovered | **NEUTRAL** | "URL is unknown to Google" | absent |

**`verdict` is the signal, and it is an enum.** Google itself does not call the redirecting twin a
failure — it returns `NEUTRAL`, not `FAIL`. So the expected-variant case is decidable without ever
reading translated prose:

```
verdict == NEUTRAL
  and googleCanonical is set
  and googleCanonical != inspectionUrl
  and inspect(googleCanonical).verdict == PASS
      → expected. The variant redirects to a canonical that is indexed. Not a defect.
```

Two more shapes worth naming, both also enum-decidable:

- `verdict == NEUTRAL` with `googleCanonical` **absent** and every state `*_UNSPECIFIED` — Google
  has never discovered the URL. Nothing is wrong with the page; it simply is not in the index and
  is not in any "not indexed" count either. A legacy alias nobody links to lands here.
- `verdict == FAIL` — this is where `pageFetchState` and `indexingState` earn their place:
  `NOT_FOUND`, `SOFT_404`, `BLOCKED_ROBOTS_TXT`, `BLOCKED_BY_META_TAG` each point at a different
  file.

Quote `coverageState` to the human — it is the string they see in the interface — and never test
against it.

## The verdict that is usually right

**Most "not indexed" counts on a healthy site are not defects.** A site with 40 canonical URLs whose
slashless twins all return a permanent redirect will show roughly 39 "Page with redirect" entries —
and every one of them is Google correctly consolidating on the canonical.

So the first output of any Search Console diagnosis is one of two sentences:

- *"This is expected. Here is the arithmetic: N canonical URLs, M generated variants, and the
  variants account for the excluded count."*
- *"This is a defect. Here is the check that failed, the URL, and the file or dashboard setting
  that causes it."*

Say the first one when it is true. Chasing a false positive costs more than not looking, because it
ends in changes to a site that was working.

## Quotas

Per site: URL Inspection **600 QPM / 2,000 QPD**; Search Analytics 1,200 QPM; other resources
20 QPS / 200 QPM. Per project the ceilings are far higher.

The per-site daily cap is the one that binds: a site over ~2,000 URLs cannot be fully inspected in a
day, and several properties in one day accumulate against separate site caps but the same project
cap. Inspect the candidate set in priority order — sitemap URLs first, generated variants second —
and say what was not inspected rather than silently truncating.

## What to hand the user

The API cannot act, so end a diagnosis with the two or three things only they can do, each with the
deep link `inspectionResultLink` gives you:

1. Open the affected report in Search Console and press **Validate Fix** after the change ships.
2. **Request indexing** on the handful of URLs that matter most.
3. Any dashboard-level change — CDN redirect rules, managed robots.txt — that no code change can
   reach.
