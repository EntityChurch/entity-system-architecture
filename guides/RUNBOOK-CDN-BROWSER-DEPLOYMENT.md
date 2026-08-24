# RUNBOOK — CDN Browser Deployment (CORS / CSP / HTTPS / cache)

**Status:** v1 deliverable (promoted from "deferred to guide" per egui-rust review cgid-10-218, absorption §2.1). The browser is the headline consumer of the static-HTTP CDN path; its deployment headers cannot be a forward-reference.
**Date:** cgid-10-218
**Author:** entity-core-architecture
**Owner:** W3 (deployment); arch authors the contract, egui-rust validates in real browsers.
**Scope:** what a CDN origin serving an entity peer's tree+content MUST send so a browser/WASM consumer can fetch+verify. Covers both deployment shapes. Native (Go/CLI) consumers can ignore everything here — CORS/CSP/cache are browser-only concerns.

---

## §0 Why this exists

BRIDGE-HTTP §5 correctly classifies CORS / CSP / HTTPS as deployment config, not spec body. For a **native** consumer that's right. For a **browser** consumer it's the difference between working and a blank screen. The corridor-close impl-evidence (`python3 -m http.server`) is native and sends **no** CORS headers — so the cited demo cannot replicate in a browser. This runbook closes that gap with worked headers for the two deployment shapes, and the conformance vector (§4) proves the browser-success path, not just graceful failure.

**The load-bearing fact:** hash-verify (BRIDGE-HTTP §2) defeats *forgery* — a tampered body fails the hash. It does **not** defeat *staleness* — a stale-but-valid manifest still hash-verifies. So mutable-endpoint cache discipline (§3) is a separate, required concern; hash-verify is not a substitute.

---

## §1 The two deployment shapes

| Shape | CDN origin vs app origin | CORS | Cache risk |
|---|---|---|---|
| **Cross-origin** | `cdn.example.com` ≠ app origin | **applies** — origin MUST send `Access-Control-Allow-Origin` | service worker bypasses cross-origin → no staleness flag |
| **Same-origin** | CDN served from app origin | none — no CORS preflight | **mutable-endpoint staleness** (§3) bites; SW intercepts same-origin GET cache-first |

**You don't get to dodge both at once.** Cross-origin needs CORS headers; same-origin needs cache discipline. The runbook covers both; a deployment picks one shape and applies that column.

---

## §2 Cross-origin CDN — CORS + CSP

### §2.1 Required response headers on the CDN origin

Every response from the CDN origin a browser peer fetches (tree leaves, content blobs, manifest):

```
Access-Control-Allow-Origin: https://app.example.com    ; or * for fully-public content
Access-Control-Allow-Methods: GET, HEAD
Access-Control-Allow-Headers: Range                     ; if range requests used
```

