# Changelog

All notable changes to the `web-launcher` plugin are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Content in this plugin carries verification dates. A release that only re-verifies claims still
gets an entry, because "checked on this date" is the product.

## [Unreleased]

## [0.3.0] - 2026-08-28

### Fixed

- **C1 passed a direct 404.** Yesterday's fix added the `000` no-response branch but left the gap
  underneath it: a path that 404s without ever redirecting matched no branch, so `fail` stayed 0 and
  the check printed `OK redirect targets resolve to 200`. Only an explicit `200` at the end of the
  chain is a pass now. **C5 printed success on a sitemap that never downloaded** — zero URLs read is
  reported as "check did not run", not as a clean site. **C7** gained the same guard.
- **The baseline `wrangler.jsonc` shipped an SPA fallback to every site.**
  `not_found_handling: "single-page-application"` turns any unknown route on a multi-page site into a
  200 serving the homepage — a soft 404 that C1 cannot catch either, because the response genuinely
  is 200. Now tied to the site shape the scan phase already establishes.
- **`03`'s meta template contradicted `01` and `10` on icon format**, using `favicon.svg` for
  `apple-touch-icon` and `Organization.logo` where the other two files say, with sources, that both
  must be raster. Copying the template shipped an apple-touch-icon iOS will not render and a logo
  Google will not put in the Knowledge Panel.
- **The README promised a POSIX shell and the checks were bash-only.** Two blocks used process
  substitution. Rewritten; all 11 blocks in `14-diagnostic-checks.md` and every block in `09` and
  `11` now pass `dash -n` as well as `bash -n`, so the promise is true rather than softened.
- **`pnpm audit … || npm audit …` ran both scanners.** Audit tools exit non-zero when they *find*
  something, so the fallback fired on a real vulnerability and ran the second scanner against the
  wrong lockfile. Replaced with `packageManager`-then-lockfile detection running exactly one, `bun`
  included.
- `09-audit-workflow.md` ended a step with "Commit logically", which is a git mutation the rest of
  the skill does not authorise. Commit, push, deploy, cache purge, sitemap submit and DNS changes now
  each need their own approval, named — approval to apply a change is not approval to publish it.

### Changed

- `16-search-console.md` now defaults to the `webmasters.readonly` scope. The read-write scope buys
  exactly one operation, `sitemaps.submit`, and a token that can submit can submit by accident.
  Widening it requires the user to have asked, and the sitemap URL confirmed immediately before the
  call. The file also states up front that it is a procedure, not shipped client code — there is no
  OAuth handler or API client in this plugin and there is not meant to be.
- `14-diagnostic-checks.md` notes that C5, C7 and C9 request URLs taken from fetched content
  (sitemap `<loc>`, `og:image`), so against a site you do not control those values decide what your
  machine connects to.

### Fixed

- **`15-geo-measurement.md` misclassified `ClaudeBot` as a retrieval crawler.** Anthropic documents
  it as the training crawler; `Claude-User` and `Claude-SearchBot` are the retrieval side. The error
  sat inside the file's own worked example of *verifying a tool's claim against the vendor's
  documentation*, so it undercut the section it was illustrating. The same paragraph explained
  `Google-Extended` while the check GeoDaddy actually runs is `GoogleOther` — a generic
  product-team crawler that is neither a training nor a retrieval control. Both corrected with
  quoted, dated vendor sources, and the "what to do" step now runs a `curl` against the tokens that
  genuinely decide citation.
- **`10-brand-serp.md`'s canned user message reported work that may never have happened.** Line 3 of
  the script stated "Requested indexing via GSC" as a completed fact, but that is a manual
  authenticated UI action. A model pasting the script tells the user the request was made, and the
  user then does not make it.
- **`SKILL.md` ran a full-scope scan in Mode C.** Mode C is defined as brand-only, "without touching
  SEO / deploy", but the scan phase was marked "all modes" and steps 4-7 inventory exactly those
  areas — producing a gap report of unrequested work. Mode C now stops at step 3.
- **`14-diagnostic-checks.md` C1 reported a dead host as healthy.** `curl` returns `000` when it
  never connected; neither branch matched, so the check fell through to its OK line. Now reported as
  an explicit unverified state. **C9 printed nothing at all on a clean site**, which the file's own
  contract says means "the check did not run" — it now prints a counted OK line.
