# GUIDE: Peer Compositions and Topologies

**Status**: Active

**Audience:** peer operators and application developers wiring more than one peer together; reviewers comparing the system to actor frameworks, sidecar patterns, federation models.

**What this guide is — and is not.** This is the **composition** pattern guide — peers + capability grants + coupling deployed together to produce a property no single peer can produce alone. It is **not** the cross-peer-messaging guide (delivery/durability mechanics live in `GUIDE-CROSS-PEER-MESSAGING.md` — a single pattern *within* the framework here). It is not the per-peer setup guide (extension choice and storage for one peer at a time lives elsewhere). The vocabulary is grounded in `EXPLORATION-PEER-COMPOSITIONS.md` v2 §12; the catalog in v2 §13; the no-reactive-cycle rule in v2 §10 and `PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md` §6.

---

## 1. The frame: a peer is a configuration, a composition is a family

A **peer** is not a role. It is a *configuration*: cryptographic identity (V7 §1.5) + installed handlers + installed extensions + capability grants held and issued + tree state + active continuations + active subscriptions + operational state (V7 §3.13). Two peers identical in identity and extensions still differ if any other facet differs. The peer is the whole configuration. (`EXPLORATION-PEER-COMPOSITIONS.md` v2 §11.)

A **composition** is a family of configurations sharing enough character to discuss as a class: peers wired together via capability grants and coupling (subscriptions, continuations, direct EXECUTE) that yield a property no single peer can yield alone. Compositions are **not** hierarchical — there is no "parent peer." Peers are peers; they differentiate by *role* and *configuration*, not by structural relationship. (`PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md` §5.1.)

A composition's **topology** is its network-shape aspect — who couples to whom. A composition *has* a topology; the two are not synonyms. (Exploration §12.)

A composition's **mode** is a deployment property, not a composition property:

- **Surface / endosymbiotic (internal).** External clients see one identity; supporting peers are invisible infrastructure. Eukaryotic-cell analogy: one organism outside, a cooperative composition inside. (workbench-go is the existence proof — one `AppPeer` you inhabit; the operational peer inside is not externally visible.)
- **Flat.** All member peers are equally externally-visible (federation, mesh, public hub-and-spoke). The configuration is the same; only the visibility differs.

The same composition can deploy in either mode.

> **Terminology guard.** "Role" is taken — `EXTENSION-ROLE` is RBAC (`system/role/*`). "Operational peer" / "observer peer" are *configurations*, not entries in the role extension. A configuration *may* include role assignments as one facet.

## 2. The core property the catalog rests on (K1)

The catalog only makes sense because the substrate has K1.

**K1 — Layer-1 liveness:** *No single component can indefinitely stall a peer. The emit/delivery pathway is non-blocking-or-error: saturation surfaces as an error to the caller, never an indefinite block, and never a silent drop.* (`EXPLORATION-PEER-COMPOSITIONS.md` v2 §4.)

K1 is what makes **Class I (intra-peer backpressure stall)** impossible. Every Class I row in the catalog (queue/outbox saturation) is prevented by K1 holding, not by any topology choice. **Class II (reactive-cycle stall)** is a different prevention class — handled by the no-reactive-cycle rule (§4). (Exploration §9, "Two-layer deadlock prevention.")

**Today, K1 is not yet in normative spec text.** It is two-team-converged across this repo's exploration and entity-workbench-go's peer-compositions-liveness review (§3.2/§3.3/§8). Its three scope facets — (a) termination boundary, (b) coverage boundary, (c) shed-vs-durability boundary — are drafted in `EXPLORATION-PEER-COMPOSITIONS.md` v2 §6.1 and gated on a one-pass joint sign-off (proposal §11 Q5). When ratified, K1 lands at SYSTEM-COMPOSITION emit pathway or V7 §6.10 (proposal §10). The catalog assumes implementations behave consistent with K1 in practice today; the catalog's correctness becomes audit-checkable once K1 lands normative.

Two other already-specified invariants underpin the catalog and are mentioned here so they don't fall out of view:

- **Per-binding atomicity** (V7 §6.10) — the emit binding update is the atomic state crossing.
- **Bounded cascade** — hard ceiling 32 (SYSTEM-COMPOSITION §3.2).

## 3. The two classes of stall, and the two distinct prevention mechanisms

