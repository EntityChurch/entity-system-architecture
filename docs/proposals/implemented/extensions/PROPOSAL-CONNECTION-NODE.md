# PROPOSAL — the connection node (the standalone rendezvous + reflector service)

**Status:** **RATIFIED + FOLDED (2026-07-31)** — landed as `specs/extensions/EXTENSION-SIGNALING.md` **v1.0**
(node-side: §2.1 roles, §4 operations, §5 bucket semantics, §9 the unwrapped protocol, §8 admission).
**The spec is source of truth from here**; this document is the design record and the rationale that did not fold.
**Build state (2026-07-31):** the carrier, key derivation and coordination are built in all three; §9's unwrapped surface and the punch are not. §11.5's full gate has not run.
*Was: DRAFT (2026-07-28).*
**Target:** a new **`system/signaling`** service definition — the node-side half of the rendezvous carrier: the
`offer` / `collect` / `advertise` core verb surface plus `reflect` on the unwrapped listener (§1.4), the two
admission **paths** (capability-gated entity / open public bypass, coexisting), and the deployment posture.
Consumes `PROPOSAL-NETWORK-REACHABILITY-FACTS` (what `reflect` returns) and
`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` (how a peer picks a node). No V7/wire renumber; no keystone delta.
**Provenance:** promoted out of `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.1 (node-side verbs) + §2.3
(admission modes + deployment lenses), which flagged the node as the one Phase-1 component without its own
proposal (`HANDOFF-2026-07-24-p2p-connectivity-consolidation-and-phase1-plan` next-session #1). The signaling
proposal retains the **peer-side** half — the coordination messages (§3), the key derivation (§2.2), and the punch
(§5). Split so the node is independently ratifiable, because it is the piece that gets *built and deployed* first.
**Scope:** the mutually-reachable third party two NAT'd peers need in order to meet. **Not** the punch, **not** the
key modes, **not** chat — those are peer-side and live in the signaling proposal.

---

## 0. What this is, in one paragraph

Two peers both behind NAT cannot dial each other and have no public party holding a socket to either of them. The
connection node is the minimum thing that fixes that: a public box that (a) tells a caller how it looks from the
outside, and (b) holds an opaque blob at an opaque key for a few seconds so the other peer can pick it up. It
introduces; it never carries data, never decodes an entity payload, and never learns who met whom beyond a hash.
Everything else in the connectivity stack — the four key modes, the punch, the fallback to relay — is peer-side
behavior layered on those two facilities.

## 1. The wire surface (three core verbs, plus `reflect` on the unwrapped listener)

```
system/signaling:offer(rendezvous_key, message)   → { ok }
system/signaling:collect(rendezvous_key)          → { messages: [<blob>, ...] }
system/signaling:advertise                        → { endpoint, limits }
```

- **`offer` / `collect`** — the keyed opaque mailbox, and the whole of the core. `rendezvous_key` is derived
  **peer-side** and pinned by `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.2 (diagnostics in §2.2.1); the node
  treats it as **33 opaque bytes compared byte-wise** and derives nothing. `message` is an opaque byte string
  whose framing is pinned **peer-side** (that proposal's §3.1); the node never decodes it and never needs to.
- **`advertise`** — announces the endpoint, its rate/TTL limits, and (if it overrides the default) its `lobby`
  constant, for `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` pool membership.
- **`reflect`** — the STUN-role verb. Returns the caller's observed public address, from which a peer derives its
  reachability facts and NAT type (`PROPOSAL-NETWORK-REACHABILITY-FACTS` §0: agreement across several reflectors ⇒
  endpoint-independent mapping, punchable; disagreement ⇒ symmetric, prefer relay). **It is served on the
  unwrapped listener only and is not one of the core's operations** — see §1.4.

**`collect` returns the blobs themselves, not their hashes** `[ruling 2026-07-28, from the Rust Stage-1 build]`.
An earlier draft of this line read `[<hash>, ...]`; that was residue from a content-addressed sketch and **cannot
be implemented as written**. A hash reply needs a fetch surface the node structurally does not have — §1.3 pins
the state silhouette at TTL-reaped buckets with *no bulk storage*, and §5.1 pins that an unwrapped client holds no
entity machinery with which to resolve a hash to bytes. It would also break §2.1: one verb would be completable on
the wrapped surface and not on the unwrapped one, which is precisely the divergence §2.1 says cannot exist.

**The node is mode-blind.** The four key modes (`pair` / `tag` / `secret` / `lobby`) are *entirely* a peer-side
key-derivation concern. The node implements one keyed opaque mailbox and all four modes fall out for free — which
is why "prove it connects through all the modes" is **one code path exercised with four derived keys, not four
features.**

### 1.1 Bucket semantics `[single-impl server — MUST pin]`

Three questions a lone server implementer answers by fiat and never re-asks. Because the server role is
**single-impl** (§1.2), nothing downstream will surface a wrong answer — so they are pinned here, before code:

| # | Question | Pin | Why |
|---|---|---|---|
| 1 | Does `collect` **drain** the bucket or leave the messages? | **Non-destructive.** `collect` returns what is at the key and removes nothing; TTL is the only reaper. A `collect` at a key with no bucket returns an **empty list, not an error** — absence and not-yet are indistinguishable to the node and both are normal. | A handshake has both peers polling, and a `pair` bucket may be collected by both sides and re-read on retry. A draining read makes a retry lose the peer's offer — a silent handshake failure indistinguishable from absence. |
| 2 | Does `offer` at an existing key **append or replace**? | **Append.** A key holds a set of deposited blobs, deduplicated by content hash. | `lobby` and `tag` modes are inherently multi-party; replace would make the last writer erase everyone. Dedup by hash makes a peer's retry idempotent rather than an accumulation. |
| 3 | Is the advertised TTL **binding on peers or advisory**? | **Advisory to peers, binding on the node.** The node reaps on its own TTL; peers MUST NOT assume a blob is still there and MUST be prepared to re-`offer`. | Peers cannot enforce a remote node's reaping, and a peer that treats TTL as a guarantee will hang instead of retrying. |
| 4 | In what **order** does `collect` return a bucket's blobs? | **Deposit order, oldest first.** | A reader scans a bucket for the first message it can act on (§3.2 of the peer-side proposal). Two impls scanning in different orders answer *different peers* out of one shared `lobby` bucket — a cross-peer divergence wearing the costume of a preference. |
| 5 | What are the **size bounds** — per blob, and per bucket? | **8 KiB per blob; 32 blobs per bucket.** An over-size `offer` is **refused with `message_too_large`**, never truncated; a bucket at capacity refuses with **`bucket_full`** rather than evicting. | §4 sizes a handshake at ~1 KB, so 8 KiB is generous by 8×. A limit the client does not know is a cross-impl reject boundary: Go offers 64 KiB at a node that stops at 4 KiB and the failure presents as a rendezvous miss. **Refuse, don't evict** — eviction reproduces exactly the silent-never-meet shape this surface exists to avoid, while a refusal tells the offerer it failed and can be retried. Filling a bucket to deny service is a rate-limit concern (open item 2), not a bucket-semantics one. |
| 6 | What **TTL value** does a node default to? | **60 s.** Node-configurable, published in the `advertise` limits. | Bound by the lifetime of what the blob *describes*, not by node memory: a candidate list's `srflx` entry expires with the NAT binding that produced it (commonly 30–120 s), so a longer TTL only serves candidates that are already unpunchable. |

The common thread: **every one of these defaults to the choice that makes a retry safe**, because a rendezvous
that fails silently is the failure mode this whole surface is built to avoid.

Pins 4–6 were added 2026-07-28 from the Rust Stage-1 build, which is the §1.2 prediction arriving on schedule:
a single-impl server does not surface these, so they get answered by fiat in one language unless they are pinned
here first.

### 1.2 Roles & conformance `[departs from the RELAY precedent — deliberate]`

`EXTENSION-RELAY` §10.1 sets the ecosystem's pattern: *a conformant implementation MUST implement the service
role; a deployment MAY choose to enable it.* **This proposal departs from that**, and records the departure:

> **The `system/signaling` server role is OPTIONAL for a conformant implementation. The client role is the
> conformance surface.**

Rationale: RELAY is a **peer capability** — any peer may relay for a neighbor, so every impl needs it. The
connection node is **deployed infrastructure** with a deliberately non-entity hot path; nobody will run a Python
rendezvous box. Requiring three server implementations is work with no consumer. Concretely: **Rust implements the
server; go/rust/py all implement the client**, and the cross-impl conformance category exercises clients against
the one server.

**The cost, stated:** a single-impl server means **underspecification stays invisible** — one implementation
cannot disagree with itself, so nothing forces the §1.1 questions into the open. Two mitigations, both load-bearing:
(a) §1.1 pins them in advance; (b) the **clients are three independently-written impls** hitting the server from
outside, which is a real convergence signal on the semantics even without a second server. If a fourth question of
the §1.1 kind surfaces during the build, it comes back here as a spec fix — it does not get settled in the Rust.

### 1.3 State silhouette

Per-key TTL-reaped buckets and nothing else. No bulk storage, no cross-node coordination, no presence map, no
durable data. Losing a node drops in-flight handshakes — peers retry — and loses nothing that mattered. This is
what makes the node cheap enough to be a universal fallback and trivial to scale horizontally (rendezvous-hash
selection, no shared mutable state; adding capacity is "run more instances in the advertised pool").

### 1.4 `reflect` belongs to the listener, not to the core `[ruling 2026-07-28]`

The Rust Stage-1 build found that `reflect` **cannot be served on the wrapped surface**: a handler is dispatched an
entity operation, not a socket, and the observed source address never reaches it. The right fix is not to plumb it.

**The fact already has a canonical home, and it is not here.** `PROPOSAL-NETWORK-REACHABILITY-FACTS` §2.1 owns
"how a peer learns its observed address" and it is **landed spec** as of 2026-07-29: `EXTENSION-NETWORK` v1.5
**§6.7.1**, the operation `system/network:observe-address()`, gated by `system/capability/network-reflect`. Any
peer you can already reach yields the fact, **inside the capability model, with no node involved.** A wrapped
`system/signaling:reflect` would therefore be a **second mechanism for a fact that already has one home** — the
duplication the one-canonical-home rule exists to prevent, and the more expensive of the two besides: it needs a
*node*, while `observe-address` needs only a peer you were already dialing.

> **Updated 2026-07-29.** This paragraph previously described the v1 mechanism as *"an optional responder-filled
> `observed_address` field on the HELLO handshake (the v1 default), plus `observe-address()` for periodic
> re-checks."* **That is superseded.** The handshake field is a change to `system/protocol/connect/hello`, which
> is normatively defined **upstream** in `entity-core-protocol` and is not this repo's to ratify; it routes there
> as its own proposal and gates nothing here. The **operation is the v1 path**, and it is folded. The ruling in
> this section is unaffected either way — it never depended on *which* NETWORK mechanism won, only on the fact
> having a canonical home that is not this node.

`reflect` also reports a transport-level fact, which is the layer the entity wrapper exists to abstract away and
the layer the entity system is careful never to interpret (`IP:port` is opaque above the transport). And a peer
concludes its NAT type from *agreement across several reflectors*, so a plausible-but-wrong address is worse than
none — it gets concluded on rather than discarded.

> **Correction (2026-07-28).** The first statement of this ruling argued that a wrapped `reflect` "answers the
> wrong question" because it reports the **TCP/WS** mapping while the punch needs **UDP**. **That is wrong for
> v1:** §5 of the peer-side proposal sequences **TCP simultaneous-open first** (our-QUIC is declared-but-unbuilt),
> so the TCP mapping is exactly what the v1 punch needs. The ruling stands on the two arguments above — which are
> stronger — and the UDP point applies only to the later QUIC substrate. What the *node's* `reflect` is genuinely
> for is the **UDP** mapping for that later substrate and the **browser leg**, where `REACHABILITY-FACTS` §2.2
> already accepts standard STUN/TURN. Both are STUN-shaped, both belong on the unwrapped listener, neither is v1.

So it is **the unwrapped listener's verb** — the listener owns the socket and therefore owns the observation. §5.1
already names plain STUN as the obvious fit, which is this ruling arrived at from the other direction.

**What follows:**

- **The core is three verbs.** This makes §2.1's "identical across surfaces" *exactly* true rather than nearly
  true — `reflect` was the one verb that never could have been, because it was never an operation on the mailbox.
- **A wrapped-only node does not implement `reflect` at all.** A call is an ordinary unknown-operation error, not a
  placeholder awaiting plumbing. No `HandlerContext` field is added and no observed-address side channel is
  invented — both were on the table and both are now unnecessary.
- **NAT-type detection does *not* wait for the unwrapped surface.** It comes from NETWORK §6.7.1 + the
  agreement-across-reflectors rule (`REACHABILITY-FACTS` §2.3) — a path that does not involve this node at all.
  That path was a DRAFT and unbuilt in all three impls when this was written; it is **landed spec as of
  2026-07-29** (`EXTENSION-NETWORK` §6.7.1) and **still absent from all three trees** — buildable now, one ordinary
  capability-gated operation, no listener and no lifecycle contract. It is **not free**: the handler answering it
  needs the connection's source address, which is a narrow accept-side seam per impl (§6.7.1).
- **So Stage 1 loses less than first stated.** An earlier version of this ruling said NAT-type detection moved to
  Stage 2 with `reflect`; it moves to §6.7.1 instead, which is independently sequenceable and cheap.

## 2. The operations are the primitive; the entity wrapper is optional (operator ruling, 2026-07-28)

**The three core verbs are the core.** `offer` / `collect` / `advertise` are plain operations over a stateless
keyed mailbox — they do not need the entity system, and they are defined without reference to it (§1, §1.1). On
top of that core sit **two invocation surfaces**, and the extension supports both. (`reflect` sits beside the
core rather than in it — §1.4.)

| Surface | How the op is invoked | What it adds |
|---|---|---|
| **Wrapped — cross-peer `execute`** | an ordinary entity handler operation: connect, handshake, capability check, entity-encoded request and result | **this is where security comes from** — the capability grant is the admission control |
| **Unwrapped — plain public** | a plain client request straight to the service; **no entity machinery in the path at all** | nothing — admission is per-source and per-key **rate limits only** |

**Both reach the same operations, because they are the same operations.** The wrapper translates a cross-peer
`execute` into a call on the core; the public front end translates a plain request into a call on the same core.
Neither reimplements the verbs.

**Why the unwrapped surface exists** (operator): the introduction service is **public infrastructure by nature** —
anyone may use it, so a capability-grant handshake in that flow **buys nothing**. It costs a round trip, a grant
evaluation, and an entity encode/decode per request and returns no access control the rate limiter is not already
providing. Running public nodes is how the service meets demand. The wrapped surface stays first-class for the
deployment where admission genuinely matters — the private device mesh.

**Unwrapped means unwrapped all the way down.** A client talking to a public node does **not** run a handshake,
negotiate capabilities, or encode entities. That is the point of the surface, not a shortcut within it.

### 2.1 What is pinned

- **The core operations are wrapper-agnostic.** They are specified without the entity system (§1.1) and MUST behave
  identically however they are invoked. This is **structural, not a discipline to maintain** — one implementation
  of the verbs, two thin front ends over it. A divergence between surfaces means someone implemented the verbs
  twice, which is the defect, not the divergence. **The core is `offer`/`collect`/`advertise`**; `reflect` is a
  property of the unwrapped listener rather than an operation on the mailbox (§1.4), which is why it is the one
  verb this pin does not range over.
- **Which surfaces a node exposes is deployment configuration.** A node may present the wrapped surface, the
  unwrapped one, or both; that is a config choice about who the node is for, not a per-request negotiation.
- **The client knows which surface it is using** — from the node's advertisement
  (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`) or local configuration, never by probing. A client that silently
  retried gated traffic against a public endpoint would put a `secret`-mode key's hash somewhere anyone may poll.
- **The unwrapped surface has no capability gate — by design, not by omission.** There the rate limiter is the
  *entire* admission story. It is not a weakened version of the wrapped surface; it is the surface with no
  security layer, offered because on public infrastructure that layer earns nothing.

| Lens | Surface | Admission |
|---|---|---|
| **Private device mesh** — only the operator's own peers | wrapped | capability grant |
| **Shared public infrastructure** — anyone, the universal fallback | unwrapped | rate limits |

**The one rule that holds in both modes:** *both peers of a handshake must meet at the same provider.* They
rendezvous-hash the shared key into a shared pool. A private node works when both peers share it (named in the
contact/link that introduced them); a well-known public node is the common denominator that guarantees any two
strangers can always meet.

## 3. Implementation staging (operator ruling, 2026-07-28)

The node is **staged inside `entity-core-rust` as an isolated, optional, feature-gated extension** — an
`extensions/signaling` crate plus a thin `cmd/` binary — rather than starting life in its own repo. Extraction to
a sibling service repo (the 2026-07-24 shape) becomes an option once the surface stabilizes; it is an **additive
`git subtree` split**, never a history rewrite. Rationale (operator): the ecosystem has **little practice building
isolated optional handlers**, and this is a well-scoped, low-blast-radius place to build that muscle — get the
isolation boundary right first, then extraction is a mechanical move rather than a redesign.

**Two consequences that are decisions, not accidents:**

- **Build the core verbs first, then the wrapped surface, then the unwrapped one.** The core (§1.1) is plain code
  with no entity dependency. The **wrapped** surface over it is an ordinary handler and is **buildable today**;
  the **unwrapped** surface is a service-owning listener needing the lifecycle contract (§3.1) and the §5.1
  protocol spec. So the wrapped surface goes first because it is **unblocked**, not because the unwrapped one is
  lesser. v0's constraint is proving reflect, rendezvous, and the punch at a handful of handshakes per second on
  one box — the scale at which the entity overhead the unwrapped surface exists to avoid does not yet bite.
  *What keeps this honest:* the core is implemented **once**, so adding the second front end later is additive
  and cannot redefine the verbs. A v0 shortcut that pushes entity concerns down into the core breaks that and is
  out of bounds.
- **The isolation boundary is the deliverable, not just the feature.** Optional-by-default, off unless its feature
  is enabled, no imports from unrelated extensions, and no edits to shared core crates to accommodate it. If the
  node cannot be built as a clean optional extension, that is a finding worth having *before* a second repo makes
  the coupling invisible.

### 3.1 This is the first real test of the handler abstraction — and it needs a contract that doesn't exist yet

The operator's framing (2026-07-28): the handler abstraction exists so that *any* functionality can slot into a
peer, and this is one of the first genuine tests of it — **both** the server (which starts a non-capability-gated
service) and the clients (which use it to reach another peer) should be **ordinary handler extensions that slot in
without special-casing**.

A survey of `entity-core-rust` `ae93443` found that the abstraction **does not currently support the server half**:
the `Handler` trait and `register_handler` cover request/response dispatch only, while every existing
resource-owning extension (clock, subscription, inbox, history, local-files, network) gets its lifecycle from
`PeerInner::start_engines` — a hardcoded call site that constructs each engine **by name**. Building the node that
way would make it a fourth privileged built-in, which is precisely *not* the test.

**`PROPOSAL-SDK-HANDLER-OWNED-SERVICES` is that missing contract** and is a hard dependency for the node's
service-owning half. It binds all three impls, because the punch is service-owning in every language.

**The two surfaces (§2) sort the work by dependency, not by importance:**

| Stage | Content | Handler shape | Blocked? |
|---|---|---|---|
| **1** | core verbs + the **wrapped** surface (Rust server) + go/rust/py clients doing `offer`/`collect` | plain dispatch handlers both sides | **No — Rust server + client shipped 2026-07-28; go/py clients next** |
| **2** | the **unwrapped** surface (Rust server + a plain client per language) + **`reflect`** (§1.4) + the punch (all three impls) | service-owning | **No longer blocked as of 2026-07-29** — the lifecycle contract is folded (`SDK-OPERATIONS` v1.11 §11.6.9) and §5.1 is specified. Sequenced after Stage 1's cross-impl gate, not gated on arch. |

So Stage 1 ships while the contract is settled in parallel — the right order, since Stage 2 is the expensive half
that should not be redesigned mid-flight. **That is what happened:** Stage 1 shipped 2026-07-28 and the contract
landed 2026-07-29 (`SDK-OPERATIONS` v1.11 §11.6.9), with §5.1 specified the same day.

> **Ruling 2026-07-29 — do not bypass the contract, even though it turns out you can.** `entity-core-rust`
> reported that the node's binary could own a public listener directly (`#[tokio::main]`, the same spawn/abort
> shape as its TTL reaper), so the lifecycle contract was **never a functional blocker** — every prior document
> calling it "the expensive one" mischaracterized it. **The ruling is unchanged regardless:** a hardcoded
> built-in is precisely what this section says not to do, and the whole reason the node was staged in-repo was
> to test whether a handler can own a service *without* special-casing. A bypass produces a working listener and
> forfeits the only thing the staging was buying. **The finding is the deliverable; the listener is a side
> effect.**

**The unwrapped surface is what makes this a real abstraction test.** A handler whose job is to own a service the
entity system does not mediate at all is the strongest case for the "declared, not gated" boundary rule
(`PROPOSAL-SDK-HANDLER-OWNED-SERVICES` §4): the manifest must convey *"this handler runs an open service the
capability model does not cover."* If the declaration can't express that clearly enough for an operator to read
the exposure off the tree, the declaration shape is wrong — the finding worth having early.

It also states the abstraction's value precisely: **the handler supplies the entity wrapper and owns the
lifecycle; the service underneath needs no awareness of the entity system, and the entity system needs none of
it.** Anything that can be expressed as operations can be slotted in, with or without the wrapper.

## 4. Deployment (deliberately minimal)

One public instance on a small VM — the existing **DigitalOcean** box, or **Hetzner**; the node's footprint is
small enough that the choice is a capacity/cost matter, not an architectural one. Zero bulk storage, transient
~1 KB per handshake. Packaging rides the ecosystem's existing `make <verb>`-over-podman convention.

**Scoped out of this proposal (operator, 2026-07-28):** the full fleet / service-management arc. The 2026-07-24
direction packet §6 proposed standing the node's management plane on `system/device` + operational-state **from
the start**, so the first managed service would double as the first test of the fleet model. That is **no longer
in this feature's path** — what is needed here is discovery/interconnectivity plus enough deployment to get the
service running on a box. `system/device`-backed management, health views, placement, and multi-node orchestration
stay designed and tracked under **W-FLEET**, seeded by this node but **not gating it**. Likewise the W-SUPPLY
Lane-B `system/package` provenance demonstrand: still the intended first demonstrand, no longer a precondition.

## 5. Open items

| # | Item | Disposition |
|---|---|---|
| ~~1~~ | ~~**Specify the unwrapped surface's protocol**~~ | **CLOSED 2026-07-29 — specified in §5.1.1–§5.1.4.** Mailbox = length-prefixed CBOR over TCP; `reflect` = RFC 5389 STUN over UDP, unmodified. Written on `entity-core-rust`'s ask so three clients target one document. |
| 2 | Rate-limit shape for the public mode — per-source and per-key thresholds, plus the reflection/amplification hygiene a public reflector needs. **The rate limiter is the *entire* admission story here** (§2.1), so this is load-bearing, not hygiene | **needs a call before a public node is exposed**; not before Stage-1 first light |
| ~~3~~ | ~~Bucket TTL default~~ | **RESOLVED 2026-07-28** — **60 s**, node-configurable, published in `advertise`; advisory to peers, binding on the node (§1.1 pins 3 + 6) |
| ~~3a~~ | ~~Whether `reflect` is served on the same listener as `offer`/`collect`~~ | **RESOLVED 2026-07-28** — `reflect` is unwrapped-only (§1.4). Whether it shares a *port* with unwrapped `offer`/`collect` remains an impl detail |
| 4 | Extraction trigger — what specifically has to be true before the subtree split | revisit after the §4 gate of the direction packet passes |
| 5 | Who runs and secures *open* public infrastructure at scale (incentive, anti-Sybil) | **parked** — a deployment/product decision on the managed → federated → self-organizing spectrum, explicitly not a spec gate |

### 5.1 The unwrapped protocol — **SPECIFIED 2026-07-29** (open item 1 closed)

A public node is **not an entity peer**. It is a stateless service with its own small protocol, outside the entity
system entirely — no V7 framing, no handshake, no capability negotiation, nothing to bypass because none of it is
in the path. Clients hold no entity machinery to talk to it.

That was never the question; the question was what bytes. Written down here so **Rust, Go, and Python clients are
written against one document rather than against the Rust** — which is the whole reason it is arch's to answer
(`entity-core-rust`, 2026-07-29: "three client languages get written against it, so it can't be invented here").

#### 5.1.1 Two listeners, two protocols — and why that is not a compromise

| Surface | Transport | Protocol |
|---|---|---|
| `offer` / `collect` / `advertise` — **the mailbox** | **TCP** | length-prefixed CBOR (§5.1.2) — ours |
| `reflect` — **the STUN role** | **UDP** | **RFC 5389 STUN Binding, unmodified** (§5.1.3) |

The instinct is to make one protocol carry both. **It cannot, and the reason is the consumer, not taste.**

`reflect`'s consumers are (a) the later UDP/QUIC substrate and (b) the **browser leg**, whose ICE agent gathers
`srflx` candidates by speaking real STUN over UDP and **cannot be taught anything else**
(`PROPOSAL-NETWORK-REACHABILITY-FACTS` §2.2, now `EXTENSION-NETWORK` §6.7.1). A STUN-shaped lookalike of our own
serves neither: it is useless to a browser by construction, and it is a worse STUN than STUN. So `reflect` is
**plain STUN or it is pointless** — §5.1's original "obvious candidate" was right, and §1.4's ruling arrives at
the same place from the other direction.

The mailbox has the opposite constraint: no external standard fits an opaque keyed bucket, and it must be
implementable in three languages **without adding a dependency to any of them**.

Whether the two share a port is an impl detail (§5, item 3a) — they cannot share a *transport*, so in practice
they are two sockets.

**`reflect` is not on the v1 path.** `EXTENSION-NETWORK` §6.7.1 `observe-address` is the v1 reflection mechanism
and needs no node at all. §5.1.3 exists so the node can serve the QUIC substrate and the browser leg later, and
so nobody builds a private reflect in the meantime.

#### 5.1.2 The mailbox protocol (TCP)

**Framing.** Each message is a **4-byte big-endian unsigned length** followed by exactly that many bytes of
**CBOR**. Requests and responses use the same framing.

- A node **MUST** reject a declared length above **1 MiB** by closing the connection without a response. This is
  the anti-DoS floor and is deliberately far above the semantic limits below, which are the real contract.
- A connection is **strictly alternating request → response**. **No pipelining.** A client MAY reuse a connection
  for further requests; a node MAY close an idle one at any time, and a client MUST treat that as ordinary and
  reconnect rather than as an error.

**Why CBOR and not HTTP or JSON.** All three impls already carry a CBOR codec for ECF, so this costs **zero new
dependencies** — while HTTP costs Rust a client crate (no stdlib HTTP), against an ecosystem convention of stock
tools and minimal dependencies. HTTP would also drag in header parsing, chunked encoding, status-code mapping and
keep-alive semantics: a large ambiguity surface for three independently-written clients, for a **three-verb**
protocol. CBOR here is an *encoder*, not entity machinery — the client still holds no entity code, which is §5.1's
actual requirement.

*This does mean a browser cannot speak the unwrapped mailbox (no raw TCP). That is correct and not a gap:* a
browser peer uses the **wrapped** surface over WebSocket, and §2.1 guarantees both surfaces complete the same
verbs.

**Requests.** `op` is the operation name (kebab); data keys are snake (`STYLE-NAMING-CONVENTIONS`).

```
{ op: "offer",     key: bstr(33), message: bstr }
{ op: "collect",   key: bstr(33) }
{ op: "advertise" }
```

**Responses.**

```
{ ok: true }                                        ; offer
{ ok: true, messages: [bstr, ...] }                 ; collect — deposit order, oldest first (§1.1 pin 4)
{ ok: true, endpoint: tstr, limits: { ... } }       ; advertise
{ ok: false, error: tstr }                          ; any — closed enum below
```

`advertise`'s `limits` publishes the §1.1 pins so a client **reads them rather than assuming them** — the
cross-impl reject boundary pin 5 exists to prevent:

```
limits: {
  max_blob_bytes:   uint,   ; default 8192   (§1.1 pin 5)
  max_bucket_blobs: uint,   ; default 32     (§1.1 pin 5)
  ttl_seconds:      uint,   ; default 60     (§1.1 pin 6) — advisory to peers, binding on the node
  lobby_constant?:  bstr    ; OPTIONAL; present only if the node overrides the default (§1)
}
```

**The `key` is exactly 33 bytes** — matching the `system/hash` wire width, and **treated as opaque and compared
byte-wise** (§1). A key of any other length is `bad_request`. The node derives nothing and knows nothing about
the four key modes (§1, mode-blindness).

**Error codes (closed enum, snake per the naming convention).**

| Code | Meaning |
|---|---|
| `message_too_large` | blob exceeds `max_blob_bytes` — **refused, never truncated** (§1.1 pin 5) |
| `bucket_full` | bucket at `max_bucket_blobs` — **refused, never evicted** (§1.1 pin 5) |
| `bad_request` | malformed frame or CBOR, unknown `op`, wrong key length, missing field |
| `rate_limited` | admission control refused it (§5 open item 2 sets the thresholds; the **code** is pinned now so clients handle it from the first build) |

A client receiving an unrecognized code **MUST** treat the request as failed and **MUST NOT** retry it as if it
had succeeded. Nodes **MUST NOT** invent codes outside this set.

**Behavior is §1.1 and is not restated here.** `collect` is non-destructive and returns an **empty list, not an
error**, at an absent key; `offer` appends and dedups by content hash; TTL is the only reaper. Those pins are the
contract; this section is the encoding of them.

#### 5.1.3 `reflect` — RFC 5389 STUN, unmodified (UDP)

The node answers a **STUN Binding Request** with a **Binding Success Response carrying `XOR-MAPPED-ADDRESS`**.
That is the entire specification; RFC 5389 is the document, and this proposal deliberately adds nothing to it.

- The node **MUST NOT** require authentication (no short-term/long-term credential mechanism). A public reflector
  that demands credentials is unusable by a browser ICE agent, which is one of the two consumers.
- The node **MAY** additionally emit the legacy `MAPPED-ADDRESS`; clients **MUST** read `XOR-MAPPED-ADDRESS`.
- A peer **MUST** consult **several** reflectors and require agreement before concluding a NAT type
  (`EXTENSION-NETWORK` §6.7.1) — a single reflector is advisory, never trusted.
- **`EXTENSION-NETWORK` §6.7.3's socket rule applies unchanged:** a `srflx` candidate derived from this reflector
  is the mapping of **the socket that sent the Binding Request**, and the peer MUST punch from that socket.

**Anyone's STUN server works, and that is the point.** A deployment MAY point peers at coturn or any public STUN
server instead of running `reflect` at all; nothing in the punch path knows the difference. The node offers it as
a convenience for operators who would rather run one box than two.

#### 5.1.4 What is deliberately absent — and the one honest cost

**No TLS on the mailbox.** Blob confidentiality and authenticity are already **peer-side**: blobs are opaque to
the node, and `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §3.1/§3.3 pin their framing and signature carriage. The
node is untrusted **by design** — it is a dumb keyed mailbox — so terminating TLS at it would protect the one leg
that already needs no protection, while adding certificate lifecycle to a service whose scaling story is "run more
instances."

> **The cost, stated rather than skipped:** a passive on-path observer sees the **33-byte rendezvous key** in
> cleartext. Blob contents stay opaque, but in `pair` mode the key is derived from both peer identities
> (`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.2), so an observer who can guess or enumerate a candidate pair
> can confirm **that those two peers are trying to meet**. That is a real linkability leak and it is metadata, not
> content. It is accepted for v1 — the punch itself is observable on the wire anyway — and it is **the reason
> `secret` and `tag` modes exist**, since their keys are not derivable from identities. Logged as an open item.

