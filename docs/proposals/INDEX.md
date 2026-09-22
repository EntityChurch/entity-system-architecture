# Proposal index — what is in motion, what landed, what died

**Triage + analysis:** `docs/status/STATUS-2026-08-13-e-proposal-backlog-triage.md` — categories,
priority, and the mechanism. **Read at cycle open, not cycle close.**

**Maintained by:** architecture. **Update on every state change** — a proposal that changes
state and does not move directory is the drift this file exists to stop.

**Workstream:** this file is an instrument of **`WS-2 · CORPUS`** (`docs/status/WORKSTREAMS.md`) — the spec
cleanup-and-refinement workstream, which carries that work's live ledger. *(Was `W-CORPUS` with a C1–C8 ledger;
renamed in the 2026-08-17 consolidation. The C-numbers and their history are in
`docs/archive/status/WORKSTREAMS-2026-08-17-pre-consolidation.md`.)*

> **Note for a reader of the published repository.** Proposals cite the internal working documents
> a decision was reached in. Those are session working memory and are **not published**
> ([ADR-0031]) — a citation to one records *where the reasoning happened*, not a document you can
> open here. **The internal families, in full, so this is checkable rather than a blanket excuse:**
> `STATUS-*` · `ROUTING-*` · `HANDOFF-*` · `PLAN-*` · `CHECKPOINT-*` · `AUDIT-*` · `TRIAGE-*` ·
> `SIGNOFF-*` · `MAP-*` · `CENSUS-*` · `WORKSTREAMS*` · `ARCH-RESPONSE-*` · `ABSORPTION-*` ·
> `REVIEW-*` — **plus everything under `docs/status/` and `docs/archive/`, by path.**
> **A citation to anything outside that set is expected to resolve**; one that does not is a
> defect, and reporting it is welcome.
>
> The proposals themselves, the specs and the guides all publish, so every *normative* trail is
> followable end to end. What stops at the boundary is the session log, on purpose — see this
> repository's `CHANGELOG.md` under *Known limitations*.

---

## 0. The model — two axes, both on disk

**State is the directory. Tier is the subdirectory.** Neither is a sentence someone has to find
and read.

> **What `DRAFT` does and does not tell you `[2026-09-08]`.** The two axes above are mechanical and
> gated. **The `Status:` line is neither**, and across the active set it currently spans at least five
> distinct situations: ready to be ruled on · a first pass whose own author says it is not ready to be
> ruled on · one whose stated ratification gate is explicitly **not met** · one **partially folded**,
> where some items have landed and the rest have not · and one the author believes is **superseded**.
>
> **So read a proposal's own header before treating `DRAFT` as a state** — every one of those five
> says which it is, in its first ten lines. A reader who needs a single machine-readable answer does
> not have one yet; `PROPOSAL-DOCUMENT-CLASS-HEADER-FIELD` is where that would land.
>
> **Tier is the target, not the topic.** A proposal sits in the subdirectory of the surface it edits.
> Some in `applications/` target the SDK specs, a guide, or the core protocol rather than
> `specs/applications/` — so the per-tier counts below describe **where the edits land**, not how many
> proposals are about the application tier.

```
docs/proposals/
  active/{core,extensions,applications,process}/       ← work is owed
  implemented/{core,extensions,applications,process}/  ← the spec edit landed
  deferred/                                            ← parked on purpose
  superseded/                                          ← retired without landing
```

**Axis 1 — state.** What is owed.

| Directory | Means |
|---|---|
| `active/` | DRAFT, under review, or partially executed. Work is owed. |
| `implemented/` | **Folded** — the spec edit landed. The proposal is now a design record. |
| `deferred/` | **Parked on purpose** — explicitly not-this-cycle. Not active work, and must not be counted as such. |
| `superseded/` | Retired without landing on its own terms; another document carries it. |

**Axis 2 — tier.** *Who can answer a question about it.* The tier is **not a taxonomy we
invented** — it is the `specs/` home the proposal folds into, so it is derivable from the
proposal's own declared target and cannot drift from the corpus:

| Tier | Folds into | Who can confirm a build |
|---|---|---|
| `core/` | `ENTITY-CORE-PROTOCOL`, V7, the wire, core types | go · rust · py · keystone |
| `extensions/` | `specs/extensions/EXTENSION-*` | go · rust · py (whoever implements that extension) |
| `applications/` | `specs/sdk/SDK-*`, `specs/applications/`, the `GUIDE-CORE` authoring stack | **workbench-go · browser-rust** |
| `process/` | authoring standards, conformance tooling, the corpus, maturity | core-go (oracle half) · arch-tools (gate half) |

### 0a. Why the tier axis was added `[2026-08-13]`

**Because a validation round was routed to a seat that does not implement the surface.** The
fold-debt audit sent all 22 active proposals to `entity-core-go` for confirm/refute. Their board
scored `APP-CONVENTION-CHAT` as **built** — a false positive they caught and withdrew themselves,
since core-go has no chat implementation at all — and matched `APP-CONVENTION-COMPUTE-PROGRAM` on
the single token `event-driven`, which they labelled noise.

**Neither was a defect in their tool.** They were asked a question about L5 and the authoring
stack, which is **workbench-go's and browser-rust's** ground, not theirs. A seat scanning for
vocabulary it does not implement returns noise rather than silence, and in an aggregate board
noise and *"not built"* are indistinguishable.

> **Rule.** A proposal is routed to the seats its **tier** names. **A tier with no seat that
> implements it reports `unvalidated` — never `unbuilt`.** Absence of a validator is not evidence
> of absence of a build.

