# EXTENSION-SUBSTITUTE

**Version**: 1.0
**Status**: Active
**Tier:** Operational — Tier 1 (CDN release v1 critical path).
**Authors:** Architecture team.
**Two-mechanism disambiguation (load-bearing, read first):** the HTTP convention here is **Mechanism A — HTTP-as-storage-transport**: an inline HTTP GET whose body bytes *are* entity-encoded content, verified by content hash, with the hash as the sole trust anchor. It is **NOT** `system/bridge/http:get` and does **NOT** use the `system/capability/bridge-http-fetch` cap — that is `EXTENSION-BRIDGE-HTTP` / **Mechanism B** (foreign content wrapped as `system/bridge/http/fetched`), a structurally distinct surface. See NETWORK §6.5.3/§6.5.5 and `GUIDE-EXTENSION-DEVELOPMENT.md §3.7`. (Earlier draft text in the source HTTP proposal conflated the two; that text is superseded by this spec.)

---

## §1 Concept

A peer that misses on a local `system/content:get` SHOULD be able to consult an ordered chain of **substitute sources** before returning the terminal miss. The model follows Nix substituters: ordered, priority-driven, each with an explicit authority claim + fetch mechanism.

The substrate is **type-agnostic**. This extension defines:

- the substitute-source entity (§2.1) and its endpoint/manifest companions (§2.2, §2.4),
- the chain-consultation algorithm (§3),
- the trust contract (§4),
- the composition hook into CONTENT's miss path (§5),
- the convention-dispatch pattern by which per-`substitute_type` fetch mechanics plug in (§6),
- the capability model (§8).

Per-`substitute_type` fetch mechanics are delegated to **convention extensions** that register a handler at `system/substitute/<type>:try`. The v1 concrete convention — `substitute_type: "http"` — ships as §7 of this spec.

**Three positions are load-bearing:**

1. **Companion, not core.** The substitute machinery lives in its own extension with its own `system/substitute/*` namespace, NOT inside CONTENT. CONTENT carries only a small miss-hook (§5). This was a deliberate Sketch-B decision (per `ANALYSIS-BREAKING-CHANGE-TOUCHPOINTS-PRE-RELEASE.md`): prefer new optional extensions over growing bedrock CONTENT. Installing this extension is additive; not installing it leaves CONTENT's miss behavior (404) unchanged.
2. **Content-trust is self-sufficient.** A substitute fetch is trustworthy when the returned bytes hash-match the requested content hash — regardless of who served them. The substitute *entry* additionally needs the publisher's signature, because the entry makes an authority claim ("peer P endorses this source for P's content"); the *bytes* need only the hash.
3. **Conventions plug in via standard dispatch.** There is no in-process type registry. A convention extension ships a handler at `system/substitute/<type>:try`; the consultation finds it via normal V7 §6 handler-URI dispatch.

---

## §2 Entity types

### §2.1 `system/substitute/source`

The substitute-entry entity. Stored at the peer-relative path `system/substitute/sources/{hex(content_hash)}` — its own namespace, cleanly separated from `system/content/*`.

```
type: "system/substitute/source"
data: {
  name:              primitive/string,        ; human-readable label
  substitute_type:   primitive/string,        ; "http" | "peer-to-peer" | "nix-cache" | …
  source_peer_id:    system/hash,             ; whose authority this entry claims
  endpoint:          primitive/any?,          ; OPAQUE, polymorphic per substitute_type (see below)
  fetch_template:    primitive/string?,       ; LEGACY URL template; deprecated; overridden by endpoint
  priority:          int,                      ; ascending; lower = consulted first
  enabled:           bool,
  expires_at:        time?,                    ; standard cap-style expiry
  supersedes:        system/hash?              ; per ATTESTATION §5 supersession chain
}
```

The entry's signature is carried per V7 §5.2 target-matching and reachable at the invariant pointer `system/signature/{hex(source.content_hash)}`, signed by `source_peer_id` (§4). (No `refs:` block — V7 is refless; this matches the REGISTRY/DISCOVERY consolidation discipline.)

