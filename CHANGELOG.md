# Changelog

All notable changes to the `web-launcher` plugin are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Content in this plugin carries verification dates. A release that only re-verifies claims still
gets an entry, because "checked on this date" is the product.

## [Unreleased]

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

[Unreleased]: https://github.com/voyvodka/web-launcher/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/voyvodka/web-launcher/releases/tag/v0.2.0
