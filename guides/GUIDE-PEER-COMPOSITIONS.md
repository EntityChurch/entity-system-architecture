# Peer Compositions and Topologies

**Status**: Draft
**Audience:** Peer operators and developers wiring **more than one peer** together; reviewers comparing this system to actor frameworks, sidecar patterns and federation models.
**Spec reference:** `SYSTEM-COMPOSITION.md` §3.2 (bounded cascade), §6.7.4 (restart equivalence) · `ENTITY-CORE-PROTOCOL.md` §3.13 (operational state), §5 (capabilities), §6.10 (per-binding atomicity).
**Related guides:** `GUIDE-CROSS-PEER-MESSAGING.md` (the delivery/durability slice — one pattern *within* this framework) · `GUIDE-CAPABILITIES.md` · `GUIDE-IDENTITY-SDK.md` · `GUIDE-MULTISIG.md` · `GUIDE-RESTART-AND-PERSISTENCE.md`.

---

> ## ⚠ What this guide answers, and what it does NOT
>
> **This is the DEPLOYMENT question: *I have several peers — how do I wire them together, what do I
> grant, and what does the arrangement give me?*** It is written for whoever runs the peers.
>
> **It is not the application-development question.** *"What do I install to build a thing, which
> conventions apply, when do I write a handler instead of an extension, and what can I build on top?"*
> is a different question with a different audience, and this guide does not answer it. Nothing here
> tells an application developer what to install. See `GUIDE-APPLICATION-DEVELOPMENT.md` for the
> convention tier and `GUIDE-EXTENSION-DEVELOPMENT.md` for the extension tier — and note that **the gap
> between those two and this one is real and is not yet closed by any document.**
>
> **Nothing in this guide requires a protocol change.** Every composition below is a configuration of
> primitives that already exist.

---

## 1. The frame: a peer is a configuration, a composition is a family

**A peer is not a role. It is a *configuration*:** cryptographic identity + installed handlers +
installed extensions + capability grants held and issued + tree state + active continuations + active
subscriptions + operational state. **Two peers identical in identity and extensions still differ if any
other facet differs.** The peer is the whole configuration.

**A composition** is a family of configurations sharing enough character to discuss as a class: peers
wired together via capability grants and coupling (subscriptions, continuations, direct EXECUTE) that
yield a property no single peer can yield alone.

> **Compositions are NOT hierarchical — there is no "parent peer."** Peers are peers; they differentiate
> by *role* and *configuration*, never by structural relationship. An early framing of this material used
> "sub-peer" language that implied hierarchy, and it was wrong.

A composition's **topology** is its network-shape aspect — who couples to whom. A composition *has* a
topology; the two are not synonyms.

A composition's **mode** is a **deployment property, not a composition property**, and should never
appear in spec text:

- **Surface / endosymbiotic (internal).** External clients see one identity; supporting peers are
  invisible infrastructure. One organism outside, a cooperative composition inside.
- **Flat.** All member peers are equally externally visible — federation, mesh, public hub-and-spoke.

**The same composition deploys in either mode.**

> **Terminology guard.** "Role" is taken — the role extension is RBAC. **"Operational peer" and
> "observer peer" are *configurations*, not entries in that extension.** A configuration *may* include
> role assignments as one facet.

---

## 2. The liveness property the catalog rests on

**K1 — Layer-1 liveness:** *No single component can indefinitely stall a peer. The emit/delivery pathway
is non-blocking-or-error: saturation surfaces as an error to the caller, never an indefinite block, and
never a silent drop.*

K1 is what makes **Class I stalls impossible** (§3). Every Class I row in the catalog — queue or outbox
saturation — is prevented by K1 holding, **not by any topology choice.**

> ⛔ **K1 IS NOT IN NORMATIVE SPEC TEXT, and that is a real gap rather than a formality.** It is
> converged across two independent implementations and carried here and in this guide's source
> exploration; its three scope facets — the termination boundary, the coverage boundary, and the
> shed-versus-durability boundary — are drafted and were gated on a sign-off that never happened.
> **The catalog assumes implementations behave consistently with K1 in practice today; the catalog's
> correctness becomes audit-checkable only once K1 lands normative.** Where a deployment depends on a
> liveness reading below, that reading currently rests on observed behaviour rather than on a contract.

Two already-normative invariants also underpin the catalog and are named so they do not fall out of view:

- **Per-binding atomicity** — the emit binding update is the atomic state crossing.
- **Bounded cascade** — a hard ceiling of 32.

---

## 3. Two classes of stall, and two prevention mechanisms that are not interchangeable

- **Class I — intra-peer backpressure stall.** A producer inside one peer blocks on a queue or outbox
  whose consumer is absent or persistently slow. **No cycle.** This is the common case. **Prevented only
  by K1.**
