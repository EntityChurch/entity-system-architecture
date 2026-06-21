# Guide: Building an Extension — When, Why, and How

**Status**: Active

**Audience**: Anyone proposing or authoring a new extension in this codebase — architecture team, implementation teams, third-party extension authors.

**Premise**: The system is designed to be extended. The substrate is small on purpose; capability-scoped extensions are how it grows. But extensions accrete differently from the substrate — they accrete by *pattern matching* rather than *first-principles derivation*. This guide is the discipline that keeps extensions from becoming the substrate's drag.

---

## 1. What an extension is

An extension is a coherent surface — types, handlers, conventions, conformance — that **composes onto** the entity core protocol (`ENTITY-CORE-PROTOCOL.md`) without changing it. Extensions are installed per peer; a peer that does not install an extension does not see its types or its handlers. The core protocol does not depend on any extension.

Two roles for extensions today:

- **Substrate primitives.** Extensions that are themselves the foundation for other extensions (`EXTENSION-ATTESTATION`, `EXTENSION-QUORUM`). The "substrate" framing means they are foundational for others; they remain extensions in the sense that the core protocol does not depend on them.
- **Standard / application extensions.** Extensions that solve a concrete deployment problem above the core protocol (`EXTENSION-INBOX`, `EXTENSION-REVISION`, `EXTENSION-CONTINUATION`, `EXTENSION-IDENTITY`, `EXTENSION-GROUP`, `EXTENSION-ROLE`, `EXTENSION-SUBSCRIPTION`, `EXTENSION-CONTENT`, `EXTENSION-QUERY`, `EXTENSION-NETWORK`).

A third category exists as a placeholder for *withdrawn* designs:

- **Exploratory.** Reference designs preserved with a strong "not actively developed" status header. `EXTENSION-DURABILITY` is the first instance. Exploratory extensions are not normative for anything; no deployment is required to install them; they are kept on disk because the analysis behind them may be useful if a concrete driver emerges.

---

## 2. The substrate / userspace boundary

Before authoring an extension, locate the proposed work on this axis:

- **Substrate.** The protocol's bare minimum: entity, hash, path, capability chain, signature, content addressing, universal namespace, EXECUTE/EXECUTE_RESPONSE. If you find yourself amending V7, you are touching substrate. The bar is very high.
- **Substrate primitives.** Attestations, quorums. Things other extensions need to depend on.
- **Standard extension.** Solves a concrete deployment problem above substrate. The bar is "name the deployment."
- **Userspace.** Solved by an *application* installing extensions and writing handlers; not part of any spec in this repo.

**The most common scope-overreach is treating userspace concerns as standard-extension concerns.** Durability is the canonical example: a durability *property* (the request was preserved) is something an application can verify in many ways — by reading content-addressed state, by subscribing to changes, by application-level replication policy. Treating it as a *system extension* required inventing durability levels, status branching, advertisement, zone characteristics — apparatus to support a property that didn't need apparatus.

A working heuristic:

> **If the property can be observed by reading the tree, it does not need a system-extension contract to deliver it.**

The entity tree is content-addressed and capability-scoped. Most "is the data there?" / "did this happen?" / "what's the current state?" questions answer themselves against the tree without a new extension. The cases that need new extensions are the ones where the *operation* itself doesn't fit substrate verbs (versioning, async delivery, capability chains, group membership, role-based access).

---

## 3. The pre-merge checklist

Before any extension or proposal lands normative, work this list. Answer each in writing in the proposal. Vague answers fail the check.

### 3.1 The deployment-driver test

**Question.** Name the specific deployment that does not work without this. Not "a deployment of this shape might want…" — name one.

**Pass.** "The `entity-core-go` CLI tool sets `deliver_to` and the result vanishes silently; no way to tell whether the receiver accepted it. The reactive `:re-derive` flow in EXTENSION-ROLE depends on subscription delivery." Specific, file:line-grounded ideally.

**Fail.** "A Kafka-like deployment might want to know whether the message was preserved." Hypothetical; no concrete consumer.

### 3.2 The borrowed-vocabulary test

