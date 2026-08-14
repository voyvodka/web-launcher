# Cloudflare Workers deployment (wrangler + worker script + zone settings)

*Loaded by `web-launcher` SKILL for Mode A deploy phase, or Mode B when changing infrastructure.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

**Avoid the dashboard "Upload and deploy" flow** — it sometimes creates workers without the Assets binding (Bindings=0 → "There is nothing here yet" placeholder). Wrangler CLI is deterministic.

## Baseline `wrangler.jsonc`

```jsonc
{
  "$schema": "https://unpkg.com/wrangler/config-schema.json",
  "name": "PROJECT",
  "compatibility_date": "YYYY-MM-DD",
  "assets": {
    "directory": "./deploy",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application"
  },
  "routes": [
    { "pattern": "DOMAIN", "custom_domain": true },
    { "pattern": "www.DOMAIN", "custom_domain": true }
  ]
}
```

## www → apex 301 redirect (Worker script)

Zone-level Redirect Rules require token scope most OAuth sessions don't have. Use a Worker script instead.

Add `"main": "./src/worker.js"` to `wrangler.jsonc`, then create `src/worker.js`:

```js
/**
 * Site worker
 *   - www.DOMAIN → 301 apex redirect (SEO canonical consolidation)
 *   - Everything else → static assets (with SPA fallback from wrangler config)
 */
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.hostname === 'www.DOMAIN') {
      url.hostname = 'DOMAIN';
      return Response.redirect(url.toString(), 301);
    }
    return env.ASSETS.fetch(request);
  },
};
```

## `_headers` — security + caching

Place in `deploy/_headers`:

```
/*
  Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: interest-cohort=()
  X-Frame-Options: DENY
  Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; script-src 'self' 'unsafe-inline'; img-src 'self' data:; base-uri 'self'; frame-ancestors 'none'; form-action 'self'
  Link: </llms.txt>; rel="alternate"; type="text/markdown", </sitemap.xml>; rel="sitemap", </.well-known/security.txt>; rel="security-txt"

/og-cover.png
  Cache-Control: public, max-age=31536000, immutable

/favicon.svg
  Cache-Control: public, max-age=604800
```

CSP notes:
- `'unsafe-inline'` in script-src allows inline JSON-LD. Remove if you hash JSON-LD with SRI.
- Google Fonts requires explicit `fonts.googleapis.com` (style) + `fonts.gstatic.com` (font) — adjust if using a different font host.

## Deploy sequence

```bash
# 1. Login once (interactive, browser OAuth)
npx wrangler@latest login

# 2. Verify
npx wrangler@latest whoami

# 3. Deploy (uploads assets + script + registers custom domains)
npx wrangler@latest deploy
```

**Expected output**:
```
✨ Read N files from the assets directory /…/deploy
✨ Success! Uploaded X files
Uploaded PROJECT (~10s)
Deployed PROJECT triggers (~8s)
  DOMAIN (custom domain)
  www.DOMAIN (custom domain)
```

Custom domains auto-register DNS + fetch Universal SSL cert (1-2 min to propagate globally).

## Zone-level settings (CF dashboard — OAuth scope insufficient)

Give user this checklist for `dash.cloudflare.com → DOMAIN`:

| Path | Setting | Value | Why |
|---|---|---|---|
| SSL/TLS → Overview | Encryption mode | **Full (strict)** | Default `Flexible` is MITM-open |
| SSL/TLS → Edge Certificates | Always Use HTTPS | **On** | HTTP→HTTPS 301 |
| SSL/TLS → Edge Certificates | Automatic HTTPS Rewrites | **On** | Mixed-content protection |
| Security → Bots | Bot Fight Mode | **Off** | Blocks legit AI crawlers (conflicts with robots.txt allowlist) |
| Caching → Configuration | Purge Everything | (after redirect changes) | Clear stale 200s before retesting worker |

## Gotchas

- **`_redirects /*  /  302` on root creates infinite loop** — matches `/` too. Use SPA fallback via `not_found_handling` in wrangler config instead.
- **Finder drag-upload hides dotfiles** (`.well-known/` dropped silently) — always `wrangler deploy` CLI for sites with hidden folders.
- **Dashboard "Upload and deploy"** can leave Bindings=0 → "There is nothing here yet" placeholder. Wrangler CLI creates the Assets binding correctly every time.
- **OAuth token lacks zone scope** — can't create Redirect Rules, can't change SSL mode, can't purge cache. All zone-level ops are user-via-dashboard.
- **Edge cache holds pre-redeploy 200s** — use fresh random paths to test post-redeploy, or ask user to Purge Everything.
- **Workers.dev URL auto-disabled** when custom domain added (by default). Add `"workers_dev": true` to wrangler.jsonc if you want both.

## Verification after deploy

```bash
IP=188.114.97.3  # any CF anycast
for p in / /favicon.svg /robots.txt /sitemap.xml /.well-known/security.txt /og-cover.png; do
  echo -n "$p  ->  "
  curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN$p" | head -1
done

# www → apex redirect
curl -sI --resolve www.DOMAIN:443:$IP "https://www.DOMAIN/test-$(date +%s)" | head -3
# Expect: 301, Location: https://DOMAIN/test-…
```

All 200 except www (301) and random paths (200 via SPA fallback). See `11-validation-toolkit.md` for full probe suite.