- **`08-cloudflare-deploy.md` justified CSP `'unsafe-inline'` with a false premise.** Inline JSON-LD
  needs no CSP exception: a `<script type="application/ld+json">` is a data block, and the HTML
  spec's "prepare the script element" algorithm returns at the type step before ever reaching the
  inline-behavior CSP check. The baseline no longer carries `'unsafe-inline'` on that basis.
- **`01-brand-application.md` said "one SVG, one ICO, one PNG"** while the `site.webmanifest` in the
  same file references three more PNGs — ship both verbatim and every manifest icon 404s. Its
  verification checklist also grepped for `brand-dot|placeholder-logo`, strings that none of the
  placeholder shapes it describes actually contain; replaced with per-category searches.
- **`og:url` was hardcoded to the site root** in `03`'s meta template and absent from `05`'s
  per-route override list, so every subpage of a multi-page site advertised the homepage as its
  social target. `03` now also gives a bright line for when `05` must be loaded.
- `06-agent-ready.md`'s section header said "ship the top two always" while its own priority matrix
  rates signal 2 as having no known effect and being fine to skip.
- `02-coming-soon-scaffold.md`'s skeleton loaded one font family; its typography section specifies
  three, including the italic serif used in the skeleton's own H1.
- `11-validation-toolkit.md`'s post-deploy step told the model to request reindexing on every
  meaningful change, which `12-indexing.md` separately warns against for the same URL. Now scoped to
  once per URL per content change.
- `12-indexing.md` suggested automating IndexNow via "CF Worker on wrangler deploy event". There is
  no such event; it is a step in the deploy pipeline.
- `13-dependency-security.md` gave no rule for a repo already running both Dependabot and Renovate,
  though `SKILL.md`'s pitfalls list names it. Now says how to choose and that deleting the config
  alone does not stop Renovate.

## [0.2.0] - 2026-08-28

### Fixed

- **Two action pins in `13-dependency-security.md` could not resolve.** The `github/codeql-action`
  v4 row gave a SHA absent from that repository entirely, so a workflow copying it failed to find
  the action rather than running a stale one. `pnpm/action-setup` v6 was pinned to the annotated
  *tag object* instead of its commit. Both came out of the resolution command the file itself
  recommended, which returns `.object.sha` without dereferencing annotated tags; the file now gives
  the one-step form and explains the trap.
- **`06-agent-ready.md` had the AIPREF status backwards.** It described
  `draft-ietf-aipref-attach` as expired and told the reader to wait for it to revive. It revived as
  `-05` on 2026-08-19 and runs to 2027-02-20; `vocab` moved to `-07`. The recommendation built on
  the expiry is corrected in all three places it appeared. Nothing is known to parse
  `Content-Usage` yet, so it stays an intent-only signal.
- `$schema` added to `plugin.json`, pointing at the SchemaStore entry.

### Changed

- `.mcp.json` pins `geodaddy-mcp@0.2.2`. It previously ran `npx -y geodaddy-mcp`, so every session
  executed whatever the registry served as `latest`.
- Version claims refreshed against the registries on 2026-08-28: satori `0.29.0` → `0.33.4`,
  `codeql-action` → `v4.37.9`, TruffleHog → `v3.97.1`, Syft → `v1.51.1`, pnpm → `11.24.0` with a
  note that `12.0.0` is staged on the `next-12` tag. The satori jump spans four minor releases whose
  notes have not been read — the CSS-support claims below it are marked not re-verified rather than
  carried forward silently.

### Added

- `.github/workflows/validate.yml` — runs `claude plugin validate --strict` and checks that every
  row of the capability index in `SKILL.md` points at a reference file that exists, and that every
  reference file appears in the index. Both directions, because a reference nobody routes to is as
  broken as a row pointing at nothing.
- `.github/dependabot.yml` — monthly `github-actions` updates, which is what keeps the pinned
  action SHAs in the workflow from going stale. This plugin recommends the practice; it now
  follows it.

[Unreleased]: https://github.com/voyvodka/web-launcher/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/voyvodka/web-launcher/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/voyvodka/web-launcher/releases/tag/v0.2.0
