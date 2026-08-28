# Validation toolkit

*Loaded by `web-launcher` SKILL after every deploy, or when validating any change.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

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
- All real paths **200**; `www.*` **301**
- `/random-unknown-path` — **200** on an SPA (index fallback), **404** on a static or SSR site. Decide
  which shape the site is *before* reading this line, otherwise it proves nothing either way.
- `Link:` header present with `rel="alternate"`, `rel="sitemap"`, `rel="security-txt"`
- `Content-Signal:` present in robots.txt
- Content-Type markdown when `Accept: text/markdown`
- **`hops=0`** on every internal-link probe. Any non-zero is trailing-slash drift or a legacy-alias chain — see `09-audit-workflow.md` §1d for the source-grep + fix.

> Probe syntax executed against a live public host on **2026-08-14** (curl 8.x / LibreSSL, macOS):
> the `--resolve` form, the `-w 'hops=%{num_redirects} status=%{http_code} final=%{url_effective}'`
> format string, and every `grep` above run as written. A `grep` that finds nothing exits 1 and
> prints nothing — that is a *finding* (header absent), not a broken command.

These probes overlap deliberately with `14-diagnostic-checks.md`, which is the deeper set. Use
§11.1 as the fast post-deploy smoke test; when a probe here comes back wrong, switch to
`14-diagnostic-checks.md` (C1–C10) to find the cause rather than debugging it from this file.

## 11.2 External validator matrix

Every row below was probed on **2026-08-14**; `(checked …)` means the URL was requested and
answered, not that the tool's output was independently judged correct.

| Tool | URL | Checks |
|---|---|---|
| **Rich Results Test** | https://search.google.com/test/rich-results *(checked 2026-08-14)* | Google-parseable schemas, rich snippet eligibility |
| **Schema.org Validator** | https://validator.schema.org/ *(checked 2026-08-14)* | Full JSON-LD syntax + type correctness |
| **PageSpeed Insights** | https://pagespeed.web.dev/ *(checked 2026-08-14)* | Field CrUX data **and** a Lighthouse lab run in one report — the only place to see both side by side |
| **OG Preview** | https://www.opengraph.xyz/ or https://metatags.io/ *(both checked 2026-08-14)* | og:*, twitter:*, image resolves, card renders |
| **Facebook Sharing Debugger** | https://developers.facebook.com/tools/debug/ *(checked 2026-08-14 ⚠️)* | og:* parse + force re-scrape of the Meta cache. ⚠️ Endpoint is live but login-gated — an anonymous request returns 400, so only the *reachability* of the tool was verified, not its behaviour |
| **LinkedIn Post Inspector** | https://www.linkedin.com/post-inspector/ *(checked 2026-08-14)* | LinkedIn share preview + cache re-scrape |
| **Nu HTML Checker** (W3C) | https://validator.w3.org/nu/ *(checked 2026-08-14)* | HTML syntax, unclosed tags, invalid attribute values |
| **CSS Validator** (W3C) | https://jigsaw.w3.org/css-validator/ *(checked 2026-08-14)* | CSS syntax against the current spec profile |
| **SSL Labs** | https://www.ssllabs.com/ssltest/ *(checked 2026-08-14)* | TLS grade, cipher suites, cert chain — target **A+**. Scriptable via `https://api.ssllabs.com/api/v4/` |
| **MDN HTTP Observatory** | https://developer.mozilla.org/en-US/observatory *(checked 2026-08-14)* | Security-header grade: CSP, HSTS, `X-Content-Type-Options`, cookie flags. **Moved** — the old `observatory.mozilla.org` host now redirects here; update any bookmark still pointing at it |
| **securitytxt.org validator** | https://securitytxt.org/ *(checked 2026-08-14)* | `/.well-known/security.txt` syntax per RFC 9116 |
| **Google Safe Browsing** | https://transparencyreport.google.com/safe-browsing/search *(checked 2026-08-14)* | blocklist status |
| **WAVE accessibility** | https://wave.webaim.org/ *(checked 2026-08-14)* | a11y errors, contrast, alerts |
| **Hreflang checker** (i18n only) | https://hreflang.org/ *(checked 2026-08-14)* | language/region targeting |
| **Is It Agent Ready?** | https://isitagentready.com/ *(checked 2026-08-14)* | Agentic-web readiness: Link headers, Content-Signal, Markdown-for-Agents, Web Bot Auth, MCP/WebMCP discovery. Operated by Cloudflare, and its recommendations are LLM-generated — the page says so itself, so treat its advice as a lead, not a verdict |

