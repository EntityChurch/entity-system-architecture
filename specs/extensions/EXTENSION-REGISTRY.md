# EXTENSION-REGISTRY

**Version**: 1.2
**Status**: Active
**Tier:** Operational — Tier 2b (network), per `core-protocol-domain/specs/SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.

> ## ⚠ COMPLETENESS — this extension is **v1, NOT finished**
> The substrate + the two concrete v1 backends are landed and implemented. Several pieces are **specified-and-deferred** or **not-yet-designed**. Do **not** read "Landed" as "complete."
>
> **✅ Landed + implemented (v1):** resolver substrate (§2–§5); local-name backend (§6); peer-issued resolve + curated registration (§6a.1–§6a.8).
>
> **🟡 Design folded, build in flight / deferred:** peer-issued live registration `open`/`allowlist`/`manual` (§6a.9 — buildable now, cohort dispatch in flight); signed binding-manifest impl (§6a.7 — format locked, impl deferred).
>
> **🔴 NOT yet designed — outstanding work before this extension is "done":**
> - **`domain-control` DNS-challenge format** (§6a.9.1) — must be ONE mechanism shared with the web-native `dns-txt`/`well_known_url` backends; settles with *that* proposal, not here.
> - **Other backends** — did-web, dns-txt, dht, consensus-anchored (§12) — each its own proposal.
> - **Aggregator Mode-A federation** (§8.2, v1-deferred); **outbound DID/DNS bridge** (§12, v1-deferred).
>
> Tracking: the deferred items are named in §6a.9.1, §8.2, and §12. The `PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND` that produced §6a is **closed/ratified** — the `domain-control` loose end is a *forward dependency* of the web-native backend work, not an open question of that proposal.

---

## §1 Concept

A **registry** is a function: `name → (peer_id, bootstrap_endpoints, attestations, trust_anchor, ttl)`. It indirects from human-shareable names to cryptographic peer identities + dial-able endpoints.

This extension specifies the registry substrate (§2 resolver-handler contract; §3 binding entity; §4 resolver-config; §5 capability model) **and** ships the local-name backend (§6) as the v1 concrete backend that exercises the substrate end-to-end. Additional backends (peer-issued, DID:web, DNS-TXT, DHT, consensus-anchored, aggregator) compose on the substrate and ship in their own proposals; this spec defines the contract they bind to.

Five positions are load-bearing:

1. **Backends are parallel.** A deployment installs whichever backends fit its trust model + use cases. No hierarchy is implied by the substrate.
2. **The substrate gates no name claims.** Anyone can publish a binding entity claiming any name. Whether a receiver TRUSTS that binding is the receiver's policy.
3. **Mechanism is the mechanism.** The substrate exposes the resolver contract + binding shape + trust-evidence machinery + cryptographic verification + composition with RELAY and CONTENT. Deployments configure which backends, what trust anchors, what name formats. Substrate provides knobs.
4. **Registry is just a peer.** A registry peer is a peer publishing `system/registry/binding/...` entities. Mode S relay can host its tree. No special infrastructure role.
5. **Bootstrap-with-precedes.** Distributions ship with pre-configured resolver-config + pre-cached precedes; users inherit by accepting the build; override is structural.

**Discovery (peer-finding) is a distinct concern** — finding peers you don't know exists. That's a separate extension (DISCOVERY, currently DRAFT proposal). Registry resolves names you already know to peers.

---

## §2 The resolver-handler contract

### §2.1 The handler operations

The substrate exposes two handler operations across all backends:

```
system/registry:resolve(name, [hints]) → ResolutionResult
system/registry:invalidate-cache(name | null) → ()    ; null = flush all
```

`ResolutionResult` is:

```
{
  status:        "resolved" | "not_found" | "chain_exhausted",
  binding:       <hash of system/registry/binding entity>,
  peer_id:       <Base58 peer-id per V7 §1.5>,
  transports:    [<endpoint per NETWORK §6.5>],         ; reachable endpoints, ordered
  attestations:  [<hash of supporting attestation entities>],
  trust_anchor:  <variant identifying which backend resolved>,
  ttl:           <ms-since-epoch duration | null>,      ; positive-result cache hint
  neg_ttl:       <ms-since-epoch duration | null>,      ; OPTIONAL negative-cache hint on not_found / chain_exhausted; SHOULD per backend
  backend_id:    <peer_id_hash | identifier>            ; which backend produced this
}
```

All durations in `ResolutionResult` (and timestamps throughout this spec) are **milliseconds since Unix epoch (UTC, signed int64)** unless otherwise stated — aligned with V7 cap-expiry convention.

**Wire encoding (MUST; erratum).** On the wire, `ResolutionResult` is the entity type **`system/registry/resolution-result`** with the data fields above carried **flat** under `data`. Implementations MUST NOT wrap under another envelope type (e.g. `system/protocol/status` with a `{result: {...}}` payload). The prose name "ResolutionResult" throughout this spec refers to this on-wire shape. Caught when three impls picked three encodings in the cross-impl run; pinned in place in the registry/discovery cross-impl-run absorption (Ruling-3).

`hints` is an optional opaque payload for the backend (e.g., DNS resolver override; DHT bootstrap node list). Backends MAY ignore.

### §2.2 The meta-resolver pattern

Multiple backends installed simultaneously; the meta-resolver dispatches by name format and/or consults the configured resolver chain:

```
meta_resolve(name, hints) → ResolutionResult
  for each backend in resolver_chain (per resolver-config):
    if backend.matches(name):
      r = backend.resolve(name, hints)
      if r.status == "resolved":
        validate(r)          ; signature + trust-anchor + receiver policy
        if validated:
          return r
        else:
          continue          ; failed validation; try next
      ; otherwise advance
  return { status: "chain_exhausted" }
```

The meta-resolver is convention; each backend is independent and registers via the standard handler-registration mechanism.

### §2.3 `:resolve` in the transport-fallback path

**REGISTRY isn't just first-contact name resolution.** `:resolve` is part of the transport-fallback loop. When a cached transport endpoint for a peer fails, the dispatcher re-resolves the **original name** (stored in session state per NETWORK §6.6) to refresh endpoints, then retries the transport:

```
try(transport T1) → fail → try(T2) → fail
  → registry:resolve(original_name_from_session)   ; name-keyed, not peer-id-keyed
  → refresh transports
  → try(T3 from refreshed set) → ...
