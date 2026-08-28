# Classic SEO discoverability — the standard set

*Loaded by `web-launcher` SKILL for every site. Read this first for SEO-related work.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

Every site gets these files + meta tags. Templates below. See `04-geo.md` for GEO extensions,
`06-agent-ready.md` for 2025 agent-ready signals.

**Load `05-multipage-seo.md` as well the moment the site has more than one indexable URL** — more
than one entry in `sitemap.xml`, or more than one file in `src/pages/` (or the framework's
equivalent). That is the bright line; it is not a judgement about whether the site "feels" like a
multi-page site. A two-page site needs per-route canonical and `og:url` exactly as much as a
fifty-page one, and the meta template below is written for the single-page case.

## `robots.txt`

Allow all + explicit AI crawler acknowledgement.

**Token accuracy matters more than the directive.** Every token below was checked against the
operator's own published list on 2026-08-14. A dead token is not harmless: it reads as coverage,
so anyone who later flips this template to `Disallow` blocks nothing while believing they did.

```
# PROJECT — https://DOMAIN
# Open to discovery by classic search crawlers AND AI agents.

User-agent: *
Allow: /

# OpenAI — verified 2026-08-14 (https://developers.openai.com/api/docs/bots)
User-agent: GPTBot
Allow: /
User-agent: ChatGPT-User
Allow: /
User-agent: OAI-SearchBot
Allow: /
User-agent: OAI-AdsBot
Allow: /

# Anthropic — verified 2026-08-14 (https://support.claude.com/en/articles/8896518)
User-agent: ClaudeBot
Allow: /
User-agent: Claude-User
Allow: /
User-agent: Claude-SearchBot
Allow: /

# Others — token names from operator lists, not re-verified individually
User-agent: Google-Extended
Allow: /
User-agent: Applebot-Extended
Allow: /
User-agent: PerplexityBot
Allow: /
User-agent: CCBot
Allow: /
User-agent: Amazonbot
Allow: /

Sitemap: https://DOMAIN/sitemap.xml
```

**Removed on 2026-08-14, do not put back:**

| Token | Why |
|-------|-----|
| `Claude-Web` | Retired. Anthropic's current list has only `ClaudeBot`, `Claude-User`, `Claude-SearchBot` |
| `anthropic-ai` | Same — retired token, listing it gives false coverage |
| `cohere-ai` | Could not be confirmed in any operator-published list; unverifiable, so it is out |
| `Bytespider` | Token is real, but it is widely reported to ignore `robots.txt`. Listing it implies a control that does not exist |

**`Content-Signal` is deliberately not in this template.** It is a policy declaration with no
technical effect today — see `06-agent-ready.md`. Add it if the site wants to state intent, not
because it changes crawler behaviour.

## `sitemap.xml`

Minimal for single-page:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://DOMAIN/</loc>
    <lastmod>YYYY-MM-DD</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

Multi-page: see `05-multipage-seo.md` — use framework integrations (`@astrojs/sitemap`, `next-sitemap`, Hugo native).

## `llms.txt` (Markdown summary for LLMs)

Emerging convention (llmstxt.org). Factual, scannable. This is also the GEO primary surface (see `04-geo.md`).

**Reality check (2026) — don't oversell it.** Google confirmed (15 Jun 2026 AI-optimization guide update) that `llms.txt` has **no effect, positive or negative, on Search rankings or AI Overviews** — Search ignores it. Bot-traffic studies show requests touching `/llms.txt` are statistically negligible vs total LLM crawler volume. So it is **not** an SEO or GEO *ranking* factor — never was. Where it still earns its place: (a) the **agentic web / B2A layer** — agents acting on a user's behalf can consume a clean entry brief, and (b) **Chrome Lighthouse's "Agentic Browsing" audit** (default since 13.3.0, 7 May 2026) checks for its existence. Ship it as cheap agent-facing infrastructure; never pitch it to a user as a rankings lever.

```markdown
# PROJECT_NAME

> One-sentence description of what this is.

Longer paragraph with features, stack, and context.

## Primary links

- [GitHub](https://github.com/USER/REPO)
- [Landing](https://DOMAIN)
- [Latest release](https://github.com/USER/REPO/releases/latest)

## Project facts

- License: LICENSE_ID
- Maintainer: NAME / HANDLE
- Stack: PRIMARY_STACK (framework, language, runtime)
- Platforms: WHERE_IT_RUNS
- Data posture: e.g. local-only, no telemetry
- Category: 3–6 terms someone would actually search for, including the category name and the
  closest well-known alternative
```

