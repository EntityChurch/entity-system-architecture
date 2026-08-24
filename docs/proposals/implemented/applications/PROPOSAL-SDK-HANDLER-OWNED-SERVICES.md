# PROPOSAL — handler-owned services (the registration lifecycle contract)

> **Status: RATIFIED + FOLDED (2026-07-29)** — landed as `specs/sdk/SDK-OPERATIONS.md` **v1.11, §11.6.9**
> (+ a forward pointer at §11.6.2). Open item 1 — the manifest declaration's field shape — was **ruled first**
> (§6.1 below) and folded with the rest. **The spec is source of truth from here**; this document is the design
> record. **Gate 1 of the v1 punch is closed.**
> **Not validated:** no implementation has built against §11.6.9. `PROPOSAL-CONNECTION-NODE`'s unwrapped surface
> is the intended first proof, and §7's criteria are what actually close it — folded ≠ proven.

**Status:** **RATIFIED + FOLDED 2026-07-29** — landed as `specs/sdk/SDK-OPERATIONS.md` §11.6.9 (v1.11), including the §6.1 field-shape ruling. *Mis-filed as DRAFT until 2026-08-13: the fold landed and this header was never updated, so it sat in the active set for two weeks. The spec's own changelog said "Folds PROPOSAL-SDK-HANDLER-OWNED-SERVICES" the whole time.* **Build state (observed 2026-07-31, and NOT re-verified since):** no service declaration in any of the three reference impls — re-measure before citing.
**Target:** `specs/sdk/SDK-OPERATIONS.md` — a lifecycle addition to **§11.6** (dynamic handler registration) plus a
clarifying note in **§11.3** (handler execution models). Additive; no V7/wire change, no new entity type, no new
capability, no keystone delta.
**Scope:** the contract for a handler that **owns a resource for as long as it is registered** — a listener, a
background loop, a watcher — as opposed to one that only answers dispatch. Names and bounds a pattern six
extensions already implement by hand.
**Driver:** `PROPOSAL-CONNECTION-NODE` is the first extension we intend to build *as a slot-in extension* that
needs this, and the first that deliberately owns a **non-capability-gated** service. It is the forcing function,
not the reason — the gap predates it.

---

## 1. The gap

The handler abstraction's promise is "slot any functionality into a peer." Today it delivers that for
**request/response** and not for **resource ownership**.

What exists (pinned to `entity-core-rust` `ae93443`, and mirrored in go/py):

| Surface | Contract | Lifecycle for an owned resource |
|---|---|---|
| `Handler` trait (`core/handler/src/lib.rs`) | `handle` / `pattern` / `name` / `operations` / `internal_scope` | **none** — purely request/response |
| `register_handler`, `RegisteredHandler` (`bindings/sdk/src/register_handler.rs`) | writes interface + handler entities + grant, binds a body in the dispatch index; `close()` reverses it | **none** — dispatch only |
| SDK-OPERATIONS **§11.3** three execution models | precompiled / SDK language-native / entity-native | all three describe **where the body lives and how dispatch resolves it** |
| SDK-OPERATIONS **§11.6.2 / §11.6.3** | close ordering, idempotence, body concurrency, cancellation, internal scope | nothing about a resource the handler holds |

Yet resource-owning extensions are everywhere: `entity_clock::engine::ClockEngine`,
`entity_subscription::engine::Engine`, `extensions/inbox`, `extensions/history/src/engine.rs`,
`extensions/local-files/src/watcher.rs`, and `NetworkHandler` + its `PeerLink` seam
(`extensions/network/src/lib.rs`). None of them acquires its lifecycle through the handler contract. They get it
from **`PeerInner::start_engines` (`core/peer/src/lib.rs`) — a hardcoded call site that constructs and starts each
engine by name**, with `system/network` further requiring core/peer to retain its handler so `start_engines` can
inject `PeerLink`.