```

`:resolve` is name-keyed by contract. Reverse `peer_id → binding` lookup is address-discovery and is scoped to EXTENSION-IDENTITY per §12. NETWORK §6.6's session entity holds the original name that produced the current peer-id binding; the transport-fallback loop re-resolves THAT name. Re-resolves from the fallback loop SHOULD be tagged `is_fallback_reresolve: true` when logged (see §11.1) — they are NOT counted toward the resolution-log's per-call sampling budget.

The `ResolutionResult` shape (§2.1) returns `transports` + `ttl` — the right output for this loop. Consumers MUST understand that `:resolve` is invoked **on transport failure, not only on cold-start.** TTL-bounded caching of resolutions interacts with this: a resolution MAY be re-fetched before its TTL expires if the cached endpoints stop working.

The full loop mechanics (when to re-resolve, how many retries, demotion of failed endpoints) are operational integration; the substrate's contract is the `ResolutionResult` shape + the "invoked on failure" semantic + the name-keyed re-resolve discipline.

### §2.4 Trust-anchor variants

Each backend's `trust_anchor` identifies what authority chain the receiver must validate against. Built-in variants:

| Variant | Meaning |
|---|---|
| `self_certifying` | name IS peer-id; trivial; no resolution authority needed |
| `local_name` | user's own local assignment; no global meaning |
| `dns_txt:{zone}` | DNS TXT record at zone; **unauthenticated unless qualified** (raw DNS TXT carries no integrity; DNSSEC-signed / DoH / DoT qualifications can be expressed as `dns_txt:{zone}:dnssec`, etc.; receiver policy distinguishes) |
| `well_known_url:{domain}` | well-known URL at domain; HTTPS PKI authority |
| `did_web:{domain}` | W3C did:web; same authority as well-known-url |
| `peer_issued:{registry_peer_id}` | a registry peer signed the binding; trust the peer |
| `consensus_anchored:{chain}:{block}` | blockchain consensus; trust the chain |
| `out_of_band` | explicit user input; also used for pinned bindings (§5.5) |

Backends MAY define additional variants. Receiver policy decides which variants are acceptable for which operations.

### §2.4.1 Vocabulary mapping table (cohort-convergence pin)

Three concept axes touch the same backend identity; the canonical mapping (all hyphen-spelled):

| `binding.kind` | `resolver-config.backend_kind` | `trust_anchor` variant |
|---|---|---|
| `self-certifying` | `self-certifying` | `self_certifying` |
| `local-name` | `local-name` | `local_name` |
| `dns-txt` | `dns-txt` | `dns_txt:{zone}` |
| `well-known-url` | `well-known-url` | `well_known_url:{domain}` |
| `did-web` | `did-web` | `did_web:{domain}` |
| `peer-issued` | `peer-issued` | `peer_issued:{registry_peer_id}` |
| `consensus-anchored` | `consensus-anchored` | `consensus_anchored:{chain}:{block}` |
| `out-of-band` | `out-of-band` | `out_of_band` |

Hyphenation is normative. `trust_anchor` variants use underscores per V7 enum convention because they're encoded as discriminator strings, not type names; `binding.kind` and `backend_kind` are field-type enums with hyphenated values matching the spec convention.

---

## §3 The binding entity type

```
type: "system/registry/binding"
data: {
  name:               <string>,                ; the user-facing name
  kind:               "self-certifying"        ; binding mechanism (see §2.4.1 vocab table)
                    | "local-name"
                    | "dns-txt"
                    | "well-known-url"
                    | "did-web"
                    | "peer-issued"
                    | "out-of-band"
                    | "consensus-anchored",
  target_peer_id:     <Base58 peer-id per V7 §1.5>,  ; the identity, NOT a content-hash
  transports:         [<endpoint per NETWORK §6.5>], ; preferred order
  issued_at:          <ms-since-epoch>,
  ttl:                <ms duration | null>,    ; null = sticky until revoked
  supersedes:         <system/hash, BARE | null>,  ; per ATTESTATION supersedes chain
  issuer_attestation: <system/hash, BARE | null>,  ; for peer-issued: the registry's authority cert
  metadata:           <opaque object | null>   ; backend-specific
}
```

Stored at `system/registry/binding/{binding_hash}` (universal — every kind, including local-name bodies, lives here). Aggregators MAY re-publish at their own path (see §8). Local-name bindings ADDITIONALLY have a tree pointer at a name-keyed path (see §6.3).

**Hash-field shape.** All bare-hash fields in this entity — `supersedes`, `issuer_attestation`, and `system/registry/revocation.revokes` — are **bare `system/hash`** values (33 bytes, `0x00`+digest), NOT wrapped in any envelope or object. Conformance: impls MUST NOT wrap or double-encode them.

**`target_peer_id` is an identity, not a content-hash.** It is the Base58-encoded peer-id per V7 §1.5 multikey form (key_type ‖ hash_type ‖ digest, encoded). Self-certifying naming uses this string directly (`name == target_peer_id`), NOT `hex()` of a hash. This is the V7 §1.5 alignment pin.

**Self-certifying bindings** have no issuer signature; `name == target_peer_id`; trivially verified by checking that `name` Base58-decodes to a valid V7 §1.5 peer-id structure.

**Local-name bindings** have no issuer signature; the user is the trust source (see §6).

**All other kinds MUST carry an `issuer_signature` `system/signature` entity per V7 §5.2 / §975**, carried in the envelope's `included` map with:
- `data.target == binding.content_hash`
- `data.signer == issuer_peer_id` (the registry peer / DNS authority / etc.)

Per V7 §989 invariant-pointer carriage, the signature MUST also be reachable via `tree:get system/signature/{hex(binding.content_hash)}` so cross-peer fetches (e.g., the §7.4 http-poll ESR flow) can verify without round-tripping to the issuer. This is the V7 §5.2 / §833 refless target-matching contract — NOT a `refs:` block.

Receiver verifies by:
1. Locating the `system/signature` entity via target-matching (`data.target == binding.content_hash`) in `included`, OR by invariant-pointer fetch at `system/signature/{hex(binding.content_hash)}` if not inlined.
2. Verifying signature cryptographically against the issuer's published key (varies by kind: DNS resolver result; HTTPS fetch; cached registry peer's identity).
3. Applying receiver policy to the `trust_anchor` variant returned by the backend.

### §3.0a Unknown binding `kind` (forward-compat, mirrors §4.2)

An unknown `kind` value on a received binding MUST cause the binding to be ignored during `meta_resolve` with a warning; the binding entity itself remains valid on the wire (forward-compat). Known kinds proceed as normal. This mirrors the §4.2 `backend_kind` forward-compat rule.

### §3.0 One name, many target peers (multiplexing pattern)

A `binding` carries a single `target_peer_id`. The "many peers behind one logical name" case — backend pool, multi-region failover, load-balanced front — is handled at the **transport layer, not the registry layer**: the same `target_peer_id` may publish multiple `transports` (per NETWORK §6.5 priority-selection) covering distinct addresses, and a binding's `transports` field MAY enumerate several. Genuinely multiple distinct *peers* fronting one name is the **aggregator/federation case** (§8.2), v1-deferred. v1 registry returns a single binding per resolution (§4.1.1).

`target_peer_id` rotation under DNS-backed bindings (the DNS record rotates faster than the cached binding's `ttl`) is mitigated by the §2.3 re-resolve-on-failure loop — a stale binding whose endpoints stop working triggers `:resolve` re-fetch before TTL expires.

### §3.1 Revocations

Revocations travel as supersedes-chain entries (per ATTESTATION extension) — new binding with `kind` unchanged + a marker that the previous is revoked — OR as a separate `system/registry/revocation` entity referencing the binding hash:

```
type: "system/registry/revocation"
data: {
  revokes:    <binding_hash, BARE>,
  revoked_at: <ms-since-epoch>,
  reason:     <string | null>
}
```

The revocation's authenticating `system/signature` entity is carried per the same target-matching + invariant-pointer contract as bindings (§3): `data.target == revocation.content_hash`, `data.signer == authority` (same authority as the revoked binding), and reachable at `system/signature/{hex(revocation.content_hash)}`.

`:resolve` MUST check for a `system/registry/revocation` targeting a candidate binding before returning `status: "resolved"`. If a revocation is found and verifies against the same authority as the binding, the binding is excluded and meta_resolve advances to the next chain entry. Subscription-driven cache invalidation is the MAY path on top.

---

## §4 The resolver-config entity

Deployment-side configuration of which backends + order + trust acceptance:

```
type: "system/registry/resolver-config"
data: {
  resolver_chain: [
    {
      backend_kind:           <"local-name"|"did-web"|"dns-txt"|"peer-issued"|...>,
      backend_id:             <peer_id_hash | identifier>,    ; e.g., the registry peer-id
      priority:               u32,                            ; ascending; lower = consulted first
      accepted_trust_anchors: [<variant filter>],             ; receiver policy
      hints:                  <opaque object | null>          ; backend-specific config
    }
  ],
  pinned_bindings: [
    {
      name:           <string>,
      target_peer_id: <peer_id_hash>,
      reason:         <string | null>                         ; documentation
    }
  ],
  name_format_dispatch: [                                     ; meta-resolver routing
    {
      pattern:        <POSIX shell-glob>,
      backend_kinds:  [<kind>]                                ; which backends to consult for this format
    }
  ]
}
```

Stored at `system/registry/resolver-config` (peer-local; not synced).

**`pattern` grammar.** The `name_format_dispatch[].pattern` field is a **POSIX shell-glob**, matched against the user-facing name string. This deliberately reuses the same glob grammar already chosen for BRIDGE-HTTP §4-RES.2 (URL patterns) rather than introducing a second matcher language. Examples: `*@*.*` → DNS-style handles; `did:web:*` → did:web; `*.eth` → ENS; `*` → catch-all (typically local-name). Deployments needing richer matching layer it in the backend, not the dispatch config.

### §4.1 Precedence order on resolution

When `meta_resolve(name)` is called:

1. **Pinned bindings** override everything. If `name` matches a pinned entry, return the synthesized result (§4.1.2) immediately.
2. **`name_format_dispatch` filter** — narrow the resolver-chain to backends whose dispatch pattern matches the queried `name` (POSIX shell-glob). Backends without a `name_format_dispatch` entry default to "match all" (no filtering); backends with one are consulted ONLY when the pattern matches. **This is the primary privacy mechanism** — without it, the queried name leaks to broad-matching backends earlier in priority.
3. **Filtered resolver-chain backends in priority order** — try each, returning the first validated result.
4. If all backends miss / fail validation: return `chain_exhausted` (fail-closed; no silent fallback).

The local-name backend (§6) participates as a resolver-chain entry like any other backend. Local-name-first ordering is a deployment convention realized by setting the local-name entry's `priority` to `0` (or another low value); the substrate stays uniform.

### §4.1.2 Synthesized result for pinned bindings

When a `pinned_bindings` entry matches, `meta_resolve` returns a `system/registry/resolution-result` entity (§2.1 wire encoding) with flat data fields:

```
ResolutionResult {
  status:       "resolved",
  binding:      <hash of a synthetic system/registry/binding entity constructed from {name, target_peer_id, transports: []}, kind: "out-of-band"; deterministic per-pin>,
  peer_id:      <pin.target_peer_id>,
  transports:   [],                  ; empty — transport-layer resolution per NETWORK §6.5 follows
  attestations: [],
  trust_anchor: "out_of_band",
  ttl:          null,                ; pins are sticky until removed
  neg_ttl:      null,
  backend_id:   "pinned"
}
```

Empty `transports` on a pin is acceptable; pins assert binding authority. Transport resolution per NETWORK §6.5 finds reachable endpoints.

### §4.1.1 Single binding per name per resolution

`meta_resolve` returns the **first hit** that passes validation. If two backends would resolve the same name to different `target_peer_id` values, the higher-priority backend wins; the lower-priority backend's binding is never surfaced for that resolution. This is by design for v1 — the alternative (surface all hits, let the caller choose) is the aggregator/federation case, deferred per §8.2. **Caller-side multi-hit awareness** lives at the aggregator layer when Mode A ships.

### §4.2 Schema versioning

New backend kinds will be added over time. Resolver-config is forward-compatible: an unknown `backend_kind` MUST cause the entry to be skipped with a warning, NOT cause the whole config to be rejected.

---

## §5 Capability model

**The substrate gates no name claims.** Anyone can publish a binding entity claiming any name. The substrate enforces:

- **Signature verification** on non-self-certifying / non-local-name bindings.
- **Receiver policy compliance** (the `accepted_trust_anchors` filter).
- **Revocation honor** (revoked bindings NOT used even if still cached).
- **Fail-closed** on chain exhaustion (no silent acceptance of unsigned or policy-rejected bindings).

**Trust is receiver-side.** Whether a binding gets used depends on:
- Does its `trust_anchor` variant pass the receiver's `accepted_trust_anchors` filter?
- Does its `issuer_signature` validate against the expected authority?
- Is it pinned (auto-trusted) or revoked (auto-rejected)?

The substrate's cap surface:

| Cap | Purpose | Operation(s) |
|---|---|---|
| `system/capability/registry-resolve` | who may invoke `:resolve` against the registry handler | `:resolve` |
| `system/capability/registry-configure` | who may edit the resolver-config | tree-write `system/registry/resolver-config` |
| `system/capability/registry-pin` | who may add or remove pins | tree-edit `resolver-config.pinned_bindings` |
| `system/capability/registry-cache-control` | who may invalidate cached resolutions | `:invalidate-cache(name | null)` (null = flush all) |
| `system/capability/registry-local-name-bind` | who may create or update local-names (§6) | `:bind`, `:update-transports` |
| `system/capability/registry-local-name-unbind` | who may remove local-names (§6) | `:unbind` |
| `system/capability/registry-local-name-list` | who may enumerate local-names (§6) | `:list` |

Per-backend caps for backends shipped in their own proposals (e.g., "may publish a binding to the peer-issued registry") live in those backend extension specs.

**No `system/capability/registry-publish-binding-for-name-X` cap exists.** Anyone can publish a binding claiming any name. Receiver policy decides.

### §5.1 Connection-authority invariant

**Resolution never confers connection authority.** A binding from an untrusted or low-trust backend can be safely consumed for its `transports` field because dialing those transports does NOT admit the peer — IDENTIFY (per NETWORK / EXTENSION-IDENTITY) is the gate. The dispatcher (NETWORK §10) is responsible for cap-verifying the dialed peer post-IDENTIFY. Symmetric to DISCOVERY §2.2's pin: the registry surfaces candidates; trust is established at IDENTIFY.

### §5.2 Default grants on first install

The REGISTRY seed-policy bootstrap (per V7 §6.9a) grants the local peer all seven caps above. Otherwise the user cannot use their own local-name store or run resolutions, and §4.4 advertised-handler discipline is violated. Distributions MAY tighten via `--default-grants` flags, but the v1 floor grants the local peer full self-access.

---

## §6 Local-name backend (v1)

The local-name backend is the v1 concrete backend that exercises the substrate's resolver-handler contract end-to-end. It's the simplest possible backend (no network dependencies, no signature ceremony, no authority semantics) and useful immediately for organizing known contacts.

### §6.1 Concept

A **local-name** is a user-assigned local name for a peer identity. Local-names have no global meaning; they exist only in the assigning user's local configuration. Trust source is the user themselves — the user is asserting *"this name means this peer-id; I take responsibility for the binding."*

### §6.2 Backend identity

The local-name backend identifies as `backend_kind: "local-name"` in resolver-config. There is exactly one local-name store per peer; `backend_id` for local-name entries is the local peer's identity.

A local-name backend MAY register with a custom `backend_id` distinguishing multiple local-name namespaces (e.g., personal vs work local-names). v1 is single-store.

### §6.3 Local-name binding (specialization of §3)

```
type: "system/registry/binding"
data: {
  name:           <string>,                ; the local-name (user-chosen)
  kind:           "local-name",
  target_peer_id: <Base58 peer-id per V7 §1.5>,
  transports:     [<endpoint per NETWORK §6.5>],  ; optional; cached from last contact
  issued_at:      <ms-since-epoch>,
  ttl:            null,                    ; local-names are sticky until user removes
  supersedes:     <system/hash, BARE | null>,  ; previous local-name for same name (rebound)
  metadata: {
    notes:        <string | null>,         ; user-facing note
    pinned:       bool                     ; whether user has pinned (default true)
  }
}
```

**No `system/signature`** — the user IS the trust source; the binding lives in the user's local store; signing is meaningless (the user trusts themselves by definition). This is the local-name carve-out from the §3 universal signature requirement.

**Two-layer storage** (universal entity-system pattern):
- **Binding body** lives at `system/registry/binding/{binding_hash}` per the §3 universal rule. The body is content-addressed and immutable.
- **Tree pointer** lives at `system/registry/binding/local-name/{name}` and holds the bare `system/hash` of the current head binding body. The tree pointer is the live name→hash index — mutable per `:bind` / `:unbind` / `:update-transports`.
- Supersedes-chain integrity is the audit log: walking `body.supersedes` refs from the head binding reconstructs the rebind history. `:list` (§6.5) reads the tree-pointer prefix (the live index); supersedes-chain history is accessed by hash lookup when needed.

**Name-path safety (normative).** Because the storage path embeds `{name}` as a path segment, local-name names MUST satisfy:

- MUST NOT contain `/` (would create ambiguous parent/child path semantics).
- MUST NOT contain control characters U+0000 through U+0020 or U+007F (the C0 + DEL range).
- MUST be Unicode-normalized to NFC at `bind` time (so `Café` NFC vs `Café` NFD don't produce two distinct local-names).
- Implementations MAY apply additional length / character-class restrictions via `local-name-config`.

`bind` rejects names violating these rules with `bind_invalid_name` and does not write to storage. Already-stored bindings that violate (legacy / migration) MUST be either rejected at load with a warning or normalized + re-bound; impls MUST NOT silently treat ambiguous-path bindings as valid.

### §6.4 Local-name-store config

```
type: "system/registry/local-name-config"
data: {
  default_pinned:     bool,                 ; new entries default to pinned (recommended true)
  allow_supersede:    bool,                 ; allow rebinding existing names (default true)
  case_normalization: "none" | "lower"      ; local-name case handling (default "none")
}
```

Stored at `system/registry/local-name-config`.

### §6.5 Local-name handler operations

The local-name backend implements `system/registry:resolve` per §2.1 + four backend-specific operations:

**`system/registry:resolve`** (substrate-required):

```
resolve(name, hints) → ResolutionResult
  normalized = nfc_normalize(name)
  if local-name-config.case_normalization == "lower":
      normalized = lowercase(normalized)
  pointer = lookup_tree_pointer("system/registry/binding/local-name/" + normalized)
  if pointer is null:
    return { status: "not_found" }
  entry = fetch_binding_body(pointer)        ; from the content-tree at system/registry/binding/{hash}
  return {
    status:        "resolved",
    binding:       entry.hash,
    peer_id:       entry.target_peer_id,
    transports:    entry.transports,         ; MAY be empty
    attestations:  [],                       ; local-name carries no attestations
    trust_anchor:  "local_name",
    ttl:           null,
    neg_ttl:       null,
    backend_id:    <local_peer_id>
  }
