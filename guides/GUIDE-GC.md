# GUIDE-GC — Storage cleanup in entity-core

**Status**: Active
**Companion proposals:** REVISION amendment defining `version_retention_window` (TBD); CONTENT §1.2 reframe replacing the EXTENSION-GC deferral with a pointer to this guide.

**Audience:** implementers of the entity-core protocol (reference
impls and downstream peers); extension authors auditing their own
GC posture per §10.

**Scope.** Operational guidance for cleaning up the local content
store. **Not normative.** The protocol does not require any
peer to do GC — a peer with infinite storage is perfectly valid.
What the protocol cannot tolerate is cleanup that *corrupts the
data semantics other peers depend on*. This guide names the
invariants you must preserve and the pitfalls that violate them.

---

## §1 Why a guide and not an extension

A peer with infinite storage never needs GC. So GC isn't
protocol-mandatory the way INBOX delivery or REVISION versioning
are — it's an operational concern of resource-constrained peers.
This matches every other mature content-addressed system: Git,
IPFS, and Nix all treat GC as a *local porcelain command*, not a
wire-level feature. Their protocols (Git fetch/push, IPFS
exchange, Nix substituter) have no GC notion at all.

What *is* protocol-mandatory is the **retention semantics each
extension declares for the state it owns** — REVISION's
`version_retention_window`, HISTORY's `max_depth`,
CONTINUATION's `dispatch_capability.ttl`, INBOX's
`deliver_token.ttl`. Those live in the extension specs because
peers reference each other's retained state across the wire. This
guide takes those retention windows as inputs and explains how to
respect them when cleaning up locally.

The split:

| Concern | Where it lives |
|---|---|
| **Retention semantics** — "how much do I promise to keep?" | Each extension's spec (declared in the extension; consumed by GC) |
| **Cleanup mechanism** — "how do I drop data safely?" | This guide |

---

## §2 What "live" means

A hash is **live** if any of the following holds. A hash that is
not live is GC-eligible. The list is a *union*; satisfying one
condition is enough.

### 2.1 The seven live-root categories

1. **Authoritative tree binding.** A path under
   `/{local_peer_id}/...` is bound to the hash. The local peer
   produced or accepted authoritative responsibility; if you GC
   this and another peer asks for it, you become an unreliable
   authority.

2. **Pinned hash.** The deployment has explicitly marked this
   hash as retained. Pinning is a deployment convention; this
   guide §6 covers pinning patterns. Pin set is *additive over*
   the other categories — pinning something already live is a
   no-op; pinning something otherwise-eligible keeps it.

3. **Cached / derived tree binding with non-expired retention.**
   A path under `/{other_peer_id}/...` is bound to the hash, and
   the binding's storage-entry TTL has not crossed. These are
   foreign-peer caches; eviction is fine within retention rules
   but premature eviction creates sync-lag failures (see §3.3).

4. **In-flight runtime state.** The hash is referenced by:
   - a suspended continuation at
     `system/continuation/suspended/*` whose `dispatch_capability`
     is unexpired (EXTENSION-CONTINUATION §5.2)
   - a pending inbox delivery whose `deliver_token` is unexpired
     (EXTENSION-INBOX §2.3)
   - an active subscription's payload
   - an in-flight chain walk (capability chain, attestation
     chain, version chain) — the walker holds references
     transiently

5. **Version-history reachable.** Reachable from any current
   version head per EXTENSION-REVISION's `parents` graph, AND
   within the configured `version_retention_window` (see proposal
   amendment).

6. **Transition-history reachable.** Reachable from
   `system/history/head/{path}` via `previous` field within
   EXTENSION-HISTORY §3.3's `max_depth`.

7. **Transitively reachable.** Referenced via any live entity's
   `refs` map. Recursively applies — if entity A is live by
   1–6 and refs B, then B is live; if B refs C, C is live; etc.

This is the **structural reachability closure**. Categories 1–6
define the *roots*; category 7 is the transitive closure over
the ref graph.

### 2.2 What isn't live

The complements to 1–6:

