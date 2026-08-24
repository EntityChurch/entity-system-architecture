# EXTENSION-REGISTRY

**Version**: 1.16
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.40+); EXTENSION-ATTESTATION.md (v1.3+) — the supersedes-chain discipline that binding revocation and superseded-binding retention are defined against (§3, §6.5, §7)
**Related**: EXTENSION-RELAY.md (Mode S can host a registry peer's tree; Mode A gates cross-registry federation, deferred from v1 — §8.2); EXTENSION-CONTENT.md (binding entities live in the content tree); EXTENSION-DISCOVERY.md (the sibling mechanism — peer-finding, not name lookup); EXTENSION-NETWORK.md (bootstrap endpoints)
**Tier:** Operational — Tier 2b (network), per `SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.

> ## ⚠ COMPLETENESS — this extension is **v1, NOT finished**
> The substrate + the two concrete v1 backends are landed and implemented. Several pieces are **specified-and-deferred** or **not-yet-designed**. Do **not** read "Landed" as "complete."
>
> **✅ Landed + implemented (v1):** resolver substrate (§2–§5); local-name backend (§6); peer-issued resolve + curated registration (§6a.1–§6a.8).
>
> **🟡 Design folded, implementation deferred or partial:** **service advertisement** (§3b, folded v1.5) — the `system/registry/service-advertisement` entity and the §3b.1 `services` field are specified; §3b.2/§3b.3's rendezvous-hash **selection function over a caller-supplied pool** is a separable half and is the simpler build. **Peer-issued live registration** `open`/`allowlist`/`manual` (§6a.9); the **manual-approval path** (§6a.9.3, ruled v1.3, corrected v1.4) together with the four gaps v1.4 rules — the `denied` status enumeration, the un-typed decision input, the superseded-head code, and supersession observability. **Signed binding-manifest** (§6a.7) — format locked, implementation deferred.
>
> *Which peers have built which of these is deliberately not recorded here. A spec that carries build state goes stale silently and gets cited as authority while wrong; per-peer status lives in `ROADMAP-EXTENSIONS.md` and the cohort's own reports.*
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
  ttl:           <ms duration | null>,                  ; positive-result cache hint (§3 is the canonical declaration)
  neg_ttl:       <ms duration | null>,                       ; OPTIONAL negative-cache hint on not_found / chain_exhausted; SHOULD per backend
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
2a. **Checking `binding.name` against the name the binding was located under (MUST).** A signature proves *who issued* a binding, never *what it was issued for*. Any binding located through an index the receiver did not itself author — a by-name pointer, a served listing, a manifest entry — was located at a position **the party serving the bytes chose**, while the signature covers only the body. A receiver that skips this accepts a validly-signed binding for name *X* in answer to a query for name *Y*.

   **This step is here, at §3, rather than in a backend section, because it is a property of the body and not of any backend.** Every `kind` carrying an `issuer_signature` over a body containing `name` — `peer-issued` today, and `dns-txt` / `well-known-url` / `did-web` / `consensus-anchored` when their backends ship — inherits the identical substitution. Scoping the rule to §6a would make the next backend's author re-derive it, which is exactly the separability that the defect consists of.

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
  ttl: uint                     ; ms DURATION, per §3 (NOT since-epoch)
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
                                                              ; PINNED KEYS: max_ttl (§6a.9.1 resolver ceiling, ms), neg_ttl
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
      pattern:        <closed wildcard pattern — `*` is the ONLY metacharacter; see "pattern grammar" below>,
      backend_kinds:  [<kind>]                                ; which backends to consult for this format
    }
  ]
}
```

Stored at `system/registry/resolver-config` (peer-local; not synced).

**`name_format_dispatch` is a filter, not a routing table, and it expresses no precedence.** Two entries in any realistic configuration match the same name — a catch-all matches everything, and a domain-shaped pattern and a bare `authority` pattern overlap on every dotted authority. A name matching several entries is eligible at the **union** of their `backend_kinds`; evaluation does not stop at the first matching entry. **Precedence is `resolver_chain[].priority`** — the filtered backends are consulted in ascending priority order (§4.1 step 3) and the first validated hit wins (§4.1.1). What bounds a broad pattern is therefore not its position in the list but **what it is permitted to name** — §4.1 step 2's configuration MUST.

**A name matching no entry yields the empty set, and the chain reports `chain_exhausted` (§4.1 step 4, fail-closed) `[MUST, v1.14]`.** This is the same disposition §4.1a already gives the mirror case — a dispatch entry narrowing to a backend absent from the chain — reached from the other side. **It is not "no filtering":** an unmatched name resolving through the *whole* chain would consult every name-transmitting backend in it, which is a strictly larger disclosure than the one this section's MUST forbids, arriving by fallthrough. A name that resolves nowhere is a visible, recoverable misconfiguration; a name that resolves everywhere is an irreversible disclosure.

> **Earlier text here read *"a name matching no entry is treated as matching the catch-all"*, and that sentence could never execute.** The catch-all is `*`, which matches **every** name — so if a catch-all row is configured, no name fails to match one, and if none is configured, the sentence names a row with no referent. Two implementations read it as *"no filtering"*, which is the one meaning it cannot carry: the catch-all is the most **restrictive** row in the recommended list.

**`pattern` grammar.** The `name_format_dispatch[].pattern` field matches against the **user-facing name string** — not against a tree path — and it is therefore a **registry-local matcher**.

**This is the registry's name matcher, and it is the ONLY one `[MUST, v1.15]`.** Every field in this specification that globs a user-facing name uses the grammar below: `name_format_dispatch[].pattern` (§4) and the issuer policy's `name_constraints` (§6a.9.1). **There is one matcher per registry.** Two matchers over one domain diverge silently — the fields sit two subsections apart, both match the same flat name string in the same handler, and the one example the spec gives for each (`*.eth`, `*.lab`) is grammar-identical under every candidate reading, so nothing in the document discriminates them. *(Earlier text scoped this grammar "to this field." That sentence was written to fence the matcher off from `ENTITY-CORE-PROTOCOL` §5.4 — which it still does, below — and it fenced off `name_constraints` as collateral, leaving an admission gate with an undefined grammar.)* `*` matches any run of characters within the name, including none. Examples: `*@*.*` → DNS-style handles; `did:web:*` → did:web; `*.eth` → ENS; `*` → catch-all (typically local-name). Deployments needing richer matching layer it in the backend, not the dispatch config.

**The grammar is CLOSED, and every character that is not `*` is a LITERAL `[MUST, v1.13]`.** The matcher is a pure wildcard match over the whole name string, anchored at both ends:

```
dispatch_match(pattern, name):
  ; `*`  — matches any run of characters, including none.
  ; ANY other byte, including `?` `[` `]` `\` `.` `:` `@` `/`, matches only itself.
  ; Any NUMBER of `*` is permitted — `*@*.*` is three, and it is in the table below.
  ; `/` is NOT a separator here: a name is a flat string with no segment structure.
  ; The match spans the WHOLE name; there is no unanchored/substring form.
```

**Implementations MUST NOT delegate this to a path-glob or shell-glob library.** Go's `path.Match`, POSIX `fnmatch`, and their equivalents all give `?` and `[…]` character-class meaning this grammar does not grant, and most of them stop `*` at a `/`. **A matcher that merely *omits* those features and one that treats them as literals are indistinguishable until a name or a pattern carries one** — the same reason `**` had to be rejected rather than left unmentioned in `EXTENSION-REVISION` §2.4.

**No pattern is invalid, so there is no write-time rejection here** — every string is a well-formed pattern, because every non-`*` byte is a literal. That is a deliberate difference from `EXTENSION-REVISION`'s four closed forms, which need a `400` because that grammar *can* be violated. **State it rather than infer it:** a registry MUST NOT reject a dispatch pattern for containing `?`, `[`, or `\`.

**Conformance `REG-DISPATCH-GRAMMAR-1` (REQUIRED, cross-impl-observable).** Four rows: pattern `a?c` matches the literal name `a?c` and **not** `abc`; pattern `a[bc]d` matches `a[bc]d` and **not** `abd`; pattern `*@*.*` matches `alice@example.com`; pattern `x*z` matches `x/y/z` — **`*` crosses `/`**, which is the row that fails against every path-glob implementation.

**This is not `ENTITY-CORE-PROTOCOL` §5.4 and MUST NOT be read as it.** §5.4 governs *paths*, where `pattern/*` is a subtree prefix match; a name is a flat string with no segment structure and no peer-id head, so the path matcher's forms (`/*/` peer strip, trailing `/*` subtree) have nothing to bind to here. The two are separate matchers over separate domains, and neither confers a reading on the other. **No `**` token exists in either.**

### §4.1 Precedence order on resolution

When `meta_resolve(name)` is called:

1. **Pinned bindings** override everything. If `name` matches a pinned entry, return the synthesized result (§4.1.2) immediately.
1a. **Authority-part peer-id decode precedes glob dispatch (MUST).** For a name of the form `name@X`, attempt to decode `X` as a V7 §1.5 Base58 peer-id **before** applying step 2. If it decodes, `X` is a **verification pin**, not a targeting instruction: resolve `name` through the ordinary chain and require the result's `peer_id` to equal `X`, refusing fail-closed and advancing the chain on mismatch (§6a.4's disposition — a pin that resolves elsewhere is the §6a.1a substitution case caught one layer up). This ordering is normative because a broad `*@*` dispatch entry would otherwise capture `alice@z6Mk…` and send a private name to a remote registry to answer a question the consumer can answer locally.

