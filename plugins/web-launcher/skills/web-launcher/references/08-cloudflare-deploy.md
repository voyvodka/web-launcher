# Cloudflare Workers deployment (wrangler + static assets + zone settings)

*Loaded by `web-launcher` SKILL for Mode A deploy phase, or Mode B when changing infrastructure.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

## Platform choice: Workers Static Assets

Deploy static sites as a Worker with an `assets` directory. Cloudflare documents `_headers` and
`_redirects` as natively supported here, and publishes a Pages → Workers migration guide stating
"Unlike Pages, Workers has a distinctly broader set of features available to it"
(verified 2026-08-14 — [Migrate from Pages to Workers](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/), page last updated 2026-07-28).

⚠️ Cloudflare's docs do **not** say Pages is deprecated or closed to new projects — the Pages docs
are still maintained (last updated 2026-04-21) and carry no such banner. Do not tell users Pages is
going away; say only that Workers is where static-asset features are documented and that a
migration guide exists. (The docs *do* say "Do not use Workers Sites for new projects" — that is
the older `[site]` product, not Pages.)

## Baseline `wrangler.jsonc`

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "PROJECT",
  "compatibility_date": "YYYY-MM-DD",
  "assets": {
    "directory": "./deploy",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application"
  },
  "routes": [
    { "pattern": "DOMAIN", "custom_domain": true }
  ]
}
```

Field notes (verified 2026-08-14 — [Wrangler configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)):

| Field | Today's behaviour |
|---|---|
| `$schema` | Docs use the local path `./node_modules/wrangler/config-schema.json`. `https://unpkg.com/wrangler/config-schema.json` still resolves (302 → 200, checked 2026-08-14) but the local path is what the docs show |
| `compatibility_date` | Required, `yyyy-mm-dd`. Set to the day you scaffold |
| `assets.directory` | Folder of static assets to serve |
| `assets.binding` | Only needed if a Worker script calls `env.ASSETS.fetch()`. A purely static site needs no `main` and no binding |
| `assets.not_found_handling` | `"single-page-application"` \| `"404-page"` \| `"none"` — **default `"none"`** |
| `assets.html_handling` | `"auto-trailing-slash"` (default) \| `"force-trailing-slash"` \| `"drop-trailing-slash"` \| `"none"` |
| `assets.run_worker_first` | `boolean` or an array of route patterns (negation with `!` supported). Array form needs Wrangler ≥ 4.20.0 |
| `routes[].custom_domain` | `true` → Cloudflare creates the DNS records and issues certificates for you |
| `workers_dev` | Defaults to `true`, but see the gotcha below — adding `routes` flips it to `false` |

## Routing order — read this before adding a Worker script