**No authentication and no capability check** — by construction (§2, §4 of `PROPOSAL-SDK-HANDLER-OWNED-SERVICES`).
**The rate limiter is the entire admission story** (§2.1), which is exactly why §5 open item 2 is load-bearing and
must be answered before a public node is exposed. `rate_limited` is pinned above so clients are built to handle it
before the thresholds exist.

*This is also the clean statement of what the handler abstraction is for:* the handler owns the service's
lifecycle and nothing else. The service itself needs no awareness of the entity system, and the entity system
needs none of it.

## 6. Validation

Per the CDN-corridor meta-rule, none of this is real until exercised live. The gate, in order **(revised
2026-07-28: `reflect` moves to Stage 2 per §1.4, and step 2 gains a second node)**:

| # | Step | Stage |
|---|---|---|
| 1 | **Rendezvous in each mode** — two peers meet and exchange candidates at a `pair`, `tag`, `secret`, and `lobby` key. One code path, four derived keys. | **1** |
| 2 | **Pool selection converges** — the pool holds **two** node instances; every impl independently selects the same one for the same key. | **1** |
| 3 | **Reflect** — a peer learns its public address from the deployed node's unwrapped listener. | 2 |
| 4 | **Punch** — a real `fire_at` simultaneous-open produces a direct transport between two NAT'd peers, held by keepalive. | 2 |
| 5 | **A message rides it** — a plain-text chat round-trip over the punched link. | 2 |

