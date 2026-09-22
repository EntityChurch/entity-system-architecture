# Guide: Bridge Extension Development

**Status**: Active
**Audience:** Authors of an extension that connects the entity system to a technology outside it — a filesystem, a version-control system, a package store, a mail system, a web origin.
**Scope:** The discipline every bridge shares. The mechanics of a specific bridge live in its own specification; this guide is what an author should understand before starting one. **It is non-normative.** Where it and a bridge specification disagree, the specification wins.

**Read `GUIDE-EXTENSION-DEVELOPMENT.md` first.** A bridge is an extension, and everything that guide requires of an extension it requires of a bridge. This guide adds only what is true because the other side of the boundary is not the entity system.

---

## 1. What a bridge extension is, and the three layers

A deployment meets the outside world at three separable layers. **Keeping them separate is the single most useful thing in this guide**, because every recurring confusion in this area is two of them being treated as one.

| Layer | What it holds | Pluggable? |
|---|---|---|
| **Protocol** | Envelope, signature, capability chain, identity, entities. | **No.** Frozen across every transport and every bridge. |
| **Transport** | How bytes get from one peer to another — live, async-queued, polling-static, mesh-pull, one-way-authenticated, routed-circuit, physical. | Yes, per-peer transport profiles. |
| **Adapter** | **Bridges.** Translating a foreign system's idioms into protocol calls. | Yes, one extension per bridge. |

**The transport layer moves entity bytes between peers. The adapter layer turns something that is not an entity into one, or an entity into something that is not.** A transport's counterparty runs the protocol; a bridge's counterparty has never heard of it.

### 1.1 One protocol can legitimately appear at two layers

The case that makes this area look confused is the one the model handles cleanly. A ubiquitous protocol — the web's, most obviously — shows up **twice**, and the two appearances are different things:

- **As a transport binding.** Envelopes moving between peers over a polling-static reachability shape. The bytes on the wire are already entity-encoded; the substrate's only job is to carry them. A peer-to-peer concern.
- **As a bridge.** Acting as an ordinary client against arbitrary third-party servers, or serving ordinary clients that have no idea an entity system is behind the surface. An interop-with-the-outside concern.

**These compose rather than conflict — a bridge may run over a transport, including a transport speaking the same protocol.** The model places the protocol twice on purpose.

⇒ **The disambiguation question, asked before you author anything that fetches:** *are the bytes on the wire already entity-encoded, or are they foreign content that needs wrapping?* The first is transport and belongs in the transport specification; the second is a bridge and belongs here. They are not one mechanism scoped two ways. `GUIDE-EXTENSION-DEVELOPMENT.md` §3.7 states the discipline for the web protocol specifically, and records that two independent analyses once merged the two before anyone caught it.

> ⚠ **A first example that is simultaneously infrastructure is a bad teacher.** The web protocol makes the general shape look like a special case, because it is also how peers reach each other. **A version-control or package-store bridge teaches the category better** — neither is load-bearing for the protocol itself, so nothing about them is confusable with transport.

## 2. One extension per bridge, and no umbrella

**There is no general bridge extension, and adding one is not a deferred item.** *Bridge is a development-time category, not a runtime surface.*

Unification at the all-bridges level was examined and rejected, and it fractures on four axes at once:

- **Trust models differ.** A filesystem you own is not a third-party server you do not.
- **Idempotency differs.** Re-reading a file and re-sending a message are not the same repeated operation.
- **Capability flows differ.** Some bridges receive authority from the foreign system; some project authority onto it; some do both in different directions.
- **Failure surfaces differ.** "Not there", "not reachable", "reachable and lying" are distinct in different proportions per technology.

**A shared runtime across those would be a lowest common denominator that fits nothing.** What the family genuinely shares is *knowledge* — §3 through §5 of this guide — and knowledge belongs in a guide, not in code.

## 3. Direction is a mode, not a second extension

Egress (the system acting on the foreign world) and ingress (the foreign world reaching the system) **flip the threat model, the direction of capability flow, the idempotency properties and the failure surface.** That is a large difference, and it still does not justify two extensions.

**Each bridge is one extension with install-time modes: `egress-only`, `ingress-only`, or `both`.** The client direction and the server direction are the same specification configured differently.

⇒ **A bridge's conformance section covers all three configurations**, including the negative case: an `ingress-only` deployment MUST reject an egress operation with a clear capability error, not with a generic failure and not by silently succeeding.

> ⭐ **What this says about what an extension is, because the question is load-bearing here.** An extension is a coherent **specification** surface — what it contributes: types, operations, storage conventions, a conformance section. It is not a unit of installation. What a deployment installs is that surface **in a declared mode**. ⇒ **Half a bridge is a conformant deployment, not a partial install.** Saying so explicitly is what stops the mode axis from being read as a licence to ship an arbitrary subset.

## 4. The seven questions every bridge specification answers

**All seven, in the specification, explicitly.** They were extracted from the filesystem bridge, which is the family's prior art — it answers them, and reading how is the fastest way to calibrate what a sufficient answer looks like.

1. **Authority assignment.** Who may act on the foreign system, and on whose behalf? The foreign system has its own notion of who is acting; state the correspondence.
2. **Capability translation, both directions.** Protocol capability → foreign permission, and foreign permission → protocol capability. Both directions, even when one of them is "none, and here is why."
3. **Namespace convention.** Where do the foreign system's objects land in the tree, and what is the mapping rule? A reader must be able to predict the path for an object they have not seen.
4. **Sync-state visibility.** ⭐ **A bridge introduces a third state beyond *have* and *do-not-have*** — *fetching*, *stale*, *remote-unreachable*. **It must be observable rather than inferred.** A consumer that cannot distinguish "not there" from "not reached yet" will invent a wrong answer, and this is the question most often skipped.
5. **Cache invalidation.** When does a held copy stop being an answer? State the trigger, not just the fact that one exists.
6. **Hash-identity translation.** A foreign identifier is not a content hash. State the mapping, and state what happens when the foreign system changes the bytes under a stable identifier.
7. **Idempotency semantics on egress.** What does a repeated outbound operation mean? "Exactly once" is rarely available across a foreign boundary; say what you actually offer.

