# OG image generation (satori + resvg-js)

*Loaded by `web-launcher` SKILL when generating or updating 1200×630 social cards.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

Ship a raster image (PNG or JPEG). Meta documents a **1200×630** recommendation, a 200×200 minimum,
a 1.91:1 target aspect ratio and an 8 MB file-size ceiling for `og:image`
(verified 2026-08-14 — [Meta sharing image docs](https://developers.facebook.com/docs/sharing/webmasters/images/)).

⚠️ **SVG `og:image` does not render on major networks** — this could not be re-verified from a
primary source: Meta's image page documents sizes but not accepted formats, and the X card
reference pages returned 404 on every URL tried on 2026-08-14. Secondary sources agree X accepts
JPG/PNG/WEBP/GIF under 5 MB and not SVG, and they disagree on X's preferred ratio (1.91:1 vs 2:1 vs
16:9). Treat "PNG, 1200×630" as the safe intersection, not as a quoted spec.

## Why satori

- Declarative JSX-style tree → SVG → PNG pipeline
- Native font embedding (no system font dependency at render time)
- Vercel's own OG generator is a wrapper around it: `@vercel/og@1.0.2` declares
  `satori@0.33.3` + `@resvg/resvg-wasm@2.4.1` as its only runtime dependencies
  (verified 2026-08-28 — npm registry metadata for `@vercel/og`)
- Pairs with `@resvg/resvg-js` for SVG→PNG conversion

**Versions as of 2026-08-28** (npm registry):

| Package | Latest stable | Published | Note |
|---|---|---|---|
| `satori` | 0.33.4 | 2026-08-24 | Moving fast: 0.30.0 → 0.33.4 all landed between 2026-08-20 and 2026-08-24. `0.0.30-beta.1` still sits on the `beta` tag — it is **older** than `latest`, not newer |
| `@resvg/resvg-js` | 2.6.2 | 2024-03-26 | Stable line is over two years old; `2.7.0-alpha.2` (2026-01-28) sits on `next`, repo's last commit 2026-03-26 |
| `@vercel/og` | 1.0.2 | 2026-08-24 | Wraps the two above; 1.0.2 moved its pin to `satori@0.33.3` |
| `astro-og-canvas` | 0.13.0 | 2026-06-30 | Astro-specific wrapper |

Runtime floors: `satori` still declares `engines.node >= 16` at 0.33.4 and its README states it
runs in the browser, Node ≥ 16 and Web Workers; `@resvg/resvg-js` declares `engines.node >= 10` and
its README claims Node 12–22 (`engines` re-checked 2026-08-28 — npm metadata; the README claims
still carry their 2026-08-14 stamp).

> ⚠️ **The four minor releases between 0.29.0 and 0.33.4 have not been read.** Version numbers here
> are current as of 2026-08-28, but the CSS-support and layout claims further down this file were
> written against 0.29.0 and are **not re-verified**. Check them against the satori README before
> quoting them.

## Minimum pattern (Node ESM)

```js
import satori from 'satori';
import { Resvg } from '@resvg/resvg-js';
import { readFile, writeFile } from 'node:fs/promises';

// Fonts must be TTF, OTF or WOFF — WOFF2 is not supported by satori
const fontRoot = 'node_modules/@fontsource';
const plexSans500 = await readFile(`${fontRoot}/ibm-plex-sans/files/ibm-plex-sans-latin-500-normal.woff`);
const plexSans600 = await readFile(`${fontRoot}/ibm-plex-sans/files/ibm-plex-sans-latin-600-normal.woff`);
const plexMono400 = await readFile(`${fontRoot}/ibm-plex-mono/files/ibm-plex-mono-latin-400-normal.woff`);
const instrumentItalic = await readFile(`${fontRoot}/instrument-serif/files/instrument-serif-latin-400-italic.woff`);

const fonts = [
  { name: 'IBM Plex Sans', data: plexSans500, weight: 500, style: 'normal' },
  { name: 'IBM Plex Sans', data: plexSans600, weight: 600, style: 'normal' },
  { name: 'IBM Plex Mono', data: plexMono400, weight: 400, style: 'normal' },
  { name: 'Instrument Serif', data: instrumentItalic, weight: 400, style: 'italic' },
];

const tree = {
  type: 'div',
  props: {
    style: {
      width: '1200px', height: '630px', display: 'flex',
      background: 'linear-gradient(135deg, #0a0b0e 0%, #1e2229 100%)',
      fontFamily: 'IBM Plex Sans', color: '#e7ecf2',
    },
    children: [
      // accent left rule
      { type: 'div', props: { style: { position: 'absolute', top: 0, left: 0, width: '12px', height: '630px', background: '#ffb020', display: 'flex' } } },
      // brand mark as an inline SVG node — no file path to resolve at build time
      {
        type: 'svg',
        props: { width: 72, height: 72, viewBox: '0 0 64 64', style: { position: 'absolute', bottom: '56px', right: '80px' },
          children: [{ type: 'path', props: { d: 'M35 9.85 L42.6 12.31 L29 54.15 L21.4 51.69 Z', fill: '#ffb020' }}]
        }
      },
      // … title + tagline divs …
    ],
  },
};

const svg = await satori(tree, { width: 1200, height: 630, fonts });
const png = new Resvg(svg, { fitTo: { mode: 'width', value: 1200 } }).render().asPng();
await writeFile('./deploy/og-cover.png', png);
```

Both call shapes above are current: `satori(element, options)` with `width` / `height` / `fonts`
(plus `embedFont`, `graphemeImages`, `loadAdditionalAsset`, `debug`, `pointScaleFactor`), and
`new Resvg(svg, opts).render().asPng()` with the `fitTo: { mode, value }` object
(verified 2026-08-14 — [satori README](https://github.com/vercel/satori),
[resvg-js README](https://github.com/thx/resvg-js)).

## Typographic guidance inside OG

- **Primary display**: Instrument Serif italic (or equivalent) at 96-112px for emotional moments
- **Body / tagline**: IBM Plex Sans 500 at 24-30px
- **Eyebrow / URL**: IBM Plex Mono 400 at 18-22px
- **Color**: accent `#ffb020` exact; body `#e7ecf2`; bg charcoal `#0a0b0e`

## Per-page OG variants

For multi-page sites, parameterize by entry:

```js
function template(entry) {
  return { type: 'div', props: { /* … */ children: [
    /* eyebrow: entry.category */
    /* title: entry.title — font-size = entry.title.length > 28 ? 64 : 76 */
    /* description: entry.description */
    /* footer url: entry.url */
  ]}};
}
```

In Astro, wire up `src/pages/og/[...route].ts`: export a `GET` returning a `Response` whose body is
the PNG buffer with `Content-Type: image/png`, and export `getStaticPaths()` enumerating every
entry. Astro writes the response body out as a static file at build time. If the project's `output`
is server rather than static, add `export const prerender = true` to the endpoint or it renders per
request (verified 2026-08-14 — Astro endpoints + on-demand-rendering docs via Context7,
`/withastro/docs`). Serve it with a long immutable cache.

## Gotchas

- **Font formats** — satori supports TTF, OTF and WOFF; **WOFF2 is not supported**
  (verified 2026-08-14 — satori README). `@fontsource/*` ships `.woff` and `.woff2`; take `.woff`.
  The earlier claim in this file that `.ttf` is rejected was wrong.
- **Images are fetched, but only from absolute sources** — satori accepts `props.src` as an HTTP
  URL, a `data:image/…;base64,…` URI, an `ArrayBuffer` or a `Buffer`, and it *will* fetch HTTP URLs.
  There is no base URL, so a relative path resolves to nothing. Its README recommends base64/buffer
  data anyway, to skip the extra I/O (verified 2026-08-14 — satori README). Inline SVG nodes, as in
  the sample above, sidestep the question.
- **Layout engine is a CSS subset, not CSS** — `display` accepts `flex`, `contents` and `none` and
  **defaults to `flex`**; `position` accepts `relative`, `static` and `absolute`, defaulting to
  `relative`; grid is not implemented. `gap`, `linear-gradient`/`radial-gradient` backgrounds,
  `filter` and the `mask-*` properties are in the supported table; `aspect-ratio` is not
  (verified 2026-08-14 — satori README CSS table). Check that table before reaching for a property.
- **Do not truncate the PNG buffer.** A previous version of this file suggested
  `render().asPng().slice(0, X)` to shrink the file — that cuts a PNG mid-stream and produces a
  corrupt image. To reduce size, re-encode: `sharp`, `oxipng`, or drop to JPEG.
- ⚠️ **Expected file size** — the "~100-150 KB at 1200×630" figure carried by earlier revisions is
  unverified and depends entirely on the artwork (a gradient plus text compresses very differently
  from a photo). Measure the output rather than trusting a number.

## Alternatives — tradeoffs, no default winner

- **satori + `@resvg/resvg-js` (this file)** — smallest dependency surface, native Rust rasterizer,
  runs in any Node build step, framework-agnostic. Cost: you write the layout tree by hand, and the
  rasterizer's stable release has not moved since 2024-03-26.
- **`@vercel/og`** — one `ImageResponse` class over the same satori core, JSX in, `Response` out,
  targets edge runtimes. Cost: an extra abstraction and Vercel-shaped defaults; it rasterizes with
  `@resvg/resvg-wasm@2.4.1` (WASM), which is slower than the native binding but portable to runtimes
  with no native addons.
- **`astro-og-canvas`** — declarative config instead of a layout tree, wired into Astro's build.
  Cost: Astro-only, and its template vocabulary is what it is — bespoke layouts fight it.

Pick on runtime and control: native Node build step → satori direct; edge/WASM or an existing
Next.js app → `@vercel/og`; Astro site wanting no layout code → `astro-og-canvas`.

## Verification after deploy

```bash
curl -sI https://DOMAIN/og-cover.png | grep -i content-type  # image/png
curl -sI https://DOMAIN/og-cover.png | grep -i cache-control  # public, max-age=31536000, immutable
```

External validators (see `11-validation-toolkit.md`), reachability checked 2026-08-14:

- https://developers.facebook.com/tools/debug/ — Meta Sharing Debugger, force re-scrape. HTTP 200.
- https://www.linkedin.com/post-inspector/ — LinkedIn preview. HTTP 200.
- https://www.opengraph.xyz/ — paste URL, see rendered card. ⚠️ returned HTTP 429 (rate-limited) on
  the check; the service exists but may throttle automated hits.
- ~~`cards-dev.twitter.com/validator`~~ — **dead for anonymous use**: it 307-redirects to a login
  page (verified 2026-08-14 — `curl -sI`). X publishes no working public card debugger; the only
  reliable check is posting the URL in a draft and reading the preview.

## Skip when

- Coming-soon is one-off, never changes, and a hand-designed static PNG is fine
- Budget doesn't justify Node build tooling — export a PNG from a design tool once
