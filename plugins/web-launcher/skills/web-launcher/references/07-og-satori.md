# OG image generation (satori + resvg-js)

*Loaded by `web-launcher` SKILL when generating or updating 1200×630 social cards.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

Social platforms (Twitter/X, Meta, LinkedIn) require PNG/JPEG — SVG og:image does NOT render on major networks. Always ship PNG.

## Why satori

- Declarative JSX-style tree → SVG → PNG pipeline
- Native font embedding (no system font dependency at render time)
- Used by Vercel for Next.js OG images; widely battle-tested
- Pairs with `@resvg/resvg-js` for SVG→PNG conversion

## Minimum pattern (Node ESM)

```js
import satori from 'satori';
import { Resvg } from '@resvg/resvg-js';
import { readFile, writeFile } from 'node:fs/promises';

// Load fonts as Buffers from @fontsource/*
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
      // amber left rule
      { type: 'div', props: { style: { position: 'absolute', top: 0, left: 0, width: '12px', height: '630px', background: '#ffb020', display: 'flex' } } },
      // brand mark inline SVG (avoid relative paths — satori doesn't fetch)
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

## Typographic guidance inside OG

- **Primary display**: Instrument Serif italic (or equivalent) at 96-112px for emotional moments
- **Body / tagline**: IBM Plex Sans 500 at 24-30px
- **Eyebrow / URL**: IBM Plex Mono 400 at 18-22px
- **Color**: amber accent `#ffb020` exact; body `#e7ecf2`; bg charcoal `#0a0b0e`

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

In Astro, wire up `src/pages/og/[...route].ts` with `getStaticPaths` enumerating all doc / compare / blog entries. Emit Content-Type image/png with long immutable cache.

## Gotchas

- **Fonts must be .woff** (not .woff2 or .ttf) for satori. `@fontsource/*` ships both; use the `.woff` variant.
- **No relative images inside satori tree** — satori doesn't fetch. Either inline SVG paths or base64-embed images as data URLs.
- **Layout engine differs from CSS** — flexbox-only, no grid. Some properties aren't supported (`gap` works, `aspect-ratio` doesn't). Refer to https://github.com/vercel/satori for exact CSS subset.
- **PNG size** — 1200×630 comes out ~100-150 KB typically; acceptable. If larger, reduce via `resvg.render().asPng().slice(0, X)` or re-optimize with `sharp`.

## Verification after deploy

```bash
curl -sI https://DOMAIN/og-cover.png | grep -i content-type  # image/png
curl -sI https://DOMAIN/og-cover.png | grep -i cache-control  # public, max-age=31536000, immutable
```

External validators (see `11-validation-toolkit.md`):
- https://www.opengraph.xyz/ — paste URL, see rendered card
- https://developers.facebook.com/tools/debug/ — FB Sharing Debugger, force re-scrape
- https://cards-dev.twitter.com/validator — Twitter/X card render
- https://www.linkedin.com/post-inspector/ — LinkedIn preview

## Skip when

- Coming-soon is one-off, never changes, and a hand-designed static PNG is fine
- Budget doesn't justify Node build tooling — use Figma export PNG once
