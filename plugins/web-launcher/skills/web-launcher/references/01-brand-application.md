# Brand application

*Loaded by `web-launcher` SKILL for Mode C (brand-only) or the brand phase of Mode A/B.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

Take a brand kit (colors, typography, logomark SVG) and apply it across existing components.

## Deliverables

1. **Placeholder swap** — find and replace generic marks in existing code:
   - Amber dots / colored circles used as brand-dot (Header/Footer)
   - Initial letters in avatars
   - Emoji stand-ins
   - Generic Lucide / Heroicons placeholders

   Replace with inline SVG logomark from `/brand/monogram.svg` or equivalent. Size consistently (16px in tight UI, 20px in headers, 24px in hero).

2. **Color normalization** — scan CSS / Tailwind config for:
   - Any color near brand hex but not exact (e.g., `#ffb873` when brand is `#ffb020`)
   - Tinted variants that brand rules don't allow
   - Multiple sources of truth (inline style colors + CSS custom properties both defining brand color)

   Consolidate into CSS custom properties in one file (e.g., `src/styles/tokens.css`):
   ```css
   :root {
     --brand-amber: #ffb020;
     --brand-charcoal: #0a0b0e;
     --brand-reverse: #e7ecf2;
   }
   ```

3. **Font stack alignment** — if brand kit specifies IBM Plex Sans / Inter / Instrument Serif / etc., update:
   - `<link>` imports in HTML / BaseLayout
   - `font-family` CSS declarations (both `body` default and component overrides)
   - Tailwind `theme.fontFamily` config

   Verify by grep:
   ```bash
   grep -rn "font-family" src/ public/ 2>/dev/null | grep -v "IBM Plex" | head -20
   ```

4. **Favicon + icon suite** — see the next section. The short version: one SVG, one ICO, the
   180×180 apple-touch PNG, and the manifest — plus whatever PNGs the manifest itself references
   (the `site.webmanifest` below lists `icon-192`, `icon-512` and a maskable 512, so shipping that
   manifest means generating those three too, or trimming the manifest to match). Still far short
   of the eight-file set older guides ship, but count the manifest's entries before calling it
   done: a manifest pointing at icons nobody generated 404s on every entry and breaks
   add-to-home-screen.

5. **OG image** — replace old OG cover with new brand mark. Regenerate via satori (see `07-og-satori.md`).

## The icon set that is actually needed

```html
<link rel="icon" href="/favicon.ico" sizes="32x32" />
<link rel="icon" href="/favicon.svg" type="image/svg+xml" sizes="any" />
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
<link rel="manifest" href="/site.webmanifest" />
<meta name="theme-color" content="#0a0b0e" />
```