> "By default, if a requested URL matches a file in the static assets directory, that file will be
> served — without invoking Worker code. If no matching asset is found and a Worker script is
> present, the request will be processed by the Worker."
> (verified 2026-08-14 — [Static Assets](https://developers.cloudflare.com/workers/static-assets/))

Consequence: **a Worker script does not see requests that match an asset.** Any Worker logic meant
to run on every request (host redirects, header injection, auth) silently does nothing on `/`,
`/about/`, `/favicon.svg` and every other real file. It only fires on misses — which is exactly why
a random-path smoke test passes while the homepage does not.

To run the Worker ahead of asset serving, set `assets.run_worker_first: true`, or an array of
patterns for selective control
(verified 2026-08-14 — [Routing / Worker script](https://developers.cloudflare.com/workers/static-assets/routing/worker-script/)).

A second mechanism moves the same line and is easy to miss. The compatibility flag
**`assets_navigation_prefers_asset_serving`**, default since **2025-04-01**, makes navigation
requests (`Sec-Fetch-Mode: navigate`) prefer asset serving *even when no asset matches exactly* —
so SPA and custom-404 fallbacks are served ahead of the Worker. Without it the old behaviour
applies and a miss invokes the Worker. **`run_worker_first: true` overrides the flag entirely**
(verified 2026-08-14 — [Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Consequence when debugging "my Worker does not run": check the compatibility date as well as
`run_worker_first`. A project scaffolded after April 2025 inherits the flag silently.

> **Verify before you rely on a key here.** This is the fastest-moving reference in this skill. If
> this session can reach Cloudflare's documentation directly — a Cloudflare docs MCP server, or any
> documentation tool — query it for the specific field before recommending it, and cite what came
> back. It is a search, not an oracle: a query that returns nothing relevant leaves the claim
> unverified, it does not confirm it. Nothing here depends on that tool being present.

## www → apex redirect

**Recommended: a zone-level redirect rule, not a Worker.** Cloudflare's own Custom Domains page
documents this:

> "Because Custom Domains require an exact hostname match, a Worker attached to `example.com` will
> not receive requests sent to `www.example.com`, and vice versa. To make both versions of your
> domain work, set up a redirect rule … You also need a proxied DNS record for the hostname you are
> redirecting from … Add a proxied DNS A record for `www` pointing to `192.0.2.0`, or a proxied AAAA
> record pointing to `100::`"
> (verified 2026-08-14 — [Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/), page last updated 2026-06-23)

`192.0.2.0` / `100::` are documented reserved placeholders for originless setups; the record is
proxied, so the request never leaves Cloudflare. Steps for the user, in the dashboard:
`DOMAIN → DNS → Records` (add proxied `www` A record `192.0.2.0`), then
`DOMAIN → Rules → Redirect Rules` (source `www.DOMAIN`, target apex, 301, preserve path + query).
The Pages equivalent guide uses Bulk Redirects with the same DNS trick
(verified 2026-08-14 — [Redirecting www to domain apex](https://developers.cloudflare.com/pages/how-to/www-redirect/)).

**Worker variant** — only if the user cannot or will not touch zone rules. It requires *both*
`www.DOMAIN` as a custom domain *and* `run_worker_first`, otherwise it never fires (see routing
order above):

```jsonc
{
  "main": "./src/worker.js",
  "assets": {
    "directory": "./deploy",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "run_worker_first": true
  },
  "routes": [
    { "pattern": "DOMAIN", "custom_domain": true },
    { "pattern": "www.DOMAIN", "custom_domain": true }
  ]
}
```

```js
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

**Cost of this variant:** the docs warn that with `run_worker_first` configured "you will likely need
to attach any custom headers you wish to apply directly within that Worker script", because
`_headers` rules "are not applied to responses generated by your Worker code"
(verified 2026-08-14 — [Headers](https://developers.cloudflare.com/workers/static-assets/headers/)).
⚠️ Whether a response returned straight from `env.ASSETS.fetch()` still picks up `_headers` is not
stated either way in the docs — untested here. If you take this route, verify the security headers
are present on a real page (`curl -sI https://DOMAIN/`) before calling the deploy done, and be ready
to set them in Worker code instead. The redirect-rule approach avoids the question entirely.

## `_headers` — security + caching

Place in `deploy/_headers` (the asset directory root). Limits: 100 rules per file, 2,000 characters
per line (verified 2026-08-14 — [Headers](https://developers.cloudflare.com/workers/static-assets/headers/)).

```
/*
  Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: browsing-topics=()
  X-Frame-Options: DENY
  Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; script-src 'self' 'unsafe-inline'; img-src 'self' data:; base-uri 'self'; frame-ancestors 'none'; form-action 'self'
  Link: </llms.txt>; rel="alternate"; type="text/markdown", </sitemap.xml>; rel="sitemap", </.well-known/security.txt>; rel="security-txt"

/og-cover.png
  Cache-Control: public, max-age=31536000, immutable

/favicon.svg
  Cache-Control: public, max-age=604800
```

Notes:
- `Permissions-Policy: interest-cohort=()` was the FLoC opt-out and is **no longer a listed
  directive**; the Topics API opt-out is `browsing-topics=()`
  (verified 2026-08-14 — [MDN Permissions-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy)).
  Sites still shipping `interest-cohort=()` are sending a header nothing reads.
- `'unsafe-inline'` in `script-src` allows inline JSON-LD. Note that if Bot Fight Mode is ever
  turned on, Cloudflare injects challenge scripts and the docs "highly discourage the use of
  `unsafe-inline`", recommending CSP nonces instead (see zone settings below).
- Google Fonts requires explicit `fonts.googleapis.com` (style) + `fonts.gstatic.com` (font).
- To keep the free `workers.dev` URL out of search results, the docs give this rule
  (verified 2026-08-14 — same page):
  ```
  https://:version.:subdomain.workers.dev/*
    X-Robots-Tag: noindex
  ```

Default headers Workers already attaches to static assets, so do not re-declare them:
`Content-Type`, `ETag`, `CF-Cache-Status`, and
`Cache-Control: public, max-age=0, must-revalidate` — which the docs describe as ensuring "stale
content will never be served" (verified 2026-08-14 — same page).

## `_redirects`

Same directory, one rule per line, `[source] [destination] [code?]`, code defaults to 302. Limits:
2,000 static + 100 dynamic redirects (2,100 total), 1,000 characters per declaration. Order matters
— the top-most match wins, and static rules must precede dynamic ones. Redirects "are not applied to
requests served by your Worker code"
(verified 2026-08-14 — [Redirects](https://developers.cloudflare.com/workers/static-assets/redirects/)).
Past 2,100 rules, the docs point to Bulk Redirects.

## Deploy sequence

Latest wrangler at time of writing: **4.123.0**, published 2026-08-13
(verified 2026-08-14 — `curl -s https://registry.npmjs.org/wrangler | jq -r '."dist-tags".latest'`).

```bash
# 1. Login once (interactive, browser OAuth)
npx wrangler@latest login          # add --use-keyring to store credentials in the OS keychain
                                   # instead of a plaintext TOML file

# 2. Verify
npx wrangler@latest whoami

# 3. Deploy (uploads assets + script + registers custom domains)
npx wrangler@latest deploy
```

`login`, `whoami` and `deploy` are all current; note that the Wrangler command reference was split —
`login`/`whoami` now live under
[general commands](https://developers.cloudflare.com/workers/wrangler/commands/general/) and `deploy`
under [Workers commands](https://developers.cloudflare.com/workers/wrangler/commands/workers/)
(verified 2026-08-14).

Expected output names the uploaded file count and the registered triggers. ⚠️ Do not pattern-match
the exact wording — Wrangler ships weekly (12 releases between 2026-07-21 and 2026-08-13) and the
output format is not a documented contract. Assert on `curl` results instead.

Custom domains create the DNS records and issue certificates automatically; the docs note this
generates an **Advanced Certificate** on the zone, and that deleting the custom domain does *not*
delete that certificate — it must be removed by hand under SSL/TLS → Edge Certificates
(verified 2026-08-14 — [Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)).

## Zone-level settings (CF dashboard — OAuth token is read-only on zones)

Give the user this checklist for `dash.cloudflare.com → DOMAIN`:

| Path | Setting | Value | Why |
|---|---|---|---|
| SSL/TLS → Overview | Encryption mode | **Full (strict)** | `Flexible` leaves the origin leg unencrypted |
| SSL/TLS → Edge Certificates | Always Use HTTPS | **On** | HTTP→HTTPS redirect |
| SSL/TLS → Edge Certificates | Automatic HTTPS Rewrites | **On** | Mixed-content protection |
| Security → Settings → Bot traffic | Bot Fight Mode | **Off** | See below |
| Rules → Redirect Rules | www → apex | 301, preserve path/query | The documented www redirect method |
| Caching → Configuration | Purge Everything | (only if you set long `max-age` yourself) | Defaults already revalidate |

Bot Fight Mode, with sources (verified 2026-08-14 — [Bot Fight Mode](https://developers.cloudflare.com/bots/get-started/bot-fight-mode/), page last updated 2026-08-03):
- It "cannot be customized, adjusted, or reconfigured via WAF custom rules" and cannot be skipped —
  Skip/Bypass/Allow actions have no effect on it.
- It **auto-enables JavaScript Detections, which cannot be disabled**, and that injected script
  fights a strict CSP: you must allow `/cdn-cgi/challenge-platform/` and use CSP nonces, or accept
  `unsafe-inline`.
- It issues CPU-intensive challenges to traffic "matching patterns of known bots" and "may challenge
  API or mobile app traffic".

⚠️ The older claim that Bot Fight Mode "blocks legitimate AI crawlers" is not something Cloudflare's
docs state — removed rather than repeated. The CSP conflict above is the sourced reason to leave it
off for a static marketing site. For deliberate AI-crawler policy, that is a separate documented
feature ([Block AI bots](https://developers.cloudflare.com/bots/additional-configurations/block-ai-bots/)).

## Gotchas

- **A Worker script does not run for requests that match a static asset.** The single most expensive
  mistake in this file's history: host redirects and header logic written in a Worker appear to work
  (random paths 301 correctly) while every real page bypasses them. Set `run_worker_first`, or move
  the logic to a zone rule. (verified 2026-08-14 — [Routing / Worker script](https://developers.cloudflare.com/workers/static-assets/routing/worker-script/))
- **`workers_dev` flips to `false` when you add `routes`** — "If you do not specify
  `workers_dev = false` but add a routes component … the value of `workers_dev` will be inferred as
  `false` on the next deploy." Set `"workers_dev": true` explicitly to keep both. Preview URLs
  follow the same setting unless configured separately.
  (verified 2026-08-14 — [workers.dev](https://developers.cloudflare.com/workers/configuration/routing/workers-dev/))
- **`_headers` and `_redirects` do not apply to Worker-generated responses**, even when the request
  URL matches a rule. Any site with a Worker script must verify its security headers on a live URL.
  (verified 2026-08-14 — headers/redirects pages above)
- **`_redirects` with `/*  /  302` matches `/` itself.** The docs document no loop protection; they
  only guarantee that the top-most matching rule wins and that "redirects are always followed". Use
  `not_found_handling: "single-page-application"` for SPA fallback rather than a catch-all redirect.
  (verified 2026-08-14 — [Redirects](https://developers.cloudflare.com/workers/static-assets/redirects/))
- **OAuth token carries `zone:read` only.** `npx wrangler@4.123.0 login --scopes-list` (run
  2026-08-14) lists exactly one zone scope, read-level. So the CLI cannot create redirect rules,
  change SSL mode, or purge cache — every zone-level operation is user-via-dashboard. Confirm with
  that command rather than trusting this line.
- **Finder drag-upload hides dotfiles** — `.well-known/` disappears silently. ⚠️ macOS behaviour,
  not a Cloudflare claim, and not verifiable from a vendor doc; treated as an operational habit:
  always deploy via `wrangler deploy`.
- **Removed: "stale 200s survive redeploy, purge everything before retesting."** Static assets are
  served with `Cache-Control: public, max-age=0, must-revalidate` plus an `ETag` by default, which
  the docs say guarantees "stale content will never be served". Purging is only relevant for paths
  where *you* set a long `max-age`/`immutable` in `_headers` — and browser-side, no purge reaches
  those. Test with a cache-busting query string instead.
- **Removed: "dashboard Upload and deploy leaves Bindings=0."** ⚠️ This was an undated field
  observation with no vendor documentation behind it, and the dashboard upload flow has changed
  since. The CLI-first advice stands on its own merits (reproducible, dotfile-safe, config in git) —
  do not attribute it to a bug that cannot be shown to exist today.

## Verification after deploy

```bash
# Resolve through Cloudflare's current anycast IPs rather than a hardcoded address
IP=$(dig +short DOMAIN | head -1)

for p in / /favicon.svg /robots.txt /sitemap.xml /.well-known/security.txt /og-cover.png; do
  printf '%s -> ' "$p"
  curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN$p" | head -1
done

# Security headers actually present on a real page (not just on a 404 miss)
curl -sI --resolve DOMAIN:443:$IP "https://DOMAIN/" | grep -iE 'strict-transport|content-security|x-frame|referrer-policy'

# www -> apex, tested on BOTH a real page and a random path
curl -sI --resolve www.DOMAIN:443:$IP "https://www.DOMAIN/" | head -3
curl -sI --resolve www.DOMAIN:443:$IP "https://www.DOMAIN/test-$(date +%s)" | head -3
# Expect 301 + Location: https://DOMAIN/... on BOTH. If only the random path redirects,
# the Worker is running behind asset serving — see the routing-order section.
```

All 200 except www (301) and random paths (200 via SPA fallback). See `11-validation-toolkit.md` for
the full probe suite.