**`active/core/` held one item as of 2026-09-04, and nothing in flight targets the wire** — which is
still what "the locked wire core is never renumbered" looks like when it is working. Several extension proposals carry *routed* core deltas (NETWORK's
§3.13 fields, CONTINUATION's §3.11 `chain_id`); those land in `entity-core-protocol` as separate
commits and do not make the proposal core-tier.

**`deferred/` and `superseded/` stay flat** — one and two documents respectively. A deferred
proposal is re-tiered when it reactivates; a superseded one never does.

**Why the directory and not the header.** Both. The header is the detail; the directory is the
signal. Post-split this repo had **39 proposals in one flat directory and no visible state** —
every status *was* recorded, in a `**Status:**` line inside each file, and answering "what is in
flight?" meant opening thirty-nine documents. Thirteen were finished and indistinguishable from
the twenty-four that were not. That is precisely the failure the pre-split layout prevented, and
losing it in the split was silent.

> **Where this came from.** The pre-split repo ran `proposals/` + `implemented/` (121) +
> `deferred/` (12), a `CROSS-IMPL-PARITY-MATRIX.md`, and `WORKSTREAMS.md`. Only the roadmaps
> survived here. The parity matrix's own §0 names the problem it was built for, in the user's
> words: *"we aligned on everything we just never followed up and got it all implemented; we all
> got distracted."* **§4 is that matrix, and it is not rebuilt yet.**

## 0b. The content roster — what each proposal SETTLES

**This index counted proposals for months and never said what any of them concluded.** Everything
above this section is about *state* — which directory a file sits in, whether that agrees with its
own header, how many are owed. **State is what a gate can check, so state is what got tracked**, and
the result was an index where a fully-argued conclusion was findable only by someone who already
knew the filename.

**The cost is measurable and it was paid in one arc:** six conclusions were re-derived from scratch
— the reachability-record serving rule, the ownership rule, the build and supply-chain case, the
refresh loop, static-route audience control, and the reader cost model — **and every one of them was
sitting in this directory.** A seventh was caught mid-draft; an eighth had already been routed to
another seat before the correction. **A proposal's conclusion reads as SETTLED, which makes this the
last place anyone re-searches** — an exploration reads as an open question and gets re-opened, so the
expensive documents are exactly the ones that were unindexed.

**Why a title roster is enough here.** This corpus's house style is that a proposal's title *is* its
conclusion in a sentence — *"unknown-field preservation is a MUST and the fidelity contract names its
authority"* — so the H1 line is a genuine one-line answer, not a filename. Generated, so it cannot
drift from the tree.

**What it is not.** A title is a conclusion *claim*, not the conclusion, and not the reasoning or the
scope. **Open the document before citing it** — the title tells you a question has an answer, which
is the whole job of an index, and nothing more.

**The `In register` column.** A ✓ means the design register carries a row pointing here, so the
conclusion is findable from the *question* side as well as by name. **It went 9 → 96 of 99 on
2026-09-07**, in the sweep this roster was built to make possible; the remainder are documents whose
conclusion is genuinely carried by another row. **Check it with `spec register`, which ratchets to
zero and treats a blank as owed** — a proposal with no design conclusion to record says so with a
`Design-Conclusions: none` marker rather than staying silent, because a blank is indistinguishable
from *"nobody has looked at this one."*

#### `active/extensions`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `PROPOSAL-THE-LOCATOR-TABLE-IS-A-REGISTRY-BACKEND-PLUS-A-SUBSTITUTE-DELTA-AND-NOTHING-ELSE-IS-NEW` | ⭐⭐⭐ **The content lookup assembled: a registry backend keyed on a hash instead of a name, consumed through the substituter's existing miss path, carried as ordinary published data, merged by the reader.** **Four of five layers need NOTHING new** — the table is an ordinary signed entity, aggregation is `SYSTEM-DATA-EXCHANGE` §1.1's closure (*the fixed point is `gathered → gathered`*), currency is the copy-current loop, and `EXTENSION-REGISTRY` §1 already reserves the backend slot. ⭐⭐ **The structural reason so little is new: the table is CONTENT, named by the TREE, moved by the DATA LAYER, resolved by the REGISTRY and consumed by the SUBSTITUTER** — the system pointed at a question about its own bytes, with the recursion terminating because a table is fetched **by coordinate, never by bare hash**. ⭐⭐⭐ **The join is exact and the spec names it: `EXTENSION-SUBSTITUTE` §3 step 3 returns 404 on a bare hash and §4 says *"No wildcards in v1… out of scope"* — a locator lookup SUPPLIES the `claimed_source_peer_id` step 3 requires, and the entire rest of the chain applies unchanged.** **Tier 2b and OPTIONAL BY REQUIREMENT** (scale-invariance is the survey's best survival predictor; one peer must stay a working system). ⛔ **The deliverable of revision 1 is SIX SEAMS, four of them collisions with landed text** — headed by **§4's *"Peer B serving peer A's content without A's signature is invalid"*, which forbids the one property the mechanism exists for, while §1 position 2 two sections earlier says the bytes are trustworthy *regardless of who served them***; the resolution is that §4's rule is written as a trust rule but protects against wasted work, not bad bytes ⇒ **a third-party claim is admissible but UNPRIVILEGED, ranked last, volume-bounded per issuer, expiring.** Plus: the referral versus §4's *no transitive following*; resolution cardinality against §4.1.1's single-binding rule; the derived key for `content` subjects; and the abuse bound, whose **shape is now known — bound PER ISSUER, never per key** — with thresholds unnumbered. **Explicitly NOT ratifiable and NOT routable** | **PROVISIONAL** | ✓ |
| `CLOCK-ADDRESSABLE-TICK-AND-DEADLINE` | Addressable ticks + a wall-deadline coordinate (the clock gains a scheduler surface, no new mechanism) | DRAFT | ✓ |
| `COMPUTE-BUDGET-PREEMPTION` | compute budget: de-confound the reduction budget from the cost ceiling; preempt instead of only failing | DRAFT | ✓ |
| `COMPUTE-ERROR-MATERIALIZATION-DETERMINISM` | the materialized `compute/error` entity is content-hashed over `code` alone | DRAFT | ✓ |
| `CONSUMER-TRUST-ANCHOR-ON-THE-STATIC-PATH` | a path→hash answer from a host is an authority claim, and the route that serves one says otherwise | DRAFT | ✓ |
| `EXTENSION-PACKAGE` | EXTENSION-PACKAGE — the foreign-artifact on-ramp (a build-provenance attestation kind) | DRAFT | ✓ |
| `IDENTITY-ROTATION-AUTHORITY-PARITY` | rotation authority parity (`identity-rotation-handoff`) | DRAFT | ✓ |
| `INBOX-OPEN-DELIVERY-ADMISSION` | INBOX open-delivery admission — the "message-me" capability, done inside the capability model | DRAFT | ✓ |
| `MEMBERSHIP-CONDITIONED-GRANTS` | membership-conditioned grants: let a capability name a condition instead of a person | DRAFT | ✓ |
| `PEER-TRANSPORT-SET` | a peer's statement of where it is: signed by the peer, servable by anyone | DRAFT | ✓ |
| `REFLECTION-LISTENER-ADMISSION` | the reflection listener has no satisfiable admission rule, and the one it needs is already written one spec over | DRAFT | ✓ |
| `RELAY-COMPLETE-THE-MODE-SET` | RELAY: the mode set, the void deferrals, and say the modes in English | DRAFT | ✓ |
| `REVISION-MERGE-DELEGATION-AND-CASCADE` | REVISION custom-merge delegation + the strategy cascade | DRAFT | ✓ |
| `SIGNALING-ICE-PROVISIONING-LIFETIME` | ICE provisioning: consumption lifetime, the TURN half, and merge precedence | DRAFT | ✓ |
| `SYSTEM-DEVICE` | `system/device`: read-only host-substrate introspection | DRAFT | ✓ |
| `THE-PUBLISHED-WALK` | the published walk: one object for every "who contributed to this" question | DRAFT | ✓ |
| `TREE-WALK-COMPLETENESS` | the absence of a node is never an answer | DRAFT | ✓ |


#### `active/applications`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `ACQUISITION-SURFACE-PROTOTYPE-TIER` | the acquisition surface as a PROTOTYPE tier: shell verbs, the room type, and how fast-moving app-tier work gets tracked | DRAFT | ✓ |
| `APP-CONVENTION-CHAT` | APP-CONVENTION-CHAT — conversational messaging as an L5 format convention (the proof-point app) | DRAFT | ✓ |
| `APP-CONVENTION-COMPUTE-PROGRAM` | `app/program`: the hostable-compute-program interface (a runtime contract) | DRAFT | ✓ |
| `APP-CONVENTION-FEED` | `APP-CONVENTION-FEED` — the entry, the reference, the index and the mirror | DRAFT | ✓ |
| `COMPUTE-LOWERING-CONTRACT` | the lowering contract: what a compiled compute interior MUST preserve | DRAFT | ✓ |
| `COMPUTE-LOWERING-TOOLKIT` | the lowering toolkit as a specified layer-2 surface | DRAFT | ✓ |
| `EXTENSION-HOST-INSTALL-SEAM` | the extension host install seam: naming the third installer, and making a generated peer a thing you can build an extension into | **IMPLEMENTED** | ✓ |
| `THE-REFERENCE-ATOM` | the reference atom: one shape for "this points at that", and a string form for the half that is typed by a human | DRAFT | ✓ |
| `SYSTEM-NAMESPACE-RESERVATION-DEPENDENTS` | the two specifications that were leaning on the `system/*` reservation's old scope | DRAFT | ✓ |
| `FOLLOW-THE-PATTERN-THE-SET-AND-THE-TWO-ROUTES` | follow: the pattern, the set, and the two routes | DRAFT | ✓ |
| `SUB-PEER-ISOLATION-MODEL` | the sub-peer isolation model (where a hosted app runs, and its boundary to the primary) | DRAFT | ✓ |
| `THE-COORDINATE-AND-WALK-DISCOVERY` | the coordinate, and how a walk is found: discharging §3.5 for the published walk | DRAFT | ✓ |
| `THE-REPLY-HINT` | the reply hint: how an author learns a reply exists without opening an inbox | DRAFT | ✓ |
| `WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX` | `{window_id}` is a session-scoped slot address, and the set of windows needs a home | DRAFT | ✓ |


#### `active/process`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `CONFORMANCE-ORACLE-CONTRACT` | the conformance oracle is a specified thing, not a program | DRAFT | ✓ |
| `CORPUS-REFERENCE-INTEGRITY` | Corpus reference integrity — a reference nothing can follow is not a citation | DRAFT | ✓ |
| `DOCUMENT-CLASS-HEADER-FIELD` | a document cannot say what kind of document it is, and three analyzers guessed | DRAFT | ✓ |
| `MATURITY-MODEL-AND-ROADMAP-COMMUNICATION` | maturity is multi-dimensional, and it should be measured, not maintained | DRAFT | ✓ |


#### `implemented/extensions`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `AVAILABILITY-DESCRIPTOR` | Availability Descriptor (an honest field primitive for partial/heterogeneous data) | IMPLEMENTED | ✓ |
| `COMPUTE-ALT-ENGINE-ADMISSION` | the alternate-engine admission contract (one rule for every execution strategy) | IMPLEMENTED | ✓ |
| `COMPUTE-BUDGET-CEILING-DETERMINISM` | the observable operation-budget ceiling charges `evaluate()` steps only | IMPLEMENTED | ✓ |
| `COMPUTE-CLOSURE-RESULT-POSITIONS-AND-CONCAT-ARGS-SHAPE` | the closure-result positions, `concat-args`' evaluated shape, and entity-key equality bytes | IMPLEMENTED | ✓ |
| `COMPUTE-COLLECTION-PRIMITIVES` | compute collection primitives (keyed grouping, indexed iteration, concat, indexed update) | IMPLEMENTED | ✓ |
| `COMPUTE-IS-ERROR-PREDICATE` | define `is_error()`, the predicate §4.1 uses 39 times and never defines | RULED | ✓ |
| `COMPUTE-V324-CORNERS` | the v3.24 collection primitives' four corners: a lost return shape, a re-litigated error code, and a rule that was already written | FOLDED | ✓ |
| `CONNECTION-NODE` | the connection node (the standalone rendezvous + reflector service) | RATIFIED | ✓ |
| `CONNECTIVITY-SIGNALING-AND-PUNCH` | Connectivity: the signaling service + punch coordination (native, TCP-simopen first) | RATIFIED | ✓ |
| `CONTAINED-ERROR-BOUNDARY-AND-CONTENT-URL-HASH-SHAPE` | the contained-error boundary, and the content-URL hash shape | FOLDED | ✓ |
| `CONTINUATION-DELIVER-TOKEN-AND-LEVEL-2` | CONTINUATION: define the `deliver_token`, and say what Level 2 consults | DRAFT | ✓ |
| `CONTINUATION-LOST-ERROR-MARKER-MUST` | Lost-error marker: SHOULD → MUST, under the handler's own authority | IMPLEMENTED | ✓ |
| `CONTINUATION-STANDING-MODEL` | Standing continuations: own authority, own completion policy (the subscription-shaped model) | ✅ |  |
| `DEFAULT-NAME-FORMAT-DISPATCH` | pin the default dispatch globs, before two app tiers ship two of them | IMPLEMENTED | ✓ |
| `DISCOVERY-RENDEZVOUS-BACKEND` | `rendezvous` as a DISCOVERY backend on the SIGNALING carrier | IMPLEMENTED | ✓ |
| `DISPATCH-AUTHORIZATION-FRAME` | the resource check follows the field, not the door; and a stored capability is still its granter's | IMPLEMENTED | ✓ |
| `ENCRYPTION-NAMESPACE-AND-RECIPIENT-RESOLUTION` | ENCRYPTION: namespace ownership at Tier C, and §4.4 recipient resolution | DRAFT | ✓ |
| `EXTENSION-ENCRYPTION` | EXTENSION-ENCRYPTION v1.0: the three-mode entity encryption layer | IMPLEMENTED | ✓ |
| `EXTENSION-SIGNALING-COORDINATION-ENVELOPE` | SIGNALING §6.3 self-contained coordination envelope | ✅ |  |
| `EXTENSION-WEBRTC-TRANSPORT` | EXTENSION-WEBRTC-TRANSPORT — the browser P2P leg (workstream D / S2) | FOLDED | ✓ |
| `HISTORY-CONFIG-SELECTION-TOTAL-ORDER` | HISTORY config selection is a two-key order, and a scalar cannot carry it | DRAFT | ✓ |
| `NAMESPACE-CLEANUP-AND-BROWSER-LEG` | namespace cleanup across the extensions, and the browser leg back on track | IMPLEMENTED | ✓ |
| `NETWORK-LIVE-ESTABLISHMENT-SEAM` | NETWORK live-establishment seam (§10.3) | FOLDED | ✓ |
| `NETWORK-LIVENESS-REACTIVE-BUILDOUT` | NETWORK reactive lifecycle: the liveness write + continuation build-out (Amendment 12) | IMPLEMENTED | ✓ |
| `NETWORK-REACHABILITY-FACTS` | NETWORK reachability facts (observed-address reflection + dial-back + candidate gathering) | RATIFIED | ✓ |
| `NETWORK-SEAM-CONSULTATION-BOUND` | §10.3 obligation 6: a bound on seam-consultation frequency | RATIFIED | ✓ |
| `PEER-ISSUED-REGISTRY-BACKEND` | Peer-issued registry backend + registration flow | RATIFIED | ✓ |
| `PIN-THE-PATH-REQUIRED-STATUS` | pin the status on `path_required`, at the site that generalizes it | IMPLEMENTED | ✓ |
| `PUBLISHED-ROOT-FRONT-DOOR` | the published-root front door: one tree path, and discovery is not convention | DRAFT | ✓ |
| `PUBLISHED-ROOT-PREFIX-AND-REPUBLISH` | `published-root`: land the type, carry its prefix, and pin the republish obligation | RATIFIED | ✓ |
| `REGISTRY-DISPATCH-CONFIG-ENFORCEMENT-POINT` | the privacy MUST binds the write, not the load; and four corrections it exposed | IMPLEMENTED | ✓ |
| `REGISTRY-DISPATCH-FILTER-SEMANTICS` | the dispatch filter is one function of the name, and the privacy MUST binds the configuration rather than a row | IMPLEMENTED | ✓ |
| `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` | the registry's fourth actor: check the name you were already given, and bound what you cannot check | IMPLEMENTED | ✓ |
| `REGISTRY-NAME-LEGALITY-INPUT-DOMAINS` | one matcher, two input domains: `name_constraints` never sees a `/` | IMPLEMENTED | ✓ |
| `REGISTRY-ONE-NAME-MATCHER` | one name matcher per registry, and a `<glob>` in a schema block is an undefined referent | IMPLEMENTED | ✓ |
| `REGISTRY-PEER-ISSUED-REGISTRATION` | REGISTRY peer-issued registration: authorization, the 202 carrier, and manual approval | DRAFT | ✓ |
| `REGISTRY-PIN-AUTHORITY-CAPABILITY-ENCODING` | pin authority needs an operation name, because a grant cannot say "the same write, but different" | IMPLEMENTED | ✓ |
| `REGISTRY-RENEW-TTL-CASCADE` | REGISTRY `renew-request` TTL: the second mint site, and the cascade that closes it | DRAFT | ✓ |
| `REGISTRY-RESOLVER-CEILING-CONFIG-SITE` | the load-bearing TTL ceiling has no config site, and four seats built four | IMPLEMENTED | ✓ |
| `REGISTRY-SERVICE-ADVERTISEMENT` | REGISTRY service advertisement (A+MX+SRV in one signed zone) | IMPLEMENTED | ✓ |
| `REVISION-AUTO-VERSION-EXCLUDE-PARITY` | auto-version adopts a root it did not filter, and `commit` builds one it did | RULED | ✓ |
| `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` | the static publish handshake: close the phantom, name the publish-side obligation, and say the freshness bound out loud | IMPLEMENTED | ✓ |
| `SUBSTITUTE-CONFORMANCE-REACHABILITY` | SUBSTITUTE: a conformance section that required what nothing could reach | DRAFT | ✓ |
| `SUBSTRATE-DISCHARGED-OBLIGATIONS` | what a substrate supplies vs. what an implementation owes | RULED | ✓ |
| `SYMMETRIC-REENTRY-MUTUAL-MINTING` | symmetric origination authority for rendezvous-established peers | FOLDED | ✓ |
| `TREE-NODE-SHAPE-BOUNDED-FANOUT` | PROPOSAL v4.3 — EXTENSION-TREE node-shape: bounded-fanout content-addressed trie (IPLD HashMap algorithmic reference) | LANDED | ✓ |
| `TYPE-OPERATION-ERROR-TAXONOMY` | `EXTENSION-TYPE` declares eight operations and zero error codes, so the one seat that built them minted its own | FOLDED | ✓ |


#### `implemented/core`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL` | an empty `grants` array: one glyph, three objects, and only one of them is defined | FOLDED | ✓ |
| `CAPABILITY-MINT-TEMPORAL-CEILING-AND-THE-WITHDRAWAL-BOUND` | "revoked on leave" names a hash the granter never holds, and the TTL that would bound it is unenforced | FOLDED | ✓ |
| `CONNECT-PREAUTH-SCOPE-AND-ADDRESS-BEFORE-AUTH` | two pre-establishment orderings the corpus never stated, one of which is holding a skip open | FOLDED | ✓ |
| `CONTINUATION-BOUNDS-PROPAGATION` | Cross-peer chain bound: wire `chain_depth`, pin cross-peer TTL, de-confound the two | REV | ✓ |
| `GENERIC-400-SYNONYMS-AND-THE-PRE-ESTABLISHMENT-EXECUTE` | `MUST NOT mint a synonym` has three live violations, and the third one hides a conformance gap two releases old | FOLDED | ✓ |
| `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1` | `system/capability/policy/{key}`: §6.2 and §6.9a.1 contradict, and the newer text removes a capability | FOLDED | ✓ |
| `STATUS-CODE-SLOT-AND-THE-DEFAULT-CODE-FORCE` | §3.3's default-code column has no stated force, and the sweep that closed one synonym was scoped by the token instead of the slot | FOLDED | ✓ |
| `STATUS-TABLE-DEFAULT-CODES-AND-THE-501-SYNONYM` | §3.3 names a default code for three statuses and leaves three bare, and implementations minted a synonym in one of the gaps | FOLDED | ✓ |
| `THE-400-SLOT-FALLBACK-AND-THE-TREE-PUT-ROWS` | the 400 slot already has what the 500 slot was given, two seats read it as a gap anyway, and `tree:put` needs two rows | FOLDED | ✓ |
| `THE-PUT-ADMISSION-PREDICATE-AND-THE-CONTENT-CODE-TABLES` | the `put` admission predicate, and the two handlers whose codes were never a set | IMPLEMENTED | ✓ |
| `UNKNOWN-FIELD-PRESERVATION-IS-A-MUST-AND-THE-FIDELITY-CONTRACT-NAMES-ITS-AUTHORITY` | unknown-field preservation is a MUST, and the fidelity contract names its authority | IMPLEMENTED | ✓ |


#### `implemented/applications`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `APPLICATIONS-DOMAIN-AND-CONVENTION-EMBED` | Proposal — stand up the `applications/` domain + `APP-CONVENTION-EMBED` (v0.2) | DRAFT | ✓ |
| `SDK-HANDLER-OWNED-SERVICES` | handler-owned services (the registration lifecycle contract) | RATIFIED | ✓ |
| `SHARE-AS-GRANT-AND-THE-AUDIENCE-CARRIER` | the share-as-grant cluster: what a share is, where it lives, and which slot carries the audience | RULED | ✓ |


#### `implemented/process`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` | the conformance coverage-failure taxonomy (§2.4a / §5.2a / §5.2b / §5.2c) | DRAFT | ✓ |
| `CORE-TYPE-EXTENSION-TIERING` | tiering the permission to put optional fields on core types | ✅ |  |
| `DEVERSION-TEST-VECTOR-CORPUS` | de-version the test-vector corpus | FULLY | ✓ |
| `HASH-WIDTH-IS-NEVER-FIXED` | a hash has no width, and a derived hash has no automatic format | DRAFT | ✓ |


#### `deferred`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `IDENTITY-CROSS-ALGORITHM-MIGRATION` | cross-algorithm identity migration (F-PQ) | DRAFT | ✓ |


#### `superseded`

| Proposal | What it settles | State | In register |
|---|---|---|---|
| `INBOX-TYPE-NAMESPACE-CORRECTION` | ~~inbox namespace rename + retire `ENTITY-CORE-MACHINE-SPEC.md`~~ **[SUPERSEDED]** | SUPERSEDED | ✓ |
| `NAMESPACE-SEGMENT-DISCIPLINE` | ~~top-level namespace discipline: one segment per owner~~ **[SUPERSEDED — rule folded]** | SUPERSEDED | ✓ |


## 1. Active — 44 files (ext 22 · app 13 · process 7 · conformance 2 · core 0)

> **−5 on 2026-09-06, and none of them was new work.** A bearings sweep found **five `EXTENSION-REGISTRY` proposals fully folded and still filed `active/`** — at **v1.16** and **v1.17**, against a spec since at **v1.24**. `proposal-state-mismatch` reported **zero** the whole time and was right to: it reads a *declaration*, and every one of the five read exactly `**Status:** DRAFT`, declaring nothing. Every delta row was verified against the corpus before the move (**L3**), including three that read as unlanded to a token grep and were not — two withdrawal notices quoting their own withdrawn text, and a `did-key` that is a DID-method maturity row rather than the `backend_kind` token. **`spec ledger` now also reports `proposal-undated-status`**, the cheap rung under the state rule.

> **29, not 22, as of 2026-08-15** — seven **reference proposals** opened by the provenance
> reconstruction (`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`). These are a
> **different animal from the July backlog below** and must not be read as more unstarted design
> work: their spec edits have **already landed**, and the proposal is being written after the fact
> to carry the reasoning and make the change reversible. The count going up is the backlog being
> *recorded*, not growing.
>
> **First pass complete 2026-08-15 — 7 reference proposals**, one per arc: the conformance
> coverage-failure taxonomy, hash-width, ENCRYPTION namespace + §4.4, REGISTRY peer-issued
> registration, SUBSTITUTE reachability, REVISION merge delegation (v3.9→v3.11), and CONTINUATION's
> `deliver_token`. **Three arcs remain and two of them are ledger updates to existing proposals**
> (`SYMMETRIC-REENTRY-MUTUAL-MINTING`, `NETWORK-LIVENESS-REACTIVE-BUILDOUT`), not new documents.
> **All seven are first-pass and none has had cohort review** — the scope of what the pass skipped is
> in that plan's §8.

> **The count is a directory listing, not a work queue, and reporting it as one was wrong twice.**
> Measured by last-touched (git, 2026-08-13): **5 touched today — four of them *created* today**;
> 4 touched between 07-27 and 08-10; **13 untouched since 07-24 or earlier**. **Eight have exactly
> one commit** — written once, never revisited. See
> `docs/status/HANDOFF-2026-08-13-d-the-audit-that-has-not-been-done.md` §1.
>
> **And the peer-side state of nearly all of them is `unknown`, not "implemented".** Every
> impl-state sentence in these proposals is a dated measurement that has never been re-verified,
> and three sibling repos moved the same day those pins were written. §3 of that handoff is the
> audit that fixes it.

> **Audited 2026-08-13, because 23 is a lot and the first pass was not good enough.** The
> initial classification trusted each proposal's own `**Status:**` line — the author's
> self-report at write time. **That is the same class of evidence this repo has been burned by
> four times, and it was wrong once here:** `SDK-HANDLER-OWNED-SERVICES` sat in DRAFT while
> `SDK-OPERATIONS` §11.6.9 (v1.11) had carried it since 2026-07-29, with the spec's own
> changelog reading *"Folds `PROPOSAL-SDK-HANDLER-OWNED-SERVICES`"* the entire time. Moved.
>
> **The rest were re-checked against the landed specs, not their headers** — target file exists?
> target section carries the substance? `system/device`, the availability descriptor, and the
> compute collection primitives appear **nowhere** in `specs/`. `EXTENSION-CLOCK` has no
> addressable-tick or deadline surface. `EXTENSION-INBOX` has no open-admission mode.
> `CONTINUATION-LOST-ERROR-MARKER-MUST` was the sharpest case: the marker **existed** in
> `EXTENSION-CONTINUATION` §3.10 — as a **SHOULD**, which was precisely the thing the proposal
> asked to change. **Present-but-not-as-asked is not folded.** *(Closed 2026-08-15: elevated to
> `MUST` in v1.23; the audit's reading was right and the fold is what it was waiting for.)*
>
> **So the backlog is real.** The count is not a filing artifact.

**Why there are 23.** Creation dates: **17 of them landed in a 13-day window, 2026-07-12 →
07-24.** One on 07-31, one on 08-02, five on 08-13. **A design burst produced proposals roughly
ten times faster than the ratify-and-fold cycle consumed them, and then attention moved on and
never came back for the batch.** Nothing has been ratified out of the July burst. That is not
a tracking problem — restoring the directories made it *visible*, it did not create it — and it
is the thing to decide about, not the number.

**One proposal is in a state the three directories cannot express.**
`COMPUTE-ERROR-MATERIALIZATION-DETERMINISM` is, per `EXTENSION-COMPUTE`'s own header,
*"folded in §2.4 but NOT yet corpus-gated cross-impl"* — the spec text landed, the cohort gate
has not run. **It stays active**, because this repo's own meta-rule is that a normative claim
about behavior is not validated until a cross-impl conformance test exercises it. Moving it on
the strength of landed prose would be the exact mistake that rule exists to prevent.

**Ratified but not fully folded — these are the ones that are actually owed:**

| Proposal | State |
|---|---|

### 1a. The active roster, by tier

**`active/core/` — 0.**

**Filled and emptied the same day, 2026-09-09** — `PROPOSAL-THE-SYSTEM-PEER-HASHABLE-BASIS-IS-PUBLIC-KEY-AND-KEY-TYPE`
**folded at `ENTITY-CORE-PROTOCOL` 0.8.2.15 + `EXTENSION-ROLE` v2.1**; now in `implemented/core/`.
**A `MUST NOT` in §3.5's type system was contradicted by §4.5a item 1a and §4.6's pseudocode**, so
two conformant readings computed different `content_hash` for one identity and the §5.2
byte-equalities failed at connect, on a correct signature with a correct key. **Enumerating the
rule's homes by its subject found five where the filing named two**, and the two it added are in
*this* corpus — `SPECIFICATION-FORMAT` §8.4.6 and `EXTENSION-ROLE` §3.6, the second stating the
preimage byte-wise — so a core-only fold would have shipped the contradiction intact. **Then the
post-fold grep found a SIXTH, in `GUIDE-RESTART-AND-PERSISTENCE`**, which shares none of the rule's
vocabulary and sits in a guide: **the enumeration finds the homes that *argue about* a rule; the
literal grep finds the ones that merely *use* it.** Do both, in that order — this is the second
consecutive core fold whose home enumeration was incomplete, and the note below is the first.
**Raised by conformance measurement across a population of independently built peers**, which is the
same lesson as that note: the tier being empty was never evidence the surface was clean.

**Emptied 2026-09-06** — `PROPOSAL-UNKNOWN-FIELD-PRESERVATION-IS-A-MUST-AND-THE-FIDELITY-CONTRACT-NAMES-ITS-AUTHORITY` folded at `ENTITY-CORE-PROTOCOL` **0.8.2.10** / `ENTITY-CBOR-ENCODING` **v1.6** and moved to `implemented/core/`. **The fold-time L23 re-run its own §6.1 required found two homes the enumeration below missed** — `ENTITY-CBOR-ENCODING` §4.6 and §9.3's unknown-**format** SHOULDs, the rule's own subject on a different noun, in the same document as the canonical home — so the fold would otherwise have shipped one document carrying the field arm at MUST and the format arm at SHOULD. Kept below as the row it was, because the enumeration's honest limit is the reason the re-run happened.

| Proposal | What it does |
|---|---|
| `PROPOSAL-UNKNOWN-FIELD-PRESERVATION-IS-A-MUST-AND-THE-FIDELITY-CONTRACT-NAMES-ITS-AUTHORITY` `[FOLDED 2026-09-06]` | **Spec-text only — no wire change, no new field, no behaviour change for any conformant peer.** One rule (*a peer preserves content it did not model*) is stated in **thirteen places at two strengths**, and `ENTITY-CBOR-ENCODING` §5.4 — *which declares itself the canonical home of the entity-fidelity contract* — carries the **SHOULD**, while the correctness argument (*"a peer that strips unknown fields… breaks content addressing for all downstream peers"*) sits in `ENTITY-CORE-PROTOCOL` §2.10 under a **MUST**. §5.4 also disagrees with itself four lines apart, and §9.1's conformance floor lists the rule **twice, at both strengths**. Raises the two weak statements, collapses the duplicate §9.1 rows, corrects **two dangling `§2.7` cross-references** (that section is *Type Name Type*; open types are §2.10), and makes each restatement name its authority. **The L23 home enumeration is done and is in §2**, with its own honest limit. Found by the 2026-09-04 landscape read against Thrift's history of dropping unknown fields. **Already gated at MUST strength by `ENTITY-CBOR-ENCODING` Appendix E and by `FEED-6` — a convention gates at MUST what the core states at SHOULD, and this fixes the direction** |

*(Previously 0, and it emptied the same day it filled — 2026-08-17: three opened out of one cohort packet — `entity-browser-rust`'s `ROUTING-2026-08-17-c` A1/A2/A3 plus `entity-core-go`'s B — ruled, folded into `entity-core-protocol` `30ca731` as `0.8.1 CAP-1`…`CAP-7`, and moved to `implemented/core/` the same session: `CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL` · `CAPABILITY-MINT-TEMPORAL-CEILING-AND-THE-WITHDRAWAL-BOUND` · `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1`, the last **re-tiered in from `process/`** on the way through.)* · *validated by go / rust / py; **conformance-validated by nobody yet** — `GUIDE-CONFORMANCE` §9 checks (r)–(v) are the validating half and are unbuilt, so this row means the text landed, not that the cohort agrees.*

> **`active/core/` being empty was read as a finding, and it was — but not only the one recorded.**
> The note below said an empty core roster is *"what 'the locked wire core is never renumbered' looks
> like when it is working."* That reading still holds for the **wire**. What it obscured is that
> §6.2's capability-handler surface is core-tier and had been accumulating unrouted defects the whole
> time — four of them, surfaced in one day, once two seats went looking at the same section from
> opposite ends. **An empty tier is evidence about what is filed, never about what is defective**, and
> the one proposal that did target this surface was filed under `process/`, where its routing seats
> were arch-tools rather than the three impls that build it. *(The tier axis exists precisely to stop
> that — §0a. It stopped it for `applications/`; nobody re-checked `core/`.)*

**`active/extensions/` — 22** *(**2026-09-16: `THE-RESOURCE-COLUMN-IS-THREE-VALUED-AND-SECTION-11-ALREADY-COMPUTES-WHICH` opened AND folded the same day — `EXTENSION-TREE` v4.11 → v4.12.** Kept active pending the cohort withdrawal notice reaching the three seats holding on it. §2.2a's `resource` column was two-valued where `ENTITY-CORE-PROTOCOL` §3.3's test is three-valued, so `diff`/`create`/`destroy` were declared `required` against four normative homes in their own document — two of them inside §11's pseudocode, which names all three operations by name and already computes the classification. **The table now cites `map_operation` as the authority instead of restating its result.** Cohort cost zero; the correction removes an obligation. Filed independently by three seats within 24 hours of the v4.11 fold, none of which implemented it.)* *(**2026-09-14: `THE-INSTALL-SET-IS-THE-MISSING-VOCABULARY-AND-PROFILE-IS-THE-WRONG-WORD-FOR-IT` opened — DRAFT, NOT folded.** The install-set vocabulary the architecture document has filed as owed in two places. ⭐ **Most of it is a census of the word *profile*: 238 occurrences across 24 documents carrying at least FIVE distinct senses, the dominant one wire-visible and pinned inside a three-way-green ordering** — so closing the gap under that noun would mint a sixth, in the same cycle that retired a quadruple-overloaded one. Recommends `INSTALL-SET` on a recognition argument (it is the phrase both filing documents already use) and records the choice AS a choice, since no mechanism fails if the word differs. Two worked sets, both lifted verbatim from existing prose; four exclusions; two mechanical checks.)* *(**2026-09-14: `THE-BRIDGE-EXTENSION-FAMILY-AND-THE-DIRECTORY-THAT-NAMES-NOTHING` opened AND folded the same day — moved to `implemented/`; see §4 for what it landed.** It left this directory at 19.)* *(**2026-09-13: `THE-TIE-BREAK-KEY-IS-A-PATH-SEGMENT-AND-NEITHER-CARRIAGE-CARRIES-A-PATH` opened — DRAFT, NOT folded**: a schema change to a landed type, so it goes to the implementations first. `NETWORK` §6.5.1a D1 orders transport profiles by `(priority asc, profile-id lex)`; **`profile-id` is a path segment and not a field**, and neither carriage that moves profiles between parties carries a path — so the rule is **unsatisfiable**, in the case the sections exist for. ⭐ **The sharp half: §6.5.1c justifies inlining its members by asserting `priority` and `profile-id` "both live inside the members", which is false about half its own subject** and is the argument for inline-vs-hash — a section that tells you the key is in the member gives nobody a reason to look. **§6.5.1a D7 splits the answer the four filed options could not**: a transport-set is authoritative and pathless (⇒ carry the id on the *set*, one source per carriage, the `page`-equals-its-key discipline), while a **registry binding is not one of D7's three ways at all** (⇒ D1 never applied; the elements are hints, and a consumer MUST NOT present their order as the peer's preference or synthesize an id). Invisible until now because it needs **two profiles of one family at equal priority** and there is one live federation with one profile per peer.)* *(**2026-09-12 — `SNAPSHOT-AND-EXTRACT-CANNOT-SEE-THE-CALLER-AND-THE-DIFF-EXEMPTION-RESTS-ON-THEM` opened, DRAFT.** `EXTENSION-TREE` v4.9 → v4.10. Three independent implementations reported unfiltered bulk reads and each fixed its own tree; measured at the line, **the landed algorithms specify the unfiltered behaviour** — `execute_merge` takes a `capability` and checks every target path, while `compute_snapshot` and `execute_extract` **have no capability parameter at all**, and §11's table says *"Snapshot prefix"* / *"Extract prefix"*. So it is conformance to landed text, not a gap against it. ⭐ **The durable rule: a path-check exemption is a claim about its upstream producer** — §11 exempts `diff` because it *"operates on stored snapshots"*, so an unfiltered snapshot moves the disclosure into the exempt operation, with both steps authorized and neither section able to see the composition. Also settles the §8.2-vs-§6.3 citation trap: §8.2 is **view-tree-scoped under §12.2 SHOULD**, so a peer without view trees reads it as not binding — correctly — and the unconditional home is core §6.3.)* *(prior: **2026-09-11 — `KEEPING-A-COPY-CURRENT-IS-ONE-MECHANISM-AND-THE-ABSENCE-IT-MUST-EXPLAIN-IS-WHAT-SHAPES-IT` opened, DRAFT.** Proposes a new **Tier 2b** extension plus a `SYSTEM-COMPOSITION` chapter, derived from three independently-filed decompositions. **Seven implementations across two code bases, sharing no code, and not one replays a change log** — currency is answerable by comparison because the substrate is content-addressed, which **inverts `EXTENSION-SUBSCRIPTION` §5.5**. The boundary is decided by `SPECIFICATION-FORMAT` §8.8 **paired with §8.9**, not by taste: consumers behave identically across subject identity, address, commitment, currency, position and intent, and differ completely on lowering and disposition. ⭐ **The frame is not *a replication mechanism* — it is *make absence attributable*:** four absences that are byte-identical to their opposites, with a different party owing each, and the two that matter most cannot be paid by the same one. **Twelve deltas; two of them (`D10`/`D11`, the application tier declaring no architectural tier at all, and *tier* meaning two different things) are independent of the rest and should land regardless.** Eight open questions, the name deliberately last.)* *(prior: **2026-09-09 — `THE-IDENTITY-ATTESTATION-QUORUM-REPAIR-SET-AND-THE-UNSPECIFIED-ARRIVAL-PATH` opened, DRAFT, and HELD.** 27 decisions against `IDENTITY` v3.10 / `ATTESTATION` v1.3 / `QUORUM` v1.2, from a formal-methods track's machine-checked models plus a three-implementation source census. **Held behind core convergence on purpose** — folding 27 decisions into three specs sitting above a tier that is still moving buys a second pass. **Three were re-verified here against our own text rather than carried on report.** The one with a claim on being pulled forward: `ATTESTATION` §3.3 says the primitive validates a revocation's signature, §4.3 and §6 say the substrate is signature-agnostic, and **TV-A8 resolves the conflict by delegating to `identity_verify_cert` "at topology-dispatch step" — which rejects `kind="revocation"` at step 1, before topology dispatch, because `identity_lifecycle_kinds()` does not contain it.** What is left standing is `is_attestation_live` (structural) plus `identity_is_authorized_revoker`, whose body is `return revoker == quorum_id` — two field comparisons — so **an unsigned revocation naming the quorum strips authority from any cert whose chain roots there, with no key compromise.** Second: `§6.3` phase 2's `(quorum-publish, *) → seed_contacts_cache` row is **unreachable** through the same step-1 gate, and `§9.4` keys compromise recovery on the cache that row fills — so a recovery signed by the identity's real quorum is rejected, **while the fail-closed prohibition beside it passes vacuously.** Third: `§9.2`'s MUST and `§4.2`'s valid-modes table cannot both hold in the recommended three-key default. **The gating decision is §2 — who owns the arrival path:** all three implementations added a pre-phase-1 kind branch no document has, no two the same, and §9.4 keys on exactly that state, so the peers do not interoperate on compromise recovery. One ruling resolves nine of the 27.)* *(**2026-09-09 — `REFLECTION-LISTENER-ADMISSION` opened.** The §8.2 rate-limit `[MUST]` scopes both unwrapped listeners and **two of its three clauses do not exist on the reflection one** — a Binding Request carries no key, and `rate_limited` is a mailbox code RFC 5389 cannot express — while the third, *refuse rather than drop silently*, **prescribes the amplification it exists to prevent** against a spoofed source. The rule that surface needs is already written for a **less** exposed one: `EXTENSION-NETWORK` §6.7.2's dial-back hygiene, which runs behind a capability over an observed connection. The two share a purpose and no vocabulary, which is why no term sweep joined them.)* *(**2026-09-08 — `HISTORY-V1-8-THE-PATTERN-GRAMMAR-THE-CHAIN-AND-THE-CODE-SET` opened and FOLDED same session** — four defects found by the seat generating the extension plus one found verifying them; `EXTENSION-HISTORY` 1.7 → 1.8, and the corpus sweep corrected the same peer-wildcard misspelling in `ENTITY-SYSTEM-REFERENCE` and `EXTENSION-TYPE`. **2026-09-08 — `TREE-WALK-COMPLETENESS` FOLDED**, `EXTENSION-TREE` 4.5 → 4.6 + `EXTENSION-REGISTRY` 1.24 → 1.25; now in `implemented/`. The live false sentence is gone: REGISTRY §6a.3a asserted that a hostile origin *"cannot omit a node from the walk without the walk failing"* while **the walk pseudocode it depended on never specified the branch**, so a collector skipping an unloadable child returned a correct, complete, shorter key set that verified. **The rules landed as ONE new normative section — TREE §3.8, the walk contract — rather than at the five sites that use them**, because R1 governs enumeration, lookup, diff, extract and a registry browse alike — one rule with five normative homes is the failure mode. §6.2 **deletes** the *"unfiltered subtree nodes MAY be omitted"* sentence — v3.x path-navigation residue contradicting the rebuild four lines above it, and the sentence that made a tolerant walk look defensible — and states what is true instead: publishing a subset is **re-rooting, not filtering**, so an extract is complete against its own root, filtered or not. **Two corrections went to the proposal, not the spec:** its delta table pointed at §3.5 for the walk and §11 for vectors and **neither is right** (§3.5 is Prefix Nesting, §11 is Capability Requirements), and its ask to apply R1 to the revocation lookup **would have been wrong as stated** — R1 bites only where a declaring parent exists, so it upgrades the signed-root walk and cannot upgrade a bare keyed fetch. **§12.1's six vectors are commissioning a first measurement, not describing behaviour:** no implementation held R1 when this was written, and the control vector is required so a walk that always fails cannot score green.)* *(**2026-09-07 — `THE-PUBLISHED-WALK` opened, DRAFT, first pass.** The deliverable from the reader's-loop / two-level-index arc, and it is a **generalization of `app/feed/mirror`, not a new mechanism.** Every application has one unanswered question — *N writers own only their own namespace and contribute to a subject nobody owns; how does a reader holding the subject find the contributions?* — and thread/repo/wiki-page/listing/room/document are **one problem in six costumes**. **The base case needs nothing** (a contribution is an entry in the contributor's own stream, and every entry names its outbound edges so the graph walks forward for free); this is only for **contributors you do not follow**. **One object, three rungs, one knob:** `participants` (~32 B each — who to ask) · `+ entry_hashes` (what to ask for, dedups) · `+ entries` (the bytes — **and that rung IS `app/feed/mirror`**, which should become an instance rather than a parallel design). **Merge is set union** — associative, commutative, idempotent, so *which walk is right* is never a question. **A `participants` claim is a POINTER, never evidence**: verify by fetching the contributor's own signed entry, so a fabricated participant costs one wasted fetch and the rung needs no trust. **§3 is the load-bearing clause — a walk is an ordinary signed entity and MUST NOT be an API**, because then walks are walkable; the counterfactual is the whole field (an AppView emits an API, a relay a query response, an instance a database — **none is the type of its own input, which is why each concentrated**). **§4 closes the redundancy question with `subsumes`**: a walker cites the walks it folds in **including other peers'**, so cost is proportional to what it *added*, union becomes incremental, and **a walker with nothing to add publishes nothing** — git's parent citation and BitTorrent's bitfield, and the corpus already has the shape in `published-root.predecessor`. **§5 aims announce at a WALKER, never at an author** — no open delivery grant, nobody's inbox opened, optional in both directions so the offline tier is untouched. **§6 is explicit about what it does not do**, including that **global search is not on offer** (the split answers coordinate-anchored queries; *what contains X* has no anchor by the nature of the question). **§7.1 is the real open question and it is bigger than the container: what a walk should CONTAIN** — raw participants, a digest, or pre-computed navigational structure — plus discovery, publish-policy, tier, the **undemonstrated** offline-to-offline encrypted case, and the CDN economics risk.)* *(**2026-09-06 — `MEMBERSHIP-CONDITIONED-GRANTS` opened, DRAFT, first pass.** The one deliverable from the CT / capability-literature arc; §6 records why the rest of that arc produced none. A capability's `grantee` is a concrete peer (`401 unresolvable_grantee`), so `EXTENSION-GROUP` §7.4's pattern is *"subgroup distributes among members"* — every new member of a group-moderated forum needs a grant issued before they can act. **`EXTENSION-ROLE` §2.3 already solves the LEAVE side and is the precedent**: an exclusion *"denies all access within that context regardless of held tokens"*, which is a live tree-consulted decision overriding a validly-held capability. This proposes its **positive dual** — a grant naming a condition (*holds a current member entry for group G*) instead of a person, discharged by an entity the requester carries. **A third-party caveat, and the macaroon construction's cost does not transfer**: their VID/CID machinery exists only to move a symmetric secret between two parties, and we have public keys and signed claims. **Convention, not mechanism** — `constraints` is already `map_of: primitive/any` and handler-interpreted; the ask is the **pin**, because a grant travels and a verifier that ignores the key widens access while one that does not recognize it denies. §5 leaves four open, and the first is whether this belongs in `EXTENSION-ROLE` as an assignment-by-condition rather than as a bare `constraints` key.)* *(**2026-08-21 — `COMPUTE-CLOSURE-RESULT-POSITIONS-AND-CONCAT-ARGS-SHAPE` opened, extended and FOLDED the same day, `EXTENSION-COMPUTE` 3.26 → 3.27, D1–D8; now in `implemented/`.** The three §3.5 corners v3.26 left open, routed by `entity-core-rust` and `entity-core-py` and concurred by `entity-core-go` (spec-issue `2026-08-21-a`) — all three measured, none converged unilaterally, because each forks a boundary hash. **(1)** `map` containing a value-form error and short-circuiting a minted one is **provenance-dependence**, which **§2.4 already forbids** — *"the in-flight representation is implementation-private"* — so the exhibiting seats are non-conformant against **v3.26**, not against this proposal. Contained set **three → five** (`map`'s output element, `fold`'s final accumulator), `filter`'s predicate result **short-circuits** (read for truthiness — containing it silently drops the element), and the *"exactly three positions"* **count is replaced by the rule that generates it**, since an enumeration written at the width of one table is wrong again at the next primitive. **(2)** `concat-args.collections` → a **single** `system/hash` — and **arch's published derivation for it was FALSE and is retracted.** It argued §7.1's `walk` descends only on scalar hash fields, so the array-of-hashes shape leaves every `lookup/tree` inside every concat sub-collection unregistered. **No implementation has that defect** (go `c1b0708`, rust `2ee6bf7`, py `f09ae70` all descend into containers, implementing §7.1's *prose* rule rather than its pseudocode), and **`concat-args` was never the only container-of-hashes** — `compute/apply.args` is `{map_of: system/hash}` and `compute/let.bindings` is an array of maps carrying one, both in §2.1, both older and far more common. Arch reasoned from **this spec's own pseudocode** to a claim about three implementations' behaviour: L8's fifteenth form. **The standing reason is uniformity + arity** — every sibling collection argument is one scalar hash of an expression, and the literal-array shape alone **freezes `concat`'s arity at authoring time**, making a concat over a computed number of collections inexpressible. **(3)** Entity-key equality is over the **materialized** form — C-9's ruling reached from the key side. **(4) D5/D6 — the evaluation-limit dispositions**, ruled from §5.1's *"restored on return"*: `depth_exceeded` **contains** (element-local), `budget_exhausted` and `cascade_limit` **short-circuit** (cumulative / chain-wide — and §10.4 makes memoization impl-defined, so containment would let two conformant peers exhaust at different elements and fork the boundary), **keyed on the `code` in both arms** since §7.3 *mandates* the value form's creation. **(5) D7** — D3's rollout: no lockstep and no tolerance in the checker (dual-kind acceptance, which this corpus forbids); named landing order, transient FAIL labelled and never baselined. **(6) D8 — chasing the retraction found the real defect:** §7.1 **contradicts itself**, its prose requiring every reachable `lookup/tree` to register while its pseudocode enters no container, so it registers nothing inside any function argument or any `let` binding. Prose promoted to `[MUST]`, pseudocode corrected, and the scalar-reference grammar invariant stated with its two enumerated exceptions. Class declared per L19: a **`validate-peer`** check, not a §7c corpus vector — dependency registration produces no boundary, which is how three walkers of three different strengths passed everything.)* *(**2026-08-20 — `REGISTRY-PIN-AUTHORITY-CAPABILITY-ENCODING` opened + FOLDED same day, REGISTRY 1.19→1.20; now in `implemented/`. It gated `REG-DISPATCH-CONFIG-REFUSED-1` row 8, which is now green three-way on the wire — `entity-core-go` drove its own harness against fresh rust `951f0ce` and py `5d9fdd1` peers from one seat, 18/18 · 0F · 0S at all three, which is a live cross-impl run rather than three self-reports.** From `entity-core-go`'s spec-issue `2026-08-20-a`. §4.3's pin-delta `[MUST, v1.19]` requires `system/capability/registry-pin`, and **§5 makes it name the same operation as `registry-configure`** — so no conformant grant distinguishes them and a portable vector cannot mint one without the other. **Derived from V7, not from the cohort:** a grant scopes on `path-scope` and `id-scope` only, and the path axis is **explicitly non-portable** (*"peers that diverge remain conformant"* on path locations), so **the operation name is the sole portable discriminator**. **§4.3's own paragraph diagnoses the defect it reproduces** — a bare tree-write *"cannot refuse selectively, cannot carry a qualifier, and cannot be distinguished from any other write to the same entity"* — and fixes the first two. **§5's maintained rule was satisfied in letter, not substance:** `registry-pin`'s column names *a condition on another row's operation*, which a grant cannot carry; D4 fixes the rule, not just the row. **Ruled: the non-dispatchable operation `pin-bindings`, and cap checks precede config validation** (unruled and cross-impl-observable, and validating first leaks a config-shaped violation list to a caller with no authority to change it). All three core seats had independently shipped `pin-bindings` including the non-obvious non-dispatchable half — **corroboration, cited last and labelled, per L18.** **L17's second shape; L17 ratified on it.**)* *(**2026-08-20 — `PEER-TRANSPORT-SET` opened, DRAFT, v2, first pass.** The address-discovery hole, and it turned out to be two things. **The lookup:** a holder of a bare `peer_id` has no way to learn where that peer is — NETWORK §6.5.4 assigns it to REGISTRY, REGISTRY §12 assigns it to an EXTENSION-IDENTITY amendment that does not exist, and §2.3 and §12 cite **each other** for the scoping with no derivation anywhere. **Ruled NETWORK's, derived rather than cited** (operator correction — the exploration's first pass had adopted §12's pointer as its conclusion): `peer_id` is V7 §1.5 substrate, the answer is a NETWORK type, IDENTITY's mapping is *durable identity → {peer_id} across rotations* which shares no key/value/lifetime/signer with this, and DISCOVERY puts identity **after** the dial so it cannot gate it. **The deeper half:** `system/peer/transport/*` carries `peer_id` **inside** the entity with **nothing signing it** — authority is *positional*, and position is what transit destroys, so once a profile travels by hash the **registry's** signature is what vouches for where a peer is, while §6.5.1a's own model is that a peer self-publishes. Proposes `system/peer/transport-set`: signed by the id-holder, `seq`-monotonic, expiring, servable by anyone — `published-root`'s properties applied to reachability, which is libp2p's signed-peer-record fix, ATProto's DID doc and Nostr's NIP-65 in one shape. Seven deltas (four design, three withdrawing text that is wrong today), **five open questions marked as open** including one-record-vs-walk-the-root and the A/B/C lookup shape. `entity-browser-rust` is working the same question independently; authored without waiting per L18, with §9 naming the rows their result lands on.)* *(**2026-08-20 — `CONSUMER-TRUST-ANCHOR-ON-THE-STATIC-PATH` opened, DRAFT, v2.** Derived from the corpus while verifying `entity-browser-rust` `7bc1ccf`, and **against our own text rather than their wiring** — they asked that their finding *not* be read as a spec defect report and that is the one thing in it we overruled. Four landed texts against each other: `EXTENSION-TREE` §3.3a (*"never trusting paths the host claims outside that chain"*), `EXTENSION-SUBSTITUTE` §7.2 (a path→hash index is an authority claim; *"anyone serving the URL can forge it"*; signature **MUST**), `GUIDE-SERVING-MODE` §8's three display states — and `EXTENSION-NETWORK` §6.5.3.1's `TREE_GET` leaf, **the route that actually serves `path → hash`**, which carries none of it under the heading sentence *"the consumer trusts the math, not the host."* On `CONTENT_GET` the consumer supplies `H`; on `TREE_GET` **the host does**, so the hash check is self-consistency, not authorship — §7.2's own words for the shape it forbids. `signed_pointer` is publisher-optional and its absence is indistinguishable from a non-signing publisher, so **the assurance level is chosen by the party being trusted**; and `EXTENSION-REGISTRY` §5.1's *"IDENTIFY is the gate"* has **no counterpart on `http-poll`**, which has no IDENTIFY — the resolved `target_peer_id` is present and never used. Five deltas, none of which makes `signed_pointer` mandatory: **the rule is about what a consumer may claim, not what it may fetch.** Does not reopen registry v1 by §0a — the *binding* is signed; the unsigned thing is the content one layer down. **Creates an app-tier sequencing constraint**: D2/D3 are a prerequisite of whichever of resolved-name-open / typed URL / foreign link ships first.)* *(**2026-08-19 — `REGISTRY-DISPATCH-CONFIG-ENFORCEMENT-POINT` opened + folded same day**, from `entity-core-go`'s consolidated cohort reply and settled on `entity-browser-rust`'s built implementation. §4.1 step 2 and §11.1 answered one question incompatibly — the MUST binds *a distribution shipping* a config, the vector bound *a resolver loading* one, and **a loading resolver cannot observe which act produced the bytes** (§6a.9.2 puts both in one entity at one path), so it over-enforced and deleted the operator `MAY` granted in the same paragraph. Ruled: enforcement moves to **packaging lint + config-write refusal**; at load, surface but never normalize and never refuse to start; and **a resolver never rewrites stored config as a side effect of reading it** — which closes SA-PY-22 by dissolving it. Eligibility ruled **kind-scoped**: browser-rust's `validate_rules`/`eligible_backends` take no `resolver_chain` and are the only built implementation, and chain-scoped would let a broad row **arm itself** the day an operator adds the backend — v1.14's "evadable by omission" one level up. New `REG-DISPATCH-CONFIG-REFUSED-1`, a config-read-back vector, because the old vector's observable (the absence of a request) **could not discriminate the two readings at all**. Also: **R-4 closed** — §6a.6 binds the remote-registry path, §3.1 constrains no storage path, and `entity-core-rust`'s revert stands; **row (d)'s "or pinned" withdrawn** as unconstructible, §4.1 step 1 returning a pin before the chain is reached — the same category error §4.1a already names, made twice in two sessions; and `REG-TTL-CEILING-REREAD-1` added for the read-at-resolution MUST, which had **no instrument anywhere**. REGISTRY 1.16→1.17.)* *(**2026-08-19 — two opened + folded same day, both from `entity-core-go`'s consolidated registry packet, and one of them was arch's own defect from the day before.** `REGISTRY-NAME-LEGALITY-INPUT-DOMAINS`: v1.15's `REG-NAME-CONSTRAINTS-GRAMMAR-1` row 3 required an admission gate to accept `x/y/z`, which §6a name-path safety refuses `400` three subsections earlier — **unsatisfiable by every conformant implementation**, verified in all three engines. One matcher, **two input domains**: dispatch sees the raw `meta_resolve` argument, `name_constraints` sees a post-normalization name. L12's shape on arch's own text — the matcher was verified, the path by which input reaches it was not. · `REGISTRY-RESOLVER-CEILING-CONFIG-SITE`: the ceiling §6a.9.1 calls *"the load-bearing one"* had **no config site**, so four seats built three — py and rust converged on `resolver_chain[].hints.max_ttl`, go put it in a Go builder option, and **`entity-browser-rust` carried a fourth key nobody had counted** (L16's second payoff in two days). Ratified py+rust's site, pinned `0`-is-undeclared and the null-`ttl` arm that all four had already agreed on off-spec, and added `REG-TTL-RESOLVER-CEILING-1` — the resolver-side vector this spec never had for its own load-bearing half. REGISTRY 1.15→1.16.)* *(**2026-08-18 — `PUBLISHED-ROOT-FRONT-DOOR` opened + folded same day** from `entity-workbench-go`'s Go-published/Rust-consumed cross-check, the first result on this surface that is **not** same-language: three of four surfaces align with no shared code (content sharding, bare-hashable bodies, two-hop signature keying) and it stops at **hop 0**. Two defects. **(a)** The published-root head-pointer tree path was **never pinned anywhere in `specs/` or `guides/`** and three impls picked — rust and py at `{peer}/system/peer/published-root`, go appending `/{base58_peer_id}`, with a source comment claiming it *"supersedes"* NETWORK §6.5.3, a supersession this corpus never made. Ruled **no peer-id segment**, on the foreground don't-double-qualify invariant. **(b)** `signed_pointer` is a **tree path** and was read as a fetch location; ruled **discovered from `manifest_url_prefix`, never conventional**, and a consumer MUST NOT join `signed_pointer` onto an origin. "Serve it at both paths" rejected — two front doors is the divergence ratified. TREE 4.2→4.3, NETWORK 1.7→1.8.)* *(**2026-08-18 — `HISTORY-CONFIG-SELECTION-TOTAL-ORDER` opened + folded same day**, found by arch while reviewing go's REVISION R15 fold and routed by nobody: `grep best_specificity specs/` returns exactly two sites and R15 fixed one. §2.2 states a **two-key** order (literal segments, then depth) and §6.2 compares it as a single value with `>` over unspecified enumeration — **a scalar cannot carry a two-key order.** py returns the faithful tuple; go and rust independently reached byte-identical 2-per-literal scalars and **tie** on `a/b/c/d` vs `a/*/c/*/e`. Folded: tuple compare + a lexicographic third key for totality, §2.2's keys unchanged. **Not hash-determining** — history is opt-in per-peer state, so this is a determinism defect, not a convergence one, and it is pinned because the fix costs one comparison. HISTORY 1.6→1.7.)* *(**2026-08-18 — `REGISTRY-RENEW-TTL-CASCADE` opened + folded same day** from `entity-core-go`'s spec-issue `-c`, raised by `entity-core-py` as SA-PY-13: `renew-request` is a **second producer of peer-issued bindings** and the D11/D12 null-`ttl` fold swept only `register`. Ruled **inherit, not refuse** — a three-step cascade ending at the superseded binding's own `ttl`, non-null by §6a.3, so it cannot resolve null and never refuses. The distinction that decides it: the predecessor's `ttl` is **recovered, not chosen**, so §6a.9.2's ban on an implementation-chosen default does not reach it. REGISTRY 1.8→1.9. **§5 files an unruled finding found while verifying it — `ttl` has no ceiling anywhere on this surface**, and §6a.3 makes `ttl` the only bound on a withheld revocation, so an unbounded requester-chosen value reproduces the unrevokable binding without ever being null. Operator decision.)* *(**2026-08-18 — `TREE-WALK-COMPLETENESS` opened** from `entity-browser-rust` `a0145a7`: `EXTENSION-REGISTRY` §6a.3a asserts a hostile origin *"cannot omit a node from the walk without the walk failing"* and **no implementation has that property** — rust `ed993bb`, go `88615f6`, py `79f38a9` all silently skip a declared-but-absent child. **The first draft got the design backwards and the correction is the substance:** it assumed a publisher serves a filtered view of one big tree, so incompleteness could be legitimate and the fix was a mode. §3.4 and §6.2 say otherwise — a tracked prefix is *its own trie* and `execute_extract` **rebuilds** (`root_hash = build_trie(bindings)`), so publishing a subset is **re-rooting, not filtering**: nothing about the unpublished tree leaks, and any unresolvable child is withholding, unambiguously. §6.2's *"unfiltered subtree nodes MAY be omitted"* is **v3.x residue** contradicting the algorithm four lines above it, and is the only text making a published tree legitimately incomplete. One invariant replaces the modes — **the absence of a node is never an answer** — which also covers §6a.6's revocation `404`. Fifth instance of the absent-vs-withheld seam.)* *(**2026-08-18 — final state: all five are folded and are in `implemented/`, plus `SHARE-AS-GRANT` from the applications tier.** The intermediate reversal recorded here has itself been reversed: only the **version cuts** were unauthorized, not the folds. Ruled-and-folded is complete and moves out — that is the ordinary lifecycle, not an operator decision, and treating `implemented/` as needing sign-off was L14 over-generalized a second time. What stands from the correction: **`ENTITY-CORE-PROTOCOL`'s version is never ours; extension versions are ordinary work**, and the reasoning previously recorded here — that a peer declining a DRAFT showed ruling-without-folding had split the cohort — **is withdrawn and was backwards.** `EXCLUDE-PARITY` reopened, ruled D6 (the four-form closed grammar, `**` rejected at config write) and folded. Original same-day entries follow: `DISPATCH-AUTHORIZATION-FRAME`, `REVISION-AUTO-VERSION-EXCLUDE-PARITY`, `DEFAULT-NAME-FORMAT-DISPATCH`, `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` and `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` were moved to `implemented/` and five spec version headers were cut, **on no authority** — cutting a version and marking a proposal complete are the operator's calls (L14). All five headers are reverted and all five proposals are back here; **the folded spec text stays landed**, unversioned, which is this repo's ordinary cohort-finding mode. The reasoning previously recorded here — that a peer declining to implement a DRAFT showed ruling-without-folding had split the cohort — **is withdrawn and was backwards**: we work from draft proposals as a matter of course, a seat that thinks a draft is wrong implements it and reports back, and a peer's schedule is not an authorization to declare a release. `REVISION-AUTO-VERSION-EXCLUDE-PARITY` is further **reopened as DRAFT** — its SA-PY-11 doublestar pin is withdrawn for contradicting `ENTITY-CORE-PROTOCOL` §5.4, see `docs/status/AUDIT-2026-08-18-the-glob-rule-…`. · Original same-day entries follow: `DEFAULT-NAME-FORMAT-DISPATCH` opened + **folded out** same day — browser-rust's last arch blocker (B16b); pins the ordered default globs and makes the catch-all local-only, because §4.1 step 2 calls itself the primary privacy mechanism and that property lives in the defaults, not the mechanism · `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` **folded out** — all twelve deltas; the association check closes a live resolution-integrity defect in three shipping impls, and §6a.1a names the byte-server the threat model never had; REGISTRY 1.6→1.7 · `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` **folded out** — all seven deltas landed, D4 adjudicated first per its own *do not fold blind* (the phantom resolves to `EXTENSION-SUBSTITUTE` §7.2); NETWORK 1.7→1.8, TREE 4.0.2→4.1 · `DISPATCH-AUTHORIZATION-FRAME` opened + RULED same day — core-py's SA-PY-9/SA-PY-10: §5.2's resource check binds **any dispatch carrying a resource**, and a stored `dispatch_capability` is **granter**-framed; `EXTENSION-CONTINUATION` §3.5's worked example spells the losing form and taught it to a cohort fixture · `REVISION-AUTO-VERSION-EXCLUDE-PARITY` opened + RULED same day — core-py's SA-PY-8, the only one of their four that gated code: `EXTENSION-REVISION` §6.1 **contradicts itself**, its Amendment-2 rule requiring auto-version to *build* a trie while its Algorithm block *adopts* an unfiltered one · `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` opened — browser-rust's two static-host findings, confirmed 3-of-3; the name check is a live resolution-integrity defect and the `ttl` bound that backstops revocation was never enforced · 2026-08-17: `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` opened — `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` was never written and three normative sentences hand it obligations; both app tiers built against it · `DISCOVERY-RENDEZVOUS-BACKEND` opened + RULED same day — browser-rust's Q8, their #1 blocker: rendezvous is a DISCOVERY backend on the SIGNALING carrier, token `rendezvous` · `RELAY-COMPLETE-THE-MODE-SET` opened — retract two deferrals, complete the mode set, name the modes in English · 2026-08-14: `REGISTRY-SERVICE-ADVERTISEMENT` folded out, `SIGNALING-ICE-PROVISIONING-LIFETIME` opened in · 2026-08-15: `REVISION-MERGE-DELEGATION-AND-CASCADE` opened, then four folded out — `NETWORK-LIVENESS-REACTIVE-BUILDOUT`, `CONTINUATION-LOST-ERROR-MARKER-MUST`, `AVAILABILITY-DESCRIPTOR`, `NAMESPACE-CLEANUP-AND-BROWSER-LEG` · 2026-08-16: `COMPUTE-IS-ERROR-PREDICATE` opened, ruled and folded out the same day — the one arch-side item the cohort was blocked on · `SUBSTRATE-DISCHARGED-OBLIGATIONS` opened + folded from browser-rust's rig routing — **and left in `active/` until 2026-08-17**, see below)* · *validated by go / rust / py*

> **Fourth instance of the half-applied Rule 1, and this time the halves were reversed.**
> `SUBSTRATE-DISCHARGED-OBLIGATIONS` carried `**Status:** RULED + FOLDED 2026-08-16` in its own header
> and the line above recorded the fold — **the ledger moved and the file did not.** The three prior
> instances (§2's note) were all the other way round: file moved, ledger stale. Same rule, opposite
> half, so a reader checking either artifact alone saw a consistent story.
>
> **Why no gate caught it.** `spec ledger` compares declared counts against directory listings; both
> agreed at 15, because the file was still there and the count still counted it. **A folded proposal
> sitting in `active/` is arithmetically invisible** — the checkable property is not the count but the
> disagreement between a file's own `Status:` and the directory it sits in. That is a gate ask
> (`W-CORPUS` C11 shape, at the state layer rather than the count layer): **grep the `Status:` header
> against the parent directory.** Cheap, mechanical, and it would have fired on 08-16.

> **`REVISION-MERGE-DELEGATION-AND-CASCADE` — opened 2026-08-15, partially folded before it existed.**
> The 2026-08-14/15 cohort round returned a formal spec-issue (go, E1–E5 + a §0) and an arch ask (py) against
> `EXTENSION-REVISION` §2.3/§4.4.18/§5.1/§5.3. **v3.11 folded three of those items with no proposal in
> existence** — the deviation is recorded in the proposal's §0, along with the second-order failure it caused:
> with no proposal to hold the rationale, the provenance narrative was written into the spec instead. R11
> reverses that. **R1–R3 are folded; R4–R11 are open and unreviewed.** The version bump is the hazard to watch —
> a rev bump reads to every downstream consumer as *this area has been dealt with*, and two-of-seven was folded.

**Reference proposals (opened 2026-08-15, spec edits already landed):** `REVISION-MERGE-DELEGATION-AND-CASCADE` *(v3.9→v3.11)* · `REGISTRY-PEER-ISSUED-REGISTRATION` *(v1.2→v1.5)* · `ENCRYPTION-NAMESPACE-AND-RECIPIENT-RESOLUTION` · `SUBSTITUTE-CONFORMANCE-REACHABILITY` *(v1.1, v1.2)* · `CONTINUATION-DELIVER-TOKEN-AND-LEVEL-2` *(v1.22)*

> **2026-08-30 — `RELAY-COMPLETE-THE-MODE-SET` gained a SECOND THREAD (§9–§11), and it is deliberately
> not a second file.** From an operator question about store-and-forward under churn: volunteer
> relays, capacity, retry, and *"where do I publish 'if you're looking for it, find it here'."*
> **Three of those four are already answered and two are landed entity types** — store-vs-forward is
> NETWORK §10 → §10.2 → RELAY §6.2.2; *"when do I retry"* is **you don't**, because delivery is pull;
> and the publish question is `system/peer/inbox-relay` (§3.5), the MX-equivalent. **The fourth — how
> much, how long, what happens when I drop — is answered nowhere**, and it is **seven deltas
> (D1–D7)**, now `§7` checklist items **11–17**.
>
> **It lives in this proposal because §7 already is the relay-completeness checklist and its item 3 is
> "retention bounded — unlimited is not a default"** — the finding is that the same defect sits on
> **Mode S, the mode that ships**, and that §4's *stop deferring spec text* names the disease for all
> seven. A second RELAY proposal would split one checklist across two files, which is the drift this
> index exists to stop.
>
> **Three of the seven are transplants of rules this corpus already ruled elsewhere, not new design:**
> retention as a **declared, clamped ceiling** is `EXTENSION-REGISTRY` §6a.9.1's shape with `retention`
> for `ttl`; **refuse-don't-evict** is `EXTENSION-NETWORK` §8.4's own table, already ruled for the
> *sender-side* queue and never inherited by the relay's inbound store (L23 — one rule, more homes
> than the document stating it); and the poll-visibility rule is **already 3-way green and written
> down nowhere**, cited by one engine's test as a §8 sentence §8 does not contain.
>
> **Two of the seven were changed by their own stress tests, and both first drafts were worse than
> what they replaced** (§10). Retiring §6.2.1's default-convention `MAY` would have made every peer
> that never published a declaration **unreachable-when-offline, loudly**; the surviving rule instead
> makes the predicate **decidable** — *has this peer polled me* is local state, where *will this peer
> poll me* is not. And refuse-don't-evict **alone is a denial-of-service** (null-expiry entries, a
> store that never drains), so it is now paired with the retention clamp and neither lands alone.
> A seventh delta exists only because D1 was attacked: **the originator's delivery deadline is
> stated in `bounds.ttl_absolute`, inside the envelope the relay is forbidden to read** — L12 on
> RELAY's own surface, fixed the way `ttl_hops` already fixes the same problem for hops.
>
> **`D2` is executed, and it is the only one.** `guides/GUIDE-CROSS-PEER-MESSAGING` §3A's email
> mapping listed *"DSN / bounce"* as **landed**, against CONTINUATION `deliver_to`. **A reply is the
> recipient's answer; a DSN exists because there is no recipient** — one row of nine where the analogy
> inverts, on a canonical published surface, and it is the row a reader consults for exactly this
> case. Removing a false capability claim from a published guide is not a normative change and should
> not wait on a DRAFT. **`D6` — the give-up notice — is the only genuine new design and is broken down
> with six open questions rather than ruled** (§11), including one that is an **L23 enumeration**: its
> rule has homes in RELAY *and* CONTINUATION.
>
> **Sequencing: D1 · D2 → D4 · D5 · D7 → D3 → D6.** D3 is the only one that costs a peer anything —
> it invalidates four conformance checks that currently pin the defect, one of which passes with the
> message *"never a silent drop"* over a store the destination cannot reach.
> **Analysis: `docs/status/AUDIT-2026-08-30-b-store-and-forward-under-churn-…`.**