- Tree bindings under foreign-peer roots with expired retention.
- Suspended continuations whose dispatch capability has expired.
- Inbox deliveries with expired tokens (per INBOX §2.3 the
  handler MUST discard these anyway).
- Lost-error markers past their retention window (24h default
  per CONTINUATION §5.2).
- Version-history entries past `version_retention_window`.
- Transition-history entries past `max_depth`.
- Provenance / storage-entry / pin-set sidecars whose target is
  not live (cleaned up *with* the target, not independently).

A hash satisfying *none* of the live categories and *none* of
the sidecar exceptions is GC-eligible.

---

## §3 The pitfall catalog

The whole point of this guide.

### 3.1 Don't reap a hash with a live tree binding

Naive: "iterate paths, find unreferenced hashes, reap them."
Buggy if a path-based scan is incomplete or stale. A hash that
loses its ref-count from path scanning but still has a binding
elsewhere will produce `get` failures on the binding.

**Discipline.** Before reaping any hash, verify: is any path in
the location index currently bound to this hash? If yes, do not
reap. This is the *minimum* condition.

### 3.2 Don't reap a hash referenced by any live entity

The revision-graph corruption case the user named directly.
Version V_n refs V_{n-1} via `parents`. V_{n-1} has no current
tree binding (versions retire from being head-pointed). Path-
based GC would mark V_{n-1} unreferenced and reap it. V_n is
now dangling — `walk_history()` traverses into a void.

The same case occurs across the protocol:
- Attestation chains reference previous attestations (`supersedes`).
- Identity chains reference predecessor certs (`identity-rotation-handoff`).
- Capability chains reference parent caps.
- Trie-root snapshots reference subtree node hashes.
- Continuation graphs reference prior states.

**Discipline.** Walk the ref graph, not just the path index. For
every candidate reap, ask: is any live entity transitively
referencing this hash? Implement this as a tracing pass — mark
all live roots, walk refs, mark visited, reap unvisited.

This is the single most expensive operation in a correct GC
implementation. Incremental approaches (track refs as bindings
change, maintain a refcount-per-hash) are valid optimizations but
require careful invariants — the tracing pass is the spec for
correctness.

### 3.3 Respect the cross-peer grace window

Scenario: peer A caches hash H from peer B. A's policy says "drop
caches after 10 minutes of inactivity." A drops H. Meanwhile,
peer B sends a follow-up message referencing H, expecting A to
still have it locally. A asks B for H again. Round-trip wasted.

Worse case: B *also* drops H (B's binding went away; H became
GC-eligible on B too). Now neither peer has H, and any third
peer relying on the cache is also broken.

**Discipline.** Cached entries (category 3 above) should respect
a minimum grace window between binding-removal-or-touch and
content-reap-eligibility. Cassandra calls this
`gc_grace_seconds`; it's sized to typical sync-lag plus margin.

**Grace is a knob, not a value.** Specs expose `cross_peer_grace_seconds` (or its analogue per extension); deploying users pick. Operational considerations to think through when choosing a value:

- **Cached foreign-peer entries.** Grace bounds sync-lag-vs-disk-cost tradeoff. Order-of-magnitude anchors: shorter than typical-RTT-plus-margin loses cross-peer safety; longer than useful-retention-horizon costs disk for no benefit. Tune to the deployment's RTT, sync cadence, and how often cross-peer references arrive after their referent's binding has been touched.
- **Authoritative own-peer entries that lost their binding.** Grace = operator-recovery window. Git's `gc.pruneExpire=2.weeks.ago` is the philosophical anchor (recovery is what this grace buys, not safety). A peer with strict audit obligations may need much longer; an ephemeral test peer needs none.

The spec doesn't enshrine numbers; the deployment picks. This guide names the considerations and the tradeoff axes; pick accordingly.

### 3.4 Respect each extension's declared retention

REVISION says "keep N versions" — GC respects N.
HISTORY says `max_depth=K` — GC respects K.
CONTINUATION says "dispatch capability expires at T" — GC respects T.

**Discipline.** GC is downstream of extension-declared retention.
Read each extension's config; treat declared windows as
inviolable. Never reap something the extension itself considers
retained.