- **Class II — reactive-cycle stall.** Peer A's reactive paths depend on peer B's state changes and B's
  depend on A's. The loop closes; the system spins or stalls. **Prevented by the no-reactive-cycle rule
  (§4).**

> ⛔ **The anti-pattern to avoid on first reading: treating the cycle rule as "the deadlock answer."** It
> is necessary for Class II and **structurally invisible to Class I**, which is the common case. Read
> that way, K1 gets under-invested and the case that actually happens is left unprevented.

---

## 4. The no-reactive-cycle coupling rule (Class II)

**The rule:** *In a composition where you want to prevent reactive feedback from peer B back into peer A,
peer A's reactive paths — subscriptions, continuations, compute reactivity — must not depend on peer B's
state changes.*

It is a **design discipline for the coupling pattern**, verifiable by analysing installed subscriptions,
continuations and compute graphs in tree state. **It is not a capability restriction and not a runtime
check** — the capability layer remains the runtime security boundary.

In an A → B composition where A should not reactively depend on B:

| Direction | Allowed? | Why |
|---|---|---|
| A **reads** from B (sync) | ✅ | Reads do not trigger A's reactive cascade |
| A **subscribes** to B's state | ⛔ **avoid** | Closes the cycle the composition exists to break |
| B **reads** A's tree | ✅ | One-shot; does not subscribe back |
| B **subscribes** to A's outbox-write events | ✅ | The delivery loop's whole purpose |
| B **writes** to A's tree (e.g. a delivery ack) | ✅ | One-shot; A decides whether its reactive paths depend on it |
| B **runs continuations** | ✅ | As long as the chain terminates without re-entering A's reactive paths |
| B **has extensions installed** | ✅ | Each extension's reactive surface should be audited for cycle closure |

### 4.1 Strictness levels

- **Strictest (trivial audit).** The secondary peer runs **core protocol only** — no extensions, no
  reactive paths, no possible cycle.
- **Typical (small audit).** The secondary peer installs **revision** (versions the outbox auditably;
  monotonic, no cross-peer reactivity), **history** (records only) and **identity**. Three extensions,
  rule holds, audit is straightforward.
- **Less strict (full audit per extension).** Anything else, with an explicit audit that each added
  extension's reactive surface respects the rule.

---

## 5. The catalog — seven named compositions

| # | Composition | Gives you | Coupling | Typical mode | Liveness-sensitive coupling |
|---|---|---|---|---|---|
| 5.1 | **Operational peer** | Durable operational state surviving feature-peer crash; cycle-safe self-observation | Feature peer holds a cap to write specific operational paths; op peer does **not** subscribe back | Internal | Outbox enqueue if drain absent → Class I |
| 5.2 | **Observer** | External observation/audit without modifying observed peers | Observer holds subscription caps; observed peers do not subscribe back | Flat | None — best-effort by construction |
| 5.3 | **Service pool** | Horizontal scale; fault tolerance; per-member cap scope | Members mesh- or hub-subscribe; a routing layer selects | Either | Member inbound queue → Class I |
| 5.4 | **Hub-and-spoke** | Centralized coordination without shared storage; independent spokes | Spokes hold caps to subscribe/pull; hub does not depend on any spoke | Flat | Hub notification queue → Class I |
| 5.5 | **Recovery cluster** | Identity recovery without single-party trust; K-of-N quorum | Recovery peers hold quorum-attested-op caps granted at setup | Flat | Quorum round **timeout**, not stall |
| 5.6 | **Bridge** | Connect mutually distrusting domains; boundary policy; format translation | Bridge subscribes one domain, emits to the other; asymmetric per side | Either | Slower boundary → Class I **or II by wiring** |
| 5.7 | **Compute pool** | Isolate expensive computation; scale compute independently; per-member accounting | App peer holds dispatch cap; members hold reverse delivery caps; results via continuation | Either | Result-delivery queue → Class I |

### 5.1 Operational peer

**Gives you** durable operational state — outbox entries, failure log, telemetry, audit records — that
survives feature-peer crash **and does not re-enter the feature peer's emit cascade.** Three properties at
once: durability, cycle-safe self-observation, and external observability without re-introducing cycles.

**Coupling.** Feature peer → operational peer, one direction, write to specific operational paths. The
operational peer does not subscribe back (or its subscriptions are scoped to paths the feature peer's
reactive cascade does not depend on).

**Use it** where outbox durability matters, where the feature peer's reactive cascade must not see
operational writes, or where external observation is wanted without re-introducing cycles.
**Do not** use it for edge or development deployments where in-memory failure is acceptable, or where
best-effort delivery is the contract — it adds cost without payoff.