2. **`name_format_dispatch` filter** — narrow the resolver-chain to backends whose `backend_kind` is eligible for the queried `name`, using the registry-local name matcher defined above (**not** `ENTITY-CORE-PROTOCOL` §5.4).

**Eligibility is a pure function of the name `[MUST, v1.14]`:**

```
eligible_kinds(config, name):
  rules := config.name_format_dispatch
  if rules is absent or empty:
      return ALL                                          ; the filter is disabled — see below
  matched := [ r for r in rules if dispatch_match(r.pattern, name) ]
  return union( r.backend_kinds for r in matched )        ; the EMPTY SET if nothing matched

; a resolver_chain entry is consulted IFF entry.backend_kind ∈ eligible_kinds(config, name)
```

**A kind reaches eligibility only by being named.** There is no per-backend default and no "match all" for a kind that appears in no rule: `matched` is a set union over the rules, so a kind named nowhere is eligible nowhere. Row order is irrelevant — the union is order-free, which is why this list carries no precedence (§4) and `resolver_chain[].priority` carries all of it.

**An absent or empty `name_format_dispatch` disables the filter entirely**, and that is the only place "no filtering" is correct. It is a real discontinuity — zero rules admit every kind, one non-matching rule admits none — and it is deliberate: it is the ordinary filter-absent versus filter-present-and-excluding distinction, and a peer whose chain holds only name-blind backends transmits nothing either way. What makes it safe is the MUST below, which reaches it.

> **This paragraph previously carried a second, per-backend sentence** — *"backends without a `name_format_dispatch` entry default to match all; backends with one are consulted ONLY when the pattern matches"* — **and it contradicted the union rule above.** It was a category error rather than a wording problem: rules name `backend_kinds`, not backends, so *"a backend without an entry"* has no referent. Three implementations reached three behaviours from one paragraph. The mechanism is now stated once, as a function.

**This is the primary privacy mechanism** — without it, the queried name leaks to broad-matching backends earlier in priority.

**Consequently: a distribution's shipped `system/registry/resolver-config` MUST NOT make a name-transmitting backend eligible for an unscoped name `[MUST, v1.14]`.** The name-transmitting kinds are `dns-txt`, `well-known-url`, `did-web` and `consensus-anchored` (the table below). **The rule binds the configuration as a whole, not one row**, and it has two doors:

- naming such a kind in **any** rule whose pattern matches unscoped names — the catch-all `*` is the usual one, and an unscoped name is the path every bare name takes; and
- shipping an **absent or empty** `name_format_dispatch` while such a kind sits in the `resolver_chain` — the filter is disabled, every kind is eligible, and there is no catch-all row to inspect.

A third door — leaving a name-transmitting kind out of every rule so it "defaults to match all" — is closed **by construction** by the union rule above, and needs no clause.

An unscoped name discloses **every bare name a user types** — including a private handle or a typo — silently, on the happy path, in a configuration the user did not choose, and irreversibly. An operator MAY override this on their own peer; a distribution MUST NOT ship it.

> **Stated at the width of the invariant, not of the instance.** An earlier form bound only *the catch-all row*, which made it evadable by not writing that row.

**The banned property is name transmission, not remoteness**, and the two are not the same thing:

