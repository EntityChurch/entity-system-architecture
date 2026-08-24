# PROPOSAL — REGISTRY service advertisement (A+MX+SRV in one signed zone)

**Status:** **IMPLEMENTED — ratified + folded 2026-08-14** into `EXTENSION-REGISTRY.md` **§3b** (v1.4 → v1.5):
§3b the entity + trust/fail-closed · §3b.1 the `services` field on `ResolutionResult` (§2.1) ·
§3b.2 per-service-type selection · §3b.3 the byte-pinned weight function · §3b.4 prefer-cheap-path ·
§3b.5 trust of advertised infra. Conformance vectors folded into §11.1. *(DRAFT 2026-07-22 →
ratified 2026-08-14.)*

> **Two fold deltas, recorded so neither reads as a transcription slip.**
> 1. **§3's prefer-cheap-path MUST folded as SHOULD** (§3b.4). Two peers ordering their paths
>    differently still connect — the divergence costs the operator money, it does not split a pair
>    or corrupt a seam, so it fails the test a MUST exists to meet. §3.1's rendezvous-hash stays
>    MUST, because a wrong choice there silently never-meets.
> 2. **§3.1's `[+ {...}]` pool notation and the CBOR block were re-expressed** in this spec's
>    prevailing type-definition style. No field, name, or semantic changed.
>
> **What ratification did NOT do: build it.** The entity is implemented in no tree. §3.1's
> *selection function* was built three-way against this DRAFT — `ext/signaling/pool.go`,
> `extensions/signaling/src/pool.rs`, `signaling/pool.py`, each `select(key, pool)` over a
> caller-supplied pool — which is why the INDEX briefly recorded this proposal as "built
> three-way." That named the wrong half and is corrected. The cohort ask is the entity and the
> resolve field.

**Why it sat.** §7 below blocked this on `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` being
proposed-not-landed. **That cleared on 2026-07-31**, when it landed as `EXTENSION-SIGNALING` v1.0 —
and nothing re-triaged this proposal against its own cleared blocker. The 2026-08-13 backlog audit
read the §7 text as current and filed this P3 "real design work, correctly open." It was ratifiable
for two weeks. The consumer ask that surfaced it — `entity-browser-rust` `ROUTING-2026-08-14`, a
browser leg stuck on host candidates for want of exactly this advertisement — came from a different
repo, and the audit had no consumer axis.

**Target:** `specs/extensions/EXTENSION-REGISTRY.md` (amendment — a new advertised entity + one resolve-result
field) + a cross-ref from `EXTENSION-NETWORK.md`/`EXTENSION-RELAY.md`.
**Provenance:** the managed-infrastructure pattern in `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md`
(Part B) + the infra-economics in `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` (which pins the
service set). Completes the "managed infrastructure connected with the registry" ask.
**Scope:** additive; **static, signed, cacheable** — a coral-reef-compatible advertisement served as bytes, **zero
live compute, zero per-connection state.** No wire renumber; no new dispatch behavior.

---

## 0. Motivation

