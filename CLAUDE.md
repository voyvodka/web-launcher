# web-launcher

> Main context source for this repository. Read this first, then `docs/product/00-state.md`.
> Written in English by convention; the product documents under `docs/` are in Turkish and are
> **not committed** — see "Where the truth lives".

## What this is

A Claude Code plugin that diagnoses why a live site is not showing up in Google, fixes what it
finds, and verifies the fix — plus the launch side it grew out of: discoverability scaffolding,
Cloudflare Workers deployment, and repository hardening.

Diagnosis is the primary job. A site is launched once and diagnosed for the rest of its life.

The distinguishing promise is **zero stale advice**: every technical claim carries the date it was
verified and the source it came from, and the plugin puts itself on a 60-day clock.

## Stack

| Layer | Choice | Why |
|---|---|---|
| Form | One skill: thin `SKILL.md` + on-demand references | Triggering stays in one place; progressive disclosure keeps the always-loaded part small |
| Frontmatter | Agent Skills spec fields only | Also runs on claude.ai, Cowork, routines — portability is free here |
| Distribution | Dual: `@skills-dir` for the maintainer, GitHub marketplace for everyone else | An installed plugin lives in a versioned cache; edits to it do not survive an update |
| Search Console | Read-only API, desktop OAuth client | The write side is closed — see Rules. ADC was rejected: it needs the gcloud SDK, which is a heavier install than one browser round |
| Deploy target (of sites it builds) | Cloudflare Workers | Knowing one platform properly beats knowing four halfway |

Reasoning in full: `docs/product/02-decisions.md`. Do not swap a component without reading the
"Elenenler" line for it — the alternatives were considered and rejected for stated reasons.

## Where the truth lives

`docs/README.md` indexes every document and what each one answers. Read it rather than listing the
directory, and keep it current when adding to `docs/` — the index is maintained there, not here.

**`docs/` is gitignored.** It exists only on the maintainer's machine. Never link to it from a
committed file: for everyone who clones this repository, that link is dead.

Read `docs/product/00-state.md` before proposing any implementation plan, then check
`02-decisions.md`; ideas that look obvious are often already settled there, with the reason and
the accepted cost.

Product documents are the memory. When code and documents disagree, raise it — do not silently
follow one of them.

## Rules

- **No technical claim without a verification date and a source.** This is the product, not a
  documentation preference. A claim that cannot be sourced is written as unverified or left out.
- **Never write state into `${CLAUDE_PLUGIN_ROOT}`.** It is a versioned cache directory and changes
  on every update. Anything the plugin must keep goes to `${CLAUDE_PLUGIN_DATA}`.
- **The plugin never updates itself, never cuts a release, never changes git state.** It proposes
  and waits for approval. "Self-improving" means exactly this and nothing more.
- **Search Console is read-only apart from `sitemaps.submit`.** There is no request-indexing and no
  start-validation endpoint; do not write code that pretends otherwise.
- **Never branch on `coverageState` text.** It is free-form and localized — the same state reads
  differently in Turkish and English. Branch on `pageFetchState`, `indexingState`, `googleCanonical`.
- **Audit is platform-agnostic; deploy and scaffolding are Cloudflare-only.** Diagnosis runs over
  live HTTP and must not assume the origin.
- **A written rule is not a check.** Anything the plugin claims to catch needs a runnable command
  that produces a one-line verdict. This gap is the reason the project exists.
- **Nothing under `plugins/` names a real person, site, repository, or machine.** This ships
  publicly. Examples use placeholders (`DOMAIN`, `PRODUCT`, `NAME`); the author fields in the
  manifests and `LICENSE` are the only identity in the repository. Findings from real audits are
  generalized before they enter a reference — the pattern travels, the site name does not.
- **A tool's own message is not a source.** GeoDaddy reports that blocking `GPTBot` removes a site
  from ChatGPT search; OpenAI's documentation says `GPTBot` is training-only. When an integrated
  tool makes a claim, verify it before acting on it — and record where it is wrong.
- Plugin content — skills, references, agents, identifiers, commit messages — is English.

## Standing assumptions

> These were never confirmed. When work touches one, ask rather than guess.

| ID | Assumption | What breaks if wrong |
|---|---|---|
| V4 | Most users will not enable auto-update, so a "new version available" notice never reaches them | The freshness promise holds for the maintainer but collapses for users |
| V6 | The maintainer file is genuinely absent from every installed copy | Maintainer instructions become visible to users, breaking the privacy requirement it exists for |

Live list: `docs/product/00-state.md`.

## Out of scope

> Deliberately excluded. Do not helpfully add these.

- Deploying anywhere other than Cloudflare Workers
- Content and copywriting
- Analytics, consent banners, GDPR/KVKK flows
- Deep application SEO — hreflang, e-commerce product schema, logged-in SPAs
- WebMCP and the other new agent surfaces — parked, not rejected
- Fixing the maintainer's own sites; this repository builds the tool, not the sites

Full list with reasons: `docs/product/03-mvp.md`.
