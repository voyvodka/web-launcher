# Validation toolkit

*Loaded by `web-launcher` SKILL after every deploy, or when validating any change.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

## 11.1 Curl probe — every deploy

```bash
IP=188.114.97.3  # CF anycast; --resolve bypasses local DNS cache
for p in / /favicon.svg /robots.txt /sitemap.xml /llms.txt /humans.txt /og-cover.png /.well-known/security.txt /.well-known/http-message-signatures-directory /random-unknown-path; do
  printf "%-48s " "DOMAIN$p"
  curl -sI --resolve DOMAIN:443:$IP --max-time 10 "https://DOMAIN$p" | head -1
done

# www → apex redirect (must be 301)
curl -sI --resolve www.DOMAIN:443:$IP "https://www.DOMAIN/test-$(date +%s)" | head -3

# Agent-ready signals
curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN/" | grep -i "^link:"
curl -s  --resolve DOMAIN:443:$IP "https://DOMAIN/robots.txt" | grep -i "^content-signal:"
curl -sI -H "Accept: text/markdown" --resolve DOMAIN:443:$IP "https://DOMAIN/" | grep -i "^content-type:"

# Internal-link redirect-chain probe — every internal href should resolve in 0 hops.
# hops=1 usually means trailing-slash drift (framework forces /foo/ but link is /foo).
# hops≥2 usually means meta-refresh + 308 chain via a legacy alias target missing its slash.
for url in / /docs /download /compare /changelog /community /privacy /license; do
  printf "%-30s " "$url"
  curl -sI -L --max-time 10 -w 'hops=%{num_redirects} status=%{http_code} final=%{url_effective}\n' \
    -o /dev/null --resolve DOMAIN:443:$IP "https://DOMAIN$url"
done
```

Expected:
- All paths **200** except `/random-unknown-path` (200 via SPA fallback) and `www.*` (301)
- `Link:` header present with `rel="alternate"`, `rel="sitemap"`, `rel="security-txt"`
- `Content-Signal:` present in robots.txt
- Content-Type markdown when `Accept: text/markdown`
- **`hops=0`** on every internal-link probe. Any non-zero is trailing-slash drift or a legacy-alias chain — see `09-audit-workflow.md` §1d for the source-grep + fix.

## 11.2 External validator matrix

| Tool | URL | Checks |
|---|---|---|
| **Rich Results Test** | https://search.google.com/test/rich-results | Google-parseable schemas, rich snippet eligibility |
| **Schema.org Validator** | https://validator.schema.org/ | Full JSON-LD syntax + type correctness |
| **OG Preview** | https://www.opengraph.xyz/ or https://metatags.io/ | og:*, twitter:*, image resolves, card renders |
| **Facebook Sharing Debugger** | https://developers.facebook.com/tools/debug/ | og:* parse + force re-scrape fb cache |
| **Twitter/X Card Validator** | https://cards-dev.twitter.com/validator | twitter:card render |
| **LinkedIn Post Inspector** | https://www.linkedin.com/post-inspector/ | LinkedIn share preview |
| **Mobile-Friendly Test** | https://search.google.com/test/mobile-friendly | viewport, tap targets, legible fonts |
| **SSL Labs** | https://www.ssllabs.com/ssltest/ | TLS grade, cipher suites, cert chain — target **A+** |
| **securitytxt.org validator** | https://securitytxt.org/ | `/.well-known/security.txt` syntax |
| **Google Safe Browsing** | https://transparencyreport.google.com/safe-browsing/search | blocklist status |
| **WAVE accessibility** | https://wave.webaim.org/ | a11y errors, contrast, alerts |
| **Hreflang checker** (i18n only) | https://hreflang.org/ | language/region targeting |
| **Is It Agent Ready?** | https://isitagentready.com/ | Agentic-web readiness: Link headers, Content-Signal, Markdown-for-Agents, Web Bot Auth |

## 11.3 Lighthouse deep dive

Run headless with both viewport presets:

```bash
# Desktop
npx lighthouse https://DOMAIN \
  --preset=desktop \
  --output=html --output=json --output-path=lh-desktop \
  --chrome-flags="--headless=new"

# Mobile (Google primarily ranks on mobile)
npx lighthouse https://DOMAIN \
  --preset=mobile \
  --output=html --output=json --output-path=lh-mobile \
  --chrome-flags="--headless=new"

open lh-mobile.report.html
```

**Target scores: 95+ desktop, 90+ mobile** for static sites.

