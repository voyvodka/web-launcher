# Brand application

*Loaded by `web-launcher` SKILL for Mode C (brand-only) or the brand phase of Mode A/B.*

> **Not re-verified in the 2026-08-14 audit · review by 2026-10-13.** Nothing in this file was
> checked against a current source. Treat version numbers, tokens, API shapes and vendor
> behaviour as unverified until confirmed.

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

4. **Favicon + icon suite** — ensure all carry the mark:
   - `public/favicon.svg` — primary SVG (Beam mark or equivalent)
   - `<link rel="apple-touch-icon" href="/favicon.svg" />`
   - `<link rel="mask-icon" href="/favicon.svg" color="#ACCENT" />` for Safari pinned tabs
   - PNG variants (16, 32, 48, 180, 192, 512) if targeting older browsers — generate via `rsvg-convert` or similar

5. **OG image** — replace old OG cover with new brand mark. Regenerate via satori (see `07-og-satori.md`).

## SVG logotype outlining (production ship)

SVG logotypes using `<text font-family="...">` depend on the viewer having the font installed. For distribution-safe files, outline text to paths:

| Tool | Command / action |
|---|---|
| Inkscape | `inkscape --export-text-to-path input.svg --export-filename=output.svg` |
| Figma | Select text → Flatten (Cmd+E) |
| Illustrator | Type → Create Outlines |
| `svgo` post-process | Doesn't outline; only compresses. Pair with above. |

## Verification checklist

- [ ] No more placeholder marks (`grep -rn "brand-dot\|placeholder-logo" src/`)
- [ ] All brand colors come from `:root` tokens, not hardcoded hex scattered around
- [ ] Font stack matches brand kit in all declarations
- [ ] Favicon renders at 16px distinguishably from neighboring browser tabs
- [ ] Apple-touch-icon + mask-icon both reference updated SVG
- [ ] OG cover regenerated with new brand

## Gotchas

- **Amber exactness** (or whatever the locked color is) — brand rules often forbid tinted variants. Catch `#ffb873`, `#ffcc44`, `#ffa500` drift.
- **Tailwind `theme.extend.colors`** — just adding `amber: "#ffb020"` doesn't override Tailwind's built-in `amber-500`. Rename to `brand-amber` to avoid collision, or use `theme.colors` (replace entirely).
- **Dark vs light mode icon** — mask-icon color is per-user-theme in some browsers; pick the color that works on both dark and light backgrounds (usually the brand accent, not black/white).
