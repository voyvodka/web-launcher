# Mode B — Existing site audit workflow

*Loaded by `web-launcher` SKILL for Mode B (audit & fix of an existing deployed site).*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

When user points at a deployed site and asks for improvements, run this sequence.

## How this file relates to `14-diagnostic-checks.md`

**This file is the workflow; `14-diagnostic-checks.md` is the instrument set.** They must not be
read as alternatives, and where they overlap, 14 wins.

| | `09` (this file) | `14-diagnostic-checks.md` |
|---|---|---|
| Answers | *What order do I do things in, and how do I report it?* | *Is this specific behaviour actually true on the live site?* |
| Shape | Six ordered steps, one gap report, one handoff | Ten independent checks (C1–C10), each printing `OK` or `FAIL` |
| Use when | Starting an audit | A probe here looks wrong and you need the cause |

The probes in Step 1 and Step 1d below are **smoke tests** — deliberately shallow, fast enough to
run on every deploy. They tell you *something is wrong*. They do not tell you *why*. The moment one
comes back unexpected, stop and run the matching check from 14:

| Symptom from this file | Run this check from `14-diagnostic-checks.md` |
|---|---|
| A redirect resolves, but to an error | **C1** — redirect targets must return 200 |
| Live `robots.txt` disagrees with the repo | **C2** — edge-injection hunter |
| apex/www redirect is `302`/`307`, not `301`/`308` | **C3** — consolidation must be permanent |
| `hops` ≥ 1 in Step 1d | **C4** — hop count *including meta-refresh*, which `curl -L` misses |
| Sitemap URL not 200 | **C5** |
| Canonical mismatch | **C6** |
| A thin `/index.html` shell served alongside `/` | **C7** |
| `Link:` / HSTS header missing in Step 1b | **C8** — declared headers must actually be served |
| `og:image` blank or SVG | **C9** |

Nothing in Step 1d duplicates C4 by accident: Step 1d greps **source** for slashless hrefs, C4
measures **live** hop count and catches the meta-refresh hop `curl -L` does not count. Run both —
source drift and live chains are different defects with the same symptom.

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

## Step 1b: Agent-ready probe (agentic-web signals)

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
*(checked 2026-08-14 — live, and confirmed to be operated by Cloudflare)*. Its remediation tips are
LLM-generated and the page says so; use them as leads and verify each against `06-agent-ready.md`
before acting.

A `grep` above that prints nothing exits non-zero — that is the finding (header or directive
absent), not a broken pipeline. All four probes were executed as written on 2026-08-14.

## Step 1c: Repo audit (if user has local checkout)

See `13-dependency-security.md` §13.11 ("Mode B rapid checklist") for the full list — the old
pointer to "§11.11" was a typo for a section that does not exist *(corrected 2026-08-14)*. Quick
version:

