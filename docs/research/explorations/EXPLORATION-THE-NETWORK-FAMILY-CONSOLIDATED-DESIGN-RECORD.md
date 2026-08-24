# Exploration — the network family: the consolidated design record

**What this is.** The substance of the p2p / information-travel design work, carried into this corpus
so it survives without the legacy file tree. Roughly twenty legacy documents produced this material
across the `cgid-10-215` → `cgid-10-231` arc; **they are not copied here** — the source tree is
another team's, read-only, and duplicating it would give us two homes for every fact. What is here is
what they *established*, what is settled, what is open, and where to look when a detail is needed.

**Why it exists.** The network family's design record did not cross the V8 repo split. Every one of
these documents was unreachable from this corpus, and the cost was measured: three published rulings
on RELAY's mode set written without opening the study that produced it, a boundary map rediscovered
after a second one was started, and a deferral list four-for-four void on landed text.

**Legacy root**, for every path below (read-only; do not edit, do not back-sync):
the internal legacy corpus (read-only)

**Companion:** `EXPLORATION-THE-THREE-CONSTRAINT-REGIMES.md` — the regime column below is its.
**Canonical map:** `guides/GUIDE-NETWORKING-MODEL.md` owns the six-concern boundary map and the
deployment topologies. This document is the *design record behind it*, not a second map.

---

## §1 The three primitives — the boundary, at source

**`explorations/EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP-cgid-10-229.md`** — the document
that drew the line the operator later drew again independently.

| Primitive | Addressing | Question | Home |
|---|---|---|---|
| **Unicast relay** | addressed (one destination) | "carry *this* to *D*" | `EXTENSION-RELAY` |
| **Routing** | the *decision*, not the message | "to reach D, who's next?" | `EXTENSION-ROUTE` |
| **Gossip / scatter** | **unaddressed** | "spread this to whoever cares" | `GOSSIP` — roadmap |

Three separations it establishes, each of which has since been re-derived at cost:

- **Relay is not routing.** Relay *forwards* one hop; routing *decides* the hop. SMTP does not route;
  IP/BGP underneath it does. The next hop comes from **three sources in precedence order** — a source
  route in the envelope, a pluggable resolver, trivial direct-reachability — and *"the algorithm is
  never relay's."*
- **Gossip is not relay.** Addressed unicast vs unaddressed spread: different delivery semantics,
  different loop control, different dedup. **A gossip message has no `destination`.**
- **Routing serves both**, with different policies.

**Loop control is three different mechanisms and they are routinely confused:** `ttl_hops` bounds
relay's transport path (so a pathological source route needs no separate loop state);
`bounds.await_stack` detects request/response cycles *inside* the opaque inner envelope, invisible to
the relay; and gossip needs a **seen-set / message-id dedup**, because epidemic spread has no single
path to TTL-bound. *That third one is a reason gossip is a separate primitive, not a relay mode.*

### §1.1 The history — this is recovered ground, not new ground

| Revision | What existed |
|---|---|
| **V0.5** | QUERY forwarding with `ttl`, `hop_count`, `seen_peers`, `expand` — **gossip-shaped discovery** bolted onto QUERY |
| **V1.0** | Forwarding + a `system/routes` entity: `{peer_pattern, via, address}` with a default route. **The routing table as an entity already existed.** Three transit patterns named — Relay / Tunnel / Circuit — the ancestors of Mode F / ENCRYPTION / Mode C |
| **V2.0** | A clean four-way Layer-5 split: **`system/routes` · `system/relay` · `system/gossip` · `system/replicate`**. `system/relay/v1/forward` carried `{destination, payload, ttl, path}` — **`path` is source routing; we had it.** `system/gossip` was a real design: digest-based anti-entropy, explicitly "epidemic broadcast" |
| **V2.0** | `bounds {ttl, budget, chain_id, await_stack}` — constant-size cycle detection, the generic substrate under all propagation |
| **V7** | Relay survived and matured, but **lost `path`** (collapsed to single `next_hop`) — the multi-hop gap. `bounds` survived. **`system/routes` and `system/gossip` did not.** |

**`EXTENSION-ROUTE` v1.0 is now Active — `routes` is recovered. `GOSSIP` is the one still
outstanding**, and its absence is called *"a real gap for the federation/mesh vision."*

## §2 Relay — the four modes, and what they actually are

**`explorations/EXPLORATION-RELAY-AND-AGGREGATOR-PATTERN-cgid-10-217.md`** — the study `RELAY` §1
cites. Surveys Nostr · AT Protocol · ActivityPub/Mastodon · libp2p circuit-relay-v2 · SMTP/NNTP · IPFS.