The composition is where the pitfall is. Naive GC that respects
tree bindings but ignores REVISION's retention will reap version
history that REVISION said to keep. Naive GC that respects
REVISION but ignores CONTINUATION's runtime state will break
in-flight async work.

§4 (cross-extension composition) covers each extension's
contribution.

### 3.5 Don't GC own-peer authoritative content (default)

It's tempting on space-constrained peers (browser, mobile). It
breaks the peer's role as authority — other peers cached your
content expecting you'd still be there.

**Discipline.** Authoritative paths are roots (§2.1 cat 1). If a
deployment needs to drop authoritative content (genuinely
shrinking the peer's authority surface), do it by *removing the
binding first* — the binding-removal is the protocol-visible
event, signaling to other peers that they should not expect this
path. Then the content becomes eligible per the normal flow.

This is the difference between "I no longer host /foo" (binding
gone) and "I'm internally inconsistent: I host /foo but the
content is missing" (binding present, content GC'd).

### 3.6 Don't reap subscription/continuation/inbox runtime state

These are live roots even when no tree binding visibly references
them. A suspended continuation at `system/continuation/suspended/{id}`
holds references to its captured state, which may include hashes
the rest of the tree doesn't currently reference. Reaping that
state breaks resumption.

**Discipline.** Treat `system/continuation/suspended/*` and the
analogous inbox/subscription state paths as **always-live tree
bindings**. Walk into them from the root set; do not eliminate
them by path-pattern.

### 3.7 Don't independently reap sidecar entities

Storage entries (`system/storage-entries/*`), provenance
(`system/provenance/*`), pin records (`system/gc/pinned/{hash}`),
audit trails — these are *about* other entities. Their lifetime
is tied to their target.

**Discipline.** When a target hash is reaped, clean up its
sidecars in the same pass. When a sidecar's target is live, the
sidecar is live. Don't run independent GC on the sidecar
namespace.

---

## §4 Cross-extension composition

Different combinations of installed extensions surface different
cleanup ramifications. This section is the per-extension
contribution to the live-set definition; consult before designing
a GC pass.

### 4.1 CONTENT

The store itself. CONTENT is the target of cleanup, not a source
of retention. §1.2 lists the *informative* reachability roots
(promoted to "see this guide §2.1" per the proposed amendment).

### 4.2 REVISION

**Retention concept.** `version_retention_window` (to be added;
see proposal). Default: unlimited.

**Reachability shape.** Version DAG via `parents` field. Each
version refs its trie root (which transitively refs subtree
nodes). Walking from `current head` backward through `parents`,
respecting `version_retention_window`, defines the live version
subset. Older versions and their unique trie subtrees are
eligible.

**Pitfall specific.** Versions share trie subtrees by hash
(structural sharing). When V_n and V_{n-1} share an unchanged
subtree, the subtree hash is referenced by both. Reaping V_{n-1}
must *not* reap the shared subtree unless V_n also retires.
The tracing pass handles this naturally; refcounting needs care.

### 4.3 HISTORY

**Retention concept.** `max_depth` per path. Existing spec §2.2 +
§3.3 truncation algorithm.

**Reachability shape.** Transition chain via `previous` field
from `system/history/head/{path}`. The first `max_depth`
transitions are live; older are unreachable from the head
pointer (the truncation algorithm severs the chain).

**Pitfall specific.** Transitions may include hash references
that aren't reachable through anything else once truncated — the
transition is the only thing keeping them alive. Reap correctly
or lose history.

**Interaction with REVISION.** Both track change-over-time but
differently. HISTORY transitions are per-path, fine-grained,
short-lived. REVISION versions are coarse-grained, long-lived.
A deployment can configure aggressive HISTORY truncation while
keeping REVISION retention long, or vice versa. GC treats them
independently — each extension declares its own retention.

### 4.4 CONTINUATION

**Retention concept.** `dispatch_capability.ttl` (existing).
24h marker grace (existing). SHOULD-clean for suspended
continuations whose dispatch capability has expired (existing
§5.2).

**Reachability shape.** Suspended continuations at
`system/continuation/suspended/*` are tree-bound (live by §2.1
cat 1 in their own peer's namespace). They internally reference
captured state hashes.

**Pitfall specific.** A continuation's `dispatch_capability` TTL
determines when the *continuation itself* becomes eligible. But
the *state hashes the continuation references* may be live by
other categories. GC the continuation; let the ref-tracing
handle the captured state (which becomes eligible only if
nothing else references it).

### 4.5 INBOX

**Retention concept.** `deliver_token.ttl` (existing). §2.3
mandates discarding queued deliveries with expired tokens.

**Reachability shape.** Pending deliveries hold references to
result entities and deliver tokens. Both are live while the
token is unexpired.

**Pitfall specific.** Once the deliver-token expires, the
pending delivery is structurally retired. The handler discards
the queued delivery (per §2.3); GC then reclaims any unreferenced
content.

### 4.6 SUBSCRIPTION

**Retention concept.** Subscription lifetime (until unsubscribe
or session end).

**Reachability shape.** Subscription payload refs and the
subscription entity itself are live while the subscription is
active. Subscription on a tree path implicitly keeps live the
*entity currently bound* at that path (because the subscriber
expects to be notified of changes from the current state).

**Pitfall specific.** A subscription on a path doesn't
necessarily keep *historical versions* live — only the current
binding. Don't conflate "subscribed to /foo" with "retain all
history of /foo." Those are REVISION's concern.

### 4.7 ATTESTATION + QUORUM + IDENTITY (trust stack)

**Retention concept.** Active identity certs and attestations
are forever-live until explicitly retired
(`identity-retirement`, `cert-rotation-recovery`). Retired
certs may have a long archive window (deployment policy).

**Reachability shape.** Cert chains via `supersedes` and the
identity-binding graph. Walk-to-root invariant (V7 §5.5)
guarantees full chain reachability is required for
verification.

**Pitfall specific.** Trust-stack entities have **long-tail
retention requirements** — an attestation issued years ago may
need to be verifiable today. Default: never aggressively GC
trust-stack entities. Reap only on explicit retirement event,
and even then respect a long archive window for forensic /
dispute scenarios.

This is the area most at risk of "we cleaned up old stuff and
broke verification of a months-old signature." Be especially
conservative here.

### 4.8 ROLE

**Retention concept.** Role definitions, assignments, exclusions
live while the role exists (no automatic expiry).

**Reachability shape.** Tree-bound under `system/role/{context}/...`.
Role-derived caps live until unassigned.

**Pitfall specific.** Role-derived capability tokens
(v2.0 PR-1: root caps) reference role definitions. GC of role
definitions while assignments exist breaks cap-chain verification.
Don't reap a role definition if any assignment still names it.

### 4.9 GROUP

**Retention concept.** Group identity stack inherits trust-stack
discipline — long-lived. Membership entities live while the
member is in the group.

**Reachability shape.** Tree-bound under
`system/group/{group_id}/...` with the identity stack mirrored.

**Pitfall specific.** Same as 4.7 for the identity-stack pieces.
Membership entities are normal tree state; standard rules
apply.

### 4.10 NETWORK

**Retention concept.** Session state lives while the session is
active; pending deliveries (queued for disconnected peers)
follow INBOX-like deliver-token TTL semantics.

**Reachability shape.** Tree-bound under
`system/network/{session_id}/...` and `system/outbound/{peer_id}/*`.

**Pitfall specific.** Pending outbound deliveries hold references
to EXECUTE payloads. When a session terminates (or `release-peer`
runs), pending outbound items become eligible per the cleanup
list in `release-result`. GC follows; don't reap pending
outbound items independently of the network handler's session
state.

### 4.11 TREE, TYPE, QUERY, CLOCK, COMPUTE

**TREE.** Tree-binding-and-listing operations only; not itself a
retention source. (Trie node hashes are referenced by REVISION
roots and tree bindings; live by ref.)

**TYPE.** Type definitions are protocol metadata. Treat as
forever-live unless explicitly uninstalled.

**QUERY.** Read-only; no retention.

**CLOCK.** Timestamps; no retention beyond their use in storage
entries.

**COMPUTE.** Handler graphs (`closure`, `scope` entities) live
while installed. The compute graph references value entities; GC
follows refs as usual.

---

## §5 Trigger patterns

When to actually run cleanup.

### 5.1 Pressure-driven

Monitor storage utilization. When utilization crosses a
threshold (say 80% of allocated quota), trigger GC.

**Pros.** Does work when needed; idle peers don't burn CPU on
empty sweeps.
**Cons.** Surprising latency spike when threshold is crossed
during a busy period.

### 5.2 Periodic

Run on a fixed interval (e.g., every 5 minutes). Legacy rs uses
this model with `interval_secs`.

**Pros.** Predictable cost. Workload-independent.
**Cons.** Wastes work on quiet peers. May lag pressure events.

### 5.3 Hybrid (recommended for most deployments)

Periodic floor at a long interval (e.g., every hour) to handle
slow accumulation. Pressure-driven override at a high threshold
(e.g., 90% utilization) to handle bursts.

**Pros.** Bounded latency in both quiet and busy regimes.
**Cons.** Two knobs instead of one.

### 5.4 Manual / on-demand

Operator runs GC explicitly via CLI / admin API. Useful for
debugging and for peers without quota visibility (some browser
contexts).

Per-substrate notes in §7.

---

## §6 Pinning conventions

Pinning is *not* a normative protocol feature. It's a deployment
convention. Several patterns work; pick one that fits.

### 6.1 Tree-namespace pins

Pinned hashes are recorded as tree entries under a deployment-
chosen path (e.g., `system/local/pinned/{hash}` with a small
metadata entity per pin). GC reads this path as an additional
root.

**Pros.** Pin set survives restart. Visible via standard tree
inspection. Capability-controllable via tree path patterns.
**Cons.** Adds tree-binding overhead per pin.

### 6.2 Config-file pins

Pinned hashes listed in a deployment config file outside the
tree. GC reads at startup.

**Pros.** No tree pollution. Easy to script.
**Cons.** Not introspectable through the protocol. No
capability scoping.

### 6.3 SDK-level pins

The SDK exposes `pin(hash)` / `unpin(hash)` API; the SDK
manages the pin set internally (could be either of the above).

**Pros.** Operationally clean for application developers.
**Cons.** Per-impl-SDK divergence in pin behavior.

### 6.4 Recursive pinning

IPFS borrowing: a recursive pin protects the hash AND everything
transitively reachable via refs. Useful when pinning a
"document root" without enumerating its parts.

Whichever pinning mechanism a deployment chooses, the pitfall
rules from §3 still apply. Pinning is *additive* — it expands
the live set, never contracts it.

---

## §7 Per-substrate notes

Each reference impl has its own storage substrate. Trigger
patterns and reaping mechanics differ.

### 7.1 sqlite (Go reference impl)

- DELETE statements reclaim row visibility but not file space.
  VACUUM reclaims file space. Schedule VACUUM separately from
  GC reaping — VACUUM is expensive.
- Foreign-key cascades can automate sidecar cleanup if the
  schema is set up for it.
- Use a transaction per GC batch; failures should roll back, not
  partially reap.

### 7.2 OPFS (browser peer)

- Browser quota is *adversarial* — the browser can evict the
  origin's storage independently. Defensive caching (re-fetch
  on miss) is mandatory.
- No VACUUM equivalent; file unlinks are immediate but quota
  accounting is browser-defined.
- Trigger on `navigator.storage.estimate()` rather than self-
  monitored utilization.

### 7.3 Filesystem (Rust reference impl, server peers)

- `unlink` is atomic; safe to reap mid-run.
- Filesystem metadata costs (inode pressure on huge stores)
  matter at scale; batch operations to reduce syscall overhead.
- Atomic rename to a "graveyard" directory followed by async
  delete avoids holding locks during expensive deletes.

### 7.4 Memory (test / ephemeral peers)

- All store entries are ephemeral by definition. Process
  restart is GC. Periodic GC for memory-pressure reasons is
  optional but cheap when implemented.

---

## §7a Delete is unbind-plus-intent, not erase — sensitive data

**Authors of L5 conventions and applications that collect sensitive
data (passwords, card numbers, CVVs, auth tokens, keys-in-flight,
PII) need to understand this.** The substrate has deletion
mechanisms — `tree:put(path, null)` unbinds; canonical deletion
markers (`PROPOSAL-DELETION-MARKERS`) handle distributed-
merge deletion semantics. But **delete is not erase** on this
substrate, for three structural reasons:

1. **GC is optional and non-normative** (§1). Unbinding a path does
   not reclaim the content-addressed bytes; the blob remains in the
   content store until an optional, impl-specific GC runs *and* no
   other reference holds it. Cleanup is local porcelain, not a wire
   guarantee.
2. **Revision history retains prior versions.** Deletion markers
   live in version tries, never in the live location index
   (`PROPOSAL-DELETION-MARKERS §1`). `put(secret, X)` followed by
   `put(secret, null)` leaves `X` in the version trie until
   `version_retention_window` prunes it. This is **by design** — F10
   proved "absence = deletion" causes cascading data loss, so
   markers exist precisely because absence cannot mean "forget."
3. **Propagation is irreversible.** Anything synced to another peer
   before deletion cannot be recalled. The deletion marker
   propagates the *intent*; a peer that already cached the content
   plus a prior version retains it per its own retention/GC.

**Net: a written-then-deleted secret can persist** (a) as
unreclaimed content-store bytes, (b) in local revision history for
the retention window, and (c) on any peer it reached. **Plaintext-
then-delete is not a secret-handling strategy on this substrate.**

**Two correct homes for secrets:**

- **Never-at-rest.** Data that should never become an entity (form
  submit payloads carrying card/CVV/password; transient assembly
  state) is held in **renderer or application memory only**, then
  dispatched directly via the wire — never `tree:put`-ed. Nothing to
  forget if it was never written. This is the available answer
  today. For example, the `site-form` convention in SITE v0.6
  assembles dispatch payloads in renderer memory and dispatches
  directly; sensitive fields never materialize as entities.
- **Encrypted-at-rest.** Durable but secret data that must live in
  the tree belongs in `EXTENSION-ENCRYPTION` — encryption is a
  property of the entity, "secure wherever it lives: memory, disk,
  transit, remote peer." The encrypted entity propagates via sync;
  any peer holding the ciphertext without the key holds opaque
  bytes. **DRAFT, W2 post-release** per `WORKSTREAMS.md`; not
  available today.

**For application authors targeting the research preview
or near-term post-release:** never-at-rest is the only available
answer for sensitive data. Encrypted-at-rest is the post-release
home for the durable-secret case. Plaintext sensitive data in user
trees, expecting delete to forget it, is unsupported by the
substrate and not a viable pattern.

The audit for an L5 author: *"if this data type goes through
`tree:put`, can it be (a) retained by GC declining to run, (b)
retained in version history, (c) propagated to peers I can no
longer reach?"* If yes to any, route the data through never-at-rest
or wait for encrypted-at-rest. The convention's `site-form`
conformance vector asserts in-memory submit-assembly precisely so
this discipline is testable cross-impl.

---

## §8 What this guide deliberately doesn't cover

- **Wire-level cross-peer GC coordination.** Each peer GCs
  locally. If peer A reaps a cache and peer B wanted it, B
  re-fetches. The grace window (§3.3) bounds the surprise; full
  coordination is out of scope.
- **Pinning capability scoping.** §6 lists deployment patterns.
  A deployment that needs role/identity-scoped pinning builds it
  with role-extension caps; not this guide's concern.
- **Tuning specific thresholds.** §5 names knobs; specific
  numeric defaults are deployment-dependent (workload, peer
  role, substrate). Don't enshrine numbers here.
- **Recovery from corruption.** If GC has run incorrectly and
  data is lost, recovery is a deployment-specific problem.
  Keep backups.

---

## §9 Extension-author GC posture checklist

**For every new extension** — and for every existing extension that hasn't done this audit yet — the author MUST declare the extension's GC posture as part of the spec. This is a cross-cutting discipline: every extension introduces entities, and every extension's entities interact with the live-set definition in §2. Without this declaration, the implementer has to reverse-engineer the GC contract from prose, which is exactly the bug class this guide exists to prevent.

The declaration goes in the extension spec (recommended location: a `§N. Lifecycle and retention` section near the end, or inlined into a `§N. GC and reachability` subsection of the spec's overview). It MUST answer:

1. **What entity types does this extension introduce?** List them. For each, name the path convention (`system/<ext>/<type>/{key}` or wherever they bind).

2. **Which of those entities are roots? Which are reachable only via refs?**
   - A root is something GC starts from. Roots are tree-bound, or they are runtime state the protocol explicitly treats as always-live (suspended continuations, in-flight chains).
   - Ref-reachable entities are reached *from* roots via the `refs` map. They survive GC iff something live references them.

3. **What retention semantics does this extension declare?** Name each retention knob (e.g., `version_retention_window`, `max_depth`, `dispatch_capability.ttl`). For each:
   - The config field name + type + default (default SHOULD be "unlimited" or "off" — most-conservative).
   - The operational considerations (the deploying-user-facing tradeoff: what does shrinking this knob cost / save?).
   - Specs expose knobs; deployments pick values. Don't enshrine specific numeric values in the spec.

4. **Cross-peer obligations.** Does any entity this extension owns get referenced cross-peer? If yes:
   - What's the worst-case sync lag between binding-removal and a peer following a stale ref?
   - Does the extension need its own grace window separate from the global `cross_peer_grace_seconds`? (Usually no; only call out if the workload differs meaningfully.)

5. **Cross-extension composition.** What other extensions' entities does this extension reference, and what entities of this extension do other extensions reference? Note retention coupling:
   - If your extension's entities reference REVISION versions, your retention should not be *longer* than REVISION's retention window for the same path (or you keep references alive that REVISION wanted retired).
   - If other extensions reference your entities (e.g., a role definition referenced by role assignments), call out the obligation explicitly so consumers know what holds them alive.

6. **Functional state vs historical metadata.** For each entity type, classify:
   - **Functional state** — the system relies on it to operate correctly. Loss = correctness break. Examples: current revision head, active capability chain, suspended continuation.
   - **Historical metadata** — observable but not load-bearing. Loss = audit gap, not functional break. Examples: closed-session audit entries, expired delivery records, retired-cert archive.
   - This classification drives deployment GC policy: aggressive deployments reap historical metadata, never functional state.

7. **What can be lost on cleanup, and what cannot.** State explicitly:
   - **Cannot lose:** the functional-state set from #6, plus anything other extensions transitively depend on.
   - **Can lose:** historical metadata past retention; sidecars whose targets are reaped; cached cross-peer state past grace.
   - **MUST coordinate before lose:** anything cross-peer-referenced — even if locally GC-eligible, signal the binding removal before reaping the content (see §3.5).

8. **Conformance impact.** Does this extension's GC declaration introduce any cross-impl observable behavior? If yes, state which behavior is conformance-tested. (Most extensions' GC is locally-observable only — cross-impl divergence is sync-lag, not corruption — so the answer is usually "none.")

**Format note.** This checklist is questions, not a template. Answer it in spec prose, not as a checkbox table. The point is for the implementer reading the spec to understand the GC contract without having to derive it.

**Retroactive sweep.** As of this guide's ingestion, no current extension spec has this declaration. The sweep is an open project (not corridor-blocking). When sweeping: existing extensions usually do declare retention semantics in scattered prose; the sweep consolidates into one section per spec. New extensions MUST declare during authoring.

`GUIDE-EXTENSION-DEVELOPMENT.md` §3.6 references this checklist as a step in the extension-authoring discipline.

---

## §10 If you remember nothing else

Three sentences:

1. **Live = anything in the seven-root set + transitively
   reachable via refs.**
2. **Pitfall #1: don't reap a hash referenced by any live
   entity, even if no tree path binds it.** (Walk refs, not
   paths.)
3. **Each extension declares its own retention window — GC reads
   those windows, never overrules them.**

The rest is mechanism.

---

*Draft v1.0, ingested from an internal meta-repo draft. The §3.3 specific-number suggestions have been reframed as knobs + operational considerations; §9 (extension-author checklist) is new this round. Original §9 TL;DR is now §10.*
