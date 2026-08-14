# Mode B — Existing site audit workflow

*Loaded by `web-launcher` SKILL for Mode B (audit & fix of an existing deployed site).*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

When user points at a deployed site and asks for improvements, run this sequence.

## Step 1: Live probe (HTTP + meta harvest)

```bash
IP=188.114.97.3  # CF anycast IP; change if origin not behind CF
for p in / /robots.txt /sitemap.xml /llms.txt /humans.txt /og-cover.png /favicon.svg /.well-known/security.txt /nonexistent-xyz; do
  printf "%-36s " "$p"
  curl -sI --resolve DOMAIN:443:$IP --max-time 10 "https://DOMAIN$p" | head -1
done

# Harvest head: meta, link, JSON-LD blocks
curl -sL --resolve DOMAIN:443:$IP "https://DOMAIN/" | \
  grep -E '<(title|meta|link rel|script type="application/ld\+json")[^>]*' | head -80
```

## Step 1b: Agent-ready probe (2025 agentic-web signals)

See `06-agent-ready.md` for what these signals mean.

```bash
# RFC 8288 Link headers on homepage
curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN/" | grep -i "^link:"

# Content-Signal directive in robots.txt
curl -s  --resolve DOMAIN:443:$IP "https://DOMAIN/robots.txt" | grep -i "^content-signal:"

# Markdown-for-Agents content negotiation
curl -sI -H "Accept: text/markdown" --resolve DOMAIN:443:$IP "https://DOMAIN/" | grep -i "^content-type:"
# Expect: text/markdown; charset=utf-8 (not text/html)

# Web Bot Auth JWKS
curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN/.well-known/http-message-signatures-directory" | grep -i "^content-type:"
# Expect: application/json (not HTML fallback)
```

Also run the hosted Cloudflare scanner for a UI report: `https://isitagentready.com/?url=DOMAIN`

## Step 1c: Repo audit (if user has local checkout)

See `13-dependency-security.md` §11.11 for rapid checklist. Quick version:

```bash
# Dep-bot present?
ls .github/dependabot.yml renovate.json 2>/dev/null

# CI audit workflow?
ls .github/workflows/*.yml 2>/dev/null | xargs grep -l -E "(audit|snyk|gitleaks)" 2>/dev/null

# Lockfile?
ls pnpm-lock.yaml yarn.lock package-lock.json 2>/dev/null

# Third-party actions pinned to SHA?
grep -rE "uses: [^a-zA-Z].*@v[0-9]" .github/workflows/ 2>/dev/null

# Pending CVEs?
pnpm audit --audit-level=moderate 2>/dev/null || npm audit --audit-level=moderate 2>/dev/null
```

## Step 1d: Internal-link canonicalization audit (catches trailing-slash drift)

Single biggest cause of crawl-budget waste on otherwise-healthy static sites. Framework config declares the canonical URL form (`trailingSlash: 'always'` in Astro, `trailingSlash: true` in Next.js, etc.), then internal links drift over time as developers paste paths without trailing slashes. Sitemap and canonical tags stay correct, so it's invisible in standard probes — but every header/footer/body link triggers a 308, and Googlebot stops at the redirect during initial discovery.

```bash
# 1. Detect the project's canonical URL form
grep -E "trailingSlash|cleanUrls|uglyURLs" astro.config.* next.config.* hugo.* svelte.config.* 2>/dev/null

# 2. Find slashless internal hrefs in source (Astro/Next/Vite layout)
grep -rnE 'href="/[^"#]*[^/"]"' src/ public/ \
  | grep -vE '\.(svg|png|jpg|jpeg|webp|avif|gif|ico|css|js|json|xml|txt|pdf|woff2?|zip|dmg|exe|rss|atom)"' \
  | grep -vE 'href="(http|mailto:|tel:|#)'

# 3. Same for committed redirect target lists
grep -nE "'/[^']*[^/']'" astro.config.* next.config.* 2>/dev/null  # framework redirects map
cat public/_redirects 2>/dev/null                                   # CF Pages style
cat public/llms.txt | grep -oE 'https?://[^ )"]*'                   # llms.txt entries (reach Google as referral)

# 4. Live redirect-chain probe — every internal href should be 0-hop
for url in /docs /download /compare /changelog /community; do
  printf "%-30s " "$url"
  curl -sI -L --max-time 10 -w 'hops=%{num_redirects} final=%{url_effective}\n' \
    -o /dev/null --resolve DOMAIN:443:$IP "https://DOMAIN$url"
done
# Expect: hops=0 for every URL. hops=1 means slashless internal links; hops≥2 means redirect chain.
```

