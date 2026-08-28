# Dependency & supply-chain health

*Loaded by `web-launcher` SKILL for repo hardening phase, or when user asks about dep-bot / security / audit / SBOM / license compliance.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.
>
> **Partially re-verified 2026-08-28**: every version number and every pinned SHA in this file was
> re-resolved on that date. Nothing else was re-checked — the prose, pricing and policy claims still
> carry their 2026-08-14 stamps.

Static/marketing sites still ship third-party code (framework, build tools, wrangler, satori, resvg, font packages). Dependencies rot: CVEs appear, breaking changes land, transitive deps get yanked. Automate this — manual review doesn't scale.

## What to audit in any repo

- `.github/dependabot.yml` or `renovate.json` — automated dep updates. **Exactly one of the two.**
  If the scan finds both live, that is the finding: they open duplicate PRs for the same upgrade and
  each closes the other's. Keep Renovate if the repo already uses grouping, scheduling or a shared
  preset; keep Dependabot otherwise, since it needs no config file to be useful. Removing the loser
  means deleting its config *and* disabling it in repo settings or uninstalling the app — a deleted
  `renovate.json` alone leaves the app running on defaults.
- `.github/workflows/*audit*.yml` — CI audit gate
- GitHub repo Settings → Security → Advanced Security toggles (user confirms in UI)
- A ruleset (or classic branch protection) on `main` — PR + status checks required
- Lockfile committed (pnpm-lock.yaml / yarn.lock / package-lock.json) and used with `--frozen-lockfile` in CI
- Pre-commit hook for secret scanning

## 13.1 Automated dep updates — pick ONE

**Don't run both Dependabot and Renovate on the same repo — duplicate PR flood.**

### Option A: GitHub Dependabot (zero-install, GitHub-native)

`.github/dependabot.yml`:
```yaml
version: 2
updates:
  # npm / pnpm / yarn (auto-detected from lockfile)
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "05:00"
      timezone: "Europe/Istanbul"
    open-pull-requests-limit: 5
    groups:
      minor-patch:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    # Wait out the window where a compromised release is still unnoticed
    cooldown:
      default-days: 7
      semver-major-days: 14
    labels: ["dependencies"]
    commit-message:
      prefix: "chore(deps)"
      include: "scope"

  # GitHub Actions workflow pinning
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: "weekly" }
    labels: ["dependencies", "ci"]

  # Docker (if Dockerfile present)
  - package-ecosystem: "docker"
    directory: "/"
    schedule: { interval: "monthly" }
```

> Schema still `version: 2`; every key above is current (verified 2026-08-14 against the
> [Dependabot options reference](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference)).
> Notes from that check:
> - `open-pull-requests-limit` defaults to `5` for version updates — the line above is explicit, not
>   a change in behaviour.
> - `directory` is still valid; `directories` (a list, glob-capable) is the newer plural form.
> - `schedule.interval` accepts `daily`, `weekly`, `monthly`, `quarterly`, `semiannually`, `yearly`,
>   `cron`. `day` / `time` / `timezone` narrow the non-cron intervals.
> - `cooldown` is a real option, not invented for this doc: `default-days`, `semver-major-days`,
>   `semver-minor-days`, `semver-patch-days`, plus `include` / `exclude`. ⚠️ Availability per
>   ecosystem is not stated on that page — if PRs still arrive same-day, the ecosystem may not
>   honour it yet.

### Option B: Renovate (more configurable, supports automerge)