| File | Why it is in the list |
|---|---|
| `favicon.svg` | Primary mark, scales to every tab size. Supported by Chrome/Edge 80+, Firefox 41+, and **Safari only from 26.0** — global support ~90.5% (verified 2026-08-14 — [caniuse: SVG favicons](https://caniuse.com/link-icon-svg)) |
| `favicon.ico` | The fallback that the ~9.5% gap needs, Safari before 26 included. ICO can hold 16/32/48 in one file (verified 2026-08-14 — [MDN: link icon sizes](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/link)) |
| `apple-touch-icon.png` at 180×180 | iOS home-screen icon. **Must be PNG** — Apple's web-content docs specify a PNG file and never an SVG (verified 2026-08-14 — [Configuring Web Applications](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html)). Give it an opaque background; a transparent dark mark can disappear on the home screen. ⚠️ The specific claim that iOS composites transparency to *black* is widely repeated but has no primary source — the safe practice stands either way |
| `site.webmanifest` icons | Install/launch surfaces — see below |

`sizes` is required whenever `rel` is `icon` or a non-standard relation such as `apple-touch-icon`;
`sizes="any"` is the correct value for a vector file (verified 2026-08-14 — MDN, link element).

⚠️ **Generate only what you list.** The old 16/32/48/180/192/512 PNG spread predates SVG favicon
support and mostly ships dead bytes. `rsvg-convert`, `resvg` or equivalent still handles the two
PNGs you do need.

### `mask-icon` — drop it

`<link rel="mask-icon">` was Safari's monochrome pinned-tab icon. Apple's only documentation for it
sits in the Documentation Archive, last updated **2016-12-12** (verified 2026-08-14 —
[Creating Pinned Tab Icons](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/pinnedTabs/pinnedTabs.html)),
MDN's `<link>` reference does not document the relation at all, and Safari has supported real SVG
favicons since 26.0. Keeping it means maintaining a second, single-layer, 100%-black,
`viewBox="0 0 16 16"` file for an ever-shrinking set of older Safari installs.

⚠️ Apple has published no formal deprecation notice — this recommendation rests on the archived
documentation plus MDN's silence, not on a vendor statement. If a brand genuinely targets pre-26
Safari pinned tabs, keeping the tag costs nothing.

### `site.webmanifest`

```json
{
  "name": "PRODUCT",
  "short_name": "PRODUCT",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#0a0b0e",
  "background_color": "#0a0b0e"
}
```

`purpose` takes `any` (default), `maskable` (art drawn inside a safe zone, the rest croppable) and
`monochrome` (color discarded, alpha used as a mask); values may be space-separated. Specify `type`
so the browser does not have to sniff, and use `sizes: "any"` if an icon is SVG (verified
2026-08-14 — [MDN: manifest icons](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference/icons)).

⚠️ The 192/512 pair is convention, not specification — MDN's icons reference prescribes no sizes.
It is what install prompts in practice expect; do not present it as a requirement.

### `theme-color` — set it, do not depend on it

Support is uneven enough that it is a nice-to-have, not part of the brand's visual contract
(verified 2026-08-14 — [caniuse: theme-color](https://caniuse.com/meta-theme-color), ~75% global):

- **Chrome/Chromium**: partial from 73 — desktop applies it to installed PWAs only, Android to the toolbar
- **Safari**: 15 through 18.7 full; **26.0+ only partial** — the meta tag no longer drives the
  browser chrome the way it did
- **Firefox**: no support, desktop or Android

The `<meta>` value overrides `theme_color` in the manifest. Never let a design depend on the
browser chrome picking up a brand color.

## SVG logotype outlining (production ship)

SVG logotypes using `<text font-family="...">` depend on the viewer having the font installed. For distribution-safe files, outline text to paths:

| Tool | Command / action |
|---|---|
| Inkscape | `inkscape --export-text-to-path --export-filename=output.svg input.svg` — the flag converts text objects to paths on export for SVG, PDF, PS and EPS (verified 2026-08-14 — [Inkscape man page](https://inkscape.org/doc/inkscape-man.html)) |
| Figma | Select text → Flatten (Cmd+E) ⚠️ UI step, not checked against current vendor docs |
| Illustrator | Type → Create Outlines ⚠️ UI step, not checked against current vendor docs |
| `svgo` post-process | Doesn't outline; only compresses. Pair with above. |

## Verification checklist

- [ ] No more placeholder marks. There is no single grep for this — the shapes listed in
      deliverable 1 (coloured dots, avatar initials, emoji stand-ins, generic icon-library glyphs)
      usually carry no identifying string. Search by category instead, and read the hits:
      ```bash
      grep -rnE 'lucide|heroicons|react-icons|@tabler/icons' src/     # generic icon imports
      grep -rnE 'rounded-full|border-radius: *50%' src/ | grep -iE 'header|nav|logo|brand'
      grep -rnP '[\x{1F300}-\x{1FAFF}\x{2600}-\x{27BF}]' src/       # emoji stand-ins
      grep -rniE 'placeholder|lorem|TODO.*logo|brand-dot' src/        # only catches labelled ones
      ```
- [ ] All brand colors come from `:root` tokens, not hardcoded hex scattered around
- [ ] Font stack matches brand kit in all declarations
- [ ] Favicon renders at 16px distinguishably from neighboring browser tabs
- [ ] `favicon.svg` **and** `favicon.ico` both present — the SVG alone leaves older Safari blank
- [ ] `apple-touch-icon.png` is a 180×180 **PNG** with an opaque background, not the SVG
- [ ] `site.webmanifest` linked, with a `maskable` icon and `theme_color` matching the meta tag
- [ ] OG cover regenerated with new brand

## Gotchas

- **Amber exactness** (or whatever the locked color is) — brand rules often forbid tinted variants. Catch `#ffb873`, `#ffcc44`, `#ffa500` drift.
- **Tailwind `theme.extend.colors`** — just adding `amber: "#ffb020"` doesn't override Tailwind's built-in `amber-500`. Rename to `brand-amber` to avoid collision, or use `theme.colors` (replace entirely).
- **Pointing `apple-touch-icon` at an SVG** is the most common bug in this area, and it fails
  silently — no build step, validator or desktop browser flags it, because only iOS reads the tag.
  Apple's documentation specifies PNG; ⚠️ what iOS falls back to when the file is not PNG is not
  documented, so verify on a device rather than assuming a particular fallback.
- **Dark vs light mode icon** — an SVG favicon can adapt via `prefers-color-scheme` inside the SVG
  itself, but the ICO and the apple-touch-icon cannot. Pick a mark that reads on both dark and
  light chrome, usually the brand accent rather than black or white.
