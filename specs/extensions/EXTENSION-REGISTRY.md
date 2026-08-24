# EXTENSION-REGISTRY

**Version**: 1.6
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.40+); EXTENSION-ATTESTATION.md (v1.3+) — the supersedes-chain discipline that binding revocation and superseded-binding retention are defined against (§3, §6.5, §7)
**Related**: EXTENSION-RELAY.md (Mode S can host a registry peer's tree; Mode A gates cross-registry federation, deferred from v1 — §8.2); EXTENSION-CONTENT.md (binding entities live in the content tree); EXTENSION-DISCOVERY.md (the sibling mechanism — peer-finding, not name lookup); EXTENSION-NETWORK.md (bootstrap endpoints)
**Tier:** Operational — Tier 2b (network), per `core-protocol-domain/specs/SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.

> ## ⚠ COMPLETENESS — this extension is **v1, NOT finished**
> The substrate + the two concrete v1 backends are landed and implemented. Several pieces are **specified-and-deferred** or **not-yet-designed**. Do **not** read "Landed" as "complete."
>
> **✅ Landed + implemented (v1):** resolver substrate (§2–§5); local-name backend (§6); peer-issued resolve + curated registration (§6a.1–§6a.8).
>
> **🟡 Design folded, build in flight / deferred:** **service advertisement (§3b — ratified + folded 2026-08-14 (v1.5), from `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`, DRAFT since 2026-07-22). The `system/registry/service-advertisement` entity and the §3b.1 `services` field are built in NO tree — source-read in all three 2026-08-14 (`entity-core-go` `8765e1f`, `entity-core-rust` `cc6cb56`, `entity-core-py` `808d9e6`); a dated observation, not a standing fact. What IS built three-way is §3b.2/§3b.3's rendezvous-hash *selection function over a caller-supplied pool* (`ext/signaling/pool.go`, `extensions/signaling/src/pool.rs`, `signaling/pool.py`) — so all three select correctly from a pool nothing can yet deliver. The build ask is the entity + the resolve field; the selection half is already green.** Peer-issued live registration `open`/`allowlist`/`manual` (§6a.9 — buildable now, cohort dispatch in flight); the manual-approval path (**§6a.9.3 — ruled 2026-08-13 (v1.3), corrected 2026-08-14 (v1.4). Built in all three within a day of the ruling and measured green: `entity-core-go` `7e0fb7c`, `entity-core-rust` `4107c32`, `entity-core-py` `808d9e6`; go's `registry_issuer` oracle 27/27 against each sibling, 0F. Source-read in each tree 2026-08-14, not carried from a report — a dated observation, not a standing fact: re-read the peers' trees before citing it (`docs/DOCTRINE-COHORT-STATE-TRACKING.md` D8).** The four gaps the builds exposed — the `denied` status enumeration, the un-typed decision input, the superseded-head code, and supersession observability — are **ruled in v1.4** and are the remaining fold for all three); signed binding-manifest impl (§6a.7 — format locked, impl deferred).
>
> **🔴 NOT yet designed — outstanding work before this extension is "done":**
> - **A credential channel for `data_relay`** (§3b.0a, **new in v1.6**) — §3b advertises where a relay is and whether a peer may use it, and specifies nothing that authenticates a peer to it. **`policy: open` is therefore the only interoperable data-relay deployment; `members` and `metered` are reserved shape, not usable capability.** Needs an issuer, a rotation model, and a delivery route for a secret — none of which the endpoint-shaped machinery of §3b extends to.
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
  backend_id:    <peer_id_hash | identifier>,           ; which backend produced this
  services:      <system/registry/service-advertisement | absent>
                                                        ; OPTIONAL — the deployment's shared infrastructure
                                                        ; set (§3b). Absent is a valid floor (§3b.1).
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

**Hash-field shape.** All bare-hash fields in this entity — `supersedes`, `issuer_attestation`, and `system/registry/revocation.revokes` — are **bare `system/hash`** values (format byte + digest — 33 bytes under `0x00`/SHA-256, 49 under `0x01`/SHA-384; **the length follows the format byte and MUST NOT be fixed**, `SPECIFICATION-FORMAT.md` §8.4.5), NOT wrapped in any envelope or object. Conformance: impls MUST NOT wrap or double-encode them. **The conformance claim here is the *shape* — bare and unwrapped — never the width**; a CBOR `bstr` is self-delimiting, so nothing downstream needs the length to parse the field.

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

## §3b The service-advertisement entity

A binding answers *who a name is and how to reach them*. It does not answer **what shared
infrastructure a deployment offers** — the STUN reflector and signaling/rendezvous carrier a peer
needs to *establish* a direct connection, and the optional relay fallbacks. Before this section
that infrastructure was out-of-band configuration, and the consequence was concrete: all three
implementations built §3b.2's pool-selection rule and had no protocol way to learn a pool to select
from. The browser-leg face of it is a peer negotiating with host candidates only, because nothing
fills its ICE server list.

A deployment publishes **one signed, static-served `system/registry/service-advertisement`** in its
registry zone, alongside its bindings. It is the entity-native SRV-analog to §3's A-record and
`EXTENSION-RELAY.md` §3.5's MX — the same one-signed-zone pattern, one more record type.

```
system/registry/service-advertisement := {
  deployment: <peer_id>,        ; the deployment/registry identity this set belongs to
  services: {
    reflector:    [+ {endpoint: primitive/string, priority: uint}]   ; STUN URI (§3b.0) — core path
    signaling:    [+ {endpoint: primitive/string, priority: uint}]   ; carrier URL (§3b.0) — core path
    ? data_relay: [+ {endpoint: primitive/string, priority: uint,
                      policy: "open" / "members" / "metered"}]       ; TURN URI (§3b.0) — optional
    ? inbox_relay:[+ {peer_id: <peer_id>, priority: uint}]           ; RELAY Mode-S — optional
  },
  ttl: uint                     ; ms since Unix epoch, per §2.1
}
```

### §3b.0 `endpoint` is a URI string, and a reflector is not a peer `[cross-peer seam — MUST; corrected 2026-08-14]`

**`endpoint` here is a `primitive/string` carrying a URI — it is NOT an `EXTENSION-NETWORK.md` §6.5 endpoint object.** Per service type:

| Service | `endpoint` form | Reference |
|---|---|---|
| `reflector` | **`stun:<host>[:<port>]`** or `stuns:` — **RFC 7064**. Non-hierarchical: there is **no `//`**. | RFC 7064 §3 |
| `data_relay` | **`turn:<host>[:<port>][?transport=udp\|tcp]`** or `turns:` — **RFC 7065**. Also non-hierarchical. | RFC 7065 §3 |
| `signaling` | a scheme-prefixed carrier URL — `ws://`, `wss://`, `tcp://host:port` | matches `EXTENSION-NETWORK.md` §6.5.1a D4's inner `url` **value**, as a bare string |
| `inbox_relay` | *(no endpoint — carries `peer_id`)* | §3b |

**Why not §6.5, stated as a category rather than a type mismatch.** A §6.5 transport profile describes how to reach an **entity peer**: it carries `supported_ops`, `freshness`, `nonce_required`, and `cap_flow`, none of which mean anything for a STUN reflector. **A reflector is not a peer.** It speaks RFC 5389 over UDP (`EXTENSION-SIGNALING.md` §9.3), holds no identity, completes no handshake, and answers no entity operation — `EXTENSION-NETWORK.md` §6.7.1 already refuses to conflate the two in the other direction. A TURN relay is the same. Pointing this field at §6.5 was a **category error**, and the type mismatch below was its symptom.

**The mismatch it caused `[the reason this is a MUST]`.** §6.5.1a D4 pins the live form to the object `{url: "<scheme>://…"}`, while §3b.3 hashes "the advertised endpoint **string's** bytes exactly as published." **An object has no string's bytes**, so two implementers could each be textbook-correct and diverge — one hashing the CBOR encoding of the `{url: …}` map, the other the UTF-8 of the inner `url` value. Different bytes → different SHA-256 → different `argmax` → **the two peers select different `signaling` members and never meet**, which is the exact silent failure §3b.2 makes MUST and §3b.3 pins to the byte to foreclose, reintroduced one level down in *what identifies a member* — a free variable §3b.3's own preamble names.

**A second consequence, and the reason the form is pinned and not merely the type.** `"<scheme>://…"` **cannot express a valid ICE URL at all.** A browser hands these to `RTCIceServer.urls`, which requires the RFC 7064/7065 non-hierarchical form, and a malformed entry does not degrade — it **throws at `RTCPeerConnection` construction**, taking out the establisher rather than falling back to host-only. Publishing the URI in its final form means a consumer hands the **published bytes to the ICE agent verbatim, with no transform** — which is also the ENTITY-CORE-PROTOCOL.md §1.8 byte-preservation posture, since a re-encode on a boundary is *the* interop hazard.

It **MUST** carry a `system/signature` at the invariant-pointer path (V7 §3.5), verified against the
pinned deployment identity **exactly as a binding is** (§6a.4). A tampered or expired advertisement
**fails closed** — it is discarded entire, never partially honored and never silently downgraded to
"no services." Absence and rejection are distinguishable to the peer; only absence is a valid floor.

**Service semantics.**

- **`reflector`** — a STUN-role endpoint echoing a peer's observed public address
  (`EXTENSION-NETWORK.md` §6.7.1 is the native operation; `EXTENSION-SIGNALING.md` §9.3 is the
  unwrapped STUN listener). Stateless, core path.
- **`signaling`** — a rendezvous endpoint carrying offers/candidates between two peers
  (`EXTENSION-SIGNALING.md` §4). Transient per-handshake state only, core path. The payload MAY be
  ENCRYPTION-opaque; the carrier need not read it.
- **`data_relay`** *(optional)* — a TURN / RELAY Mode-C endpoint relaying the **data path** when a
  punch fails (symmetric NAT). `policy` declares who may use it and is **the budget lever**: a
  `metered` relay bills its operator for every byte, for the lifetime of every connection that falls
  back to it. Absent ⇒ the deployment offers no data-relay, and symmetric-NAT peers fall to
  store-and-forward or do not connect. **This is the field that answers "whose credential, and who
  pays"** for the browser leg's TURN half (§3b.4).
- **`inbox_relay`** *(optional)* — a deployment-shared Mode-S mailbox for offline delivery.
  **Distinct from a peer's own `system/peer/inbox-relay` MX**, which is per-peer; this is
  deployment-shared infrastructure a peer MAY adopt.

**Why deployment-scoped and not per-peer.** Reflector and signaling are *shared* infrastructure, not
attributes of any one peer; advertising them once per deployment rather than once per peer matches
both reality and cost. A peer MAY still override per-peer where it genuinely differs — and the
connector-entered-by-URL path, which never performs a resolve at all, is served by
`EXTENSION-SIGNALING.md` §4.5 rather than by this entity.

### §3b.0a `data_relay` carries no credential, and that is a named gap — not a floor

**This section specifies where a data relay *is*, and whether a peer *may* use it. It does not
specify how a peer *authenticates* to it, and nothing else in this corpus does either.** The
`data_relay` member carries `endpoint`, `priority`, and `policy` — there is no username, no
credential, and no mechanism that mints one.

**What that costs, per policy:**

| `policy` | Usable today | Why |
|---|---|---|
| `open` | **Yes** | an unauthenticated relay needs nothing this field does not carry |
| `members` | **No** | admission is asserted but not provable — there is no credential to present |
| `metered` | **No** | the same, and this is the policy whose whole purpose is attributing billed bytes to a payer |

**So two of the three policies are declared and unreachable.** A consumer handed a `turn:` endpoint
under `members` or `metered` gathers **no relay candidates**, and — because ICE reports that as an
ordinary failure to find a path — the result is indistinguishable from a NAT that could not be
punched. **An implementation SHOULD refuse a `data_relay` endpoint it has no credential for, rather
than install it and fail silently**; a refusal at configuration time is diagnosable and a silent
empty candidate set is not.

**This is a deferral, stated so the absence is explicit rather than discovered.** Specifying a
credential channel means choosing an issuer, a rotation model, and a delivery route for a **secret**
rather than an endpoint — none of which the endpoint-shaped machinery of this section extends to.
Until it exists, **`policy: open` is the only interoperable data-relay deployment**, and the
`members`/`metered` values are reserved shape, not usable capability. **The credential channel is
upstream of any question about how relay provisioning reaches a peer or how often it is re-read** —
there is no rotating secret to route or refresh until something issues one.

### §3b.1 The `services` field on `ResolutionResult`

When the resolved zone carries a `service-advertisement`, `:resolve` returns it in the OPTIONAL
`services` field of `ResolutionResult` (§2.1). **The whole cheap core path is therefore learned in
the resolve a peer already performs** — no extra round-trip and no second lookup surface.

The field is **OPTIONAL and MUST-ignore when absent**: a Tier-0 static registry that advertises no
live services resolves normally with `services` absent, and that is a valid floor, not a degraded
result.

### §3b.2 Intra-pool selection is per-service-type `[cross-peer seam — MUST]`

Each service entry is a **pool**, so a deployment can scale a service horizontally. **How a peer
selects within a pool differs by service type, and getting `signaling` wrong is a silent cross-peer
bug** — so the rule is pinned per type, never left to "lowest priority wins."

| Service | Intra-pool selection | Why |
|---|---|---|
| **`reflector`** | **any member; SHOULD consult several and require agreement** (`EXTENSION-NETWORK.md` §6.7.1; `EXTENSION-SIGNALING.md` §9.3 states the same MUST). `priority` is a preference/failover hint only. | Every reflector independently yields the same fact. No pair-convergence needed. |
| **`signaling`** | **client-side rendezvous-hash (MUST)** — both peers compute the §3b.3 weight over the pool and independently select the **same** member. **NOT lowest-`priority`, and never a round-robin load balancer.** | Two peers must meet at the *same* carrier for the seconds of the handshake. Priority-order or an LB **splits the pair across servers and the punch never completes** — a silent never-meet, exactly like a key-derivation mismatch. |
| **`data_relay`**, **`inbox_relay`** | **priority-order failover (MX semantics)** — lowest `priority` first, next on failure. | Any relay carries the data; the *sender* alone picks. For a mailbox the recipient's MX **is** the shard map. |

**Signaling scales with zero shared state:** pool capacity is the sum of its members, each owning a
shard of the key space; a hot key is one handshake rather than a hot shard, and no member needs any
other's state.

### §3b.3 The weight function, pinned to the byte `[cross-peer seam — MUST]`

**"Highest-random-weight" is a family, not a function** — operand order, the digest, what identifies
a member, and how weights compare are all free variables, and **two implementations can each write
textbook-correct HRW and split every pair.**

> For a pool member advertised at `endpoint`:
>
> ```
> weight(k, endpoint) = SHA-256( k ‖ endpoint_bytes )
> member              = argmax over the pool, weights compared lexicographically
> ```
>
> - **`k`** is the 33-byte rendezvous key **exactly as derived** (`EXTENSION-SIGNALING.md` §3.1) —
>   the same bytes that go on the wire, never a re-hash of them.
> - **`endpoint_bytes`** are the **UTF-8 bytes of the `endpoint` string** (§3b.0) **exactly as
>   published** — no normalization, no case-folding, no scheme or default-port canonicalization,
>   and **never the CBOR encoding of any enclosing map**. Both peers read the *same* advertisement,
>   so byte-preservation makes them agree without either running a URL canonicalizer — and a
>   canonicalizer is precisely where two implementations drift apart (`EXTENSION-SIGNALING.md` §3.3
>   is the same rule for the key's own string inputs).
> - **No separator between the two, because `k` is fixed at 33 bytes**, which makes the
>   concatenation unambiguous by construction. Recorded explicitly so it is not read as the
>   missing-separator defect the `pair` key derivation had to fix.
> - **Highest weight wins**; ties break to the **lower `endpoint_bytes`**. A tie is a SHA-256
>   collision away and will never be observed, but `argmax` alone is not a total order and an
>   implementer must not have to invent the rest.
> - It is a **plain SHA-256, not the substrate content-hash primitive.** This weight is a comparison
>   scalar: it never appears on the wire and addresses no content, so the ECF `{data, type}` envelope
>   would be ceremony — and a *format-carrying* digest would reintroduce the home-format divergence
>   the key derivation exists to pin away.

**`priority` partitions; it does not weight.** Select the **lowest `priority` tier present in the
pool, then rendezvous-hash within that tier.** "Weight the hash" is explicitly **not** the rule: a
weighting function is itself unpinned bytes, and stacking a second invented rule on the first is
worse than not tiering at all. Both peers read the same advertisement, so both land in the same tier
before the hash runs.

**Stale-pool skew (SHOULD).** Two peers holding slightly different advertisements may hash to
different members. Each peer SHOULD try its **top-2** choices, which covers a single-member pool
delta cheaply. A "looking-for-you" beacon forwarded within the pool is the heavier mitigation and is
**not** v1 — add it only under measured skew.

#### §3b.3.1 What the selection vector MUST discriminate `[MUST]`

**A two-member pool is necessary and not sufficient.** It catches an implementation that computes
the weight *wrongly*. It **cannot** catch several implementations that each compute it *correctly
over different operands* — each is internally self-consistent, and a fixture authored by one of them
ratifies whichever operand its author chose. That is the cohort-consistency trap ([ADR-0012]: a
cohort all passing one author's vectors is cohort-consistent, **not** independent convergence),
sitting one level inside the very rule §3b.3 exists to pin. **Two members are not enough to
discriminate**, which is why the properties below are stated rather than a member count.

This section states the **properties the vector must have**. The vector's own bytes are **not
authored here** — they are generated and pinned by the conformance oracle, and cited
`N·0F @ <oracle-commit>` like every other published conformance number ([ADR-0012]). A digest
hand-written into prose cannot be checked by reading it, which is precisely the failure mode this
section exists to prevent.

The selection vector **MUST**:

1. **Use a pool of at least two members** in the same `priority` tier. `argmax` over one member
   returns that member whatever the weight computes, so every construction agrees and a green gate
   says nothing.
2. **Discriminate the operand (§3b.0/§3b.3).** The member endpoints MUST be chosen so that hashing
   the enclosing map's CBOR encoding, or the inner value of a `{url: …}` object, or a canonicalized
   form of the endpoint, yields a **different selected member** than hashing the endpoint string's
   published UTF-8 bytes. A vector every candidate operand passes tests nothing.
3. **Separate weight order from endpoint order.** The selected member MUST NOT be the
   lexicographically-first endpoint in the pool, so that an implementation sorting by endpoint
   rather than by weight fails.
4. **Exercise the tier rule (§3b.3)** with at least one member outside the lowest `priority` tier
   present, so that hashing before partitioning fails.
5. **Use a real rendezvous key.** `k` MUST be well-formed per `EXTENSION-SIGNALING.md` §3.1 —
   `varint(format) ‖ digest`, 33 bytes, with the format at the **SHA-256 floor `0x00`** and never
   the deriving peer's home format — and the vector MUST publish the §3.1 derivation inputs
   (`mode`, `mode_input`) **alongside** the key bytes, so `k` is auditable rather than asserted.
   A leading tag no conformant derivation emits yields a key an implementation may reject before it
   hashes anything, and a fixture that will not load discriminates nothing; opaque bytes that merely
   start at the floor parse cleanly and leave a reviewer nothing to check. **An implementation
   consumes the published bytes** — the inputs are there to be recomputed, not to make §3.1 a
   prerequisite for a selection test.

Properties 2–4 constrain which member wins and 5 removes `k` from the free variables, so the search
that satisfies all four runs over the endpoint strings, the `priority` assignment and the
`mode_input`. **That search is the oracle's work, and it is why the bytes are not authored here.**

> **`[§11.5-class]` — single-impl-invisible.** This is the same invisibility as
> `EXTENSION-SIGNALING.md` §7.2 `fire_at`, and it is why the pin went unnoticed while three
> implementations built the selection function against pools they configured themselves.

### §3b.4 Prefer the cheap path `[SHOULD]`

A peer establishing a connection SHOULD attempt, in order: **direct dial → reflector + signaling
punch → `data_relay` → store-and-forward via an `inbox_relay`.** This is simultaneously the correct
latency order and the budget-preserving one — a metered relay is touched only after the free path
fails.

> **Stated SHOULD, not MUST, and the reason is the MAY/SHOULD test.** Two peers ordering their paths
> differently still connect: the divergence costs the operator money, it does not split a pair or
> corrupt a seam, so it is not the latent interop bug a MUST exists to foreclose (contrast §3b.2,
> where a wrong choice silently never-meets and is therefore MUST). No conformance test can fail a
> peer for dialing a relay first without knowing the deployment's intent. Its **source proposal
> wrote MUST**; the fold downgraded it deliberately and this note records that, so the change is not
> read as a transcription slip.

### §3b.5 Trust of advertised infrastructure

A deployment **vouches for the infrastructure it advertises** — that is what signing the set means.
A peer MAY additionally pin or allow-list specific reflector / relay identities on top. Cross-deployment
shared community relays compose with the aggregator federation pattern (§8.2) and are **deferred with
it**, not resolved here.

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
system/registry/peer-issued:register-request(request)
    → system/registry/register-result  (200 approve | 202 queue)
    | system/protocol/error            (4xx reject)
  1. verify request signature by target_peer_id          ; layer-1 (always)
  2. apply issuer-policy admission (§6a.9.1)              ; layer-2 → approve | reject | queue
  3. on approve: registry-issue-binding(...) (§6a.8)      ; signs with K_registry, publishes, sets by-name pointer
     return 200 register-result { status: "bound", binding_hash }
  4. on reject:  error (name_taken | not_entitled | policy_rejected)   ; REGISTRY code domain
  5. on queue:   store the pending request (§6a.9.3)      ; manual mode
     return 202 register-result { status: "pending_review", pending_hash }
```

**Result type `[MUST]` `[RULED 2026-08-12]` — `system/registry/register-result`.**

```
type: "system/registry/register-result"
data: {
  status:        "bound" | "pending_review" | "denied",  ; which outcome; snake per STYLE (status code)
  binding_hash?: <system/hash>,                ; REQUIRED on "bound",         absent otherwise
  pending_hash?: <system/hash>                 ; REQUIRED on "pending_review", absent otherwise
}
```

> **`"denied"` added `[2026-08-14]` — the defect this box was written about, recurring one subsection
> later, in the section written to fix it.** §6a.9.3's operations table returns `register-result
> {status: "denied"}` from `deny-request`, and this declaration enumerated two values. **Both
> `binding_hash` and `pending_hash` are absent on `"denied"`** — the request is decided and nothing was
> published, so there is no hash to hand back; the decided head is reached through the by-request pointer
> (§6a.9.3). All three implementations emitted `"denied"` as the table said and recorded the
> enumeration as the stale half; **that reading is correct and is now the text.**
>
> **The recurrence is the finding, not the fix.** This box already stated the general rule — *"an
> operation whose declared return type does not enumerate every branch of its own pseudocode is an
> interop bug already in flight"* — and the very next subsection reproduced it, because the rule was
> written as prose next to one instance instead of as a check over the corpus. A rule that can only be
> obeyed by whoever remembers reading it does not bind the next author, who is usually the same author.
> Routed to the corpus gate as a mechanical rule (declared enumeration vs. emitted value).

> **Why this was a three-way divergence, and it is our defect `[2026-08-12]`.** The signature above previously read `→ binding_hash | rejection` — **two outcomes** — and then step 5 introduced a **third** in the pseudocode without extending the return type. `pending_hash` appeared **nowhere in this specification at all.** So each implementation invented a carrier: one a dedicated result type, one a result field, one `system/protocol/status`. **Three shapes is what an undeclared outcome produces**, and no implementation was wrong — there was nothing to be wrong against. **An operation whose declared return type does not enumerate every branch of its own pseudocode is an interop bug already in flight.**
>
> **`system/protocol/error` MUST NOT carry the 202.** An error entity denotes a *failed* operation; `202` denotes *accepted-pending*. Emitting one on the other makes the status line and the result type disagree by construction, and a client branching on result type reaches the opposite conclusion from one branching on status. *(Adopted from `entity-core-go`'s design argument, which stands on its own merits and not on how many implementations held it — see `GUIDE-CONFORMANCE` §4: the spec arbitrates, the cohort does not vote.)*
>
> **A generic status type is rejected on structure, not on taste.** The 200 must carry `binding_hash` and the 202 must carry a poll handle, so a carrier with no room for either forces the payload somewhere else and re-opens the divergence one field down. **`register-request` also MUST NOT borrow another operation's result type** because the payload happens to match — that coupling breaks silently the first time either operation's result grows a field.
>
> **The value's spelling does not change.** `pending_review` stays snake_case: `STYLE-NAMING-CONVENTIONS` puts **error and status codes** in snake regardless of which field carries them. Only the carrier is being ruled.

**`pending_hash` `[MUST]` `[RULED 2026-08-12]` — it names the stored pending request entity, not the request the client sent.** It is the `content_hash` of the `system/registry/pending-binding` entity the registry stored at step 5 (§6a.9.3), resolvable by the ordinary `tree:get` / `content:get` machinery every other registry read uses (§6a.3). **A handle the client can already compute is not a handle** — echoing the request hash tells the requester nothing it did not have before sending, and nothing is fetchable at it. The divergence here was real and undecidable from the text: one implementation named the stored entity, another named the request.

**Owed, named rather than invented `[2026-08-12]` — ✅ DISCHARGED `[2026-08-13]`, see §6a.9.3.** The manual-approval path itself — the `system/registry/pending-binding` schema, the by-request pointer a requester polls when it no longer holds the 202 response, and the operator's approve/deny operation — was **not specified anywhere in this document**, which is why step 5 could say "queue" and stop. That ruling pinned the cross-peer-observable surface (result type, status value, what `pending_hash` refers to) and deliberately stopped there. **Stopping there had a cost that is worth recording: it left `pending_hash` a `MUST` naming an entity with no schema**, so `entity-core-rust` withheld the value on principle while `entity-core-go`'s oracle failed peers for withholding it — the spec manufactured a conformance failure out of its own reserved section. **A `MUST` may not name a referent the corpus does not define**; if the referent must wait, the `MUST` waits with it. §6a.9.3 now defines it.

**Statuses `[MUST]` `[RATIFIED 2026-08-11]` `[RATIONALE CORRECTED 2026-08-12]`.** §6a.9 pinned the reject *codes* and left the *statuses* open, the way §6a.9.2 later pinned `400` / `501`. It is now text:

| Outcome | Status | Value | Carried by |
|---|---|---|---|
| layer-1 proof absent / wrong signer | **401** | `signature_invalid` | `system/protocol/error` `.code` |
| manual mode — queued for review | **202** | `pending_review` | `register-result` `.status` — **not an error code** |
| name already bound | **409** | `name_taken` | `system/protocol/error` `.code` |
| layer-2 admission refused | **403** | `not_entitled` \| `policy_rejected` | `system/protocol/error` `.code` |

> **The fourth column exists because its absence caused the divergence `[added 2026-08-12]`.** This table shipped with the third column headed **`Code`**, which is a category error on the `202` row: `pending_review` is a **status field value**, and §6a.9's own pseudocode said so (`on queue: status "pending_review"`) while the table said otherwise. **An implementation that trusted the table emitted an error entity on a 2xx** — a faithful reading, and the same failure shape as §5.4/§8.3, where the normative artifact and the prose disagreed and the artifact won. **A status table that names a value without naming what carries it is under-specified by exactly one column**, and the missing column is the one a wire implementer needs.

401 (not 403) for layer 1 follows V7 §5.2a's discriminator: an unverifiable signer is an **authentication** failure, and §4.2/§4.4's F32 ruling already put that class at 401. The `409` / `403` rows are **derived** from V7 §3.3's class rules rather than measured — the cohort converged on the first two rows' **statuses** only (and *not* on their codes — see the correction below), so treat these two as new and report a divergence rather than assuming it is yours.

> **Correction `[2026-08-12]` — the ratification rationale was wrong, and the error is worth more than the fix.** This paragraph previously read *"Three implementations converged on the same answers with no MUST to point at — go first, rust and py by deference."* **That was never verified against py's tree, and it is false.** What the cohort converged on was the **status**; the **code** diverged and still does. Read live 2026-08-12: `entity-core-go` `419a715` answers `signature_invalid` (`RegistryErrSignatureInvalid`, `core/types/registry_peerissued.go`); `entity-core-rust` `21eb223` answers `signature_invalid` (`REG_ERR_SIGNATURE_INVALID`, `extensions/registry/src/registration.rs`, converged at `0caf911` **from** `invalid_signature` — so rust did not agree at ratification time either); **`entity-core-py` `2c1aa1b` answers `401 proof_failed`** at all three layer-1 sites (`_error(status, code, message)` in `entity_handlers/registry.py` — `proof_failed` is the *code* argument, not the message). py's own comment still reads *"The spec pins neither code; this converges on core-go"* — true when written, now wrong twice over: the spec does pin it, and py matched go's **status** while diverging from go's **code**.
>
> **We asserted a cohort build-state fact inside a normative table without opening the tree** — the failure this repo's `AGENTS.md` foregrounds, this time committed by us, in the one place where a wrong build-state claim gets cited as authority. The rule stands unchanged (`signature_invalid` is normative and correct on its merits — see the 401 derivation above); only the claim that the cohort had already converged on it is retracted. **`entity-core-py` is non-conformant on this row and is owed that plainly.**

**Error *strings* are free; codes are the contract.** The `code` values above are normative and are what a peer branches on. Human-readable messages accompanying them are impl-local and MAY differ — **a conformance check MUST NOT assert on message text.** *(py observed the strings diverging three ways and left them deliberately, correctly noting no check reads them. Stated here so "no check reads them" stays a design choice rather than becoming a latent expectation.)*

**Conformance `[MUST]` `[RULED 2026-08-12]` — a check of a pinned row MUST assert the `code`, not the status alone.** A layer-1 refusal check that asserts only `401` cannot distinguish a conformant peer from one answering an unpinned code, so it scores the contract's *weaker* half and reports green on a divergence. This is not hypothetical: it is exactly why the py divergence above survived a full cohort cycle. In the cross-impl instrument that found it, the **layer-2** rows were asserted as `status != 403 || code != not_entitled` while every **layer-1** row was asserted as `status != 401` — the same file, two rows apart, one of them checking the contract and the other checking half of it. `REG-REGISTER-PROOF-1` / `REG-REVOKE-PROOF-1` / `REG-RENEW-PROOF-1` therefore assert **status + code + publishes-nothing**, all three.

**Scope `[extended 2026-08-12]` — this is not confined to the layer-1 row.** The cohort audit prompted by this ruling found the same status-only shape on two further pinned rows, both naming their code **only inside the check's own failure string**: **§6a.9's `202 pending_review`** and **§6.5's `409 bind_already_exists`**. The rule binds every row of every pinned status table in this specification, not the three named vectors.

> **And audit the extractor in the same pass — `[MUST]`, because a check can be unpassable by construction.** The same audit found the harness helper feeding these assertions harvested a `code` only when `status >= 400`, encoding *"codes ride failures."* **§6a.9 pins a code on a 2xx row**, so `pending_review` was silently dropped to `""` and that assertion **could not have passed against any peer, however conformant.** Moving the gate to `>= 300` does not fix it — 202 is still excluded; **the status class was never the right discriminator, the result's type is.** Gate on the body being error-shaped, keeping the status class only as a fallback trigger. This matters cohort-wide because the failure presents as a *sibling* bug: anyone auditing assertions without auditing their plumbing writes checks that cannot pass and then hunts a peer defect that is not there. Generalized at `GUIDE-CONFORMANCE` §5.2b.2.

> **This is the third member of the family §2.4a opened, and the shape is now stable enough to name.** `GUIDE-CONFORMANCE` §2.4a was *a surface the suite reaches and scores backwards* (asserting acceptance certifies the hole). `EXTENSION-NETWORK` §5.4a was *a reachable state no vector visits* (the §A1/§5.4 join). This one is *a pinned value nothing asserts* — the spec made `code` the contract and the instrument measured `status`. **Common root: what gets implemented and what gets scored are both driven by the vector list, not the prose** — the same finding §6a.9 already records one screen up, where all three impls skipped layer 1 on exactly the two ops with no named vector. **A row pinned in a table is not covered until a vector reads that row's value.**

Two follow-on ops, with explicit schemas (the design fold left these as bare "follow-on ops"; cohort impl surfaced the gap — each impl guessed a different shape, so they are pinned here):

- **`:renew-request` — replay-defended (carries `nonce` + `issued_at`).** Extends a binding's lifetime via the supersedes-chain. Signed by `target_peer_id` (layer-1).
  ```
  type: "system/registry/renew-request"
  data: { binding_hash: <system/hash>, ttl: <ms | null>, nonce: <bytes>, issued_at: <ms-since-epoch> }
  ```
  Renew has a **non-idempotent state effect** — each accepted renew extends the binding's expiry — so a captured renew can be **replayed to keep a binding alive past the registrant's intended lapse.** It therefore carries the same `nonce` + `issued_at` replay defense as `register-request` (§6a.9.1).

- **`:revoke-request` — NOT replay-defended.** Revokes a binding (emits a §3.1 revocation). **Signed by `target_peer_id` (layer-1).**
  ```
  type: "system/registry/revoke-request"
  data: { binding_hash: <system/hash>, reason: <string | null> }
  ```
  Revocation is **monotonic and target-pinned** — it acts on one content-addressed `binding_hash`, cannot be undone, and cannot target a later re-issued binding (different hash) — so replay is a provable no-op. `nonce` / `issued_at` are **not** part of the schema (they would be harmless but add no security and break cohort convergence).

> **Layer 1 binds all three write ops `[MUST]` — this was the hole `[RULED 2026-08-11]`.** `register-request`, `renew-request` and `revoke-request` each **MUST** verify the layer-1 signature by `target_peer_id` (for revoke/renew, the `target_peer_id` of the binding named by `binding_hash`) **before any state change and before any publication.** A request failing layer 1 MUST be refused **and MUST publish nothing** — no revocation, no superseding binding, no queue entry. *All three implementations shipped `revoke` and `renew` with no verification at all: any peer that could reach a registry could permanently revoke any binding in it, and revocation is monotonic, so there is no undo. It was found because core-py declined to converge and reported instead (go `e91817f`).* **Replay defense is not authorization** — `renew`'s `nonce`/`issued_at` stops a *captured* request being re-run while leaving a *fresh unsigned* one accepted, which reads as authorization at a glance and is not.
>
> **"Or the operator" is not a second wire credential.** The prior text read "signed by `target_peer_id` or the operator", and there is **no operator identity, no operator key, and no request field on this operation that an operator could populate** — so as a *wire* credential the phrase named nothing. The *wire* surface therefore accepts `target_peer_id` proof **exclusively**; an operator revokes by acting on its own registry through its own capability.
>
> **The justification is irreversibility, not the absence of an operator-gated operation.** `system/capability/registry-issue-binding` **does** gate operations reachable over the wire — §6a.9.3's `approve-request` / `deny-request` are exactly that, an operator decision driven by a dispatched request under that capability. So "the operator's authority is local, therefore no wire surface" would prove too much, and it would be falsified by this document's own §6a.9.3. What actually decides it is the direction of the error: this is the op where guessing permissively is **unrecoverable**. A registry that widens its accepted proof later un-accepts nothing; one that guessed wide has already handed out permanent denial-of-name. This is the intersection of the three readings an implementer could have taken, and it is chosen deliberately for the op where guessing permissively is unrecoverable: a registry that widens later un-accepts nothing, while one that guessed wide has already handed out permanent denial-of-name. *(core-go implemented exactly this reading as its interim and routed the ambiguity rather than picking — `2026-08-11-revoke-names-an-operator-with-no-proof-shape.md`.)*
>
> **Conformance — `REG-REVOKE-PROOF-1` / `REG-RENEW-PROOF-1` (new).** Each requires **both halves**: a valid layer-1 proof is accepted, **and** an absent-or-wrong-signer proof is refused **and publishes nothing**. §6a.9 previously named `REG-REGISTER-PROOF-1` and nothing for the other two ops, and **all three implementations skipped layer 1 on exactly the two ops with no named vector** — the vector list, not the prose, is what got read. See `GUIDE-CONFORMANCE.md` §2.4a: a check that asserts only acceptance certifies the hole.

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

`registry-issue-binding` (the internal sign+publish act) is gated by `system/capability/registry-issue-binding`, held by the policy logic / operator only. `register-request` is the *external* surface, gated by `system/capability/registry-request-binding` (open → granted broadly; allowlist → narrow). `system/capability/registry-manage-issuer-policy` gates editing the policy — **via the two operations defined in §6a.9.2.**

##### §6a.9.2 Managing the policy — `set-issuer-policy` / `get-issuer-policy` `[RATIFIED 2026-08-10]`

> **This capability named an act the corpus never defined**, and all three implementations diverged into the vacuum: go `2d6c993` and rust `b8e0ae2` expose only the three registration ops and arm the policy **out-of-band** (a CLI flag or a direct entity write); py `e60c822` invented `set-issuer-policy` / `get-issuer-policy` (`manifest.py`). **A client written against this spec had nothing to call anywhere.** Python's names are the obvious ones and the payload type already existed, so its shape is ratified rather than replaced — it costs one subsection and converges three implementations.

| Operation | Input | Output | Gate |
|---|---|---|---|
| `set-issuer-policy` | `system/registry/issuer-policy` | `system/registry/issuer-policy` (the stored policy, as written) | `system/capability/registry-manage-issuer-policy` |
| `get-issuer-policy` | none — the §3.2 empty-params shape | `system/registry/issuer-policy`, or `404 not_found` when unset | `system/capability/registry-manage-issuer-policy` |

- **`set-issuer-policy` replaces the policy whole `[MUST]`** — it is not a partial merge. An absent optional field means *unset*, not *unchanged*; a merge semantics would make the resulting policy depend on write order, which two peers cannot reconstruct.
- **Resolution order is store-first `[MUST]`.** The issuer reads `system/registry/issuer-policy` from its tree; out-of-band arming (a CLI flag, an operator write) is a **seed for that entity**, never a parallel source consulted at request time. *(This order is load-bearing and predates the operations: it is what lets a conformance run drive all three modes against a **single** peer by writing the entity, which is how `registry_issuer` reached 12 checks — core-go `559f44c`. An implementation that let a flag shadow the stored entity would make that untestable.)*
- **Unset is not a mode.** With no policy entity stored, the registry does not run live registration at all (§6a.9's handler is unregistered) — it is a conformant curated-only registry per §6a.8. `get-issuer-policy` returns `404`; it MUST NOT synthesize a default `open`, which would silently turn a curated registry into a first-come-first-serve one.
- **`domain-control` remains deferred** (§6a.9.1) — `set-issuer-policy` MUST reject `mode: "domain-control"` with `400 unsupported_mode` until the challenge format lands, rather than storing a policy it cannot enforce.
- **A stored `domain-control` policy fails closed with `501` `[MUST]`** *(ratified 2026-08-10 (b); core-go read it this way and asked)*. The `400` above binds `set-issuer-policy`, which refuses to *store* the mode; it does not answer what a registry does when the entity is already there — seeded out-of-band, written directly to the tree, or predating the refusal. **The registry MUST answer live registration `501 unsupported_mode` and MUST NOT fall back to `open`, `manual`, or an unset-style `404`.** Falling back to `open` turns an operator's unenforceable curation into first-come-first-serve, which is the §6a.9 threat model exactly inverted; falling back to `404` reports "no policy" while a policy is stored. This is a cross-impl-observable answer with four plausible codes, so it is pinned rather than left to converge.
- **`get-issuer-policy` takes no params content**, so callers send the `ENTITY-CORE-PROTOCOL.md` §3.2 **empty-params shape** — a `primitive/any` entity whose `data` is canonical-CBOR `a0`. It is **not** a zero-value entity (rejected `400 invalid_params` at the envelope layer, before the handler) and **not** a `primitive/map` (a handler SHOULD reject a mismatched params *type* with `400 unexpected_params`). *Stated here because a unit test that calls the handler directly never crosses the envelope layer and cannot see either failure — core-go found both on first contact with a live peer.*

**Replay defense (normative discriminator).** A signed request carries `nonce` + `issued_at` (the registry tracks seen `nonce`s per requester within an `issued_at` window; a replayed request is rejected) **iff replay has a non-idempotent state effect.** This holds for `register-request` (replay can roll a name back to a superseded binding) and `renew-request` (replay can extend a binding's life past intended lapse). It does **not** hold for `revoke-request`, which is monotonic on a content-addressed target (replay cannot un-revoke and cannot reach a later re-issued binding) — so revoke omits `nonce` / `issued_at`. The discriminator, not the op name, decides: future ops are replay-defended exactly when their replay mutates state.

**Conformance:** `REG-REGISTER-PROOF-1` (signature not by `target_peer_id` → rejected), `REG-REGISTER-POLICY-1` (allowlist reject → `not_entitled`; allow-listed → issued + resolvable), `REG-REGISTER-REPLAY-1` (seen nonce → rejected). `REG-REGISTER-DOMAINCTRL-1` gates the deferred `domain-control` mode; **`REG-ISSUER-DOMAINCTRL-STORED-1`** gates the fail-closed `501` above (write the policy entity directly, then attempt live registration — the `set-issuer-policy` refusal cannot be the only thing standing between a stored unenforceable mode and an open registry).

**Implementation status:** the **design is pinned here**; the `open` / `allowlist` / `manual` modes are buildable now (no external dependency); `domain-control` waits on the web-native domain-proof co-design. A registry shipping curated-only (§6a.8) is conformant — it simply does not run the handler.

##### §6a.9.3 The manual-approval path — `pending-binding`, the by-request pointer, approve / deny `[RULED 2026-08-13]`

**Reserved on 2026-08-12, filled here.** The 08-12 ruling pinned *what `pending_hash` refers to*
— the `content_hash` of the stored `system/registry/pending-binding` — and then stopped, because
that entity had **no schema anywhere in this document**. That left `pending_hash` naming a shape
nobody had defined, and the three seats split accordingly: **Rust deliberately emits no
`pending_hash` at all** (`extensions/registry/src/registration.rs:303` — REQUIRED by the ruling,
withheld because the referent is unspecified), **Python invented the entity and an
`approve-request` operation** (`entity_handlers/registry.py:1188`, `:1303`), and **core-go's
oracle FAILs a peer that returns no `pending_hash`** (`cmd/internal/validate/registry_issuer.go:401`).
So the reference oracle currently fails a seat for withholding a value the spec gave it no way to
compute. **A `MUST` whose referent is unspecified is not a requirement, it is a trap**, and this
section removes it.

*Python's shape was ratified rather than replaced — at ruling time it was the only worked implementation
and it already followed §6.3's body/pointer split. **One seat is not convergence**, so the additions
below (the by-request pointer, the decision states, deny, retention) were arch's design and were marked
unbuilt. **They are built in all three as of 2026-08-14** (pins in the outcome note at the end of this
section); the marking is kept because it is what the reader needs to know about how the section was
derived, not because the build state is still open.*

**The storage shape follows §6.3 exactly — an immutable content-addressed body plus a mutable tree
pointer.** This is not a new pattern; re-deriving one here is how the two would drift.

```
type: "system/registry/pending-binding"
data: {
  name:           <string>,                  ; name-path safety per §6.3
  target_peer_id: <Base58 peer-id, V7 §1.5>,
  transports:     [<endpoint per NETWORK §6.5>],
  requested_ttl:  <ms | null>,
  queued_at:      <ms-since-epoch>,
  status:         "pending_review" | "approved" | "denied",
  binding_hash?:  <system/hash>,             ; REQUIRED on "approved", absent otherwise
  reason?:        <string>                   ; OPTIONAL on "denied"; operator-supplied, never parsed
}
```

- **Body** at `system/registry/pending/{pending_hash}` — immutable, content-addressed, matching
  §3's universal `binding/{binding_hash}` rule.
- **Pointer** at `system/registry/pending/by-request/{target_peer_id}/{name}` — holds the bare
  `system/hash` of the **current head** pending-binding. This is the *"by-request pointer a
  requester polls when it no longer holds the 202 response"* the 08-12 note named as owed.

**`target_peer_id` precedes `name` in the pointer path, and the order is normative.** A
`target_peer_id` is a single Base58 segment; a `name` is name-path-safe but **not guaranteed
single-segment** (§6.3 pathes names directly, and `binding/local-name/{name}` already relies on
that). Putting the variable-depth value **last** keeps the prefix parseable and keeps
`pending/by-request/{peer}/` enumerable. *(Same failure this corpus corrected in
`EXTENSION-NETWORK` §4.1 the same day: a multi-segment value in a non-terminal position makes the
path unwalkable at a fixed depth.)*

**`pending_hash` is registry-local and its bytes need no cross-peer agreement `[MUST NOT be
gated on reproducibility]`.** `queued_at` is a local wall-clock reading, so two registries will
not compute the same hash for the same request — **and they never need to.** A pending-binding is
**pre-decision, single-registry state**: no second registry stores it, aggregators do not
re-publish it (§8 republishes *bindings*), and the only obligation is that the hash **resolves at
the registry that minted it**. This is stated explicitly because the corpus's other content-hash
rulings run the opposite way (`EXTENSION-COMPUTE` §2.4 materialized errors must agree byte-for-byte
cross-impl), and an implementer generalizing from those would strip `queued_at` to chase a
determinism this surface does not require.

**One pending head per `(target_peer_id, name)` `[MUST]`.** A `register-request` that queues while
a pending head already exists for the same pair **supersedes** it: a new body is written, the
pointer repoints, and the 202 returns the **new** `pending_hash`. Replace-whole, never merge —
the same rule and the same reason as §6a.9.2's policy write, and it is what stops a requester's
retries (each of which carries a fresh `nonce`, so each is a distinct request by construction)
from filling an operator's queue with duplicates of one intent.

**Operator decisions** — both gated by `system/capability/registry-issue-binding`, the capability
§6a.9.1 already defines for the internal sign-and-publish act. **No new capability**: approving a
queued request *is* issuing a binding.

| Operation | Input | Output | Effect |
|---|---|---|---|
| `approve-request` | `{pending_hash}` | `system/registry/register-result` `{status: "bound", binding_hash}` | issues the binding (§6a.8), writes a new body `status: "approved"` carrying `binding_hash`, repoints |
| `deny-request` | `{pending_hash, reason?}` | `system/registry/register-result` `{status: "denied"}` | writes a new body `status: "denied"`, repoints; nothing is signed or published |

- **`404 not_found`** when `pending_hash` names no stored pending-binding.
- **`409 name_taken`** when the name was bound by someone else between queue and approval `[MUST]`
  — the queue is not a reservation, and a registry that issued anyway would silently overwrite a
  live binding. *(Python already returns exactly this.)*
- **A decision on an already-decided request returns `409 already_decided`** — approve and deny are
  not idempotent-by-replay, and re-approving would mint a second binding for one request.
- **A decision on a *superseded* head returns `404 not_found` `[MUST]` `[RULED 2026-08-14]`.**
  Supersession repoints the by-request pointer and deliberately leaves the prior body in the store for
  audit, so a stale `pending_review` body stays fetchable forever and an operator holding an old
  `pending_hash` can address it. Deciding it would issue a binding on terms the operator's queue no
  longer shows and leave the pointer naming a different head than the one decided — an inconsistency no
  later read untangles. `already_decided` is wrong (it was never decided) and `name_taken` is wrong (the
  name is free), so this reuses the pinned `404` rather than inventing a cohort-divergent code.
  **"One pending head per pair" is a rule about what is *decidable*, not only about what is listed** —
  that sentence is the one the section was missing. *(All three implementations reached this reading
  independently and two record that their first draft let the case through, which is why the vector
  below matters more than the code choice.)*
- **The decision states are the reason deny is not a delete `[MUST]`.** A denied request MUST leave
  a `status: "denied"` head reachable through the by-request pointer; the registry MUST NOT simply
  remove the entry. A requester polling a vanished pointer cannot distinguish *denied* from *never
  received* — that is a silent drop, which the substrate floor forbids (deliver-or-signal). Python's
  current approve path removes the entry on issue; **that is the one place its shape is not
  ratified**, and it is a spec-side correction rather than a defect it should have caught.
- **No `list-pending` operation.** An operator enumerates `system/registry/pending/by-request/`
  with the ordinary `tree`/`query` machinery, exactly as every other registry read works (§6a.3).
  Adding an operation for a list a tree walk already answers is the live-registry cost the coral-reef
  posture (§7.4) exists to avoid.

**The decision operations take an un-typed input, and handlers MUST decode by shape `[MUST]`
`[RULED 2026-08-14]`.** Every other write operation on this handler names a `system/registry/*` params
type; these two deliberately do not. **No implementation may register a type definition for
`system/registry/approve-request` or `.../deny-request`** — the names are not carried by this
specification, and publishing a definition for one would manufacture a cross-impl type-census divergence
out of a spec gap, making the divergence the publisher's. A handler MAY name a local type on the wire for
its own dispatch, but **MUST NOT assert the params type on receipt**: an operator tool sending a
differently-typed params entity carrying `pending_hash` interoperates. *(This is the narrow exception,
not the pattern. It is chosen because the alternative — arch inventing two type names right now — pins a
wire surface that no operator tooling exists to consume, and the corpus's own history says an invented
name is the one the next implementation invents differently. When operator tooling lands, the types land
with it.)*

**Supersession must be observable, and the schema alone does not make it so `[MUST]`
`[RULED 2026-08-14]`.** A `register-request` retry carries a fresh `nonce`, but `pending-binding` does
**not** carry the nonce, so two retries of one intent inside a single millisecond encode to identical
bytes and content-address to **one** body — at which point the 202's "new `pending_hash`" is the old
`pending_hash` and a superseding write is indistinguishable from a no-op. **That collapse is correct and
intended** — one head, one hash, and adding the nonce to the body would defeat the dedup for no gain.
What follows from it is a conformance obligation, not a schema change: **`REG-PENDING-DECIDE-1`'s
supersession half MUST vary a field the schema actually carries** (`requested_ttl` or `transports`), and
an implementation MUST NOT rely on the returned `pending_hash` *changing* as its supersession signal.
The observable invariant is the one stated above — **exactly one head reachable through the pointer for
the pair** — which holds whether or not the two bodies collide. *(Raised by `entity-core-rust` while
building the vectors; recorded because the next author to write a supersession test will reach for the
nonce first.)*

**Retention `[SHOULD]`.** A decided pending-binding (`approved` / `denied`) is GC-eligible after a
configured retention window; the body stays content-addressed and auditable independently of the
pointer. A **`pending_review`** head is **never** GC-eligible — it is live queue state. *(A
MUST-write paired with an unbounded queue is a leak by construction; the retention knob is what
keeps the queue an operator's inbox rather than a log.)*

**Conformance — `REG-PENDING-HANDLE-1` / `REG-PENDING-DECIDE-1` (new).** Per `GUIDE-CONFORMANCE`
§2.4a both halves are required, and the negative half is the load-bearing one:

- **`REG-PENDING-HANDLE-1`** — `mode: manual`; a valid request returns **202** with a
  `pending_hash` **distinct from the request's own `content_hash`**, and that hash **resolves** to
  a `system/registry/pending-binding` whose `status` is `pending_review`. The by-request pointer
  resolves to the same body. *(The distinctness half is already asserted by core-go's oracle; the
  resolvability half is what the schema makes assertable and is the reason this section exists —
  a handle that names nothing fetchable was indistinguishable from a conformant one.)*
- **`REG-PENDING-DECIDE-1`** — approve issues a binding that resolves by name **and** leaves an
  `approved` head; deny leaves a `denied` head **and publishes nothing** (assert the name does not
  resolve — a deny that silently issued would pass an outcome-only check); a second decision on
  either returns `409 already_decided`; a superseding request leaves exactly **one** head for the
  pair; and **a decision on the superseded head returns `404 not_found`** — the case the section's own
  supersession rule creates, which neither of the other assertions visits (`GUIDE-CONFORMANCE` §5.4a: a
  reachable state no vector visits). The supersession steps MUST vary a field `pending-binding` actually
  carries (`requested_ttl` / `transports`), never the request `nonce` — see the observability ruling
  above, or the two bodies collide and the assertion is vacuous.

**Outcome, measured `[2026-08-14]`.** All three implementations built this section within a day of the
ruling — `entity-core-go` `7e0fb7c`, `entity-core-rust` `4107c32`, `entity-core-py` `808d9e6` — and go's
reference oracle reports **27/27, 0F for each of them** on `registry_issuer`. Rust's withheld
`pending_hash` was the correct posture and is discharged; Python's remove-on-approve was corrected to a
retained `approved` head. **The four items ruled above (`denied` in the enumeration, the un-typed
decision input, the superseded-head `404`, supersession observability) were each found by a build, not by
review, and three of the four were reached independently by all three seats before anything was ruled** —
which is the signal that the text was under-determined rather than misread, and the reason they are
ratified as written rather than re-litigated.

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
- Service advertisement (§3b) — `system/registry/service-advertisement` entity: signature-verify against the pinned deployment identity, fail-closed on tamper/expiry (§3b), the OPTIONAL `services` field on `ResolutionResult` with **absent as a valid floor** (§3b.1), and the §3b.2 per-service-type selection rules including the §3b.3 byte-pinned rendezvous-hash for `signaling`.
  - **Vectors.** Publish a signed advertisement → `resolve` returns it in `services`; a tampered and an expired advertisement each fail closed with no silent downgrade; a Tier-0 zone carrying no advertisement resolves normally with `services` absent. **The selection vector MUST use a two-member pool** — a one-member pool makes every construction agree and proves nothing (§3b.3).

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
- `docs/proposals/implemented/extensions/PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` — landed-into source proposal (service advertisement = §3b of this spec; §3b.4 folded SHOULD where the proposal wrote MUST, per the note there)
- `specs/extensions/EXTENSION-SIGNALING.md` — §4 the rendezvous carrier and §9.3 the STUN reflector, the two core-path services §3b advertises; §3.1 supplies the 33-byte key `k` that §3b.3 hashes
- `guides/GUIDE-REFERENCE-DEPLOYMENT.md` — the operator-facing tier model these services are priced in (§3.2 reflector, §3.3 signaling, §3.4 data-relay, §5 the worked budget)
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