| Backend kind | Catch-all | Why |
|---|---|---|
| `local-name`, `self-certifying`, `out-of-band` | **MAY** | No network consultation at all. *(A **pinned** binding never reaches this table — §4.1 step 1 returns it before dispatch runs. `out-of-band` is the **kind** a pin's synthesized binding carries (§4.1.2), and per §6a.4 it matches only when explicitly configured as its own chain entry — so it is dispatchable where `pinned` is not.)* |
| `peer-issued` **resolved per §6a.4 through the signed root** | **MAY** | Every request is content-addressed. The queried name is matched inside a node already fetched and never appears in a request. |
| `dns-txt`, `well-known-url`, `did-web`, `consensus-anchored` | **MUST NOT** | Consultation *is* disclosure — the name goes to a third party as a query, a path segment, or a document name. |

**§6a.4 is what makes the `peer-issued` row safe, and it is already mandatory.** A resolver MUST verify signature, name-association and revocation *inside the signed tree*; §6a.3a states that the host-served listing **MUST NOT** be presented as authoritative. A conformant `peer-issued` resolution therefore has no host-trusted-pointer path to fall back to — the mechanism is fixed by the kind, in the direction that makes it safe. **A non-conformant resolver that trusts a host-served `by-name` pointer does put the name in a URL, and is excluded here for the same reason it is excluded there.**

**What this does not claim is zero disclosure, and the difference is worth stating precisely.** The walk descends by the name's own hash, so an origin observes which interior nodes were fetched — a **hash-prefix oracle** over the queried name, shared by every name in that bucket. On a hit it additionally observes a fetch of the binding blob, whose hash identifies a name the registry has **published**, and therefore already public. Neither discloses the queried string, and neither reaches a name the registry does not carry. That is categorically weaker than handing a private name to a third-party resolver, which is the harm this rule exists to prevent.

A name-transmitting backend stays fully reachable through an explicit scoped form (`alice@example.org`), which is the user stating which authority they are willing to tell. An operator MAY override this on their own peer; a distribution MUST NOT ship it as the default.
3. **Filtered resolver-chain backends in priority order** — try each, returning the first validated result.
4. If all backends miss / fail validation: return `chain_exhausted` (fail-closed; no silent fallback).

The local-name backend (§6) participates as a resolver-chain entry like any other backend. Local-name-first ordering is a deployment convention realized by setting the local-name entry's `priority` to `0` (or another low value); the substrate stays uniform.

### §4.1a The recommended default dispatch list

A distribution **SHOULD** ship the list below; the catch-all rule inside it is a **MUST**
(§4.1 step 2). It realizes the four name shapes of `guides/GUIDE-RESOLUTION.md` §6.1 as eligibility.
The `#` column numbers the rows for reference; it is **not** an evaluation order (§4).

| # | `pattern` | `backend_kinds` | Shape |
|---|---|---|---|
| 1 | `did:web:*` | `["did-web"]` | scheme-typed |
| 2 | `did:key:*` | `["self-certifying"]` | scheme-typed, self-certifying |
| 3 | `*.eth` | `["consensus-anchored"]` | scheme-typed by suffix |
| 4 | `*@*.*` | `["dns-txt", "well-known-url"]` | domain-scoped — **dotted** authority |
| 5 | `*@*` | `["peer-issued"]` | registry-scoped — **undotted** handle |
| 6 | `*` | `["local-name", "self-certifying", "out-of-band", "peer-issued"]` | catch-all — **no name-transmitting backend (MUST)** |

**Two tokens were corrected here `[v1.12]`, and both were dead config in every conformant peer.** §2.4.1 is the canonical `backend_kind` vocabulary and neither appeared in it, so §4.2's forward-compat rule — *an unknown `backend_kind` MUST cause the entry to be skipped with a warning* — **discarded rows this spec recommends shipping.**

- **`did-key` → `self-certifying`.** Row 2's own Shape column already said *self-certifying*: a `did:key:` name carries its key, so the self-certifying backend decodes it. No new vocabulary is needed and none is added.
- **`pinned` → removed** (replaced by `self-certifying`, which the catch-all table below already admits). **`pinned` is not a backend kind and cannot be reached from dispatch:** §4.1 step 1 returns a pinned match **immediately**, before the step-2 filter runs, and §4.1.2 uses `pinned` as a **`backend_id`** on the synthesized result — a result label, not a dispatch target. Naming it in `backend_kinds` was a category error that no configuration could act on.

**Rules 4 and 5 overlap, and `priority` resolves it — not the row order.** The dispatch matcher cannot
express "undotted" — it has one metacharacter — so `*@*` necessarily also matches a dotted authority: `alice@example.org` is
eligible at `dns-txt`, `well-known-url` **and** `peer-issued`. The dispatch list does not choose
between them; `resolver_chain[].priority` orders them and §4.1.1 returns the first validated hit.
A deployment that does not want a dotted name reaching its peer-issued registry expresses that by
**priority**, or by narrowing rule 5 to its own registry handle (`*@entity-church`) — never by
relying on the order of the rows above.

**Entries naming a backend that is not in the resolver-chain are inert, not harmful.** Rules 1–4
name backends that are not yet built; a dispatch entry narrowing to an absent backend yields the
empty set and the chain reports `chain_exhausted` (§4.1 step 4, fail-closed). **Reserving the
routing now is deliberate:** without these entries a web-native name shape falls through to the
catch-all, which is the disclosure §4.1 step 2 forbids, reached by a different door.

**Deployments MAY override.** This is the interoperable default, not a wire format: two peers
shipping it interoperate; one that does not simply routes its own way. The catch-all MUST is the
exception, and it binds what a distribution ships rather than what an operator may configure.

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

**An unknown kind is not a name-transmitting kind, and MUST NOT be treated as one `[MUST, v1.14]`.** The §4.1 step-2 refusal is scoped to the four kinds this spec **declares** name-transmitting (`dns-txt`, `well-known-url`, `did-web`, `consensus-anchored`); an undeclared kind falls to the rule above and is skipped, so it consults nothing and discloses nothing. Refusing a whole configuration because a broad pattern names a kind this build does not recognize rejects a deployment authored against a **newer** vocabulary, which is the case this section exists to permit.

**The forward risk this raises is real and is discharged by *when* the check runs, not by refusing early.** A kind that is unknown today may be declared name-transmitting tomorrow, and a config validated once would then carry a violation nobody re-examined. §11.1 requires the check **at load** — so the peer that upgrades its vocabulary re-runs it against the same stored config on its next load, and the entry that was inert becomes a refusal at the moment it stops being inert. A write-time-only check is the variant that fails here; a load-time check does not need to be conservative about kinds it cannot classify.

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

#### §6a.1a The fourth actor — the party that serves the bytes

**Because the backend's reads are transport-agnostic, the party serving the bytes is not necessarily the registry.** For a static registry — the coral-reef deployment this section is written for — the registry *signs* and an origin (a bucket, a CDN, a mirror) *serves*. When this extension's threat model was written those two were one party, and every actor in it is a **requester**, the **registry itself**, or the **consumer's own fallback chain**. **There is no actor for the byte-server**, and no review of the existing rows finds a missing row.

The static deployment splits them, and the split hands the origin two powers that no signature revokes:

- **It may substitute which signed artifact answers a read.** Every artifact it serves is genuinely signed by the registry; it chooses *which one* answers *which name*. This is the §6a.4 association check's entire reason for existing.
- **It may withhold an artifact indefinitely** — most consequentially a revocation. It cannot be caught doing so: a withheld revocation and a revocation that was never issued are byte-identical at the consumer. This is why §6a.3 requires a finite `ttl`; **the TTL is the only bound on a withheld revocation**, so a null one makes a binding permanently unrevokable.

**What this actor cannot do, and the limits are what make the two rules sufficient rather than merely helpful:** it cannot forge a signature, alter a body, or move the consumer's clock. So every remaining defense is one the consumer computes locally over bytes it has verified — which is exactly the shape of the two `require`s added at §6a.4.

Consumers pinning a registry SHOULD understand that pinning the registry's **key** does not pin its **host**, and that the honest revocation bound against a hostile origin is *the binding's TTL*, not *the revocation's publication*.

### §6a.2 Backend identity

The peer-issued backend identifies as `backend_kind: "peer-issued"` in resolver-config. Its `backend_id` is the **registry peer-id**, which doubles as the **pinned trust root**: a resolver-chain entry for peer-issued names carries that peer-id and accepts a binding only if signed by it.

### §6a.3 The peer-issued binding + by-name index

The binding body is a standard §3 `system/registry/binding` with `kind: "peer-issued"`, `ttl` set (issued bindings expire, unlike sticky local-names), and — unlike §6.3 — it **carries a `system/signature`** (the registry is a remote authority; the signature is the whole point). Signature is reachable at the invariant-pointer `system/signature/{hex(binding_hash)}`, target-matching the binding's `content_hash` (V7 §5.2).

Two-layer storage, the direct analog of §6.3:
- **Binding body** at `system/registry/binding/{binding_hash}` (§3 universal rule).
- **By-name pointer** at `system/registry/binding/by-name/{nfc(name)}` → the bare `system/hash` of the current binding body. This is the live name→hash index — same pattern as local-name's `local-name/{name}` pointer, different prefix. Served over http-poll like any tree node (with the `tree_leaf_suffix` disambiguator, SUBSTITUTE §2.2).

**Name-path safety (normative):** identical to §6.3 — no `/`, no C0/DEL control chars, NFC at issue time. Domain-shaped names (`billslab.com`) are fine (dots allowed).

**Units — `ttl` is a duration, `issued_at` is an instant (pointer, not a redefinition).** Both are declared canonically at **§3**: `issued_at: <ms-since-epoch>`, `ttl: <ms duration | null>`. Stated here because a reader working from §6a never reaches §3 and has twice implemented `ttl` as an absolute timestamp. §3 remains the only home; this is a cross-reference.

**A `kind: "peer-issued"` binding MUST carry a non-null `ttl` `[MUST]`.** §6a.4's expiry check is the **only** check on this path that a hostile byte-server cannot influence — it cannot forge a signature, alter a body, or move the consumer's clock, but it *can* withhold a revocation indefinitely. The stated bound on a withheld revocation is *"ttl + revocation"*; with `ttl: null` that bound is not weak, it is **absent**, and the binding is **permanently unrevokable**. The `(or ttl null)` allowance is scoped to the **local-trust kinds** (`local-name`, `pinned`), where stickiness is the user's own assertion and the user is the trust root. It has no place on an *issued* binding, whose trust root is the registry.

**A `kind: "peer-issued"` binding MUST carry a non-empty `transports` `[MUST]`.** §4.1.2's *"transport resolution per NETWORK §6.5 finds reachable endpoints"* is sound for a pin and for a live target, and **has no static counterpart**: NETWORK §6.5.4 makes profile discovery out-of-band in v1, so for a statically published peer there is nothing to find. A consumer resolves a peer-id and stops. The §4.1.2 **pin carve-out is unchanged**, and the contrast is the reason: a pin is the *user's* assertion, so reaching the peer is the user's problem; an issued binding is the *registry's* assertion and is worth nothing operationally without a way to reach the target.

#### §6a.3a Enumerating a registry — the walk is the authority, the listing is a menu

**The authenticated form of *"what names does this registry carry"* is a walk of the published trie from `published-root.root_hash`** (`EXTENSION-TREE.md` §3.1, §3.5). Trie leaves are `[key, value_hash]` — **the key is in the node** — so walking every node from the signed root yields the complete key set the signature commits to. No new mechanism is required and none is added.

A registry intending to be browsable **SHOULD** publish at `prefix: "system/registry/binding/by-name/"`, so the trie's key set **is** the name set. This is the same per-purpose tracked-prefix ruling `EXTENSION-TREE.md` §3.3a already gives for any other published extent.

**A served listing artifact (`{path}{tree_listing_suffix}`) is a transport-trusted convenience and MUST NOT be presented as the registry's authoritative contents.** Keep it — it is one fetch instead of O(N) and it is the right first-paint artifact — but resolve every listed name through the signed root before presenting it as a binding. **The asymmetry is the point:** a hostile origin can omit an entry from a served listing undetectably, and **cannot omit a node from the walk without the walk failing.** *Silently hidden* becomes *visibly incomplete*, which is the strongest completeness property a static origin admits of.

*(Note: `EXTENSION-TREE.md` §3.4.2's "consumers use LocationIndex rather than walking trie subtree structure" is about **efficient prefix scan** under hash-keyed routing, which scatters related keys. It is not a claim that the key set is unrecoverable, and for a registry the distinction dissolves, because the registry chooses its own published prefix.)*

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
  require binding.name == norm            ; ← THE ASSOCIATION CHECK. See below.
  require binding.ttl != null             ; §6a.3: peer-issued MUST carry a finite ttl
  require not revoked(registry, binding_hash)                           ; §6a.6
  require binding.issued_at + binding.ttl > now()

  ; 4. surface
  return ResolutionResult {
    status: "resolved", binding: binding_hash,
    peer_id: binding.target_peer_id, transports: binding.transports,
    trust_anchor: "peer_issued:" + registry, ttl: binding.ttl, backend_id: registry }
```

**The association check (`binding.name == norm`) is normative, and it closes a live substitution `[MUST]`.** A signature proves **who issued a binding**, never **what it was issued for**. The by-name pointer at `system/registry/binding/by-name/{norm}` is *transport-supplied* — for a static registry it is a file on an origin — so the party serving the bytes chooses which signed binding answers which name. Without this comparison, an origin repoints one pointer file and `foundation.example` resolves to the binding the registry legitimately issued for `protocol.example`: **valid signature, correct signer, unexpired, unrevoked, wrong name.** Every other `require` above passes.

The fix costs nothing because **the association was already committed**: the signature covers a body that contains `name`, so `sig(R, {name, target_peer_id, …})` *is* the registry's assertion of the pairing. The defect is not a missing commitment, it is a **discarded** one — the resolver decodes the body (it reads `issued_at`, `ttl`, `target_peer_id` from it) and never compares the one field that binds the result to the question asked. Zero extra fetches, zero new artifacts, no publishing change.

**Framing to carry, because the wrong one was in circulation:** this is *not* "per-binding signatures give authenticity but not association." They give both. It is "the verifier throws the association away."

**Fail-closed (normative):** any verify / association / revocation / expiry failure returns `None`/dead-end at this rung; the meta-resolver (§4.1) advances the chain. It **MUST NOT** silently downgrade to an `out_of_band` pin — a pin matches only if explicitly configured as its own chain entry.

**Local diagnostics SHOULD distinguish the failures; the chain value MUST NOT.** The rule above collapses signature failure, name mismatch, expiry, revocation and unsupported-kind into one undifferentiated dead end, which surfaces to an operator as *"the registry is broken"* — and a unit error, a clock skew and an active substitution attempt are then indistinguishable during exactly the incident where telling them apart matters. A resolver **SHOULD** surface which `require` failed **to its own operator**, and **MUST NOT** let that distinction change what the chain sees or what crosses a peer boundary. There is **no confidentiality argument against this**: V7 §5.5a's `Denied`-vs-`NotFound` discipline governs what a peer tells a **remote requester**, and this is a local resolver reporting to the operator running it.

**Levels (P2/P3):** the backend returns `not_found` + a first-class `neg_ttl` slot on the negative result (backend-scoped); the meta-resolver collapses a whole-chain miss to `chain_exhausted` (§4.1) carrying the aggregated `neg_ttl`. `neg_ttl` is a defined optional top-level field, not an opaque hint-bag entry.

**Offline / precedes path:** when the binding is pre-cached as a precede (§7), steps 1–2 read the local store instead of the wire; step 3 verify is identical. Precedes are just a warm cache.

### §6a.5 Trust-anchor floor (v1)

The v1 trust anchor is an **Ed25519 identity-multihash** registry peer-id: the pubkey **is** the peer-id digest (V7 §1.5 canonical form), so `pinned_key_of(registry)` is derived from `backend_id` and config carries only the peer-id. For a **non-self-describing** peer-id form (SHA-256-form / Ed448), the resolver-chain entry MUST carry the pubkey explicitly (or a locally-resolvable `system/peer`); this config-carried-key path is **deferred** (no v1 demo needs it). Consistent with the core §9.1 floor.

### §6a.6 Revocation (by-target index, normative)

`revoked(registry, binding_hash)` is an O(1) index lookup, **not** a scan: `system/registry/revocation/by-target/{hex(binding_hash)}` → the revocation entity (presence = revoked, if it verifies against `registry` per §3.1). This is the revocation analog of the §6a.3 by-name index. A live registry MAY layer subscription-driven invalidation on top (§3.1).

**The index is a performance structure and carries no integrity (MUST NOT be read as evidence).** A resolver **MUST NOT** treat a missing `by-target/{hex(binding_hash)}` key as proof that the binding is not revoked. The key is served by the party §6a.1a names as the fourth actor, and an absent key and a withheld key are byte-identical at the consumer — so **presence proves revocation, absence proves nothing.** The bound on a withheld revocation remains the binding's `ttl` (§6a.3), exactly as §6a.1a states, and moving from a scan to a keyed lookup does not change that bound.

**What the keyed form does change is the cost of a *targeted* withholding, and that is worth stating plainly.** Under a prefix scan, suppressing one revocation means manipulating a listing; under a keyed lookup it means answering `404` to one URL. A resolver that wants better than the TTL bound does **not** get it from this index — it walks the published trie from the signed root over the revocation prefix (§6a.3a), where withholding a node makes the walk **fail visibly** instead of returning a short answer. *(Both shapes ask the host the same question. The index makes the question cheap, not trustworthy.)*

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
  0. check name-path safety (§6a: no '/', no C0/DEL, NFC)  ; else 400 bind_invalid_name
  1. verify request signature by target_peer_id          ; layer-1 (always)
     on failure: 401 signature_invalid                   ; layer-1 domain — see the status table below
  2. apply issuer-policy admission (§6a.9.1)              ; layer-2 → approve | reject | queue
  3. on approve: registry-issue-binding(...) (§6a.8)      ; signs with K_registry, publishes, sets by-name pointer
     return 200 register-result { status: "bound", binding_hash }
  4. on reject:  error (name_taken | not_entitled | policy_rejected)   ; layer-2 reject domain ONLY
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
> **`system/protocol/error` MUST NOT carry the 202.** An error entity denotes a *failed* operation; `202` denotes *accepted-pending*. Emitting one on the other makes the status line and the result type disagree by construction, and a client branching on result type reaches the opposite conclusion from one branching on status. *(Adopted from a design argument that stands on its own merits and not on how many implementations held it — see `GUIDE-CONFORMANCE` §4: the spec arbitrates, the cohort does not vote.)*
>
> **A generic status type is rejected on structure, not on taste.** The 200 must carry `binding_hash` and the 202 must carry a poll handle, so a carrier with no room for either forces the payload somewhere else and re-opens the divergence one field down. **`register-request` also MUST NOT borrow another operation's result type** because the payload happens to match — that coupling breaks silently the first time either operation's result grows a field.
>
> **The value's spelling does not change.** `pending_review` stays snake_case: `STYLE-NAMING-CONVENTIONS` puts **error and status codes** in snake regardless of which field carries them. Only the carrier is being ruled.

**`pending_hash` `[MUST]` `[RULED 2026-08-12]` — it names the stored pending request entity, not the request the client sent.** It is the `content_hash` of the `system/registry/pending-binding` entity the registry stored at step 5 (§6a.9.3), resolvable by the ordinary `tree:get` / `content:get` machinery every other registry read uses (§6a.3). **A handle the client can already compute is not a handle** — echoing the request hash tells the requester nothing it did not have before sending, and nothing is fetchable at it. The divergence here was real and undecidable from the text: one implementation named the stored entity, another named the request.

**Owed, named rather than invented `[2026-08-12]` — ✅ DISCHARGED `[2026-08-13]`, see §6a.9.3.** The manual-approval path itself — the `system/registry/pending-binding` schema, the by-request pointer a requester polls when it no longer holds the 202 response, and the operator's approve/deny operation — was **not specified anywhere in this document**, which is why step 5 could say "queue" and stop. That ruling pinned the cross-peer-observable surface (result type, status value, what `pending_hash` refers to) and deliberately stopped there. **Stopping there had a cost that is worth recording: it left `pending_hash` a `MUST` naming an entity with no schema**, so one implementation withheld the value on principle while the reference oracle failed peers for withholding it — the spec manufactured a conformance failure out of its own reserved section. **A `MUST` may not name a referent the corpus does not define**; if the referent must wait, the `MUST` waits with it. §6a.9.3 now defines it.

**Statuses `[MUST]` `[RATIFIED 2026-08-11]` `[RATIONALE CORRECTED 2026-08-12]`.** §6a.9 pinned the reject *codes* and left the *statuses* open, the way §6a.9.2 later pinned `400` / `501`. It is now text:

| Outcome | Status | Value | Carried by |
|---|---|---|---|
| layer-1 proof absent / wrong signer | **401** | `signature_invalid` | `system/protocol/error` `.code` |
| manual mode — queued for review | **202** | `pending_review` | `register-result` `.status` — **not an error code** |
| name already bound | **409** | `name_taken` | `system/protocol/error` `.code` |
| layer-2 admission refused | **403** | `not_entitled` \| `policy_rejected` | `system/protocol/error` `.code` |

> **The fourth column exists because its absence caused the divergence `[added 2026-08-12]`.** This table shipped with the third column headed **`Code`**, which is a category error on the `202` row: `pending_review` is a **status field value**, and §6a.9's own pseudocode said so (`on queue: status "pending_review"`) while the table said otherwise. **An implementation that trusted the table emitted an error entity on a 2xx** — a faithful reading, and the same failure shape as §5.4/§8.3, where the normative artifact and the prose disagreed and the artifact won. **A status table that names a value without naming what carries it is under-specified by exactly one column**, and the missing column is the one a wire implementer needs.

401 (not 403) for layer 1 follows V7 §5.2a's discriminator: an unverifiable signer is an **authentication** failure, and §4.2/§4.4's F32 ruling already put that class at 401. The `409` / `403` rows are **derived** from V7 §3.3's class rules rather than observed in practice, so an implementer meeting a divergence on those two rows should report it rather than assume the defect is local.

**Error *strings* are free; codes are the contract.** The `code` values above are normative and are what a peer branches on. Human-readable messages accompanying them are impl-local and MAY differ — **a conformance check MUST NOT assert on message text.** Stated explicitly so that "no check reads them" remains a design choice rather than hardening into a latent expectation.

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
  name_constraints: <name pattern per §4 | null>,  ; §4's grammar — `*` only; e.g. only issue "*.lab"
  default_ttl:      <ms | null>,                ; MUST NOT exceed max_ttl
  max_ttl:          <ms duration>               ; REQUIRED for any mode reaching *approve* (§6a.9);
                                                ;   requests above it are CLAMPED, not refused
}
```

Two separable proof layers:
- **Layer 1 — peer-id control (always, §6a.9):** the request is self-signed by `target_peer_id`.
- **Layer 2 — name entitlement (policy):** whether *this requester* may have *this name*.
  - **`open` — first-come-first-serve.** Any layer-1-valid request for a free name is approved and signed. This is the trivial default: no entitlement check beyond "is the name taken." It is barely more than the curated tool with a handler wrapper.
  - **`allowlist`** — only `target_peer_id`s in `allowlist` may register (optionally bounded by `name_constraints`).
  - **`manual`** — requests queue as `pending_review`; the operator approves out-of-band.
  - **`domain-control`** — for domain-shaped names, the registry requires proof of DNS-domain control before signing. **The challenge format is DEFERRED** — it MUST share one mechanism with the web-native `dns-txt` / `well_known_url` backends rather than inventing a second domain-proof scheme, so it is settled jointly with those proposals, not here. A v1 registry uses `open` / `allowlist` / `manual`; `domain-control` lands with the web-native co-design.

**`name_constraints` uses §4's name matcher `[MUST, v1.15]`, and therefore no `name_constraints` value is invalid.** It globs the same user-facing name string §4 globs, in the same handler — so it is the same matcher, and `*` is its only metacharacter. Two consequences bind:

- **A registry MUST NOT reject a policy for a malformed `name_constraints`**, and `set-issuer-policy` MUST NOT return an error on its pattern's syntax. Every string is a well-formed pattern because every non-`*` byte is a literal (§4).
- **A registry MUST NOT fail a registration on pattern syntax.** Delegating this field to a path-glob or shell-glob library reintroduces a parse error the grammar cannot produce: `name_constraints: "a[b"` under such a library makes **every** register-request against that policy fail with an internal error — a policy the operator installed, silently un-registerable, with no diagnostic naming the pattern. Under §4's grammar that arm is unreachable, not handled.

**One matcher, two input domains `[MUST, v1.16]`.** The grammar is one (above); the two sites that apply it do **not** receive the same inputs, and a vector written for one site does not transfer to the other. `name_format_dispatch` (§4) matches the **raw argument to `meta_resolve(name)`** — an arbitrary string, since dispatch runs before any backend is consulted and nothing has rejected or normalized it yet. `name_constraints` matches a name that has **already passed §6a's name-path safety** (*"identical to §6.3 — no `/`, no C0/DEL control chars, NFC at issue time"*), which a `register-request` fails with `400 bind_invalid_name` before the policy is read at all. **Consequently no name reaching `name_constraints` contains `/`**, and whether `*` crosses `/` is unobservable at this gate — it is a real property of the matcher, asserted once, by `REG-DISPATCH-GRAMMAR-1` at the site whose input domain admits it. A conformance row requiring this gate to admit a `/`-bearing name is unsatisfiable by construction: the only way to pass it is to remove the path-injection check that §6a makes normative.

**This is an admission gate, so the divergence it hid is the expensive kind.** `name_constraints` decides whether a binding is *issued at all* — `403 not_entitled` versus a signed, published binding — so two registries running the **same operator policy** would admit different names. The field carried `<glob>` with the single example `*.lab`, which is grammar-identical under every candidate reading and therefore discriminates nothing; two implementations read it two ways and neither could have found the other by reading the spec.

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
- **A policy that could mint an invalid binding is refused at write `[MUST]`.** `set-issuer-policy` MUST reject with **`400`** a policy running live registration (any mode that can reach *approve*) whose `default_ttl` is `null`. §6a.3 requires a peer-issued binding to carry a finite `ttl`; a request may omit `requested_ttl`, so a policy with no `default_ttl` can resolve to a null `ttl` and mint a binding **no conformant resolver will honor** (§6a.4). Same reason and same subsection as the `domain-control` refusal above: *rather than storing a policy it cannot enforce.* **The gate is here because this is where the missing input lives** — `default_ttl` is the operator's field, set through the operator's operation, under `registry-manage-issuer-policy`. Refusing the *requester's* register-request instead would bill a well-formed request for the registry's own misconfiguration, and would teach requesters to send `requested_ttl` defensively — handing TTL selection to the party §6a.1a treats as untrusted.
- **A stored `domain-control` policy fails closed with `501` `[MUST]`.** The `400` above binds `set-issuer-policy`, which refuses to *store* the mode; it does not answer what a registry does when the entity is already there — seeded out-of-band, written directly to the tree, or predating the refusal. **The registry MUST answer live registration `501 unsupported_mode` and MUST NOT fall back to `open`, `manual`, or an unset-style `404`.** Falling back to `open` turns an operator's unenforceable curation into first-come-first-serve, which is the §6a.9 threat model exactly inverted; falling back to `404` reports "no policy" while a policy is stored. This is a cross-impl-observable answer with four plausible codes, so it is pinned rather than left to converge.
- **`get-issuer-policy` takes no params content**, so callers send the `ENTITY-CORE-PROTOCOL.md` §3.2 **empty-params shape** — a `primitive/any` entity whose `data` is canonical-CBOR `a0`. It is **not** a zero-value entity (rejected `400 invalid_params` at the envelope layer, before the handler) and **not** a `primitive/map` (a handler SHOULD reject a mismatched params *type* with `400 unexpected_params`). *Stated here because a unit test that calls the handler directly never crosses the envelope layer and cannot see either failure — core-go found both on first contact with a live peer.*

**Fail closed when the stored policy is already bad `[MUST]`.** §6a.9.2's refusal binds `set-issuer-policy`; it does not answer the policy that is **already there** — seeded out-of-band by a CLI flag, written directly to the tree, or predating the rule, all of which §6a.9.2's store-first resolution admits. When the resolved `ttl` for a register-request would be null (the request omitted `requested_ttl` and the stored policy has no `default_ttl`), the registry MUST refuse with **`403 policy_rejected`** and **MUST publish nothing** — no binding, no queue entry.

It **MUST NOT substitute an implementation-chosen default.** That is the same move §6a.9.2 already rejects one bullet up, where `get-issuer-policy` MUST NOT synthesize a default `open`: it converts an operator's omission into a silently-invented policy. On a security-relevant field it is worse — two registries would answer identically-stored policies with different binding lifetimes, a §5.10 cross-peer determinism split the operator never sees. A protocol-wide TTL floor is rejected for that reason plus one more: there is no defensible number, and choosing one makes every unconfigured registry look configured.

*(Structure mirrors `REG-ISSUER-DOMAINCTRL-STORED-1` beside `REG-REGISTER-DOMAINCTRL-1` — the write-time refusal cannot be the only thing standing between a bad stored policy and a bad outcome.)*

**`renew-request` resolves its `ttl` by a three-step cascade and never refuses for a missing one `[MUST, v1.9]`.** `renew-request` is a **second producer of peer-issued bindings**, and the null-`ttl` rules above swept only `register-request`. A renew omitting `ttl` against a policy with no `default_ttl` minted a null-`ttl` successor — a binding §6a.3 forbids and §6a.4 will not honor. The resolution order is:

| Step | Source | Note |
|---|---|---|
| 1 | the request's own `ttl` | as for `register-request` |
| 2 | the issuer policy's `default_ttl` | current operator intent outranks history — an operator who lowers `default_ttl` sees renewals pick it up |
| 3 | **the superseded binding's `ttl`** | non-null by §6a.3 on every conformant mint path |
| — | **all three yielded nothing** | **refuse `403 policy_rejected`, publish nothing** — see the fail-closed clause below |

**The cascade fails closed when the predecessor itself is invalid `[MUST, v1.10]`.** Step 3 is non-null *on every conformant mint path*, which is not the same as non-null. A predecessor carrying `ttl: null` can be **already there** — seeded out-of-band, written directly to the tree, or predating these rules — the identical "stored state is already bad" case that the `set-issuer-policy` refusal above does not answer and that D12's backstop exists for. **A renew whose three steps all yield null MUST refuse with `403 policy_rejected` and MUST publish nothing.** It MUST NOT mint the successor with a null `ttl`, and it MUST NOT substitute a default.

> **This clause corrects v1.9, which asserted the cascade was total *because* §6a.3 guarantees step 3.** That inference reads an invariant as a fact about stored bytes. It is the same mistake §6a.9.2 already documents one paragraph up — *"the write-time refusal cannot be the only thing standing between a bad stored policy and a bad outcome"* — and the cascade was written without carrying that lesson across. **An unreachable branch that is asserted rather than enforced is how the shape it forbids gets minted**: an implementation that guards the dereference to avoid a panic, and falls through, produces exactly the null-`ttl` binding the whole rule set exists to prevent. The refusal above is unreachable on any conformant path and is required precisely for that reason.

**This is not the implementation-chosen default the paragraph above forbids, and the distinction is the whole ruling.** A synthesized default is a number the implementation invents, so two registries answer identically-stored policies differently. The superseded binding's `ttl` is **the registry's own prior signed act on this exact name** — one value, already published, byte-identical at every conformant peer. It is recovered, not chosen, so it creates no §5.10 determinism split.

**Refusing instead is the wrong side, by this subsection's own reasoning.** §6a.9.2 puts the register-time gate at `set-issuer-policy` *"because this is where the missing input lives,"* and refuses to bill the requester for the registry's misconfiguration. At renew **the input is not missing** — the registry holds a valid `ttl` it issued itself — so the argument that forces a refusal at register does not reach here. Refusing would revoke a name by inaction, for a policy defect the registrant cannot see or fix, on the one operation whose purpose is to keep the name alive.

**Vector `REG-RENEW-TTL-NULLPRED-1` `[v1.10]`** — write a peer-issued binding with `ttl: null` **directly to the tree** (the same two-stage shape as `REG-ISSUER-DOMAINCTRL-STORED-1`), then renew it with no `ttl` against a policy with no `default_ttl`: **`403 policy_rejected`, nothing published.** The control is `REG-RENEW-TTL-CASCADE-1` row (b), which must still return `200`. Without this row a peer that drops the final refusal passes every other renew vector.

**What the cascade does not do:** it does not extend a binding beyond what the registry already granted (step 3 re-grants the same duration to the same layer-1-authenticated `target_peer_id`), it does not weaken §6a.3 (the resolved `ttl` is non-null on every path), and it does not touch revocation (a revoked binding is not renewable regardless of `ttl`).

**`ttl` is bounded on both sides, and the binding side is not the one that matters `[MUST, v1.11]`.** Step 1 accepts the requester's own number and nothing capped it. Since §6a.3 makes `ttl` the **only** bound on a withheld revocation, an unbounded requester-chosen `ttl` reproduces the permanently-unrevokable binding §6a.3 exists to prevent, without ever setting the field to null.

**Issuer side — `max_ttl` is REQUIRED on any policy that can reach *approve*.** `system/registry/issuer-policy` gains `max_ttl: <ms duration>`. `set-issuer-policy` MUST reject with **`400`** a live-registration policy whose `max_ttl` is absent or null — **the same trigger, the same site, and the same reason as the `default_ttl` rule above**: it is the operator's field, set through the operator's operation, and this is where the missing input lives. `default_ttl` MUST NOT exceed `max_ttl`; a policy violating that is refused `400`.

**A request above the ceiling is CLAMPED, not refused `[MUST]`.** `register-request` and `renew-request` resolve `ttl` through their cascades, then apply `effective = min(resolved, policy.max_ttl)`. **Refusing would bill a well-formed request for a policy the requester cannot read** — §6a.9.2's own stated reason for not gating the requester — and it teaches requesters to probe for the ceiling. Clamping is silent to the requester by design: the issued binding carries the clamped value, which is signed, published, and readable.

**Resolver side — a resolver MAY impose its own ceiling, and this is the half that protects the consumer `[MUST when present]`.** A resolver that declares a local maximum MUST treat a binding's effective lifetime as **`min(binding.ttl, local_max)`**, computed at resolution and never written back into the binding (the binding's content hash is unchanged; this is a *use* bound, not a re-issue).

**The ceiling is declared at `resolver_chain[].hints.max_ttl` (ms) `[MUST, v1.16]`.** Until v1.16 this rule named a value and no place to put it, and every implementation invented one — a `[MUST when present]` with no declared config site is a rule two conformant peers cannot both implement, and this one had four seats across three keys. `hints` is the slot §4 already declares for backend-scoped configuration (it carries `neg_ttl` likewise), so pinning the key here costs no change to `system/registry/resolver-config`'s type hash. **Per chain entry, not per peer**, and that granularity is the point: the ceiling bounds how long *this* resolver will honor *this* backend's answers, and a registry you operate does not deserve the same number as one you barely trust.

**It MUST be durable configuration read at resolution `[MUST, v1.16]`** — not a process-lifetime setting fixed at construction. A ceiling read at start-up applies on a cold boot and silently does not on a warm one, and **a security control present on one boot path and absent on the other is worse than absent: it tests green on whichever path the test happens to take.** An operator editing `resolver-config` MUST be able to set it; out-of-band arming is a **seed for that entity**, never a parallel source consulted at resolution (the store-first rule of §6a.9.2, same reasoning).

**`max_ttl: 0` MUST be treated as undeclared `[MUST, v1.16]`.** Honored literally it expires every binding instantly and the operator sees *"no binding for this name"* — indistinguishable from a bad signature, a revocation, or an offline registry, which is the worst available diagnostic for what is almost certainly a typo or an unset field serialized as zero. *(All four implementing seats reached this independently before it was written down.)*

**A binding carrying no `ttl` takes the ceiling as its lifetime `[MUST, v1.16]`.** `min(binding.ttl, local_max)` has no arm for a null `ttl`, and the sticky kinds (`local-name`, `pinned`) carry none. The resolver's ceiling is a bound on **how long a value may be honored**, so an absent lifetime becomes `local_max` rather than staying unbounded — applying the control only where a bound already exists would leave exactly the unbounded case uncovered, which inverts it. *(`peer-issued` never reaches this arm: §6a.4 requires a non-null `ttl` before a result is surfaced at all.)*

> **Why the resolver's ceiling is the load-bearing one.** §6a.3's argument is entirely about the **consumer**: a hostile byte-server withholds a revocation, and `ttl` bounds the exposure. **A ceiling enforced by the registry does not protect a consumer from that registry** — a hostile or compromised issuer simply sets `max_ttl` high. Only the party bearing the risk can bound it. This is the split DNS settled decades ago: the authority sets the record's TTL, and the **resolver** caps what it will honor (`max-cache-ttl`), because the resolver is the one holding stale data. The issuer-side `max_ttl` is operator hygiene — it stops a careless registrant asking for a decade — while the resolver-side clamp is the actual security property.
>
> **The shape is §4.10's, applied to freshness: mandate that the bound exists and is declared and enforced; leave the value to the deployment.** No number is written here for the same reason §4.10 writes none — there is no defensible constant, and choosing one makes every unconfigured deployment look configured. This is also the mainstream answer across the surveyed field: DNS caps at the resolver, TUF sets expiry per role, and the X.509 and ACME ecosystems put the ceiling in policy rather than in the protocol. **None of them put an unbounded lifetime in the hands of the requesting party, and none of them writes the maximum into the wire format.**

**Conformance:** **`REG-TTL-CEILING-1`** — `set-issuer-policy` with a live mode and absent `max_ttl` → `400`; with `default_ttl > max_ttl` → `400`; **control:** a policy with both, `default_ttl <= max_ttl`, is accepted. **`REG-TTL-CLAMP-1`** — a `register-request` and a `renew-request` each carrying `ttl` above `max_ttl` → both accepted `200`, and both issued bindings carry **exactly `max_ttl`**. The clamp is asserted on the *binding's* value, not on the response code, because a peer that refuses instead of clamping also returns a non-`200` and would otherwise be indistinguishable.

**`REG-TTL-RESOLVER-CEILING-1` `[v1.16]` — the resolver-side vector this spec has never had for the half it calls load-bearing.** Both existing rows above test the **issuer** side; nothing tested the clamp that actually protects a consumer, which is how four seats put it in three places without any instrument noticing. Against a chain entry carrying `hints.max_ttl`, four rows: **(a)** a binding whose `ttl` exceeds `max_ttl` resolves with effective lifetime **exactly `max_ttl`**, and `result.binding`'s **content hash is unchanged** — assert the hash, not only the number, because a resolver that rewrites the binding to carry the clamped value moves its address and invalidates every signature over it; **(b)** a binding whose `ttl` is below `max_ttl` is returned untouched; **(c)** `hints.max_ttl: 0` behaves **identically to an absent `hints`** — the binding's own `ttl` survives (the undeclared rule above); **(d)** a sticky binding (`local-name` or `pinned`, no `ttl`) resolves with effective lifetime `max_ttl`. **Row (d) is the one an implementation passes by accident and fails on inspection**, since `min` over a null has no natural answer.

**Conformance:** **`REG-RENEW-TTL-CASCADE-1`** — three rows against a curated registry whose stored policy has no `default_ttl`: (a) renew **with** explicit `ttl` → accepted, successor carries it; (b) renew **omitting** `ttl` → accepted, successor carries **the superseded binding's** `ttl`, and the successor resolves under §6a.4; (c) the same renew against a policy that **does** carry `default_ttl` → successor carries the **policy's** value, not the predecessor's. Row (b) is the one that fails against both a null-minting peer and a refusing peer; row (c) is the one that fails against a peer that implemented inherit-first.

**Replay defense (normative discriminator).** A signed request carries `nonce` + `issued_at` (the registry tracks seen `nonce`s per requester within an `issued_at` window; a replayed request is rejected) **iff replay has a non-idempotent state effect.** This holds for `register-request` (replay can roll a name back to a superseded binding) and `renew-request` (replay can extend a binding's life past intended lapse). It does **not** hold for `revoke-request`, which is monotonic on a content-addressed target (replay cannot un-revoke and cannot reach a later re-issued binding) — so revoke omits `nonce` / `issued_at`. The discriminator, not the op name, decides: future ops are replay-defended exactly when their replay mutates state.

**Conformance:** `REG-REGISTER-PROOF-1` (signature not by `target_peer_id` → rejected), `REG-REGISTER-POLICY-1` (allowlist reject → `not_entitled`; allow-listed → issued + resolvable), `REG-REGISTER-REPLAY-1` (seen nonce → rejected). `REG-REGISTER-DOMAINCTRL-1` gates the deferred `domain-control` mode; **`REG-ISSUER-DOMAINCTRL-STORED-1`** gates the fail-closed `501` above (write the policy entity directly, then attempt live registration — the `set-issuer-policy` refusal cannot be the only thing standing between a stored unenforceable mode and an open registry).

**Implementation status:** the **design is pinned here**; the `open` / `allowlist` / `manual` modes are buildable now (no external dependency); `domain-control` waits on the web-native domain-proof co-design. A registry shipping curated-only (§6a.8) is conformant — it simply does not run the handler.

##### §6a.9.3 The manual-approval path — `pending-binding`, the by-request pointer, approve / deny

**Reserved earlier, filled here.** The earlier ruling pinned *what `pending_hash` refers to*
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
unbuilt. **They are built in all three implementations** (pins in the outcome note at the end of this
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
- **A decision on a *superseded* head returns `404 not_found` `[MUST]`.**
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

**The decision operations take an un-typed input, and handlers MUST decode by shape `[MUST]`.** Every other write operation on this handler names a `system/registry/*` params
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

**Supersession must be observable, and the schema alone does not make it so `[MUST]`.** A `register-request` retry carries a fresh `nonce`, but `pending-binding` does
**not** carry the nonce, so two retries of one intent inside a single millisecond encode to identical
bytes and content-address to **one** body — at which point the 202's "new `pending_hash`" is the old
`pending_hash` and a superseding write is indistinguishable from a no-op. **That collapse is correct and
intended** — one head, one hash, and adding the nonce to the body would defeat the dedup for no gain.
What follows from it is a conformance obligation, not a schema change: **`REG-PENDING-DECIDE-1`'s
supersession half MUST vary a field the schema actually carries** (`requested_ttl` or `transports`), and
an implementation MUST NOT rely on the returned `pending_hash` *changing* as its supersession signal.
The observable invariant is the one stated above — **exactly one head reachable through the pointer for
the pair** — which holds whether or not the two bodies collide. *(Recorded because the
obvious supersession test reaches for the nonce first, and that test would assert on an artifact rather
than on the invariant.)*

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

**The four items ruled above (`denied` in the enumeration, the un-typed
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

## §9 GC posture (per `GUIDE-GC.md`)

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

- **`REG-DISPATCH-CATCHALL-LOCAL-1` (§4.1 step 2, §4.1a).** **The discriminator is name transmission, not remoteness `[v1.14]`.** A resolver-config that makes a **name-transmitting** backend (`dns-txt`, `well-known-url`, `did-web`, `consensus-anchored`) eligible for an unscoped name MUST be refused or normalized at load — whether by naming it in a rule that matches unscoped names or by carrying no `name_format_dispatch` at all; and resolving a bare (unscoped) name MUST produce no request carrying that name to any third party. **The observable is the absence of a request**, so the check asserts on the third-party endpoint receiving nothing — a resolver that leaks still returns a correct answer, which is why no result-asserting vector reaches this.
  - **`peer-issued` resolved per §6a.4 through the signed root is explicitly admitted** and MUST NOT be asserted against: it is a read of a **remote** registry, and it is name-blind (§4.1 step 2's table). **Asserting on remoteness here contradicts §4.1a row 6**, which recommends `peer-issued` in the catch-all — the vector was written at v1.7 when the catch-all was `["local-name", "pinned"]` and the two properties coincided, and the re-key to name transmission did not reach it. A fixture asserting *"no read against any remote registry"* fails the default list this spec ships.
- **`REG-NAME-CONSTRAINTS-GRAMMAR-1` (§6a.9.1, §4).** `name_constraints` uses §4's matcher, and the spec's own `*.lab` example cannot discriminate that from a shell-glob. Against a live-mode issuer policy, four rows plus a control:
  1. `name_constraints: "a?c"` → a register-request for the literal name **`a?c`** is admitted; **`abc`** is refused `403 not_entitled`. *(Inverted under `path.Match` / `fnmatch`.)*
  2. `name_constraints: "a[bc]d"` → **`a[bc]d`** admitted, **`abd`** refused.
  3. `name_constraints: "a[b"` → `set-issuer-policy` **accepts** the policy, and a register for the literal name `a[b` is **admitted**. **No `5xx` on any path.** Asserted separately from row 2 because a shell-glob implementation fails this one by *erroring* rather than by answering wrongly, and an error is not a wrong answer a result-asserting check would catch.
  - **No `/`-crossing row appears here, and its absence is normative `[v1.16]`.** `*`-crosses-`/` is a real property of §4's matcher and is asserted by `REG-DISPATCH-GRAMMAR-1`, whose input is the raw `meta_resolve` argument. **This gate's input has already passed §6a name-path safety**, so a `/`-bearing name is refused `400 bind_invalid_name` before the policy is consulted (§6a.9.1, "one matcher, two input domains"). A row requiring this gate to admit `x/y/z` was carried at v1.15 and was **unsatisfiable by every conformant implementation**; it is withdrawn rather than re-scoped.
  - **Control:** `name_constraints: "*.lab"` → `alice.lab` admitted, `alice.dev` refused. This is the example the spec already carried; it passes under **both** readings and proves nothing alone, and it is included only so a failure of rows 1–4 cannot be misread as the constraint being ignored entirely.

- **Hostile-origin vectors (§6a.1a).** The four pre-existing peer-issued vectors all pass on an implementation open to both defects below, because each tests a forgery the origin never attempts. **A suite where every implementation passes every vector while all of them share one hole is not evidence of convergence; it is evidence the suite does not reach the surface.**
  - **`REG-PEERISSUED-NAME-SUBSTITUTION-1`** — a **validly signed, current, unrevoked** binding for name *X*, served at `by-name/{Y}`. The resolver MUST refuse and advance the chain (§6a.4 `binding.name == norm`). Contrast with `REG-PEERISSUED-VERIFY-FAIL-1`, which covers a binding signed by a **non-pinned key** — the origin forging its own binding, which every implementation already refuses. Substitution requires no forgery at all.
  - **`REG-PEERISSUED-NULL-TTL-1`** — a peer-issued binding with `ttl: null`; the resolver MUST refuse and advance (§6a.3, §6a.4).
  - **`REG-ISSUER-NULLTTL-POLICY-1`** — two-stage, matching `REG-ISSUER-DOMAINCTRL-STORED-1`: `set-issuer-policy` with a live mode and `default_ttl: null` MUST be refused `400`; then write that policy entity **directly** and attempt live registration with a request omitting `requested_ttl` — MUST refuse `403 policy_rejected` and publish nothing.

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
- `EXTENSION-NETWORK.md` — §6.5 transport profiles (`endpoint` shape consumed by §3 `transports` field)
- `EXTENSION-IDENTITY.md` — peer-id substrate + identity publish surface
- `EXTENSION-ATTESTATION.md` — supersedes-chain discipline referenced in §3 / §6.5

---

## §14 Open questions (informative)

- **Q1: Aggregator conflict surfacing UX.** §8.3 names fail-closed-by-default + explicit-pin-override; aggregator conflict annotation is MAY. Worth per-impl review for actual deployment ergonomics when Mode A lands.
- **Q2: Cross-peer cache propagation.** When a binding is revoked at the source registry, how fast does the revocation propagate through aggregators + consumers? Bound by TTL; subscription-based for live registries; explicit refresh-on-use for cached. Per-backend.
- **Q3: Identity-rotation interaction.** When the publisher of a peer-issued binding rotates their identity, do existing bindings remain valid? Per EXTENSION-IDENTITY §9.5 cap-survival semantics: yes, cap chains rebind; the published binding is signed by the cert at issuance; that cert remains live or is properly superseded via supersedes-chain. Worth cross-checking against EXTENSION-IDENTITY in cross-impl review.
- **Q4: Local-name-store size limits.** Operator concern; exposed as `max_local-names` knob (default unlimited).
- **Q5: Local-name namespace partitioning.** §11.3 MAY but undefined; defer to revision when a driver emerges.
