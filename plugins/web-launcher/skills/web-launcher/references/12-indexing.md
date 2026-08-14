# Indexing acceleration

*Loaded by `web-launcher` SKILL after initial deploy to accelerate discovery.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

## Google Search Console setup

1. https://search.google.com/search-console → **Add property** → **Domain** type (covers all subdomains + protocols)
2. Verify via TXT record:
   - CF Dashboard → DNS → Add TXT
   - Name: `@`
   - Content: paste `google-site-verification=...` string from GSC
   - Save → back to GSC → **Verify**
3. Post-verification:
   - **Sitemaps** → submit `https://DOMAIN/sitemap.xml`
   - **URL Inspection** → paste homepage → **Request Indexing**
   - Repeat Request Indexing for 3-5 key pages (docs landing, pricing, about)
4. Monitor weekly:
   - **Coverage** report — fix any errors immediately
   - **Performance** report — impressions, CTR, average position per query
   - **Core Web Vitals** report — field data from real Chrome users
   - **Enhancements** — rich results eligibility

## Bing Webmaster Tools setup

1. https://www.bing.com/webmasters → **Add site**
2. **Import from GSC** (fastest — inherits verification)
3. Submit sitemap
4. Enable **IndexNow** (generate key, host as `.txt` on site — see below)

## IndexNow protocol (Bing, Yandex, Naver, Seznam — NOT Google)

- Generate key at https://www.bing.com/indexnow
- Host key file: `https://DOMAIN/<key>.txt` containing just the key as plain text
- On every publish:
  ```bash
  curl "https://api.indexnow.org/indexnow?url=https://DOMAIN/new-page&key=<key>"
  ```
  Indexed in minutes.
- Automate via deploy hook (CF Worker on wrangler deploy event, or GitHub Actions)

## Backlink + entity strategy

- **GitHub repo README** prominently links to site, includes OG-quality screenshot
- **Personal site ↔ brand site** bidirectional `rel="me"` links (Mastodon-style verified identity)
- **Launch posts**:
  - Hacker News (Show HN)
  - Product Hunt
  - Relevant subreddit
  - dev.to
  - Personal blog
- **Submit to curated directories** in your vertical:
  - AlternativeTo (for apps)
  - Awesome-lists in the category (PRs to `awesome-X` repos)
  - F-Droid (if FOSS Android)
  - Homebrew (if macOS CLI)
- **Social profiles** all link back, and are all listed in `Organization.sameAs` JSON-LD array
- **Wikidata entry** (if notable) — strongest Google entity signal available (takes weeks to curate)

## Realistic timeline (set with user honestly)

- **First Google crawl** after "Request Indexing": 1-3 days
- **Brand query top result**: 2-6 weeks for new domains; longer if older competing sites hold the slot
- **Rich snippets appear**: 1-4 weeks after valid schema deploys
- **Knowledge Panel + logo in SERP**: 4-12 weeks with strong entity signals
- **Wikidata → Knowledge Panel**: weeks to months after Wikidata entry accepted

## GSC "Page indexing" categories — what each one means and how to fix

When the user shares a GSC report, map each "Why pages aren't indexed" reason to a root cause before recommending action. The categories look similar but call for very different fixes.

| GSC category (EN / TR) | What Google did | Root cause | Fix lives in |
|---|---|---|---|
| **Discovered – currently not indexed** / *Keşfedildi - şu anda dizine eklenmiş değil* | Saw URL (sitemap or referral), did not crawl yet | Crawl-budget throttling: young domain, low authority, weak internal-link graph, internal-link 308s eating budget | Backlink + brand signal + Step 1d redirect-chain fix |
| **Crawled – currently not indexed** / *Tarandı - şu anda dizine eklenmiş değil* | Crawled the page, decided not to index | Quality / uniqueness signal: thin content, list-only changelog, hub page that just routes to detail pages | Strengthen on-page content; not a technical fix |
| **Page with redirect** / *Yönlendirmeli sayfa* | URL responded with a redirect | Slashless internal link, legacy alias not in sitemap, or sitemap entry that 30x's | Source-grep slashless `href`s; fix `redirects` map targets to canonical form; remove dead URLs from sitemap |
| **Redirect error** / *Yeniden yönlendirme hatası* | Redirect chain too long, looped, or final URL failed | Almost always meta-refresh + 308 = 2-hop, or framework redirects pointing at slashless target which then 308s to slash form | Update framework `redirects` config so every target matches the `trailingSlash` policy |
| **Alternate page with proper canonical tag** / *Doğru standart etikete sahip alternatif sayfa* | URL has canonical pointing elsewhere, Google indexed the canonical instead | Expected on legacy redirect HTML pages (canonical points to destination) — usually not actionable. Investigate only if it's a real content URL whose canonical accidentally points away | Verify canonical is intended; fix if accidental |
| **Soft 404** | Page returned 200 but looks empty/error | SPA fallback that doesn't 404 properly, or genuinely thin page | Return real 404 status, or add unique content |
| **Duplicate without user-selected canonical** | Multiple URLs with same content, no canonical declared | Missing canonical or canonical pointing to itself instead of the consolidated URL | Add explicit `<link rel="canonical">` everywhere |
| **Blocked by robots.txt** | Disallow directive matched | Intentional or stale rule | Audit robots.txt against current paths |
| **Excluded by 'noindex'** | Page has noindex meta | Intentional (drafts, legacy redirect HTML) or stale staging leftover | Remove if accidental; ignore if intentional |

When the user pastes the GSC summary panel:
1. Sum the row counts and compare against sitemap entry count to gauge indexing-rate.
2. For each non-zero row, name the root cause from this table before suggesting action.
3. Distinguish **technical fixes** (Page with redirect / Redirect error / Soft 404 / Duplicate) from **authority/content fixes** (Discovered- / Crawled-not-indexed). Conflating them is the most common diagnostic mistake.

## Gotchas

- **Don't over-submit** to GSC — repeatedly hitting "Request Indexing" for the same URL doesn't accelerate; it just gets queued. Once is enough.
- **TXT verification can take a few minutes** to propagate — if GSC says "not verified" after adding, wait 5-10 min and retry.
- **Bing import-from-GSC** only works if GSC is verified first — do GSC before Bing.
- **IndexNow key file MIME** must be `text/plain` — serve as static file, not JSON.
- **Sitemaps with >50k URLs** need sitemap index files (`sitemapindex` root). Rare for static sites but relevant for large docs.

## Post-GSC-submit validation

```bash
# After submitting sitemap, poll GSC API for index status (if API access enabled)
# Or manually check GSC Coverage report after 24-72 hours
```

Visual check in GSC:
1. Sitemaps tab: status "Success" next to your sitemap URL
2. Coverage tab: pages listed as "Indexed" or "Discovered — currently not indexed" (normal initially)
3. URL Inspection: paste any URL, see crawl status + "Last crawl" timestamp