When you find drift, the fix surface is small but global: header/footer components, body content templates, `redirects` config, and `llms.txt`. One commit canonicalizes them all.

The reverse case (framework set to `trailingSlash: 'never'` but internal links carry slashes) produces the same crawl waste — the audit asks "do internal links match the framework setting", not "do they have a slash".

## Step 2: External validator suite

Ask user to run (can't be automated without browser):

- **Rich Results Test** https://search.google.com/test/rich-results → paste URL, confirm WebSite + SoftwareApplication + Organization (or whatever schemas expected) detected
- **Schema Validator** https://validator.schema.org/
- **OG Preview** https://www.opengraph.xyz/
- **Is It Agent Ready** https://isitagentready.com/
- **Lighthouse** (run yourself via `npx lighthouse` — see `11-validation-toolkit.md`)

## Step 3: Severity gap report

Build the report in this format (use the SKILL.md severity rubric):

```
## Gap report — DOMAIN

### 🔴 Critical (ship same day)
- <issue> — <why critical> — <reference §>
- …

### 🟡 Recommended (ship this week)
- <issue> — <why recommended> — <reference §>
- …

### 🟢 Nice (ship when possible)
- <issue> — <benefit> — <reference §>
- …
```

Present to user, get approval per severity band or per fix group.

## Step 4: Patch plan

Group fixes by file to minimize churn:

```
Fix group A: robots.txt
  - Add Content-Signal directive
  - Add missing AI crawler blocks
  - Reference sitemap.xml URL

Fix group B: <head> (all routes or BaseLayout)
  - Add Organization schema JSON-LD
  - Add apple-touch-icon, mask-icon
  - Add og:locale, og:image:width/height

Fix group C: new files
  - /llms.txt
  - /humans.txt
  - /.well-known/security.txt
  - /.well-known/http-message-signatures-directory (JWKS placeholder)
```

Get approval per group before writing. Apply changes. Commit logically.

## Step 5: Re-validate

Repeat Step 1, Step 1b on patched paths. Confirm:
- Previously-missing files now return 200
- New meta tags show up in head harvest
- Rich Results Test re-detects expected schemas
- `isitagentready.com` score improves

## Step 6: Post-audit handoff

Give user:
1. **Summary** — what was fixed, what remains, deferred items
2. **Manual action list** — zone settings they need to change (SSL mode, Bot Fight, etc.)
3. **Indexing acceleration** — if meta or schema changed, suggest GSC "Request Indexing" (see `12-indexing.md`)
4. **Expected propagation** — logo-in-SERP 1-4 weeks, rich snippets 1-3 weeks, full reindex after major meta changes 2-8 weeks

## Common Mode B findings

Across every audit I've seen:

- ❌ Organization schema missing → logo not in SERP Knowledge Panel
- ❌ `Content-Signal` missing → Cloudflare scanner fails, GEO weaker
- ❌ `Link:` headers missing → agentic-web probe fails
- ❌ JWKS placeholder missing → returns HTML fallback at `.well-known/http-message-signatures-directory`
- ⚠ Multiple og:image sizes not declared — Meta debugger shows "image too small" warning
- ⚠ `apple-touch-icon` + `mask-icon` missing — iOS home screen generic icon
- ⚠ Repo missing Dependabot.yml — passive CVE exposure
- ⚠ GitHub Actions unpinned to tag — supply-chain risk
- ⚠ Trailing-slash drift — header/footer/body/redirects/llms.txt link to slashless paths while framework forces trailing slash → every internal click is a 308. Surfaces in GSC as "Page with redirect", "Redirect error", or chronically slow indexing of sub-pages. Run §1d.