Steps may run **Rust-peer ↔ Rust-node** for first light; the **cross-impl run (go/rust/py peers against the
deployed pool) is the ratify gate**, and it is what actually validates the §2.2 key derivation across impls. Prose
review does not catch rendezvous-key or punch-timing bugs.

**Why step 2 needs two nodes, and why it is nearly free.** §4 deploys one instance, and `argmax` over a one-member
pool returns that member *whatever the weight function computes* — so a green single-node gate says nothing about
`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3.1, and three clients converge trivially no matter what each
implements. That makes the §1.2 mitigation — "three independently-written clients are a real convergence signal" —
**not reach the selection rule**, which is the §2.2 failure shape one layer out and just as silent. A second
instance is a second port on the same box (the node is stateless by §1.3), so this converts an unvalidatable pin
into a validated one for the cost of a config line.

## Pointers

- **Peer-side half:** `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` (§2.2 key modes, §3 coordination messages, §5 punch).
- **Handler contract (Stage-2 dependency):** `PROPOSAL-SDK-HANDLER-OWNED-SERVICES`.
- **Reflect semantics:** `PROPOSAL-NETWORK-REACHABILITY-FACTS`.
- **Provider selection:** `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3.1.
- **Design context:** `docs/research/explorations/EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY.md` (§3a, §4, §10).
- **Ruling + sequence:** `HANDOFF-2026-07-28-connection-node-staging-and-sequence.md`.