**Retraction / completion — `RELAY-COMPLETE-THE-MODE-SET`** *(opened 2026-08-17, operator-directed;
**top of the post-release block**, see `TRIAGE-2026-08-17-…` A3; **second thread added 2026-08-30**,
above)*. A class of its own: not new design
and not a reference record, but **a landed spec's own scope cuts withdrawn.** RELAY enumerates four
modes and normatively specifies two; the aggregate deferral rests on a substrate absence that
`EXTENSION-SUBSCRIPTION` §8.1/§8.2 contradicts, and the circuit deferral's stated condition
(*"when a driver materializes"*) has already fired. Needs go · rust · py — the advertise entity
carries `modes: [<mode_id: "F"|"S">]`, so the naming half is a three-peer agreement on string values.
**Not keystone, and not gated on the census** (both claims withdrawn 2026-08-17: keystone carries no
relay handler, `modes` is `[primitive/string]` in the typestore so nothing regenerates, and the
census runs `--profile core` while relay is an optional extension).

**Design backlog:** `CLOCK-ADDRESSABLE-TICK-AND-DEADLINE` *(scope corrected — the tick
half is folded; only the addressable/deadline half is owed)* · `COMPUTE-BUDGET-PREEMPTION` ·
`COMPUTE-COLLECTION-PRIMITIVES` · `COMPUTE-ERROR-MATERIALIZATION-DETERMINISM` ·
`EXTENSION-PACKAGE` · `IDENTITY-ROTATION-AUTHORITY-PARITY`
*(**security-relevant; pins stale — re-pin before review.** Nearly closed on a vocabulary hit —
see §4a-i)* · `INBOX-OPEN-DELIVERY-ADMISSION` ·
`SIGNALING-ICE-PROVISIONING-LIFETIME` *(**opened 2026-08-14** — the residue
`EXTENSION-SIGNALING` v1.1 deliberately did not fold: consumption lifetime for a rotating
credential, the TURN half for the connector-entered-by-URL path, and merge precedence.
Named in-spec as §13 item 5 so the absence is explicit deferral. **Blocked on evidence, and
the evidence is one rig** — the two-NAT harness browser-rust offered, which also unblocks
`EXTENSION-NETWORK` §6.7.5 dial-back and §11.5.1 S5)* · `SYSTEM-DEVICE`

