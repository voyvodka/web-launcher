# Agent-Ready discoverability (2025–2026 standards)

*Loaded by `web-launcher` SKILL when user mentions `isitagentready.com`, agentic web, MCP-facing sites, or wants programmatic agent discovery.*

> **Verified 2026-08-14 · review by 2026-10-13.** Claims that can rot carry their own date and
> source inline. A claim without one has not been checked — treat it as unverified, not as fact.
>
> This file changed substantially in that audit: two of its four headline signals turned out to
> have **no measurable effect today**. Read the priority matrix before recommending anything here.

Parallel to classic SEO and GEO, a new discovery layer is forming: **AI agents that programmatically consume your site** (ChatGPT tool use, Perplexity retrieval, Claude MCP servers, custom enterprise bots). Cloudflare's 2025 `isitagentready.com` scanner encodes four signals agents look for — different surface from classic crawlers.

**Hosted readiness check**: `https://isitagentready.com/?url=YOUR_DOMAIN`

## 4 signals (in priority order — ship the top two always)

### 1. `Link:` response headers (RFC 8288)

Agents prefer machine-discoverable entrypoints over scraping HTML. Emit `Link` on homepage / key routes using IANA-registered `rel` types (https://www.iana.org/assignments/link-relations/):

```http
Link: </llms.txt>; rel="alternate"; type="text/markdown"
Link: </sitemap.xml>; rel="sitemap"
Link: </.well-known/security.txt>; rel="security-txt"
Link: </api/catalog.json>; rel="api-catalog"
Link: </docs/api>; rel="service-doc"
Link: </openapi.json>; rel="service-desc"
```

**Cloudflare Pages / Workers `_headers`** (simplest):
```
/*
  Link: </llms.txt>; rel="alternate"; type="text/markdown", </sitemap.xml>; rel="sitemap", </.well-known/security.txt>; rel="security-txt"
```

**Express / Node middleware**:
```ts
app.use((_req, res, next) => {
  res.setHeader("Link", [
    `</llms.txt>; rel="alternate"; type="text/markdown"`,
    `</sitemap.xml>; rel="sitemap"`,
    `</.well-known/security.txt>; rel="security-txt"`,
  ].join(", "));
  next();
});
```

**Audit**: `curl -sI https://DOMAIN/ | grep -i "^link:"`

### 2. `Content-Signal` directive in `robots.txt`

> **Reality check, verified 2026-08-14.** No crawler is known to act on this. Google's John Mueller,
> July 2026: *"AFAIK none of the crawlers / llms use the content-signal robots.txt directives"* and
> *"It was made up by a CDN, afaik it has no effects whatsoever for any crawler or llm."* ⚠️ Sourced
> from secondary write-ups quoting him ([1](https://searchengineoptimization.blog/article/google-says-cloudflares-content-signals-tag-does-nothing),
> [2](https://www.tryvizup.com/blog/cloudflare-content-signals-what-google-actually-said)); the
> original thread could not be reached directly.
>
> Cloudflare stamping this on millions of domains via managed robots.txt is **publisher-side**
> adoption, not crawler-side. Those are different claims and this file used to conflate them.
> An earlier version also called it "IANA-compatible" — that could not be sourced against any IANA
> registry and has been removed.
>
> **Ship it as a statement of intent, not as a control.** It is free and may matter for
> copyright posture. It changes no crawler's behaviour today. Never present it to a user as
> protection.

Cloudflare 2025 robots.txt extension. Declares preference for how AI systems may use your content — independent of `Allow`/`Disallow`. **Not a hard block; a policy signal.**

**Watch for edge injection.** Cloudflare's managed robots.txt / AI Crawl Control writes its own
`Content-Signal` and `Disallow` block into the served file, which can invert what the repository
declares. Always diff live against repo — see `11-validation-toolkit.md`.

```
Content-Signal: search=yes, ai-train=yes, ai-input=yes
```

Three axes:
- **`search`** — classic crawler indexing (Googlebot, Bingbot)
- **`ai-train`** — model training datasets (CCBot, GPTBot training crawls)
- **`ai-input`** — real-time AI assistant retrieval (ChatGPT-User, Claude-User)

**Decision matrix — pick row matching site type**:

| Content type | `search` | `ai-train` | `ai-input` |
|---|---|---|---|
| Open marketing / portfolio / docs / OSS | yes | yes | yes |
| Paid journalism / proprietary research | yes | **no** | yes |
| Premium education (course content) | yes | **no** | yes |
| Legal / medical (attribution-critical) | yes | no | **no** |
| Staging / draft / internal | **no** | **no** | **no** |

For most marketing / portfolio / OSS sites: **`yes, yes, yes`** — maximum LLM citation visibility.

**Audit**: `curl -s https://DOMAIN/robots.txt | grep -i "^content-signal:"`

#### `Content-Usage` — the IETF AIPREF successor (emit alongside `Content-Signal`)

`Content-Signal` was Cloudflare's 2025 proposal. The **IETF AI Preferences WG (AIPREF)** put the same idea on the standards track as **`Content-Usage`**.

> **Status, verified 2026-08-14 — worse than this file previously claimed.** The draft that
> actually defines the robots.txt field and the HTTP header, `draft-ietf-aipref-attach-04`, is
> **expired**: *"This Internet-Draft is no longer active."* Last revision 2025-10-28, nothing since
> ([Datatracker](https://datatracker.ietf.org/doc/draft-ietf-aipref-attach/)). The vocabulary draft
> `draft-ietf-aipref-vocab-06` is still active (2026-04-27), and the WG milestones for both are
> 2026-08-31, still open ([WG charter](https://datatracker.ietf.org/wg/aipref/about/)). **No RFC
> has been published by this working group.**
>
> Consequence: the field name and syntax can still change. Emitting `Content-Usage` today is
> harmless — nothing parses it — but calling it "standards-track, forward-compatible" oversells it.
> Wait for the attach draft to revive before recommending it to anyone.

Two delivery surfaces, same vocabulary:

```
# robots.txt rule (optional path-prefix, longest-prefix match like Allow/Disallow)
Content-Usage: train-ai=n
Content-Usage: /ai-ok/ train-ai=y
```
```http
# HTTP response header (RFC 9651 structured-field dictionary)
Content-Usage: train-ai=n
```

**Vocabulary** (`draft-ietf-aipref-vocab`, Proposed Standard track) currently defines **only two** categories, each `y`/`n`:

| Category | Meaning |
|---|---|
| `search` | indexing assets to direct users to them (classic search) |
| `train-ai` | using assets to produce/refine a generative AI model |

There is **no** category yet for real-time RAG / answer-engine input — so the `ai-input` axis you express in Cloudflare `Content-Signal` has **no AIPREF equivalent**; keep it on `Content-Signal` until AIPREF adds one. Map the common open-content policy `search=yes, ai-train=no, ai-input=yes` to `Content-Usage: search=y, train-ai=n`.

**Caveat**: AIPREF drafts are not yet a ratified RFC (they expire-and-revise between WG revisions). Emit `Content-Usage` as an early-adopter, forward-compatible signal — do **not** drop `Content-Signal` for it yet.

**Audit**: `curl -s https://DOMAIN/robots.txt | grep -i "^content-usage:"`

### 3. Markdown for Agents (`Accept: text/markdown` content negotiation)

Agents that parse content prefer clean markdown over HTML — fewer tokens, no CSS/JS noise, stable structure. On `Accept: text/markdown` requests return markdown; keep HTML default for browsers.

**Cloudflare Pages / Workers**: a platform toggle handles negotiation. ⚠️ Reported to be gated to
paid plans rather than available on the free tier — not confirmed against Cloudflare's own pricing
page (2026-08-14). Check the dashboard before promising it; if the toggle is absent, use the
self-hosted middleware below or emit static `.md` twins per route.

**Self-hosted (Express / Node)**:
```ts
app.use(async (req, res, next) => {
  const accept = req.headers.accept ?? "";
  const wantsMd = accept.includes("text/markdown") && !accept.includes("text/html");
  if (!wantsMd) return next();

  const md = await renderMarkdownFor(req.path);
  res.setHeader("Content-Type", "text/markdown; charset=utf-8");
  res.setHeader("X-Markdown-Tokens", "available");
  res.send(md);
});
```

Per-route markdown strategy:
- `/` → content of `llms.txt` (or richer home brief)
- `/blog/:slug`, `/docs/:slug` → raw markdown source
- `/projects/:slug` → README + metadata concatenated
- Unknown route → fall back to `llms.txt`

**Audit**:
```bash
curl -sI -H "Accept: text/markdown" https://DOMAIN/ | grep -i content-type
# Expect: text/markdown; charset=utf-8 (not text/html)
```

**Skip when**: strictly single-page site (llms.txt alone is sufficient) or authenticated app (agents don't crawl auth).

### 4. Web Bot Auth — `/.well-known/http-message-signatures-directory`

HTTP Message Signatures (RFC 9421) lets your site cryptographically identify *itself* on outbound requests. Publishes a JWKS (JSON Web Key Set) that receivers use to verify signed requests.

**Apply when** your site makes outbound requests (APIs, webhooks, server-to-server) and wants receivers to distinctly whitelist or rate-limit you.

**Skip when** your site only receives traffic — but still ship the placeholder (passes `isitagentready.com`, costs nothing).

**On the empty placeholder.** Serving `{"keys": []}` passes the scanner and signs nothing. It
declares a capability the site does not have, and a receiver fetching it learns only that there
are no keys. Recommend it only when the user's explicit goal is the scanner score — never as a
security or identity measure, and say which one you are doing.

```json
{
  "keys": []
}
```
Serve at `/.well-known/http-message-signatures-directory` with `Content-Type: application/json`.

**Full implementation**: generate Ed25519 keypair, publish public JWK here, sign outbound with `Signature-Input` + `Signature` headers per RFC 9421. Cloudflare `signed-agent` helpers exist.

**Audit**: `curl -sI https://DOMAIN/.well-known/http-message-signatures-directory | grep -i content-type` → expect `application/json`, not HTML SSR fallback.

## Priority matrix

| Signal | Effort | Risk | Real effect today | Skip if |
|---|---|---|---|---|
| `Link:` response headers | 15 min | Low | Yes — RFC 8288, universally parsed | Never |
| Markdown-for-Agents | 30 min – 2 h | Low–Med | Yes, for agent consumers | Single-page or auth-only site |
| `Content-Signal` in robots.txt | 5 min | 0 | **None known** — policy statement only | Fine to skip; add if stating intent matters |
| `Content-Usage` (AIPREF) | 5 min | 0 | **None** — defining draft expired | Skip until the attach draft revives |
| Web Bot Auth JWKS placeholder | 10 min | Low | **None** — empty key set signs nothing | Skip unless chasing the scanner score |
| Web Bot Auth real signing | 2 h | Low | Yes, for outbound identity | No outbound requests |

**Link headers first.** They are the only item here that is both ratified and actually parsed.
Markdown-for-Agents is a real content-delivery investment. The rest are declarations: cheap,
harmless, and currently inert — ship them if the user wants the scanner green or wants to state
intent, and say which of the two you are doing.

## Standards declaration — verified 2026-08-14

| Signal | Standing | Source |
|---|---|---|
| **RFC 8288** — Link headers | Stable IETF standard since 2017 | Published RFC |
| **RFC 9421** — HTTP Message Signatures | Stable IETF standard since 2024 | Published RFC |
| **`Content-Signal`** | Cloudflare convention, **not IETF**, no crawler known to honour it | See §2 above |
| **`Content-Usage`** (AIPREF) | Vocabulary draft active; **the attach draft that defines the syntax is expired**; no RFC published by the WG | [Datatracker](https://datatracker.ietf.org/wg/aipref/documents/) |
| **`Accept: text/markdown`** negotiation | Emerging convention, no ratification | — |

Explain this honestly if a user asks. Two of these are standards; the rest are conventions, one of
them from a single vendor and one resting on an expired draft. Saying so costs nothing and is the
difference between advice and salesmanship.
