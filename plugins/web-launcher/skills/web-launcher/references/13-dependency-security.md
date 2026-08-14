# Dependency & supply-chain health

*Loaded by `web-launcher` SKILL for repo hardening phase, or when user asks about dep-bot / security / audit / SBOM / license compliance.*

Static/marketing sites still ship third-party code (framework, build tools, wrangler, satori, resvg, font packages). Dependencies rot: CVEs appear, breaking changes land, transitive deps get yanked. Automate this — manual review doesn't scale.

## What to audit in any repo

- `.github/dependabot.yml` or `renovate.json` — automated dep updates
- `.github/workflows/*audit*.yml` — CI audit gate
- GitHub repo Settings → Code security toggles (user confirms in UI)
- Branch protection on `main` — PR + status checks required
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

## 13.2 GitHub repo-level security toggles (user action)

Enable in `Settings → Code security`:
- ✅ **Dependabot alerts** — CVE notifications
- ✅ **Dependabot security updates** — auto-PR on alert
- ✅ **Secret scanning** (free on public, paid on private)
- ✅ **Secret scanning push protection** — blocks commits with secrets at push time
- ✅ **Code scanning** (CodeQL) — static analysis (optional; adds CI time)
- ✅ **Private vulnerability reporting** — lets researchers file via `security/advisories`

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
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm audit --audit-level=moderate
      - run: pnpm outdated --long || true

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

Package manager equivalents:
| Manager | Audit | Outdated |
|---|---|---|
| pnpm | `pnpm audit --audit-level=moderate` | `pnpm outdated --long` |
| yarn (berry) | `yarn npm audit --severity moderate` | `yarn outdated` |
| npm | `npm audit --audit-level=moderate` | `npm outdated` |

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
    rev: v8.18.0
    hooks: [{ id: gitleaks }]
```

**If a secret was committed — order of operations**:
1. **Rotate the credential** at the provider (revoke, regenerate) FIRST
2. Then scrub history: `git filter-repo --path <file> --invert-paths` (or BFG Repo Cleaner)
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
> extra install step.

Attach to GitHub Release artifacts.

## 13.7 Supply-chain hardening

- **npm provenance** — `npm publish --provenance` attests package built from claimed source repo at claimed commit (Sigstore-backed, free)
- **Lockfile discipline** — always commit, always `--frozen-lockfile` / `--immutable` in CI (fails if lockfile drifts)
- **Pinned third-party Actions** — use commit SHA, not floating tags:
  ```yaml
  # Good
  uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
  # Risky
  uses: some-user/some-action@v1
  ```
  First-party (actions/*, github/*) can stay on tags; third-party pin SHA.
- **socket.dev** (free for public repos) — GitHub App reviews dep-change PRs for supply-chain red flags (install scripts, shell exec, network calls, typosquat). Zero config, high signal.
- **Production install** — CI deploys with `pnpm install --prod` (skip devDeps, reduce attack surface)

## 13.8 Branch protection baseline

`Settings → Branches → Add rule for main`:
- ☑ Require PR before merging
- ☑ Require status checks: audit, lint, build, test
- ☑ Require branches up to date
- ☑ Require conversation resolution
- ☑ Require signed commits (optional; GPG or SSH signing)
- ☑ Restrict who can push (include admins if strict)
- ☑ Disallow force pushes

Solo maintainers can relax to "status checks only"; still use PRs for big changes.

## 13.9 Scanner tool matrix

| Tool | Free tier | Scope | Delivery |
|---|---|---|---|
| Dependabot | ✅ all repos | CVE alerts + auto-PR | GitHub native |
| Renovate | ✅ all repos | Deeper config + more ecosystems | GitHub App |
| `pnpm/yarn/npm audit` | ✅ | Node CVEs | CLI + CI |
| socket.dev | ✅ public | Supply chain / malicious detection | GitHub App |
| Snyk | 200 tests/mo | CVE + license + IaC | GitHub App + CLI |
| CodeQL | ✅ public | Static analysis | GitHub Actions |
| Gitleaks | ✅ | Secret scan | Pre-commit + CI |
| TruffleHog | ✅ | Secret scan (deeper entropy) | Pre-commit + CI |
| license-checker-rseidelsohn | ✅ | License audit | CLI + CI |
| CycloneDX / Syft | ✅ | SBOM | CLI + release |

## 13.10 Day-1 baseline for any public repo

Ship these together on repo creation or first audit pass. ~30 min total, then autopilot:

1. `.github/dependabot.yml` — weekly grouped npm + actions + docker updates
2. GitHub Settings → Code security: alerts + security updates + secret scanning + push protection all ✓
3. `.github/workflows/audit.yml` — audit + outdated + Gitleaks on PR + weekly cron
4. Pre-commit hook: Gitleaks staged scan
5. Branch protection on main: PR + status checks required
6. Lockfile committed; CI uses `--frozen-lockfile`
7. Pinned SHA on any third-party GitHub Action
8. (Optional) socket.dev GitHub App installed

## 13.11 Mode B rapid checklist

```bash
# 1. Dep-bot?
ls .github/dependabot.yml renovate.json 2>/dev/null

# 2. CI audit workflow?
ls .github/workflows/*.yml 2>/dev/null | xargs grep -l -E "(audit|snyk|gitleaks|trufflehog)" 2>/dev/null

# 3. Lockfile committed?
ls pnpm-lock.yaml yarn.lock package-lock.json 2>/dev/null

# 4. Third-party actions SHA-pinned?
grep -rE "uses: [^a-zA-Z].*@v[0-9]" .github/workflows/ 2>/dev/null

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

Roll output into severity gap report:
- 🔴 **Critical**: active CVE, secret detected, no branch protection
- 🟡 **Recommended**: no Dependabot/Renovate, no CI audit gate, unpinned third-party actions, license violation, no `Content-Signal`, no `Link:` headers
- 🟢 **Nice**: no SBOM, no socket.dev, no pre-commit secret scan, no Markdown-for-Agents, no Web Bot Auth JWKS