**It extracts five orthogonal dimensions, not four modes:** push/pull · persistent/transient ·
single-author/multi-author · routed/broadcast · transparent/stateful. *"A relay configuration picks
values on each axis. **Different canonical modes are different points in this 5D space.**"* The four
modes are named clusters, **not a partition of a category** — which is why pruning one is the
expensive direction of a wrong guess (**L11**).

| Mode | Shape | Real systems |
|---|---|---|
| **F — Forward** | push, transient, routed | ActivityPub push, SMTP forward |
| **S — Store-and-poll** | put-then-poll, persistent, namespace-addressed | static CDN, IMAP, NNTP read-side |
| **A — Aggregate** | pull/stream, persistent, **multi-author**, may be stateful | Nostr relays, ATProto Relay/firehose, Mastodon relay servers |
| **C — Circuit** | bidirectional, transient, routed, transparent | libp2p circuit-relay-v2, TURN |

**No surveyed system is single-mode** — ATProto S+A · Mastodon F+S · Nostr A+S · SMTP F+S. And §9.3
names the unification as the contribution: *"**no surveyed system has this unification** — they have
separate codebases per mode. We have one extension; modes are config."*

**§7.7 pins what v1 deliberately does not specify**, under
`[[feedback_design_space_not_authority]]`: routing algorithms (F), **aggregation semantics (A)**, NAT
mechanism (C), rate-limiting. *All four are called knobs.*

### §2.1 The two load-bearing rulings — do not disturb

**`reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md`** — **static IS relay.** A static store is a relay
intermediary that is *passive* instead of active: sender PUTs, receiver polls. *"The protocol layer —
envelope, signature, capability chain, ingest — is **byte-identical**."* It dissolved the async-
interaction gap: store-and-forward has **no session**, so the liveness/nonce handshake *does not
apply* — each envelope is self-authenticating by signature. Three real residual pieces:
**rendezvous addressing · the signed mutable pointer · poll cadence.** *(The signed mutable pointer
has since landed as `EXTENSION-TREE` §3.3a `system/peer/published-root`.)*

**`reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md`** — **relay is transport, not authority.** Two
orthogonal permission relationships: Alice↔Charlie (the actual authority, rooted at Charlie) and
Alice↔Bob (the right to *use* Bob as a relay, rooted at Bob). *"Neither check consults who relayed
this."* Bob constructs no link and cannot escalate; the worst he can do is drop or delay — **an
availability problem, not an integrity one.** RELAY is a **non-site** in V7 §5.8's chain-construction
registry. *The one case that would touch the chain — Bob vouching for Alice — is **delegation, not
relay**, and already has a home.*

## §3 Delivery — the unified model

**`explorations/EXPLORATION-NETWORK-EXTENSION-LANDSCAPE-AND-DELIVERY-MODEL-cgid-10-220.md`** and its
capstone **`-NETWORKING-SUBSYSTEM-COMPREHENSIVE-LANDSCAPE-cgid-10-220.md`**.

**The core claim:** delivery is *placing an entity at its destination address*. The destination
address never changes; what varies is **who holds the authoritative copy at rest** and **who
initiates the transfer**. This rides V7 §1.4's authority split — **cache authority** (any peer may
write `/{D}/…` in its own local view) vs **keyholder authority** (only D's key is canonical for what
is *true* there).

| Arrangement | Where it lands | Who transfers | Needs S→D reachable? |
|---|---|---|---|
| Push-to-inbox | D's inbox | S dials D | **yes** |
| Queue + push-drain (`NETWORK` §8 today) | S's `system/outbound/{D}/…` | S re-pushes on reconnect | yes, eventually |
| **Queue + pull-drain** | same queue, **readable** | **D drains via TREE_GET** | **no** |
| **Stage-in-view** | `/{D}/…` **in S's own view of D** | D pulls its own namespace from S, verifies, applies | **no** |
| Static publish | S's published tree | D fetches | no |

**Two results worth keeping:**

1. **Poll is not degraded — it is the transport-independent form of delivery.** Push is the
   low-latency end of a synchrony spectrum; pull-publish is strictly better on reachability (NAT,
   firewalls, browser sandboxes, offline intervals, untrusted relays) and on *not transferring
   information until it is wanted*. Hence `delivery_mode: push | poll` as a **first-class choice, not
   a failure fallback**.
