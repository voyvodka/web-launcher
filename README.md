# web-launcher

A Claude Code plugin that works out why a live site is not showing up in Google, traces each
reason to the file or setting that causes it, fixes it, and verifies the fix.

```
C1  redirect targets resolve to 200 ............................. OK
C2  live robots.txt matches repo ............... FAIL  a CDN is injecting a managed block
C3  http://example.com/ -> 307 ................. FAIL  temporary; must be 301/308
C7  /index.html returns 200 at 1224 bytes (root: 72763) ... FAIL  SSR bypassed
      and: no canonical
      and: indexable (no noindex)
```

Every line above is a command that ran, not a rule that was read.

## Why

Search Console tells you *what* — "Page with redirect ×37", "Not found (404) ×3" — and stops
there. Turning that into a change in a file is the whole job, and it is where most tooling leaves
you: a score, a checklist, a list of symptoms.

Two things make that harder than it looks.

**Most of those counts are not defects.** A site with 40 canonical URLs whose slashless twins all
return a permanent redirect will show ~39 "Page with redirect" entries, and every one of them is
Google correctly consolidating. Chasing them changes a working site. This plugin does the
arithmetic first and is willing to conclude that nothing is wrong.

**Written rules drift from live behaviour.** A config comment saying "no 2-hop chain" was wrong
for months in the codebase this plugin was built against, because nothing tested it. So the
checks here are runnable commands with one-line verdicts, not prose.

## Install

```
/plugin marketplace add voyvodka/claude-plugins
/plugin install web-launcher@voyvodka
```

Requires Claude Code. The diagnostic checks need `curl`, `grep` and a POSIX shell — present on
macOS and Linux by default. GEO scoring uses the [GeoDaddy](https://geodaddy.dev/) MCP server,
declared by the plugin and fetched via `npx` on first use; Search Console access needs a one-time
Google OAuth setup, and both are optional.

## Usage

Invoke it in a site's repository:

```
/web-launcher
```

It picks a mode — launch, audit, or brand-only — scans the project, and produces a
severity-ranked gap report before changing anything. For a live site it runs the diagnostic checks
first and reports what they returned.

Ask it directly when you already know the question:

> why isn't this page indexed?
> score this site for AI search
> set up Cloudflare deployment with the apex redirect

## What it covers

| Area | What it does |
|---|---|
| **Diagnosis** | Maps each Search Console reason to its cause; ten runnable checks for redirect chains, edge-injected `robots.txt`, temporary redirects, hop counts, sitemap health, canonical mismatches, SSR shell leaks, header delivery, and social cards |
| **Discoverability** | `robots.txt`, sitemaps, `llms.txt`, `security.txt`, JSON-LD, the full meta head, multi-page canonical and breadcrumb structure |
| **GEO** | Scores AI-search signals with GeoDaddy, and says where that tool's scoring disagrees with this one — including one finding of its own that is measurably wrong |
| **Deployment** | Cloudflare Workers: `wrangler` config, headers, redirects, custom domain and SSL, plus the failure modes that cost real hours |
| **Hardening** | Dependabot or Renovate, CI audit gates, secret scanning, SBOM, licence sweep, pinned actions |

## How it works

**It reads, then proposes, then asks.** Diagnosis is entirely read-only: HTTP requests to the
public site and reads of the repository. Nothing is written until you approve a plan.

**Search Console access is read-only apart from submitting a sitemap.** The API has no
request-indexing and no start-validation method, so the plugin says what it changed and hands you
the interface link rather than pretending it triggered a recrawl. Credentials stay on your machine.

**Every technical claim carries the date it was verified and the source it came from.** Reference
files are stamped at three levels — verified, partially verified, not re-verified — and the plugin
puts itself on a 60-day review clock. An unstamped claim is unverified, and it says so rather than
sounding certain.

## Limitations

- **Cloudflare only, for deployment.** Diagnosis is platform-agnostic and runs over live HTTP, but
  scaffolding and deploys target Cloudflare Workers. Knowing one platform properly beat knowing
  four halfway.
- **No content, no analytics, no consent flows.** It builds the technical scaffolding; the words
  and the tracking are someone else's job.
- **No deep application SEO** — hreflang for multi-region sites, e-commerce product feeds, and
  logged-in single-page apps are out of scope.
- **"Crawled – currently not indexed" cannot be closed technically.** It is a content and
  authority question. The plugin names it and does not pretend otherwise.
- **Search Console's coverage report cannot be downloaded.** No API exposes it, so the plugin
  reconstructs a candidate URL set and inspects it — the numbers will approximate the interface,
  not match it, and it says which URLs it could not reach.

## Contributing

Issues and pull requests are welcome, particularly measurements: a check that misses a real defect,
or one that fires on a healthy site, is the most useful thing you can report. Include the command
and its output.

A claim without a source will not be merged — that rule is the product, not a preference.

## License

MIT — see [LICENSE](LICENSE).