**Question.** Does this proposal introduce names that come from a category of system (logs, RDBMSs, queues, RPC frameworks, etc.) the project is not trying to be?

**Pass.** Borrowed *concepts* derived from first principles, with first-principles vocabulary. Example: REVISION's three-way merge uses the concept from version control but the vocabulary is rooted in the system's own terms (`prefix`, `version-trie`, `binding`).

**Fail.** Borrowed *vocabulary* directly. Example: durability with `requested` / `applied` / `committed` / `max_available` levels — this is direct vocabulary import from "message-queue durability." Same with isolation levels (`snapshot` / `serializable`) — direct RDBMS import.

### 3.3 The apparatus / property ratio test

**Question.** Compare the surface area of the apparatus (types, status codes, conformance rules) to the surface area of the property delivered.

**Pass.** Apparatus ≈ property. EXTENSION-INBOX's pre-§10 surface delivered async delivery via one handler, one type, one extension field. Tight.

**Fail.** Apparatus >> property. Durability §10: eight MUSTs, ten types/fields, four pinned reason strings, zone-characteristic framing, eight-row status table — to deliver "respond instead of silently dropping the request." Property could have been delivered by one V7 sentence.

### 3.4 The substrate-vs-userspace test

**Question.** Could this be built *on top of* the system by someone who needed it, using extensions they author themselves?

**Pass-as-standard-extension.** No — building it on top requires duplicating substrate primitives. (Example: REVISION must be a system extension because cross-peer merge requires coordination with the content store and the universal namespace.)

**Pass-as-userspace.** Yes — an application can build this using extensions it authors. Move it to userspace; don't gate the system on it.

**Fail.** Couldn't tell. The answer is unclear because the proposal hasn't distinguished apparatus from property. Re-derive from first principles before continuing.

### 3.5 The first-principles derivation test

**Question.** Can this be derived from entity + hash + path + capability + signature + universal namespace, without importing vocabulary from outside the system?

**Pass.** The derivation works using the system's own terms. EXTENSION-CONTINUATION amendments 1–8 all derive from "the chain-construction site is where the signature path matters" — first-principles, system-own.

**Fail.** The derivation requires loading vocabulary from outside. Durability levels (`stored` / `replicated` / `committed`) aren't derived from the system; they're imported from the durability-property category.

### 3.6 The cross-impl-convergence-is-necessary-not-sufficient test

**Question.** If three impl teams ratify this spec, what does that prove?

**Pass.** It proves the spec text is *internally coherent and cross-impl determinable*. Useful, but does not answer whether the property belongs in the system.

**Fail.** Treating cross-impl convergence as proof the proposal was the right work. Three teams converged on Amendment 1 of the durability spec — that did not stop the retraction.

**Implication.** Apply tests 3.1–3.5 *before* the cross-impl validation phase. The cross-impl validation answers a different question (do we agree on what the spec says); it does not answer the gate-question (does the spec say something that belongs).

### 3.7-pre. Three legitimate reasons to spec something

Tests 3.1–3.5 originally treated "no concrete deployment driver" as a near-automatic fail. A later review surfaced a wider framing — there are **three** legitimate reasons to spec an extension, not one:

1. **Irreducible technical requirement.** The system fundamentally cannot function without it. Pass 3.1 with a concrete deployment / inevitability answer.
2. **Operational requirement.** Production multi-peer deployments need it. Pass 3.1 with a production-scenario answer.
3. **Anti-fragmentation grounding.** A major CS concept (transactions, encryption, etc.) the community will expect the system to address. Without a coherent first-pass design, downstream implementers will invent four competing semantics. Pass 3.1 with an "expected community demand + no existing coverage" answer.

The first two are clean fails when missed. The third **requires both halves**: expected community demand *and* no existing coverage through composition of existing extensions. Durability had the first half (community would expect a durability story) but failed the second half (transaction + revision+sync + inbox+continuation already covered the space — see 3.7 overlap-coverage). Transaction has both halves (large-scale-consensus / Raft future + no existing multi-binding-atomic mechanism).

The right disposition for anti-fragmentation work: **a first pass is good even if wrong.** The architecture team's job for this tier is establishing coherent grounding, not certainty. Convergence happens through community use over time.