2. **Two one-way publish surfaces compose to full duplex.** If A maintains an outbox for B and B one
   for A, and each polls the other, you get bidirectional communication **with no live connection at
   all** — and with encryption the surface sits safely on an untrusted relay. *This is the operator's
   own insight, recorded, and it is why the addressed tier is complete without gossip or a CDN.*

**The reachability-class taxonomy** — the organizing frame, and the dispatch decision table:

| Class | Accepts inbound? | Delivery | Example |
|---|---|---|---|
| Full-duplex listener | yes | push over the live socket | TCP/WS/WebRTC server |
| Half-duplex listener | yes (separate flow) | push to its HTTP listener | HTTP server |
| Held-connection client | no, holds one it opened | push down its own socket | browser over WebSocket |
| **Pollable client** | no | **pull-drain** | browser over HTTP, CLI, NAT'd peer |
| Static publisher | not live at all | consumer fetches | CDN-hosted peer |

A peer may occupy several at once; reachability is **per-peer and asymmetric**.

### §3.1 The nine-layer subsystem map

L0 wire framing (V7 §1.6) · L1 multiplexing/reentry (V7 §6.11 — **mature, the Class-G linchpin**) ·
L2 handshake/auth · L3 session (`NETWORK` §6.1–6.4) · L4 transport profiles (§6.5) · L5 dispatch
(§10) · L6 delivery (INBOX/SUBSCRIPTION/CONTINUATION + §8) · L7 peer lifecycle · L8 discovery.

**The headline: L0–L7 contain every primitive the story needs; the gaps are composition, not
invention.** *"This is the CDN-corridor lesson one level up: each layer passed prose review; the
integration is where it bites."*

## §4 What is open — carried forward, with regime

| # | Open item | Regime | State |
|---|---|---|---|
| 1 | **Cap-direction asymmetry / per-peer-session cap** (R3b) | II | **The load-bearing structural hinge.** Bidirectional dispatch needs two caps from two granters; a reused inbound socket carries the wrong-direction one. Holding the cap on the **per-peer session** decouples transport from auth and resolves reuse + multi-transport together. *Caveat: `NETWORK` §6.1 is per-peer for **serial reconnect** and silent on **concurrent** multi-transport — the per-peer claim there is an extrapolation and must be named as one.* |
| 2 | **`NETWORK` §10 dispatch is stale** | II | The integration point. Predates §6.5; uses the old flat `resolve_address`; no `delivery_mode` branch; does not say whether an inbound connection counts as active. **Rewriting it around the reachability-class table is the single highest-leverage structural fix.** |
| 3 | **Mode A aggregation ordering** | II vs II | **A genuine conflict between two of our own rules.** §7.7 pins semantics as a per-deployment knob; *pin-the-cross-impl-observable-surface* says a MAY diverging across a peer boundary is a latent interop bug — and the order a consumer sees a union of N sources in is exactly that. **Operator call.** Recommended: a determinism floor, merge policy free. |
| 4 | **Aggregate-artifact attestation** | II | Membership set, ordering and merge have **no upstream signature**. `published-root` (`EXTENSION-TREE` §3.3a) is the anchor — consumer walks from the signed root, never trusting paths the host claims outside the chain. Spec text unwritten. |
| 5 | **Chunk-peer discovery** | II | `CONTENT`'s blob manifest and transfer are specified; **which peers hold which chunks** is not. The one open piece of the torrent-like tier. |
| 6 | **`GOSSIP` unauthored** | **I–II** | Shape known: anti-entropy digest exchange (Merkle-set reconciliation), topic/subtree-scoped, seen-set dedup. **Its structure is a proof-carrying CS result — draft it from the literature, not from preference.** |
| 7 | **NAT traversal / WAN reachability** | **III** | Circuit mode's deferral condition *"until a driver materializes"* **has fired** — browser-rust is the driver: listener-less, rendezvous-only, symmetric NAT untested, `turn:` refused for want of a credential channel. |
| 8 | **Continuation-driven lifecycle: validated or aspirational?** | II | `NETWORK` §4.1 builds reconnect continuations; §52 blesses imperative implementations; **no impl is known to drive lifecycle through them.** Audit → validate or mark informative. |
| 9 | **WASM durable-storage floor** | **III** | Cross-peer signatures at invariant-pointer paths, revocation index cost, identity, cap-across-reload. **What is the minimum a cross-peer-participating browser peer must persist?** Unanswered. |
| 10 | **Message-transport framing** | **III** | V7 §1.6 defines length-prefix for TCP/WS; `MessagePort`/`postMessage` already carry boundaries. Needs a WASM-binding note: keep the prefix for symmetry, or waive it. |