```

**Normalization symmetry** — `:resolve` applies the same NFC normalization + optional case fold (per `local-name-config.case_normalization`) BEFORE lookup that `:bind` (§6.5.1) applies before storage. The normalized form is the storage key.

An empty `entry.transports` (the user bound a local-name before ever observing a reachable endpoint) still returns `status: "resolved"` — the binding IS authoritative for the name-to-peer mapping; `transports` are a cached hint, not the binding's substance. The downstream Layer-B logic (transport-profile resolution, transport-fallback per §2.3) handles "no reachable transport" separately.

**`system/registry/local-name:bind`** — create a new local-name binding:

```
bind(name, target_peer_id, transports, notes) → binding_hash
  validate name (per §6.3 name-path safety + per local-name-config)
  normalized = nfc_normalize(name)
  if local-name-config.case_normalization == "lower":
      normalized = lowercase(normalized)
  existing_pointer = lookup_tree_pointer("system/registry/binding/local-name/" + normalized)
  if existing_pointer and not allow_supersede:
    return error("bind_already_exists", 409)
  if existing_pointer and allow_supersede:
    new_body = construct_binding(supersedes = existing_pointer.hash, …)
  else:
    new_body = construct_binding(supersedes = null, …)
  store_body("system/registry/binding/" + new_body.hash, new_body)        ; universal §3 location
  update_tree_pointer("system/registry/binding/local-name/" + normalized, new_body.hash)  ; live index
  return new_body.hash