**New category — "Agentic Browsing" (Lighthouse ≥ 13.3.0, default since 7 May 2026).** Standard Lighthouse runs now emit an agent-readiness category alongside the classic four. It evaluates how ready a page is for an AI agent to *operate* it — it does **not** affect Search ranking — and includes a check for an `llms.txt` file (see `03-discoverability-classic.md`) plus other programmatic-access signals. Treat it as the lab-side companion to `isitagentready.com` (§11.2) and to the agent-ready signals in `06-agent-ready.md`.

> **lhci lag**: `@lhci/cli@0.15.1` (current) still bundles Lighthouse **12.6.1**, which has *no* agentic-browsing category — so a `categories:agentic-browsing` assertion in `lighthouserc.json` will not work via lhci yet (and would error). Get the category from the **standalone** `npx lighthouse@latest` for now; add the lhci assertion only once lhci bundles Lighthouse ≥ 13.3.0. Also: pin lhci/Lighthouse versions in CI (`@lhci/cli@<version>`, not `@latest`) so scores stay reproducible.

Common failures and fixes per category:

| Category | Target | Common fail → fix |
|---|---|---|
| **Performance** | ≥95 | Render-blocking CSS → inline critical, defer rest / Large LCP image → preload + `fetchpriority="high"` + AVIF or WebP / Long main thread → remove unused JS, code-split / Slow TTFB → CF edge cache, HTTP/3 / Unused CSS → purge (tailwind), tree-shake / Font swap flash → preload + `font-display: swap` |
| **Accessibility** | 100 | Missing `<html lang>` → add `<html lang="en">` / Low contrast → bump token colors / Missing alt → `alt=""` decorative, descriptive otherwise / Buttons lack name → aria-label / No skip link → `<a class="skip-link" href="#main">` / Headings skip levels → fix h1→h2→h3 order |
| **Best Practices** | ≥95 | Mixed content HTTP → HTTPS rewrites / Console errors → fix / Deprecated APIs → migrate / No CSP → add (we ship one in `08-cloudflare-deploy.md`) / Missing HSTS → we ship one / Images wrong aspect ratio → set width/height attributes |
| **SEO** | 100 | Missing meta description → add per page / No canonical → add / robots.txt blocks → fix / Link lacks discernible name → aria-label / Non-descriptive link text "click here" → rewrite / Font too small on mobile → ≥12px base |

## 11.4 Core Web Vitals (Google ranks on these)

| Metric | Good | Needs improvement | Poor |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | ≤2.5s | 2.5–4.0s | >4.0s |
| **INP** (Interaction to Next Paint) | ≤200ms | 200–500ms | >500ms |
| **CLS** (Cumulative Layout Shift) | ≤0.1 | 0.1–0.25 | >0.25 |

**LCP fixes**:
- Preload hero image: `<link rel="preload" as="image" href="..." fetchpriority="high">`
- Use AVIF/WebP modern formats
- Avoid client-side hydration on hero
- Preconnect to font/CDN origins

**INP fixes**:
- Debounce/throttle handlers
- Split long tasks with `scheduler.yield()` (or `setTimeout(0)` split)
- Avoid synchronous layout thrash in event handlers
- Lazy-load non-critical JS

**CLS fixes**:
- Set explicit `width`/`height` on `<img>` and `<iframe>`
- Reserve space for ads/embeds with aspect-ratio CSS
- Avoid inserting content above the fold post-load
- Use `font-display: optional` or preload fonts to kill swap shift

CF Web Analytics auto-collects field data from real visitors; Lighthouse gives lab data. Both matter — field drives ranking, lab drives debugging.

## 11.5 Accessibility deep audit

Lighthouse catches ~30% of a11y issues. Go deeper:

- **axe DevTools** browser extension — run on every template
- **WAVE** https://wave.webaim.org/ — paste URL
- **Screen reader sanity** — VoiceOver (macOS, Cmd+F5) top-to-bottom on landing:
  - Hierarchy makes sense
  - Headings announce correctly
  - Links have context
- **Keyboard-only nav** — Tab through:
  - Focus visible
  - No keyboard trap
  - Skip-link works
  - Esc closes modals
- **Color contrast** — all text passes WCAG AA (4.5:1 body, 3:1 large). Use Contrast.app or Stark plugin.

## Post-deploy validation sequence

After every deploy, run in this order:

1. **11.1 Curl probes** — all 200 / SPA fallback / 301 as expected
2. **Visual browser check** — flush DNS cache first:
   ```bash
   sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder   # macOS
   ```
3. **11.2 External validators** — Rich Results + Schema + OG preview at minimum
4. **11.3 Lighthouse** — both desktop and mobile presets, target 95+ / 90+
5. **Indexing acceleration** — if content changed meaningfully, request GSC reindexing (see `12-indexing.md`)
6. **(If redirects changed)** Purge Everything in CF Caching before final test
