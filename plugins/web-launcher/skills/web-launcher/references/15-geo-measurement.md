# GEO measurement — scoring a site with GeoDaddy

*Loaded by `web-launcher` SKILL whenever GEO work is proposed, and in any Mode B audit that touches
AI-search visibility. This is the measurement half of `04-geo.md`, which is the advice half.*

> **Verified 2026-08-14 · review by 2026-10-13.** Versions, tool names and check IDs were read
> from the tool's own repository and the npm registry on that date.

## Why this exists

`04-geo.md` recommends GEO work and has no way to tell whether any of it landed. GEO has no Search
Console: no impressions, no citation report. A recommendation nobody can score is a recommendation
nobody can defend.

[GeoDaddy](https://geodaddy.dev/) closes part of that gap. It is an **open-source (MIT), fully
local** analyzer — no account, no API key, no cloud round-trip — that runs 22 checks across four
categories and returns a scored JSON report with fix recommendations.
✓ ([README](https://github.com/borabiricik/geodaddy-cli), verified 2026-08-14)

**What it actually measures: the inputs, not the outcome.** It scores whether a page carries the
signals believed to drive AI citation. It cannot tell you whether ChatGPT cites you. Say that to
the user before showing a score, or the number will be read as visibility.

## Two ways in — pick per task

| Path | Use when | Cost |
|---|---|---|
| **MCP server** — this plugin declares `geodaddy` in its `.mcp.json` | Interactive audits. The `analyze_url` tool returns a structured report the model can reason over directly | First run pulls `geodaddy-mcp@0.2.2` via `npx` — the version is **pinned** in `.mcp.json`, so a new upstream publish does not silently change what runs; the wrapper then downloads the binary from GitHub releases |
| **CLI** — `geodaddy <URL>` | CI, scripted runs, anything reproducible. Single binary, no runtime dependency | One-time install |

```bash
# install (macOS/Linux)
curl -fsSL https://raw.githubusercontent.com/borabiricik/geodaddy-cli/main/install.sh | sh
geodaddy --version

geodaddy https://example.com --beauty              # human-readable
geodaddy https://example.com                       # JSON, for CI
geodaddy https://example.com --max-pages 50        # crawl the site
geodaddy https://example.com --vitals              # + Core Web Vitals (downloads Chromium ~150 MB)
geodaddy compare https://mysite.com https://rival.com --beauty
```

**Version drift — measured, not assumed (2026-08-14).** The standalone CLI is `v0.8.0` (released
2026-07-06). The npm wrapper `geodaddy-mcp` is `0.2.2` (published 2026-03-25), and **the analyzer
binary it downloads reports `geodaddy 0.2.2`** — checked by running the bundled binary directly.
So the MCP path runs an analyzer roughly four months behind the CLI. Both paths were exercised on a
live site and returned coherent reports, but when a number matters, prefer the CLI and say which
path produced the score.

## Scoring model — read this before quoting a number

- Severity → points: **Critical 10 · Medium 5 · Minor 2**. Pass earns full, **warn earns half**,
  fail earns zero.
- Category score = points earned ÷ points possible × 100.
- Overall = mean of Technical, Content, GEO — plus Performance only when `--vitals` ran.
- **A category with no applicable checks scores 100.** A site can post a high overall while a whole
  category was never exercised. Always report which categories actually ran.

The four categories: `tech-` (meta, headings, HTTPS, redirects, robots, sitemap), `cont-` (heading
structure, alt text, JSON-LD, semantic HTML), `geo-` (listicle formatting, AI bot access, schema
stacking), `perf-` (Core Web Vitals, `--vitals` only).

## Where this tool and this skill disagree

**Treat these as known divergences, not as findings.** Do not "fix" a site to raise a score when
the fix contradicts a sourced position — say which one you are optimizing for and let the user
choose.

| Check | GeoDaddy expects | This skill's position |
|---|---|---|
| `geo-schema-stacking` (Medium, 5 pts) | `Article` **+ `ItemList` + `FAQPage`** all present | `FAQPage` was dropped: Google removed the rich result 2026-05-07 and deleted the documentation page. See `04-geo.md`. **Following this skill costs points here.** |
| `geo-ai-bot-*` (Critical, 10 pts each) | `GPTBot`, `ClaudeBot`, `PerplexityBot`, `GoogleOther`, `Bytespider`, `CCBot` not blocked | Right that a catch-all `Disallow` is expensive — six critical checks, 60 points. **But four of the six tokens it checks are training or generic crawlers, not the retrieval crawlers that decide citation.** See below |
| `geo-listicle` (Medium) | Numbered headings, "Top N" patterns, ordered lists, comparison tables | Reasonable as a formatting heuristic. It scores *shape*, not usefulness — do not restructure good prose into a listicle purely for this |

The `FAQPage` conflict is the one that will come up. Both positions are defensible: GeoDaddy scores
for AI extraction across engines, this skill declines markup Google no longer renders. **Report the
score, name the divergence, and let the user decide** — do not silently re-add `FAQPage` to chase a
number, and do not hide that the number is lower because of a deliberate choice.

### The training-vs-retrieval error — do not act on this one

When `geo-ai-bot-gptbot` fails, GeoDaddy reports: *"GPTBot is blocked in robots.txt. This prevents
your content from appearing in ChatGPT search."* **That sentence is wrong**, and acting on it
reverses a policy the site may hold deliberately.

OpenAI's own documentation, verified 2026-08-14
([developers.openai.com/api/docs/bots](https://developers.openai.com/api/docs/bots)):

| Token | What it actually does |
|---|---|
| `GPTBot` | **Training only.** *"Disallowing GPTBot indicates a site's content should not be used in training generative AI foundation models."* |
| `OAI-SearchBot` | **Search.** *"Sites that are opted out of OAI-SearchBot will not be shown in ChatGPT search answers."* |
| `ChatGPT-User` | User-initiated fetches; *"not used to determine whether content may appear in Search"* |

So **blocking `GPTBot` costs you nothing in ChatGPT search** — that is `OAI-SearchBot`'s job.

The same split exists for Anthropic and for Google, and the tokens are easy to mix up. Verified
2026-08-28 against each vendor's own documentation:

| Token | What it actually does | Source |
|---|---|---|
| `ClaudeBot` | **Training.** *"collecting web content that could potentially contribute to their training"* | [Anthropic crawler support article](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler) |
| `Claude-User` | User-initiated fetches — *"When individuals ask questions to Claude, it may access websites using a Claude-User agent"* | same |
| `Claude-SearchBot` | **Search.** *"navigates the web to improve search result quality for users"* | same |
| `Google-Extended` | **Training + grounding opt-out** for Gemini/Vertex. *"does not impact a site's inclusion in Google Search nor is it used as a ranking signal"* | [Google crawler docs](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers) |
| `GoogleOther` | **Neither.** A generic agent *"used by various product teams for fetching publicly accessible content… one-off crawls for internal research and development"* | same |

Two things follow, and GeoDaddy gets both wrong:

- **`ClaudeBot` is the training crawler, not the retrieval one.** Allowing it does not make a site
  citable in Claude; that is `Claude-SearchBot` and `Claude-User`. A site that blocks `ClaudeBot`
  and allows those two is opted out of training and fully available for citation — GeoDaddy scores
  it as a failure anyway.
- **`GoogleOther` is not a training control at all**, so blocking it neither protects the site from
  training use nor costs it Search visibility. The Gemini training opt-out is `Google-Extended`,
  which GeoDaddy does not check.

A site that allows retrieval crawlers (`OAI-SearchBot`, `ChatGPT-User`, `Claude-SearchBot`,
`Claude-User`, `PerplexityBot`) while blocking training crawlers (`GPTBot`, `ClaudeBot`, `CCBot`,
`Google-Extended`, `Bytespider`) holds a coherent position: *cite me, do not train on me.* GeoDaddy
scores exactly that as multiple critical failures, because it checks the training tokens and calls
them citation blockers.

**What to do:** report the lost points, state that the failures are a stated policy rather than a
misconfiguration, and then check the tokens that actually decide citation — none of which GeoDaddy
scores. Run:

```bash
curl -s https://DOMAIN/robots.txt | grep -iE 'OAI-SearchBot|ChatGPT-User|Claude-SearchBot|Claude-User|PerplexityBot'
```

No `Disallow` against those means citation visibility is intact regardless of the score. Report that
result, not the score, when the user asks whether they are visible to AI search. Only recommend unblocking training crawlers
if the user actually wants to be trained on, and say that is what the change means.

Bytespider is a related case: GeoDaddy treats blocking it as critical, while `03-discoverability-classic.md`
notes it is widely reported to ignore `robots.txt` anyway. Allowing it earns the points and denies
nothing in practice; just do not describe blocking it as a control.

## Where it overlaps `14-diagnostic-checks.md`

`tech-redirect-chains`, `tech-robots-txt`, `tech-sitemap-xml`, `tech-https` cover ground the
diagnostic checks also cover — but as **pass/fail scores, not as diagnoses.** GeoDaddy says a
redirect chain exists; C1/C4 say which URL, how many hops, and which file causes it.

**Order of operations:** GeoDaddy first for the scored picture, `14-diagnostic-checks.md` for
anything it flags that needs a cause. Never report a GeoDaddy failure as a finding without tracing
it to a file or a dashboard setting.

## Reporting

Report the four category scores, then **what changed since the last run** if there is a previous
report — a single score is a number, a delta is evidence. Re-run after every fix and show both.

Never present the overall score as AI-search visibility. It is a signal checklist, and it says so.