**Items 7, 9 and 10 are Regime III** — they are not ours to simplify, only to place. Items 1–6 and 8
are Regime II and are genuinely open design work.

## §5 Where to look, by question

| Question | Legacy path (under the root above) |
|---|---|
| Why are relay / routing / gossip separate? | `explorations/EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP-cgid-10-229.md` |
| Where did the four modes come from? | `explorations/EXPLORATION-RELAY-AND-AGGREGATOR-PATTERN-cgid-10-217.md` |
| Is a static CDN really a relay? | `reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md` |
| Does relay touch the capability chain? | `reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md` |
| What is the delivery model? | `explorations/EXPLORATION-NETWORK-EXTENSION-LANDSCAPE-AND-DELIVERY-MODEL-cgid-10-220.md` |
| The whole subsystem, nine layers + gap catalog | `explorations/EXPLORATION-NETWORKING-SUBSYSTEM-COMPREHENSIVE-LANDSCAPE-cgid-10-220.md` |
| Why may a relay not decode the inner envelope? | `explorations/EXPLORATION-RELAY-RECEIVE-SIDE-OPACITY-AND-CROSS-PROTOCOL-cgid-10-229.md` |
| Routing paradigms (source / DV / link-state / DHT) | `explorations/EXPLORATION-DISTRIBUTED-ROUTE-GRAPH-AND-ROUTING-PARADIGMS-cgid-10-229.md` |
| NAT / WAN reachability | `explorations/EXPLORATION-NAT-TRAVERSAL-AND-WAN-REACHABILITY-cgid-10-229.md` + `proposals/PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` |
| Gossip's intended shape | `proposals/PROPOSAL-EXTENSION-GOSSIP.md` *(STUB / EXPLORATORY)* |
| Registry / discovery / naming | `reviews/PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE.md` · `explorations/EXPLORATION-REGISTRY-CRUX-SYNTHESIS-cgid-10-217.md` |
| The SMTP analogy, worked | `explorations/EXPLORATION-SMTP-MODEL-AND-GENERIC-MESSAGE-PASSING-cgid-10-228.md` |
| Transport family / bridge shape | `explorations/EXPLORATION-TRANSPORT-FAMILY-AND-BRIDGE-SHAPE-cgid-10-215.md` |
| **The identity / auth / coordination stack** | `reviews/PLAN-EXTENSION-LANDSCAPE.md` — **not the network family; its §8 excludes it by name** |

> **What we track here, and why — a spec's citation is NOT the signal.**
>
> **We carry a legacy document forward when it is load-bearing for work under active refinement** —
> when an open question is being argued and losing the analysis behind it would mean re-deriving it.
> That is the whole test. It is a judgment about *what is live*, not a mechanical rule.
>
> **A landed spec citing a support document is a defect, not a demand signal**, and reading it as one
> gets the cleanup exactly backwards. `SPECIFICATION-FORMAT.md` §11.3 already says a normative spec
> references three kinds of target and *"process and team artifacts — review notes, session handoffs,
> arch-team update memos, status docs — MUST NOT be cited from normative text."* `AGENTS.md` L5 says
> the same thing plainly: **spec text is not our log.** And §13.6 of `SYSTEM-ARCHITECTURE.md` already
> ruled the disposition — *"the architectural rationale is preserved, but not in the same surface as
> the published specs and guides … rationale lives in the archive."*
>
> **So most of the citations that reach into this material are cleanup targets.** `spec address`
> reports six of them as `leak` today, from `EXTENSION-RELAY`, `-REGISTRY`, `-TREE`, `-SIGNALING`,
> `-SUBSCRIPTION` and `-ENCRYPTION`. The fix is to move the rationale into the proposal and drop the
> citation — **not** to import the target so the reference resolves. Importing to satisfy a citation
> would grow the corpus in proportion to our own worst habit.
>
> **The synthesis is the tracker; the legacy tree is the archive.** Copying it wholesale gives every
> fact two homes and re-creates the drift the split already cost us once.

## §6 What this does NOT claim

- **Not a substitute for the sources.** It is a synthesis; for a normative detail, open the path.
- **Not current build state.** Every implementation claim here is *design-time* and none has been
  re-pinned. `AGENTS.md`'s build-state rule applies in full: re-read the live worktree before
  asserting what any impl has.
- **Not a ruling.** §4's ten items are carried forward as open, including the three where a lean is
  recorded.
- **Not complete on the identity/addressing arc.** The peer-id and universal-addressing work
  (`cgid-10-222/223`) is adjacent and deliberately out of scope here.