## 5. Hazards that recur at every foreign boundary

- ⛔ **External path resolution is adversarial input.** The symlink-escape class is the well-known instance, and it generalizes: version-control refs, redirect chains and package-store path resolution all surface the same shape — *a name the foreign system resolves, on its rules, to a location you did not sanction.* Resolve and then re-check containment; never check the name.
- ⛔ **Time-of-check / time-of-use under external mutation.** The foreign system changes without telling you, and between your check and your use is a window you do not control. Design so that a stale check degrades to a refusal rather than to a wrong success.
- **Boundary capability discipline is explicit, never implicit-by-reference.** A bridge holds different capability sets on each side by construction. `GUIDE-PEER-COMPOSITIONS.md` §5.6 covers the composition-level form of this and flags the cycle risk, which is the highest of the seven compositions.
- **The foreign system's failure vocabulary is not yours.** Map it deliberately. An unmapped foreign error reaching a protocol consumer as a generic failure destroys the sync-state visibility §4's fourth question requires.

## 6. Naming, placement, and namespace

| | |
|---|---|
| **Document name** | The `EXTENSION-` name class, then `BRIDGE`, then the technology's own name — see the note below |
| **Home** | `specs/bridge-extensions/`, flat |
| **This guide's counterpart** | `GUIDE-EXTENSION-DEVELOPMENT.md` — read it first; a bridge is an extension |

> **Worked names (none of these is authored — every one is a forward reference):** a version-control
> bridge would be `EXTENSION-BRIDGE-GIT.md` *(planned)*, a package-store bridge
> `EXTENSION-BRIDGE-NIX.md` *(planned)*, a web bridge `EXTENSION-BRIDGE-HTTP.md` *(planned)*. **There
> is deliberately no umbrella document with no technology in its name**, for the reasons in §2 above.

**Why a separate directory from `specs/extensions/`, when a bridge is an extension.** The directories group by what a specification is *about*, and the distinction is for the reader: **`specs/extensions/` is the entity system's own surface — what a peer is and what a deployment needs. `specs/bridge-extensions/` is how the system reaches technologies that are not it.** A reader opening the first should find the system; a reader opening the second should see a spread of unrelated foreign technologies and understand immediately that that spread is the point. Mixing them invites a reader to conclude that the system *is* the technologies it happens to bridge.

**The name class still says `EXTENSION-`**, because the artifact genuinely is one, and because the citations that already name these documents use that spelling.

### 6.1 The namespace question is open, and the first bridge decides it

⛔ **A new bridge author will find three plausible prefixes in the corpus and no rule.** State the one you choose and why, and expect it to be reconciled:

- `bridge/` is **reserved** for external system connections in `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1 and named as a reserved prefix in `SDK-OPERATIONS.md` — **and it has no occupants.**
- `system/bridge/...` is what the corpus's forward references to the web bridge actually spell, for both the entity type and the capability.
- `app/bridge/...` is what the version-control bridge sketch in `EXTENSION-REVISION.md` §10.2 spells.

⇒ **Reserving a prefix and specifying one are different acts, and only the second leaves a document** — which is how a reserved prefix loses its declared subject without anything noticing. Do not treat any of the three as settled by precedent.

### 6.2 What we are standing on versus what we are reaching out to

**The built world is not one category, and a taxonomy with one bucket for both will keep feeling wrong.**

- A **filesystem** or **process** bridge is *assumed present*. The system runs **on** it. It is substrate as well as adapter, and it is not an optional integration a deployment may or may not have.
- A **version-control**, **package-store** or **mail** bridge is an optional integration *reached across a boundary*.

**That distinction is real, it is what the existing prefix split encodes, and it is why the assumed-present bridges attract different naming from the reached-across ones.** State which kind yours is in its overview section.

### 6.3 Naming a bridge for where it was first built

⚠ **The family's existing member is named for the deployment it was first prototyped against rather than for the technology it bridges** — and a name that asserts *locality* is falsified the first time the same interface reaches a network-mounted filesystem. **Name a bridge for the technology, not for the deployment shape you happened to build against first.** The general case arrives later and the name does not stretch.

**Where an existing name already shipped, the correction is additive: a new bridge for the general case, alongside the old one, which is deprecated once consumers have somewhere to go.** A rename across independent implementations buys comprehensibility and costs coordination; a successor buys the same comprehensibility and costs nobody anything until they choose to move.

## 7. What is not settled

**Named here so a bridge author meets them in a guide rather than in a deployment.**

- ⚠ **Resource contention between a bridge and a transport over the same substrate.** A peer running both a web transport and a web bridge server may want the same listening port. **Two protocols' worth of semantics over one substrate, and nothing in the corpus adjudicates it.** It belongs in the per-bridge specification; there is no general answer yet.
- **Whether a web-protocol bridge is one extension or a stratified substrate-plus-conventions shape.** The web bridge is the one candidate member where the single-extension answer has been questioned, precisely because it is simultaneously infrastructure (§1.1). Re-examine when its specification hardens.
- **The namespace prefix (§6.1).**
- **Which bridges are worth specifying first.** Version control and package store are the strongest candidates — both are structurally better teachers than the web protocol, for the reason in §1.1.