> **`REGISTRY-SERVICE-ADVERTISEMENT` left this tier 2026-08-14 — ratified + folded** as
> `EXTENSION-REGISTRY` §3b (v1.4 → v1.5); moved to `implemented/extensions/`. Two corrections
> worth keeping, because both are failure shapes this file exists to catch:
>
> 1. **This entry used to read "ratify ruled — built three-way." It named the wrong half.** What
>    is built three-way is §3b.2/§3b.3's rendezvous-hash *selection function over a
>    caller-supplied pool* — `ext/signaling/pool.go` (go `8765e1f`),
>    `extensions/signaling/src/pool.rs` (rust `cc6cb56`), `signaling/pool.py` (py `808d9e6`).
>    The `system/registry/service-advertisement` entity is built in **no** tree. All three
>    implementations select correctly from a pool nothing can deliver. That is the
>    **ahead-of-reality** shape (`AGENTS.md` — a capability note can be wrong by being *ahead*,
>    not only behind), and it survived because nobody opened `pool.rs`.
> 2. **It was ratifiable for two weeks and nothing noticed.** Its §7 blocker
>    (`CONNECTIVITY-SIGNALING-AND-PUNCH` proposed-not-landed) cleared **2026-07-31** when that
>    landed as `EXTENSION-SIGNALING` v1.0. The 08-13 audit read §7 as current and filed it P3
>    "correctly open." **A proposal's self-declared blocker is a dated claim and expires like any
>    other** — re-verify it, don't read it. What surfaced it was a *consumer in another repo*
>    (`entity-browser-rust` `ROUTING-2026-08-14`, a browser leg on host candidates only), and the
>    audit had no consumer axis. It noted that gap for `active/applications/` and never
>    generalized it.