### 3.7 The overlap-coverage test

**Question.** Does this proposal deliver a property reachable through composition of existing extensions?

**Pass.** No — the property requires a new mechanism, not a new composition of existing ones. *Example:* `REVISION`'s three-way merge is not constructible from `CONTENT` + `SUBSCRIPTION` + handler code; it requires the version-trie machinery.

**Fail.** Yes — the property is already reachable in pieces. *Example (the canonical fail):* `EXTENSION-DURABILITY` tried to specify "store reliably and find again by id." Atomic preservation: `TRANSACTION` (multi-binding atomic write with CAS + rollback). Replicated preservation: `REVISION` + sync. Async preservation: `INBOX` + `CONTINUATION` (the request stays in the tree, retrievable by `(author, request_id)`). The space was already covered three ways before durability tried to specify a coordinating contract on top.

**Implication.** When a proposal fails this test, the right deliverable is usually a *guide* — describing how to compose existing extensions to get the property — not a new extension. The guide carries the *learning* without the *apparatus*.

**A subtlety.** This test is not "is the property useful" — properties are usually useful. It's "is the *apparatus that wraps the property* the minimal way to deliver it." Useful properties don't always need new extensions; they sometimes need new guides.

---

### 3.8 The V7-modification justification test

**Question.** If the proposal requires modifying V7, would V7 still need this change *if this extension didn't exist*?

**Pass.** Yes — V7 needs this independently because some other class of extension or use case requires it. The V7 change is legitimate.

**Fail.** No — V7 needs this *only because this extension needs it*. The change should live in the extension, not in V7.

**Why this test matters.** The team controls V7. Quick V7 changes feel cheap during design — "we got this new extension, just make this change in V7 real quick." The cost is hidden until V7 freezes: every modification we made because-it-was-easy becomes a community-cost every implementer must absorb. The durability thread's v7.47 changes (no-silent-ignore in §3.2, 412 in §3.3, 202 in core table) all failed this test in retrospect — none would have been V7 work without EXTENSION-INBOX §10 above them. The retraction reverted V7 to v7.46.

**Practical posture.** Treat V7 as if it were already frozen. Require explicit V7-justification for any modification. Most proposals do not need to touch V7; if you find yourself adding "and V7 gets this one small change," restart the analysis at §3.1 — that change is the smell, not a detail.

## 4. The cold re-read

Before merge, designate a *cold reviewer* — a team member who reads the proposal adversarially with §3 in hand. The cold reviewer's job is to ask the gate questions, not to confirm internal coherence.

This is **not** a new bureaucratic step. It is naming an existing implicit step explicitly. The team did do a cold re-read during the v3.2 identity substrate split (the v3.0 design was withdrawn). The durability thread bypassed this step.

If the cold reviewer cannot answer §3.1 ("name the specific deployment") with evidence, the proposal does not land normative. Three legitimate outcomes:

1. **Land as a standard extension** with the deployment driver named explicitly in the Status header.
2. **Land as exploratory** if the work is real but no deployment driver exists yet. `EXTENSION-DURABILITY` is the model.
3. **Retract at the proposal stage** before any spec text touches V7 or any standard extension. Lowest cost; preserve the analysis in the proposal file as design record.

The choice depends on the apparatus / property ratio (3.3) and whether the proposal touches V7 (high bar) vs. only the proposing extension (lower bar).

---

## 5. Status-header discipline

Extension and proposal headers carry status. Use the vocabulary deliberately. Five categories:

- **Substrate.** Primitive other extensions depend on. Removal cascades. *Examples:* `ATTESTATION`, `QUORUM`.
- **Core / derived.** Addresses a concrete problem the system inherently has — irreducible given the distributed-content-addressed-peer model. Defensible by deployment driver or by the inevitability test. *Examples:* `REVISION`, `CONTINUATION`, `IDENTITY`, `INBOX`, `SUBSCRIPTION`, `NETWORK`, `CONTENT`, `ROLE`, `GROUP`, `COMPUTE`, `QUERY`, `HISTORY`, `TYPE`. (HISTORY and TYPE are *deferred-but-irreducible*: the proper form is real even when the prototype enforces it manually.)
- **Extra / parity.** Borderline — could be built in userspace entirely, but the tie-ins (request/response surface, capability-bearer, dispatch) make it look like it should be a system extension. These reach parity with legacy / familiar shapes — Kafka-like durability, RDBMS-like isolation levels, MQ-like credit-flow. May be useful; not irreducible. Authors should explicitly justify why this is a system extension rather than a guide describing composition of existing extensions (§3.7).
- **Operational.** Held in reserve. Some extensions will be discovered to belong as operations-driven additions only when production deployments surface their need. Reserved as a vocabulary slot so the team can mark cleanly when it emerges.
- **Exploratory.** Preserved as reference design after a retraction or before a driver is identified. Not actively developed. No deployment is required to install. *Example:* `EXTENSION-DURABILITY` (the first).

The boundary between *core* and *extra* is the **inevitability test**: does the underlying problem exist whether or not we want it? Cross-peer merge: yes (CRDTs are forced). Path-level audit history: yes (records the system has anyway). Durability-property guarantee: no — the property is already reachable through composition (§3.7).

**Honest disclaimer.** This vocabulary is the team's current read; the boundary between *core* and *extra* will move with experience. The system is new; the team is learning what belongs as the system gets used. The right posture for borderline extensions is to mark them honestly — exploratory when the driver isn't there, extra when the apparatus exists but the inevitability isn't established. Community feedback over time will converge the boundaries.

---

## 6. What an extension owns

When you do build an extension, it owns a specific subtree of the entity tree — its closed namespace. Two normative invariants the substrate already enforces (per `EXTENSION-ATTESTATION` §7 / `EXTENSION-QUORUM` §3.4):

- **Closed-namespace ownership.** An extension owns `system/{extension-name}/...` and writes only within it. Other extensions do not inject paths into your subtree; you do not inject into theirs.
- **Kind namespacing.** When using substrate primitives (attestations, quorums), the `kind` field is namespaced (`{extension}/{kind}`); the substrate-universal `revocation` kind is the documented exception.

Add new ones if you find yourself crossing boundaries — but the default is *don't cross*.

---

## 7. The durability case — what not to do (canonical anti-example)

The durability thread (`PROPOSAL-DELIVERY-AND-DURABILITY`, since retracted) is preserved precisely because it illustrates how to fail the checklist. Walking it:

| Test | Durability proposal | Verdict |
|---|---|---|
| 3.1 Deployment driver | "A Kafka-shaped deployment might want…" Hypothetical; no concrete consumer. | **Fail.** |
| 3.2 Borrowed vocabulary | `requested` / `applied` / `committed` / `max_available` / `level` / `zone characteristics` — direct import from message-queue durability vocabulary. | **Fail.** |
| 3.3 Apparatus / property ratio | Eight MUSTs, ten types/fields, four pinned reason strings, eight-row status table — to deliver "respond instead of silently dropping the request." Property could have been one V7 sentence. | **Fail.** |
| 3.4 Substrate-vs-userspace | The durability *property* is observable by reading content-addressed state. No substrate amendment was inevitable. | **Fail.** |
| 3.5 First-principles | The derivation required loading vocabulary from outside. Decision 2's "anti-borrowed-enum lesson" was explicitly applied to *level values* — but the apparatus around the values was itself borrowed. | **Fail.** |
| 3.6 Cross-impl convergence sufficient? | Three teams ratified Amendment 1. Convergence happened. Did not save the thread. | **Confirms necessary-not-sufficient.** |

Five fails on tests 3.1–3.5. The cold re-read happened post-merge instead of pre-merge.

Two of the moves the proposal made *did* succeed:

- The `handle`-in-response model (Amendment 1) was a real spec improvement — it collapsed three earlier ambiguities. **Sound work that simply belonged in a different file.**
- The pinning of reason codes (`no_durable_store`, `durability_required_unmet`, `unknown_level`, `duplicate_request_id`) was rigorous cross-impl discipline. **Same.**

Both moves now travel with the lifted material in `EXTENSION-DURABILITY.md`. Sound analysis, wrong location.

---

## 8. Examples of well-scoped extensions