Composition stalls divide into two structurally distinct classes. **The mechanisms that prevent them are distinct and not interchangeable.**

- **Class I — intra-peer backpressure stall.** A producer inside one peer blocks on a queue/outbox whose consumer is absent or persistently slow. *No cycle.* This is the common case; this is what the standalone-peer incident was. Prevented *only* by K1.
- **Class II — reactive-cycle stall.** Peer A's reactive paths depend on peer B's state changes; peer B's reactive paths depend on peer A's. The loop closes; the system spins or stalls. Prevented by the no-reactive-cycle rule (§4).

Reading the catalog (§5): every "liveness-sensitive coupling → class" entry is a Class I prevented by K1. The no-reactive-cycle rule is necessary for Class II compositions but is *invisible* to Class I and therefore does not prevent the common case on its own. **If the team reads §4 as "the cycle rule is the deadlock answer," K1 is under-invested and the common case is left unprevented.**

## 4. The no-reactive-cycle coupling rule (Class II)

When you compose peers and want to prevent reactive cycles between them, this rule applies. It is a design discipline for the *coupling pattern* — a verifiable property of how subscriptions and continuations are wired — **not** a capability restriction or runtime check. Capability grants establish what's authorized; the no-cycle rule is the design pattern operators apply for cycle-sensitive compositions. (Proposal §6.)

**The rule:** *In a composition where you want to prevent reactive feedback from peer B back into peer A, peer A's reactive paths (subscriptions, continuations, compute reactivity) must not depend on peer B's state changes.*

What this allows and forbids, in an A → B composition where A should not reactively depend on B:

| Direction | Allowed? | Why |
|---|---|---|
| A **reads** from B (sync RPC) | Allowed | Reads don't trigger A's reactive cascade. |
| A **subscribes** to B's state | **Avoid** in cycle-sensitive composition | Closes the cycle the composition exists to break. |
| B **reads** A's tree | Allowed | One-shot; doesn't subscribe back. |
| B **subscribes** to A's outbox-write events | Allowed | The delivery loop's whole purpose. |
| B **writes** to A's tree (e.g., delivery ack) | Allowed | One-shot write; A decides whether its reactive paths depend on it. |
| B **runs continuations** | Allowed | As long as the chain terminates without re-entering A's reactive paths. |
| B **has extensions installed** | Allowed | Each extension's reactive surface should be audited for cycle-closure. |

### 4.1 Strictness levels

- **Strictest (trivial audit).** The secondary peer runs **core protocol only** — no extensions, no reactive paths, no possible cycle. The strictest form of the operational peer composition.
- **Typical (small audit).** The secondary peer installs `revision` (versions outbox auditably; monotonic, no cross-peer reactivity), `history` (records only; no cross-peer side effects), `identity` (verifiable peer identity; no cross-peer side effects). Three extensions, rule holds, audit is straightforward.
- **Less strict (full audit per extension).** Any other extensions added with explicit audit that their reactive surface respects the rule. Composition-specific and operator-owned.

### 4.2 Where the rule lives

This is operator/implementation guidance, **not normative spec text**. Compositions multiply over time; each may have its own coupling pattern. The rule is verifiable by analyzing the installed subscriptions / continuations / compute graphs in tree state, not by runtime enforcement; the capability layer remains the runtime security boundary. (Proposal §6.3.)

## 5. The catalog — seven named compositions

The seven compositions below are configurations of the existing primitives. **None requires protocol changes.** Each is named so operators and impl teams converge on vocabulary; each is grounded in the workload that motivated it.

The catalog index (`EXPLORATION-PEER-COMPOSITIONS.md` v2 §13):

