# PROPOSAL — Peer-issued registry backend + registration flow

**Status:** **Ratified v1.0 — cgid-10-231. Folded into EXTENSION-REGISTRY §6a (peer-issued backend). Cohort impl landed 3-way GREEN; Rust's seven spec-doubts (P1–P7) ruled + folded (§2.5).** v0.4 closed the cohort's post-impl spec-doubt log: P4 (`sig.signer` wording) / P6 (empty trust-anchors = fail-closed) / P7 (by-target revocation index = normative) fold into §2.1/§2.3; P1+P5 pin the v1 Ed25519-identity-multihash trust-anchor floor (pubkey derived from peer-id; non-derivable forms carry the key in config — deferred); P2/P3 confirm the cohort's layering reading. **Conformance status:** the 6 Part-A `REG-PEERISSUED-*` vectors run **green per-impl** in Go, Rust, and Python (injected-reader: live vectors via in-test Reader, offline via local pre-seed). They are NOT owed a cross-impl Keystone byte-gate — the only primitives that could diverge across impls (CBOR entity encoding, Ed25519 signature verify) are already cross-impl locked by the codec corpus + agility work; peer-issued resolve is ordinary handler logic over those proven primitives, validated per-impl. A real-wire http-poll end-to-end against a served ECR coral-reef is a **demo-validation** nicety (does the wire path light up), not a cross-impl correctness gate, and is not release-blocking. v0.3 below for history. v0.3 absorbs core-go's critical read: the **manifest absence rule** is now NORMATIVE (§2.2a — `coverage` flag: `partial` default falls through to per-name, `complete` is authoritative denial-of-existence, DNSSEC-NSEC / TUF model) + its vector `REG-PEERISSUED-MANIFEST-ABSENCE-1`; URL-path `registry/` segment rationale stated (collision-avoidance, intentional); per-name-vs-manifest divergence rule pinned (per-binding TTL/revocation/supersedes governs, manifest never wins over a still-valid newer per-name read); "no operator-trust override" stated; manifest framing softened from "verbatim-in-shape" to "same pattern, registry-domain fields" (the SUBSTITUTE deltas are named). Part A + curated tool were independently buildable before this rev (they don't touch the manifest); v0.3 closes the manifest's last fracture-surface before the doc circulates. Part A (resolve-side backend) + Part B.curated (operator signing tool) are the **release ask** and are fully specified — handed to the peers to build (Go reference first). Part B.live (`register-request` + issuer-policy + domain-control) is **explicit follow-on**, not release-blocking; its open questions (§8) are deferred, not answered. Open questions resolved this rev: §8 Q1 — **spec both, implement the index for v1** (per-name pointers ship; the signed manifest is now fully specified in **§2.2a** mirroring SUBSTITUTE §2.4/§7.2, so its format is pinned against fracturing even though its impl is deferred); §8 Q2/Q3/Q4 deferred to the Part B.live cycle. No V7 change, no cohort blocker for the demo.
**Scope:** the **release unblock** for the universal namespace. Two parts: **(A) the resolve-side `peer-issued` backend** — turn a name into a *verified* binding by fetching the registry's signed binding and checking it against the registry's pinned key (REGISTRY §7.4 specs the *flow*; no impl ships the *backend*). **(B) the registration flow** — how a name gets *into* a registry: curated (v1, build-able now) + live `register-request` (the protocol, with admission policy + ownership-proof).
**Landed as:** EXTENSION-REGISTRY **§6a Peer-issued backend** (sibling section to §6 local-name — following the consolidation precedent where every concrete backend lives in the one EXTENSION-REGISTRY spec, not a separate file), binding to the REGISTRY substrate contract (REGISTRY §2, §3, §5). **Core impact:** none — no V7 change, no kernel handler, no new wire format. New error codes live in REGISTRY's code domain (V7 §3.3). **Part B.live (`register-request` + issuer-policy) is folded too** — §6a.9, with `open` (first-come) / `allowlist` / `manual` modes buildable now. The **only** deferred piece is the `domain-control` mode's challenge format, co-designed with the web-native dns-txt/well-known backends.
**Reads on landed:** REGISTRY §2/§3/§4/§5/§7/§8, NETWORK §6.5 (http-poll transport), SUBSTITUTE §7 (HTTP Mechanism A / Mode S), V7 §1.5 (peer-id), §5.2 (signature target-matching / invariant-pointer). **Companion to:** `PROPOSAL-UNIVERSAL-RESOLUTION`, `PROPOSAL-NAME-GRAMMAR`.

---

## 1. Motivation — the gap between "demo on pins" and "a real registry"

Today a fresh user can browse the curated set via **pinned bindings** (`out_of_band` trust — "the distro asserts it"). REGISTRY §7.4 describes the *verified* story — *"the registry signed this, and I verify against a key shipped with my build"* — but the **backend that performs that fetch+verify is not implemented**, and there is **no protocol for a publisher to get a name into the registry.** This proposal closes both. It is the difference between a hardcoded demo and *"stand up your own registry, register your own name, and have it verified by anyone who trusts you."*

The whole point is that **almost nothing here is new mechanism** — the resolve side is a structured fetch from a coral-reef tree + signature verification, both already built. We are wiring existing primitives into a backend + naming a registration protocol.

---

## 2. Part A — the resolve-side `peer-issued` backend

### 2.1 The insight: the backend is **trust logic over transport-agnostic reads** — it does NOT know about HTTP

A registry peer is **just a peer**; its bindings are ordinary entities in its tree. The backend reads them with the **normal `tree:get` / `content:get` machinery against the registry peer** — and it **does not know or care** whether that peer is reached over http-poll (a static coral-reef, the demo case) or a live socket. *How* you reach the registry is the **transport layer's** job (NETWORK §6.5 / §10; for a static coral-reef, the http-poll transport profile = SUBSTITUTE §7 Mode S, **built**). The backend says "read X from the registry peer"; transport decides the wire.

So the backend contains **only the registry-specific part: trust verification** (step 3). Everything else is a logical read that any peer already performs. This is exactly why `peer-issued` is a distinct backend from `local-name`: **same reads, different trust source** — a pinned *remote* registry key instead of *local* authority.

```
peer_issued.resolve(name, config):           ; config = the resolver_chain entry (REGISTRY §4)
  registry = config.backend_id                ; P_ecr — its PINNED key (trust root) + its transport profile.
                                              ;   The transport (http-poll for a static coral-reef) is resolved
                                              ;   by NETWORK §6.5 — NOT chosen or performed by this backend.
  norm     = nfc_normalize(name)              ; name-path safety, REGISTRY §6.3

  ; 1-2. transport-agnostic reads against the registry peer (the transport layer maps these to
  ;       http-poll GETs for a static coral-reef, or to a live socket if the registry runs one):
  binding_hash = tree:get(registry, "system/registry/binding/by-name/" + norm)   ; the by-name index, §2.2
  if binding_hash is null: return { status: "not_found", neg_ttl: config.neg_ttl }
  binding      = content:get(registry, binding_hash)        ; bytes hash-verified == binding_hash

  ; 3. VERIFY (REGISTRY §3 steps 1-3, §5) — the ONLY registry-specific logic; this IS the backend:
  sig = read(registry, "system/signature/" + hex(binding_hash))   ; invariant-pointer (V7 §5.2/§989)
  require sig.target == binding_hash
  require sig.signer == content_hash(canonical(registry.system_peer))   ; P4: signer is a hash; == the registry's
                                                                        ;   canonical system/peer content_hash (V7 §1.5/§5.2)
  require verify_crypto(sig, pinned_key_of(registry))       ; P1/P5: pinned key — for an Ed25519 identity-multihash
                                                            ;   registry peer-id the pubkey IS the peer-id digest (V7 §1.5);
                                                            ;   non-self-describing forms carry the pubkey in config
  require ("peer_issued:" + registry) in config.accepted_trust_anchors   ; P6: empty set ⇒ fail-closed (accept nothing)
  require not revoked(registry, binding_hash)               ; §2.3 (P7: by-target index)
  require binding.issued_at + binding.ttl > now()  (or ttl null)

  ; 4. surface
  return ResolutionResult {
    status: "resolved", binding: binding_hash,
    peer_id: binding.target_peer_id, transports: binding.transports,
    trust_anchor: "peer_issued:" + registry, ttl: binding.ttl, backend_id: registry }
```

The only **new convention** is the **by-name index** (§2.2). The reads are the **already-built, transport-agnostic** `tree:get`/`content:get` — the http-poll mapping is the transport layer's, not the backend's. The backend's actual substance is step 3: signature-verify against the pinned key + the REGISTRY §3 trust checks.

### 2.2 The by-name index (the one new artifact)

A peer-issued registry publishes, alongside each binding body at `system/registry/binding/{binding_hash}` (REGISTRY §3 universal location), a **name-keyed tree pointer**:

```
system/registry/binding/by-name/{nfc(name)}   →  bare system/hash of the current binding body
```

This is the **direct analog of local-name's `system/registry/binding/local-name/{name}` pointer** (REGISTRY §6.3 two-layer storage) — same pattern, different prefix (`by-name/` for issued bindings vs `local-name/` for local ones). It is the live name→hash index; the binding body carries the signature; supersedes-chain on the body is the rebind audit log. Served over http-poll like any tree node (with the `tree_leaf_suffix` disambiguator, SUBSTITUTE §2.2). Name-path safety per REGISTRY §6.3 applies (no `/`, no control chars, NFC) — domain-shaped names like `billslab.com` are fine (dots allowed).

**Offline / precedes path:** when the binding is pre-cached as a precede (REGISTRY §7), steps 1–2 read the local store instead of http; the verify step (3) is identical. So precedes and live-fetch converge on the same verification — precedes are just a warm cache.

### 2.2a The signed binding-manifest (OPTIONAL optimization — specified now, implemented later)

The by-name index (§2.2) is the **floor**: one tree pointer per name, one HTTP round-trip per name resolved. It is the direct localname analog, it is what **v1 implements**, and a conformant resolver MUST support it.

A registry with many names (or one optimizing for static-HTTP first-paint) MAY additionally publish a **single signed manifest** listing the whole name→hash index, so a consumer fetches the entire registry in **one round-trip + one signature check** instead of one per name. This is the **direct analog of SUBSTITUTE §2.4 / §7.2 `snapshot-manifest`** — same proven design pattern, registry-domain fields (the deltas vs SUBSTITUTE §2.4: `source_peer_id`→`registry_id`, `path_index`→`bindings`; SUBSTITUTE's content-store helpers `endpoint`/`content_count`/`root_hashes` dropped as not load-bearing for name resolution — `endpoint` already lives in the resolver-chain config). **We pin its shape here so no registry invents its own** (anti-fracturing); we do **not** require v1 to implement it.

```
type: "system/registry/binding-manifest"
data:
  registry_id:   <Base58 peer-id>            ; the issuing registry (MUST == signer)
  snapshot_at:   <ms-since-epoch>
  seq:           <u64>                        ; monotonic freshness (anti stale-replay)
  coverage:      "partial" | "complete"       ; default "partial"; see absence rule below
  bindings:      { <nfc(name)>: <system/hash of the binding body> , ... }   ; the name→hash index
  predecessor:   <system/hash>?               ; optional chain to the prior manifest's content hash (audit)
```

- **Published** at `{tree_url_prefix}/registry/manifest/current`. The extra `registry/` segment vs SUBSTITUTE's `/manifest/current` is **intentional**: a peer MAY serve both a SUBSTITUTE `snapshot-manifest` and a registry `binding-manifest` from the same tree, and the segment prevents the two `/manifest/current` artifacts from colliding. Same pattern, namespaced path.
- **Signature is MUST** — the index is an authority claim; anyone serving the URL can forge it, so hash-self-consistency proves nothing about authorship. Signature reachable at `system/signature/{hex(manifest.content_hash)}`, signed by `registry_id`, verified against the **pinned** registry key. Verification + `seq` freshness rule (`seq > cached` accept+supersede; `==` re-affirm; `<` reject `manifest_stale_seq`; first-ever any seq) are **identical to SUBSTITUTE §7.2** — reuse, don't re-derive. **No operator-trust override** — signature verification is mandatory (unlike SUBSTITUTE §4's unsigned-source knob; the receiver pinned the registry key for exactly this purpose, so a registry binding is never accepted unsigned).
- **Resolver behavior (the "both" pattern, SUBSTITUTE §7.4):** a resolver MAY fetch the manifest first; if present, current-`seq`, and signature-valid → read `bindings[nfc(name)]` directly (skipping the per-name pointer round-trip). On a missing / stale / signature-invalid manifest it **MUST fall through to the §2.2 per-name pointer**. The manifest never *replaces* the per-name index — it's a read optimization layered over it. The binding body is then fetched by hash (self-verifying content) exactly as in §2.1 step 2; the body's own signature (§2.1 step 3) remains the per-name-path authority.
- **Absence rule (name not in `bindings` of an otherwise-valid manifest) — NORMATIVE:** governed by `coverage`. **`partial` (default): a hit on a name not present is NOT a negative claim → the resolver MUST fall through to the §2.2 per-name pointer** (which is the source of truth for `not_found`). This is the floor and the only rule v1's per-name-only impls need: a registry that ships a manifest gets the **same** not-found semantics as one that doesn't. **`complete`: the registry asserts `bindings` lists every name it issues → absence IS authoritative; the resolver MAY return `not_found` without a per-name fall-through** (authenticated denial-of-existence, the DNSSEC-NSEC / TUF-complete-targets model). `complete` reclaims the fast-negative path; `partial` keeps the optimization strictly additive. **Manifest is never authoritative for absence unless `coverage: complete`.**
- **Per-name vs manifest divergence:** the per-name pointer is **live truth**; the manifest commits a `bindings` slice as of its `seq` and the per-binding TTL is the freshness ceiling. A divergence (manifest at `seq T` points at a body the per-name pointer has since superseded at `T+1`) resolves via the **binding body's own TTL / revocation / supersedes-chain checks** (§2.1 step 3, §2.3) — an expired or revoked body is rejected and the chain advances. The manifest **never wins over a per-name read of a still-valid newer binding**.
- **Error code:** `manifest_signature_invalid` / `manifest_stale_seq` reused from SUBSTITUTE's code domain (no new REGISTRY codes; these are explicitly part of SUBSTITUTE's owned domain per §7.2/§9).