The registry already serves **A** (name → peer + transports) and **MX** (`system/peer/inbox-relay` — where a
peer's mail is held) in **one signed zone** (`EXTENSION-RELAY.md` §3.5, the "A+MX-in-one-zone" pattern). But a
fresh peer that resolves a deployment learns **who + how-to-reach**, not **what shared infrastructure the
deployment offers** — the STUN reflector and the signaling/rendezvous tunnel it needs to *establish a direct
connection*, and the optional data-relay / inbox-relay fallbacks. Today that infra is out-of-band configuration.

`ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` establishes the exact set a peer needs to walk the
cheap core connection path — **reflector + signaling** (core, near-stateless) plus **data-relay + inbox-relay**
(optional, heavy fallbacks). This proposal lets the registry advertise that set, so **one static resolve teaches a
peer the entire deployment: who to reach, how, and what infra is available** — the literal meaning of "managed
infrastructure connected with the registry."

**Why this is budget-aligned:** the advertisement is **static signed bytes on the CDN the registry already is**
(§7.4 coral reef) — advertising the services costs the same as advertising a name: **~nothing.** It also lets a
budget deployment **honestly advertise only what it runs** (Tier 0/1/2, per the reference-deployment guide) and
lets peers **prefer the cheap path** (punch via reflector+signaling) before touching a metered fallback.

## 1. The advertised entity

A deployment publishes one signed, static-served `service-advertisement` in its registry zone (alongside its
bindings). Priority-ordered like MX; **optional services reflect the deployment's budget tier.**

```cbor
service-advertisement = {                    ; type = system/registry/service-advertisement
  "deployment":  <peer_id>,                  ; the deployment/registry identity this set belongs to
  "services": {
    "reflector":  [+ { "endpoint": {url}, "priority": uint }],   ; STUN — observed-address (core path)
    "signaling":  [+ { "endpoint": {url}, "priority": uint }],   ; rendezvous/punch-offer carrier (core path)
    ? "data_relay":  [+ { "endpoint": {url}, "priority": uint,
                          "policy": "open" / "members" / "metered" }],  ; TURN / RELAY Mode C (optional fallback)
    ? "inbox_relay": [+ { "peer_id": <peer_id>, "priority": uint }]     ; RELAY Mode S (optional; deployment-shared)
  },
  "ttl": uint
}
; MUST carry a system/signature at the invariant-pointer path (V7 §3.5); verified against the pinned
; deployment identity exactly like a binding (EXTENSION-REGISTRY §6a.4). Fail-closed on verify/expiry.
```

**Service semantics (the pinned set, from the analysis):**
- **`reflector`** — a STUN-role endpoint that echoes a peer's observed public address. Stateless; core path.
- **`signaling`** — a rendezvous endpoint that carries punch offers/candidates between two peers. Transient
  per-handshake state only; core path. The signaling payload MAY be ENCRYPTION-opaque (the tunnel need not read it).
- **`data_relay`** *(optional)* — a TURN/Mode-C endpoint that relays the *data path* when a punch fails
  (symmetric NAT). `policy` declares who may use it (`open`/`members`/`metered`) — the budget lever. Absent ⇒ the
  deployment offers no data-relay (symmetric-NAT peers fall to store-and-forward or don't connect).
- **`inbox_relay`** *(optional)* — a deployment-shared Mode-S mailbox for offline delivery. Distinct from a
  peer's own `system/peer/inbox-relay` MX (that is *per-peer*; this is *deployment-shared infra a peer MAY adopt*).

## 2. The resolve-result field

`system/registry:resolve(name)` gains an optional `services` field on `ResolutionResult` (§2.1): when the resolved
zone carries a `service-advertisement`, the resolver returns it alongside `(peer_id, transports, ttl)`. **So the
whole cheap core path is learned in the one resolve a peer already does** — no extra round-trip, no new lookup.
The field is OPTIONAL and MUST-ignore when absent (a Tier-0 static registry that advertises no live services is
valid — the valid-floor discipline).

## 3. Selection & the prefer-cheap-path rule (normative guidance)

A peer establishing a connection MUST attempt the **cheap core path first**: direct dial → reflector+signaling
punch → **only then** a `data_relay` (metered) → **only then** store-and-forward via an `inbox_relay`. This is
both the correct latency order and the **budget-preserving order** — a metered relay is touched only when the free
path fails.

### 3.1 Intra-pool selection is **per-service-type** — the rendezvous-hash rule (normative)

Each service entry is a **pool** (`[+ {endpoint, priority}]`) so a deployment can scale a service horizontally
(`ANALYSIS-INFRA-HORIZONTAL-SCALING-AND-STATE-COORDINATION.md` Part E). **How a peer selects *within* a pool
differs by service type, and getting `signaling` wrong is a cross-peer bug** — so the rule is pinned, not left to
"lower priority wins":

| Service | Intra-pool selection | Why |
|---|---|---|
| **`reflector`** | **any member(s)** — a peer SHOULD consult **several** and require agreement (NAT-type detection, `PROPOSAL-NETWORK-REACHABILITY-FACTS` §2.3). `priority` is a preference/failover hint only. | Any reflector independently yields the same fact; no pair-convergence needed. |
| **`signaling`** | **client-side rendezvous-hash (MUST).** Both peers compute `k = rendezvous_key(sorted(peer_a, peer_b))` and `server = rendezvous_hash(k, pool)` (highest-random-weight) → **both independently pick the *same* server** for the handshake. **NOT** lowest-`priority`. | Two peers **must meet at the same rendezvous** for the seconds of the handshake. Naive priority-order or a round-robin LB **splits the pair across servers** and the punch never completes. Rendezvous-hash converges with **zero shared state** — each server owns a shard of the key space (P1: state is addressable; P2: routing is declared, not discovered). |
| **`data_relay`**, **`inbox_relay`** | **priority-order failover (MX semantics)** — lowest `priority` first, next on failure. | Any relay carries the data; no pair-convergence — the *sender* alone picks (mailbox: the recipient's MX **is** the shard map). |

**Why signaling scales with zero shared state:** capacity = sum of the pool; a hot key is one handshake, not a hot
shard; no server needs any other server's state (class-1 transient state is rebuildable). `priority` MAY still
tier a pool, but the **pair-convergence MUST be by rendezvous-hash**, never by bare priority — and how tiering
composes with the hash is pinned in §3.1.1, not left as "weight the hash."

**Stale-pool-skew robustness (SHOULD).** If two peers hold slightly different advertisements (one stale) they may
rendezvous-hash to different servers. Mitigate by (a) each peer trying its **top-2** rendezvous choices (cheap,
covers a single-server pool delta), and only if needed (b) a "looking-for-you" beacon forwarded within the pool
(addresses only, a tiny gossip). Start with (a); add (b) only under measured skew.

#### 3.1.1 The weight function — the bytes, pinned `[cross-peer seam — MUST]` (2026-07-28)

**"Highest-random-weight" is a family, not a function.** The Rust Stage-1 build enumerated at least four free
variables left open above — operand order, the digest, what identifies a member, and how weights compare — and
**two impls can each write textbook-correct HRW and split every pair.** That is §2.2's silent-never-meet exactly
one layer out: same failure, same invisibility to same-impl tests, one level up the stack.

> **Pin.** For a pool member advertised at `endpoint`:
>
> ```
> weight(k, endpoint) = SHA-256( k ‖ endpoint_bytes )
> server              = argmax over the pool, weights compared lexicographically
> ```
>
> - **`k`** is the 33-byte rendezvous key **exactly as derived** (`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH`
>   §2.2) — the same bytes that go on the wire, not a re-hash of them.
> - **`endpoint_bytes`** are the advertised endpoint string's bytes **exactly as published** — no normalization,
>   no case-folding, no scheme or default-port canonicalization. Same rule and the same reason as §2.2's string
>   inputs: both peers read the *same* advertisement, so byte-preservation makes them agree without either running
>   a URL canonicalizer, and a canonicalizer is precisely where two impls would drift apart.
> - **No separator between the two — because `k` is fixed at 33 bytes**, which makes the concatenation unambiguous
>   by construction. Recorded explicitly so it is not read as the missing-separator bug §2.2 had to fix in `pair`.
> - **Highest weight wins**; ties break to the **lower `endpoint_bytes`**. A tie is a SHA-256 collision away and
>   will never be seen, but "argmax" alone is not a total order and an impl must not have to invent the rest.
> - It is a **plain SHA-256, not the substrate content-hash primitive.** This weight is a comparison scalar: it
>   never appears on the wire and addresses no content, so the ECF `{data, type}` envelope §2.2 requires would be
>   ceremony — and a *format-carrying* digest here would reintroduce the very home-format divergence §2.2 exists
>   to pin away.

**`priority` partitions; it does not weight.** "Weight the hash" is withdrawn as an option: a weighting function
is itself unpinned bytes, and stacking a second invented rule on top of the first is worse than not tiering at
all. Instead — **select the lowest `priority` tier present in the pool, then rendezvous-hash within that tier.**
Deterministic, gives `priority` a real meaning, and both peers read the same advertisement so both land in the
same tier before the hash ever runs.

**This pin is not validatable by a single-node deployment**, which is why it went unnoticed until now: `argmax`
over a one-member pool returns that member whatever the weight computes, so every construction agrees and a green
gate says nothing. `PROPOSAL-CONNECTION-NODE` §6 step 2 now requires a **two-instance pool** in the cross-impl
gate for exactly this reason.

> **The entity-native alternative (even less bespoke, forward pointer):** model signaling as **ephemeral entities
> in a rendezvous namespace** sharded by `{k}` — Mode-F relay + SUBSCRIPTION over the system's own coordination
> substrate, TTL-reaped, no durable storage. The rendezvous-hash rule above is the selection primitive either way
> (a dedicated signaling endpoint *or* a namespace shard). The carrier choice lands in
> `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` (open operator call G2); this proposal pins only that the selection
> **is** rendezvous-hash.

## 4. Efficiency & scale (why this stays cheap)

- **Advertisement:** static signed bytes, content-addressed, CDN-edge-cached under TTL — **zero marginal cost,
  infinite read scale** (it is the coral reef, §7.4). Rotating the set = publishing a new signed entity.
- **No live registry compute:** advertising services does not make the registry live — a Tier-0 static registry
  advertises a Tier-1/2 deployment's live services by *reference*, holding no connection state itself.
- **The peer does the selection**, not the registry — the registry never brokers a connection; it only *names*
  the infra. All per-connection work stays at the reflector/signaling/relay (per the analysis' state table).

## 5. Alternatives considered

- **A live registry `:find-service` op** — rejected; it makes the registry live + stateful (a lookup server),
  defeating the coral-reef minimalism. A static signed advertisement is strictly cheaper and cacheable.
- **Per-peer service declarations only** (extend `inbox-relay` MX to all services) — rejected as the *primary*
  shape: reflector/signaling are **deployment-shared** infra, not per-peer facts; advertising them once per
  deployment (not once per peer) is far cheaper and matches reality. (A peer MAY still override per-peer.)
- **DNS SRV records** — rejected as the substrate (it reintroduces a DNS dependency); this is the *entity-native*
  SRV-analog, served by the registry-that-is-just-a-peer, verifiable against the pinned identity.

## 6. Conformance

A `registry` vector: publish a signed `service-advertisement`; `resolve` returns it in `services`; a tampered/
expired advertisement fails closed (no silent downgrade, §6a.4); a Tier-0 zone with no advertisement resolves
normally with `services` absent (valid floor). Rides the REGISTRY conformance category.

## 7. `[ASK-ARCH]` / open

- **Reflector/signaling handler specs** are proposed-not-landed (`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH`, from
  the connectivity exploration); this advertisement *names* their endpoints but their protocol is that proposal's.
  This proposal is publishable ahead of them (it advertises endpoints; the endpoints' wire protocol lands with the
  connectivity track).
- **Trust of advertised third-party infra:** a deployment vouches for the infra it advertises (signed set); a peer
  MAY additionally pin/allow specific reflector/relay identities. Cross-deployment shared community relays are a
  follow-on (composes with the aggregator/federation, REGISTRY §8.2, deferred).

## References

- `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` (the service set + prefer-cheap-path order + the
  state/cost table), `ANALYSIS-INFRA-HORIZONTAL-SCALING-AND-STATE-COORDINATION.md` **Part E** (the pool +
  rendezvous-hash selection rule folded into §3.1; P1/P2), `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md`
  §B (A+MX+SRV), `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the four separable services).
- `PROPOSAL-NETWORK-REACHABILITY-FACTS.md` §2.3 (defines the `reflector` op this advertisement names; multi-reflector
  agreement = the reflector intra-pool rule in §3.1).
- `EXTENSION-REGISTRY.md` §1/§3/§6a.4/§7.4 (binding shape, verify, coral reef), `EXTENSION-RELAY.md` §3.5 (MX,
  A+MX-in-one-zone), §3.4/§11.1 (Mode C data-relay), `EXTENSION-NETWORK.md` §10.