Keep it under ~1000 words. Entity-dense, avoid prose filler. LLMs extract verbatim.

## `.well-known/security.txt` (RFC 9116)

```
Contact: https://github.com/USER/REPO/security/advisories/new
Contact: mailto:REAL_EMAIL
Preferred-Languages: en
Canonical: https://DOMAIN/.well-known/security.txt
Expires: <12-to-18-months-from-now>T00:00:00.000Z
Policy: https://github.com/USER/REPO/blob/main/SECURITY.md
```

**Verify before including**:
- Email address actually exists (don't list `security@DOMAIN` if no MX record — bounces)
- `SECURITY.md` exists in the repo root

## `humans.txt`

Credits file. Short.
```
/* TEAM */
Maintainer: NAME
GitHub: https://github.com/USER
From: Internet

/* SITE */
Last update: YYYY/MM/DD
Doctype: HTML5
Language: English
Status: production
Source: static (Cloudflare Workers)

/* STACK */
App: Tauri 2 + Rust + React
Site: Astro 5

/* THANKS */
Adalight protocol: Adafruit
Typography: IBM Plex Sans, IBM Plex Mono, Instrument Serif
```

## JSON-LD structured data

Typical triad for software / marketing projects, in `<head>`:

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "WebSite",
      "@id": "https://DOMAIN/#website",
      "url": "https://DOMAIN",
      "name": "PROJECT",
      "description": "…",
      "inLanguage": "en"
    },
    {
      "@type": "SoftwareApplication",
      "@id": "https://DOMAIN/#app",
      "name": "PROJECT",
      "url": "https://DOMAIN",
      "applicationCategory": "UtilitiesApplication",
      "operatingSystem": "macOS, Windows, Linux",
      "description": "…",
      "offers": { "@type": "Offer", "price": "0", "priceCurrency": "USD" },
      "license": "https://opensource.org/licenses/MIT",
      "sameAs": "https://github.com/USER/REPO"
    },
    {
      "@type": "Organization",
      "@id": "https://DOMAIN/#org",
      "name": "MAINTAINER_NAME",
      "url": "https://MAINTAINER_URL",
      "logo": "https://DOMAIN/logo.png",
      "sameAs": [
        "https://github.com/USER",
        "https://twitter.com/USER",
        "https://linkedin.com/in/USER"
      ]
    }
  ]
}
</script>
```

**Critical**: `Organization.logo` absolute URL (not relative) is what makes your logo appear in Google Knowledge Panel / SERP. See `10-brand-serp.md` for troubleshooting.

Validate after deploy with:
- https://search.google.com/test/rich-results
- https://validator.schema.org/

## Full meta head (required every site)

```html
<meta name="description" content="…≤160 char…" />
<meta name="theme-color" content="#HEX" />

<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
<!-- These two are PNG on purpose and must not be switched to favicon.svg:
     apple-touch-icon is not rendered from SVG by iOS, and Google's Organization.logo
     must be a raster image to appear in the Knowledge Panel. `01-brand-application.md`
     and `10-brand-serp.md` carry the sourced versions of both claims. -->
<link rel="mask-icon" href="/favicon.svg" color="#ACCENT" />

<meta name="application-name" content="PROJECT" />
<meta name="apple-mobile-web-app-title" content="PROJECT" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="author" content="NAME" />
<meta name="format-detection" content="telephone=no" />
<link rel="author" href="/humans.txt" />
<link rel="canonical" href="https://DOMAIN" />              <!-- per-route on multi-page: see 05 -->

<meta property="og:type" content="website" />
<meta property="og:title" content="PROJECT — tagline" />
<meta property="og:description" content="…" />
<meta property="og:url" content="https://DOMAIN" />          <!-- per-route on multi-page: see 05 -->
<meta property="og:site_name" content="PROJECT" />
<meta property="og:locale" content="en_US" />
<meta property="og:image" content="https://DOMAIN/og-cover.png" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:image" content="https://DOMAIN/og-cover.png" />
```

## Verification

After deploy, run (see `11-validation-toolkit.md` for full suite):
```bash
curl -sI https://DOMAIN/robots.txt    # 200
curl -sI https://DOMAIN/sitemap.xml   # 200
curl -sI https://DOMAIN/llms.txt      # 200
curl -sI https://DOMAIN/.well-known/security.txt  # 200
curl -sL https://DOMAIN/ | grep -E 'application/ld\+json' # JSON-LD found
```
