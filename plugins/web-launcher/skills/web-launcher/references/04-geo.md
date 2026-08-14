# GEO — Generative Engine Optimization

*Loaded by `web-launcher` SKILL when user mentions AI search, ChatGPT, Claude Web, Perplexity, answer engines, or wants brand visibility in LLM-generated answers.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.

Traditional SEO ranks your site in 10 blue links. **GEO aims to get you cited inside AI-generated answers** (ChatGPT Search, Claude Web, Perplexity, Google AI Overviews, Bing Copilot).

**Set expectations before doing any of this.** GEO has no measurement surface comparable to Search
Console: no impression count, no citation report, no ranking. Everything below is a plausible
input, not a demonstrated lever. Say that to the user rather than promising visibility.

## Core requirements (beyond classic SEO)

### 1. `llms.txt` — agent-facing, not a ranking lever

See `03-discoverability-classic.md`, which carries the honest framing and the sources: Google
confirmed it has no effect on Search or AI Overviews (verified 2026-08-14), and measured crawler
traffic to `/llms.txt` is negligible. Ship it because it is cheap infrastructure for agents and
because Lighthouse's Agentic Browsing audit checks for it — never as a rankings argument.

Content rules, when you do ship one:
- **Factual, not promotional** — LLMs strip marketing language
- **Scannable** — short paragraphs, bullet lists, H2/H3 hierarchy
- **Entity-dense** — proper nouns, versions, dates inline
- **Linked** — pointers to GitHub, releases, docs (LLMs follow for verification)

### 2. Schema.org types — what Google still renders

**Do not ship `FAQPage` or `HowTo` for rich results. Both are dead on Google.**

| Type | Status | Source |
|---|---|---|
| `FAQPage` | Rich result removed from Search 2026-05-07; documentation page deleted June 2026; Search Console FAQ report and Rich Results Test support withdrawn | [Google — FAQPage](https://developers.google.com/search/docs/appearance/structured-data/faqpage) (verified 2026-08-14) |
| `HowTo` | Absent from the current Search Gallery; desktop rich result removed years earlier | [Google — Search Gallery](https://developers.google.com/search/docs/appearance/structured-data/search-gallery) (verified 2026-08-14 — absence confirmed against the live gallery) |

Both remain valid schema.org vocabulary, and other consumers may parse them. That is not a reason
to add them: markup you cannot verify the effect of is markup you will keep maintaining for
nothing. If a site already ships them, leaving them costs nothing — say so and move on.

**What to ship instead**, all still rendered by Google (verified 2026-08-14 against the Search
Gallery): `Organization` (see `10-brand-serp.md` — this is the one that puts a logo in the SERP),
`Article` / `BlogPosting`, `BreadcrumbList`, `Product`, `SoftwareApplication`, `Event`, `Recipe`,
`VideoObject`.

Write answer-shaped content anyway — a heading that asks the question and a first paragraph that
answers it in two sentences. That helps extraction with no markup and nothing to maintain.

## E-E-A-T signals (Experience, Expertise, Authoritativeness, Trust)

AI engines check authority. Ship:

- **`Person` schema** for authors with full `sameAs`:
  ```json
  {
    "@type": "Person",
    "name": "NAME",
    "url": "https://MAINTAINER_URL",
    "sameAs": [
      "https://github.com/USER",
      "https://linkedin.com/in/USER",
      "https://twitter.com/USER",
      "https://bsky.app/profile/USER",
      "https://mastodon.social/@USER"
    ]
  }
  ```
- Explicit `datePublished` + `dateModified` on content pages
- Editorial transparency: About page, public changelog, methodology notes
- `sameAs` array on `Organization` — every social profile and code host

## Factual extraction patterns

Write content LLMs can pull verbatim:

- **TL;DR at top** — ≤3 sentences, extractable answer to "what is X?"
- **Definition-style headings** — "What is X?", "How does Y work?", "When to use Z?"
- **Fact tables with plain values** — no prose noise
- **Version + date stamps inline** — "As of v2.3, December 2025, X supports Y"
- **Canonical entity naming** — no pronouns far from their antecedent (LLMs lose the thread)
- **Short paragraphs** (≤4 sentences) — chunk-retrievable

## No JS-gated content

Most LLM crawlers do NOT execute JS. SSR or static-render everything critical. Dynamic content loaded post-hydrate is invisible to ChatGPT / Claude / Perplexity bots.

Verification:
```bash
curl -s https://DOMAIN/ | grep -c "important brand text"
# If 0, content is JS-hydrated → LLM won't see it
```

## Entity disambiguation

If your brand name collides with a common word, add disambiguating context everywhere:
- ❌ "PRODUCT works great"
- ✅ "PRODUCT, the open-source CATEGORY tool for PLATFORM, works great"

Prevents LLMs confusing you with unrelated matches. Critical for generic names.

## Testing GEO coverage

Quarterly, query each engine for brand + common intent:
- **ChatGPT** (chat.openai.com with web search on) — "What is $BRAND?", "How do I install $BRAND?"
- **Claude.ai** — same
- **Perplexity** — same, check cited sources
- **Google AI Overview** — search brand + common query on google.com
- **Bing Copilot** — same via Bing

**Red flags**: brand absent from response, or misrepresented (wrong category, wrong platform).

**Fix actions when absent**:
- Update `llms.txt` content (richer, more factual)
- Add an answer-shaped section covering the query topic — question as the heading, answer in the
  first two sentences. No `FAQPage` markup; see above for why
- Check backlinks — LLMs often cite via backlink graph
- Verify no JS-gated content blocking crawlers

## GEO-specific validator

**Measure before recommending.** `15-geo-measurement.md` covers GeoDaddy — open-source, fully
local, 22 checks, available to this plugin as an MCP tool (`analyze_url`) and as a CLI. Run it
before proposing GEO work and again after, and report the delta rather than a single number.

Note the one deliberate disagreement: its `geo-schema-stacking` check wants `FAQPage`, which this
skill drops for the reason given above. Report the lost points as a choice, not as a defect.

The hosted Cloudflare scanner https://isitagentready.com/ overlaps with agent-ready (see
`06-agent-ready.md`) but also flags some GEO basics (llms.txt presence, markdown negotiation).

Manual test every 90 days: ask each AI engine the brand query, paste response here for gap analysis.