Install Renovate GitHub App, then `renovate.json`:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":preserveSemverRanges"],
  "schedule": ["before 5am on monday"],
  "timezone": "Europe/Istanbul",
  "dependencyDashboard": true,
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    },
    {
      "matchPackageNames": ["wrangler"],
      "schedule": ["at any time"],
      "labels": ["wrangler"]
    }
  ],
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "schedule": ["at any time"]
  }
}
```

> `config:recommended` is the current name and `:preserveSemverRanges` still exists (verified
> 2026-08-14 against the [Renovate preset list](https://docs.renovatebot.com/presets-config/)).
> If you are copying from an older guide: `config:base` was the pre-rename name and no longer
> appears in the preset docs — replace it. The `$schema` URL above is the one Renovate publishes,
> so a schema-aware editor will flag a bad key before Renovate ever runs.

## 13.2 GitHub repo-level security toggles (user action)

Enable in `Settings → Security → Advanced Security` (the menu path moved — an older guide saying
`Settings → Code security` is describing the same toggles under the previous label; verified
2026-08-14 against the
[code scanning default setup docs](https://docs.github.com/en/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning)):

- ✅ **Dependabot alerts** — CVE notifications
- ✅ **Dependabot security updates** — auto-PR on alert
- ✅ **Secret scanning** — runs automatically and free on **public** repos. Private/internal repos
  need a paid GitHub Secret Protection license (verified 2026-08-14 —
  [about secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning))
- ✅ **Secret scanning push protection** — blocks the push, not the commit, when a supported secret
  is detected. Two distinct things share the name: *push protection for users* is *on by default*
  and stops any user pushing a secret to a **public** repo; *push protection for the repository* is
  **off by default** and must be switched on by an admin/owner (verified 2026-08-14 —
  [about push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection))
- ✅ **Code scanning** (CodeQL default setup) — free where the repo "is publicly visible, or GitHub
  Code Security is enabled"; private repos need the paid license (verified 2026-08-14, same code
  scanning page). Optional; adds CI time
- ✅ **Private vulnerability reporting** — lets researchers file via `security/advisories`

⚠️ Every free/paid line here is licensing, which GitHub renames and re-tiers more often than it
changes behaviour ("Advanced Security" has already split into Code Security and Secret Protection).
Confirm in the repo's own settings page before promising a user something is free.

## 13.3 CI audit gate

`.github/workflows/audit.yml`:
```yaml
name: Audit dependencies

on:
  pull_request:
    paths:
      - "package.json"
      - "pnpm-lock.yaml"
      - "yarn.lock"
      - "package-lock.json"
  schedule:
    - cron: "0 5 * * 1"   # weekly Mon 05:00 UTC
  workflow_dispatch:

permissions: { contents: read }

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: pnpm/action-setup@v6
        with: { version: 10 }
      - uses: actions/setup-node@v7
        with: { node-version: 24, cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm audit --audit-level=moderate
      - run: pnpm outdated --long || true

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Organization-owned repos only; personal accounts leave this unset
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

Every version in that workflow was checked on 2026-08-14 via `gh api repos/<owner>/<repo>/releases/latest`.
What changed since the previous major-version generation of this example:

| Action | Was | Now | Latest release |
|---|---|---|---|
| `actions/checkout` | v4 | **v7** | `v7.0.1`, 2026-07-20 |
| `actions/setup-node` | v4 | **v7** | `v7.0.0`, 2026-07-14 |
| `pnpm/action-setup` | v4 | **v6** | `v6.0.10`, 2026-08-03 |
| `gitleaks/gitleaks-action` | v2 | **v3** | `v3.0.0`, 2026-05-30 |
| `github/codeql-action` | v3 | **v4** | tag `v4.37.9` (2026-08-26) |

Three things that bite when copying an older version of this file:

- **`node-version: 20` is end-of-life.** Node 20 (Iron) went EOL 2026-04-30; Node 22 (Jod) is in
  maintenance since 2025-10-21 and ends 2027-04-30; **Node 24 (Krypton) is the active LTS** until
  it enters maintenance 2026-10-20 (verified 2026-08-14 against the
  [Node release schedule](https://github.com/nodejs/Release/blob/main/schedule.json)). Use 24 unless
  something pins you lower.
- **`pnpm/action-setup` needs `version:`** unless `package.json` declares `packageManager` (or
  `devEngines.packageManager`). The old example omitted it and silently depended on that field
  existing. It also now carries a successor notice: for **pnpm v11+** the replacement is
  [`pnpm/setup`](https://github.com/pnpm/setup) (`v2.0.2`, 2026-08-09), which installs the
  standalone pnpm binary and a runtime in one step, replacing `actions/setup-node`:
  ```yaml
  - uses: pnpm/setup@v2
    with: { version: 11, runtime: node@24, cache: true }
  ```
  `pnpm/action-setup` remains correct for pnpm v10 and older. (pnpm latest on npm: `11.24.0`,
  2026-08-26. `12.0.0` is already published on the `next-12` dist-tag as of 2026-08-26 — when it is
  promoted to `latest`, re-check the `version: 11` in the snippet above and the v11-vs-v10 split
  in this bullet.)
- **gitleaks-action v3 wants `GITLEAKS_LICENSE` for organization repos**, not personal accounts —
  that is a v2-era condition that still holds, and a missing license is the usual reason the job
  fails on an org repo. v3 is otherwise a drop-in: same inputs and outputs, only the Actions runtime
  moved from Node 20 to Node 24 (verified 2026-08-14 against the
  [gitleaks-action README](https://github.com/gitleaks/gitleaks-action)).

Package manager equivalents (flags verified 2026-08-14 against each tool's own CLI docs):
| Manager | Audit | Outdated |
|---|---|---|
| pnpm | `pnpm audit --audit-level=moderate` | `pnpm outdated --long` |
| yarn (berry) | `yarn npm audit --severity moderate --recursive` | ⚠️ no `yarn outdated` — see below |
| npm | `npm audit --audit-level=moderate` | `npm outdated` |

- `--audit-level` accepts `info` `low` `moderate` `high` `critical` `none` on npm
  ([npm-audit](https://docs.npmjs.com/cli/v11/commands/npm-audit)); pnpm accepts `low` `moderate`
  `high` `critical`, defaulting to `low` ([pnpm audit](https://pnpm.io/cli/audit)). `moderate` is
  valid on both.
- **`yarn outdated` does not exist in modern Yarn.** It was a Yarn 1 command and is absent from the
  Yarn Berry command list — the old table entry would fail with an unknown-command error. The
  built-in equivalent is `yarn upgrade-interactive` (interactive, not CI-friendly); for a
  non-interactive check, run `npm outdated` against the same `package.json`. Yarn's audit flag is
  `--severity` (`info`/`low`/`moderate`/`high`/`critical`), not `--audit-level`
  ([yarn npm audit](https://yarnpkg.com/cli/npm/audit)). Note the npm `yarn` package is still
  `1.22.22` (2024-03-09) — that is the Yarn 1 shim, not a sign Berry is unmaintained.

## 13.4 Secret scanning — pre-commit + CI

CI: already in §13.3 (Gitleaks).

Pre-commit via husky:
```bash
pnpm add -D husky
pnpm exec husky init
echo 'gitleaks git --pre-commit --staged --verbose' >> .husky/pre-commit
```

> `gitleaks protect` was deprecated in v8.19.0 — still present but hidden from `--help`. Current
> modes are `git`, `dir`, `stdin`. Verified 2026-08-14 against the
> [gitleaks README](https://github.com/gitleaks/gitleaks). Note gitleaks is a Go binary, not an npm
> package: `npx gitleaks` does not install it. Use Homebrew, the release binary, or the Docker image.

Or via `pre-commit` framework in `.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks: [{ id: gitleaks }]
```

> `rev: v8.18.0` was two years of releases behind — gitleaks latest is `v8.30.1`, published
> 2026-03-21 (verified 2026-08-14 via `gh api repos/gitleaks/gitleaks/releases/latest`). A stale
> `rev` here is not cosmetic: the rule set that detects new provider token formats ships with the
> binary. Pin a real tag and let Dependabot's `pre-commit` ecosystem or `pre-commit autoupdate`
> move it.

**If a secret was committed — order of operations**:
1. **Rotate the credential** at the provider (revoke, regenerate) FIRST
2. Then scrub history: `git filter-repo --path <file> --invert-paths` (or BFG Repo Cleaner).
   `git-filter-repo` is a separate install (`brew install git-filter-repo`), not a built-in git
   subcommand; latest release `v2.47.0`, 2024-12-04 — quiet, not abandoned (verified 2026-08-14)
3. `git push --force-with-lease` with team coordination
4. Invalidate any deploys that used the old secret

Never just `git rm` the file — old commits retain it forever in the object store.

## 13.5 License compliance

Catch incompatible licenses (GPL polluting MIT is the classic footgun):
```bash
pnpm licenses list                                    # pnpm native, no extra dependency
npx license-checker-rseidelsohn --summary             # maintained fork
npx license-checker-rseidelsohn --failOn 'GPL-3.0;AGPL-3.0;SSPL-1.0'
```

> Do not use `license-checker`: latest is `25.0.1`, published 2019-01-10 — unmaintained for over
> seven years. The fork `license-checker-rseidelsohn` is at `5.0.1` (2026-05-27) and is the drop-in
> replacement. Both verified 2026-08-14 against the npm registry. Prefer the package manager's own
> command where one exists — that is the dependency you already have.

Add as CI step if project has license policy.

## 13.6 SBOM (Software Bill of Materials)

Some enterprise customers require. US EO 14028 pushed adoption:
```bash
# CycloneDX (industry standard) — npm package, npx works
npx @cyclonedx/cyclonedx-npm --output-file sbom.cdx.json

# SPDX (Linux Foundation standard) — Syft is a Go binary, install it first
brew install syft            # or: curl -sSfL https://get.anchore.io/syft | sh -s -- -b /usr/local/bin
syft dir:. -o spdx-json > sbom.spdx.json
```

> **`npx @anchore/syft` does not work — there is no such npm package** (registry returns 404,
> verified 2026-08-14). `@cyclonedx/cyclonedx-npm` is real and current: `6.0.1`, published
> 2026-08-11 (same check). If only one SBOM format is needed, CycloneDX is the one with no
> extra install step. Syft latest is `v1.51.1`, 2026-08-27.

Third option, zero install, if the repo is on GitHub with the dependency graph enabled — GitHub
generates an SPDX SBOM itself:
```bash
gh api repos/OWNER/REPO/dependency-graph/sbom > sbom.spdx.json
```
Verified working 2026-08-14 (returns an SPDX document; the endpoint emitted `SPDX-2.3` on the
repo tested). Free on public repos, and it needs no local toolchain — but it describes what the
dependency graph knows, not what a build actually produced, so it is the weaker artifact of the
three for attestation purposes.

Format status, verified 2026-08-14 via each specification's own releases:
- **CycloneDX** specification `1.7.1`, 2026-06-02 — still actively versioned. Check which spec
  version your generator emits; a tool can be current while defaulting to an older spec.
- **SPDX** specification `3.0.1`, 2024-12-17. ⚠️ Most tooling still emits SPDX **2.3** by default
  (Syft's `spdx-json` output and the GitHub endpoint above both did on the check date). If a
  customer contract names SPDX 3.x specifically, verify the emitted `spdxVersion` field rather
  than assuming.

Attach to GitHub Release artifacts.

## 13.7 Supply-chain hardening

- **npm provenance** — `npm publish --provenance` attests the package was built from the claimed
  source repo at the claimed commit (Sigstore-backed, free). Still the documented flag, with two
  conditions worth knowing (verified 2026-08-14 —
  [generating provenance statements](https://docs.npmjs.com/generating-provenance-statements)):
  the build must run on a cloud-hosted runner of a supported provider — **GitHub Actions or GitLab
  CI/CD, those two only** — and `package.json` must point at a public `repository`. If you publish
  via **trusted publishing**, provenance is generated automatically and the `--provenance` flag is
  not needed; that is now the better path, since it also removes the long-lived npm token from CI.
  Consumers verify with `npm audit signatures`.
- **Lockfile discipline** — always commit, always `--frozen-lockfile` / `--immutable` in CI (fails if lockfile drifts)
- **Pinned third-party Actions** — use commit SHA, not floating tags:
  ```yaml
  # Good — SHA, with the tag it resolved to in a trailing comment
  uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1  # v7
  # Risky
  uses: some-user/some-action@v1
  ```
  First-party (actions/*, github/*) can stay on tags; third-party pin SHA.

  > The SHA previously given here (`b4ffde65…`) still resolves — but to a commit from **2023-10-17**,
  > i.e. the `actions/checkout` v4 line, four majors behind. That is the failure mode of pinning
  > without a bot: it keeps working and quietly stops being maintained. The SHAs above and below
  > were re-resolved 2026-08-28; re-resolve rather than copying them, and let Dependabot's
  > `github-actions` ecosystem move the pins (it rewrites SHAs and updates the trailing comment).
  >
  > **Dereference annotated tags or the pin is wrong.** `gh api repos/<owner>/<repo>/git/ref/tags/<tag>`
  > returns `.object.type == "tag"` for an annotated tag, and its `.object.sha` is the *tag object*,
  > not a commit — pinning it does not resolve. Two rows of this table were wrong for exactly that
  > reason until 2026-08-28. Resolve in one step instead:
  >
  > ```bash
  > gh api repos/<owner>/<repo>/commits/<tag> --jq .sha    # follows tag -> commit for both kinds
  > ```
  >
  > | Action | Major tag | Commit SHA on 2026-08-28 |
  > |---|---|---|
  > | `actions/checkout` | v7 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
  > | `actions/setup-node` | v7 | `820762786026740c76f36085b0efc47a31fe5020` |
  > | `github/codeql-action` | v4 | `cdf488f595d80d6e07e03d4674febd5ab45fa938` |
  > | `pnpm/action-setup` | v6 | `0977fd99725f1db4007ccb2928dbb4e90d06cc86` |
  > | `gitleaks/gitleaks-action` | v3 | `e0c47f4f8be36e29cdc102c57e68cb5cbf0e8d1e` |

- **socket.dev** — GitHub App reviews dep-change PRs for supply-chain red flags (install scripts,
  shell exec, network calls, typosquat). Zero config, high signal. ⚠️ "Free for public repos" was
  imprecise: the published Free plan is metered, not repo-visibility-scoped — unlimited repos and
  developers but **1,000 scans/month, 3 team members**, and open-source projects are told to ask
  for a complimentary Team account (read from [socket.dev/pricing](https://socket.dev/pricing) on
  2026-08-14; vendor pricing is the fastest-rotting claim in this file — re-read the page before
  quoting a number to anyone).
- **Production install** — CI deploys with `pnpm install --prod` (skip devDeps, reduce attack surface)

## 13.8 Branch protection baseline

Two mechanisms exist. **Rulesets** (`Settings → Rules → Rulesets`) are the newer one; classic
branch protection (`Settings → Branches`) still works. GitHub's own docs stop short of calling
classic rules deprecated — the wording is that the two "work alongside each other, and all
applicable rules are enforced" — but rulesets are what the docs describe as offering "more
flexible ways to manage and understand protections": several can apply at once, each has an
enforcement status (including evaluate/dry-run), and anyone with read access can see them
(verified 2026-08-14 — [about rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)).
**Prefer a ruleset for a new repo**; don't rewrite a working classic rule just to migrate.

⚠️ Availability differs from classic protection: rulesets are available on **public** repos under
GitHub Free, and on public *and private* repos under Pro/Team/Enterprise Cloud. A private repo on
a free plan cannot use them. Limit is 75 rulesets per repo.

Whichever mechanism, the baseline is the same:
- ☑ Require PR before merging
- ☑ Require status checks: audit, lint, build, test
- ☑ Require branches up to date
- ☑ Require conversation resolution
- ☑ Require signed commits (optional; GPG or SSH signing)
- ☑ Restrict who can push (include admins if strict)
- ☑ Block force pushes

Both REST endpoints answered on 2026-08-14, so `gh` drives either one:
```bash
# Rulesets — read what is already there, then inspect one
gh api repos/OWNER/REPO/rulesets
gh api repos/OWNER/REPO/rulesets/RULESET_ID

# Classic branch protection — read (404 simply means "no protection set")
gh api repos/OWNER/REPO/branches/main/protection
```

Writing either config from the CLI means a `PUT`/`POST` with a full JSON body — there is no
`gh` shorthand flag for it, and a partial body **replaces** the existing settings rather than
merging. For an audit, read with the commands above and let the user apply changes in the UI;
that matches the plugin's propose-then-approve rule.

Solo maintainers can relax to "status checks only"; still use PRs for big changes.

## 13.9 Scanner tool matrix

⚠️ **The "free tier" column is the least durable thing in this file.** Vendors re-tier quietly and
none of them announce it in a changelog you can diff. Every row was read from the vendor's own
pricing or docs page on 2026-08-14; treat anything below as needing a re-read before it is quoted
to a user as a commitment.

| Tool | Free tier (checked 2026-08-14) | Scope | Delivery | Alive? |
|---|---|---|---|---|
| Dependabot | ✅ all repos | CVE alerts + auto-PR | GitHub native | current |
| Renovate | ✅ all repos | Deeper config + more ecosystems | GitHub App | current |
| `pnpm/yarn/npm audit` | ✅ | Node CVEs | CLI + CI | current |
| socket.dev | ⚠️ metered: 1,000 scans/mo, 3 members, unlimited repos; OSS can request a free Team account | Supply chain / malicious detection | GitHub App | current |
| Snyk | ⚠️ per-product monthly test counts, not one pool: **200** Open Source (SCA), **100** Code (SAST), **100** Container, **300** IaC | CVE + license + IaC | GitHub App + CLI | current |
| CodeQL | ✅ public repos; private needs paid Code Security | Static analysis | GitHub Actions | `codeql-action` `v4.37.9` |
| Gitleaks | ✅ CLI; ⚠️ the **Action** needs `GITLEAKS_LICENSE` on org-owned repos | Secret scan | Pre-commit + CI | CLI `v8.30.1`, 2026-03-21 |
| TruffleHog | ✅ open source | Secret scan (deeper entropy) | Pre-commit + CI | `v3.97.1`, 2026-08-24 |
| license-checker-rseidelsohn | ✅ | License audit | CLI + CI | `5.0.1`, 2026-05-27 |
| CycloneDX / Syft | ✅ | SBOM | CLI + release | `6.0.1` / `v1.51.1`, Aug 2026 |

The old table said Snyk gave "200 tests/mo" flat — that number is only the Open Source product's
allowance ([snyk.io/plans](https://snyk.io/plans/), read 2026-08-14). Release dates in the last
column come from `gh api repos/<owner>/<repo>/releases/latest` and the npm registry, re-run
2026-08-28; all four scanners are actively released, none are abandoned.

## 13.10 Day-1 baseline for any public repo

Ship these together on repo creation or first audit pass. ~30 min total, then autopilot:

1. `.github/dependabot.yml` — weekly grouped npm + actions + docker updates, with a `cooldown`
2. GitHub Settings → Security → Advanced Security: alerts + security updates + secret scanning +
   push protection (remember repository-level push protection is off by default) all ✓
3. `.github/workflows/audit.yml` — audit + outdated + Gitleaks on PR + weekly cron
4. Pre-commit hook: Gitleaks staged scan
5. Ruleset (or classic branch protection) on main: PR + status checks required
6. Lockfile committed; CI uses `--frozen-lockfile`
7. Pinned SHA on any third-party GitHub Action
8. (Optional) socket.dev GitHub App installed

## 13.11 Mode B rapid checklist

```bash
# 1. Dep-bot?
ls .github/dependabot.yml renovate.json 2>/dev/null

# 2. CI audit workflow? (-rl also catches .yaml, and survives zero matches)
grep -rlE "(audit|snyk|gitleaks|trufflehog)" .github/workflows/ 2>/dev/null || echo "MISSING: no audit workflow"

# 3. Lockfile committed?
ls pnpm-lock.yaml yarn.lock package-lock.json 2>/dev/null

# 4. Third-party actions NOT SHA-pinned? (lists offenders; silence = all pinned)
grep -rhoE "uses: [^ ]+@v[0-9][^ ]*" .github/workflows/ 2>/dev/null \
  | grep -vE "uses: (actions|github)/" | sort -u

# 5. Pending CVEs?
pnpm audit --audit-level=moderate 2>/dev/null || npm audit --audit-level=moderate 2>/dev/null

# 6. Outdated summary
pnpm outdated --long 2>/dev/null || npm outdated

# 7. License sweep
pnpm licenses list 2>/dev/null || npx license-checker-rseidelsohn --summary 2>/dev/null || true

# 8. Agent-ready signals (see 06-agent-ready.md)
curl -sI "https://DOMAIN/" | grep -iE "^link:" || echo "MISSING: Link headers"
curl -sI -H "Accept: text/markdown" "https://DOMAIN/" | grep -i "^content-type: text/markdown" || echo "MISSING: Markdown-for-Agents"
curl -sI "https://DOMAIN/.well-known/http-message-signatures-directory" | grep -i "^content-type: application/json" || echo "MISSING: Web Bot Auth JWKS"
```

> Check 4 was rewritten on 2026-08-14 because the previous regex — `uses: [^a-zA-Z].*@v[0-9]` —
> **matched nothing on any real workflow**: it required a non-letter immediately after `uses: `,
> while every action reference starts with an owner name. Confirmed by running it against a
> fixture containing two deliberately unpinned third-party actions; it exited 1 and printed
> nothing, i.e. it reported "all pinned" for a repo that was not. The replacement lists every
> tag-pinned action and filters out first-party `actions/*` and `github/*`, so anything printed
> is an actual finding. Beware the inverted reading of this one: **output means a problem**,
> silence means clean.

Roll output into severity gap report:
- 🔴 **Critical**: active CVE, secret detected, no branch protection
- 🟡 **Recommended**: no Dependabot/Renovate, no CI audit gate, unpinned third-party actions, license violation, no `Content-Signal`, no `Link:` headers
- 🟢 **Nice**: no SBOM, no socket.dev, no pre-commit secret scan, no Markdown-for-Agents, no Web Bot Auth JWKS
