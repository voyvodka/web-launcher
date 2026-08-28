# Indexing acceleration

*Loaded by `web-launcher` SKILL after initial deploy to accelerate discovery.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

**Scope split — read this before anything else.** This file is the *interface* playbook: what a
human clicks in Search Console, Bing Webmaster Tools and IndexNow after a deploy. Everything about
the Search Console **API** — what it can read, what it cannot write, and how to turn a reported
reason into a cause — belongs to `16-search-console.md`, which is the single owner of that
material. Where this file and 16 could be read as disagreeing, **16 is authoritative** and this
file points at it rather than restating it.

## Google Search Console setup

1. https://search.google.com/search-console → **Add property** → **Domain** type (covers all subdomains + protocols)
2. Verify via TXT record:
   - CF Dashboard → DNS → Add TXT
   - Name: `@`
   - Content: paste `google-site-verification=...` string from GSC
   - Save → back to GSC → **Verify**
3. Post-verification:
   - **Sitemaps** → submit `https://DOMAIN/sitemap.xml`. This is also the *only* write the API
     offers (`sitemaps.submit`) — see `16-search-console.md`.
   - **URL Inspection** → paste homepage → **Request indexing**. Interface only. There is no
     request-indexing endpoint for ordinary web pages in any Google API; the Indexing API covers
     `JobPosting` and `BroadcastEvent` alone (`16-search-console.md`, "The four walls"). Never
     write code or promise output that implies otherwise.
   - Repeat for 3-5 key pages (docs landing, pricing, about). Google confirms a quota exists but
     does not publish the number: *"there's a quota for submitting individual URLs and requesting a
     recrawl multiple times for the same URL won't get it crawled any faster"* (verified
     2026-08-14 — [Ask Google to recrawl](https://developers.google.com/search/docs/crawling-indexing/ask-google-to-recrawl)).
     ⚠️ The "10-12 URLs per day" figure that circulates in SEO writing has no primary source — do
     not quote a number to the user. For anything past a handful of URLs the documented path is the
     sitemap, not the inspection tool.
4. Monitor weekly. Current report names (verified 2026-08-14 — [Page indexing report](https://support.google.com/webmasters/answer/7440203),
   [Reports at a glance](https://support.google.com/webmasters/answer/9133276)):
   - **Page indexing** — this is the report formerly called *Index coverage*. Google's help now
     names it "Page indexing report"; saying "Coverage report" to a user sends them looking for a
     menu entry that is not there.
   - **Performance** — impressions, CTR, average position per query
   - **Core Web Vitals** — still a current report; field data from real Chrome users
   - **Rich result status** reports — one per structured-data feature, plus **Sitemaps**,
     **Links**, **Manual actions**, **Security issues**. ⚠️ The old single "Enhancements" nav group
     is not in Google's current report list, but no Google document states it was removed — treat
     the grouping, not the reports, as uncertain.

## Bing Webmaster Tools setup

1. https://www.bing.com/webmasters → **Add site**
2. **Import from Google Search Console** (fastest — imported sites arrive already verified, up to
   100 per import). Bing re-checks ownership by syncing with the linked GSC account, so if that
   access is later revoked the sites drop back to needing meta-tag or DNS verification (verified
   2026-08-14 — [Import sites from Search Console to Bing Webmaster Tools](https://blogs.bing.com/webmaster/september-2019/Import-sites-from-Search-Console-to-Bing-Webmaster-Tools)).
   This is why GSC comes first.
3. Submit sitemap
4. Enable **IndexNow** (generate key, host as `.txt` on site — see below)

**URL submission quota.** Bing's published ceiling is **10,000 URLs/day per site**, but it is
adaptive, not an entitlement: *"The daily quota per site will be determined based on the site
verified age in Bing Webmaster tool, site impressions and other signals"* (verified 2026-08-14 —
[bingbot Series: submitting up to 10,000 URLs per day](https://blogs.bing.com/webmaster/january-2019/bingbot-Series-Get-your-content-indexed-fast-by-now-submitting-up-to-10,000-URLs-per-day-to-Bing)).
A freshly verified domain gets a small fraction of that. ⚠️ The 2019 announcement remains the most
specific published figure; the current help pages do not restate it, so treat it as a ceiling of
unknown present accuracy rather than today's number.

## IndexNow protocol (not Google)

Participants named by the protocol's own FAQ (verified 2026-08-14 —
[indexnow.org/faq](https://www.indexnow.org/faq)): **Amazon, Bing, Naver, Seznam.cz, Yandex, Yep**.
Submitting to any one endpoint propagates the URL to the others.

**Google is absent from that list.** ⚠️ No Google document says "we do not support IndexNow" — the
evidence is absence from the participant list plus the lack of any support announcement, not a
positive statement. Say that to the user rather than asserting a Google policy.
`16-search-console.md` reaches the same conclusion from the API side.

**Key file** (verified 2026-08-14 — indexnow.org/faq and
[bing.com/indexnow/getstarted](https://www.bing.com/indexnow/getstarted)):

- 8-128 characters from `a-z`, `A-Z`, `0-9`, `-`
- UTF-8 plain-text file at `https://DOMAIN/<key>.txt` containing only the key
- Generate one at https://www.bing.com/indexnow

**Submitting.** Single URL:

```bash
curl "https://api.indexnow.org/indexnow?url=https://DOMAIN/new-page&key=<key>"
```

Batch: POST JSON with `host`, `key`, `keyLocation`, `urlList` — up to **10,000 URLs per POST**.
Wait at least five minutes before resubmitting the same URL; over-submitting earns
`429 Too Many Requests`.

Any one of these endpoints is enough: `api.indexnow.org/indexnow`, `www.bing.com/indexnow`,
`indexnow.amazonbot.amazon/indexnow`, `searchadvisor.naver.com/indexnow`,
`search.seznam.cz/indexnow`, `yandex.com/indexnow`, `indexnow.yep.com/indexnow`.

⚠️ Do not promise "indexed in minutes". The protocol guarantees only that the *notification* is
delivered immediately; whether and when a participant crawls or indexes is not part of it.

Automate as a **step in the deploy pipeline**, not as a Cloudflare trigger.

> ⚠️ **Partly verified, 2026-08-28.** The
> [Workers configuration docs](https://developers.cloudflare.com/workers/configuration/) were read
> on that date and document no "deployment finished" event a Worker can subscribe to. That is an
> absence, not a documented denial — a full enumeration of trigger types was not confirmed, so treat
> "there is no deploy event" as unverified rather than as fact. What *is* certain is that a step
> after `wrangler deploy` works, so use that regardless: a line in the CI job, or a `postdeploy`
> npm script. Whatever already runs the deploy runs the ping.

## Backlink + entity strategy

> These are established practice, not vendor-documented mechanics. No search engine publishes how
> it weights these signals, so present them as conventional advice and not as cause-and-effect.

- **Project README** on the code host links prominently to the site, with an OG-quality screenshot
- **Personal site ↔ brand site** bidirectional `rel="me"` links. This is a real, checkable
  mechanism on the fediverse side: a profile link is marked verified only when the target page
  links back with `rel="me"` (verified 2026-08-14 — [Mastodon verification](https://joinmastodon.org/verification)).
  Its effect on search ranking is not documented by anyone.
- **Launch posts**: Hacker News (Show HN), Product Hunt, a relevant subreddit, dev.to, personal
  blog. ⚠️ Platform health checked 2026-08-14 by search: no shutdown or wind-down announcement
  found for any of these. Absence of such an announcement is weaker than a positive source —
  re-check before a launch plan depends on one.
- **Curated directories** in the vertical: AlternativeTo (apps), awesome-list PRs in the category,
  F-Droid (FOSS Android), Homebrew (macOS CLI)
- **Social profiles** all link back, and are all listed in `Organization.sameAs` JSON-LD array
- **Wikidata entry** (if notable) — feeds structured facts that Google's Knowledge Graph draws on;
  curation takes weeks. ⚠️ The common claim that it is the *strongest* entity signal available is
  SEO-community consensus; Google publishes no ranking of entity signals. Do not repeat the
  superlative to a user.

## Realistic timeline

⚠️ **Nothing in this section is vendor-published.** Google explicitly does not commit to a crawl or
indexing schedule, and no search engine publishes time-to-index targets. These are field
observations to set rough expectations with — always label them as estimates when saying them out
loud, and never as a commitment.

- **First Google crawl** after "Request indexing": 1-3 days
- **Brand query top result**: 2-6 weeks for new domains; longer if older competing sites hold the slot
- **Rich snippets appear**: 1-4 weeks after valid schema deploys
- **Knowledge Panel + logo in SERP**: 4-12 weeks with strong entity signals
- **Wikidata → Knowledge Panel**: weeks to months after the entry is accepted

## Reading a "Page indexing" report the user pasted

**The reason→cause table lives in `16-search-console.md` ("From reason to cause"). Do not
reproduce it here or reason from a second copy** — a second copy is how the two files start
disagreeing. Read the reasons there, branch on the enums it names, and never on the localized
`coverageState` prose.

What this file adds is the arithmetic to run first:

1. Sum the row counts and compare against the sitemap entry count to gauge the indexing rate.
2. Account for **generated variants** — slashless twins, `www`/apex, `http`, `index.html` — before
   calling anything a defect. On a healthy site these dominate "Page with redirect" and the count
   is *expected*, not broken (`16-search-console.md`, "The verdict that is usually right").
3. Only then separate **technical fixes** (Page with redirect / Redirect error / Soft 404 /
   Duplicate) from **content and authority questions** (Discovered- / Crawled-not-indexed).
   Conflating the two is the most common diagnostic mistake.

Two definitions worth quoting correctly, because the widely repeated versions are wrong (verified
2026-08-14 — [Page indexing report](https://support.google.com/webmasters/answer/7440203)):

- **Discovered – currently not indexed**: Google's documented meaning is that it *"wanted to crawl
  the URL but this was expected to overload the site; therefore Google rescheduled the crawl."* It
  is not documented as a verdict on domain age, authority or link graph. Those may still be worth
  improving — just do not attribute the label to them.
- **Crawled – currently not indexed**: *"It may or may not be indexed in the future; no need to
  resubmit this URL for crawling."* Google gives no reason. Name it, say it is a content/quality
  question, and move on (`16-search-console.md`, wall 4).

## Gotchas

- **Don't over-submit** to GSC — Google states directly that requesting a recrawl repeatedly for
  the same URL will not get it crawled faster (verified 2026-08-14 —
  [Ask Google to recrawl](https://developers.google.com/search/docs/crawling-indexing/ask-google-to-recrawl)).
  Once is enough, and it spends quota you cannot see.
- **TXT verification can take a few minutes** to propagate — if GSC says "not verified" after
  adding, wait 5-10 min and retry.
- **Bing import-from-GSC** only works if GSC is verified first — do GSC before Bing.
- **IndexNow key file** must be served as UTF-8 plain text — a static `.txt`, not a JSON route.
- **Sitemap limits**: 50,000 URLs *or* 50 MB uncompressed, whichever comes first; past either, split
  and use a sitemap index. An index may list up to 50,000 sitemaps, and a site may submit up to 500
  index files (verified 2026-08-14 —
  [Build a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap),
  [Manage large sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/large-sitemaps)).

## Post-submit validation

**There is no API call that returns this report.** The Search Console API has no coverage endpoint
and the BigQuery bulk export carries performance tables only; the status has to be reconstructed
URL by URL through `urlInspection`, with the blind spots that implies. The method, the quotas
(600 QPM / 2,000 QPD per site for URL Inspection) and the honest wording are all in
`16-search-console.md`. Do not write a polling script that assumes otherwise.

Visual check in GSC:

1. **Sitemaps** — status "Success" next to the submitted sitemap URL
2. **Page indexing** — pages under "Indexed", with "Discovered – currently not indexed" expected to
   be non-zero at first
3. **URL Inspection** — paste any URL, read crawl status and last-crawl time. Remember it reports
   the *indexed* version, not the live one: right after a fix it still shows the old state
   (`16-search-console.md`, wall 3).