**So the only way to ship a service-owning extension today is to be a privileged, in-tree built-in that core/peer
knows by name.** An out-of-tree or optional extension cannot express it at all. That is the gap.

## 2. What this proposal is *not*

It is **not a new execution model** and **not a new hook type invented to paper over a gap** (the failure mode
`entity-core-rust/AGENTS.md` names and routes upstream — this document is that routing). The 1:1 pairing of a
handler with a long-lived engine **already exists six times over**; §11.3's model taxonomy simply never named the
axis. This proposal names it and bounds it. If the shape were wrong, six extensions would not already have
converged on it.

**The axis §11.3 is missing.** Its three models vary *where the body lives*. Orthogonal to that is *whether the
handler owns anything between dispatches*:

|  | Answers dispatch only | Also owns a resource |
|---|---|---|
| **Precompiled** | `system/tree`, `system/handler` | clock, subscription, inbox, history, network — **hand-wired in `start_engines`** |
| **SDK language-native** | application handlers (§11.6 today) | **not expressible** ← the gap |
| **Entity-native** | compute-backed handlers | out of scope (an expression owns nothing) |

## 3. The proposed contract

A handler MAY declare itself **service-owning**. If it does, registration and close acquire two obligations:

1. **Start.** The peer invokes the handler's start step as part of registration, after the tree writes of §11.6.1
   succeed and before the pattern accepts dispatch. A start failure MUST be treated as a registration failure and
   MUST trigger the §11.6.4 partial-failure compensation (the tree writes are rolled back). A handler MUST NOT be
   dispatchable with a failed service.