**`endpoint` is OPAQUE (`primitive/any?`)** — *pinned ruling.* Each `substitute_type` carries a structurally different endpoint shape, dispatched on `substitute_type`, exactly as REGISTRY's `resolver-chain-entry.hints` is opaque per backend. The substrate MUST NOT pin `endpoint` to a single concrete type; the handler for `substitute_type` interprets it. The HTTP convention's concrete endpoint shape is `system/substitute/endpoint` (§2.2). An impl that tightens `endpoint` to a single endpoint type is non-conformant (it forecloses non-HTTP conventions). `endpoint` is optional only because `fetch_template` is the legacy fallback; a conformant entry carries exactly one of `endpoint` (preferred) or `fetch_template` (deprecated).

### §2.2 `system/substitute/endpoint` (carried by the HTTP convention)

The concrete endpoint shape the `http` convention (§7) expects, matching the NETWORK §6.5.3 http-poll transport profile (the shape-sharing is what lets a peer's published transport profile and its substitute entries describe the same URL space without duplication).

```
type: "system/substitute/endpoint"
data: {
  tree_url_prefix:    primitive/string,   ; URL, e.g. "https://shared.example.com/peers/<peer_id>"
  content_url_prefix: primitive/string,   ; URL, REQUIRED publisher commitment (see below)
  content_layout:     primitive/string,   ; "flat" | "sharded-2-flat" | "sharded-2-4" | "sharded-2-2"
  tree_leaf_suffix:   primitive/string?    ; default ".bin"; appended literally to tree-leaf URLs
}
```

**`content_url_prefix` is REQUIRED** — *pinned ruling.* The two-prefix model (`tree_url_prefix` + `content_url_prefix` as separate publisher commitments) exists precisely so content can be dedup'd cross-peer to a different host/prefix than the tree (scenario S4 below). A derivation default ("derive content_url_prefix from tree_url_prefix when absent") silently defeats that case, so there is no default — the publisher MUST state it. An impl that treats it as optional-with-derivation is non-conformant.

`content_layout` URL construction:
- `flat`: `{content_url_prefix}/{hash}`
- `sharded-2-flat`: `{content_url_prefix}/{hash[0:2]}/{hash}`
- `sharded-2-4`: `{content_url_prefix}/{hash[0:2]}/{hash[2:4]}/{hash}`
- `sharded-2-2`: alias for `sharded-2-4`

**Tree-leaf URL suffix.** The entity tree permits leaf-AND-subtree coexistence at one path (`/x` may be both a leaf and the root of `/x/y`); filesystems and HTTP servers cannot. Tree-leaf URLs (under `tree_url_prefix`) MUST carry the `tree_leaf_suffix` disambiguator (default `.bin`; deployments MAY override). The suffix is a publisher commitment; consumers append it literally (no consume-time URL rewriting). Validated empirically: workbench-go's `entity-fetch` + Python `http.server` round-tripped a 205-file repo on this convention.

Deployment scenarios (the CDN-corridor stress-test cases):
- **S1 single-peer-dedicated** — `tree`=`https://peer-a.example.com`, `content`=`…/content`
- **S2 single-peer-on-CDN-with-path-prefix** — `tree`=`https://cdn.example.com/my-peer`, `content`=`…/content`
- **S3 multi-peer-shared (per-peer content)** — `tree`=`…/peers/<peer_id>`, `content`=`…/peers/<peer_id>/content`
- **S4 multi-peer-shared (dedup'd content)** — `tree`=`…/peers/<peer_id>`, `content`=`https://shared.example.com/content`
- **S5 peer with multiple profiles** — one manifest per http-poll profile
- **S6 subdomain-per-peer** — `tree`=`https://peer-a.cdn.example.com`, `content`=`…/content`

### §2.3 `system/substitute/try-request`

The request entity passed to a convention handler's `:try` op (wire shape PINNED by the storage-substitute cross-impl rulings):

```
type: "system/substitute/try-request"
data: {
  entry:  system/substitute/source,   ; the FULL source entity (not its hash — the handler needs endpoint/type)
  hash:   system/hash                  ; the content hash being fetched
}
```

The handler returns the **fetched bytes/entity directly** (raw; no `try-result` wrapper — the consumer holds `hash` and verifies the raw bytes against it). `not_found` / `error` use the standard handler-result error mechanism. The claimed `source_peer_id` is local dispatcher context plumbed at the handler layer, NOT a wire field on `system/content:get-request` (the consult fires on the consumer's own local miss; it already knows whose content it is — keeps core un-grown).

### §2.4 `system/substitute/snapshot-manifest` (HTTP convention; OPTIONAL)

The optional path→hash discovery artifact for the HTTP convention (§7). See §7.2 for full semantics.

```
type: "system/substitute/snapshot-manifest"
data: {
  source_peer_id: system/hash,
  snapshot_at:    time,
  seq:            int,                       ; monotonic per source peer (§7.2 freshness)
  endpoint:       system/substitute/endpoint, ; explicit; see §2.2
  path_index:     map_of(system/hash),       ; path → content hash
  content_count:  int,
  root_hashes:    array_of(system/hash),
  predecessor:    system/hash?               ; optional chain to prior manifest's content hash
}
```

Signature reachable at `system/signature/{hex(manifest.content_hash)}`, signed by `source_peer_id` — **MUST** (§7.2). Without it, the path_index is forgeable by anyone serving the manifest URL.

---

## §3 Chain-consultation handler

When CONTENT's `system/content:get` reaches a local miss, it invokes the substitute-consultation hook (§5). The algorithm:

```
substitute_consult(hash, requester, cap_chain):
  ; (1) Pending sidecar wins — in-flight authoritative sync beats substitute (§5)
  if content_pending_sidecar.has(hash):
    return 503 blob_pending_sync            ; do NOT consult chain

  ; (2) Capability gate — fail closed (§8)
  if not cap_chain grants (system/substitute/sources, "consult", resource = ctx.resource_target):
    return 404 not_found                     ; chain not consulted

  ; (3) Bare-hash queries — no wildcard consultation in v1 (§4)
  if hash.claimed_source_peer_id == null:
    return 404 not_found

  ; (4) Enumerate matching entries
  entries = list("system/substitute/sources/")
    .filter(e => e.data.enabled
              and e.data.source_peer_id == hash.claimed_source_peer_id
              and (e.data.expires_at == null or e.data.expires_at > now())
              and signature_valid(e))         ; §4
    .sort_by(e => e.data.priority)            ; ascending

  ; (5) Consult in order
  for entry in entries:
    handler_uri = "system/substitute/" + entry.data.substitute_type
    result = invoke(handler_uri + ":try", { entry, hash }, cap_chain)
    if result is cap_denied:    return cap_denied        ; ABORT chain (§4, §8)
    if result is bytes:
      if not verify_hash(bytes, hash): continue          ; mismatch → discard, advance
      if ingest(bytes, hash, target_namespace_from_cap) is cap_denied:
        continue                                         ; ingest-denied is transient-for-entry; advance (§3.1)
      return ok(bytes)
    ; not_found / transient_error → advance

  ; (6) Exhausted
  return 404 not_found.with_meta({ substitute_chain_attempted: true, … })   ; §3.2
```

### §3.1 cap-denied vs ingest-denied

- A convention handler returning **`cap_denied`** (the consumer cannot fetch this URL at all) **ABORTS** the chain — advancing to another URL won't help if access is denied at the cap layer.
- A post-fetch **ingest cap denial** (consumer can fetch but cannot land *this* namespace) is **transient for that entry**; the chain ADVANCES (a later entry may target a landable namespace). The fetched bytes are discarded, never silently leaked into any namespace. If all entries fail this way, the chain exhausts to 404. `ingest_cap_denied` is internal-to-chain, never a wire-visible status from `system/content:get`.

### §3.2 Chain-exhaustion status + error codes

- Chain entered, all entries fail **terminally** → `404 not_found` (pending was definitively false at chain start).
- Chain entered, last entry returned a **transient** → `503 substitute_chain_pending` (distinct from `blob_pending_sync`; retry MAY help).
- Informative meta on the 404 (impl MAY omit in prod, SHOULD include in dev): `{ substitute_chain_attempted: bool, substitute_chain_length: int, substitute_chain_last_error?: string }`.

Error codes owned by this extension: `substitute_chain_pending` (503), `substitute_chain_exhausted` (informative 404 meta), `manifest_signature_invalid` (§7.2), `manifest_stale_seq` (§7.2).

---

## §4 Trust contract

A substitute entry is an authority claim — "peer P endorses this source as legitimate for fetching content P authored."

- **Signature MUST (wire default).** Every entry MUST carry a same-`source_peer_id` signature at `system/signature/{hex(source.content_hash)}`. Unsigned/invalid entries are rejected at consultation time.
  - **Operator-trust override (deployment policy, not wire relaxation).** An operator who authored a source MAY configure an explicit local trust override to consult specific operator-authored sources without signature verification. It MUST be explicit local config (never inferred) and the consultation MUST surface the skip-reason. Default is closed (verify).
- **No transitive trust.** An entry claims authority for `source_peer_id`'s content only. Peer B serving peer A's content without A's signature is invalid.
- **No transitive following.** Bytes fetched from a substitute are hash-verified + ingested locally; any substitute metadata embedded in the result is advisory, never a new chain to follow. The only chain is the consumer's local `system/substitute/sources/` set.
- **No wildcards in v1.** A bare-hash query (no claimed `source_peer_id`) does NOT trigger consultation (§3 step 3). Wildcard-source entries are out of scope for v1.

**Trust-default rationale (note vs descriptor).** The substitute default (source-peer signature required) is the *opposite* of CONTENT §5.3 descriptors (authority-free). Intentional: a descriptor lives in the publisher's own subtree (transport-trust covers it); a substitute entry endorses a *non-P intermediary* (no transport-trust path — needs P's signature directly).

---

## §5 Composition with CONTENT (the miss-hook)

CONTENT's content-get flow, on local miss + pending-clear, MUST invoke the substitute-consultation hook before returning 404, when this extension is installed. Ordering is **MUST**: `pending-check → substitute-consult → 404`.

```
content_get(hash):
  if local_store.has(hash):       return bytes
  if pending_sidecar.has(hash):   return 503 blob_pending_sync   ; in-flight sync; do NOT consult
  ; ↓ substitute miss-hook fires here (if installed) ↓
  if substitute_consult(hash) succeeds: return bytes
  return 404 not_found
```

The substitute chain is for the *miss* case, not the *racing-sync* case: if the bytes are already en route from the authoritative publisher (pending), wait. On `503 blob_pending_sync` the consumer does standard transient-retry; the chain is consulted on retry only if the pending sync resolved to a miss. Impls without this extension return 404 as today (the CONTENT-side hook is ~10 lines, additive, conditional — `EXTENSION-CONTENT` Amendment).

---

## §6 Convention-dispatch pattern

Each `substitute_type` is a separately-installed convention extension that registers a handler:

```
system/substitute/<type>:try
  try(entry: system/substitute/source, hash: system/hash) → bytes | not_found | error
```

Dispatch is normal V7 §6 handler-URI dispatch — installing a convention makes its handler discoverable; uninstalling makes it unavailable; **no new registry surface**. A consumer determines dispatchable types by enumerating installed handlers under `system/substitute/` (a `tree:list` + filter) or — preferred — by tracking installed extensions via their `system/handler` manifest declarations. An entry whose `substitute_type` has no installed handler yields `not_found` and the chain advances.

For v1 the registered convention is **`system/substitute/http:try`** (§7). Future conventions (`peer-to-peer`, `nix-cache`, …) each ship in their own extension and bind to this contract.

---

## §7 The HTTP convention (`substitute_type: "http"`)

The v1 concrete convention. Two mechanisms over one transport binding.

### §7.1 Mechanism A — bare-hash fetch (the load-bearing baseline; no manifest)

On a content-miss for hash H whose claimed source has an `http` entry, the consumer builds a URL from `endpoint.content_url_prefix + content_layout + H`, performs an **inline HTTP GET**, computes the content hash over the body, verifies it equals H (**MUST**; mismatch → discard + advance), and ingests. This is **Mechanism A** — inline fetch + hash-verify; **NOT** `system/bridge/http:get`, and it does **not** use `system/capability/bridge-http-fetch`. The bytes are already entity-encoded; the content hash is the sole trust anchor.

Mechanism A requires no manifest and no publisher commitment beyond "I publish content at this URL prefix." If a consumer already knows the hash it wants (from a prior conversation, a signed link, an out-of-band share, or a refs-walk from a known root), it fetches via A without the publisher ever publishing a manifest. The egui-WASM CDN release v1 is bare-hash throughout: the consumer has a known root hash via bundled startup state, walks refs, fetches by hash.

Error mapping: `cap_denied` ABORTS the chain (§3.1); transient errors (5xx, network_error) advance to the next entry. A `404` for a hash referenced by a (valid-signature) manifest surfaces as `network_error` and the chain advances (the path_index claim stays trusted; the content just isn't fetchable yet — §7.3).

### §7.2 Mechanism B — signed snapshot manifest (OPTIONAL optimization on top of A)

A publisher who also wants **path→hash discovery** (so consumers can resolve "what is currently at path /docs/foo" without already knowing the hash) MAY publish a signed `system/substitute/snapshot-manifest` (§2.4) at `{tree_url_prefix}/manifest/current`. The `path_index` is whatever subset of the tree the publisher elects to commit at a snapshot moment — partial slices are valid. Consumers without the manifest fall through to Mechanism A per individual hash.

**Manifest signature is MUST.** The `path_index` is an authority claim about the publisher's tree; anyone serving the URL can forge it, so hash-verifying the manifest against its own hash proves nothing about authorship. Verification:

```
verify_manifest(manifest, source_peer_id):
  1. manifest_hash = content_hash(manifest)
  2. resolve signature at system/signature/{hex(manifest_hash)}
  3. verify sig targets manifest_hash
  4. resolve source_peer_id's current identity-cert + public key
  5. verify sig over manifest_hash
  6. any failure → error manifest_signature_invalid; reject (do NOT use path_index)
  7. success → trust path_index
```

Without a valid signature the consumer MAY still fetch + hash-verify individual blobs (Mechanism A), but MUST NOT use the manifest for path resolution.

**Freshness via `seq`** (anti stale-replay). Consumer caches the highest `seq` seen per source peer: `seq > cached` → accept + supersede; `seq == cached` → accept (re-affirmation); `seq < cached` → reject `manifest_stale_seq`, do NOT use path_index. First-ever manifest from a peer: any `seq` acceptable (learn-by-observation; authentic + monotonic-newest-seen, not guaranteed-latest). `predecessor` MAY chain to the prior manifest's content hash (recommended for audit deployments).

**HTTPS defense-in-depth.** The convention SHOULD reject path_index/endpoint URLs with `scheme != "https"` at consumption time, even if a cap permits them.

**Mismatch detection.** If the URL the manifest was fetched from doesn't match `endpoint.tree_url_prefix`, impl SHOULD warn (config drift) but MAY accept — the signature is the trust anchor, the URL is just where the bytes came from.

### §7.3 Publishing-order discipline

Publishers SHOULD upload content blobs FIRST, then the manifest LAST — the manifest's presence at `{tree_url_prefix}/manifest/current` is the publisher's commitment that all `path_index` content is fetchable. Consumer on 404 during a path_index fetch: surface `network_error`, keep the (signature-valid) path_index trusted, retry per the chain. Atomic-publish (temp URL → rename) or inline-signature manifests eliminate the residual manifest-upload race; that's deployment-config, documented in the publishing runbook.

### §7.4 Relationship to descriptors (three valid publisher patterns)

CONTENT §2.4 descriptors + §5.3 path-index-by-listing already give a per-item, per-publisher path→hash affordance; the manifest is a flat-baked-snapshot optimization of the same information suited to static HTTP where per-item lookups cost round-trips. Publishers choose:

1. **Descriptors only** — per-blob descriptor walk; no bake step. Suits frequently-updated content.
2. **Manifest only** — flat baked manifest; faster first-paint over static HTTP; re-bake per tree mutation. Suits snapshot/versioned/archival publishing.
3. **Both** — consumer prefers a current-seq signature-valid manifest, falls back to descriptor walk if stale/missing. Recommended for production.

The manifest does NOT replace descriptors; it complements them as a transport-binding-specific optimization. A signed-mutable-pointer unification is future work, not v1.

---

## §8 Capability model

Three-cap composition, checked in cheapest-first order:

| Cap | Layer | Gates |
|---|---|---|
| `system/capability/content-substitute-consult` | this extension | Whether the chain is consulted at all (cheap; pre-flight; fail-fast) |
| the convention handler's own fetch cap | convention (§7) | The outbound fetch. For the `http` type (Mechanism A) this is an inline GET + hash-verify; it does **NOT** use `bridge-http-fetch`. Checked as `(handler, operation)` + constraints via V7 §5.2 per the named-capability-mapping ruling. |
| `system/capability/system/content:ingest` | CONTENT | Landing the fetched bytes into a namespace |

**The consult-cap is not a string-presence flag.** It is shorthand for a grant on `(handler = system/substitute/sources, operation = "consult", resource = ctx.resource_target)` (the triggering EXECUTE's target namespace — consume `ctx.resource_target`, not a static path), checked by V7 §5.2 `check_permission`. **Fail closed:** absent a matching grant, the chain is NOT consulted (404); "caller holds any token" is not a match. This closes the arbitrary-caller-triggered-outbound-fetch + forced-ingestion hole. The consult-cap MAY narrow via `constraints` (byte-equal under delegation, V7 §5.6): `source_peer_id` (restrict to specific publishers), `substitute_types` (restrict to specific handlers).

Composes orthogonally with CONTENT §6.4's namespace-cap matrix — the consult-cap is a side-effect gate in the capability namespace, not a read/ingest axis of the content matrix.

---

## §9 Conformance

Required for v1 (cross-impl convergent):

- §2.1 source entity (incl. opaque `endpoint`), §2.2 endpoint shape (incl. required `content_url_prefix`), §2.3 try-request wire shape.
- §3 chain-consultation algorithm incl. §3.1 cap-denied-vs-ingest-denied and §3.2 status/error codes.
- §4 trust contract (signature MUST; no transitive; no wildcards v1).
- §5 CONTENT miss-hook composition + ordering.
- §6 standard-dispatch convention model.
- §8 fail-closed capability model.
- §7 HTTP convention Mechanism A (bare-hash fetch + hash-verify); Mechanism B (manifest) signature MUST + `seq` freshness.

**Test vectors** (from the source proposals' sign-off contract):

- Substrate: `TV-SS-CORE-1..7` (chain order; hash-mismatch discard+advance; unsigned-entry reject; cap-denied abort; exhausted→404+meta; `expires_at` exclusion; `supersedes` precedence); `TV-SS-COMP-1..4` (pending→503-no-consult; pending-clear→consult; all-transient→503 chain_pending; all-terminal→404); `TV-SS-BARE-1..2` (no-claimed-origin→no-consult; claimed-origin→filtered); `TV-SS-DISP-1..3` (dispatch via installed handler; missing handler→advance; cap-denied→abort); `TV-SS-TRANS-1` (embedded substitute claims treated advisory).
- HTTP convention: `TV-CDN-CORE-1..6` (URL build per layout; expected-hash threaded; cap-denied abort; transient advance); `TV-CDN-SIG-1..4` (manifest signature valid/missing/invalid/wrong-identity); `TV-CDN-FRESH-1..3` (`seq` first-seen/higher/lower); `TV-CDN-URL-1..2` (endpoint mismatch warn-may-accept); `TV-CDN-TLS-1` (reject `http://` at consume); `TV-CDN-DESC-1..2` (prefer current manifest / fall back to descriptors); `TV-CDN-PUB-1` (not-yet-uploaded hash → network_error + advance).

SHOULD-implement / documented-acceptable: §3.2 informative 404 meta (dev SHOULD, prod MAY omit); `TV-SS-DISP-*` (against the http convention if installed); `TV-SS-TRANS-1`.

---

## §10 Out of scope (v1)

Wildcard / bare-hash substitution; transitive substitute following; cross-peer cap delegation for consultation (a peer's own consult-cap covers it); substitute-result coalescing across concurrent requests (impl-internal); health/latency/cost-based chain ordering (v1 is priority-only); substitute-specific caching (each successful fetch lands via standard CONTENT ingest + GC). The substitute-relay/aggregator case (Mode A — a wire-level claimed-source field) is deferred and scoped to the relay when a driver emerges.

---

*Consolidated from the storage-substitute sources proposal (substrate) + the storage-substitute HTTP-convention proposal (HTTP convention, §7). Q2 (`content_url_prefix` required) + Q3 (`endpoint` opaque) pinned per the cohort release-green ambiguities ruling. Mechanism-A disambiguation applied per the corridor cross-chain synthesis (supersedes the source HTTP proposal's pre-disambiguation `bridge-http-fetch` references). Cross-impl ruling provenance: the storage-substitute cross-impl rulings, the named-capability-mapping ruling, and the cycle-closeout 0.3 ruling.*
