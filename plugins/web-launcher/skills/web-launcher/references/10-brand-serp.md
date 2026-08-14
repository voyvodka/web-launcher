# Brand search / logo-in-SERP troubleshooting

*Loaded by `web-launcher` SKILL when user asks "my logo doesn't show in Google" or "my brand isn't ranking #1".*

> **Partially verified 2026-08-14 · review by 2026-10-13.** Only the claims carrying an inline
> date were checked in that pass. Everything else predates it and is unverified — check before
> relying on a version number, a token, or a vendor behaviour.

## Reality check (set expectations honestly before promising anything)

| Signal | How to provide | Propagation window |
|---|---|---|
| Organization logo in Knowledge Panel / SERP | `Organization` schema with absolute `logo` URL (≥112×112 px, PNG ≥512 ideal) | 1–4 weeks after re-crawl |
| Brand query rank #1 | Indexed + canonical + `sameAs` profile linking + ≥3 external backlinks | 2–8 weeks for new domains; older competing sites may block longer |
| Sitelinks (sub-page list below main result) | Clear site structure + sitemap + strong internal linking + age | 6+ months typically |
| Rich snippets (Product, Review, Article, Event) | Corresponding schema.org type on the page + valid content | 1–3 weeks after first crawl |

**Key lesson**: brand query ranking is a TIME problem, not a configuration problem. A 1-day-old domain can't out-rank a 10-year-old authoritative site on brand queries — even with perfect SEO. Set honest user expectations.

## Acceleration tactics (all legitimate)

1. **GSC URL Inspection → "Request Indexing"** on top 3-5 URLs (not just homepage). See `12-indexing.md`.

2. **`Organization.sameAs` completeness** — list every profile you own:
   ```json
   "sameAs": [
     "https://github.com/USER",
     "https://twitter.com/USER",
     "https://x.com/USER",
     "https://linkedin.com/in/USER",
     "https://bsky.app/profile/USER.bsky.social",
     "https://mastodon.social/@USER",
     "https://www.producthunt.com/@USER",
     "https://discord.gg/INVITE_CODE"
   ]
   ```
   Google uses this to consolidate entity identity.

3. **Logo hygiene**:
   - Absolute URL (`https://DOMAIN/logo.png` not `/logo.png`)
   - PNG format 512×512+ preferred over SVG for Knowledge Panel
   - Can be served as `/favicon.svg` AND `/logo.png` — put PNG URL in `Organization.logo`

4. **Backlinks from authoritative sources**:
   - GitHub repo README prominently links to site with quality screenshot
   - Hacker News (Show HN), Product Hunt launch, relevant subreddit launch post
   - Personal site using `rel="me"` bidirectional link to brand site (verifies entity identity)

5. **Wikidata entry** (if notable) — one of Google's strongest entity signals. Takes weeks for curation.

6. **Bing Webmaster Tools** — Bing is more permissive with new domains; often faster first ranking than Google. See `12-indexing.md`.

7. **IndexNow protocol** (Bing, Yandex, Naver, Seznam — NOT Google yet) — push URL changes instantly. See `12-indexing.md`.

## Failure modes — diagnose first

Rule these out BEFORE blaming "time":

- **`Organization` schema missing** or `logo` URL is relative → no logo rendered, even months after index
- **Canonical conflict** — `/` and `/index.html` both indexed as separate URLs → Google confused, split rank signal. Fix with consistent `<link rel="canonical">`.
- **Staging `noindex` meta left in production** — grep for `<meta name="robots" content="noindex"` or missing meta with `noindex` baked into SSR
  ```bash
  curl -s https://DOMAIN/ | grep -E 'meta.*robots.*noindex'
  ```
- **`robots.txt` blocks Googlebot specifically** — check for `User-agent: Googlebot\nDisallow: /` leftover from staging config
- **No HTTPS redirect** — http:// and https:// indexed as duplicates, authority split. Fix via CF "Always Use HTTPS" = On.
- **Brand name collides with common word** — entity disambiguation (see `04-geo.md`)
- **Hreflang conflict** (multi-region sites) — wrong hreflang tags confuse Google's region targeting

## Specific diagnostic curl suite

```bash
# 1. JSON-LD present and parseable?
curl -sL https://DOMAIN/ | grep -A 50 'application/ld+json' | head -60

# 2. Organization schema present with absolute logo URL?
curl -sL https://DOMAIN/ | grep -A 30 '"@type": "Organization"'

# 3. Canonical consistent?
curl -sL https://DOMAIN/ | grep 'rel="canonical"'
curl -sL https://DOMAIN/index.html | grep 'rel="canonical"'  # should be same

# 4. noindex anywhere?
curl -sL https://DOMAIN/ | grep -E 'robots.*noindex'

# 5. robots.txt allows Googlebot?
curl -s https://DOMAIN/robots.txt | grep -iA 2 'googlebot'

# 6. HTTPS redirect?
curl -sI http://DOMAIN/ | head -3  # expect 301 → https://DOMAIN/
```

## What to tell the user

If brand query rank is a concern and the site is <6 weeks old:

> "Your site is freshly deployed. Google prioritizes older, authoritative sites first for any brand query — even yours. Here's what's in our control:
> 1. Organization schema is in place (logo will show once re-crawled — 1-4 weeks)
> 2. I've applied sameAs with all your profile links (helps entity consolidation)
> 3. Requested indexing via GSC
> What's out of our direct control: time + backlinks. Each Hacker News / Product Hunt mention / authoritative backlink accelerates ranking. Realistic expectation: 2-6 weeks for brand query #1 if no older competing site holds the slot."