2. **Stop.** `close()` stops the owned service **before** unregistering dispatch, extending the §11.6.2 ordering
   to three steps: *stop service → dispatch index → tree entries*. Stop MUST be idempotent, and MUST NOT be relied
   upon to run via GC or finalizers (§11.6.2's existing rule). Restart-survival follows the model's existing rule:
   an SDK language-native service-owning handler is re-registered (and so restarted) by the application on
   startup, exactly as §11.3 model 2 already requires for the body.

**Declaration is mandatory and is a tree entity.** A service-owning handler MUST declare the fact — and the
shape of what it owns. It is not enough to spawn something in the body. This is the whole safety story (§4).
*(This originally said "in its `system/handler` entity"; that is a **core-protocol type** and was corrected at
fold — see the note at §6.1. As landed, the declaration is `system/runtime/owned-services/{handler-pattern}`.)*

## 4. The boundary rule — declared, not gated

A handler-owned service is frequently **outside the capability model by construction**. The connection node in its
public mode is the motivating case: that handler's entire job is to run an open service the entity system does not
mediate at all — no grant, no handshake, no entity encoding, clients doing no entity work whatsoever
(`PROPOSAL-CONNECTION-NODE` §2). That is deliberate: the introduction service is public infrastructure, so a grant
handshake in that flow buys nothing but costs a round trip per request.

It is a legitimate design, and exactly why the boundary rule matters: "slot in any functionality" here literally
includes "slot in something that answers the open network outside the capability model." An operator must be able
to discover that by inspection, not by reading the implementation.

The rule this proposal adopts:

> **A handler-owned service is declared and auditable, never ambient. Declaring it does not capability-check its
> traffic; it makes the hole visible.**

Concretely:

- **The declaration entity names the service** — that one exists, and what class of resource it holds (a listener and its
  bind address / port, a background loop, a filesystem watcher and its root). An operator or an auditing peer can
  enumerate *what is listening on this peer and which handler owns it* by walking the tree, with no out-of-band
  knowledge. **Where the service is reachable from outside the peer and not mediated by the capability model, the
  declaration MUST say so** — an operator cannot assess exposure from "this handler owns a socket" alone. The
  connection node's public mode is precisely this case.
- **Admission is peer policy, not a handler's unilateral choice.** Whether a peer permits service-owning handlers
  at all — and whether it permits ones that bind network listeners specifically — is peer configuration. A peer
  that declines simply fails the registration. A desktop app peer and a deployed connection node want opposite
  defaults, and neither should be the handler's decision.
- **No new capability type.** Admission is a peer-config predicate over the declaration, not a grant in the
  capability graph. Adding a capability kind here would imply the service's *traffic* is capability-mediated,
  which is exactly what it is not — that would be a misleading abstraction, worse than an honest declaration.

**The distinction being preserved:** the entity system's guarantee is about **entity operations**, not about every
byte a process touches. A handler-owned service is an admission that the peer process does something the entity
model does not mediate. Making that declarable keeps the guarantee honest; leaving it ambient quietly weakens what
"capability-checked" means.

## 5. Cross-impl surface

The contract binds **all three impls** (go/rust/py), because the punch half of the connectivity work is
service-owning in every language — a peer binds a local socket, runs a `fire_at` simultaneous-open, and hands a
live transport to the peer's transport layer, structurally the same seam `NetworkHandler`'s `PeerLink` occupies
today. So this is not Rust-only infrastructure plumbing; it is a genuine cohort surface.

The pieces that must agree across impls:

- **Ordering** — start after tree writes and before dispatch; stop before dispatch-unregister. A peer that stops
  in the wrong order can dispatch to a handler whose service is already gone.
- **Failed-start semantics** — registration fails and compensates; never a dispatchable handler with a dead service.
- **The manifest declaration shape** — this is the only part that is cross-peer *observable* (a remote peer or
  auditor reads it), so per the repo's interop discipline it is the part that MUST be pinned rather than left to
  converge.

Language-idiomatic *expression* of start/stop is deliberately unpinned (a Rust `impl Drop`, a Go `Close() error`,
a Python `async with` — §11.6.2 already sets this precedent).

## 6. Open items

| # | Item | Disposition |
|---|---|---|
| ~~1~~ | ~~The manifest declaration's exact field shape~~ | **RULED 2026-07-29 — see §6.1. Both**: a closed enum on the two axes an auditor gates on, a free-form descriptor for the rest. |
| 2 | Whether a failed *stop* (a service that won't shut down) blocks close or is best-effort with a logged warning | leaning best-effort + loud; a stuck service must not wedge deregistration |
| 3 | Whether the six existing precompiled engines are retrofitted onto this contract or left hand-wired | **retrofit is out of scope here** — prove the contract on the new extension first, retrofit as a follow-on if it holds |
| 4 | Whether entity-native (model 3) handlers may ever be service-owning | leaning no — an expression owning a socket breaks transferability, which is model 3's entire point |

### 6.1 The manifest declaration shape `[RULED 2026-07-29 — the cross-peer-observable surface]`

The open item asked "resource class enum, or free-form descriptor?" **The answer is both, split by who reads
it** — and the split is the whole ruling:

> **Pin a closed enum on the two axes an auditor makes a decision on. Leave everything else free-form, because
> nothing gates on it.**

> **Corrected at fold (2026-07-29, audit).** This section originally put `owned_services` as a field on
> **`system/handler`** — which is a **core-protocol type** (ENTITY-CORE-PROTOCOL.md §3.7) and therefore not
> this repo's to extend, the identical error the NAT `observed_address` HELLO field was ruled into two days
> earlier. It was also the handler-**private** entity, invisible to the remote auditor §4 requires to read it.
> **As folded (`SDK-OPERATIONS` §11.6.9) the declaration is a standalone `system/runtime/` entity** at
> `system/runtime/owned-services/{handler-pattern}` — no core change, and its own grant surface. The field
> *shape* ruled below is unchanged and is what folded; only its home moved.

```
system/runtime/service-declaration := {
  fields: {
    kind:       {type_ref: "primitive/string"}   ; CLOSED enum — see below
    exposure:   {type_ref: "primitive/string"}   ; CLOSED enum — see below
    descriptor: {map_of: {type_ref: "primitive/any"}, optional: true}
                ; free-form, kind-specific, diagnostic. Nothing gates on it.
  }
}
```

**An array, not a single declaration** — a handler may own more than one service, and the connection node already
does: the unwrapped listener *and* its TTL reaper (`entity-core-rust`, 2026-07-29). A singular field would have
forced the first impl to either under-declare or invent a convention.

**`kind` (closed enum, kebab).**

| Value | Means |
|---|---|
| `network-listener` | binds a socket and accepts connections |
| `background-loop` | a timer / reaper / periodic task holding no external surface |
| `filesystem-watcher` | watches a path outside the entity tree |

**`exposure` (closed enum, kebab) — this is the field §4 exists for.**

| Value | Means |
|---|---|
| `peer-internal` | not reachable from outside the peer process at all (every `background-loop`) |
| `entity-mediated` | reachable from outside, but every request is capability-checked like any entity operation |
| `unmediated-public` | **reachable from outside and NOT mediated by the capability model** — the connection node's public mode (`PROPOSAL-CONNECTION-NODE` §5.1) |

> **MUST.** A declaration with `exposure: "unmediated-public"` **MUST** carry the bind address and port in
> `descriptor`. That is the minimum an operator needs to assess exposure, and §4's requirement — *"an operator
> cannot assess exposure from 'this handler owns a socket' alone"* — is unsatisfiable without it.

**Why closed on these two and free-form on the rest.** The auditing question is *"is there a hole in this peer,
and how big"*, and it must be **machine-answerable identically in three implementations** — that is the definition
of a cross-peer-observable surface, which this repo's interop discipline says to pin rather than let converge. A
free-form descriptor cannot answer it: an auditor would have to pattern-match prose, and three impls would write
three different prose conventions that all look fine in isolation. Conversely *which* port, *which* path is
**diagnostic** — nothing decides on it — so pinning it would be ceremony that ages badly as service kinds appear.

**Unknown values MUST fail loud, not default.** A reader encountering a `kind` or `exposure` value it does not
recognize **MUST NOT** treat it as benign — it MUST surface it as unassessable. This is a deliberate departure
from the ecosystem's MUST-ignore-unknowns rule ([ADR-0002]), and the departure is the point: MUST-ignore is
correct for *wire extensibility*, where an unknown field is something you did not need. Here an unknown value is
**an exposure you cannot characterize**, and silently ignoring it reports "no holes" for a peer that may be
serving the open internet. The safe default for extensibility is the unsafe default for auditing.

**Adding a `kind` or `exposure` value is a spec change**, not a local extension — that is what "closed" means, and
it is what makes the enum worth having.

## 7. Validation

The contract is proven by `PROPOSAL-CONNECTION-NODE` being built **as a slot-in extension** rather than a
hand-wired built-in — specifically:

1. The Rust signaling extension registers, starts its service, serves traffic, and `close()` returns the peer to a
   clean state with nothing still bound.
2. A failed start compensates the tree writes and leaves no dispatchable handler.
3. The declaration is readable from the tree by another peer.
4. Go and py peers express the same contract for the punch's socket ownership.

If the connection node cannot be built this way, that is the finding — and it is worth more than the feature.

## Pointers

- **Driver:** `PROPOSAL-CONNECTION-NODE` (§3 staging, §1 the non-entity hot path).
- **Target spec:** `specs/sdk/SDK-OPERATIONS.md` §11.3, §11.6.1–§11.6.4.
- **Evidence:** `entity-core-rust` `ae93443` — `Handler` (`core/handler/src/lib.rs`), `PeerInner::start_engines`
  (`core/peer/src/lib.rs`), `register_handler` / `RegisteredHandler` (`bindings/sdk/src/register_handler.rs`),
  `NetworkHandler::bind` + `PeerLink` (`extensions/network/src/lib.rs`).
- **Sequence:** `HANDOFF-2026-07-28-connection-node-staging-and-sequence.md`.