**`active/applications/` — 13** *(**2026-09-16: `THE-APP-PREFIX-IS-TWO-ADDRESS-SPACES-AND-ENUMERATION-IS-THE-PUBLICATION-RECORD` opened, RATIFIED and FOLDED the same day — now under `implemented/applications/`.** Filed from an application-tier audit and measured against the taxonomy itself: `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1's reserved-prefix row declares `app/{app-id}/` to be **per-application state, workspace, settings** — private — while `APP-CONVENTION-FEED` §4.2 pins a **published** index at `/{peer}/app/feed/index`, inside it. **Neither document acknowledges the other and both are correct in isolation**, and the application-host guide's *claim-by-writing, no registry, no reservation mechanism* rule means an app calling itself `feed` writes its workspace over a normative public path. ⭐ **The empirical core is that a static analyzer written by the party that owns this vocabulary has confused a tree path for a type tag, or the reverse, four times in four shapes** — the two spaces are literally the same bytes, so a heuristic is the only discriminator available and it fails on exactly the form the convention pins. **Three corpus measurements taken for the proposal:** `FEED` is the **only** application convention that pins tree paths (the content-site convention registers a URL projection and explicitly not a tree-storage rule); §4.2 *Content Domains* already permits open top-level paths, so a published sibling namespace is **conformant today** and an implementation using one is not squatting; and every correct publisher enforces the published/private split from **a hand-written allow-list in its own source**, which is a rule no consumer can apply. ⇒ **partition by RULE, not by root** — a closed declared set of published convention namespaces, `{app-id}` MUST NOT collide, and the type-tag-versus-tree-path distinction stated once in the taxonomy. **Partition-by-root is the better design and is rejected on cost with the cost named:** it moves both pinned feed paths, which two shipping implementations publish and read, for a partition the rule delivers without moving a byte. ⭐⭐ **And §5 folds in the "what does this peer publish" question rather than answering it separately: the front door is landed, a published root commits to a key set, and walking one is a shipped operation — so generic convention-level dispatch is mechanically available TODAY and is blocked only by the address space not being partitioned.** A separate publication record is left to be judged **on its residue** (fetch cost; a convention's version; present-but-empty), and if adopted MUST be the peer's own signed statement about its own tree. **Named and distinguished so neither gets rebuilt as the other: a published walk answers *who contributed to a subject nobody owns*, and the cited-locator convention answers *how do I reach this peer at all*.**)* *(**2026-09-15: `A-PUBLISH-COMMITS-TO-THE-EVIDENCE-OR-THE-SUBJECT-IS-NOT-ATTRIBUTABLE-AND-ONE-RULE-CANNOT-TELL-TWO-ABSENCES-APART` opened — DRAFT, deliberately NOT folded while two seats build against it.** Two independent implementations publish feeds, both conformant with every landed rule, and they **disagree on whether an entry's detached signature is in the published artifact at all** — one commits it (its root projector records every binding under the publishing peer, so `system/signature/{hex}` lands in the committed key set; measured: every entry attributed, falsifier takes it to zero), one does not (its publish commits over `app/feed/`, and the key is measurably absent from the walk result; a live reader gets it over a separate grant resource, a static reader has no second channel). ⭐ **Neither is wrong against the text, because `FEED-R2` binds the COMPOSER at authoring time and nothing binds the PUBLISHER** — three obligations make an entry attributable (mint · reach · **be present in the artifact**) and the corpus names two, the second one convention over in `SITE` §7's grant rule. ⛔ **The argument that makes it a MUST is that `FEED-R4` is UNFALSIFIABLE without it:** *absent* has two causes — the author never signed (§1.1.1's permanent boundary, the whole reason the rule exists) and the publisher did not commit it — and **a reader cannot distinguish them, so a fully conformant reader publishes a false statement about an author.** That is `SITE` §7's `MUST NOT` one step earlier in the chain: there the reader's own grant manufactured a falsehood about the publisher, here the publisher's own prefix manufactures one about the author. ⭐⭐ **And the objection that blocked it did not survive reading the other tree:** *"no prefix contains both except the universal tree"* is false — the containing prefix is **the publisher's own peer namespace**, `{peer}/app/feed/…` and `{peer}/system/signature/…` both being subpaths of `{peer}/`, which is one peer's subtree rather than a wildcard and is exactly the scope V7 §3.5 designs for by placing each signer's signatures in that signer's own namespace. ⇒ **the remedy is the publish SCOPE, never the grant**, so the correct refusal of a universal-tree grant stays refused. **Two alternatives rejected against landed text:** live-only attribution contradicts §1.1's stated reason for choosing the detached signature at all (*an entry that travels alone should verify alone* — it makes the travelling form the one that does not travel) and §1.1.1's transport-independent authoring obligation; carrying the signature hash in the index rows is a wire change that stores **a second copy of a derived value** the reader can already compute (`hex(entry_hash)`, a §3.5 core-general constructable path), which `FEED-R26`/`FEED-R27` already rule against one case over. **The mirror is the real exception and one implementation already solved it:** a mirrored entry's signature lives in its *author's* namespace, which the mirror's root cannot commit to without falsifying its declared `EXTENSION-TREE` §3.3a prefix, so a mirror MUST **carry** it — recognition of shipped practice rather than a new ask. Four conformance rows and the discriminating check `FEED-14`, whose anti-vacuity arm is the same feed published over the peer root — **a single-prefix fixture passes against both rules and measures neither.** The generalization past this one convention is named as a **candidate and deliberately not written**: one measured instance is not a promotion.)* *(**2026-09-13: `THE-ASSET-POSITION-IS-NOT-SPECIAL-AND-A-SITE-SCOPE-CANNOT-VERIFY-THE-SITE` opened and FOLDED** — `SITE` v0.5.2. Two clauses, neither widening a rule. ⭐ **The first answers the arc's only user-visible interop divergence with text that landed before the argument started:** `APP-CONVENTION-REFERENCE` §3.4 says a bare-string link with a leading `/` is **root-absolute within the current site**, and the same paragraph **recommends producers emit that form** — so one implementation refusing it is a pure loss, and both implementations were searching for a rule scoped to the *asset position* when the rule is scoped to the *form*. **A correction against ourselves rides with it:** the 2026-09-09 fold named §3.4 as the authority **and then characterized the refusing implementation as conformant with it**, which is backwards, and the divergence stayed open four more days because nobody was told they had to change — *an authority named is not an authority applied.* The second: a site subgraph is the capability scope of its **bytes**, and the evidence making those bytes verifiable sits **outside** it by design, so a narrow grant yields a readable, unverifiable site that reports *"this publisher has published no root"* — a false statement about the publisher manufactured by the reader's own grant, and **the narrow grant is the one a careful implementer writes.**)* *(**2026-09-13: `THE-GATHERED-VIEW-IS-A-GROWING-SET-AND-THE-RULE-FOR-THAT-IS-ONE-TIER-UP` opened and FOLDED the same session** — `APP-CONVENTION-FEED` v0.3 + `SYSTEM-DATA-EXCHANGE` v0.2. Four defects from two seats in one week, and **three of the four are rules this corpus already states in a document the author had no reason to open**: the bounded-collection rule (`SITE` §4 removed the identical field at v0.4.1, writing *"paging belongs to `.list` … never to a manifest field"* — and `FEED` §4.3 rule 5 says it one section above §6), the derive-to-meet hash disposition (`SPECIFICATION-FORMAT` §8.4.6, with `EXTENSION-REVISION` §3.1's `prefix_hash` the landed instance over the identical input), and the live-reference resolvability requirement (`REFERENCE` §2.2.2). ⭐ **The v0.2 fold is what created the largest one**: widening the mirror's `subject` to a timeline moved a bounded object into the unbounded class and no constraint was re-scoped with it — `L21`'s dormant-field axis, committed by the fold that ratified `L21`'s fifth shape. §2.5 promotes the growth rule to the composition tier on §2.3's exact argument, one axis over: there a convention inherits closure and loses **authorship**, here it inherits closure and loses **readability at scale**, and both are invisible from the format that causes them because the cost is the *reader's*. The census is published with its surface — **one live instance, one previously caught, six correctly bounded.** The fourth defect is arch's alone and is a wrong ruling, not a missing rule: *"do not build `{page, applied}`"* was written about a wire field and read as forbidding the mechanism, so a seat built **no cursor at all** and §4.3 rule 4's `O(new)` went unimplemented.)* *(**2026-09-09: `THE-EMBED-REF-IS-A-REFERENCE-AND-F-1-OUTLIVED-ITS-UNION` opened and FOLDED the same session.** The first application-tier cross-implementation publish run surfaced four corrections, and the shape they share is the finding: **each is a rule that was correct when written and stopped being correct without anything moving in it.** ⭐ **`F-1` is WITHDRAWN, not extended.** Its own stated justification is disambiguating the untagged `(path / content-hash)` union *"at the wire layer"* — and `APP-CONVENTION-EMBED` §2 retired that union in favour of a tagged reference atom (`APP-CONVENTION-REFERENCE` §2.1 `REF-R1`: *there is no untagged atom*). **A rule outlived the union it disambiguated**, because the retirement was written in CDDL vocabulary and the rule is stated in directive-string vocabulary, so neither enumerate-by-subject nor grep-the-literal reaches the other. It was attached to a §9 conformance vector, so a dead rule was about to be pinned into a check set. **And the surviving rule contradicted the convention that owns the subject in a way that is a security difference, not a wording one:** `F-1`'s leading-`/` arm read as a **V7 §1.4 absolute entity path** (addresses the whole tree) where `APP-CONVENTION-REFERENCE` §3.4 says **root-absolute within the current site** (confined to the subgraph) — and the implementation that reported it was refusing the leading-`/` form as part of subgraph confinement, i.e. **conformant with the reference convention and non-conformant only with the dead rule.** The tell was inside the document being corrected: **§3.1 already defers links to `APP-CONVENTION-REFERENCE` §3.4, one paragraph above §3.2's incompatible two-form rule** — same subject, same section, ten lines apart. Also folded: **`G-PIN-4`'s comparand** is the `tree:snapshot` root, never the published-root head, which carries `published_at` — measured, two runs of one fixture under one pinned identity seed gave different heads and an identical structural root, so a check posed on the head reds 100% of the time *wearing a real divergence's clothes*; **§4.1** gains the walk-vs-artifact scoping for the visited-set and a **MUST** that a refused manifest be distinguishable from a decoded empty one (a decode failure was silently substituting a default, discarding `site_id` and `title`); and **`APP-CONVENTION-SHARE` §2.5** gains *withdrawal unlists, it does not retract* — for a type defined as needing no authorization there is no grant to revoke, so *stop listing* is the honest verb and *Delete* is a promise the type cannot keep. **Three questions recorded as deliberately unruled** — relative `..` (refuse-early vs resolve-then-confine), partial decode vs distinguishable refusal, and whether anyone should mint `app/site-root` — each needing a second implementation measured on it first.)* *(prior: **2026-09-09: `A-PUBLIC-SHARE-IS-A-DIFFERENT-TYPE-NOT-AN-AUDIENCE-VALUE` FOLDED and moved to `implemented/`** — `APP-CONVENTION-SHARE` v0.2, deltas D1–D4 plus SHARE-7/8/9. The noun was settled by reading both front-ends' source rather than either seat's account.)* *(prior note: *(**2026-09-09: `A-PUBLIC-SHARE-IS-A-DIFFERENT-TYPE-NOT-AN-AUDIENCE-VALUE` opened, DRAFT.** An app seat ships a **public** offer as its default and `APP-CONVENTION-SHARE` has no carrier for one — `audience` is an enumeration of authorized wielders each with a minted token, and a public offer has neither. **All three in-`audience` encodings break something already load-bearing:** the empty array is spoken for (§2.2, *authored, no members yet*), a `default` sentinel makes `grantee` accept a non-`peer-id`, and a sibling flag leaves two fields able to disagree with no way for the schema to forbid it. **The convention already made this exact call once** — §2.4 separates `app/share/follow` from `app/feed/follow` on *authorization* and *does the publisher know*, refusing to unify because that would change what an absent field means in an already-landed schema; **carrying "public" in `audience` is the same move on the same document.** Adds `app/share/offer` — title, content hash, no audience, no grant, pull-only. ⭐ **It resolves the live tag divergence as a SPLIT rather than a rename**, which is why the filing seat stopped: renaming their tag to `record` would emit entities claiming to be direct shares while carrying no audience — worse than the mismatch. Three open questions, including whether `offer` is the right noun.)* *(**2026-09-07 — `THE-COORDINATE-AND-WALK-DISCOVERY` opened, DRAFT, first pass, written to be attacked.** It discharges an obligation rather than inventing one: **`ENTITY-CORE-PROTOCOL` §3.5** (*Discovery locality*, normative v7.45) requires any entity core machinery must **discover** to declare **Strategy A** (observational, with fail-closed behaviour) or **Strategy B** (a **core-general constructable path**), and forbids path-construction against an extension-private scheme — **and `THE-PUBLISHED-WALK` declares neither.** This declares **B**, and **A is rejected structurally**: observational discovery only finds walks you have already seen, while the query is *find a walk for a coordinate I hold from a peer I have not asked yet.* **A coordinate is always a `system/hash`** over one of four canonical preimages — `entry` / `namespace` / `path` (all **owned**, and the last two close `F-35`'s *"replies to anything I wrote"* gap) and `name` (**ownerless**) — carried beside the walk so a reader **recomputes and checks it**. **§2.2 is a refusal and it is the design move: we do not try to make "the same topic" match** — NFC, no control characters, a scheme-declared case policy, and **no confusable folding, stemming, synonyms or stripping, ever**; two people who type different names get different coordinates, **which is correct because §2.3 makes aliasing an ordinary signed claim discoverable at both coordinates, merged by union.** ***Identity is exact, and agreement about identity is a first-class mergeable artifact*** — every system that instead made naming canonical had to appoint someone to canonicalize it. **§3.1's path mirrors the invariant pointer** — `/{walker}/system/walk/{hex(coordinate)}`, two-layer storage per `REGISTRY` §6.3 — so **given a coordinate and any peer you already reached, the address is computable and no lookup service is consulted at any point.** **§3.2 is the honest half: a 404 is COULD-NOT-LOOK** (the absent-vs-withheld collapse, seventh instance) **unless the walker declares `system/walk/` a tracked prefix**, which by `REGISTRY` §6a.3a converts it into a **verifiable negative** — *absence yields no conclusion unless the publisher paid for one to be drawn.* **§4.1 unifies the arc with the supply chain**: `PROPOSAL-EXTENSION-PACKAGE`'s `build-provenance` set and a walk are **the same object with a different predicate** — a grow-only set of independently-signed claims about one coordinate, merged by union, **with the reader applying an acceptance policy** (`EXTENSION-QUORUM` where K-of-N is wanted), which is also where `F-12a`'s missing importance dimension lives: **the reader's, not the set's.** **Defines no new entity type; §5 D5 touches no extension.** **§4.4 argues its own tier UP** — §3.5 forbids extension-private path schemes, so §3.1 must sit at the interoperability floor. **§6 names the blocking measurement (reader cost at scale, still unmodelled in the third document to say so) and the squatting surface; §7 is the per-seat review ask** routed to both app-tier seats.)* *(**2026-09-06 — `THE-REPLY-HINT` opened, DRAFT, first pass.** Closes the one prerequisite `EXPLORATION-THE-READERS-LOOP` §3.3 names and does not solve: route 3 — the author's own comment section — is blocked not on the index but on **the author learning a reply exists**, and `EXTENSION-INBOX` §1.1 delivers only against a `deliver_token` the recipient minted, with **no open-grant or public-inbox mode anywhere in the extension**. Deliberate, and the corpus's anti-spam posture — but it makes *"someone replied to your post"* inexpressible. **Webmention (W3C REC) is the same problem under the same constraint and its answer is that the notification carries NO CONTENT** — §3.1.3 is `source` + `target` and nothing else, and §3.2.2 makes the receiver *"MUST perform an HTTP GET on source … to confirm that it actually mentions the target."* **The authority lives at the source; the message is only a pointer to go look**, so a forgery fails verification and is dropped. Proposed here as two hashes `(replier_peer_id, parent_entry_hash)`, verified by the same code path the reader's loop already runs. **Not a new mechanism** — an open delivery grant scoped to one narrow operation, so *every route is opt-in* is preserved exactly. **Strictly stronger than the web's**: our source is a signed content-addressed entry in the sender's own namespace, so spamming requires publishing a permanent attributable act under your own peer-id, and rate-limiting has a stable subject the web does not have. **Blocking falls out of rules already ruled** — ignoring a hint IS the block, and `FEED` §4.2's *omit but never substitute* means Alice's thread lacks Bob while Bob's own readers still see him, which is the design working rather than a leak. §5 leaves five open, and the strategic one is **whether route 4 makes this redundant**: an aggregator serving backlinks gives the author a comment section by polling, with no open grant and no spam surface at all.)* *(**2026-09-04: `APP-CONVENTION-FEED` opened — DRAFT, and it is link 3 of the chain audit, the one empty link between two finished ones.** The corpus has **no vocabulary for "a thing someone posted"** — grepped for the whole family and got zero — and `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2 predicted exactly this, deferring the semantic layer *"to a named feed/index extension authored when a concrete feed use case arrives."* **The 2026-09-03/04 operator ask is that use case**, so the deferral has expired on its own stated terms and §3 **satisfies** §4.2's ruling rather than reopening it (the determinism floor is untouched and still governs raw enumeration views). Four tags — `entry` · `index-page` · `mirror` · `follow` — plus the `reference` atom `{peer, hash, path?}`, all worked out in `EXPLORATION-THE-PUBLIC-SOCIAL-STACK-THE-COMPLETE-MAP` §4 rather than invented here. **Three derived properties carry the document**: `author` **MUST** equal the namespace (the forgery gate, and what makes republication safe); **an entry never inlines another entry's bytes** — the operator's dedup point, worked out as a *granularity* rule, because content addressing dedups nothing by itself and what it actually buys is a **shared name**, which is what makes a reply cost the size of the reply and two independently-gathered mirrors merge by union; and the set only grows, so **a partial view is short, never wrong**. **§5 is new and is the operator's own distinction made normative** — monotone data (a feed) vs authoritative data (a binding, an endpoint, a revocation): a stale feed is *short* and its cadence is a reader preference, a stale binding is *wrong* and its cadence is a correctness parameter, so **a reader MUST NOT surface feed staleness as an error** and **MUST NOT extend an authoritative record's lifetime because a fetch failed.** **§7 walks the whole "I add someone" chain in nine steps** with a built/designed/missing column: seven of nine are substrate that exists, the two missing are this proposal, and the one hole is bare-id → origin (`PEER-TRANSPORT-SET`). **D3 proposes a charter discipline #7 — the compatibility contract** (new fields optional, no renames, no type changes, breaking change ⇒ new tag), which is the wire rule we already hold one layer down and hold **nothing** equivalent for at the app tier; the live `app/share/*` tag divergence is the evidence, and it is invisible from both sides because a wrong-tag `type_filter` returns a correct, complete, **empty** answer. **D4/D5 are the L23 half**: the content-site's deferral and the share convention's `follow` tag both change meaning and neither is optional — `app/share/follow` follows a **grant**, `app/feed/follow` follows a **namespace**. **Seven vectors OWED before ratification**, three of them load-bearing (author≠namespace rejected · a mirror's two authors still verify · an unknown field round-trips byte-identical) — the last two are what fail if an impl re-serializes instead of republishing, which is the one failure that turns the model from evidence into hearsay and which prose review cannot catch. **`[OPEN-FEED-1]` is the honest seam**: the cursor is carried provisionally and its home may be the follow proposal's D2 instead)* · *(**2026-08-31: `WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX` opened — DRAFT, folded at authoring, eight deltas across four guides.** Filed by **`entity-browser-rust`** with a built implementation and a two-arm falsified gate, after a user-visible symptom: `GUIDE-ENTITY-WORKBENCH-APP` §8 offers a **persist** arm for per-window state and **nothing anywhere constrains `{window_id}`'s lifetime**, so both impls made it a per-session counter and the state at `windows/3/state` belongs to whatever held ordinal 3 last boot. **Ruled: `{window_id}` is a session-scoped slot address**, derived from §8's conditional arm and §3.2's path-position-invariance rather than from what the impls do (**L18**). **The roster slot already existed and no one knew**: `app/state/layout` sits in `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §5.1 and `GUIDE-SDK-PATTERNS` §2, both canonical and published, **absent from the §4.2 table §4.1.1 names as the authority**, and §4.2's consensus had already assigned arrangement to per-impl decoration — one rule, three homes, one updated (**L23**). Retired, not repurposed; `app/state/window-index` added on the **membership-is-portable, arrangement-is-not** line. **The filed "one small thing" is the largest item**: on the transitional `app/state/window` fallback every window's entity carries the *same* type name, so the filing seat's own discriminator mitigation is structurally unavailable — the easy on-ramp is more dangerous than the shape it leads to, and `content_type` becomes required (**`entity-workbench-go` already writes it**, under a kebab key that violates `STYLE-NAMING`). **Two defects found in our own text on the way**: §1 tiers the action wire shape **MUST** while §6 calls it speculative and **NOT REQUIRED** — the contradiction the filing seat's §1 argument was standing on, corrected to §6 (**L19** — §1 is an index, §6 owns the surface); and `GUIDE-SHELL-FRAMING` §7.1's `Selection` example emits `origin: {surface, window_id}`, which §5.4 rule 1 makes a **MUST NOT** — invisible to a token grep and to workbench-go's live `legacySelectionFields` gate, so an impl reading it would be **non-conformant and silently green** (**L23** second shape, **L25**). **Two of the filing seat's claims corrected without moving the conclusion**: their "cosigned MUST" forbidding a path re-key **does not say that** (§1 pins the *prefix*; §3.1 blesses two sub-shapes) — withdrawal upheld on design grounds instead; and workbench-go's *"write-only, future restore last session"* quote is on the **shell-alias** slot, keyed by a durable alias name, while the real workbench-go evidence is **stronger** — a live read at `log_model.go:143` keyed on a session ordinal, into a read-modify-write merge (**L22**, and arch moved the sentence as much as they did))* · *(**2026-08-30: `FOLLOW-THE-PATTERN-THE-SET-AND-THE-TWO-ROUTES` opened — DRAFT, first pass, and deliberately a stress test rather than a sign-off candidate.** Answers the dependency question a design audit was opened for: **following requires `EXTENSION-TREE` + `EXTENSION-NETWORK` + core, and NOT `EXTENSION-REVISION`** — `system/peer/published-root` (TREE §3.3a) already carries `root_hash`, **`seq`** (the change check *and* the rollback defense) and `predecessor` (a history chain) in one signed entity, so the cursor, the change check and the walk entry point are all there. REVISION is required exactly where it should be — `revision:fetch-diff` and DAG convergence — and neither is a requirement of the *pattern*, only of one route through it. Establishes **two routes**: **A** (live publisher, dispatch, work on the publisher, **needs a capability grant**) and **B** (static origin, walk the root, work on the reader, **publisher may be offline**), where B gets its delta free from content addressing because dedup skips held subtrees. **T5 is the result that constrains the design**: a stranger holds no `revision:fetch-diff` grant, so **public following is Route B only** — a stronger argument for B as the base case than liveness was. **Three stress tests are UNRESOLVED and say so**: **T2** — a publisher who has not republished and an origin withholding a newer root are *byte-identical* at the consumer, so "no new posts" is not a fact a follower may assert; **T3** — a followed publisher that re-keys is a *different peer* and every follower silently points at an abandoned identity, with a successor pointer signed by the old key being exactly what a compromised key must not be able to assert; **T6** — a publisher restored from backup at `seq: 0` is **permanently unfollowable**, because the rollback defense is working as designed. Six deltas, one of them (**D2**, the cursor site) the only substrate-shaped claim, left open on Q1/Q3's promotion test — *is there a second non-social consumer of the identical loop?* **Second pass same day: T3's prior art pulled in as §6a rather than re-derived** — the property we lack is `did:plc`'s *identifier-is-a-hash-of-genesis*, core has already ruled cross-form correlation belongs to identity/registry/policy and **never core**, so T3's answer is a **registry binding kind** and not a follow-list field; a genesis-hash name form **conflicts with the landed `self-certifying` pin** (`name == target_peer_id`, explicitly not a hash) but lands cheaply **beside** it under REGISTRY's ignore-unknown-kind forward-compat rule. Carries the uncomfortable one: **a genesis that is a bare keypair with no attestation has no rotation path under any design**, so for peers publishing that way T3 is structurally unavailable rather than merely open — and *publishing freezes what consumers pin*, which is the deadline. **Two new stress tests. T10 — a follower cannot scope its attention below the peer**: one root and one `seq` per peer (forking the stream is what destroys rollback detection), and the trie is HAMT-routed by `SHA-256(relative_key)` so *"structure is determined by hash bits, not by path-segment locality"* — there is no subtree to check, no per-prefix hash without already holding the bindings, and the path-keyed LocationIndex is **local and unpublished**. So *"did anything I care about change?"* costs the same as *"what changed at all?"*; three directions named, none picked, and it is flagged as **the question most likely to change the design**. **T11 — asymmetric extension adoption**: rotation continuity that lives in an extension the *consumer* may not have leaves that consumer seeing T3, the same shape the parked key-death work already met as a support-tier flag with no feature negotiation)* · *(2026-08-21: `ACQUISITION-SURFACE-PROTOTYPE-TIER` opened — the **living ledger** for browser-rust's acquisition work, which is arriving faster than proposal→ratify→fold turns)* · *validated by **workbench-go / browser-rust**, **not core-go***)*

**`APP-CONVENTION-FEED`** *(**the stage-1 keystone — consumers: browser-rust, workbench-go, and the godot/python seats behind them.** Nothing at the assembly or curation layers can be specified until it lands, and it is the format a third party copies, so §0's publish-once constraint bites hardest here)* · `APP-CONVENTION-CHAT` *(consumers: browser-rust, workbench-go — **both asked 2026-08-17**)* · `SHARE-AS-GRANT-AND-THE-AUDIENCE-CARRIER` *(RULED; fold = author `APP-CONVENTION-SHARE`)* · `APP-CONVENTION-COMPUTE-PROGRAM` *(consumer:
workbench-go)* · `COMPUTE-LOWERING-CONTRACT` · `COMPUTE-LOWERING-TOOLKIT` *(§5 sweep built at
core-go's `compute-corpus/worked.go`; the builder/toolkit surface is what is owed)* ·
`SUB-PEER-ISOLATION-MODEL` · **`WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX`** *(filed by browser-rust,
built there; consumers: **browser-rust + workbench-go**, both routed 2026-08-31)* ·
**`ACQUISITION-SURFACE-PROTOTYPE-TIER`** *(**append items to its §3
ledger rather than opening a proposal per item** — operator direction 2026-08-21. Carries the
**PROTOTYPE** verb tier for `meet`, and the `app/chat/room` ruling)*

**This is the tier that had never been routed to anyone who implements it.** Four of the five
have a running consumer in tree; none of those consumers was asked.

**`active/process/` — 7** *(**2026-09-16: `THE-NO-REV-BUMP-CARVE-OUT-EXEMPTS-A-PROPOSAL-NEVER-A-VERSION-AND-THE-BUMP-LADDER-HAS-NO-ARM-FOR-THE-CHANGE-WE-MAKE-MOST` opened — DRAFT, and its `SPECIFICATION-FORMAT` half is FOLDED (§9.1, §9.2, §5.3; the standard bumps itself to 1.5 under the arm it adds). Answers `CQ-46`/`CQ-47`. A seat outside this estate measured a release boundary where two of three core documents changed normative content while their version headers — declared the source of truth — did not move, **and the conformance corpus was byte-identical on the same boundary**, so both surfaces a consumer is told to trust read as unchanged while a `[MUST]` was added. ✅ **The carve-out never reached the version:** `Spec-Change: cohort-finding` exempts a **commit** from the **proposal** obligation, and *“no rev bump”* in its description was a description read as a grant. **Measured: 26 `(commit, spec file)` pairs carry the trailer, 7 bumped and 19 did not** — one trailer, one author, both readings in the record. ⭐⭐ **The root cause is ours: §9's bump ladder had two arms, *additive* and *breaking*, and a CORRECTION is neither** — the change this corpus makes most often — so an author opened the right home, found no arm, and fell back to the only sentence in the estate that mentioned rev bumps. **`L23`'s enumeration shape, third instance in four days, first one inside the standard that governs the specs the other two were found in.** ⇒ third arm added, and the operative test is *could a conformant implementation of the previous text be non-conformant under the new* — **cause never decides, consequence always does**; a stale-citation fix to an unchanged rule correctly does not bump. ✅ **What a consumer pins is the version header, made true — all three alternative shapes declined**, each leaving the declared source of truth able to be false with a second, truer surface beside it. ⭐ **`CQ-47`'s residue is real and `provenance` is the wrong home on a UNIT MISMATCH** — its unit is a commit, the defect's is a pair of documents, and the side that moves is usually the authority in a commit that never touches the restatement. **`sdksync` was the instrument all along, scoped to one directory** — seventh scope-set-once in this toolkit — so `spec pointers` is **built**: 12 declared pointers over 11 pairs across both corpora, all hand-verified against their authority before pinning, 0 drifted. ⛔ **An UNDECLARED restatement stays invisible and the residue is named rather than implied.**)* *(**2026-09-15: `A-CONFORMANCE-CLASS-SAYS-WHAT-AN-ARTIFACT-IS-AND-NAMING-ITS-AUTHOR-IN-THE-SAME-CELL-IS-WHY-A-NEW-SEAT-HAS-NO-ROW` opened — DRAFT.** A seat authoring a neutral requirement corpus reported fifty files unable to satisfy `SPECIFICATION-FORMAT` §8.5's *declare your conformance class* `[MUST]`, because `GUIDE-CONFORMANCE` §7.0's four classes describe nothing it produces. ⭐ **The report has two halves with different answers and only one needs a change.** ✅ **Half 1 dissolves: §8.5's obligation is on a conformance ITEM, and §8.5a already separates the two objects in those words** — *“`R` distinguishes a requirement from a conformance item; `ROUTE-R3` is a requirement, `ROUTE-EXACT-1` is an item that may exercise it”* — so a requirement inventory declares no class and is **inapplicable rather than in breach**; a corpus named `<PREFIX>-R<n>` is declaring which object it is by the id convention. The condition that IS real: a file that both states an obligation and carries **scored arms** is two objects and splits. ⛔ **Half 2 is a genuine taxonomy defect: each class row names an AUTHOR REPOSITORY inside the class definition.** A class is what an artifact **is** and can **prove** (durable); an author is the current division of labour (dated). Bundled, the table answers *what is this thing* with *whoever currently writes those*, which is why a seat that did not exist when it was written finds no row. ⭐⭐ **And it is not filing: as written, the behavioral row routes every behavioral check to one author BY RULE, making an already-adopted position unsayable** — *one oracle cannot measure itself*, target **independent parallel check sets**, and a second independent behavioral set is exactly *a check not authored by the oracle author*, which the table classifies as nothing. ⇒ **the `Authored by` column moves into a dated §7.0a** carrying the routing rule verbatim (*file against the check set's author, never whichever seat found the defect*) plus one sentence it lacks — ***the author of a class is a fact about the estate on a date, not a property of the class*** — and a new §7.0b restates the requirement/item split in the guide where the class question is actually asked. ⛔ **No fifth class is minted, deliberately**: §8.5a already governs the object, and a fifth class gives one object two homes — *the boundary between two homes is exactly what drifts*. **Nothing in `SPECIFICATION-FORMAT` changes; the guide it points at is what needed the edit**, which is a different repair from a `MUST` that is wrong.)* *(**2026-09-09: `THE-VOCABULARY-LIFECYCLE-TRANSITIVE-COMPATIBILITY-AN-EXPERIMENTAL-SEGMENT-AND-THE-FREEZE-TRIGGER` opened, DRAFT.** Closes the three gaps that only became visible once `SPECIFICATION-FORMAT` §8.7 landed: the compatibility contract is **pairwise** where this substrate needs it **transitive** (a mirror re-serves entries indefinitely, a dormant tree is a full participant, and content addressing makes old bytes permanent — **so there is no version horizon past which old data stops arriving**, and a chain of conformant pairwise steps can still break the composition silently); there is **no way to say a shape is not a commitment yet**, which §8.7's own strictness is what creates; and **nothing states when a vocabulary stops being editable**. **The experimental segment is designed against its own known failure** — the experiment succeeds and the marker becomes permanent — so the governing clause is borrowed verbatim from the PROTOTYPE tier already designed in this corpus for shell verbs: *MUST NOT sit provisional across two releases without a disposition*, written before this problem was posed and against a different surface. **The freeze trigger is behavioural and deliberately not the author's to declare** — a vocabulary freezes when any third party implements it, with or without permission — which fits here especially well because **publication is unilateral and permissionless, so an author cannot observe the event and must write as though it has happened.** **Version-in-the-name is recorded as a known wrong answer**: the version controls handling, not identity, and a numbered suffix invites the reading that the highest number is correct, which is false for a reader holding old data. **D2 is mechanically gateable** (an `x-` segment is greppable; surviving two releases is a comparison against release history). Four open questions, including whether `x-` is the right spelling — *the most recognizable marker is recognizable because of the failure it is designed against* — and a sourcing caveat: the mode names, the history and the trigger come from a prior study in this corpus and were **not re-opened for this draft**, so they want re-verification before ratification, though no argument rests on them.)* *(**2026-09-09: `KIND-AND-AUTHORITY-ARE-TWO-AXES-AND-A-DOCUMENT-MUST-DECLARE-WHAT-GOVERNS-IT` opened, RULED and FOLDED the same day — now in `implemented/process/`, and `DOCUMENT-CLASS-HEADER-FIELD` went with it as both proposals required.** The ruling was a two-level tier model — a rule true of every document goes in the corpus-wide standard, a rule specific to one family goes in that family's standard, and a document under no family belongs to the top level, which is itself a tier — under which all three of §7's open questions dissolve. **Four disciplines promoted, not three** (#8 is general: entity-type proliferation is everyone's problem); **`Governed-by` names only the sub-tier standard**, because the project tier binds unconditionally and its ABSENCE is the positive statement; **a document with no tier is a member of the project tier**, which is a position and not a gap. `SPECIFICATION-FORMAT` §10 rewritten on two axes, §5.3 gains `Kind`/`Authority`/`Governed-by`, §8 retitled and gains §8.6–§8.9 with **no existing number moved**, and the charter is `guides/GUIDE-APPLICATION-DEVELOPMENT.md` v2.0 with nine disciplines reduced to two plus a pointer table. **The `Governed-by` gate is filed, not built** — one sub-tier standard exists, so a gate written today would be calibrated against one example, which is the mistake `spec expiry` made in this same session. The audit as filed: Asks why one tier has a `CHARTER` and no other does, and finds the answer upstream of the question: **`SPECIFICATION-FORMAT` §10 is the corpus's only document taxonomy, it has two rows, and it welds `Authority: Binding` to one of them.** Empirically there are **six kinds and authority does not track kind** — two documents classed *guide* are normative and gated, and the one classed *informational* carries nine numbered disciplines that gate five specifications. **Neither case is a defect; the table says they cannot exist**, so each is an unlabelled exception. Of the applications charter's nine disciplines, **three are general rules stated nowhere else in the corpus** — the type-vocabulary compatibility contract and *a disposition property lives on the entity, never on its container* were grepped across `specs/` and `guides/` and appear only there, **so the 26 extension specs are governed by neither**, and extensions mint containers constantly. **The cost is discovery, not filing:** nothing in a spec's header names what governs it, so #5 contradicted `SPECIFICATION-FORMAT` §8.5 and `GUIDE-EXTENSION-DEVELOPMENT` §7 for its whole life and was folded into five specifications as their ratification gate. Proposes §10 rewritten on **two axes**, a **`Governed-by`** header field with a gate that also finds *a tier standard nobody declares*, the charter renamed `GUIDE-APPLICATION-DEVELOPMENT` to match the tier pattern that already exists, and three disciplines promoted with a one-line restatement left behind — which is #6's shape and the only shape that survives a sweep. **Rides with `DOCUMENT-CLASS-HEADER-FIELD`**, same root cause from the other end. **§5's moves want a ruling before any of them lands.**)* · *(**2026-09-08: `THE-CONFORMANCE-INVENTORY-IS-ADDRESSABLE-OR-IT-IS-PROSE` opened and FOLDED same session** — a conformance requirement had no identifier anywhere in the corpus, so a check could cite `§9.1` and that section holds fifteen obligations. `<PREFIX>-R<n>` in a table, `Level` a closed six with `MUST NOT` among them, allocated once and never renumbered; `SPECIFICATION-FORMAT` v1.3 §8.5a, `EXTENSION-HISTORY` v1.9 as the worked reference, `spec inventory` as the gate with a floor that only rises. **The filed finding's first clause was wrong**: §5.1 declared the shape and 19 of 26 follow it — no enforcement point, not no rule. Stays `active/` under L3: one spec of 26 is converted and the ratchet is the fold · **2026-09-08: `RETIRE-THE-CONDENSED-WORKING-REFERENCE` opened and RULED and FOLDED the same day, now in `implemented/process/`** — `ENTITY-SYSTEM-REFERENCE` is the document class `SPECIFICATION-FORMAT` §8.4.3 forbids authoring and was found wrong in the only section ever examined. **The 49 citing files across 6 repos were read as a reason to hesitate and are not one:** a document that was never a citation target does not acquire authority by being cited, and a stale citation's fix is to cite the source. Arch's half was the archive move, the section→source banner, the undeclaration, and **two live spec citations** — one of them a normative `MUST` in `EXTENSION-SIGNALING` §6.5 naming it as the authority for core's own `scope_subset` relation. No cross-repo worklist · 2026-08-31: `DEVERSION-TEST-VECTOR-CORPUS` **moved to `implemented/`** — the ECF half landed with the `ENTITY-CBOR-ENCODING` Appendix E amendment (§9) its own §8.2 said was owed; `spec corpus` 1 → 0 · 2026-08-17: `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1` **re-tiered out to `core/`** — a core-protocol §6.2/§6.9a.1 defect that had been filed here, which routed it to arch-tools instead of to the three seats that implement the surface · 2026-08-17: `DOCUMENT-CLASS-HEADER-FIELD` opened — `specs/` holds normative specs, guides and architecture references with no way for a document to declare which it is, so the class lives only in a tool's filename-glob map and defaults to `canonical-spec` on silence; two files invented `Authoritative scope:` and two analyzers disagreed about the same corpus · 2026-08-17: `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1` opened — a core-protocol contradiction filed by browser-rust (F6) and verified here: §6.2 closes the policy-path key to two forms, §6.9a.1 defines a three-form resolution order at the same path · 2026-08-15: `CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` opened — reference
proposal for the §2.4a/§5.2a/§5.2b/§5.2c rules, folded 08-09…12 with no proposal; see
`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`)* · *validated by core-go (oracle) / arch-tools (gates)*

`CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` *(reference)* · `HASH-WIDTH-IS-NEVER-FIXED` *(reference)* · `CONFORMANCE-ORACLE-CONTRACT` *(the register is built and running — folding it describes what is
already there)* · `CORPUS-REFERENCE-INTEGRITY` · `DEVERSION-TEST-VECTOR-CORPUS` *(the `v767-`
prefix rename **is** the fold)* · `MATURITY-MODEL-AND-ROADMAP-COMMUNICATION`

**Nothing external blocks any of the four.** They touch no wire format and no impl behavior.

**The oldest are three weeks old and nothing has moved them.** That is not automatically wrong —
several are deliberately parked — but it is not visible anywhere else, which is why it is stated
here.

## 2. Implemented — 85 (moved this cycle)

**`THE-BRIDGE-EXTENSION-FAMILY-AND-THE-DIRECTORY-THAT-NAMES-NOTHING` — OPENED AND FOLDED 2026-09-14.**
`specs/domains/` is retired and **`specs/bridge-extensions/`** replaces it, a **sibling** of
`specs/extensions/` and not a merge into it — which reverses the proposal's own first draft. The
argument it reverses is sound and is recorded rather than deleted: *a bridge IS an extension by
artifact class, and being the same kind of artifact is not an argument for being in the same place.*
`specs/extensions/` is the entity system's own surface; `specs/bridge-extensions/` is how the system
reaches technologies that are not it, and a reader opening the first should not be taught that a
package manager is part of the system. Also landed: `guides/GUIDE-BRIDGE-EXTENSION-DEVELOPMENT.md`
(new, canonical — the three layers, the no-umbrella ruling, the direction-mode axis, the seven
questions, the boundary hazards), the `EXTENSION-BRIDGE-` family naming, a bridge note beside
`SYSTEM-ARCHITECTURE` §13.1's five tiers, the §2 layer diagram, a `ROADMAP-EXTENSIONS` roster section,
and the standing-on/reaching-out rule in the reserved-prefix table.
**Three audit findings preceded the fold and two of them invert what the opening study published:**
⛔ the dangling citations are **nine across seven documents, not two** — five of the seven are landed
specs, and a **second technology** (a mail bridge) was never surfaced; ⛔ **bridge material is already
in the landed corpus** — `EXTENSION-REVISION` §10 carries a complete version-control bridge mapping
plus an operation sketch, and a bridge is one of `GUIDE-PEER-COMPOSITIONS`' seven named compositions;
⛔ **three prefixes are in play and the reserved one is the empty one** (`bridge/` reserved with zero
occupants · `system/bridge/` in the web-bridge forward refs · `app/bridge/` in the version-control
sketch) — now stated in three places so the first bridge authored decides it rather than setting it by
accident. **The rename question is RULED and dropped:** measured across eight trees excluding build
output and vendored copies, the namespace is **438 files**, the document name a further **119 of which
67 are source code**, and the directory path only **7** — so the correction to the misnamed member is
an **additive successor**, never a rename.

**`A-CONVENTION-STATES-WHAT-A-CHECK-MUST-DISCRIMINATE-AND-SHIPS-NO-ARTIFACT` — FOLDED 2026-09-09**,
`specs/applications/CHARTER.md` **v1.1**. A correction, not a design: discipline **#5** said each
convention *"ships example entities + expected hashes"* and made that the ratification gate, which
assigns the artifact to the party that does not build it and inverts the sequencing. **The rule the
rest of the corpus already states** — `GUIDE-CONFORMANCE` §5.1a's `[MUST]` (*architecture sets fields,
the encoder settles bytes*), §7.0's per-class `Authored by` column, §7c.6's *"a hash computed by hand
would make architecture the oracle"* — **had never reached this domain's charter**, so five conventions
were recorded `NOT ratifiable` on work their author does not perform. **A specification names the cases
a check must discriminate and what each asserts; the implementations and the conformance oracle produce
the fixtures, the bytes and the run — and the set is pinned after two implementations exchange the
format, not before it.** All five members' sections re-titled; **no case, id or level was edited**, and
`#7`'s day-old enforcement point was re-attributed in the same pass. Register: `AP-6a`.