**Removed in the 2026-08-14 audit** — do not re-add these; both were checked and are gone:

| Tool | What the probe returned | Do this instead |
|---|---|---|
| ~~Mobile-Friendly Test~~ (`search.google.com/test/mobile-friendly`) | **Retired.** The URL now redirects to the Lighthouse documentation *(checked 2026-08-14)* | Lighthouse's mobile run (§11.3) plus the SEO category's font-size and tap-target audits cover what it used to report |
| ~~Twitter/X Card Validator~~ (`cards-dev.twitter.com/validator`) | **Login-gated and unmaintained.** Redirects to an X login page; the preview feature was withdrawn and X has shipped no replacement *(checked 2026-08-14)* | Validate `twitter:*` tags with the OG preview tools above, then confirm the real render by pasting the URL into the X composer — the card renders without posting |

⚠️ **`securityheaders.com` is deliberately absent.** It returned 403 to every probe attempted on
2026-08-14, so whether it is bot-blocking or offline could not be determined. MDN HTTP Observatory
covers the same ground and did verify.

## 11.3 Lighthouse deep dive

Current release: **Lighthouse 13.4.1**, published 2026-07-20 (`npm view lighthouse dist-tags`,
*checked 2026-08-14*). Pin this in CI rather than tracking `@latest` — scores are not comparable
across versions.

Run headless in both form factors:

```bash
# Desktop
npx lighthouse@13.4.1 https://DOMAIN \
  --preset=desktop \
  --output=html --output=json --output-path=lh-desktop \
  --chrome-flags="--headless=new"

# Mobile — the DEFAULT form factor, so pass no preset at all.
# Google primarily ranks on mobile, which is why this run is the one that matters.
npx lighthouse@13.4.1 https://DOMAIN \
  --output=html --output=json --output-path=lh-mobile \
  --chrome-flags="--headless=new"

open lh-mobile.report.html
```

> **There is no `--preset=mobile`.** `--preset` accepts exactly `perf`, `experimental`, `desktop`
> — verified in `cli/cli-flags.js` of the published 13.4.1 tarball, *2026-08-14*. Passing
> `--preset=mobile` fails argument validation; mobile emulation is what you get by default.

**Target scores: 95+ desktop, 90+ mobile** for static sites.

### The "Agentic Browsing" category

A fifth category, **Agentic Browsing**, sits alongside Performance / Accessibility / Best Practices
/ SEO in the standard config. Verified 2026-08-14 by extracting `core/config/default-config.js`
from the published npm tarballs: the category is **absent in 13.2.0 and earlier, present from
13.3.0 onward** (13.3.0 published 2026-05-07). It needs no flag and no custom config.

It scores how ready a page is for an AI agent to *operate* it. Its audits, read from
`default-config.js` in 13.4.1 (*checked 2026-08-14*):

| Audit | Group |
|---|---|
| `llms-txt` | Agent Accessibility — the `llms.txt` file from `03-discoverability-classic.md` |
| `agent-accessibility-tree` | Agent Accessibility |
| `webmcp-registered-tools` | WebMCP |
| `webmcp-form-coverage` | WebMCP |
| `webmcp-schema-validity` | WebMCP |
| `cumulative-layout-shift` | (reused from Performance) |

Two caveats, both from Lighthouse's own category description:

- It is scored as a **fraction**, not a 0–100 score — do not set a numeric threshold against it.
- Lighthouse labels it *"still under development and subject to change."* Treat a regression here
  as a signal to investigate, never as a release blocker.
- Four of its six audits target **WebMCP**, which this skill does not scaffold. A site not shipping
  WebMCP scores low here by design; that is a scope decision, not a defect to fix.

It does **not** affect Search ranking. Treat it as the lab-side companion to `isitagentready.com`
(§11.2) and to the agent-ready signals in `06-agent-ready.md`.