- For fully-public, fully-content-addressed deployments, `Access-Control-Allow-Origin: *` is acceptable (content is hash-verified; no credentials, no cookies — there is nothing CORS protects here that the hash doesn't).
- For app-scoped deployments, name the app origin explicitly.
- No `Access-Control-Allow-Credentials` — the entity CDN path is credential-free (auth is in the signed entity/cap layer, not HTTP cookies).

### §2.2 App-side CSP (if the app ships one)

The entity-WASM app today ships **no** CSP (confirmed by egui-rust against `index.html`), so it is permissive by default. ANY future hardening pass MUST include the CDN origin in `connect-src`:

```
Content-Security-Policy: connect-src 'self' https://cdn.example.com;
```

Omitting the CDN origin from `connect-src` blocks the fetch at the page level before CORS even applies — a silent blank-screen failure. If the app adds a CSP, this line is mandatory.

### §2.3 HTTPS

Browser `fetch` from a `https://` page to an `http://` origin is mixed-content-blocked. CDN origin MUST be HTTPS when the app is served over HTTPS (the normal case). Same-origin inherits the app's scheme.

---

## §3 Same-origin CDN — mutable-endpoint cache discipline

When the CDN is served same-origin, no CORS — but the consumer's caching layers (service workers, HTTP cache) will serve mutable endpoints stale unless told not to. This silently defeats the `seq`-monotonicity freshness check (STORAGE-SUBSTITUTE-HTTP §3-RES.4): the consumer never sees the newer `seq` because the cache answered from the old copy first.

### §3.1 Endpoint mutability classification

| Endpoint | Mutability | Cache policy |
|---|---|---|
| `{content_url_prefix}/{layout}/{hash}` | **immutable** (hash-addressed) | cache-forever; `Cache-Control: public, max-age=31536000, immutable` |
| `{tree_url_prefix}/{path}{tree_leaf_suffix}` | mutable (tree path can rebind) | revalidate; `Cache-Control: no-cache` (or short max-age + ETag) |
| `manifest/current` | **mutable** (seq advances) | `Cache-Control: no-store` |
| signed-pointer (current tree root) | **mutable** | `Cache-Control: no-store` |

### §3.2 Required headers on the CDN origin (same-origin)

```
# immutable content
Cache-Control: public, max-age=31536000, immutable     ; on {content_url_prefix}/...

# mutable freshness endpoints
Cache-Control: no-store                                 ; on manifest/current + signed pointers
```

### §3.3 Consumer-side obligation (the part hash-verify doesn't cover)

Consumers with their own caching layer (service workers especially) **MUST NOT serve mutable CDN endpoints cache-first.** A browser service worker that intercepts same-origin GET cache-first MUST exclude `*/manifest/*` and signed-pointer paths from cache-first (route them network-first or pass-through). This is the consumer's discipline, not the origin's — the origin's `no-store` only works if the consumer's cache honors it. (egui-rust review flag #1: their `sw.js` does its own `caches.match` and ignores response cache directives — so it must explicitly exclude mutable paths.)

---

## §4 Conformance — the browser-success path (closes the "tests the wrong direction" gap)

The existing CORS test vector (BRIDGE-HTTP §8 TV-BH-CORE-7) asserts only `CORS-blocked → network_error` — graceful *failure*. There is no vector proving a browser can *successfully* fetch+verify. This section specifies the missing vectors. **egui-rust authors + runs them** (only browser impl); arch specs the contract here.

### §4.1 TV-CDN-BROWSER-SUCCESS

```
GIVEN  a CDN origin with §2.1 CORS headers (cross-origin) OR §3.2 cache headers (same-origin)
  AND  a published peer tree + content + signed manifest at seq=N
WHEN   a browser/WASM consumer fetches {content_url_prefix}/{layout}/{hash}
THEN   the fetch succeeds (not network_error)
  AND  the in-Rust entity_hash over the body equals {hash}
  AND  the entity ingests into target_namespace
```

This is the vector the native `python3 http.server` demo cannot satisfy (no CORS headers). It MUST run in a real browser against a CORS-correct origin.

### §4.2 TV-CDN-BROWSER-FRESHNESS (the staleness catch)

```
GIVEN  a same-origin CDN + a consumer with a service worker
  AND  a published manifest at seq=N, already fetched once (now in any cache)
WHEN   the publisher updates the manifest to seq=N+1
  AND  the consumer re-resolves
THEN   the consumer observes seq=N+1 (NOT the cached seq=N)
```

This catches the "green for Go, silently wrong in browser" class — the service worker serving a stale-but-hash-valid manifest. Nothing in the native conformance suite catches it.

### §4.3 Test-method warning (elevate to all impls)

`make e2e-worker` serves same-origin with no CORS and may show **green while masking both** the staleness flag (§3) and the CORS flag (§2). **Real-browser + real-CORS verification is required** — Firefox green ≠ WebKitGTK/Tauri green; same-origin-e2e green ≠ cross-origin-CORS green. Validation per egui-rust's `verify_user_facing_surfaces` + `test_each_webview_runtime` discipline.

---

## §5 Quick reference — deployment checklist

**Cross-origin CDN:**
- [ ] CDN origin sends `Access-Control-Allow-Origin` (app origin or `*`)
- [ ] CDN origin is HTTPS (if app is HTTPS)
- [ ] If app ships CSP: `connect-src` includes CDN origin
- [ ] immutable content: `max-age` + `immutable`; mutable endpoints: revalidate / `no-store`

**Same-origin CDN:**
- [ ] immutable content: `Cache-Control: public, max-age=31536000, immutable`
- [ ] `manifest/current` + signed pointers: `Cache-Control: no-store`
- [ ] consumer service worker excludes mutable CDN paths from cache-first
- [ ] no CORS headers needed

**Both:**
- [ ] TV-CDN-BROWSER-SUCCESS passes in a real browser
- [ ] TV-CDN-BROWSER-FRESHNESS passes (publish seq=N → N+1 → consumer sees N+1)
- [ ] verified in Firefox AND Tauri/WebKitGTK, NOT just `make e2e-worker`

---

## §6 Cross-references

- `proposals/PROPOSAL-EXTENSION-BRIDGE-HTTP.md` §2 (hash-verify), §5 (deployment-config classification this runbook fulfills), §8 TV-BH-CORE-7 (the failure-path vector this complements)
- `proposals/PROPOSAL-EXTENSION-STORAGE-SUBSTITUTE-HTTP.md` §3-RES.4 (manifest seq freshness), §3-RES.2 (tree_leaf_suffix + content_layout)
- `core-protocol-domain/specs/extensions/network-peer-extensions/EXTENSION-NETWORK.md` §6.5.5 (consumer modes A1/A2), §13 (browser peer considerations)
- `reviews/ABSORPTION-PEER-REVIEW-CDN-ROUND6-W2-cgid-10-218.md` §2.1 (the finding this runbook resolves)
- `reviews/VALIDATION-PLAN-BRIDGE-HTTP-cgid-10-215.md` (the validation plan §4 vectors extend)
- egui-rust review: `egui-entity-core-rust/docs/architecture/reviews/CDN-ROUND6-AND-W2-BROWSER-REVIEW-cgid-10-218.md` §3 (SW staleness), §4 (CORS)

---

*v1 browser-deployment runbook. Promoted to a v1 deliverable because the browser is the motivating consumer of the static-HTTP path and its headers cannot be a forward-reference. Covers cross-origin (CORS) + same-origin (cache) shapes; specifies the browser-success + freshness conformance vectors that close the "conformance tests the wrong direction" gap. egui-rust validates in real browsers.*