**`APP-CONVENTION-FEED` — FOLDED 2026-09-09**, the applications domain's **fifth** member and **the
crux of the social tier**: the entry, the key-addressed index, the bounded collection and the mirror —
the vocabulary that makes *following someone across independent hosts* a format two implementations can
both produce and both read. All nine deltas landed.

**Three new charter disciplines came with it, and they govern the domain rather than this member.**
**#7** the compatibility contract, with its enforcement point named — *a shape change fails cross-impl
comparison*, because a naming divergence otherwise has no discovery path at all: a
type-filtered query on the wrong tag returns a correct, complete, **empty** answer. **#8** the growth
rule — a new product is a new body type or a new renderer, and a new *entity type* only if a conformant
consumer must behave differently. **#9** the disposition rule — *a property governing what a consumer
may DO with an entity lives on the entity, never on its container; containers do not travel, entities
do.*

**The content-site convention's deferral is discharged on its own stated terms** — it deferred the
semantic layer *"to a named feed/index convention authored when a concrete feed use case arrives"*, and
§4 is that, **with the determinism floor explicitly unchanged and still governing raw enumeration
views.** The share convention gains the `follow`-tag distinction it needed: *that* one follows a
**grant** and the publisher knows you exist; this one follows a **namespace** and they cannot.