| # | Composition | Gives you | Coupling | Typical mode | Liveness-sensitive coupling → class |
|---|---|---|---|---|---|
| 5.1 | **Operational peer** | Durable operational state surviving feature-peer crash; cycle-safe self-observation; externally observable | Feature peer holds cap to write specific operational paths; operational peer does **not** subscribe back | Internal | Outbox enqueue if drain absent/slow → Class I (K1) |
| 5.2 | **Observer** | External observation/audit without modifying observed peers | Observer holds subscription caps; observed peers do not subscribe back; strictly downstream | Flat | None — best-effort by construction; drop-with-counter OK |
| 5.3 | **Service pool** | Horizontal scale; fault tolerance; per-member cap scope | Members mesh-subscribe or hub-to-state-peer; routing layer selects | Either | Member inbound queue if saturated → Class I (K1) |
| 5.4 | **Hub-and-spoke** | Centralized coordination without shared storage; independent spokes | Spokes hold caps to subscribe/pull from hub; hub does not depend on spokes | Flat | Hub notification queue if spoke unreachable (async) → Class I (K1; durable parts) |
| 5.5 | **Recovery cluster** | Identity recovery without single-party trust; K-of-N quorum | Recovery peers hold quorum-attested-op caps granted at setup | Flat | Quorum round timeout, not stall |
| 5.6 | **Bridge** | Connect mutually-distrusting domains; boundary policy; format translation | Bridge subscribes one domain, emits to the other; asymmetric per side | Either | Slower of the two domain boundaries → Class I or II by wiring |
| 5.7 | **Compute pool** | Isolate expensive computation; independent compute scaling; per-member accounting | App peer holds dispatch cap; pool members hold reverse delivery cap; results via continuation | Either | Result-delivery queue if dispatcher gone → Class I (K1) |

The detail sections below are the per-composition references. Each is grounded in the proposal §5.4 catalog stub, the exploration's §13 table, and (where it exists) a richer companion exploration.

### 5.1 Operational peer

**What it gives you.** Durable operational state — outbox entries, failure log, telemetry, audit records — that survives feature-peer crash and does not re-enter the feature peer's emit cascade. Three properties at once: durable state; cycle-safe self-observation; externally observable without re-introducing feature-peer cycles. (Proposal §5.2.)

**Shape.**

