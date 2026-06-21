# GUIDE — Serving mode (a live peer as its own CDN origin)

**Status**: Active
**Grounding:** the serving-mode content-scope and content-body-shape rulings. Validated three-way (Go/Rust/Py).

---

## 1. What serving mode is

A live peer additionally exposes `http-poll` routes (`GET /content/{hex33(H)}`, `GET {tree-prefix}/{path}`) from its own store, on the same HTTP listener as its `http` EXECUTE route. One process is both a live peer and a static-style CDN origin. Useful for the dev loop, embedded deployments, and being a CDN origin without a separate mirror.

The route is **content-addressed fetch over HTTP**: the consumer trusts the hash, not the host. There is no capability on the request (a browser/`curl`/CDN can't present one) — so the question is never "who may fetch?" but **"what does the route answer for?"** That is `serve_scope`.

## 2. The published set — what to serve

Your live content store holds *everything you've ingested* — private app data, inbox-received blobs, other peers' content, cap tokens, signatures. The serving route must answer only for what you chose to publish. Two ways to populate the published set (they unify — both are tree-defined; serve `H` iff `H ∈ a tree-defined published-set`):

1. **Content-namespace (ship this first; simplest).** Bind the hashes you want public under a content namespace — `system/content/{ns}/{hex33(H)}` — and serve exactly what's bound there. Membership is one `tree:get`. This *is* CONTENT §6.4.2 Hash Tree Presence; the serving predicate is `tree:get({namespace}/{hex33(H)}) ≠ null`. (Intra-protocol the same namespace is capability-scoped per CONTENT §6.4.3; over HTTP, hash-knowledge replaces the cap — same scope object, different per-request auth.)
2. **Subtree closure (richer).** Designate a published subtree, walk it (TREE §4/§6 closure-bundle from `published-root`), collect each entity's `content_hash` + content refs. This is what a static publisher uploads; its closure *is* its fileset. Use when the published set is naturally a subtree.

**Tree-face + content-face.** The published set has two faces: `TREE_GET` resolves *which paths* are in scope; `CONTENT_GET` resolves *which hashes*. Both ride the same `serve_scope`. Publish a subtree → its paths and the content reachable from it are both served.

**Publish-side flow.** Typically: copy a filtered sparse subset of your full tree (types, handlers, datasets, a few signatures, the manifest, peer-info) into a publish set → bundle → snapshot → sign the root/manifest → push content for that closure. Safe by construction (you only push what you selected). What goes in the sparse selection (which peers, which system subtrees, exclude private data) is *your operator policy* — the spec gives you the mechanism, not the policy.

## 3. The `whole-store` opt-in — and why it's not the default

`serve_scope: whole-store` (CONTENT §6.4.1 single-trust-domain topology) serves **any** stored hash. It's a valid, explicit, default-off "debug-open content public" posture (a peer whose entire store is public anyway, a single-purpose mirror, a testbed). **CONTENT §6.4.1 marks multi-party use of this security-defective** — here's why (the baseline confidentiality audit):

- The route is **integrity-safe** — hash-verified; an attacker gets bytes matching the hash they asked, or nothing. Zero forgery.
- The only exposure is **confidentiality**, fully governed by `serve_scope`:
  - **T1 enumeration** — impossible (256-bit hashes, no list endpoint; "any *known* hash resolves," not "browse the store").
  - **T2 leaked-hash retrieval of *unpublished* content** — under `whole-store`, any leaked hash resolves; under `published-set`, a leaked hash for unpublished content `404`s. **Eliminated by default scope.**
  - **T3 cap-token / signature harvesting** — those are content-addressed entities; under `whole-store` their hashes resolve to bearer-ish credentials; under `published-set` they're not in a published *content* closure. **Eliminated by default scope.**
  - **T4 presence oracle** — `200`-vs-`404` leaks "this peer holds H"; mitigated by returning an **identical `404`** for out-of-scope and not-held (MUST, §6.5.6).

So default to `published-set`; reach for `whole-store` only knowingly. No type-blocklist is needed — the default scope excludes the sensitive types by construction.

## 4. Restricting *who* fetches — foreign auth (not our mechanism)

Serving mode is for mostly-public sharing. If a deployment needs to restrict *who* can fetch, that is **foreign auth** — OAuth, HTTP Basic, mTLS, an API gateway, a signed-URL scheme — fronted by a reverse proxy (nginx/Caddy). It is a deployment concern, standard HTTP, **outside the entity-core spec**. Conceptually, integrating a foreign auth standard into the entity model (rather than at the proxy) would be a BRIDGE-HTTP-family bridge — a future `bridge-oauth` *only if a driver materializes*. v1 ships scope; the rest is named so you know the levers exist.

## 5. Implementer notes

- **Route demux (G2):** one listener, method+path. `POST {http-path}` → EXECUTE; `GET {content-prefix}/{hex33(H)}` → content-by-hash; `GET {tree-prefix}/{path}` → tree binding. No separate server.
- **Body shapes (§6.5.3.1):** `CONTENT_GET` → bare hashable `ECF({type,data})` (2-key, MUST pure-body-rehash to H — verifiable by a decoder-less CDN/browser). `TREE_GET` **leaf** → the **bound hash** as a `system/hash` pointer `ECF({type:"system/hash", data:H})` (2-key), **NOT** the dereferenced entity (Amendment 6). The tree is `path → hash`; the leaf route hands back the hash and the consumer second-hops `CONTENT_GET /content/{hex33(H)}` for the bytes. Returning the dereferenced entity here duplicates entity bytes at every tree path bound to `H` — a static CDN can't dedup it (V7 §1.7). This is exactly `tree:get mode:"hash"` over HTTP; the dereferenced default-mode `tree:get` stays available in-process and over live `http` EXECUTE. `TREE_GET` **listing** → the `system/tree/listing` entity (already a map of `name → {hash, has_children}` — pointers, not inlined entities; it was always two-hop).
- **Hex:** 66-char `hex33` (format-code included) everywhere — URL, §6.4.2 binding, `ETag`. Reject 64-char digest-only (`400`); reject unverifiable algorithm byte (`400`, fail-closed).
- **Cache-Control:** `immutable` on `CONTENT_GET` (content is immutable forever); **never `immutable` on `TREE_GET`** — path→hash bindings are mutable; an immutable hint makes caches serve stale bindings after a rebind. Omit it or use a revalidation-friendly directive on tree routes.
- **`TREE_GET` is published-scope-gated, not `501`.** Resolve iff the path is in the served subtree; `404` otherwise. `501` is an acceptable deferral during bring-up, never the shipped shape.
- **Sharded layout** slices the same 66-hex `{hash}`; `[0:2]` is the algorithm partition (`00` today). The shard prefix repeating in the leaf is the normal git/CAS pattern.

## 6. Aggregators are app-layer

A multi-tenant host, or a CDN-fronting peer that serves a federated view of many peers, is an **application built on** the serving primitive — not substrate. The primitive is "one peer, scope-bounded, serving its own store." Compose upward from there.

## 7. Pathing (Amendment 5) — operator guide

**The routes (every URL a concrete object key — no trailing slash).** `GET {content}/{hex33(H)}` content-by-hash; `GET {peer_id}/{path}.bin` the bound-hash pointer (`system/hash`, then second-hop to `/content/{hex33}`); `GET {peer_id}/{path}.list` listing; `{peer_id}.list` peer-root listing; `peers.list` the all-peers (universal-tree-root) view (normal — every peer's root is a set of peer-ids); `GET {manifest}` the signed manifest (terminal). Two suffixes (`tree_leaf_suffix` `.bin` / `tree_listing_suffix` `.list`) are advertised in the profile, **must differ**; consumers read them, don't hardcode. Append-one/strip-one makes suffix-ending names collision-free (no operator action). **No redirects** — a trailing slash carries no meaning here (we rely on it carrying none, precisely because static CDNs normalize slashes and would otherwise break the listing).

**Status codes:** 200 / 400 / 404 / 405 (+`414` MAY); never `501`, never `3xx`. `{peer_id}.bin` ⇒ 404 (roots are directories); bare `peers` ⇒ 404 (only `peers.list`).

**Static pre-generation (Mode 1).** Push leaf objects (`{path}.bin`), listing objects (`{path}.list`), the content-addressed `next_page` chain (`/content/{hex33}`), and the signed manifest. **Each `next_page` chain page MUST be namespace-bound** (§6.4.2) or it 404s. Re-render the mutable head on subtree change; chain pages are immutable. **Content-store GC:** re-rendering a paginated listing orphans the old chain pages — until cohort GC exists, the pipeline tracks prior chain hashes and prunes, or an operator sweep does.

**The cap-token publish pipeline.** `serve_scope` is a `system/capability` token. Pipeline: author/update cap `C` → walk the live tree against `C` → enumerate in-scope `{(path, hash)}` → render listings (head + chain) → push leaf/listing/content/manifest objects → re-render on subtree change or cap revision. Two properties: (1) **the cap IS the audit log** — `cap-inspect serve_scope.cap` shows exactly what's exposed; version-control the cap, its hash-history is the publication audit trail. (2) **Re-render is byte-deterministic** — same `(cap, tree-state)` ⇒ byte-identical artifacts; cross-impl pipeline conformance falls out of the ECF byte-equality discipline.

**Revocation is one-way for content.** Immutable `/content/{H}` objects persist in CDN caches until eviction — effectively forever. **Revoke by NOT publishing** (rotate the publishing key/cap so future closures exclude `H`), never by trying to un-publish `H`. Scope-revocation gives zero-TTL guarantees only for the binding (listing/tree face) — re-render every listing that referenced the path AND invalidate the CDN; use short `max-age` on listing/manifest (mutable) routes.

**CORS + MIME are the deployment's job (not impl code, not ours to fix).** Cross-origin requires CORS — the runbook (`RUNBOOK-CDN-BROWSER-DEPLOYMENT`) pins **both halves**: (a) server-side `Access-Control-Allow-Origin` on every route (wildcard if needed); (b) the **consumer simple-request discipline** — poll-read consumers MUST issue plain GETs with **no custom request headers** (a custom header triggers an `OPTIONS` preflight a static CDN can't answer, silently breaking the bar). Static buckets MUST also map `.bin`/`.list` → `Content-Type: application/cbor` (Mode-1 only; live Mode-2 peers set it directly; and a conformant consumer decodes by bytes regardless, so MIME is interop-politeness, not correctness). We ship standards-compliant and document the deployment requirements; the chaotic-web (proxies/bots/CDN quirks) is acknowledged, not guaranteed.

**Implementer discipline — every path-taking layer MUST be peer-id-agnostic.** Serving exposes a scoped projection of the peer's **local view of the universal tree** (V7 §1.4 *Local view* / *Authority layers*; the model is normative there — this is the impl-application reminder, not a new rule). A peer holds bindings under `/{any_pid}/...` — its own authoritative namespace AND cached/mirrored remote namespaces — so `peers.list` enumerating several peer-ids is the *normal* universal-tree-root view, not a multi-tenant special case. The recurring failure mode (observed across the cohort three cycles running, at a *different* layer each time — enumeration, scope filter, listing trim-prefix, cap synthesis) is a **local-peer assumption baked into one layer**: a `QualifyPath(localID, ...)` that prepends the local peer-id to an already-absolute `/{otherID}/...` path; a `peers` enumerator hardcoded to the local peer; a cap synthesized as `{ns}/*` (→ `/{localID}/{ns}/*`) instead of `/*/{ns}/*` (peer-wildcard). The review rule for any new layer that takes a path argument: **"does this layer bake in a local-peer-id assumption?"** If yes, justify it (caps legitimately root the granter's namespace) or it's a bug. The durable regression guard is the cross-impl **`universal_address_space`** conformance category (wire-level `/{other}/...` round-trips: peer-relative≡absolute both directions, foreign-namespace publish/isolation, foreign listing at root and at depth) — run it in every cohort's cross-impl suite, not just one impl's. It found bugs in two of three impls within minutes of existing; the absence of a wire-level cross-peer round-trip check is what let the rot accumulate.

---

*Operator guidance for `EXTENSION-NETWORK.md §6.5.3/§6.5.3.1/§6.5.6`. The normative surface is those sections + CONTENT §6.4 + V7 §3.9 (`next_page`); this guide is the how-and-why. The universal-address-space model is V7 §1.4 + §1.7 — this guide points at it, does not restate it as authority.*