```

Error codes (REGISTRY's code domain per V7 §3.3):

| Code | Status | When |
|---|---|---|
| `bind_invalid_name` | 400 | name violates §6.3 path safety (contains `/`, control chars, or fails NFC) |
| `bind_already_exists` | 409 | normalized name already bound and `allow_supersede=false` |

**`system/registry/local-name:unbind`** — remove a local-name binding:

```
unbind(name) → ()
  remove from local_name_store
  (binding entity remains in CONTENT tree; supersedes-chain preserved per ATTESTATION discipline)
```

**`system/registry/local-name:list`** — list all current local-name bindings:

```
list([filter]) → [LocalNameEntry]
  enumerate tree-pointer prefix "system/registry/binding/local-name/*" via LocationIndex.ListPrefix
  ; this IS the live index — tree pointers are the live name→hash mapping
  ; supersedes-chain walking is the audit log, accessed by hash lookup when history is needed
  return [{name, hash, target_peer_id, notes, pinned} per tree pointer]
```

`:list` reads the index, not the audit log. Each tree pointer holds the live head binding hash; supersedes-chain history is accessed via `tree:get system/registry/binding/{hash}` when needed (auditable, but not on the hot path of listing).

**`system/registry/local-name:update-transports`** — update cached transports for an existing local-name (e.g., after a successful contact reveals new endpoints):

```
update-transports(name, transports) → new_binding_hash
  issue new binding with supersedes = existing.hash
  same target_peer_id; only transports updated
  return new binding.hash
```

### §6.6 Local-name composition with EXTENSION-IDENTITY

Local-name `target_peer_id` IS an EXTENSION-IDENTITY peer-id (V7 §1.5 multikey). A local-name can point at:
- A `Public_alice`-style identity peer-id (the stable cross-rotation identifier).
- A specific runtime-peer-id (rare; typically point at the identity, not the runtime peer).

When the target identity rotates per EXTENSION-IDENTITY §4.3/4.4, the local-name remains valid (it points at the stable Public_X identifier; runtime-peer-set walks find current runtime peers).

### §6.7 What the local-name backend does NOT do

- Sync local-name bindings to other peers (local-names are local-by-discipline).
- Publish local-name bindings to any external registry.
- Require signature on local-name bindings (the user is the trust source).
- Cross-peer local-name sharing (group-shared registry is a separate future mechanism).
- Local-name syncing between user's own devices (out of v1 scope; could be MAY in future revision).
- Automatic local-name suggestion from other resolutions (UX concern, not substrate).

---

## §6a Peer-issued backend (v1)

The peer-issued backend is the second v1 concrete backend. Where local-name (§6) trusts the **user themselves**, peer-issued trusts a **remote registry peer whose key the resolver has pinned** — turning a name into a *verified* binding ("the registry signed this, and I checked it against the key shipped with my build") rather than a *pinned* one ("the distro hard-asserts it"). It is the sibling of §6: **same reads, different trust source.** Full design rationale + the cohort spec-doubt rulings (P1–P7) live in `proposals/implemented/PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND.md`; the normative contract is here.

### §6a.1 Concept — trust logic over transport-agnostic reads

A registry peer is **just a peer** (§1 position 4); its bindings are ordinary entities in its tree. The backend reads them with the **normal `tree:get` / `content:get` machinery against the registry peer** and **does not know or care** whether that peer is reached over http-poll (a static coral-reef, the demo case) or a live socket — *how* the registry is reached is the **transport layer's** job (NETWORK §6.5; http-poll = SUBSTITUTE §7 Mode S). The backend's only registry-specific substance is **trust verification** (§6a.4 step 3). The backend MUST NOT perform or select a transport itself.

### §6a.2 Backend identity

The peer-issued backend identifies as `backend_kind: "peer-issued"` in resolver-config. Its `backend_id` is the **registry peer-id**, which doubles as the **pinned trust root**: a resolver-chain entry for peer-issued names carries that peer-id and accepts a binding only if signed by it.

### §6a.3 The peer-issued binding + by-name index

The binding body is a standard §3 `system/registry/binding` with `kind: "peer-issued"`, `ttl` set (issued bindings expire, unlike sticky local-names), and — unlike §6.3 — it **carries a `system/signature`** (the registry is a remote authority; the signature is the whole point). Signature is reachable at the invariant-pointer `system/signature/{hex(binding_hash)}`, target-matching the binding's `content_hash` (V7 §5.2).

Two-layer storage, the direct analog of §6.3:
- **Binding body** at `system/registry/binding/{binding_hash}` (§3 universal rule).
- **By-name pointer** at `system/registry/binding/by-name/{nfc(name)}` → the bare `system/hash` of the current binding body. This is the live name→hash index — same pattern as local-name's `local-name/{name}` pointer, different prefix. Served over http-poll like any tree node (with the `tree_leaf_suffix` disambiguator, SUBSTITUTE §2.2).

**Name-path safety (normative):** identical to §6.3 — no `/`, no C0/DEL control chars, NFC at issue time. Domain-shaped names (`billslab.com`) are fine (dots allowed).

### §6a.4 Resolve algorithm (normative)

```
peer_issued.resolve(name, config):           ; config = the resolver_chain entry (§4)
  registry = config.backend_id                ; the registry peer-id = PINNED trust root
  norm     = nfc_normalize(name)              ; §6.3 name-path safety

  ; 1-2. transport-agnostic reads against the registry peer (transport layer maps to http-poll
  ;       GETs for a static coral-reef, or a live socket if the registry runs one):
  binding_hash = tree:get(registry, "system/registry/binding/by-name/" + norm)
  if binding_hash is null: return { status: "not_found", neg_ttl: config.neg_ttl }
  binding      = content:get(registry, binding_hash)        ; bytes hash-verified == binding_hash

  ; 3. VERIFY (§3 steps 1-3, §5) — the ONLY registry-specific logic; this IS the backend:
  sig = read(registry, "system/signature/" + hex(binding_hash))   ; invariant-pointer (V7 §5.2)
  require sig.target == binding_hash
  require sig.signer == content_hash(canonical(registry.system_peer))   ; signer is a hash (V7 §1.5/§5.2)
  require verify_crypto(sig, pinned_key_of(registry))                   ; §6a.5 trust-anchor floor
  require ("peer_issued:" + registry) in config.accepted_trust_anchors  ; empty set ⇒ fail-closed
  require not revoked(registry, binding_hash)                           ; §6a.6
  require binding.issued_at + binding.ttl > now()  (or ttl null)

  ; 4. surface
  return ResolutionResult {
    status: "resolved", binding: binding_hash,
    peer_id: binding.target_peer_id, transports: binding.transports,
    trust_anchor: "peer_issued:" + registry, ttl: binding.ttl, backend_id: registry }
