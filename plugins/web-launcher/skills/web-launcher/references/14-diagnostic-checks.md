# Diagnostic checks — runnable, one verdict line each

*Loaded by `web-launcher` SKILL for any Mode B audit, and whenever a Search Console reason class
needs to be traced to a cause. Read this before writing a gap report about a live site.*

> **Verified 2026-08-14 · review by 2026-10-13.** Every check below was executed against two live
> production sites on that date and each one either caught a real defect or cleared a suspected
> one. They are measurements, not lint rules.

## Why this file exists

Every defect found in the audit that produced this file had the same shape: **the rule was written
down in the repository, and nothing verified that it held.**

| Defect | What the repo claimed | What was missing |
|---|---|---|
| Legacy aliases resolved in two hops | A config comment saying "no 2-hop chain" | Measuring the chain on the live site |
| CDN injected its own `robots.txt` block, reversing the stated AI policy | `public/robots.txt` declared the opposite | A diff of live against repo |
| apex→www used a temporary redirect | Nothing — it was never stated | Auditing the redirect *code*, not just that it redirected |
| `/index.html` served a thin SSR-less shell | `index: false` was assumed to prevent it | Fetching `/index.html` and looking at what came back |
| `/projects/*.html` produced 301→404 chains | Nothing | Verifying the redirect *target* returns 200 |

A comment is an intention. A check is a fact. **When a comment claims live behaviour — "single
hop", "no duplicates", "returns 404" — the check that proves it belongs in the same change.**

## How to use these

All read-only. Set `BASE` to the canonical origin, with no trailing slash:

```bash
BASE="https://example.com"
```

Run them in the order below when diagnosing "why is this page not indexed": C1 and C7 explain most
404 and redirect reasons, C3 and C4 explain most signal-consolidation problems.

Every check prints `OK` or `FAIL` with the offending URL. Nothing prints on success except the
`OK` line — silence means the check did not run.

---

## C1 — Redirect targets must return 200 (301→404 chain hunter)

**Explains:** Search Console "Not found (404)" *and* "Page with redirect" appearing together. The
highest-value check in this file — a redirect issued before the target's existence is verified
produces exactly this pair.

```bash
check_redirect_targets() {
  local base="$1"; shift
  local fail=0 p first final
  for p in "$@"; do
    first=$(curl -sI --max-time 15 -o /dev/null -w '%{http_code}' "$base$p")
    final=$(curl -sIL --max-time 20 -o /dev/null -w '%{http_code}' "$base$p")
    if [ "$first" -ge 300 ] && [ "$first" -lt 400 ] && [ "$final" -ge 400 ]; then
      echo "FAIL $p  $first -> final $final (redirect leads to an error)"
      fail=1
    fi
  done
  [ "$fail" -eq 0 ] && echo "OK   redirect targets resolve to 200"
  return $fail
}

# Probe the shapes a router is most likely to mishandle: extensions, dots, casing.
check_redirect_targets "$BASE" \
  /index.html /foo.html /foo.php /a.b.c /nonexistent-xyz
```

**Root cause to look for in code:** a normalization redirect issued *before* the lookup that
decides whether the destination exists. The fix is ordering — resolve first, redirect only to
something that resolves.

## C2 — Live `robots.txt` and `llms.txt` must match the repository (edge-injection hunter)

**Explains:** a stated crawler policy that is not the one being served. CDNs with "managed
robots.txt" features prepend their own block, and their block can invert what the repository
declares.

```bash
diff <(curl -s --max-time 20 "$BASE/robots.txt") ./public/robots.txt \
  && echo "OK   robots.txt matches repo" \
  || echo "FAIL live robots.txt differs from repo — edge injection or stale deploy"

curl -s --max-time 20 "$BASE/robots.txt" | grep -qi 'managed content' \
  && echo "FAIL a CDN is injecting a managed robots.txt block"
```

Adjust the repo path per framework (`public/`, `static/`, `assets/`). Run the same diff for
`llms.txt` and `security.txt`.

**Cannot be fixed in code.** Turn the managed feature off at the CDN dashboard, or accept its
policy and align the repo to it — but never leave two conflicting declarations live.