> **lhci lag is real and still unresolved.** `@lhci/cli@0.15.1` (latest) declares
> `"lighthouse": "12.6.1"` as a pinned dependency — read from the package manifest on the npm
> registry, *checked 2026-08-14*. Lighthouse 12.6.1 has no agentic-browsing category, so a
> `categories:agentic-browsing` assertion in `lighthouserc.json` errors rather than failing
> cleanly. Get the category from the **standalone** `npx lighthouse@13.4.1` until lhci bundles
> ≥ 13.3.0. Pin both tools by exact version in CI (`@lhci/cli@0.15.1`, not `@latest`).

Common failures and fixes per category:

| Category | Target | Common fail → fix |
|---|---|---|
| **Performance** | ≥95 | Render-blocking CSS → inline critical, defer rest / Large LCP image → preload + `fetchpriority="high"` + AVIF or WebP / Long main thread → remove unused JS, code-split / Slow TTFB → CF edge cache, HTTP/3 / Unused CSS → purge (tailwind), tree-shake / Font swap flash → preload + `font-display: swap` |
| **Accessibility** | 100 | Missing `<html lang>` → add `<html lang="en">` / Low contrast → bump token colors / Missing alt → `alt=""` decorative, descriptive otherwise / Buttons lack name → aria-label / No skip link → `<a class="skip-link" href="#main">` / Headings skip levels → fix h1→h2→h3 order |
| **Best Practices** | ≥95 | Mixed content HTTP → HTTPS rewrites / Console errors → fix / Deprecated APIs → migrate / No CSP → add (we ship one in `08-cloudflare-deploy.md`) / Missing HSTS → we ship one / Images wrong aspect ratio → set width/height attributes |
| **SEO** | 100 | Missing meta description → add per page / No canonical → add / robots.txt blocks → fix / Link lacks discernible name → aria-label / Non-descriptive link text "click here" → rewrite / Font too small on mobile → ≥12px base |

## 11.4 Core Web Vitals (Google ranks on these)

Three metrics, all at *stable* lifecycle stage — verified against the web.dev Core Web Vitals
article, *2026-08-14*. No fourth metric is pending.

| Metric | Good | Needs improvement | Poor |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | ≤2.5s | 2.5–4.0s | >4.0s |
| **INP** (Interaction to Next Paint) | ≤200ms | 200–500ms | >500ms |
| **CLS** (Cumulative Layout Shift) | ≤0.1 | 0.1–0.25 | >0.25 |

**FID is gone.** INP replaced it as the interactivity metric — promoted to stable in 2024, and FID
retired. Any advice, config, or dashboard still naming FID is out of date; the fixes below are the
INP-era ones.

The threshold must be met at the **75th percentile** of page loads, segmented separately for mobile
and desktop. A passing median is not a passing site — this is the single most common
misreading of a CrUX report.

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

Lighthouse gives **lab** data — one synthetic load, reproducible, good for debugging. Field data
comes from real visitors and is what ranking uses. PageSpeed Insights (§11.2) shows both in one
report, which is the fastest way to see a site that scores 98 in the lab and fails in the field.

⚠️ Whether a given analytics product reports Core Web Vitals field data, and at what sampling
rate, is vendor behaviour that was not re-verified here — check its current documentation before
relying on its numbers. CrUX, surfaced through PageSpeed Insights, is the source Google itself
ranks on.

## 11.5 Accessibility deep audit

Automated tooling catches only a fraction of real accessibility defects — keyboard traps, focus
order and screen-reader coherence are largely invisible to it. ⚠️ The often-quoted "~30%" figure
is not sourced here; treat "Lighthouse passing" as necessary, never sufficient, and go deeper:

- **axe DevTools** browser extension — https://www.deque.com/axe/devtools/ *(checked 2026-08-14)*, run on every template
- **WAVE** https://wave.webaim.org/ *(checked 2026-08-14)* — paste URL
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
4. **11.3 Lighthouse** — desktop (`--preset=desktop`) and mobile (no preset), target 95+ / 90+
5. **Indexing acceleration** — if content changed meaningfully, request GSC reindexing (see
   `12-indexing.md`). **Once per URL per cycle, not once per deploy.** `12-indexing.md` warns
   against re-submitting the same URL in a short window; two deploys in a week to the same page is
   one request, not two. Skip this step for a URL already submitted since its last meaningful
   content change.
6. **(If redirects changed)** Purge Everything in CF Caching before final test
