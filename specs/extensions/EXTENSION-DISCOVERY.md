# EXTENSION-DISCOVERY

**Version**: 1.1
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.40+) — the only prerequisite; the grant-prompt flow (§2) is ordinary capability machinery.
**Related**: EXTENSION-SIGNALING.md (the carrier the `rendezvous` backend rides, §5.5); EXTENSION-NETWORK.md (an admitted peer is dialed over whatever transport profiles it advertises, §6.5 — discovery hands off, it does not connect); EXTENSION-REGISTRY.md (the *sibling* mechanism, name→peer, not a prerequisite — §1, §10); EXTENSION-IDENTITY.md (identity verification is post-admission and out of scope, §10)
**Tier:** Operational — Tier 2b (network), per `SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.

---

## §1 Concept — discovery is the *holder*

REGISTRY (sibling extension `EXTENSION-REGISTRY.md`) answers **"given a name, where is this peer?"** (lookup, name→peer). Discovery answers **"what peers are out there that I don't already know?"** — a different mechanism.

Like NETWORK and REGISTRY, Discovery is a **substrate with pluggable backends**, piecemeal by design: it starts small (mDNS on the local network) and grows to hold the other ways peers find each other and decide to connect — QR exchange, registry-assisted discovery, gossip/DHT, and the connect/share UX that goes with them.

**The unifying job:** *surface candidate peers, and mediate the human decision to admit them.* Discovery never silently connects you to strangers. It *finds* and *prompts*; the user *decides*.

---

## §2 The grant-prompt flow (the core interaction)

When a backend surfaces a candidate peer, Discovery raises a decision to the user:

> "Found *peer X* on the network. Add them? Grant a token? Track them?"

Outcomes:

- **Ignore** — discard the candidate; do not connect, do not track. (Default-safe.)
- **Track only** — remember the peer exists; no grant, no connection authority yet.
- **Grant limited** — admit them with a narrow, explicit capability ("I don't know who this is — give them limited access; we'll see"). The grant is ordinary EXTENSION-IDENTITY / capability machinery; Discovery just initiates it.
- **Grant more** — later, deliberately, the user widens the grant.

This is **fail-closed and incremental**: nothing is admitted without an explicit human choice, and grants only ever widen by deliberate action. Discovery is the *initiator* of the grant, never the *authority* — the cap roots at the granting peer exactly as normal.

```
backend surfaces candidate
        │
        ▼
  system/discovery/candidate   ──► prompt ──► {ignore | track | grant-limited | grant-more}
        │                                              │
        └────────── (no admission without a choice) ───┘
```

### §2.1 Entity types

**Wire convention for optional fields (MUST; erratum).** In this spec, optional fields are encoded **absent** (the CBOR map key is not present), NOT explicit-null. The schema syntax `<X | null>` and `<X, BARE | null>` below means: present-with-value when known; **absent** when not. This follows V7 ECF canonical-form discipline for optional fields. Implementations MUST emit absent for unset optionals; implementations MUST decode both absent and (legacy) explicit-null as the unset value (tolerant read on receive, canonical on emit). Pin caught when three impls picked two encodings on `candidate.peer_id` in the cross-impl run; ECF treats null and absent as distinct canonical forms (different `content_hash`) so the spec-wide convention is load-bearing for cross-impl byte equality. Pinned in the registry/discovery cross-impl-run absorption (Ruling-6).

```
type: "system/discovery/candidate"
data: {
  peer_id:       <Base58 peer-id per V7 §1.5 | null>,  ; absent until IDENTIFY completes
  backend:       <"mdns" | "rendezvous" | "qr" | ...>,   ; "rendezvous" — §5.5
  observed_at:   <ms-since-epoch>,
  endpoint_hint: <opaque>,                  ; e.g. LAN address + port, or QR payload
  identity_hint: <system/hash, BARE | null>,  ; hash of an IdentityClaim (see §2.2); null = TOFU
  supersedes:    <system/hash, BARE | null>   ; for successor candidates per §2.2
}