## C3 — Host and scheme consolidation must be permanent

**Explains:** split authority, and "Page with redirect" that never consolidates. `301`/`308` pass
signals to the target; `302`/`307` explicitly do not.

```bash
host="${BASE#https://}"; apex="${host#www.}"
for u in "http://$apex/" "https://$apex/" "http://www.$apex/" "https://www.$apex/"; do
  code=$(curl -sI --max-time 15 -o /dev/null -w '%{http_code}' "$u")
  case "$code" in
    200)     echo "OK   $u -> 200 (canonical host)" ;;
    301|308) echo "OK   $u -> $code (permanent)" ;;
    302|307) echo "FAIL $u -> $code (temporary; must be 301/308 for a permanent move)" ;;
    *)       echo "?    $u -> $code" ;;
  esac
done
```

## C4 — No more than one hop, counting meta-refresh

**Explains:** crawl budget waste and "Page with redirect" on URLs that do eventually resolve.
`curl -L` does **not** count meta-refresh, so an HTTP hop followed by a meta-refresh reads as one
hop while actually being two. Framework-level `redirects` maps often emit exactly this shape.

```bash
hop_count() {
  local u="$1" hops final
  hops=$(curl -sIL --max-time 20 -o /dev/null -w '%{num_redirects}' "$u")
  final=$(curl -sIL --max-time 20 -o /dev/null -w '%{url_effective}' "$u")
  if curl -s --max-time 15 "$final" | grep -qi 'http-equiv="refresh"'; then
    hops=$((hops + 1))
    echo "FAIL $u -> $hops hops (last step is a meta-refresh: $final)"
  elif [ "$hops" -gt 1 ]; then
    echo "FAIL $u -> $hops hops"
  else
    echo "OK   $u -> $hops hop"
  fi
}

# Feed it every legacy alias and every entry in the framework's redirects map.
hop_count "$BASE/old-path"
```

**Fix:** move aliases from the framework's HTML-emitting redirect map to platform-level rules
(`_redirects`, edge redirect rules) so they resolve in a single HTTP hop.

## C5 — Every sitemap URL must return 200

**Explains:** "Page with redirect" and "Not found (404)" on URLs the site itself submitted. A
sitemap listing a redirecting URL is the site contradicting its own canonical choice.

```bash
sitemap_all_200() {
  local sm="$1" fail=0
  while read -r u; do
    code=$(curl -sI --max-time 15 -o /dev/null -w '%{http_code}' "$u")
    [ "$code" = "200" ] || { echo "FAIL $u -> $code (listed in sitemap, not 200)"; fail=1; }
  done < <(curl -s --max-time 30 "$sm" | grep -o '<loc>[^<]*</loc>' | sed -e 's|<loc>||g' -e 's|</loc>||g')
  [ "$fail" -eq 0 ] && echo "OK   all sitemap URLs return 200"
}

# Resolve the real sitemap first — robots.txt names it, and it is often not /sitemap.xml.
curl -s "$BASE/robots.txt" | grep -i '^sitemap:'
sitemap_all_200 "$BASE/sitemap.xml"
```

Where a sitemap index is used, run this against each child sitemap.

## C6 — Canonical must equal the URL that served it

**Explains:** "Alternate page with proper canonical tag" and "Google chose a different canonical".
A trailing-slash mismatch between the served URL and its canonical is the usual cause.

```bash
canonical_matches() {
  local u c
  for u in "$@"; do
    c=$(curl -s --max-time 15 "$u" \
      | grep -oiE '<link rel="canonical"[^>]*>' | head -1 \
      | grep -oE 'href="[^"]*"' | sed 's/href="//;s/"//')
    if   [ -z "$c" ];    then echo "FAIL $u -> no canonical"
    elif [ "$u" != "$c" ]; then echo "FAIL $u -> canonical=$c (differs)"
    else echo "OK   $u"
    fi
  done
}

curl -s "$BASE/sitemap.xml" | grep -o '<loc>[^<]*</loc>' \
  | sed -e 's|<loc>||g' -e 's|</loc>||g' | while read -r u; do canonical_matches "$u"; done
```

