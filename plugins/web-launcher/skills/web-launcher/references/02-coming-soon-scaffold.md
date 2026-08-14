# Coming-soon / landing page scaffolding

*Loaded by `web-launcher` SKILL when user wants a single-page coming-soon / landing HTML in Mode A.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

## Structure

Single-file HTML, everything inlined. Zero build step, zero external CSS, single Google Fonts `<link>`.

```
coming-soon.html           ← authoring file (lives in project root)
```

Deploy folder after build (or direct upload):
```
deploy/
├── index.html             ← renamed coming-soon.html
├── favicon.svg
├── og-cover.png           ← generated via satori (see 07-og-satori.md)
├── robots.txt
├── sitemap.xml
├── llms.txt
├── humans.txt
├── _headers               ← CF security + cache
└── .well-known/
    └── security.txt
```

## HTML skeleton

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
<title>PROJECT — Launching soon</title>
<meta name="description" content="…tagline + what it does + open-source/launch note…" />
<meta name="robots" content="index,follow" />
<meta name="theme-color" content="#0a0b0e" />
<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
<link rel="apple-touch-icon" href="/favicon.svg" />
<link rel="mask-icon" href="/favicon.svg" color="#ACCENT" />
<!-- full meta + OG + twitter + JSON-LD: see 03-discoverability-classic.md -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=PRIMARY_FONT:wght@400;500;600&display=swap" rel="stylesheet">
<style>
  /* Inline CSS — everything the page needs */
  @media (prefers-reduced-motion: reduce) {
    .pulse, .animated { animation: none; }
  }
</style>
</head>
<body>
  <main>
    <p class="brand-anchor">
      <svg viewBox="0 0 64 64" width="20" height="20" aria-hidden="true">
        <path d="…LOGOMARK PATH…" fill="#ACCENT"></path>
      </svg>
      <span>PROJECT</span>
    </p>
    <p class="eyebrow">Launching soon</p>
    <h1>PROJECT is <em>almost here.</em></h1>
    <p class="lede">Tagline sentence. Open source. What it does.</p>
    <div class="actions">
      <a class="btn primary" href="GITHUB_URL">View on GitHub</a>
      <a class="btn ghost" href="RELEASES_URL">Latest release</a>
    </div>
    <p class="meta"><span>macOS</span><span>·</span><span>Windows</span><span>·</span><span>Linux</span></p>
    <footer>© YEAR maintainer · <a href="GITHUB_URL">Source</a></footer>
  </main>
</body>
</html>
```

## Must-haves

- **Single `<h1>`** — states the launching-soon fact
- **`<main>` landmark** wrapping content, `<footer>` at bottom
- **`brand-anchor`** at top-left (mark + wordmark) — identity anchor
- **Platform / stack row** — builds credibility fast
- **CTAs** to GitHub + latest release (leverage existing launch funnel)
- **`prefers-reduced-motion`** override for any pulse / animation
- **JSON-LD structured data** in `<head>`:
  - `WebSite`
  - `SoftwareApplication` (if a product launch)
  - `Organization` with `logo` absolute URL
  - See `03-discoverability-classic.md` for exact template

## Layout gotcha

If both `brand-anchor` and eyebrow are `<p>` with `display: inline-flex`, they flow onto the same line. Fix: `.brand-anchor { display: flex; width: fit-content; }` (block-level flex). Confirmed trip-up.

## Typography rules (default)

- `body` font: IBM Plex Sans 500 (or brand primary)
- Eyebrow / meta / monospace: IBM Plex Mono 400/500
- Display / hero italic accent: Instrument Serif italic (sparingly — in H1 `<em>` only)
- Colors: body `#e7ecf2`, background `#0a0b0e`, accent `#ffb020` (exact, no tints)

## Deploy checklist

Before shipping to CF Workers:
1. Copy `coming-soon.html` → `deploy/index.html`
2. Generate `deploy/og-cover.png` via satori (see `07-og-satori.md`)
3. Write `deploy/robots.txt`, `deploy/sitemap.xml`, `deploy/llms.txt`, `deploy/humans.txt`, `deploy/_headers`, `deploy/.well-known/security.txt` (see `03-discoverability-classic.md` + `06-agent-ready.md`)
4. `wrangler deploy` (see `08-cloudflare-deploy.md`)