```

**Fail-closed (normative):** any verify / revocation / expiry failure returns `None`/dead-end at this rung; the meta-resolver (§4.1) advances the chain. It **MUST NOT** silently downgrade to an `out_of_band` pin — a pin matches only if explicitly configured as its own chain entry.

**Levels (P2/P3):** the backend returns `not_found` + a first-class `neg_ttl` slot on the negative result (backend-scoped); the meta-resolver collapses a whole-chain miss to `chain_exhausted` (§4.1) carrying the aggregated `neg_ttl`. `neg_ttl` is a defined optional top-level field, not an opaque hint-bag entry.

**Offline / precedes path:** when the binding is pre-cached as a precede (§7), steps 1–2 read the local store instead of the wire; step 3 verify is identical. Precedes are just a warm cache.

### §6a.5 Trust-anchor floor (v1)

The v1 trust anchor is an **Ed25519 identity-multihash** registry peer-id: the pubkey **is** the peer-id digest (V7 §1.5 canonical form), so `pinned_key_of(registry)` is derived from `backend_id` and config carries only the peer-id. For a **non-self-describing** peer-id form (SHA-256-form / Ed448), the resolver-chain entry MUST carry the pubkey explicitly (or a locally-resolvable `system/peer`); this config-carried-key path is **deferred** (no v1 demo needs it). Consistent with the core §9.1 floor.

### §6a.6 Revocation (by-target index, normative)

`revoked(registry, binding_hash)` is an O(1) index lookup, **not** a scan: `system/registry/revocation/by-target/{hex(binding_hash)}` → the revocation entity (presence = revoked, if it verifies against `registry` per §3.1). This is the revocation analog of the §6a.3 by-name index. A live registry MAY layer subscription-driven invalidation on top (§3.1).

### §6a.7 Signed binding-manifest (OPTIONAL — format pinned, v1 = per-name index)

The by-name index (§6a.3) is the **floor and the v1 implementation**: one tree pointer per name, one round-trip per name; a conformant resolver MUST support it. Single-name resolution is internet-scale at the floor (fetch only `by-name/{name}` — the DNS model; no zone download).

A registry MAY additionally publish a single **signed manifest** (`system/registry/binding-manifest` at `{tree_url_prefix}/registry/manifest/current`) listing the whole name→hash index, for one-round-trip bulk fetch — the **direct analog of SUBSTITUTE §2.4/§7.2 `snapshot-manifest`** with registry-domain fields (`registry_id`, `snapshot_at`, `seq`, `coverage`, `bindings`, optional `predecessor`). Its **format is pinned here so no registry invents its own** (anti-fracturing); its **implementation is deferred** (optional optimization, the way SUBSTITUTE Mechanism B sits on Mechanism A). Normative rules when present: signature is MUST (verified against the pinned key; `seq` freshness per SUBSTITUTE §7.2; no operator-trust override); a missing/stale/signature-invalid manifest MUST fall through to the §6a.3 per-name pointer; **absence governed by `coverage`** — `partial` (default) ⇒ a name not in `bindings` MUST fall through to the per-name pointer (no negative claim); `complete` ⇒ absence is authoritative `not_found` (DNSSEC-NSEC / TUF model). Past the V7 §4.10 max-payload ceiling, the sanctioned scaling direction is a **delegated/sharded manifest** (DNS-zone / TUF-delegated-targets; an entry value is a `system/hash`, a sub-manifest is another `binding-manifest`) — direction pinned, detailed format deferred. Reuses SUBSTITUTE's `manifest_signature_invalid` / `manifest_stale_seq` codes.

### §6a.8 Curated registration (v1)

A curated/static registry's operator decides what it signs; registration is operator tooling, no live protocol (this is how the release registry, e.g. the Entity Church Registry, ships):

```
registry-issue-binding(name, target_peer_id, transports, ttl) :    ; operator tool, holds K_registry
  body = system/registry/binding { name, kind:"peer-issued", target_peer_id, transports, issued_at:now, ttl }
  sig  = sign(K_registry, body.content_hash)
  publish body at system/registry/binding/{body.content_hash}
  publish sig  at system/signature/{hex(body.content_hash)}
  set pointer system/registry/binding/by-name/{nfc(name)} → body.content_hash

registry-revoke-binding(binding_hash, reason) :
  rev = system/registry/revocation { target: binding_hash, reason, issued_at:now }
  sig = sign(K_registry, rev.content_hash)
  publish rev + sig at the invariant-pointer
  set pointer system/registry/revocation/by-target/{hex(binding_hash)} → rev.content_hash
```

### §6a.9 Live registration — `register-request` (open/allowlist/manual)

Curated registration (§6a.8) is operator-signs-by-hand. **Live registration** lets a *publisher* self-register against a registry that runs the handler. A registry is just a peer (§1 position 4) and operates in one of two modes — curated/static (no live protocol) or live (runs the `register-request` handler). The request:

```
type: "system/registry/register-request"
data: {
  name:           <string>,                  ; name-path safety per §6.3
  target_peer_id: <Base58 peer-id, V7 §1.5>, ; what the name resolves to
  transports:     [<endpoint per NETWORK §6.5>],
  requested_ttl:  <ms | null>,
  nonce:          <bytes>,                    ; anti-replay
  issued_at:      <ms-since-epoch>
}
```

The request **MUST carry a `system/signature` by `target_peer_id`** (target-matching, V7 §5.2; invariant-pointer at `system/signature/{hex(request.content_hash)}`). This is **ownership-proof layer 1** and is always required: it proves the requester holds the key they are binding the name to, so no one can register *someone else's* peer-id under a name. Handler op:

```
system/registry/peer-issued:register-request(request) → binding_hash | rejection
  1. verify request signature by target_peer_id          ; layer-1 (always)
  2. apply issuer-policy admission (§6a.9.1)              ; layer-2 → approve | reject | queue
  3. on approve: registry-issue-binding(...) (§6a.8)      ; signs with K_registry, publishes, sets by-name pointer
     return binding_hash
  4. on reject:  error (name_taken | not_entitled | policy_rejected)   ; REGISTRY code domain
  5. on queue:   status "pending_review"                  ; manual mode