## C7 — Shell leak: an SSR-less thin copy served at a real URL

**Explains:** duplicate content and soft-404 candidates. On an SSR site, `/index.html` may bypass
the render path entirely and serve the raw client shell — same `<title>` as the homepage, empty
body, frequently no canonical and `index,follow`.

```bash
shell_leak_check() {
  local base="$1" root_size idx_code idx_size body
  root_size=$(curl -s --max-time 20 "$base/" | wc -c)
  idx_code=$(curl -sI --max-time 15 -o /dev/null -w '%{http_code}' "$base/index.html")
  idx_size=$(curl -s --max-time 20 "$base/index.html" | wc -c)
  if [ "$idx_code" = "200" ] && [ "$idx_size" -lt $((root_size / 4)) ]; then
    echo "FAIL $base/index.html returns 200 at $idx_size bytes (root: $root_size) — SSR bypassed"
    body=$(curl -s --max-time 15 "$base/index.html")
    echo "$body" | grep -qi 'rel="canonical"' || echo "     and: no canonical"
    echo "$body" | grep -qi 'noindex'         || echo "     and: indexable (no noindex)"
  else
    echo "OK   no shell leak at /index.html"
  fi
}
shell_leak_check "$BASE"
```

**Root cause to look for in code:** a static-file middleware mounted ahead of the SSR handler.
Disabling directory-index resolution does not stop a direct request for the file itself.

## C8 — Declared headers must actually be served

**Explains:** security and caching headers that exist in `_headers` but never reach a client —
usually because the file is in the wrong directory, or a rule earlier in the chain wins.

```bash
assert_header() {
  local url="$1" name="$2" expect="$3" got
  got=$(curl -sI --max-time 15 "$url" | grep -i "^$name:" | head -1)
  case "$got" in
    *"$expect"*) echo "OK   $name @ $url" ;;
    "")          echo "FAIL $name header absent @ $url" ;;
    *)           echo "FAIL $name is not '$expect': $got" ;;
  esac
}
assert_header "$BASE/" "strict-transport-security" "max-age="
assert_header "$BASE/" "link" 'rel="sitemap"'
```

Assert one line per declaration in `_headers`. A declaration nobody checks is a comment.

## C9 — `og:image` must exist and resolve

**Explains:** social cards that render blank. Also catches SVG cards, which the major platforms do
not render.

```bash
curl -s "$BASE/sitemap.xml" | grep -o '<loc>[^<]*</loc>' \
  | sed -e 's|<loc>||g' -e 's|</loc>||g' | while read -r u; do
      og=$(curl -s --max-time 15 "$u" | grep -oiE '<meta property="og:image" content="[^"]*"' \
           | sed 's/.*content="//;s/"//')
      [ -z "$og" ] && { echo "FAIL $u -> no og:image"; continue; }
      case "$og" in *.svg) echo "FAIL $u -> og:image is SVG (not rendered by social platforms)";; esac
      code=$(curl -sI --max-time 15 -o /dev/null -w '%{http_code}' "$og")
      [ "$code" = "200" ] || echo "FAIL $u -> og:image $og => $code"
    done
```

## C10 — Find behavioural claims that nothing verifies (meta-check)

The most generalizable check here. In the audit that produced this file, a config comment asserting
"no 2-hop chain" was **wrong, and had been wrong for months**, because nothing tested it.

```bash
grep -rnE '(no|zero|avoids?|prevents?|without) .*(2-hop|double.hop|redirect chain|duplicate|404)' \
  --include='*.mjs' --include='*.ts' --include='*.js' --include='*.json' . \
  | grep -v node_modules
```

Each hit is a claim about live behaviour. Either it has a check in this file, or write one, or
delete the comment. An unverified claim in a config file is worse than no comment — it stops the
next person from looking.

---

## Reporting

Report per finding: **the check that failed, the URL, the code path or dashboard setting that
causes it, and the fix.** A finding without a location is a symptom, not a diagnosis.

Then re-run the same check after deploying and paste both outputs. The before/after pair is the
only evidence that something was actually closed — see `09-audit-workflow.md` for the report shape.