```bash
# Dep-bot present?
ls .github/dependabot.yml renovate.json 2>/dev/null

# CI audit workflow?
ls .github/workflows/*.yml 2>/dev/null | xargs grep -l -E "(audit|snyk|gitleaks)" 2>/dev/null

# Which package manager? package.json's packageManager field wins, lockfile is the fallback.
grep -o '"packageManager"[^,]*' package.json 2>/dev/null
ls pnpm-lock.yaml yarn.lock package-lock.json bun.lock bun.lockb 2>/dev/null

# Third-party actions pinned to SHA?
grep -rE "uses: [^a-zA-Z].*@v[0-9]" .github/workflows/ 2>/dev/null

# Pending CVEs? Run the ONE that matches the project.
# Not `pnpm audit || npm audit` — audit tools exit non-zero when they FIND something, so the
# fallback fires on a real vulnerability and runs the second scanner against the wrong lockfile.
if   [ -f pnpm-lock.yaml ]; then pnpm audit --audit-level=moderate
elif [ -f yarn.lock ];      then yarn npm audit --severity moderate
elif [ -f bun.lock ] || [ -f bun.lockb ]; then bun audit
else                             npm audit --audit-level=moderate
fi
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

- **Rich Results Test** https://search.google.com/test/rich-results *(checked 2026-08-14)* → paste URL, confirm WebSite + SoftwareApplication + Organization (or whatever schemas expected) detected
- **Schema Validator** https://validator.schema.org/ *(checked 2026-08-14)*
- **OG Preview** https://www.opengraph.xyz/ *(checked 2026-08-14)*
- **PageSpeed Insights** https://pagespeed.web.dev/ *(checked 2026-08-14)* → the only one-stop view of field (CrUX) and lab data together
- **Is It Agent Ready** https://isitagentready.com/ *(checked 2026-08-14)*

Run yourself, no browser needed:

- **Lighthouse** — `npx lighthouse@13.4.1`, desktop *and* mobile. Mobile is the default form
  factor: pass **no** preset for it. See `11-validation-toolkit.md` §11.3 for the full invocation
  and for the Agentic Browsing category, which is in the standard config from Lighthouse 13.3.0.

Two tools that used to sit in this list have been **removed** — do not ask the user to run them:
Google's **Mobile-Friendly Test** is retired (its URL now redirects to the Lighthouse docs), and
the **X/Twitter Card Validator** is login-gated and unmaintained with no official replacement. Both
checked 2026-08-14; substitutes are in `11-validation-toolkit.md` §11.2.

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

Get approval per group before writing. Apply changes.

**Approval to apply is not approval to commit, deploy, or publish.** Ask separately, and name the
operation, before any of: `git commit` or `git push`, a `wrangler deploy`, a cache purge, a sitemap
submission, or a DNS change. These leave the working tree and are not all reversible — a purge and a
submitted sitemap cannot be taken back at all. Offer the diff and let the user run the commit
themselves if they prefer; that is often the right answer for a repository this skill does not own.

## Step 5: Re-validate

Repeat Step 1, Step 1b on patched paths. Confirm:
- Previously-missing files now return 200
- New meta tags show up in head harvest
- Rich Results Test re-detects expected schemas
- `isitagentready.com` score improves

Then re-run every check from `14-diagnostic-checks.md` that failed in the gap report, and **paste
the before and after output for each**. A fix is closed when the same command that printed `FAIL`
prints `OK` — a described fix is not evidence of one.

## Step 6: Post-audit handoff

Give user:
1. **Summary** — what was fixed, what remains, deferred items
2. **Manual action list** — zone settings they need to change (SSL mode, Bot Fight, etc.)
3. **Indexing acceleration** — if meta or schema changed, suggest GSC "Request Indexing" (see `12-indexing.md`)
4. **Expected propagation** — ⚠️ the ranges once quoted here (logo-in-SERP 1–4 weeks, rich snippets
   1–3 weeks, full reindex 2–8 weeks) have **no source and were not verifiable on 2026-08-14**.
   Google publishes no such SLA and recrawl intervals vary by site. Tell the user "days to weeks,
   and it depends on how often this site is crawled" and point them at the Search Console
   coverage report for the actual signal — never quote a number you cannot source.

## Common Mode B findings

Recurring patterns worth probing early. This is a checklist drawn from prior audits, not a ranked
frequency table — no measurement backs an ordering, so do not present it to a user as one.

- ❌ Organization schema missing → logo not in SERP Knowledge Panel
- ❌ `Content-Signal` missing → agent-readiness scanner fails, GEO weaker
- ❌ `Link:` headers missing → agentic-web probe fails. Confirm with **C8**, which distinguishes
  "never declared" from "declared in `_headers` but not served"
- ❌ JWKS placeholder missing → returns HTML fallback at `.well-known/http-message-signatures-directory`
- ⚠ `og:image` absent, SVG, or 404 → blank social card. **C9** checks every sitemap URL at once
- ⚠ `apple-touch-icon` + `mask-icon` missing — iOS home screen generic icon
- ⚠ Repo missing Dependabot/Renovate config — passive CVE exposure
- ⚠ **GitHub Actions pinned to a mutable tag** (`uses: owner/action@v4`) rather than a full commit
  SHA — a tag can be repointed at new code, so tag-pinning is the supply-chain risk. This is what
  the Step 1c grep is looking for; a hit means "pin to SHA", not "add a tag"
- ⚠ Trailing-slash drift — header/footer/body/redirects/llms.txt link to slashless paths while framework forces trailing slash → every internal click is a 308. Surfaces in GSC as "Page with redirect", "Redirect error", or chronically slow indexing of sub-pages. Run §1d, then **C4** for the live hop count.