type: "system/discovery/decision"
data: {
  candidate:  <system/hash, BARE>,                              ; head of the candidate chain (§2.2)
  outcome:    "ignore" | "track" | "grant-limited" | "grant-more",
  grant:      <system/hash, BARE | null>,                       ; bare hash of system/capability/grant entity per V7 §6.2; null for ignore/track
  decided_at: <ms-since-epoch>
}
```

All durations / timestamps in this spec are **milliseconds since Unix epoch (UTC, signed int64)** — aligned with V7 cap-expiry convention.

The `decision.grant` field references a `system/capability/grant` entity (per V7 §6.2 capability-mint) by bare hash, NOT via a `refs:` block — V7 uses target-matching, not refs.

### §2.2 Candidate sequencing — successor pattern (entities are immutable)

A candidate's `peer_id` field is **absent** when the candidate is first surfaced by a backend — the backend has observed a peer presence but identity is unverified. (Per §2.1 the optional-field wire convention is absent-not-null; the spec prose elsewhere uses "null" interchangeably as the conceptual unset value, but the wire encoding is absent.) The `grant-limited` / `grant-more` outcomes reference "the granting peer," which requires a `peer_id`. The sequencing:

1. Backend surfaces `candidate_0` with `peer_id` absent (per §2.1 wire convention).
2. User outcome is `track` / `grant-limited` / `grant-more` (decision_0 references `candidate_0`).
3. For grant outcomes: connection is initiated against the candidate's `endpoint_hint`.
4. IDENTIFY (EXTENSION-NETWORK) completes against the admitted channel, establishing the peer's actual `peer_id`.
5. **A successor candidate is created** (entities are content-addressed and immutable — they cannot be mutated). `candidate_1` is a new `system/discovery/candidate` entity with `peer_id` populated and `supersedes: <candidate_0.content_hash>`, per the ATTESTATION supersedes-chain idiom. A successor `decision_1` SHOULD reference `candidate_1`. The original `candidate_0` remains in the tree as the observation record.

`decision.candidate` SHOULD reference the head of the chain (most-recent non-superseded candidate).

### §2.2.1 `identity_hint` semantics

`identity_hint` is `<system/hash, BARE | null>`:

- **`null` → TOFU** (trust-on-first-use). The candidate's backend made no identity claim; admission post-IDENTIFY is at the user's discretion (the §2 grant decision IS the trust anchor).
- **non-null** → the bare `system/hash` of an `IdentityClaim` entity:

```
type: "system/discovery/identity-claim"
data: {
  peer_id:           <Base58 peer-id per V7 §1.5>,  ; claimed peer-id
  key_type:          uint,                          ; V7 §1.5 key-type byte
  hash_type:         uint,                          ; V7 §1.5 hash-type byte
  public_key_digest: <bytes>                        ; V7 §1.5 public-key digest
}
```

Comparison: post-IDENTIFY, the receiver constructs an `IdentityClaim` from the actual IDENTIFY result and computes its `content_hash`. Admission MUST fail-closed if the constructed `identity_hint` does not equal the candidate's advertised `identity_hint`. This matches V7 §1.5 multikey discipline.

If IDENTIFY fails or returns a peer-id whose identity-claim hash mismatches what the candidate advertised, the grant MUST be rejected (the user agreed to grant based on what the candidate claimed; if the claim doesn't verify, admission fails closed).

---

## §3 v1 backend — mDNS (same-network)

The first and only v1 backend: **mDNS / zero-config multicast** for finding peers on the local network. This is what the same-network demo needs.

```
system/discovery:scan(backend: "mdns", filter?)         → ScanResult
system/discovery:announce(backend: "mdns", profile_ref) → ()    ; advertise self on the LAN
system/discovery:announce-stop(backend, profile_ref)    → ()    ; end an announce session
```

`ScanResult` shape:

```
{
  candidates: [<system/hash, BARE>],   ; bare hashes of system/discovery/candidate entities; immediate snapshot
  truncated:  bool,                    ; true if per-scan candidate-count ceiling was exceeded
  code:       <string | null>          ; null normally; "discovery_scan_overflow" (503) when truncated
}
```

### §3.0 Hybrid `:scan` shape — snapshot return + watchable candidate prefix

`:scan` has a **hybrid shape**: it returns an immediate snapshot (the request/response form, suitable for one-shot backends like QR and for simple polling consumers) AND establishes/refreshes a watchable browse session that writes/removes candidate entities at `system/discovery/candidate/{backend}/*` as peers arrive and depart.

- **Snapshot return:** `:scan` returns the current set of known candidates immediately as a list of bare hashes.
- **Watchable prefix:** the same call starts (or refreshes) a browse session. The session writes `system/discovery/candidate/{backend}/{candidate_id}` entities into the tree as peers arrive (mDNS announcement, post-query response, etc.) and **removes** them on departure (mDNS goodbye records, TTL expiry — per the reap rule §3.0.1). Live consumers (the §2 grant-prompt flow; reactive UIs) subscribe to the `system/discovery/candidate/{backend}/*` prefix and react to add/remove events.
- **Reactive-default discipline:** this matches the substrate's reactive-default model — handlers pop entities into the tree; consumers subscribe and react. mDNS is a streaming protocol in its native form; modeling it as a snapshot loses arrival (post-query) and departure (goodbye / TTL expiry). The hybrid recovers correctness without breaking uniform-handler-shape with REGISTRY.
- **Backend-flexible:** QR / one-shot backends fit cleanly — they write a candidate once and never reap (no liveness signal to act on). Streaming backends use the watchable prefix as their natural surface.

### §3.0.1 Reap rule (cross-impl convergence point)

Candidate entities under `system/discovery/candidate/{backend}/*` are reaped per these rules:

1. **mDNS goodbye records (TTL=0)** MUST cause immediate removal of the corresponding candidate entity.
2. **TTL expiry without renewal** — a candidate is reaped when `now - last_seen > grace_window`, where `grace_window` defaults to `2 × last_TTL` per mDNS RFC 6762 §10.5 reap discipline. `grace_window` is operator-configurable.
3. **Explicit `:scan` re-invocation** refreshes `last_seen` for candidates still observed; missing candidates from the new query age out per (2).
4. **Backends with no departure signal** (QR and other one-shot exchanges) have no TTL semantic; impls keep them per §7's `decision_retention_window` policy — they're only reaped on explicit unbind or operator GC. **The distinction this rule draws is the *signal*, not the shape of the call `[v1.1]`** — a backend that is not mDNS but *does* observe departures is rule 5, not this one. Reading rule 4 as "every non-mDNS backend" leaves a departed peer on the list forever, showing a user someone they can no longer meet.
5. **Backends with a carrier-native departure signal `[v1.1]`** — a candidate absent from the carrier's current observation is reaped **immediately**, with no local grace window. `rendezvous` (§5.5) is the first: the carrier reaps a deposit at its own TTL, so a counterpart that stops re-offering falls out of the bucket within one TTL. That is a stronger signal than rule 2's inference-from-silence, because it is the carrier's own contract. **No second grace window is applied on top of the carrier's** — doing so doubles the disappearance latency for no gain.

`last_seen` is a per-impl operational field, not stored in the candidate entity (entities are immutable). Impls keep it in a session-local index.

### §3.1 Resource bounds

Implementations MUST honor V7 §4.10(a) on per-candidate payload + a DISCOVERY-owned per-scan-count ceiling:

- **§4.10(a) max-payload — per-candidate**. A misbehaving LAN broadcaster sending oversized records MUST be rejected at the substrate level with V7 §4.10(a) **`413 payload_too_large`**; the affected candidate is not emitted. This reuses the v7.75 cohort-converged code cleanly (allocation-safety is the substrate concern §4.10 addresses).
- **`:scan` per-call candidate-count ceiling — DISCOVERY-owned code.** Implementations MUST apply a configurable upper bound on candidates emitted from a single `:scan` snapshot. Default informative: 1024 candidates per scan; deployments configure per network size. Exceeding the bound MUST surface in `ScanResult` as `truncated: true` + `code: "discovery_scan_overflow"` (HTTP-status-mapped to 503 per V7 §3.3 DISCOVERY code domain) — NOT silent truncation. The remaining candidates are dropped from this scan's snapshot; re-invocation with a `filter` is the user-driven path forward. (This code lives in DISCOVERY's own code domain per V7 §3.3 — it is not a V7 §4.10 floor code; per-scan-count ceilings are an application-layer concern, not the substrate's allocation-safety concern. The earlier draft's `resource_exhaustion` reference was incorrect — V7 §4.10 has no such code.)
- **Sustained `:scan` invocation rate** — implementations SHOULD apply rate-limiting per `system/capability/discovery-scan` cap-holder to prevent scan-flood self-DoS. Bounds operator-configurable; defaults conservative.

**Watchable prefix is also bounded:** the same per-candidate payload bound (§4.10(a)) applies; the per-scan-count ceiling extends to the watchable session as a per-session-candidate ceiling. When the watchable session is at ceiling, new candidates are NOT written; the impl logs the drop (per inspectability discipline).

### §3.2 mDNS DNS-SD wire-interop (cohort-convergence pin)

mDNS interop constants are pinned normative — the cross-impl-divergence class workbench-go and Python both flagged as the **silent** failure mode (Go and Rust never see each other on the LAN if these differ; no error to catch).

- **DNS-SD service-type:** `_entity-core._udp.local.` (RFC 6763 §7 service-name convention).
- **MUST-present TXT keys:**
  - `version=1` — DISCOVERY major version.
  - `peer_id_hint=<Base58 peer-id>` — the advertising peer's peer-id; MAY be omitted when announcing anonymous-pre-IDENTIFY.
  - `profile_ref=<profile-id>` — the transport profile-id to dial (per NETWORK §6.5 `system/peer/transport/{peer}/{profile-id}` namespace).
- **OPTIONAL TXT keys:**
  - `proto=<comma-list>` — advertised transport-types (e.g. `webrtc,tcp,http-poll`).
  - `display_name=<UTF-8>` — user-facing label hint.
- **SRV record:** required per RFC 6763 §4.2 for port advertisement.
- **Unknown TXT keys:** MUST be ignored (forward-compat).

The candidate entity's `endpoint_hint` for the mDNS backend is the resolved SRV target + port + (when available) TXT keys assembled into the candidate's tree-write at `system/discovery/candidate/mdns/{candidate_id}` per §3.0.

### §3.3 Filter shape

`filter` is an opaque object | null, backend-specific; backends MAY ignore. Same shape as REGISTRY-RESOLVE's `hints` field. mDNS-backend filter MAY constrain by service-name predicate or TXT-key predicate; not v1-normative. When `filter` is absent or present-but-ignored, the backend emits unfiltered candidates; backends MUST NOT silently return zero candidates when filter is unparseable — surface as error code, not empty result.

**Unknown backend (MUST; erratum).** When `:scan(backend, ...)` names a `backend` the peer has not registered (the named backend string is not in the peer's registered-backend set), the handler MUST return **`400 unknown_backend`** (V7 §3.3 request-input class — unknown enum value). This is NOT 404 (`backend` is a parameter value, not a resource path/address; 404 is reserved for the addressed-resource-not-found case). This is NOT a silent empty result (per §8.4). Distinguished from the "no candidates this scan" outcome which is a successful `ScanResult { candidates: [], truncated: false, code: null }`. Pinned in the registry/discovery cross-impl-run absorption (Ruling-5).

**Unknown `profile_ref` (MUST; same class) `[added 2026-08-10; corrected 2026-08-11]`.** When `:announce(backend, profile_ref)` or `:announce-stop(backend, profile_ref)` names a `profile_ref` **the named backend does not recognize**, the handler MUST return **`400`** (`unknown_profile_ref`) — **not `500`**. This is the same Ruling-5 class as the bullet above: an unrecognized parameter *value* is a caller error, and the identical handler already applies that rule to `backend` two lines away. **A `500` additionally tells the caller to retry something that can never succeed.** *(Sentinel in go: `discovery.ErrUnknownProfileRef`, raised by the backend's resolver table — `4375cea`.)*

> **What `profile_ref` resolves against, and the correction `[RULED 2026-08-11]`.** The sentence above previously required resolution against a published `system/peer/transport/{peer}/{profile-id}` **entity**. That was a drafting slip and it made the mandatory happy path unreachable — **routed independently by all three implementations in one cycle.** The `profile_ref` is a value **the backend recognizes**, not a tree lookup: `unknown` means *this backend has no such profile*, exactly as `unknown_backend` means *this peer has not registered that backend*. The Ruling-5 class was right; the mechanism sentence described a different one.
>
> **Why the tree reading cannot be the rule, on the corpus's own terms:** `EXTENSION-NETWORK.md` §6.5.1a (D1) makes profile self-publication a **SHOULD** — "RECOMMENDED, not REQUIRED" — and then says in terms that **"Consumers MUST NOT assume the self-published path exists."** A handler `MUST 400` keyed on that path's absence *is* assuming it exists, and converts a documented optional into a hard caller error. **A MUST may not rest on a SHOULD**; that is the general form, and it is worth more than this instance.
>
> **`:announce-stop` is idempotent for a recognized `profile_ref` `[MUST]`.** Stopping a profile that is not currently announced **succeeds with `200`** — a symmetric lifecycle stop is idempotent. The corpus stated this nowhere, which is why an implementation asserting it in a conformance check had to cite §3/§8.1 loosely and why the `400` above could be written straight through it. **The two rules do not overlap once the resolution question is settled:** *unrecognized by the backend* → `400` on **both** ops; *recognized but not running* → `200` on stop. An implementation whose `announce-stop` never maps its unknown-profile sentinel to `400` is **not** conformant merely by being idempotent — the idempotency ruling covers the second case only.

mDNS is a discovery + signaling carrier only — it surfaces candidates and helps WebRTC establish; it is **not** a trust surface. Trust is the IDENTIFY handshake over the resulting channel plus the user's grant choice.

### §3.4 WebRTC follow-on + browser-flagship framing (informative)

**Native (Tauri / desktop):** mDNS LAN scan + grant-prompt + admit is end-to-end usable in v1. Admitted peers connect via whatever transport profiles they advertise (NETWORK §6.5). The full "discover → connect → dispatch over WebRTC" demo depends on `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md` (DRAFT, not landed) graduating — until then, native discovery + connection works via TCP / http-poll profiles the discovered peer advertises.

**Browser flagship (WASM):** mDNS is unavailable — browsers do not speak multicast UDP. So in the browser flagship, the v1 mDNS backend gives nothing actionable. Browser discovery is structurally different:

- **Registry resolution** (sibling `EXTENSION-REGISTRY.md`) IS the browser's name-finding mechanism — name → peer, with the preloaded ESR pattern (REGISTRY §7.4).
- **Link-following** — when browsing a host, follow its links to other hosts (rung-2 vouching policy lives in the L5 SDK browse seam, not the substrate).
- **Future QR + registry-assisted backends** (§6 staged growth) — these will be the browser-relevant DISCOVERY backends. QR is near-term, registry-assisted reuses the ESR shipping with REGISTRY.

This is a physics limit, not a spec gap. State it plainly: in the browser, "discovery" means registry + link-following + (future) QR / registry-assisted — NOT mDNS.

---

## §4 Capability model

```
system/capability/discovery-scan       ; who may run discovery scans on this peer
system/capability/discovery-announce   ; who may :announce and :announce-stop on this peer
```

- Discovery results carry **no implied trust.** A candidate is an observation, not an authorization. Admission is the §2 grant, which is ordinary capability machinery.
- There is **no** "discovery grants access" cap — by construction discovery cannot admit a peer; only the user's explicit decision can.

### §4.1 Default grants on first install

The DISCOVERY seed-policy bootstrap (per V7 §6.9a) grants the local peer both caps above. Otherwise the user cannot scan their own network or announce themselves. Distributions MAY tighten via `--default-grants` flags, but the v1 floor grants the local peer full self-access.

---

## §5 Composition with other extensions

### §5.1 EXTENSION-NETWORK

Discovery sits beside connection. Once a candidate is admitted, connection is ordinary NETWORK dispatch. The advertised profile in the candidate's `endpoint_hint` is typically `webrtc` for LAN flows or `http-poll` / `tcp` for general flows.

### §5.2 EXTENSION-WEBRTC-TRANSPORT (DRAFT, follow-on)

mDNS is WebRTC's `discovery:mdns` signaling carrier. The two compose into the same-network demo when WEBRTC lands.

### §5.3 EXTENSION-IDENTITY

IDENTIFY over the admitted channel establishes who the peer actually is. The grant (`grant-limited` / `grant-more`) is issued against that identity. Identity-hint mismatch between what the candidate advertised and what IDENTIFY returns MUST fail closed (§2.2).

### §5.4 EXTENSION-REGISTRY (sibling)

REGISTRY and DISCOVERY are distinct concerns:

- REGISTRY = lookup (given a name, find the peer).
- DISCOVERY = find (what peers are out there I don't know?).

**Registry-assisted discovery** — a candidate carrying a name that REGISTRY can resolve — is a later DISCOVERY backend, not a coupling at the substrate layer. The boundary stays clean.

### §5.5 EXTENSION-SIGNALING — the `rendezvous` backend `[v1.1]`

`EXTENSION-SIGNALING`'s keyed mailbox is a DISCOVERY **carrier**, and `rendezvous` is the backend that rides it. This is the pattern the two landed backends already use — §5.2 pairs mDNS with WebRTC's `discovery:mdns` carrier; §6 pairs QR with `manual:qr` — and the seam is stated from both sides: SIGNALING §1.2 is *"the key introduces; it never authorizes,"* and §2 here is *"discovery is the **initiator** of the grant, never the **authority**."*

#### §5.5.1 Only three of the four key modes are discovery `[MUST]`

`EXTENSION-SIGNALING` §3.2 defines four rendezvous key modes. **`tag`, `secret` and `lobby` are discovery: they surface a counterpart you did not already know. `pair` is not** — its derivation input is the two peer-ids, so the caller already holds the counterpart's identity and nothing is discovered. A backend MUST NOT surface `pair`-mode meetings as candidates; `pair` is an establishment path (`EXTENSION-NETWORK` §10.3), not a discovery one.

#### §5.5.2 `identity_hint` MUST be absent, and TOFU is the ceiling rather than a fallback `[MUST]`

**The rule:** a `rendezvous` candidate MUST carry `identity_hint` absent, and a backend MUST NOT synthesize an `identity-claim` for a peer that stood at a key.

**The reason, which is not obvious at the call site:** a carrier whose deposits are signed (SIGNALING §6.3) hands the backend a *cryptographically verified* signer at the moment it surfaces a candidate, and the rule requires discarding it. That is deliberate. **A signature at a mailbox proves who deposited a blob — not that the channel admission later opens belongs to that peer.** Binding `identity_hint` to the depositor would make §2.2.1's fail-closed IDENTIFY comparison assert a relationship nobody promised, and it would refuse admission whenever the peer you end up talking to differs from the depositor.

**The consequence, stated here so each implementer does not re-derive it:** §2.2.1's fail-closed comparison — the strongest check this extension defines — is **structurally unavailable to every `rendezvous` candidate.** For this backend TOFU is the ceiling, not a degraded mode. The §2 grant decision is the trust anchor.

#### §5.5.3 `endpoint_hint` is at deposit granularity, and is mode-dependent `[MUST]`

**Deposit granularity, not bucket granularity.** A key is reachable by anyone who holds its input, so two counterparts at one `tag` produce candidates identical in `backend`, `peer_id` (absent), `identity_hint` (absent) and — at bucket granularity — `endpoint_hint`, differing only in `observed_at`. **A consumer then cannot distinguish two peers from one peer observed twice**, and `observed_at` cannot settle it because re-observation also moves the timestamp. `endpoint_hint` MUST therefore identify **the specific deposit** — carrier node, mode, and an opaque per-deposit identifier — which is also what the admission path needs in order to act on that deposit.

**A `secret`-mode candidate MUST NOT carry the secret or the derived key.** For `tag` and `lobby` the derivation input is publishable — SIGNALING §3.2 types a tag as a *"public label — discovery convenience, not access control,"* and a lobby constant is a published deployment name. **A `secret` input *is* the access control, and the derived key is equivalent to it.** A candidate entity exists to be displayed for the §2 decision and is written to the tree at rest, so carrying either discloses the credential through the one surface whose job is display. Omit the label for `secret` mode; the local peer already holds the configuration it supplied. **A backend that handles all three modes uniformly here ships a credential leak that every test passes.**

#### §5.5.4 The successor chain

`rendezvous` follows §2.2's two-entity pattern without variation: `candidate_0` with `peer_id` absent when a counterpart is observed at the key; IDENTIFY completes over the admitted channel; `candidate_1` with `peer_id` populated and `supersedes` set to `candidate_0`'s content hash. `candidate_0` remains as the observation record — entities are immutable — and `decision.candidate` SHOULD reference the head of the chain. Per §2.1's wire convention, unset optional fields are **absent, not explicit-null**.

#### §5.5.5 A key is a meeting point; the peer met is what gets a name

Binding a rendezvous **key input** as a REGISTRY local-name is a category error: the name would resolve to a meeting point rather than to a peer, and the next occupant of that key inherits it. Binding the **peer** is ordinary — after IDENTIFY the successor candidate carries a verified `peer_id`, and binding that to a local petname is REGISTRY's local-name backend.

---

## §6 Staged growth (deferred from v1, named for shape)

Discovery is expected to keep growing as the holder for peer-finding. **`rendezvous` has landed and is no longer on this list — it is §5.5.**

- **QR / short-code backend** — out-of-band candidate exchange; pairs with WebRTC `manual:qr` signaling. Near-term, small. **Shape only: no entity shape, no backend token, and no composition subsection are defined — do not build against it as though it were settled.** It is the same mold as §5.5's `rendezvous` (out-of-band candidate exchange, then §2.2's successor chain), with one difference: a QR payload MAY legitimately carry an `identity-claim`, because the displaying peer asserts its own peer-id in the payload it controls — where §5.5.2 forbids one for a key rendezvous. Build `rendezvous` and QR is an additive second backend, not a second design.
- **Registry-assisted discovery** — surface peers a trusted registry knows about.
- **Relay / DHT / gossip backends** — internet-scale peer-finding. Later, driver-gated.

Each is an additive backend under the §1 substrate + §2 prompt; none changes the core interaction.

---

## §7 GC posture (per `GUIDE-GC.md`)

- **`system/discovery/candidate` entities under `system/discovery/candidate/{backend}/*` (live watchable surface):** reaped per §3.0.1's liveness rule (mDNS goodbye → immediate; `last_seen + grace_window` → expired). NOT wall-clock retention. The reap discipline replaces the earlier 24h-wall-clock heuristic (which produced ghost peers in any live UI).
- **Historical `system/discovery/candidate` entities (superseded chain history):** retain per `discovery-config.candidate_history_retention` (default 24h); operator MAY evict.
- **`system/discovery/decision` entities:** retain durably (auditable history of who was admitted, when, with what grant); operator MAY evict.
- **`announce` session state:** lifetime of the announce session; ended by `:announce-stop` (§3), cap revoke, or peer shutdown.
- **mDNS browse-session state (per `:scan`):** per-impl session-local; not stored in the tree.

All retention windows are operator-configurable knobs; defaults conservative.

---

## §8 Cross-impl conformance

### §8.1 MUST implement

- `system/discovery:scan(backend, filter?) → ScanResult` per §3 (snapshot-return + watchable-prefix hybrid per §3.0).
- `system/discovery:announce(backend, profile_ref)` + `system/discovery:announce-stop(backend, profile_ref)` per §3 — including §3.3's two-case rule: a backend-unrecognized `profile_ref` is `400 unknown_profile_ref` on **both** ops, and `announce-stop` on a recognized-but-not-running profile is an idempotent `200`.
- mDNS v1 backend per §3 — DNS-SD service-type + TXT key schema per §3.2.
- Liveness reap rule per §3.0.1 (mDNS goodbye → immediate; `last_seen + grace_window`).
- Grant-prompt flow per §2 — emit `candidate` entities; require explicit `decision` before admission.
- Candidate `peer_id` null-until-IDENTIFY sequencing per §2.2 — successor-candidate via supersedes-chain; `decision.candidate` references chain head.
- `identity_hint` null = TOFU; non-null = IdentityClaim hash comparison per §2.2.1 — mismatch MUST fail closed.
- Resource bounds per §3.1 — per-candidate payload bound (V7 §4.10(a) `413 payload_too_large`); per-scan candidate-count ceiling with `truncated: true` + `code: "discovery_scan_overflow"` (503) surfacing — NOT silent truncation; SHOULD rate-limit `:scan` per cap-holder.
- No "discovery grants access" cap (admission is the §2 user decision only).
- V7 §5.2 / §975 refless target-matching: `decision.grant` references the cap-grant entity by bare `system/hash`, not via a `refs:` block. The cap-grant entity is `system/capability/grant` per V7 §6.2.

### §8.2 SHOULD implement

- `:scan` invocation-rate limiting per §3.1.
- Operator-configurable retention windows per §7.
- UI / CLI surface for the §2 prompt flow (impl convenience).

### §8.3 MAY implement

- Additional backends (QR, registry-assisted, gossip, DHT) per their own follow-on proposals.
- `filter` interpretation richer than the spec's "opaque, backend MAY ignore" baseline.
- Decision history surfacing / audit UI.

### §8.4 MUST NOT

- Silently admit a candidate without an explicit `decision` outcome.
- Issue a `grant-limited` / `grant-more` cap when the post-IDENTIFY identity-claim hash doesn't match the candidate's non-null `identity_hint` (§2.2.1).
- Truncate `ScanResult` snapshot silently when over-bound — MUST surface as `truncated: true` + `code: "discovery_scan_overflow"` per §3.1.
- Mutate an existing candidate entity to populate `peer_id` post-IDENTIFY — entities are immutable; use the successor-candidate pattern per §2.2.
- Use a `refs:` block to reference the cap-grant on a `decision` — V7 uses target-matching via the bare `system/hash` field per §2.1.
- Retain candidates under the live watchable surface (`system/discovery/candidate/{backend}/*`) past the §3.0.1 reap rule (no wall-clock retention; no ghost peers).

---

## §9 Configurations (deployment patterns)

- **LAN-only demo.** mDNS scan + announce + grant-prompt; user admits known LAN peers; admitted peers connect via WebRTC (when it lands) or whatever else they advertise.
- **Headless server.** `discovery-announce` cap granted to a service; `discovery-scan` cap not granted to any peer; the server is discoverable but doesn't scan for others.
- **Air-gapped / no-LAN.** No mDNS; QR backend only when it ships; candidates exclusively from out-of-band exchange.
- **Browser / WASM.** mDNS unavailable (browsers don't speak multicast); future QR / registry-assisted backends are the browser-relevant ones.

---

## §10 What this extension does NOT cover

- **Trust / admission policy.** Discovery initiates; the grant is ordinary capability machinery; trust posture is the receiver's choice.
- **Cross-network discovery.** mDNS is LAN-scoped by design. Internet-scale peer-finding is a future backend (DHT / gossip / relay).
- **Identity verification.** EXTENSION-IDENTITY's IDENTIFY handshake establishes peer identity post-admission. Discovery just surfaces the candidate.
- **REGISTRY (name lookup).** Distinct extension; see EXTENSION-REGISTRY.md.
- **WebRTC transport.** Currently a DRAFT proposal; DISCOVERY v1 lands without it.
- **Inspect-before-grant UI** — fetching a candidate's public manifest / sites BEFORE the user decides is L5 territory; pilot lives in entity-browser-rust; promote to `GUIDE-L5-BROWSE-RESOLUTION` when SPACES validates the policy shape. Substrate stays at find-and-prompt.

---

## §11 Cross-references

- `proposals/implemented/PROPOSAL-EXTENSION-DISCOVERY.md` — landed-into source proposal
- `EXTENSION-REGISTRY.md` — sibling extension (lookup; this extension is find)
- `EXTENSION-NETWORK.md` — composition for connection post-admission
- `EXTENSION-IDENTITY.md` — IDENTIFY against the admitted channel
- `docs/proposals/implemented/extensions/PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md` — DRAFT; the eventual transport for LAN mDNS-discovered peers
- `ENTITY-CORE-PROTOCOL.md` §4.10 — resource bounds floor (referenced in §3.1)

---

## §12 Open questions (informative)

- **Q1: `filter` shape for richer backends.** §3's mDNS `filter` is opaque + backend-MAY-ignore. Richer backends (registry-assisted, gossip) MAY define structured filter shapes; substrate stays opaque.
- **Q2: Decision-history audit UX.** §7 says decisions are durable; per-impl UI / CLI surface for "show me everything I've admitted" is impl convenience, not substrate.
- **Q3: Multi-LAN / VPN-traversal scope for mDNS.** mDNS is single-broadcast-domain by protocol; future tunneling / bridged-mDNS configurations are deployment concerns, not substrate.