```

Two follow-on ops, with explicit schemas (the design fold left these as bare "follow-on ops"; cohort impl surfaced the gap — each impl guessed a different shape, so they are pinned here):

- **`:renew-request` — replay-defended (carries `nonce` + `issued_at`).** Extends a binding's lifetime via the supersedes-chain. Signed by `target_peer_id` (layer-1).
  ```
  type: "system/registry/renew-request"
  data: { binding_hash: <system/hash>, ttl: <ms | null>, nonce: <bytes>, issued_at: <ms-since-epoch> }
  ```
  Renew has a **non-idempotent state effect** — each accepted renew extends the binding's expiry — so a captured renew can be **replayed to keep a binding alive past the registrant's intended lapse.** It therefore carries the same `nonce` + `issued_at` replay defense as `register-request` (§6a.9.1).

- **`:revoke-request` — NOT replay-defended.** Revokes a binding (by registrant or operator; emits a §3.1 revocation). Signed by `target_peer_id` or the operator.
  ```
  type: "system/registry/revoke-request"
  data: { binding_hash: <system/hash>, reason: <string | null> }
  ```
  Revocation is **monotonic and target-pinned** — it acts on one content-addressed `binding_hash`, cannot be undone, and cannot target a later re-issued binding (different hash) — so replay is a provable no-op. `nonce` / `issued_at` are **not** part of the schema (they would be harmless but add no security and break cohort convergence).

#### §6a.9.1 Issuer policy — the registry's own admission decision

The substrate gates no name claims (§5); a live registry decides what it signs via its own **local config** (a knob, not a mandate):

```
type: "system/registry/issuer-policy"          ; registry-local config
data: {
  mode:             "open" | "allowlist" | "manual" | "domain-control",
  allowlist:        [<peer-id>] | null,
  name_constraints: <glob | null>,              ; e.g. only issue "*.lab"
  default_ttl:      <ms | null>
}
```

Two separable proof layers:
- **Layer 1 — peer-id control (always, §6a.9):** the request is self-signed by `target_peer_id`.
- **Layer 2 — name entitlement (policy):** whether *this requester* may have *this name*.
  - **`open` — first-come-first-serve.** Any layer-1-valid request for a free name is approved and signed. This is the trivial default: no entitlement check beyond "is the name taken." It is barely more than the curated tool with a handler wrapper.
  - **`allowlist`** — only `target_peer_id`s in `allowlist` may register (optionally bounded by `name_constraints`).
  - **`manual`** — requests queue as `pending_review`; the operator approves out-of-band.
  - **`domain-control`** — for domain-shaped names, the registry requires proof of DNS-domain control before signing. **The challenge format is DEFERRED** — it MUST share one mechanism with the web-native `dns-txt` / `well_known_url` backends rather than inventing a second domain-proof scheme, so it is settled jointly with those proposals, not here. A v1 registry uses `open` / `allowlist` / `manual`; `domain-control` lands with the web-native co-design.

`registry-issue-binding` (the internal sign+publish act) is gated by `system/capability/registry-issue-binding`, held by the policy logic / operator only. `register-request` is the *external* surface, gated by `system/capability/registry-request-binding` (open → granted broadly; allowlist → narrow). `system/capability/registry-manage-issuer-policy` gates editing the policy.

**Replay defense (normative discriminator).** A signed request carries `nonce` + `issued_at` (the registry tracks seen `nonce`s per requester within an `issued_at` window; a replayed request is rejected) **iff replay has a non-idempotent state effect.** This holds for `register-request` (replay can roll a name back to a superseded binding) and `renew-request` (replay can extend a binding's life past intended lapse). It does **not** hold for `revoke-request`, which is monotonic on a content-addressed target (replay cannot un-revoke and cannot reach a later re-issued binding) — so revoke omits `nonce` / `issued_at`. The discriminator, not the op name, decides: future ops are replay-defended exactly when their replay mutates state.

**Conformance:** `REG-REGISTER-PROOF-1` (signature not by `target_peer_id` → rejected), `REG-REGISTER-POLICY-1` (allowlist reject → `not_entitled`; allow-listed → issued + resolvable), `REG-REGISTER-REPLAY-1` (seen nonce → rejected). `REG-REGISTER-DOMAINCTRL-1` gates the deferred `domain-control` mode.

**Implementation status:** the **design is pinned here**; the `open` / `allowlist` / `manual` modes are buildable now (no external dependency); `domain-control` waits on the web-native domain-proof co-design. A registry shipping curated-only (§6a.8) is conformant — it simply does not run the handler.

### §6a.10 What the peer-issued backend does NOT do (v1)

- **`domain-control` challenge format** — deferred to the web-native domain-proof co-design (§6a.9.1), so there is one domain-proof mechanism, not two. `open`/`allowlist`/`manual` registration is fully specified above.
- Perform or choose transports (§6a.1 — that is the transport layer's job).
- Accept an unsigned binding, or downgrade to a pin on verify failure (§6a.4 fail-closed).
- Implement the signed manifest in v1 (§6a.7 — format pinned, impl deferred).

---

## §7 Bootstrap-with-precedes pattern

### §7.1 The pattern

A distribution ships its binary with:
- **Pre-configured `system/registry/resolver-config`** entity (one or more backends pre-installed; trust anchors pre-accepted).
- **Pre-cached binding entities** ("precedes") for the distribution-curated set of names.
- **Pre-trusted registry peer-ids** pinned in resolver-config.

New user inherits this trust by accepting the build. Immediate functionality without manual configuration.

### §7.2 Swap discipline

The user can:
- Edit resolver-config to remove or reorder backends.
- Remove pinned peer-ids.
- Add their own resolver-chain entries.
- Replace shipped precedes with their own bindings.

**This is the substrate-exposed override.** Distributions provide opinion; users own the configuration.

### §7.3 Structural framing

This is structurally analogous to OS installations shipping with pre-loaded root CA certificates. Pre-trust by acceptance of the distribution; remove or override at any time. The substrate exposes the mechanism (resolver-config + pin store); the discipline is "distribution opinion is opinion; user owns config."

### §7.4 Worked example — the preloaded Entity System Registry (release bootstrap)

The day-one release ships a **real, signed** base registry — not local nicknames. This worked example pins exactly what is preloaded and how "trusted" differs from a local-name.

**Trusted (peer-issued) vs local-name (local):**

| | Local-name binding | Entity System Registry binding |
|---|---|---|
| `kind` | `local-name` | `peer-issued` |
| `issuer_signature` | `null` | signature by the registry peer's identity key |
| trust source | the local user's assertion | a key the distribution preloads + pins |
| `trust_anchor` | `local_name` | `peer_issued:{registry_peer_id}` |
| scope | local-only, no global meaning | verifiable by anyone holding the registry's pinned identity |

A local-name is *"I say this name means this peer."* The Entity System Registry binding is *"the registry signed this, and I verify that signature against a key shipped with my build."* That is the difference between **asserted** and **verified**.

**What the distribution preloads (the bootstrap package / the "seed"):**

1. The registry peer's **identity entity** — peer-id + public key — pinned as a trusted authority (the verification root; like a root CA shipped with an OS).
2. A **resolver-config** (§4) with a `peer-issued` backend entry for the registry, `accepted_trust_anchors: [peer_issued:{registry_peer_id}]`, and the registry's static `http-poll` endpoint in `hints` so its binding tree can be fetched cold.
3. **Precedes** (optional) — pre-cached *signed* coral-reef bindings (Bill's Lab, entitycoreprotocol.org, entitychurchfoundation.org) so first run works fully offline; absent these, they're fetched from the registry's static tree on first connect.

```
# preloaded, pinned — the root of trust
system/peer/{entity_system_registry_peer_id}      ; identity entity: peer-id + public key