**The eleven open items moved INTO the spec (§12) rather than folding away**, because the proposal's own
rule was that an open item folding into an implemented proposal *"becomes invisible exactly when it
stops being true."* Ids changed `[OPEN-FEED-n]` → `F-n`, mapped in the proposal's §14.3.

**Authored; not yet exercised** — §11.2 names eleven required checks and the implementers build and run
them (charter #5, corrected 2026-09-09). `FEED-3`/`-5`/`-6`/`-9` are the
load-bearing four; **`FEED-10` is the one that fails loudest if the design ever drifts back toward a
hash back-chain.**

**`PEER-TRANSPORT-SET` — FOLDED 2026-09-09.** `EXTENSION-NETWORK` **1.8 → 1.9** (new **§6.5.1c**,
`system/peer/transport-set`; §6.5.1a gains **D6** absence-fails-closed and **D7** positional
authority; §6.5.4 rewritten; §6.5.2d, §6.7, §12.1 and §13 updated) · `EXTENSION-REGISTRY` **1.25 →
1.26** (§2.3 and §12, two withdrawals that were **circular** — each cited the other for a scoping
neither established) · `EXTENSION-TREE` §3.3a cross-reference. D8/D8a/D8b were already folded at
v1.21.

**A peer's own signed, complete, expiring statement of where it is**, members carried inline, bound
into the tree so a peer with a published root has **one `seq` stream and not two**. The signature is
what makes the lookup problem easy: the record is self-authenticating, so **no retrieval path is a
trust boundary and anyone may serve one** — there is no lookup service to specify or arbitrate.
Signing the **complete set** rather than the members is the load-bearing half, and it is the only
thing that lets a consumer detect a **dropped** member.

**Two things the fold added, because a normative obligation needs an instrument that reads it:** a
conformance table in §12.1 naming a vector per consumer rule, and the §13 type row. **The vectors are named and
OWED.** The two flagged as the ones nobody writes from the prose are the **per-source `seq` floor**
negative and **empty-set-is-an-answer** — both fail silently.

**One home the delta table did not name was folded with it:** D5(b) named Amendment 13's dated
build-state paragraph; **Amendment 14's banner carried the identical class one paragraph over**, and
cleaning only the named one would have been the exact failure that delta exists to correct.

**Ruled without the §10 cross-implementation round.** **§12.2 of the proposal records what that
round was for and does not pretend it was answered** — T1/R7 in practice, and the
wire consequence of the inline decision, for which §6.5.1c already carries an authoring escape hatch.
**§11's residue (the peer-keyed rendezvous mode, the intermediary locator) is NOT folded and stays
open.**

> **The payoff is on the other side of the corpus: `[OPEN-FEED-5]` is CLOSED**, and it was the only
> item on `PROPOSAL-APP-CONVENTION-FEED`'s twelve-item residue that gated **stage 1**. A reader
> holding a bare peer id now has a normative way to reach its origin.

**`THE-REFERENCE-ATOM` — FOLDED 2026-09-08** as `specs/applications/APP-CONVENTION-REFERENCE.md` v0.1,
the applications domain's **fourth and foundational** member: one tagged atom for *"this points at
that"* (`pin` / `live`, `peer` + `hash`-or-`path`, optional `at` anchor and optional advisory `via`
hints) plus a canonical `entity+ref://` string form with a normative round-trip, RFC 3986 mapping and
normalization rules. **It is the single home for the tier's shared `content-hash` / `peer-id` /
`tree-path` atoms**, which the other three members had each been defining locally.

**Two deltas were adjudicated differently from the proposal and are recorded in its §12**, because a
grep of the fold would otherwise read as incomplete: **the pointer slots do NOT uniformly take the
atom.** Only `EMBED`'s `child-payload.ref` converted — it was an **untagged** `(path / content-hash)`
union and is the only payload that can cross a peer boundary. `EMBED`'s `pointer-payload` and
`img-src` and **both** of `SHARE`'s targets stay same-peer, and the spec states the general rule
instead: they are the **implied-authority form** of the atom, the rung `site:` occupies. Adding a
required `peer` to a slot whose authority is already known just gives it somewhere to be wrong.

**Two defects were found by the fold, and both were in the premise rather than the design.**
`peer-id` was typed **`bstr` in the site convention and `tstr` in the share convention** — the site
comment cites core §1.2, the *content hash* section — while core §1.5 defines `PeerID` as a Base58
form and both deployed link classifiers carry it as a string. **Ruled `tstr`; the site convention is
corrected**, at zero cost since the only field using it has no implementation anywhere. And `via`'s
hint vocabulary (`origin`/`mirror`/`peer`) **could not express the tree-path locator it was supposed
to absorb**, so a `path` hint tag is added with the *a 404 here proves nothing* obligation made
normative. **Authored; not yet exercised** — §6.2 names eleven required checks (charter #5, corrected
2026-09-09), the same state the share convention is in.

