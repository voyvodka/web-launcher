# Multi-page SEO (docs, blog, marketing with subpages)

*Loaded by `web-launcher` SKILL when site has multiple routes (docs/, blog/, marketing subpages).*

> **Partially verified 2026-08-14 · review by 2026-10-13.** Only the claims carrying an inline
> date were checked in that pass. Everything else predates it and is unverified — check before
> relying on a version number, a token, or a vendor behaviour.

Single-page baseline from `03-discoverability-classic.md` is insufficient once you have routes. Every route needs its own meta + canonical + og + schema.

## Per-route requirements

- **Per-page canonical** — every route sets its own `<link rel="canonical" href="https://DOMAIN/route">`
- **Per-page `<title>`** (≤60 chars) + **meta description** (≤160 chars) — unique per page, not a templated fallback
- **Per-page `og:title` / `og:description` / `og:image`** — different social cards per page (otherwise every share looks identical)
- **Breadcrumbs** — both UI and `BreadcrumbList` schema:

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Docs", "item": "https://DOMAIN/docs" },
    { "@type": "ListItem", "position": 2, "name": "Getting Started", "item": "https://DOMAIN/docs/getting-started" }
  ]
}
</script>
```

## Sitemap.xml with all routes

Not just root. Framework integrations:

| Framework | Plugin / built-in |
|---|---|
| Astro | `@astrojs/sitemap` — auto from `src/pages/` |
| Next.js | `next-sitemap` — post-build generator |
| Hugo | Built-in — `hugo` command emits `/sitemap.xml` |
| Eleventy | `eleventy-plugin-sitemap` |
| SvelteKit | `svelte-sitemap` or manual endpoint |
| Plain static | Hand-write or generate from filesystem |

## Article schema on blog / changelog posts

```json
{
  "@type": "Article",
  "headline": "…",
  "datePublished": "2026-01-15",
  "dateModified": "2026-02-01",
  "author": { "@id": "https://DOMAIN/#person-author" },
  "image": "https://DOMAIN/og/article-slug.png",
  "publisher": { "@id": "https://DOMAIN/#org" }
}
```

## Internal linking strategy

- **Hub → spoke**: `/docs` → `/docs/*`. Every page has ≥2 internal links in/out.
- **No orphan pages**: every public route is reachable from at least one other public route.
- **Footer + inline contextual links**: footer repeats main nav; inline contextual links within content.

Verification:
```bash
# Find orphan pages (pages not linked from any other page)
# For Astro/Next: grep content collection + cross-ref pages/
grep -rn 'href="/docs/' src/ | awk -F'href="' '{print $2}' | awk -F'"' '{print $1}' | sort -u > linked.txt
ls src/pages/docs/*.md src/pages/docs/*.mdx | awk -F'/' '{print "/"$NF}' | sed 's/\.mdx$//;s/\.md$//' > all.txt
diff linked.txt all.txt  # missing from linked = orphan
```

## Dynamic OG images per page

Satori pattern with per-page title / category / excerpt. Most sites need 2-4 templates:
- Landing (static)
- Doc / article (title + category)
- Compare (X vs Y)
- Blog post (title + date + author)

See `07-og-satori.md` for generator pattern — extend with per-page `entry` parameter.

Example for an Astro API route at `/og/[...route].ts`:
```ts
export const getStaticPaths = () =>
  entries.map(e => ({ params: { route: `${e.id}.png` }, props: { id: e.id } }));

export const GET = async ({ props }) => {
  const entry = pages[props.id];
  const svg = await satori(template(entry), { width: 1200, height: 630, fonts });
  return new Response(new Resvg(svg, { fitTo: { mode: "width", value: 1200 } }).render().asPng(),
    { headers: { "Content-Type": "image/png", "Cache-Control": "public, max-age=31536000, immutable" }});
};
```

## Route-level robots meta

- Staging / draft / preview pages: `<meta name="robots" content="noindex,nofollow">` in the page's `<head>`
- Public: omit (default is index,follow)
- `robots.txt` Disallow is too coarse for per-page — use meta instead for fine control

## Per-route audit checklist

For each indexed route:
- [ ] Unique `<title>` (≤60 chars)
- [ ] Unique `meta description` (≤160 chars)
- [ ] `<link rel="canonical">` points to this URL
- [ ] `og:title` / `og:description` / `og:image` — unique per page
- [ ] Appropriate JSON-LD: `Article` for posts, `BreadcrumbList` everywhere nested. **Not `HowTo`, not `FAQPage`** — Google renders neither (see `04-geo.md`, verified 2026-08-14)
- [ ] ≥2 internal links in or out
- [ ] All internal `href` values match the framework's trailing-slash policy (no 308 chains — see `09-audit-workflow.md` §1d)
- [ ] Not accidentally noindex from staging leftover

Verification curl:
```bash
for route in / /docs /docs/getting-started /blog /compare; do
  echo "=== $route ==="
  curl -sL "https://DOMAIN$route" | grep -E '<(title|link rel="canonical"|meta (name|property)="(description|og:|twitter:))' | head -10
done
```