⚠ **Anti-pattern: using it as general state.** It is a *durability* composition. As a shared cache or a
low-latency read side it defeats the isolation purpose and adds cross-peer latency to operations that
should be local.

### 5.2 Observer

Read-only by construction: the observer holds subscription capabilities; observed peers do not subscribe
back. **Strictly downstream, no closing path.** Compliance audit, dashboards, security monitoring,
analytics. **Not** for a consumer that must influence the observed peers — that is §5.6 or an explicit
control plane. If the observer falls behind, drop-with-counter is acceptable; the observed peers are
never blocked by observer state.

### 5.3 Service pool

Horizontal scale for a stateless or shard-keyed handler set, fault tolerance by replacement, per-member
capability scope. **Not** for workloads needing strong cross-member consistency without coordination —
use a single peer or a quorum composition — nor where the routing layer becomes the bottleneck.

### 5.4 Hub-and-spoke

A hub holding authoritative state with spokes following, **without shared storage and without spokes
depending on each other.** Spokes are independently restartable. **Not** where the hub's single point of
failure is unacceptable without HA, and not where spokes need to coordinate among themselves.

### 5.5 Recovery cluster

A K-of-N quorum of recovery peers can collectively rotate or recover an identity that would otherwise be
lost on key loss. Coordination is **by quorum round, not by reactive subscription**, so the failure mode
is *"the round did not assemble in time"* rather than a stall. **Not** for day-to-day rotation (use the
identity rotation flow) or operational durability (use §5.1).

### 5.6 Bridge

Connects mutually distrusting domains, enforcing a boundary policy and translating formats. The bridge
holds **different capability sets for each side.** **Not** needed when both sides can use the same
identity system — direct grants suffice — and **not** usable where the bridge would have to be
transactional across both domains, since there is no protocol-level cross-peer transaction.

⛔ **The bridge has the highest cycle risk of the seven and its §4 audit is non-trivial.** It is also the
composition most often misused as a general gateway inside a single trust domain, which is unnecessary
capability layering.

### 5.7 Compute pool

Isolates expensive computation, scales compute independently of feature-peer count, and enables
per-member accounting. **Not** for lightweight compute that runs fine alongside the feature workload —
latency dominates payoff — and **not** for compute needing reactive feedback into the feature peer's
tree, which closes a cycle.

⚠ **Today, continuation chains stay within a single peer or a single endosymbiotic identity**, because
cross-peer dispatch-capability provenance is an open gap (§7). Cross-composition flow uses a single
EXECUTE plus subscription.

---

## 6. Construction

A composition is a deployment unit, and **ordering matters**:

1. **Generate or load identities** — one per peer.
2. **Initialize per-peer storage.** Default to **separate stores per peer** (strongest isolation,
   clearest restart). Co-located storage is acceptable for endosymbiotic compositions if isolation
   discipline is preserved at the storage layer.
3. **Bring up peers in dependency order.** The operational peer must be reachable before the feature peer
   accepts durable writes.
4. **Issue inter-peer capability grants.** Revocable, following the coherent capability principle.
5. **Wire subscriptions**, and **audit against §4 before bringing the wiring up.**
6. **Verify.** A composition is healthy when each peer is reachable, all expected capabilities are valid,
   all expected subscriptions are active, and the cycle audit passes.

### 6.1 The capability-bootstrap chicken-and-egg

A peer needs a capability before it can act, but the issuer must already know whom to grant. **Three
answers, all protocol-supported — the choice is operational:**

- **Config-time injection.** Bundled into initial configuration. Simplest; rotation requires redeploy.
- **Bootstrap handshake.** Peers exchange capabilities at first contact over a trusted channel. Rotation
  without redeploy; requires establishing the trusted channel.
- **Attestation-driven.** A trusted authority or quorum attests which peer identifier holds which
  composition role, and capabilities derive from the attestation chain. Most flexible; authority first.

### 6.2 Convergent setup

**Construction should be idempotent by construction.** Re-issuing a capability that already exists is a
no-op — content addressing makes it identical. Re-subscribing to a path already subscribed is a no-op —
same subscription entity, same path, same capability, content-addressed identical. **Apply the setup
twice, observe no second effect.** This is what pays off for redeploy, scale-out and recovery.

---

## 7. Persistence

Per-peer restart-equivalence is independent and solid: a peer with durable storage that stops and
restarts produces externally observable behaviour equivalent to a continuously running peer holding the
same durable state.

**Composition restart** is that plus coupling-state recovery: subscriptions reattach, capabilities
re-hydrate, outboxes drain.

⚠ **The open gap is cross-peer continuation resumption** — cross-peer dispatch-capability provenance.
Today chains stay within a single peer or a single endosymbiotic identity. **None of the seven
compositions *requires* cross-peer continuation**, so this bounds the catalog rather than blocking it.