**`EXTENSION-HOST-INSTALL-SEAM` — FOLDED 2026-09-08.** T1's deliverable #0. The corpus had three words for the party that installs a handler — §6.2 *"user-installed"*, §9.1 *"user"*, §11.6.7 *"application-owned"* — all meaning **application code**, so the **extension installer** was named nowhere and every one of the 26 standard extensions, all under `system/*`, was refused `403 forbidden_pattern` on the only install path the corpus admitted existed. Landed as `ENTITY-CORE-PROTOCOL` **0.8.2.12** (§6.2 scoped to the **dispatch path**, carrying the shadowing rationale it had never stated in any revision; §9.1 follows), `SDK-OPERATIONS` **v1.12** (§11.6 names both callers and forbids an SDK applying §6.2 inside the in-process primitive; the decline becomes **declared** rather than inferred), `GUIDE-CONFORMANCE` **§7d** (the host-seam check class — driven in-process by the peer's own harness, asserted over the wire, reference body required to return something **no `compute/literal` can produce**, both controls mandatory, plus the consumer-ordering row nothing tested), `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1, and `SPECIFICATION-FORMAT` **v1.2** §4.1 — *a requirement that constrains a party, a namespace or a path states the failure it prevents.* **Two homes the proposal's own L23 list missed, both found by sweeping for the rule's subject rather than its tokens:** `EXTENSION-TREE` **v4.7** carried the retired party-based wording, and `EXTENSION-COMPUTE` **v3.28** had declared itself *"a subset of"* the core reservation — which **inverts** under the narrowing, since the compute override prohibition becomes the only thing stopping an in-process installer from rebinding `system/compute/builtins/arithmetic`. **The transferable form: when a fold narrows a rule, sweep for the documents leaning on the part being removed, not for restatements of the rule** — those do not share its vocabulary. **The build half remains owed and has been delivered to the seats that own it:** the generation phase contract, the profile's host-declaration block, the host-seam harness and one regeneration sweep on the generation side; the §7d transport with both controls on the conformance-oracle side.

> **+1 on 2026-09-06, and it is a PULL-IN rather than a fold:
> `PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT` (v4.3, LANDED in legacy).**
> `specs/extensions/EXTENSION-TREE.md` cites it **three times** as the authority for the v4.0 substrate
> fork — including **`§3`, *"why IPLD HashMap, not JMT or other alternatives"*** — and **it had never
> existed in this repo, in any commit.** So the alternatives analysis behind the largest structural
> decision in the substrate was unreadable from the spec that rests on it, and a coverage audit
> correctly measured `merkle search tree: 0` and drew the wrong conclusion from it.
> On the **2026-08-17** dangling-citation worklist (3 citations, `pull-in` disposition) for twenty days;
> the judgment that rule requires was done first (§3.4 and §4.7 read against `EXTENSION-TREE` §3.7.2).
> **Copied verbatim, not rewritten**, per that rule. See
> `EXPLORATION-CERTIFICATE-TRANSPARENCY-…` §7.

> **+1 on 2026-09-06: `THE-PUT-ADMISSION-PREDICATE-AND-THE-CONTENT-CODE-TABLES`, written and folded in
> one session.** Answers `entity-core-go`'s `ROUTING-2026-09-06` — five sentences and a ruling, of
> which four turned out to be one rule the corpus already carried in `core/entity`. Nothing was
> unknown once the type was cited, so it did not sit in `active/` — **L26** again.

> **+1 on 2026-09-03: `PIN-THE-PATH-REQUIRED-STATUS`, written and folded in one session.** Filed from
> `entity-system-generator`'s first read of CONTENT for code emission: `path_required` is a MUST in
> eight sites across five documents and none named a status. Nothing was unknown once the derivation
> was written, so it did not sit in `active/` — **L26**, and the reason the fold-debt count is held at
> zero.

> **+1 on 2026-08-18 by PULL-IN, not by a fold.** `PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND` is cited by
> `EXTENSION-REGISTRY` §6a as the home of the design rationale and the P1–P7 rulings, and **was not in
> this corpus** — two implementations carried a security bound quoting its §5 from a document nobody
> here could open. It exists in legacy at `b42fd2e`, ratified; copied unmodified per the disposition
> rule in `STATUS-2026-08-17-the-dangling-citation-worklist-…` (*"copy it, do not rewrite it"*). One
> of the 32; the §6a citation now resolves.

> **Count corrected 2026-08-15: the table said 14 and the directory held 15.**
> `REGISTRY-SERVICE-ADVERTISEMENT` was moved to `implemented/extensions/` on 2026-08-14 and §1's prose
> records the move, **but it was never added to this table.** Rule 1 — *a state change moves the file* —
> was half-applied: the file moved, the ledger did not. Exactly the drift this file exists to stop,
> committed in this file, which is why the count is now stated against a directory listing rather than
> maintained by hand.
>
> **Corrected again the same day, 15 → 16 — and the note above is why this one is worth reading.**
> `PROPOSAL-EXTENSION-ENCRYPTION` was *created* in `implemented/extensions/` a few hours after that
> paragraph was written, and the table was not updated. **Third instance of one drift in one file,
> the second inside a single day.** The paragraph above said the count would now be "stated against a
> directory listing rather than maintained by hand" — **it was not mechanized, so it stayed
> hand-maintained and drifted again.** A resolution recorded as an intention behaves exactly like the
> rule it was written to replace. **This is `W-CORPUS` C11's shape at the ledger layer** — the count
> is checkable (`find docs/proposals/implemented -name '*.md' | wc -l`) and nothing checks it, which
> makes it a gate ask, not a discipline ask.
>
> **Note the entry class, because it is new:** this proposal did not *move* from `active/`. It was
> **reconstructed directly into `implemented/`** — a design record rebuilt for a spec that cited a
> proposal absent from every post-split checkout (C12). **Rule 1 is written about files that move and
> says nothing about files that arrive**, which is the seam this fell through.
>
> **23 → 24, 2026-08-17 — the same arrival class, and this one was *pulled in*, not reconstructed.**
> `PROPOSAL-APPLICATIONS-DOMAIN-AND-CONVENTION-EMBED` is the founding record for the whole
> `applications/` domain: it stands up the domain, authors `APP-CONVENTION-EMBED` v0.1, and is cited by
> name as the originating proposal by **both** `specs/applications/CHARTER.md` and
> `specs/applications/APP-CONVENTION-EMBED.md`. It existed the entire time — in the legacy tree, not in
> this one — so the citations resolved for a reader who knew where to look and dangled for everyone
> else. **The distinction matters and is the general rule for this class:** a design record that is
> *named by a landed spec* and *exists in legacy* is a **pull-in**, and writing a fresh one would have
> been duplicating a document we already had. Reconstruction is only for records that are genuinely
> gone. See `docs/status/` for the dangling-citation worklist this came off.

| Proposal | Landed in |
|---|---|
| `APPLICATIONS-DOMAIN-AND-CONVENTION-EMBED` | `specs/applications/CHARTER.md` v1.0 (the domain + its five disciplines) + `APP-CONVENTION-EMBED` v0.1 (since v0.2.3) — **pulled in from legacy 2026-08-17, never moved from `active/`**: it is named as the originating proposal by both specs and existed only in the legacy tree, so the citations dangled in every post-split checkout |
| `REGISTRY-SERVICE-ADVERTISEMENT` | `EXTENSION-REGISTRY` §3b (v1.4 → v1.5), folded 2026-08-14 |
| `AVAILABILITY-DESCRIPTOR` | `SPECIFICATION-FORMAT` §8.2a + §8.5 vectors — **folded 2026-08-15** as an authoring standard, not an extension: a reusable field shape with no operations and no wire mechanism. Unblocks `SYSTEM-DEVICE` |
| `COMPUTE-ALT-ENGINE-ADMISSION` | `EXTENSION-COMPUTE` §11 (v3.22) |
| `COMPUTE-BUDGET-CEILING-DETERMINISM` | `EXTENSION-COMPUTE` §4.2 (v3.21) |
| `CONNECTION-NODE` | `EXTENSION-SIGNALING` |
| `CONNECTIVITY-SIGNALING-AND-PUNCH` | `EXTENSION-SIGNALING` |
| `CONTINUATION-BOUNDS-PROPAGATION` | V7 §3.11/§5.9/§4.10(b) as amendment 0.8.1 (`1fcda2b`) + `EXTENSION-CONTINUATION` §3.6/§3.7 |
| `CONTINUATION-LOST-ERROR-MARKER-MUST` | `EXTENSION-CONTINUATION` v1.23 §3.4/§3.10.5/§8.1 — **folded 2026-08-15**. Delta 2 landed earlier as §3.10.7; Delta 1 was held on NETWORK §A6.1 and released by that fold the same day |
| `CONTINUATION-STANDING-MODEL` | cycle closed 2026-07-29 |
| `CORE-TYPE-EXTENSION-TIERING` | `SPECIFICATION-FORMAT` §8.4.1 |
| `EXTENSION-ENCRYPTION` | `EXTENSION-ENCRYPTION` v1.0 (the whole spec) — **reconstructed 2026-08-15, never moved from `active/`**: the original did not survive the repo split and the spec cited it as absent. Carries the §17/§21/§16.5 design-decision and lock-protocol material that left the spec in the same change |
| `EXTENSION-SIGNALING-COORDINATION-ENVELOPE` | `EXTENSION-SIGNALING` §6.2 |
| `EXTENSION-WEBRTC-TRANSPORT` | `EXTENSION-SIGNALING` §6.5 + `EXTENSION-NETWORK` §6.5.2d/§6.5.1b |
| `NAMESPACE-CLEANUP-AND-BROWSER-LEG` | `EXTENSION-SIGNALING` §3.1/§6.1/§6.5 · `EXTENSION-INBOX` §2.1 · `EXTENSION-SUBSCRIPTION` §2.2 · `SPECIFICATION-FORMAT` §8.4.4 · `ENTITY-CBOR-ENCODING` §5.4 — **closed 2026-08-15, all four workstreams folded.** Residue is cohort execution (S0–S5, the machine-spec §1.8 deletion), routed to owners |
| `NETWORK-LIVE-ESTABLISHMENT-SEAM` | folded 2026-07-31 |
| `NETWORK-LIVENESS-REACTIVE-BUILDOUT` | `EXTENSION-NETWORK` Amendment 12 — **folded in full 2026-08-15** across four partial folds (§5.4a · §A6.6 · the retry lifecycle · then §A3/§A4/§A6.0/§A6.1 as §4.1.1/§5.4.1/§12.1.1). Ratified 2026-07-14, built green three-way 07-17, and the fold trailed the build by a month |
| `NETWORK-REACHABILITY-FACTS` | `EXTENSION-NETWORK` |
| `PUBLISHED-ROOT-PREFIX-AND-REPUBLISH` | folded 2026-08-08, cohort-reviewed |
| `SYMMETRIC-REENTRY-MUTUAL-MINTING` | folded 2026-08-05, both impls acked + built the classification half |
| `SDK-HANDLER-OWNED-SERVICES` | `SDK-OPERATIONS` §11.6.9 (v1.11) — **folded 2026-07-29, mis-filed DRAFT for two weeks** |

**Two path-form citations were repointed** to `proposals/implemented/…` — the rest cite by name
and are unaffected.

## 2a. Deferred — 1

`IDENTITY-CROSS-ALGORITHM-MIGRATION` — its own header says *"foundational, NOT before-freeze"*.
It was never active work; counting it as active is the kind of lie that makes the number
meaningless. The pre-split repo had a `deferred/` directory holding 12 such items and this repo
did not, which is why everything parked read as in-flight.

## 3. Superseded — 3

`INBOX-TYPE-NAMESPACE-CORRECTION` → consolidated into `NAMESPACE-CLEANUP-AND-BROWSER-LEG` §3.2/§4.
`NAMESPACE-SEGMENT-DISCIPLINE` → rule folded to `SPECIFICATION-FORMAT` §8.4.4.

Kept, not deleted — both carry reasoning the surviving document does not restate.

## 4. Peer-implementation — MEASURED 2026-08-13, and the answer inverted the backlog

**Superseded: this section previously read "NOT TRACKED, and this is the real gap," and said the
peer-side state of these proposals was unknown.** It was not unknown. Eight sibling worktrees are
checked out on the same disk and nobody had opened them. The audit is
`docs/status/STATUS-2026-08-13-f-the-proposal-audit-measured.md`; it took one session.

**The rule stands and is now load-bearing** (`AGENTS.md`: *"stay in the spec lane … don't track
what impl teams owe"*): what is recorded here is **observed coverage, pinned** —
`(repo, commit, date, how-observed)` — never a checklist of obligations. A cell whose pin is older
than the tree it describes reports **`unmeasured`**, never its last value.

### 4a. The state the three directories cannot express: **built, not folded**

The directory model (§0) has no square for *the cohort implemented it and the spec never landed*.
That is why this class was invisible, and it is **seven proposals, not the one found by accident**.
Tracked here until each folds:

| Surface | Built | Spec | State |
|---|---|---|---|
| A12 §A6.6 `chain_id` spelling | go/rust/py | **FOLDED `a14b1be`** | ✅ closed |
| A12 §A2/§A6.2–§A6.5 (`reason` vocab, retry bounds, `retry-exhausted`, `failing_since` + schedule) | go/rust/py — **wire-measured 3-of-3** (core-go `a02ab5e`; rust `1152d35`, py `ad0ef98`, `network_reconnect_anchor` PASS 5/5 0 skips) | **FOLDED** — core-protocol §3.13 `9829c6f` (fields) + NETWORK `dd170bb` (semantics) | ✅ closed |
| A12 §A3 floor framing · §A4 transition-writes · §A6.1 `on_error` carve-out | go/rust/py | unfolded | **fold next** — §A6.1 releases LOST-ERROR-MARKER Delta 1 |
| A12 §7.2 remote-peer re-subscribe | go `handler.go:213` · rust `lib.rs:895` · py `network.py:409` | proposal header says *"unbuilt all three"* | **strike the false claim** |
| REGISTRY service-advertisement + §3.1.1 HRW | go/rust/py; core-go confirms their HRW **matches §3.1.1 on all four pinned variables** | `EXTENSION-REGISTRY` v1.2 has **zero** occurrences | **ratify** (ruled) — fold §1/§2/§3.1/§3.1.1 |
| `compute/error` code-only | go + rust (`engine.rs:320`, test `eval/tests.rs:3282`) | §2.4 still stamps *"validated intra-Go"* | correct the stamp; py gate is `ROUTING-…-i` |
| COMPUTE-LOWERING-TOOLKIT §5 worked-vector sweep | core-go `cmd/internal/compute-corpus/worked.go` — **names the proposal and its §5** | proposal §6 ledger still says the sweep is *"Unbuilt"* | **the proposal's own ledger is stale** — correct it |
| DEVERSION corpus rename | core-go `cmd/v767-corpus-{build,verify}` — *"the `v767-` prefix is what the proposal de-versions, so the rename **is** the fold"* | unfolded | mechanical; unblocks keystone re-vendor |
| `app/program` descriptor + static-k host | workbench `programs/{descriptor,host}.go` | DRAFT | reconcile with the consumer, then fold |
| `app/chat` convention | browser-rust `src/views/chat/` (**tree dirty — measurement void until it settles**) | DRAFT | reconcile with the consumer, then fold |

**Confirmed by `entity-core-go` at `a02ab5e` (`ROUTING-2026-08-13-k`): all six routed rows confirm,
none refute.** Extended at `4d10189` (`ROUTING-2026-08-13-l`) with four exact receipts where
**their code names our document** — the strongest evidence class on this board, because it is not
a vocabulary match.

**CLOCK scope corrected by core-go, adopted:** `PROPOSAL-CLOCK-ADDRESSABLE-TICK-AND-DEADLINE`'s
**tick half is already folded** (`tick_interval`, `system/clock/tick/latest` — `EXTENSION-CLOCK`
§2.5/§2.8, verified). Only the **addressable/deadline** half (`tick/{sequence}`, `at/{ms}`) is
owed. Our audit's "genuinely absent" was right about the addressable surface and too broad about
the proposal.

### 4a-i. **Do not close a proposal on vocabulary presence** `[rule, earned twice]`

`entity-core-go` built a detector, found its own first number wrong, and reported the correction
unprompted: matching a proposal's vocabulary against an implementation **cannot distinguish a
proposal folded a year ago from one never folded**, because a folded proposal's vocabulary is in
the spec *by definition*. That correction moved ten rows off their owed list.

**The same error survives one level deeper in the corrected list, and arch must not act on it.**
Adding "…and in no landed spec" still cannot distinguish:

| | Vocabulary in spec because | Fold state |
|---|---|---|
| (a) | the proposal folded | done |
| (b) | **the proposal's *subject* already existed and the proposal asks to CHANGE it** | **nothing folded** |

**`IDENTITY-ROTATION-AUTHORITY-PARITY` is case (b) and is the sharpest instance.**
`identity-rotation-handoff` appears 23 times in `EXTENSION-IDENTITY.md` **because the kind
predates the proposal**. The proposal's actual ask is `issuance-parity` — **0 occurrences corpus-wide**
— and §4.1/§4.3/§14.2 still read *"dual-sig (old + new)"*, which is precisely what it asks to
replace. It is **open, security-relevant, and closing it on a vocabulary hit would have been the
worst outcome of this exercise.**

Re-verified against the same test, all **0 occurrences in `specs/`+`guides/`**, all still open:
`COMPUTE-BUDGET-PREEMPTION` (`reduction_budget` — also 0 in all four impl trees) ·
`COMPUTE-COLLECTION-PRIMITIVES` (`group_by`; core-go builtins are exactly
`{construct,map,filter,fold,store}`) · `SYSTEM-DEVICE` · `INBOX-OPEN-DELIVERY-ADMISSION`
(`chat-requests`) · `EXTENSION-PACKAGE` (`build-provenance`; core-go flags their own hit as a
constant marked `// future`) · `SUB-PEER-ISOLATION-MODEL` (I-5 confirmed net-new by both seats).

> **Rule.** A proposal is closed by **reading it against the landed spec section it targets** —
> never by a token census, in either direction. A census orders the queue; it does not close a
> row. Both seats have now made the inverse error once each, which is the argument for writing it
> down rather than remembering it.

### 4b. The standing question this class raises

**Three implementations interoperate on an unratified DRAFT** (REGISTRY service-advertisement).
That is not a filing error — it is the cohort depending on a document arch never voted on, and
`guides/GUIDE-REFERENCE-DEPLOYMENT.md:71` already cites it as a rule. **It ratifies or it
retracts; it does not stay DRAFT.** Raised by core-go as a call for arch, and it is the right call
to escalate.

### 4c. Why this went unseen — the tracking rule that follows

Every automated signal was green. core-go's conformance register resolves every check's citation
against the live spec trees, and **resolution was the only thing anything measured** — so a check
citing `EXTENSION-NETWORK §4.1`, where §4.1 carries the *wrong* rule, resolved perfectly and read
as healthy. **Resolution and conflict are orthogonal** (core-go, `ROUTING-…-k`; arch's own routed
question had the wrong population in it — it asked about the 212 *unresolved* rows, and the fold
debt lives in the ~954 *resolved* ones).

> **Tracking rule, adopted.** A proposal header MUST NOT carry an impl-state sentence without
> `(repo, commit, date, how-observed)`. `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT` §B is the
> only proposal that does this correctly — a fenced banner reading *"superseded, do not re-cite as
> current"* — and it is the template.

**The full matrix (`CROSS-IMPL-PARITY-MATRIX.md`) is still owed** and is now seedable from
measurement rather than from headers. §4a is its first content.

## 5. A defect this audit turned up

`GUIDE-REFERENCE-DEPLOYMENT` §3 cites **`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` §3** for *"the
prefer-cheap-path rule"* — as a rule, in a user-facing guide, from a proposal that is **DRAFT and
unfolded** (`EXTENSION-REGISTRY` has no advertised-service entity). **A guide that depends on an
unratified proposal has given it normative force nobody voted on**, and it is the same shape as a
conformance check citing a proposal's vector list (`PROPOSAL-CORPUS-REFERENCE-INTEGRITY` §4).
Either ratify the proposal or rewrite the guide passage. Not fixed here — it is a ruling, not
hygiene.

## 6. Rules

1. **A state change moves the file.** Same commit as the status-line edit.
2. **Fold ≠ ratify.** A proposal moves to `implemented/` when the **spec edit lands**, not when
   it is agreed. ("Ratified ≠ folded" — `AGENTS.md`.)
3. **Partial stays active.** `NAMESPACE-CLEANUP` and `NETWORK-LIVENESS-REACTIVE` both have
   landed halves and both stay in §1. A proposal is done when it is *all* done.
4. **Superseded is not deleted** — archive with the breadcrumb, per `AGENTS-STANDARD`.
5. **Citing an implemented proposal uses `proposals/implemented/…`** — and per
   `PROPOSAL-CORPUS-REFERENCE-INTEGRITY` §4, a *conformance check* cites the landed spec section,
   never the proposal. A proposal is rationale; the spec is the rule.