```
┌─────────────────────────────────────────────────────────┐
│  Feature peer                                           │
│  - Whatever extensions the application needs            │
│  - Phase 1 + Phase 2 cascade for application data       │
│  - Holds capability to write op-peer's operational paths│
│                                                         │
│         │ EXECUTE with capability                       │
│         ▼                                               │
│  ┌──────────────────────────────────────────────┐       │
│  │  Operational peer                            │       │
│  │  - Minimal extensions (per use case)         │       │
│  │  - Holds: outbox/, failures/, telemetry/     │       │
│  │  - Optionally subscribed-to by observer peers│       │
│  └──────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

**Coupling and capability grants.** Feature peer → operational peer: one direction, write to specific operational paths. Operational peer does not subscribe back (or its subscriptions are scoped to paths the feature peer's reactive cascade does not depend on — §4).

**Configuration tiers.**

- **Strictest.** Operational peer runs core protocol only. No extensions, no reactive paths, trivially cycle-safe. (Proposal §5.2.)
- **Typical workbench-grade.** `revision` + `history` + `identity`. Three extensions, no-cycle rule holds (§4.1), audit straightforward.

**When to use.**

- Production deployments where outbox/durability matters.
- Where the feature peer's reactive cascade should not see operational writes (the cycle-closure trap).
- Where external observation is wanted (audit, monitoring) without re-introducing cycles.

**When not to use.**

- Edge / embedded / development deployments where in-memory failure is acceptable.
- Workloads where best-effort delivery is the contract; the composition adds cost without payoff.

**Liveness reading.** The outbox enqueue path on the feature peer is the Class I site: producer (feature peer's emit cascade) → bounded queue → consumer (operational peer's delivery handler). If the operational peer is absent or slow, the producer would otherwise block indefinitely; K1 forces the queue to surface saturation as an error to the caller instead. The composition's correctness rests on K1 holding.

**Companion docs.** Proposal §5.2 (the canonical reference); exploration v2 §13 row 1; `EXPLORATION-DURABILITY-COMPARATIVE-ACTOR-SYSTEMS.md` (ten-system survey of analogous patterns — Akka, Orleans, Erlang, Pony, Kafka); `GUIDE-CROSS-PEER-MESSAGING.md` (the delivery/durability slice this composition relies on for the durable at-least-once class).

### 5.2 Observer

**What it gives you.** External observation or audit of one or more observed peers, without modifying the observed peers' behavior. Read-only by construction.

**Coupling.** Observer holds subscription capabilities on the observed peers' state; observed peers do **not** subscribe back. Strictly downstream — no closing path.

**Mode.** Typically flat; the observer is a distinct externally-visible peer (audit, monitoring, BI).

**When to use.** Compliance audit; operational dashboards; security monitoring; downstream analytics feed.

**When not to use.** When the consumer must influence the observed peers — that is a different composition (bridge §5.6, or an explicit control plane).

**Liveness reading.** Observer's read path is best-effort by construction. If the observer falls behind, drop-with-counter is acceptable; the observed peers are not blocked by observer state. No K1-relevant stall on the observed side.

### 5.3 Service pool

**What it gives you.** Horizontal scale for a stateless or shard-keyed handler set; fault tolerance via replacement; per-member capability scope.

**Coupling.** Pool members mesh-subscribe to a shared event source or hub-subscribe to a state peer; a routing layer (an explicit routing peer, infra-level load balancer, or capability-scoped EXECUTE routing) selects which member handles a request. (Proposal §5.4.)

**Mode.** Either — flat (public service pool, federation) or internal (pool hidden behind a surface peer).

**When to use.** Horizontal scaling under load; isolating per-member failure domains; shard-keyed workloads where each member owns a key range.

**When not to use.** Workloads that need strong cross-member consistency without coordination (use a single peer or a quorum composition); workloads where the routing layer becomes the bottleneck.

**Liveness reading.** Each member's inbound queue is a Class I site. K1 holds saturation as an error rather than indefinite block; the routing layer can route to another member or surface back to the caller.

### 5.4 Hub-and-spoke

**What it gives you.** Centralized coordination — a hub holding authoritative state, spokes following — without shared storage and without spokes depending on each other. Spokes are independently restartable; the hub is the single point of coordination.

**Coupling.** Spokes hold caps to subscribe to / pull from the hub. The hub is asymmetric — it does not depend on any individual spoke. (Proposal §5.4.)

**Mode.** Flat.

**When to use.** Configuration distribution; revision/version DAG following (see `GUIDE-REVISION-AUTO-VERSION.md`); event broadcast with independent consumers; coordination patterns where the hub is the authoritative source.

**When not to use.** When the hub's single point of failure is unacceptable without HA (use a different composition — replicated hub, or a quorum-based authoritative service); when spokes need to coordinate among themselves (use mesh / service pool).

**Liveness reading.** Hub's per-spoke notification queue is a Class I site for the durable-class parts of the relationship. K1 holds; async notification to an unreachable spoke surfaces as an observable durability outcome (`GUIDE-CROSS-PEER-MESSAGING.md` §3 / `EXTENSION-INBOX` §10), not as a hub-side block.

### 5.5 Recovery cluster

**What it gives you.** Identity recovery without single-party trust — a K-of-N quorum of recovery peers can collectively rotate or recover an identity that would otherwise be lost on key loss.

**Coupling.** Recovery peers hold quorum-attested-op caps granted at setup. Coordination is by quorum round, not by reactive subscription. (Companion: `core-protocol-domain/explorations/EXPLORATION-RECOVERY-CLUSTER-MECHANICS.md`.)

**Mode.** Flat.

**When to use.** Identity recovery for high-value accounts (organizations, custodial keys); social recovery patterns; offline-key recovery.

**When not to use.** Day-to-day rotation (use the identity rotation flow directly — see `GUIDE-IDENTITY.md`); operational state durability (use the operational peer §5.1).

**Liveness reading.** A quorum round has a timeout, not an indefinite stall. Not K1-sensitive in the catalog sense — the failure mode is "round did not assemble in time," which the application handles.

### 5.6 Bridge

**What it gives you.** Connect mutually distrusting domains while enforcing a boundary policy. Format translation between two capability/identity systems; policy mediation at the seam.

**Coupling.** The bridge peer subscribes to one domain and emits to the other. Asymmetric per side — the bridge holds different capability sets for each domain.

**Mode.** Either, depending on whether the bridge is externally visible (flat) or hidden inside one domain's surface (internal).

**When to use.** Cross-organization integration with no shared identity infrastructure; gradual migration between two domains; one-way data flow where policy must be enforced at the seam.

**When not to use.** When both sides can use the same identity system (no bridge needed — direct capability grants); when the bridge would have to be transactional across both domains (no protocol-level cross-peer transaction; need application-level 2PC or quorum-attested ops).

**Liveness reading.** The slower of the two domain boundaries is the Class I site (or Class II if the wiring closes a cycle through the bridge). K1 + the no-reactive-cycle rule together determine which class applies and whether the bridge wiring is safe. **Bridge wiring is the composition with the highest cycle-risk; audit per §4 is non-trivial.**

### 5.7 Compute pool

**What it gives you.** Isolate expensive computation in dedicated peers; independently scale compute capacity; account for per-member compute usage.

**Coupling.** App peer holds dispatch capability to compute pool members; pool members hold reverse delivery capabilities; results come back via continuation. Today: chains stay within a single peer or a single endosymbiotic identity (the L2 cross-peer dispatch-capability provenance gap, §7); cross-composition flow uses L1 (single EXECUTE) + subscription. (Proposal §5.4. Exploration §16.)

**Mode.** Either — flat (public compute service) or internal (hidden behind a surface peer).

**When to use.** Workloads with expensive compute that would otherwise saturate the feature peer's emit cascade; workloads where compute capacity must scale independently of feature-peer count; per-tenant compute accounting.

**When not to use.** Lightweight compute that runs fine alongside the feature workload (the composition adds latency without payoff); compute that needs reactive feedback into the feature peer's tree (the feedback loop closes a cycle — audit per §4).

**Liveness reading.** Pool member's result-delivery queue is a Class I site. K1 holds saturation as observable rather than blocking; if the dispatcher is gone, the durability contract (`GUIDE-CROSS-PEER-MESSAGING.md`) determines what happens to the result.

## 6. Construction

A composition is a deployment unit. The construction sequence is the same shape across compositions; ordering matters. (Exploration §15.)

1. **Generate / load identities** for each peer in the composition. Each peer has a separate identity per `GUIDE-IDENTITY.md`.
2. **Initialize per-peer storage.** Default: separate stores per peer (strongest isolation, clearest restart). Co-located storage is acceptable for endosymbiotic compositions if isolation discipline is preserved at the storage layer.
3. **Bring up peers in dependency order.** Example: the operational peer must be reachable before the feature peer accepts durable writes (otherwise the durability contract returns `412 durability_required_unmet` per `EXTENSION-INBOX.md` §10).
4. **Issue inter-peer capability grants.** Each peer receives the capabilities it needs to act on the others; the grants are revocable and follow the coherent capability principle (`GUIDE-CAPABILITIES.md`).
5. **Wire subscriptions.** Establish the coupling pattern: which peer subscribes to which paths. Audit against the no-reactive-cycle rule (§4) before bringing the wiring up.
6. **Verify.** A composition is healthy when each peer is reachable, all expected capabilities are valid, all expected subscriptions are active, and the cycle audit passes.

### 6.1 The capability-bootstrap chicken-and-egg

A peer needs a capability before it can act, but the issuer must already know whom to grant. Three answers, all protocol-supported (choice is operational):

- **Config-time injection.** Capability material is bundled into each peer's initial configuration. Simplest; rotation requires redeploy.
- **Bootstrap handshake.** Peers exchange capabilities at first contact over a trusted channel. Rotation without redeploy; requires a way to establish the trusted channel.
- **Attestation-driven.** A trusted authority (or quorum, per `GUIDE-MULTISIG.md` / recovery cluster §5.5) attests which peer identifier corresponds to which composition role, and capabilities are derived from the attestation chain. Most flexible; authority comes first.

### 6.2 Convergent setup ("kubectl apply for compositions")

Construction should be **convergent**: re-issuing a capability that already exists is a no-op (content addressing makes it identical); re-subscribing to a path already subscribed is a no-op (same subscription entity, same path, same capability — content-addressed identical). The setup script for a composition should be idempotent by construction — apply it twice, observe no second effect. This pays off for redeploy, scale-out, and recovery.

## 7. Persistence

Per-peer restart-equivalence is independent and solid: each peer with durable storage that stops and restarts produces externally observable behavior equivalent to a continuously-running peer holding the same durable state (SYSTEM-COMPOSITION §6.7.4; see `GUIDE-RESTART-AND-PERSISTENCE.md`).

**Composition restart** is per-peer restart-equivalence plus coupling-state recovery: subscriptions reattach, capabilities re-hydrate, outboxes drain. The current open gap is **cross-peer continuation resumption** — the L2 dispatch-capability provenance gap (G2 in `EXPLORATION-PEER-COMPOSITIONS.md` v2 §16). Today this means continuation chains stay within a single peer (or a single endosymbiotic identity where the capability is internal); cross-composition coordination uses L1 (single EXECUTE) + subscription. **None of the seven compositions *requires* cross-peer L2;** closing the gap is a separate spec change tracked under the continuation-thread re-open work, not in this guide.

## 8. Observation

Per-peer observability is standard: operational state (V7 §3.13), subscriptions, continuations, capability inventory, cascade-depth and drop counters. Composition-*level* concerns (inter-peer flow, capability validity across the whole composition, coupling health, aggregate workload) are observable through three mechanisms, each with tradeoffs:

- **An observer peer** (§5.2) subscribed to the composition's operational paths. Lowest-overhead approach for ongoing observation.
- **Externally aggregated per-peer stats** scraped/pushed by an out-of-band system (Prometheus-style). Works without an observer peer; does not see entity-level events.
- **Static analysis** of the composition's tree state. Composition wiring (subscriptions, continuations, compute graphs) lives in tree state and is readable; cycle audit (§4) is one example of this style.

The operational peer is the natural observation interior for a feature peer: `failures/*`, `outbox/*`, `telemetry/*` all live on the operational peer, observable without re-introducing feature-peer cycles (the cycle-safe self-observation property of §5.1). This is the endosymbiosis-pays-off case.

## 9. Management

**Capability rotation.** Staged — *issue → distribute → verify → revoke*. There is no cross-peer atomic rotation; revoking before distribution lands creates a window during which the composition is misconfigured. Per-step rollback is per-step (revoke is the rollback for issue; re-issue is the rollback for revoke); the full sequence has no single rollback point.

**Pool membership.** Adding a service-pool member (§5.3) or compute-pool member (§5.7) is online: provision identity → bring up → issue scoped capability → routing layer sees it → add to rotation. Removing is the reverse. Adding a second operational peer is a *different composition* (HA pair), not "adding a peer" — it changes the coupling pattern, not just the member count.

**Backup and restore.** Per-peer tree backup + capability-chain re-validation on restore (the restored peer's identity must still be authorized by the issuers' current state) + subscription reactivation (subscribe entities exist as tree state on the subscriber; the subscribed peer accepts them when the capability validates).

**Multi-tenancy.** One operational peer with tenant-scoped paths and per-tenant capability sets is one shape; a separate operational peer per tenant is another. Both are compositions of the existing primitives; choice is driven by isolation requirements.

**Budgets.** Cascade depth resets at the wire boundary; TTL on bounds decrements per hop. Typical composition depth is 2–4; the 32-hop ceiling (SYSTEM-COMPOSITION §3.2) gives ample headroom.

## 10. Anti-patterns and what stays open

### 10.1 Anti-patterns

- **Reading "no-reactive-cycle rule" as the deadlock answer.** It is necessary, not sufficient. The Class I case (the common one) is *invisible* to it. (§3, §4. Exploration §10.)
- **Operational peer used for general state.** It is a *durability* composition; using it as a shared cache or low-latency read-side defeats the isolation purpose and adds cross-peer latency to operations that should be local.
- **Bridge as a general gateway.** The bridge composition exists for *mutually distrusting domains*; using it inside a single trust domain is unnecessary capability layering. Direct capability grants suffice.
- **Compute pool for trivial compute.** Latency cost dominates payoff; run the compute locally.
- **Composition mode declared in spec text.** Mode is a deployment property, not a protocol property; the spec should not require it.

### 10.2 What stays open

- **Ephemeral peers** — born per request, dead after result. Lifecycle inverts the catalog: no stable identity wanted; capability bootstrap *is* the request latency; the result must reach a surviving peer (durable class) before teardown — a Class I concern at exactly the teardown moment. May be the limiting case that pressure-tests "what is a composition" (closer to a process fork). Explicitly *not* forced into the seven. (Exploration §17.)
- **Catalog growth model** — ratified centrally vs accreted from usage. (Exploration §17.)
- **Composition identity** — in endosymbiosis mode, what is the externally-visible identity; is composition identity rotation different from single-peer rotation (multiple rotation boundaries, one per peer). (Exploration §17.)
- **Migration between compositions** — single peer → operational-peer composition: native support or deployment concern. (Exploration §17.)
- **Composition-level conformance testing** — per-peer conformance is what the spec tests today; composition-level properties (e.g., "the operational-peer composition does not stall under bulk-ingest load") need a different conformance shape. (Exploration §17. The workbench-go bulk-ingest scenario, `shellcmd/cmd_local_files_sqlite_diag_test.go`, is the standing reference workload.)
- **The L2 cross-peer dispatch-capability gap (G2)** — chains stay within a single peer today; closing the gap is the continuation thread's domain when re-opened. (§7. Exploration §16.)

## 11. Where things live (the layered commitment)

The layered home for each element, per proposal §10:

| Element | Where | Why there |
|---|---|---|
| **K1** core liveness invariant | Normative spec (SYSTEM-COMPOSITION emit pathway or V7 §6.10) when ratified | A protocol-level guarantee the catalog rests on. |
| **K2** path-scoped dispatch invariant | Normative spec (same place) | Emit cost is path-scoped, not whole-tree. |
| **K3** bounded cascade | Already normative (SYSTEM-COMPOSITION §3.2) | The ceiling that prevents unbounded recursion. |
| **Three delivery classes** (durable at-least-once, sync request-response, async fire-and-forget) | Sync request-response and async fire-and-forget are normative in V7 v7.48 + EXTENSION-INBOX v5.9 + EXTENSION-CONTINUATION (the v7.48 surface — v7.46 baseline after the durability retraction, plus v7.48's §4.8 inbound dispatch concurrency invariant — neither changes delivery semantics for these classes). The durable class was briefly landed (V7 v7.47 + EXTENSION-INBOX §10) then retracted — see the exploratory `EXTENSION-DURABILITY.md` and `GUIDE-CROSS-PEER-MESSAGING.md`. | Spec-level vocabulary the catalog composes from. |
| **No-reactive-cycle coupling rule** | This guide (operator/implementation guidance, not spec) | Coupling patterns multiply; not a protocol concern; not runtime-enforced. |
| **The catalog of named compositions** | This guide | Application-layer; grows as the system is used. |
| **Composition vocabulary** (composition / topology / mode) | Settled in exploration v2 §12; carried by this guide | Foundational; the proposal sweeps it across spec docs. |

## 12. Cross-references

- `core-protocol-domain/explorations/EXPLORATION-PEER-COMPOSITIONS.md` (v2) — the orientation doc; Part II/III (load-bearing spine: K1, the three axes, two-layer deadlock prevention), Part IV (this catalog's source), Part V (frontier).
- `core-protocol-domain/explorations/EXPLORATION-PEER-COMPOSITIONS-LEGACY.md` (v1) — the append-wise research record; preserved verbatim for traceability.
- `proposals/deferred/PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md` — the open proposal carrying the vocabulary and framing toward normative ratification; §11 = the ratification questions (Q1/Q2/Q4 adopted; Q5 K1 a/b/c scope is the one-pass sign-off gating K1 spec-text only).
- `core-protocol-domain/guides/GUIDE-CROSS-PEER-MESSAGING.md` — the delivery/durability slice; one pattern *within* the framework here.
- `core-protocol-domain/guides/GUIDE-RESTART-AND-PERSISTENCE.md` — per-peer restart-equivalence the composition restart pattern builds on.
- `core-protocol-domain/guides/GUIDE-CAPABILITIES.md` — capability bootstrap patterns and the coherent capability principle the per-composition cap grants follow.
- `core-protocol-domain/guides/GUIDE-IDENTITY.md` — per-peer identity provisioning the composition construction sequence calls.
- `core-protocol-domain/guides/GUIDE-MULTISIG.md` — the K-of-N primitive recovery cluster (§5.5) builds on.
- `core-protocol-domain/explorations/EXPLORATION-RECOVERY-CLUSTER-MECHANICS.md` — companion exploration for the recovery cluster.
- `core-protocol-domain/explorations/EXPLORATION-DURABILITY-COMPARATIVE-ACTOR-SYSTEMS.md` — ten-system survey contextualizing the operational peer and the broader catalog against precedent.

---

*Draft. First-pass authoring of deliverable §9 step 1 of `PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md`; written ahead of K1 normative ratification as the design-readability test for the catalog. Tracked as **I-10** on `proposals/PROPOSAL-INDEPENDENT-ITEMS-CATCHUP.md`. Standing post-merge follow-on: when K1 lands normative (after Q5 sign-off), flip the "not yet normative" markers in §2 and the §11 K1 row.*