---

## 8. Observation

Composition-*level* concerns — inter-peer flow, capability validity across the whole composition,
coupling health, aggregate workload — are observable three ways, each with a tradeoff:

- **An observer peer** (§5.2) subscribed to the composition's operational paths. Lowest ongoing overhead.
- **Externally aggregated per-peer stats**, scraped or pushed out-of-band. Works without an observer
  peer; **does not see entity-level events.**
- **Static analysis of tree state.** Composition wiring — subscriptions, continuations, compute graphs —
  lives in tree state and is readable. The §4 cycle audit is one instance of this style.

⭐ **The operational peer is the natural observation interior for a feature peer**: failures, outbox and
telemetry all live there, observable without re-introducing feature-peer cycles.

---

## 9. Management

**Capability rotation is staged — *issue → distribute → verify → revoke*.** ⛔ **There is no cross-peer
atomic rotation**, so revoking before distribution lands creates a window in which the composition is
misconfigured. Rollback is per-step (revoke undoes issue; re-issue undoes revoke); **the full sequence has
no single rollback point.**

**Pool membership** changes online: provision identity → bring up → issue scoped capability → routing
layer sees it → add to rotation; removal is the reverse. ⚠ **Adding a second operational peer is a
DIFFERENT composition (an HA pair), not "adding a peer"** — it changes the coupling pattern, not the
member count.

**Backup and restore** is per-peer tree backup, plus capability-chain re-validation on restore (the
restored identity must still be authorized by the issuers' *current* state), plus subscription
reactivation.

**Multi-tenancy** is either one operational peer with tenant-scoped paths and per-tenant capability sets,
or a separate operational peer per tenant. Both are compositions of existing primitives; isolation
requirements drive the choice.

**Budgets.** Cascade depth resets at the wire boundary; TTL decrements per hop. Typical composition depth
is 2–4 against a 32-hop ceiling — ample headroom.

---

## 10. What stays open

- **Ephemeral peers** — born per request, dead after result. The lifecycle inverts the catalog: no stable
  identity is wanted, capability bootstrap *is* the request latency, and the result must reach a
  surviving peer before teardown — a Class I concern at exactly the teardown moment. **Deliberately not
  forced into the seven**; it may be the limiting case that pressure-tests what a composition is.
- **Catalog growth** — ratified centrally, or accreted from usage.
- **Composition identity** — in endosymbiosis mode, what the externally visible identity is, and whether
  composition rotation differs from single-peer rotation given multiple rotation boundaries.
- **Migration between compositions** — single peer → operational-peer composition: native support or
  deployment concern.
- **Composition-level conformance** — per-peer conformance is what is tested today. Composition-level
  properties (*"this composition does not stall under bulk-ingest load"*) need a different shape.
- **K1 normative** (§2) and **cross-peer continuation** (§7).

---

## 11. Where each element belongs

| Element | Where | Why |
|---|---|---|
| **K1** — layer-1 liveness | **Owed: normative spec.** Not there today (§2) | A protocol-level guarantee the catalog rests on |
| **Path-scoped dispatch** | Normative spec | Emit cost is path-scoped, not whole-tree |
| **Bounded cascade** | **Already normative** | The ceiling preventing unbounded recursion |
| **Delivery classes** — sync request-response, async fire-and-forget | **Already normative.** A *durable* class was drafted, briefly landed and retracted; it survives as the exploratory durability extension | Spec-level vocabulary the catalog composes from |
| **The no-reactive-cycle rule** | **This guide** — operator guidance, not spec | Coupling patterns multiply; not a protocol concern; not runtime-enforced |
| **The catalog** | **This guide** | Grows as the system is used |
| **Composition vocabulary** | **This guide** | Foundational |

---

## 12. Provenance

This guide was authored against the V7 core revision and **transferred into this corpus on 2026-09-13**
from the pre-split architecture archive, where it had been left behind along with its source exploration,
its closed proposal, and the earlier append-wise research record. It was cited by name from
`GUIDE-CROSS-PEER-MESSAGING.md` in this corpus for the whole intervening period, as though a reader could
open it.

**What changed in transfer:** internal tracking identifiers and cross-repo paths were removed; citations
were re-pointed at documents that exist in this corpus or marked as archive-only; **K1's status was
restated as an open gap rather than as a pending sign-off**, which is what it has become. **The catalog,
the vocabulary, the two stall classes and the coupling rule are carried unchanged** — they describe
configurations of primitives, and those primitives have not moved.

**Not transferred:** the recovery-cluster mechanics exploration and the ten-system comparative survey of
durability in actor systems, both of which remain archive-only.