For contrast — these *pass* the checklist:

### 8.1 EXTENSION-REVISION (cross-peer merge)

- **3.1.** Deployment driver: any peer with cross-peer sync needs convergent versioning. Named.
- **3.2.** Vocabulary derived from version control concepts but rephrased in system-own terms (`prefix`, `version-trie`, `binding`).
- **3.3.** Apparatus is large because the underlying problem (CRDT-style merge, deletion semantics, version transcription) is large. Derivative, not accreted. The F10 trilogy (deletion markers + version-transcription + writer enumeration) closed concrete data-loss classes.
- **3.4.** Cannot be done in userspace — coordinates with the content store and the universal namespace.
- **3.5.** Derives from first principles of distributed content-addressed state.

### 8.2 EXTENSION-CONTINUATION (cross-peer dispatch chains)

- **3.1.** Deployment driver: async result delivery through capability-bearing chains. Surface area driven by real cross-peer authority problems.
- **3.2.** Borrowed concept (continuations) but the vocabulary is system-own.
- **3.3.** Amendments 1–8 each closed a specific concrete defect — apparatus tracks property additions.
- **3.4.** Substrate-bound; cannot be userspace.
- **3.5.** Derives from first principles of EXECUTE + capability chain.

### 8.3 EXTENSION-IDENTITY (peer identity, rotation, controllers)

- **3.1.** Every multi-peer deployment needs identity rotation; key rotation under active grants is a real distributed-systems problem.
- **3.2.** Vocabulary is system-own; the substrate split into ATTESTATION + QUORUM was itself a scope-overreach correction in the right direction.
- **3.3.** Surface is large but the underlying problem (key rotation without breaking cross-peer authority) is itself large.
- **3.4.** Substrate. Foundational to every other extension that uses capabilities.
- **3.5.** Derives from "we need peers to have rotatable keys without breaking chains."

### 8.4 EXTENSION-INBOX (asynchronous delivery)

- **3.1.** Receiver may be slow / disconnected / busy; synchronous request-response fails. Concrete.
- **3.2.** Vocabulary system-own.
- **3.3.** Tight pre-§10. The §10 addition was the scope-overreach; removing it restored the right ratio.
- **3.4.** Substrate-bound; coordinates with V7's `deliver_to` field.
- **3.5.** Derives from "async delivery in a content-addressed peer system."

---

## 9. Tactical recommendations

Two practical disciplines beyond the checklist:

1. **Write the spec text last.** Write the proposal narrative first — the deployment driver, the property delivered, the apparatus / property ratio table. Have the cold reviewer sign off on the narrative *before* spec text exists. The hard part of "is this the right work" is in the narrative; spec text is the easy part once the right work is named.

2. **Watch the "while we're here" growth pattern.** The durability thread re-scoped mid-flight from "three delivery classes enum" to "durability contract" while keeping the v7.47 number. Re-scopes mid-thread are a smell — they often mean "we have momentum but no concrete driver." If a thread re-scopes, restart the checklist at §3.1 against the new framing before continuing.

---

## 10. The standing question for every extension

> **What concrete deployment fails if we don't have this?**

If the answer is "none we currently care about," the extension belongs at most as exploratory — preserve the work, don't carry it as substrate weight. If the answer is "this specific deployment, here's the file:line," the extension belongs as derived. If the answer is "every deployment, structurally," it belongs as substrate.

This question is the gate. Everything else in this guide is technique for answering it honestly.

---

## 11. Cross-references

- `core-protocol-domain/explorations/EXPLORATION-SCOPE-OVERREACH-RETROSPECTIVE.md` — the retrospective and triage this guide rests on.
- `core-protocol-domain/specs/extensions/standard-peer-extensions/EXTENSION-DURABILITY.md` — the canonical anti-example, preserved as reference design.
- `proposals/PROPOSAL-DELIVERY-AND-DURABILITY.md` — the retracted proposal, kept as design record.
- `proposals/PROPOSAL-INDEPENDENT-ITEMS-CATCHUP.md` — board carries the scope-overreach watchlist row.

---

*First pass. The rules in §3 will be revised as the team applies them to actual proposals; the durability case is the seed.*