**Size bound + internet scale (the flat manifest's ceiling — pinned so big registries don't fracture):** a manifest is a single entity, so it is bounded by the V7 §4.10(a) max-payload floor (`413 payload_too_large`; 16 MiB cohort default). At ~50–100 B per `{name → hash}` entry that caps a **flat** manifest at a few hundred thousand names. This is a real ceiling, and the design **does not** rely on a flat manifest scaling past it. Three layers, by registry size:

1. **Resolution is internet-scale at the floor, unconditionally.** The per-name index (§2.2) has **no aggregate size limit** — a resolver fetches only `by-name/{name}` for the one name it wants, so a billion-name registry resolves any single name in the same 2–3 fetches as a four-name one. **This is the DNS model** (you query one name, you never download the zone). Internet-scale *resolution* is the v1 floor, not a future feature.
2. **The flat manifest is the small/moderate-registry bulk optimization** (≲ hundreds of thousands of names) — one fetch + one signature for the whole index. The demo (a handful of names) sits comfortably here.
3. **Past the §4.10 ceiling: shard via delegation — sanctioned direction, detailed format deferred.** The internet-scale form is a **delegated/sharded manifest** — a top manifest whose entries point at **sub-manifest hashes** by name-prefix, each independently signed — exactly **DNS zone delegation / TUF delegated-targets / APT `Release`→`Packages`**. The pinned format already expresses this (an entry value is a `system/hash`; a sub-manifest is just another `binding-manifest`). The strongest form is a Merkle **transparency log** with O(log n) inclusion proofs (Go sumdb / CT), which folds together with the §5 non-equivocation hardening — same mechanism. **We do not design delegated-targets here** (no internet-scale registry exists yet; that's over-speccing a deferred feature), but the **direction is pinned** so the first large registry extends *this* shape rather than inventing a parallel sharding scheme. Lands as a follow-on when a registry's name-count actually approaches the flat ceiling.

**Implementation status:** the **shape is specified (floor: pinned); the v1 implementation is the per-name index only.** Manifest publish + consume is a follow-on the same way SUBSTITUTE Mechanism B is optional on top of Mechanism A — pick it up when a registry's name-count makes per-name round-trips the bottleneck. No fracturing risk in the interim because the format is fixed here.

### 2.3 Revocation check

`revoked(registry, binding_hash)` checks for a `system/registry/revocation` entity targeting the binding (REGISTRY §3.1) on the registry peer's tree (transport-agnostic — same read machinery); if one verifies against `registry`, the binding is excluded and `meta_resolve` advances.

**By-target revocation index (P7 ruling — NORMATIVE):** revocations are looked up via a direct index `system/registry/revocation/by-target/{hex(binding_hash)}` → the revocation entity (presence = revoked), **not** a scan over `system/registry/revocation/`. This is the **revocation analog of the by-name index (§2.2)** — O(1) lookup vs O(N) scan, same anti-internet-scale reasoning as the manifest §2.2a. The cohort already reads this path; ruling makes it the pinned convention so impls don't fork the revocation lookup. The operator CLI (§3.2) grows a `revoke` subcommand that publishes the revocation entity + its `by-target/` pointer (signed by `K_registry`). For a live registry, subscription-driven invalidation is the MAY optimization on top (REGISTRY §3.1).

### 2.4 What Part A needs (mostly built)

| Piece | Status |
|---|---|
| `tree:get` / `content:get` against the registry peer (transport-agnostic; transport layer maps to http-poll Mode S for a static coral-reef) | **built** (SUBSTITUTE §7, NETWORK §6.5.3.1) |
| signature verify + invariant-pointer fetch | **built** (V7 §5.2) |
| binding entity + trust checks + accepted_trust_anchors | **built** (REGISTRY §3, §5) |
| **`by-name/` index convention** | **new — this proposal** (trivial: a tree-pointer prefix, local-name's pattern) |
| **the backend handler wiring** (compose the above into `peer_issued.resolve`) | **new — this proposal** (the unblock; ~the local-name backend's size) |

### 2.5 Cohort spec-doubt rulings (P1–P7)

Rust filed seven doubts during impl (`entity-core-rust/docs/SPEC-PROBLEMS-PEER-ISSUED-REGISTRY.md`); Go corroborated. All ruled here (none punted); folded inline above where they touch normative text.

| # | Doubt | Ruling |
|---|---|---|
| **P1** | Where does `pinned_key_of(registry)` live — derived from the peer-id, or in resolver-chain config? | **Both, by peer-id form.** For an **Ed25519 identity-multihash** registry peer-id the pubkey **is** the peer-id digest (V7 §1.5 v7.65 canonical form) — config carries only the peer-id, key is derived; this is the v1 path Go+Rust already share. For a **non-self-describing form** (SHA-256-form / Ed448) the resolver-chain entry MUST carry the pubkey explicitly (or a locally-resolvable `system/peer`). Config-carried-key for non-derivable forms is **deferred** (no v1 demo needs it); v1 floor is the derivable form (→ P5). |
| **P2** | Backend `not_found` vs meta-resolver `chain_exhausted` — which is the vector's contract? | **Two levels, both correct.** The **backend** returns `not_found + neg_ttl` (backend-scoped; this is what `OFFLINE-NOTFOUND-1` asserts). The **meta-resolver** returns `chain_exhausted` when the *whole* chain misses (REGISTRY §4.1), carrying the aggregated `neg_ttl`. No conflict — name the level on the vector. |
| **P3** | Is `neg_ttl` a first-class field or a hint-bag entry? | **First-class, on the negative result.** `neg_ttl` is a defined optional top-level slot on the `not_found` result (the resolver *acts* on it — negative caching — so it is not opaque backend pass-through). Reuses REGISTRY's existing `neg_ttl` hint shape; not buried in a generic hints map. |
| **P4** | `sig.signer == registry` — type mismatch (signer is a hash; `registry` is a peer-id). | **Wording fix (no behavior change), folded in §2.1 step 3.** `sig.signer` equals `content_hash(canonical(registry's system/peer))` (V7 §1.5 / §5.2 target-matching). |
| **P5** | Is Ed25519 / canonical-multihash-only the intended v1 floor? | **Yes — intended v1 floor.** The registry trust anchor is an Ed25519 identity-multihash peer-id (pubkey derivable). Consistent with the core §9.1 floor and [[project_dual_algorithm_is_keystone_requirement_not_core_floor]]. Other forms ride the P1 config-carried-key path (deferred). Stated explicitly so a future impl doesn't re-derive a different answer. |
| **P6** | Empty `accepted_trust_anchors` — accept-anything or reject-everything? | **Fail-closed (reject everything).** An empty anchor set accepts no peer-issued binding. Matches the resolver chain's general fail-closed posture and the security default (an unconfigured trust set never silently accepts). Folded in §2.1 step 3. |
| **P7** | Is `revocation/by-target/{hex}` a normative convention or a pragmatic optimization? | **Normative** (folded in §2.3). The revocation analog of the by-name index — O(1) lookup, no fork risk. Operator CLI grows a `revoke` subcommand. |

**Net:** P4/P6/P7 change normative text (folded above); P1/P5 pin the v1 Ed25519-identity-multihash floor + name the deferred config-key path; P2/P3 are layering clarifications confirming the cohort's reading. No new core/V7 change; no new wire format.

---

## 3. Part B — registration: how a name gets into a registry

### 3.1 Two registry operating modes

A registry is just a peer (REGISTRY §1 position 4). It operates in one of two modes, and registration differs:

- **Curated / static (v1, the release default).** The operator decides what's in the registry, signs each binding with the registry key, and publishes the static coral-reef tree. **No live protocol** — registration is operator tooling. This is how the **Entity Church Registry ships for release.**
- **Live.** The registry runs a handler that accepts `register-request` from publishers, applies an admission policy, signs + publishes on approval. This is what lets a publisher **self-register**.

### 3.2 Curated registration (v1 — build-able now)

The only requirement is a signing tool + the publish layout:

```
registry-issue-binding(name, target_peer_id, transports, ttl) :          ; operator tool, holds K_registry
  body = system/registry/binding { name, kind: "peer-issued", target_peer_id, transports, issued_at: now, ttl }
  sig  = sign(K_registry, body.content_hash)                              ; → system/signature, target = body.content_hash
  publish body at  system/registry/binding/{body.content_hash}
  publish sig  at  system/signature/{hex(body.content_hash)}             ; invariant-pointer (V7 §5.2)
  set pointer  system/registry/binding/by-name/{nfc(name)} → body.content_hash

registry-revoke-binding(binding_hash, reason) :                          ; operator tool — P7 revoke subcommand
  rev = system/registry/revocation { target: binding_hash, reason, issued_at: now }
  sig = sign(K_registry, rev.content_hash)
  publish rev  at  system/registry/revocation/{rev.content_hash} + sig at system/signature/{hex(rev.content_hash)}
  set pointer  system/registry/revocation/by-target/{hex(binding_hash)} → rev.content_hash
```

This produces a standard REGISTRY §3 binding; Part A resolves it. **No new entity, no protocol** — just operator discipline + the by-name pointer. Ship this for release.

### 3.3 Live registration (the protocol)

For publishers to self-register against a live registry. The request entity:

```
type: "system/registry/register-request"
data: {
  name:           <string>,                 ; requested name (name-path safety per REGISTRY §6.3)
  target_peer_id: <Base58 peer-id, V7 §1.5>,; what the name should resolve to
  transports:     [<endpoint per NETWORK §6.5>],
  requested_ttl:  <ms duration | null>,
  nonce:          <bytes>,                   ; anti-replay (§5)
  issued_at:      <ms-since-epoch>
}
```

The request **MUST carry a `system/signature` by `target_peer_id`** (target-matching, V7 §5.2; reachable at `system/signature/{hex(request.content_hash)}`). This is **ownership-proof layer 1** (§3.4). Handler op on the registry peer:

```
system/registry/peer-issued:register-request(request) → binding_hash | rejection
  1. verify request signature by target_peer_id          ; layer-1 proof (holds the key)
  2. apply issuer-policy admission (§3.4)                 ; layer-2 entitlement → approve | reject | queue
  3. on approve: registry-issue-binding(...) (§3.2)       ; signs with K_registry, publishes, sets by-name pointer
     return binding_hash
  4. on reject:  return error (REGISTRY code domain)      ; e.g. name_taken / not_entitled / policy_rejected
  5. on queue:   return status "pending_review"           ; manual mode
```

Two follow-on ops:

```
system/registry/peer-issued:revoke-request(binding_hash, reason) → ()    ; by the registrant or operator; emits §3.1 revocation
system/registry/peer-issued:renew-request(binding_hash, ttl) → new_binding_hash   ; supersedes-chain
```

### 3.4 Admission policy (the registry's own decision) + ownership-proof

**The substrate gates no name claims (REGISTRY §5)** — but a *real registry* must decide what it signs. That decision is the registry's **own policy**, configured, not mandated (per `[[feedback_expose_knobs_dont_pick_values]]`):

```
type: "system/registry/issuer-policy"          ; registry-local config
data: {
  mode:            "open" | "allowlist" | "manual" | "domain-control",
  allowlist:       [<peer-id>]   | null,        ; for "allowlist"
  name_constraints: <glob | null>,              ; e.g. only issue "*.lab" names
  default_ttl:     <ms | null>,
  require_domain_control: bool                  ; for domain-shaped names (see below)
}
```

**Two layers of proof, separable:**
- **Layer 1 — peer-id control (always).** The request is signed by `target_peer_id` → proves the requester holds that key (the binding will point where the requester intends; no one else can register *their* peer-id under a name).
- **Layer 2 — name entitlement (policy-defined).** Whether *this requester* may have *this name*. Modes: `open` (first-come), `allowlist`, `manual` (review queue), `domain-control`. **For domain-shaped names** (`billslab.com`), `domain-control` is the natural policy: the registry requires the requester to prove control of the DNS domain (a DNS TXT challenge or an HTTPS `.well-known` file) **before** issuing — which is exactly **Path B's verification** (GUIDE §6 / the coral-reef stress test) reused as a registration gate. This is where anti-squatting lives (REGISTRY §12: "per-backend concern") — and it ties the registry path to the web-native path.

`registry-issue-binding` (the internal signing act) is gated by the registry's own cap `system/capability/registry-issue-binding`, held by the policy logic / operator — never by arbitrary callers. `register-request` is the *external* surface; the policy decides.

### 3.5 Entities + caps (additions, all in REGISTRY's namespace)

| Entity | Purpose |
|---|---|
| `system/registry/register-request` | the publisher's registration request (§3.3) |
| `system/registry/issuer-policy` | registry-local admission config (§3.4) |
| (reuses `system/registry/binding`, `system/registry/revocation`, `system/signature`) | — |

| Cap | Gates |
|---|---|
| `system/capability/registry-request-binding` | who may submit a `register-request` (open mode → granted broadly; allowlist → narrow) |
| `system/capability/registry-issue-binding` | the internal sign+publish act (registry operator / policy logic only) |
| `system/capability/registry-manage-issuer-policy` | who may edit `issuer-policy` |

---

## 4. The full scenario, plugged in (the stress test, resolved)

`billslab.com` resolved via the Entity Church Registry, with this proposal landed:

1. **Registration (curated, §3.2):** ECR operator runs `registry-issue-binding("billslab.com", P_bl, [http-poll @ billslab.com], ttl)` → signs with `K_ecr`, publishes body + signature + `by-name/billslab.com` pointer in ECR's tree at `entitychurchregistry.org`. *(Or, live §3.3: BL submits a `register-request` signed by `P_bl`; ECR's `domain-control` policy challenges BL to prove it owns `billslab.com` via DNS/well-known; on success ECR issues.)*
2. **Resolution (Part A):** user's `peer_issued.resolve("billslab.com")` → fetch `by-name/billslab.com` from `entitychurchregistry.org` → binding hash → binding body → verify signature against the **pinned** `P_ecr` key → `ResolutionResult { P_bl, transports }`.
3. **Reach + fetch:** connect to `billslab.com` over http-poll → `published-root` → tree→content → render (SUBSTITUTE §7).

Now it's *verified*, not asserted — and a community member can stand up their own registry the same way; users who pin its key resolve through it.

---

## 5. Security analysis (baseline — not deferred, per standing rule)

| Surface | Risk | Mitigation |
|---|---|---|
| **Name squatting** | attacker registers `billslab.com` they don't own | **registry policy** (§3.4 layer 2) — `domain-control` for domain names, allowlist/manual otherwise; the substrate intentionally leaves this to the registry, receiver trust is the backstop |
| **Wrong-peer binding** | binding `name → P_attacker` | layer-1 proof: request signed by `target_peer_id`; *and* the resolved binding is signed by the trusted registry — a receiver only accepts what *their* pinned registry signed |
| **Replay** | resent `register-request` re-registers / overwrites | `nonce` + `issued_at` window; registry tracks seen nonces per requester; renew/revoke are explicit ops |
| **Registry key compromise** | attacker signs false bindings | out of scope of resolution (key hygiene); mitigated by receiver pinning + revocation + (future) registry-vouches-for-registry corroboration (REGISTRY §3 `issuer_attestation`) |
| **Stale/rollback** | registry serves an old (pre-revocation) binding | `ttl` + revocation check (§2.3) + (live) subscription invalidation; `seq`-style freshness as in SUBSTITUTE manifests is a follow-on if needed |
| **Downgrade to pins** | falling back to `out_of_band` pin when verify fails | **fail-closed** (REGISTRY §4.1 step 4) — a failed peer-issued verify advances the chain; a pin only matches if explicitly configured. No silent downgrade. |
| **Confused deputy** | n/a for resolve | the consumer fetches + verifies with its *own* pinned trust root; no third-party authority is exercised. (Contrast the deferred resolve-delegation extension, which *does* carry this risk and forwards cap-chains — GUIDE §5.) |

Open item for cohort: the `domain-control` proof (§3.4 layer 2) reuses Path B (DNS/well-known) — confirm the challenge format alongside the web-native backend proposals so they share one domain-control mechanism rather than two.

---

## 6. Conformance — `REG-PEERISSUED-*` vectors

- `REG-PEERISSUED-RESOLVE-1` — by-name index → binding → verify against pinned key → `resolved` (the happy path; assert each intermediate per the meta-rule).
- `REG-PEERISSUED-VERIFY-FAIL-1` — binding signed by a non-pinned key → rejected, chain advances (NOT accepted, NOT downgraded to pin).
- `REG-PEERISSUED-REVOKED-1` — valid binding with a verifying revocation → excluded.
- `REG-PEERISSUED-EXPIRED-1` — `issued_at + ttl < now` → excluded.
- `REG-PEERISSUED-PRECEDE-1` — same binding resolved from precedes (offline) → identical verify + result as live-fetch.
- `REG-PEERISSUED-OFFLINE-NOTFOUND-1` — name not in by-name index → `not_found` with `neg_ttl`.
- `REG-PEERISSUED-MANIFEST-1` *(gates the §2.2a optional manifest; runs when an impl ships manifest support — locks the format)* — signed binding-manifest, current `seq`, valid signature → resolver reads `bindings[name]` directly; stale `seq` → `manifest_stale_seq` + fall through to per-name pointer; non-pinned signer → `manifest_signature_invalid` + fall through. Asserts manifest never overrides a per-name miss and a bad manifest is non-fatal.
- `REG-PEERISSUED-MANIFEST-ABSENCE-1` *(locks the absence rule §2.2a)* — name absent from a valid current-`seq` manifest: under `coverage: partial` (or absent) → resolver **MUST fall through** to the per-name pointer (which decides `not_found`); under `coverage: complete` → resolver MAY return `not_found` authoritatively (no fall-through). Asserts a manifest-shipping registry has identical not-found semantics to a per-name-only one unless it explicitly claims completeness.
- `REG-REGISTER-PROOF-1` — `register-request` with a signature NOT by `target_peer_id` → rejected (layer-1).
- `REG-REGISTER-POLICY-1` — `register-request` rejected by `allowlist` policy → `not_entitled`; accepted when allow-listed → binding issued + resolvable.
- `REG-REGISTER-REPLAY-1` — replayed request (seen nonce) → rejected.
- `REG-REGISTER-DOMAINCTRL-1` — domain-shaped name without domain-control proof under `domain-control` mode → rejected; with proof → issued.

---

## 7. Built vs owed / rollout — the implementation handoff

**The release ask to the peers (decided, ready to build):**

1. **Part A — the `peer_issued` resolve backend.** In `ext/registry/`, write a `peerissued/` package as the **twin of the existing `localname/` backend** — same backend shape, **trust-verification over transport-agnostic reads** instead of in-memory authority (per §2.1: the backend issues `tree:get`/`content:get` against the registry peer and verifies the signature against the pinned key; it does **not** know or perform HTTP — the transport layer maps the reads to http-poll for a static coral-reef). Implements `peer_issued.resolve(name, config)` (§2.1, ~6 steps, all reusing built primitives: `tree:get`/`content:get`, signature verify + invariant-pointer fetch, REGISTRY §3 trust checks). Roughly the size of `localname/`. **This is the verified-demo unblock.**
2. **The one new convention — the by-name index (§2.2).** `system/registry/binding/by-name/{nfc(name)}` tree pointer per registry; the direct analog of local-name's `local-name/{name}` pointer. **v1 implements this.** The signed manifest (§2.2a) is **fully specified but NOT a v1 build item** — peers do not implement it now; the format is pinned (SUBSTITUTE §2.4/§7.2 analog) so it's ready and fracture-proof when a many-name registry needs it.
3. **Part B.curated — the operator signing tool (§3.2).** A small operator CLI (`registry-issue-binding`) that signs a binding with the registry key and publishes body + signature + by-name pointer. **No protocol — ~a 50-line tool, not a feature.** This is how the Entity Church Registry ships for release.

That trio makes the registry **verified, not pinned** — the headline — and it's mostly wiring existing primitives. Workbench-go already demonstrated it's a **drop-in at the name rung**: Part A returns the same `ResolutionResult` shape as local-name, so the SDK `resolve()` seam consumes it **unchanged** (no downstream change).

**Follow-on (NOT release-blocking) — Part B.live (§3.3):** the `register-request` handler + issuer-policy + ownership-proof, letting publishers self-register. Carries the deferred §8 Q2–Q4 (notably the domain-control challenge, shared with web-native backends). Lands as its own cycle.

**Cohort sequencing:** Go reference backend first (convention), Rust + Python catch up — **all three landed 3-way GREEN, with the 6 Part-A `REG-PEERISSUED-*` vectors (§6) passing per-impl.** No cross-impl Keystone byte-gate is owed for Part A: the cross-impl-divergent primitives (CBOR encoding, Ed25519 verify) are already locked elsewhere, and resolve is handler logic over them. An optional real-wire http-poll end-to-end against a served ECR coral-reef would validate that the *wire path lights up* (demo validation) — useful before the live demo, not a correctness gate. `REG-REGISTER-*` vectors gate Part B.live.

---

## 8. Open questions — dispositions (cgid-10-231)

**Decision rule applied (arch direction):** anything required for the verified demo is decided now toward the simplest demo-ready option; anything not demo-blocking with an unsettled shape is explicitly deferred ("come back and fix later"), not guessed at.

1. **by-name index path — RESOLVED: spec both, implement the index for v1.** Per-name pointers (`system/registry/binding/by-name/{nfc(name)}`, §2.2) are the **floor and the v1 implementation** — the localname analog, demo-ready, incremental. The **signed manifest is now fully specified (§2.2a)** rather than left to each registry's invention — the same proven SUBSTITUTE §2.4/§7.2 design pattern with registry-domain fields (+ a `coverage` flag governing the absence rule), so there is **one** manifest format, never a fractured per-registry one. The manifest's *implementation* is the deferred half (optional optimization on top, picked up when a registry's name-count warrants), exactly as SUBSTITUTE Mechanism B sits on Mechanism A. **Deferring the impl ≠ omitting the design** — the design is pinned; only the v1 build is scoped down. Per-name pointers ship; manifest format is locked for whoever needs it.
2. **domain-control challenge format (§3.4) — DEFERRED (not demo-blocking).** Domain-control is a **Part B.live** admission gate only; the demo registers via **curated mode** (§3.2, operator signs by hand — no challenge needed). Do **not** pick a format unilaterally; settle it jointly with the web-native (dns-txt/well-known) backend proposals so there is **one** domain-control mechanism, not two. Lands with the Part B.live cycle.
3. **register-request transport — DEFERRED (Part B.live).** Sync handler reply vs async submit→poll for `manual` mode. Lean (for that cycle): support both; `pending_review` status entity for manual. Not release-blocking.
4. **issuer-policy richness — DEFERRED (Part B.live).** Keep `issuer-policy` as simple registry-local config modes now; richer policy language later when a concrete driver emerges. Not release-blocking.

**Net:** the only question that touched the release path (Q1) is decided. Q3/Q4 are now folded into §6a.9 (register-request transport: support both / `pending_review` for manual; issuer-policy: simple modes now). **Q2 (domain-control challenge format) is the single remaining unresolved item — and it is NOT held open by this proposal.** It is a *forward dependency* of the web-native `dns-txt`/`well_known_url` backend proposals (one domain-proof mechanism, not two), and lands when those are designed.

**Proposal status — CLOSED / Ratified.** Everything this proposal owns is folded into EXTENSION-REGISTRY (§6a, including live registration `open`/`allowlist`/`manual` at §6a.9). This proposal is **not** kept open for `domain-control`: that loose end cannot be resolved here (it is co-dependent on the web-native backend design), so holding this proposal open would track nothing actionable. We come back to `domain-control` when we get to the web-native backends — tracked at EXTENSION-REGISTRY §6a.9.1 + §12 and the completeness banner.

---

## Spec homes

- substrate contract this binds to: EXTENSION-REGISTRY §2 (resolver-handler), §3 (binding), §5 (caps/trust), §7 (precedes), §8 (registry-as-coral-reef)
- fetch primitives reused: SUBSTITUTE §7 (HTTP Mechanism A / Mode S), NETWORK §6.5.3.1 (tree-leaf = hash over HTTP)
- signature carriage: V7 §5.2 / §989 (target-matching + invariant-pointer)
- the resolve seam that consumes this: PROPOSAL-UNIVERSAL-RESOLUTION
- name routing to this backend: PROPOSAL-NAME-GRAMMAR
- the scenario: reviews/HANDOFF-COHORT-RESOLUTION-CORAL-REEF-STRESS-TEST-cgid-10-231