# preloaded resolver-config
system/registry/resolver-config
  resolver_chain: [
    { backend_kind: "peer-issued",
      backend_id:   <entity_system_registry_peer_id>,
      priority:     0,
      accepted_trust_anchors: [ "peer_issued:<entity_system_registry_peer_id>" ],
      hints:        { http_poll_endpoint: <registry static URL> } }
  ]
  pinned_bindings: [ ]            ; optional name→peer_id pins

# preloaded precedes (optional; each SIGNED by the registry)
system/registry/binding/{hash}   ; kind: peer-issued, issuer_signature: <registry sig>,
                                 ;   name: "entitychurchfoundation.org", target_peer_id, transports: [http-poll ...]
```

**The onboarding flow ("add the Entity System Registry"):**

Fresh peer, no contacts. The UI offers: *"Add the Entity System Registry to get connected."* Accepting installs the package above. Then:
1. `resolve("entitychurchfoundation.org")` consults the peer-issued backend.
2. The registry's **signed** binding for `entitychurchfoundation.org` is fetched over HTTP poll (or read from precedes).
3. The receiver **verifies `issuer_signature` against the pinned registry identity** and checks the `peer_issued` trust anchor passes `accepted_trust_anchors`.
4. On success: `ResolutionResult` → entitychurchfoundation.org's `peer_id` + its `http-poll` endpoints (from the binding's `transports`).
5. The browser pulls entitychurchfoundation.org's content by hash over HTTP poll.

No live registry peer is required at any step — the registry is itself a coral reef (a static publisher in dormancy). Trust is cryptographic, rooted in a key shipped out-of-band: bootstrap-with-precedes made concrete.

---

## §8 Composition with RELAY (the aggregator-as-meta-registry pattern)

### §8.1 Registry-peer-as-Mode-S-publisher

A registry peer publishes its bindings as entities at `system/registry/binding/...` in its own tree. Consumers fetch via standard content flow — the static-CDN-hosted case is Mode S relay (already shipped via STORAGE-SUBSTITUTE-HTTP).

**No special registry transport.** The registry's bindings are entities; the substrate's content-fetch machinery applies directly.

### §8.2 Aggregator-as-meta-registry (federation) — **v1-DEFERRED**

A peer running RELAY Mode A subscribed to N registry peers' binding subtrees serves the union as its own `system/registry/binding/...` tree. Consumer installs this aggregator as ONE resolver backend; aggregator handles the multi-source mechanics.

> **v1 deferral.** This composition depends on RELAY **Mode A**, which is deferred from v1 per `PROPOSAL-EXTENSION-RELAY.md §11.1a` (cross-peer subscription dependency — current substrate's subscription engine is local-tree-only). The aggregator-as-meta-registry pattern is named here for forward-compatibility; it is **not shippable in v1.** Cross-registry federation lands when Mode A lands.

The aggregator does NOT re-sign aggregated bindings; receivers verify against the original issuer's signature. Aggregator is transport.

### §8.3 Conflict handling at aggregator level

When two upstream registries return different `target_peer_id` for the same `name`:
- Aggregator surfaces both bindings (does not silently pick).
- Consumer's policy decides:
  - Default fail-closed (reject ambiguous resolution).
  - Explicit pin override (consumer's resolver-config wins).
- Aggregator MAY emit a `system/registry/conflict` annotation entity for observability; not normative.

---

## §9 GC posture (per `core-protocol-domain/guides/GUIDE-GC.md`)

- **Resolved-binding cache:** subscription-driven; honor `ttl` on bindings; observed revocations propagated.
- **Resolver-config:** persistent until explicitly changed; no GC.
- **Pinned bindings:** exempt from GC until unpinned.
- **Local-name bindings:** persistent until user-unbinds.
- **Local-name superseded bindings:** kept per ATTESTATION supersedes-chain discipline (auditable history).
- **Local-name-config:** persistent.
- **Aggregator's collected bindings:** per Mode A relay retention policy (when Mode A lands).
- **Pre-shipped precedes:** persistent through distribution updates; user MAY evict.

All retention windows are operator-configurable knobs; defaults conservative (unlimited / off / largest).

---

## §10 Configurations (deployment patterns)

These are deployment configurations, not a hierarchy. Each is a valid choice for some use case.

- **Local-name-only.** Local local-name backend; no network resolution. Use for closed personal-contacts network. Resolver-chain empty after local-name.
- **Distribution-trusted.** Pre-shipped resolver-config with distribution-curated registry peer + pre-cached precedes. Default for new-user onboarding.
- **Local-name + bootstrap precedes.** Distribution-shipped precedes appear as local-name-store entries; user inherits by install; can edit.
- **Local-name as cache layer.** Local-name + (later) did-web / dns-txt; local-name-first in chain. Resolver hits local-name for known contacts (free, authoritative for user); falls through to network resolution for unknown names.
- **DID:web-only.** (When did-web ships.) Rely on DNS+TLS PKI; familiar shape for web-anchored identities.
- **Self-hosted registry.** Run your own registry peer; install yourself as a resolver backend; publish bindings for your team.
- **Aggregator-federated.** (When Mode A ships.) Install a Mode-A aggregator backend that consumes from N independent registries.
- **All-crypto.** Reject any backend whose `trust_anchor` is DNS-based or consensus-based; accept only `peer_issued` + `self_certifying` + `out_of_band`. Use for cryptographic-purist threat model.
- **Hybrid.** Combinations of the above. Most real deployments.
- **Browser / WASM.** A browser resolver-config MUST default to omitting `dns_txt` (no raw DNS/UDP in a browser) and `consensus_anchored` (no chain RPC) unless a proxy endpoint is configured in the backend's `hints` — otherwise resolution silently fails-closed mid-chain. `well_known_url` / `did_web` are **CORS-conditional** in a browser (HTTPS fetch to a foreign domain; same CORS gate as the CDN browser-deployment runbook). Browser-safe defaults: `local-name` + `self-certifying` + `out-of-band` + (CORS-permitting) `well_known_url` / `peer-issued` over an existing ws/wss session.

---

## §11 Cross-impl conformance

### §11.1 MUST implement

- `system/registry:resolve` handler with the contract per §2.
- `system/registry/binding` entity type — creation, signature verification, supersedes-chain validation, bare-hash field encoding per §3.
- `system/registry/resolver-config` entity type — load + meta-resolver dispatch per §2.2 + precedence per §4.1.
- Pinned-bindings precedence per §4.1.
- Cryptographic signature verification on non-self-certifying, non-local-name bindings.
- Honor observed revocations.
- Fail-closed on chain exhaustion.
- Forward-compatible unknown `backend_kind` handling per §4.2.
- Local-name backend (§6) — all five handler operations (`:resolve`, `:bind`, `:unbind`, `:list`, `:update-transports`); bind/unbind/supersede semantics; local-only storage; substrate's resolver-handler contract per §2.
- Peer-issued backend (§6a) — `peer_issued.resolve` per §6a.4 over transport-agnostic reads; signature-verify against the pinned trust root; by-name index (§6a.3); by-target revocation index (§6a.6); ttl + fail-closed (no pin-downgrade). Conformance vectors `REG-PEERISSUED-{RESOLVE,VERIFY-FAIL,REVOKED,EXPIRED,PRECEDE,OFFLINE-NOTFOUND}-1` — all six landed 3-way GREEN (Go/Rust/Python), injected-reader. The signed manifest (§6a.7) is NOT a v1 MUST (format pinned, impl deferred); `REG-PEERISSUED-MANIFEST{,-ABSENCE}-1` gate it when an impl ships it.
(Resolution-log is SHOULD per §11.2 — moved out of MUST to avoid the write-amplification hot-path issue surfaced by the cohort review.)

### §11.2 SHOULD implement

- TTL-based cache invalidation.
- Resolver-config validation on load (reject malformed chains).
- Case normalization per `local-name-config.case_normalization`.
- UI / CLI surface for local-name bind / unbind / list.
- **Resolution logging** at the canonical path `system/registry/resolution-log/{seq}` for inspectability — one entry per top-level `meta_resolve` invocation. Scope rules:

```
type: "system/registry/resolution-log"
data: {
  seq:                   <uint>,                       ; per-peer monotonic, persistent across restarts
  name:                  <string>,                     ; the queried name
  backend_id:            <peer_id_hash | identifier | null>,  ; which backend answered (null if chain_exhausted)
  status:                "resolved" | "not_found" | "chain_exhausted",
  reason:                <string | null>,              ; e.g. "signature_failed", "policy_rejected", "pin_short_circuit" — null if status=resolved by normal path
  binding:               <hash | null>,                ; resolved binding, if any
  attempted_at:          <ms-since-epoch>,
  is_fallback_reresolve: bool                          ; true if invoked from §2.3 transport-fallback loop
}
```

  - **One log entry per top-level `meta_resolve`.** Cache hits MAY be elided per `resolver-config.log_cache_hits` knob (default `false`).
  - **Per-backend inner attempts are NOT separately logged** (validation failures driving chain advancement are absorbed into the top-level entry via the `reason` field on the final outcome).
  - **Transport-fallback re-resolves are tagged `is_fallback_reresolve: true`** and are NOT counted toward the resolution-log's per-call sampling budget (a flapping endpoint MUST NOT write per-retry to the content store on the hot path).
  - **`seq` scope** — per-peer, monotonic across restarts; recovered on startup by walking `system/registry/resolution-log/` and taking max+1; impls MAY accelerate with a persistent counter.
  - **No log-of-log recursion** — writing a `resolution-log` entry is not itself a name resolution and does not trigger log emission.
  - **Retention** — ring-buffer per `resolver-config.resolution_log_capacity` knob (default 1024 entries); ring-buffer eviction does not affect the per-peer monotonic seq.
  - **Reason field** distinguishes "no backend had it" from "found but signature/policy rejected" — the debugging signal the log exists for.

### §11.3 MAY implement

- Additional backends (did-web, dns-txt, peer-issued, dht, consensus-anchored, aggregator) per their own proposals.
- Custom name-format dispatch rules.
- Conflict-annotation entities at aggregator layer.
- Local-name namespaces (multiple local-name stores per peer); v1 spec is single-store.
- Local-name import/export between a peer's own devices (not cross-peer).

### §11.4 MUST NOT

- Silently accept unsigned non-self-certifying / non-local-name bindings.
- Use cached bindings past their computed expiry (`issued_at + ttl`; null `ttl` = never expires).
- Use revoked bindings when revocation is observed (per §3.1).
- Construct authority chains beyond what backends provide.
- Sync local-name bindings to other peers.
- Publish local-name bindings to any external registry.
- Require signature on local-name bindings.

---

## §12 What this extension does NOT cover

- **Backends beyond local-name + peer-issued.** DID:web, DNS-TXT, DHT, consensus-anchored, aggregator each live in their own proposal. Local-name (§6) and peer-issued (§6a) ship in this spec as the v1 concrete backends; others sequenced.
- **`domain-control` challenge format (peer-issued live registration).** §6a.9 folds live registration (`register-request` + issuer-policy `open`/`allowlist`/`manual`); the one deferred piece is the `domain-control` mode's DNS-proof challenge format, which is co-designed with the web-native `dns-txt` / `well_known_url` backends so there is one domain-proof mechanism, not two. Lands with that proposal.
- **Outbound DID/DNS bridge — v1-DEFERRED.** This extension *consumes* DIDs (`kind: did-web`) and *consumes* DNS-TXT records. The reverse direction — presenting an entity peer AS a resolvable `did:key` (peer-id is multibase-encodable; the bridge is nearly free given §1.5 multikey) or as a `did:web` static document (one more artifact in §7.4's coral-reef tree) — is named here so the absence is explicit deferral, not oversight. Follow-on proposal when prioritized.
- **Name-format normalization** beyond local-name's NFC + optional case-fold. Backends MAY normalize per their own rules.
- **Caching policy** beyond the TTL hints in `ResolutionResult`. Application territory; caching strategy per deployment.
- **Anti-squatting / abuse prevention.** Per-backend concern; substrate has no opinion.
- **Privacy of the resolver query** beyond the §4.1 `name_format_dispatch` filtering. Per-backend (e.g., encrypted DNS-over-HTTPS at DNS-TXT backend; oblivious DHT for DHT backend).
- **Discovery (peer-finding).** Distinct concern; lives in sibling extension `EXTENSION-DISCOVERY.md`.
- **Address-discovery (`runtime-peer-endpoint`) — reverse peer_id → endpoint lookup.** Lives in EXTENSION-IDENTITY amendment, separately authored. `:resolve` is name-keyed by contract (§2.1 / §2.3).
- **Cross-peer local-name sharing.** Local-names are local; sharing is a different mechanism (group-shared registry).
- **Local-name syncing between user's own devices.** Out of v1 scope.
- **Ergonomic SDK seam (`browse_resolve`, `browse_reach`, `browse_fetch`, …)** — belongs to W1 (Outer Limits / SDK / Application). This extension stops at the handler contract; the L5 browse SDK consumes it.

---

## §13 Cross-references

- `proposals/implemented/PROPOSAL-EXTENSION-REGISTRY-SUBSTRATE.md` — landed-into source proposal (substrate)
- `proposals/implemented/PROPOSAL-EXTENSION-REGISTRY-PETNAME.md` — landed-into source proposal (local-name backend = §6 of this spec)
- `proposals/implemented/PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND.md` — landed-into source proposal (peer-issued backend = §6a of this spec; Part B.live registration deferred)
- `proposals/PROPOSAL-EXTENSION-DISCOVERY.md` — sibling-but-distinct (peer-finding, not name-binding); DRAFT
- `proposals/PROPOSAL-EXTENSION-RELAY.md` — RELAY proposal (compositions referenced §8)
- `proposals/PROPOSAL-STATIC-PEER-HOSTING-UMBRELLA.md` — names REGISTRY as dependency; this spec fulfills
- `core-protocol-domain/specs/extensions/network-peer-extensions/EXTENSION-NETWORK.md` — §6.5 transport profiles (`endpoint` shape consumed by §3 `transports` field)
- `core-protocol-domain/specs/extensions/network-peer-extensions/EXTENSION-IDENTITY.md` — peer-id substrate + identity publish surface
- `core-protocol-domain/specs/extensions/network-peer-extensions/EXTENSION-ATTESTATION.md` — supersedes-chain discipline referenced in §3 / §6.5

---

## §14 Open questions (informative)

- **Q1: Aggregator conflict surfacing UX.** §8.3 names fail-closed-by-default + explicit-pin-override; aggregator conflict annotation is MAY. Worth per-impl review for actual deployment ergonomics when Mode A lands.
- **Q2: Cross-peer cache propagation.** When a binding is revoked at the source registry, how fast does the revocation propagate through aggregators + consumers? Bound by TTL; subscription-based for live registries; explicit refresh-on-use for cached. Per-backend.
- **Q3: Identity-rotation interaction.** When the publisher of a peer-issued binding rotates their identity, do existing bindings remain valid? Per EXTENSION-IDENTITY §9.5 cap-survival semantics: yes, cap chains rebind; the published binding is signed by the cert at issuance; that cert remains live or is properly superseded via supersedes-chain. Worth cross-checking against EXTENSION-IDENTITY in cross-impl review.
- **Q4: Local-name-store size limits.** Operator concern; exposed as `max_local-names` knob (default unlimited).
- **Q5: Local-name namespace partitioning.** §11.3 MAY but undefined; defer to revision when a driver emerges.
