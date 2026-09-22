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

**`active/core/` is currently empty, and that is a finding rather than an omission:** nothing in
flight targets the V7 core or the wire, which is what "the locked wire core is never renumbered"
looks like when it is working. Several extension proposals carry *routed* core deltas (NETWORK's
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

## 1. Active — 32 files (ext 19 · app 9 · process 4 · core 0)

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

**`active/core/` — 0, and it emptied the same day it filled.** *(2026-08-17: three opened out of one cohort packet — `entity-browser-rust`'s `ROUTING-2026-08-17-c` A1/A2/A3 plus `entity-core-go`'s B — ruled, folded into `entity-core-protocol` `30ca731` as `0.8.1 CAP-1`…`CAP-7`, and moved to `implemented/core/` the same session: `CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL` · `CAPABILITY-MINT-TEMPORAL-CEILING-AND-THE-WITHDRAWAL-BOUND` · `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1`, the last **re-tiered in from `process/`** on the way through.)* · *validated by go / rust / py; **conformance-validated by nobody yet** — `GUIDE-CONFORMANCE` §9 checks (r)–(v) are the validating half and are unbuilt, so this row means the text landed, not that the cohort agrees.*

> **`active/core/` being empty was read as a finding, and it was — but not only the one recorded.**
> The note below said an empty core roster is *"what 'the locked wire core is never renumbered' looks
> like when it is working."* That reading still holds for the **wire**. What it obscured is that
> §6.2's capability-handler surface is core-tier and had been accumulating unrouted defects the whole
> time — four of them, surfaced in one day, once two seats went looking at the same section from
> opposite ends. **An empty tier is evidence about what is filed, never about what is defective**, and
> the one proposal that did target this surface was filed under `process/`, where its routing seats
> were arch-tools rather than the three impls that build it. *(The tier axis exists precisely to stop
> that — §0a. It stopped it for `applications/`; nobody re-checked `core/`.)*

**`active/extensions/` — 19** *(**2026-08-21 — `COMPUTE-CLOSURE-RESULT-POSITIONS-AND-CONCAT-ARGS-SHAPE` opened, extended and FOLDED the same day, `EXTENSION-COMPUTE` 3.26 → 3.27, D1–D8; now in `implemented/`.** The three §3.5 corners v3.26 left open, routed by `entity-core-rust` and `entity-core-py` and concurred by `entity-core-go` (spec-issue `2026-08-21-a`) — all three measured, none converged unilaterally, because each forks a boundary hash. **(1)** `map` containing a value-form error and short-circuiting a minted one is **provenance-dependence**, which **§2.4 already forbids** — *"the in-flight representation is implementation-private"* — so the exhibiting seats are non-conformant against **v3.26**, not against this proposal. Contained set **three → five** (`map`'s output element, `fold`'s final accumulator), `filter`'s predicate result **short-circuits** (read for truthiness — containing it silently drops the element), and the *"exactly three positions"* **count is replaced by the rule that generates it**, since an enumeration written at the width of one table is wrong again at the next primitive. **(2)** `concat-args.collections` → a **single** `system/hash` — and **arch's published derivation for it was FALSE and is retracted.** It argued §7.1's `walk` descends only on scalar hash fields, so the array-of-hashes shape leaves every `lookup/tree` inside every concat sub-collection unregistered. **No implementation has that defect** (go `c1b0708`, rust `2ee6bf7`, py `f09ae70` all descend into containers, implementing §7.1's *prose* rule rather than its pseudocode), and **`concat-args` was never the only container-of-hashes** — `compute/apply.args` is `{map_of: system/hash}` and `compute/let.bindings` is an array of maps carrying one, both in §2.1, both older and far more common. Arch reasoned from **this spec's own pseudocode** to a claim about three implementations' behaviour: L8's fifteenth form. **The standing reason is uniformity + arity** — every sibling collection argument is one scalar hash of an expression, and the literal-array shape alone **freezes `concat`'s arity at authoring time**, making a concat over a computed number of collections inexpressible. **(3)** Entity-key equality is over the **materialized** form — C-9's ruling reached from the key side. **(4) D5/D6 — the evaluation-limit dispositions**, ruled from §5.1's *"restored on return"*: `depth_exceeded` **contains** (element-local), `budget_exhausted` and `cascade_limit` **short-circuit** (cumulative / chain-wide — and §10.4 makes memoization impl-defined, so containment would let two conformant peers exhaust at different elements and fork the boundary), **keyed on the `code` in both arms** since §7.3 *mandates* the value form's creation. **(5) D7** — D3's rollout: no lockstep and no tolerance in the checker (dual-kind acceptance, which this corpus forbids); named landing order, transient FAIL labelled and never baselined. **(6) D8 — chasing the retraction found the real defect:** §7.1 **contradicts itself**, its prose requiring every reachable `lookup/tree` to register while its pseudocode enters no container, so it registers nothing inside any function argument or any `let` binding. Prose promoted to `[MUST]`, pseudocode corrected, and the scalar-reference grammar invariant stated with its two enumerated exceptions. Class declared per L19: a **`validate-peer`** check, not a §7c corpus vector — dependency registration produces no boundary, which is how three walkers of three different strengths passed everything.)* *(**2026-08-20 — `REGISTRY-PIN-AUTHORITY-CAPABILITY-ENCODING` opened + FOLDED same day, REGISTRY 1.19→1.20; now in `implemented/`. It gated `REG-DISPATCH-CONFIG-REFUSED-1` row 8, which is now green three-way on the wire — `entity-core-go` drove its own harness against fresh rust `951f0ce` and py `5d9fdd1` peers from one seat, 18/18 · 0F · 0S at all three, which is a live cross-impl run rather than three self-reports.** From `entity-core-go`'s spec-issue `2026-08-20-a`. §4.3's pin-delta `[MUST, v1.19]` requires `system/capability/registry-pin`, and **§5 makes it name the same operation as `registry-configure`** — so no conformant grant distinguishes them and a portable vector cannot mint one without the other. **Derived from V7, not from the cohort:** a grant scopes on `path-scope` and `id-scope` only, and the path axis is **explicitly non-portable** (*"peers that diverge remain conformant"* on path locations), so **the operation name is the sole portable discriminator**. **§4.3's own paragraph diagnoses the defect it reproduces** — a bare tree-write *"cannot refuse selectively, cannot carry a qualifier, and cannot be distinguished from any other write to the same entity"* — and fixes the first two. **§5's maintained rule was satisfied in letter, not substance:** `registry-pin`'s column names *a condition on another row's operation*, which a grant cannot carry; D4 fixes the rule, not just the row. **Ruled: the non-dispatchable operation `pin-bindings`, and cap checks precede config validation** (unruled and cross-impl-observable, and validating first leaks a config-shaped violation list to a caller with no authority to change it). All three core seats had independently shipped `pin-bindings` including the non-obvious non-dispatchable half — **corroboration, cited last and labelled, per L18.** **L17's second shape; L17 ratified on it.**)* *(**2026-08-20 — `PEER-TRANSPORT-SET` opened, DRAFT, v2, first pass.** The address-discovery hole, and it turned out to be two things. **The lookup:** a holder of a bare `peer_id` has no way to learn where that peer is — NETWORK §6.5.4 assigns it to REGISTRY, REGISTRY §12 assigns it to an EXTENSION-IDENTITY amendment that does not exist, and §2.3 and §12 cite **each other** for the scoping with no derivation anywhere. **Ruled NETWORK's, derived rather than cited** (operator correction — the exploration's first pass had adopted §12's pointer as its conclusion): `peer_id` is V7 §1.5 substrate, the answer is a NETWORK type, IDENTITY's mapping is *durable identity → {peer_id} across rotations* which shares no key/value/lifetime/signer with this, and DISCOVERY puts identity **after** the dial so it cannot gate it. **The deeper half:** `system/peer/transport/*` carries `peer_id` **inside** the entity with **nothing signing it** — authority is *positional*, and position is what transit destroys, so once a profile travels by hash the **registry's** signature is what vouches for where a peer is, while §6.5.1a's own model is that a peer self-publishes. Proposes `system/peer/transport-set`: signed by the id-holder, `seq`-monotonic, expiring, servable by anyone — `published-root`'s properties applied to reachability, which is libp2p's signed-peer-record fix, ATProto's DID doc and Nostr's NIP-65 in one shape. Seven deltas (four design, three withdrawing text that is wrong today), **five open questions marked as open** including one-record-vs-walk-the-root and the A/B/C lookup shape. `entity-browser-rust` is working the same question independently; authored without waiting per L18, with §9 naming the rows their result lands on.)* *(**2026-08-20 — `CONSUMER-TRUST-ANCHOR-ON-THE-STATIC-PATH` opened, DRAFT, v2.** Derived from the corpus while verifying `entity-browser-rust` `7bc1ccf`, and **against our own text rather than their wiring** — they asked that their finding *not* be read as a spec defect report and that is the one thing in it we overruled. Four landed texts against each other: `EXTENSION-TREE` §3.3a (*"never trusting paths the host claims outside that chain"*), `EXTENSION-SUBSTITUTE` §7.2 (a path→hash index is an authority claim; *"anyone serving the URL can forge it"*; signature **MUST**), `GUIDE-SERVING-MODE` §8's three display states — and `EXTENSION-NETWORK` §6.5.3.1's `TREE_GET` leaf, **the route that actually serves `path → hash`**, which carries none of it under the heading sentence *"the consumer trusts the math, not the host."* On `CONTENT_GET` the consumer supplies `H`; on `TREE_GET` **the host does**, so the hash check is self-consistency, not authorship — §7.2's own words for the shape it forbids. `signed_pointer` is publisher-optional and its absence is indistinguishable from a non-signing publisher, so **the assurance level is chosen by the party being trusted**; and `EXTENSION-REGISTRY` §5.1's *"IDENTIFY is the gate"* has **no counterpart on `http-poll`**, which has no IDENTIFY — the resolved `target_peer_id` is present and never used. Five deltas, none of which makes `signed_pointer` mandatory: **the rule is about what a consumer may claim, not what it may fetch.** Does not reopen registry v1 by §0a — the *binding* is signed; the unsigned thing is the content one layer down. **Creates an app-tier sequencing constraint**: D2/D3 are a prerequisite of whichever of resolved-name-open / typed URL / foreign link ships first.)* *(**2026-08-19 — `REGISTRY-DISPATCH-CONFIG-ENFORCEMENT-POINT` opened + folded same day**, from `entity-core-go`'s consolidated cohort reply and settled on `entity-browser-rust`'s built implementation. §4.1 step 2 and §11.1 answered one question incompatibly — the MUST binds *a distribution shipping* a config, the vector bound *a resolver loading* one, and **a loading resolver cannot observe which act produced the bytes** (§6a.9.2 puts both in one entity at one path), so it over-enforced and deleted the operator `MAY` granted in the same paragraph. Ruled: enforcement moves to **packaging lint + config-write refusal**; at load, surface but never normalize and never refuse to start; and **a resolver never rewrites stored config as a side effect of reading it** — which closes SA-PY-22 by dissolving it. Eligibility ruled **kind-scoped**: browser-rust's `validate_rules`/`eligible_backends` take no `resolver_chain` and are the only built implementation, and chain-scoped would let a broad row **arm itself** the day an operator adds the backend — v1.14's "evadable by omission" one level up. New `REG-DISPATCH-CONFIG-REFUSED-1`, a config-read-back vector, because the old vector's observable (the absence of a request) **could not discriminate the two readings at all**. Also: **R-4 closed** — §6a.6 binds the remote-registry path, §3.1 constrains no storage path, and `entity-core-rust`'s revert stands; **row (d)'s "or pinned" withdrawn** as unconstructible, §4.1 step 1 returning a pin before the chain is reached — the same category error §4.1a already names, made twice in two sessions; and `REG-TTL-CEILING-REREAD-1` added for the read-at-resolution MUST, which had **no instrument anywhere**. REGISTRY 1.16→1.17.)* *(**2026-08-19 — two opened + folded same day, both from `entity-core-go`'s consolidated registry packet, and one of them was arch's own defect from the day before.** `REGISTRY-NAME-LEGALITY-INPUT-DOMAINS`: v1.15's `REG-NAME-CONSTRAINTS-GRAMMAR-1` row 3 required an admission gate to accept `x/y/z`, which §6a name-path safety refuses `400` three subsections earlier — **unsatisfiable by every conformant implementation**, verified in all three engines. One matcher, **two input domains**: dispatch sees the raw `meta_resolve` argument, `name_constraints` sees a post-normalization name. L12's shape on arch's own text — the matcher was verified, the path by which input reaches it was not. · `REGISTRY-RESOLVER-CEILING-CONFIG-SITE`: the ceiling §6a.9.1 calls *"the load-bearing one"* had **no config site**, so four seats built three — py and rust converged on `resolver_chain[].hints.max_ttl`, go put it in a Go builder option, and **`entity-browser-rust` carried a fourth key nobody had counted** (L16's second payoff in two days). Ratified py+rust's site, pinned `0`-is-undeclared and the null-`ttl` arm that all four had already agreed on off-spec, and added `REG-TTL-RESOLVER-CEILING-1` — the resolver-side vector this spec never had for its own load-bearing half. REGISTRY 1.15→1.16.)* *(**2026-08-18 — `PUBLISHED-ROOT-FRONT-DOOR` opened + folded same day** from `entity-workbench-go`'s Go-published/Rust-consumed cross-check, the first result on this surface that is **not** same-language: three of four surfaces align with no shared code (content sharding, bare-hashable bodies, two-hop signature keying) and it stops at **hop 0**. Two defects. **(a)** The published-root head-pointer tree path was **never pinned anywhere in `specs/` or `guides/`** and three impls picked — rust and py at `{peer}/system/peer/published-root`, go appending `/{base58_peer_id}`, with a source comment claiming it *"supersedes"* NETWORK §6.5.3, a supersession this corpus never made. Ruled **no peer-id segment**, on the foreground don't-double-qualify invariant. **(b)** `signed_pointer` is a **tree path** and was read as a fetch location; ruled **discovered from `manifest_url_prefix`, never conventional**, and a consumer MUST NOT join `signed_pointer` onto an origin. "Serve it at both paths" rejected — two front doors is the divergence ratified. TREE 4.2→4.3, NETWORK 1.7→1.8.)* *(**2026-08-18 — `HISTORY-CONFIG-SELECTION-TOTAL-ORDER` opened + folded same day**, found by arch while reviewing go's REVISION R15 fold and routed by nobody: `grep best_specificity specs/` returns exactly two sites and R15 fixed one. §2.2 states a **two-key** order (literal segments, then depth) and §6.2 compares it as a single value with `>` over unspecified enumeration — **a scalar cannot carry a two-key order.** py returns the faithful tuple; go and rust independently reached byte-identical 2-per-literal scalars and **tie** on `a/b/c/d` vs `a/*/c/*/e`. Folded: tuple compare + a lexicographic third key for totality, §2.2's keys unchanged. **Not hash-determining** — history is opt-in per-peer state, so this is a determinism defect, not a convergence one, and it is pinned because the fix costs one comparison. HISTORY 1.6→1.7.)* *(**2026-08-18 — `REGISTRY-RENEW-TTL-CASCADE` opened + folded same day** from `entity-core-go`'s spec-issue `-c`, raised by `entity-core-py` as SA-PY-13: `renew-request` is a **second producer of peer-issued bindings** and the D11/D12 null-`ttl` fold swept only `register`. Ruled **inherit, not refuse** — a three-step cascade ending at the superseded binding's own `ttl`, non-null by §6a.3, so it cannot resolve null and never refuses. The distinction that decides it: the predecessor's `ttl` is **recovered, not chosen**, so §6a.9.2's ban on an implementation-chosen default does not reach it. REGISTRY 1.8→1.9. **§5 files an unruled finding found while verifying it — `ttl` has no ceiling anywhere on this surface**, and §6a.3 makes `ttl` the only bound on a withheld revocation, so an unbounded requester-chosen value reproduces the unrevokable binding without ever being null. Operator decision.)* *(**2026-08-18 — `TREE-WALK-COMPLETENESS` opened** from `entity-browser-rust` `a0145a7`: `EXTENSION-REGISTRY` §6a.3a asserts a hostile origin *"cannot omit a node from the walk without the walk failing"* and **no implementation has that property** — rust `ed993bb`, go `88615f6`, py `79f38a9` all silently skip a declared-but-absent child. **The first draft got the design backwards and the correction is the substance:** it assumed a publisher serves a filtered view of one big tree, so incompleteness could be legitimate and the fix was a mode. §3.4 and §6.2 say otherwise — a tracked prefix is *its own trie* and `execute_extract` **rebuilds** (`root_hash = build_trie(bindings)`), so publishing a subset is **re-rooting, not filtering**: nothing about the unpublished tree leaks, and any unresolvable child is withholding, unambiguously. §6.2's *"unfiltered subtree nodes MAY be omitted"* is **v3.x residue** contradicting the algorithm four lines above it, and is the only text making a published tree legitimately incomplete. One invariant replaces the modes — **the absence of a node is never an answer** — which also covers §6a.6's revocation `404`. Fifth instance of the absent-vs-withheld seam.)* *(**2026-08-18 — final state: all five are folded and are in `implemented/`, plus `SHARE-AS-GRANT` from the applications tier.** The intermediate reversal recorded here has itself been reversed: only the **version cuts** were unauthorized, not the folds. Ruled-and-folded is complete and moves out — that is the ordinary lifecycle, not an operator decision, and treating `implemented/` as needing sign-off was L14 over-generalized a second time. What stands from the correction: **`ENTITY-CORE-PROTOCOL`'s version is never ours; extension versions are ordinary work**, and the reasoning previously recorded here — that a peer declining a DRAFT showed ruling-without-folding had split the cohort — **is withdrawn and was backwards.** `EXCLUDE-PARITY` reopened, ruled D6 (the four-form closed grammar, `**` rejected at config write) and folded. Original same-day entries follow: `DISPATCH-AUTHORIZATION-FRAME`, `REVISION-AUTO-VERSION-EXCLUDE-PARITY`, `DEFAULT-NAME-FORMAT-DISPATCH`, `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` and `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` were moved to `implemented/` and five spec version headers were cut, **on no authority** — cutting a version and marking a proposal complete are the operator's calls (L14). All five headers are reverted and all five proposals are back here; **the folded spec text stays landed**, unversioned, which is this repo's ordinary cohort-finding mode. The reasoning previously recorded here — that a peer declining to implement a DRAFT showed ruling-without-folding had split the cohort — **is withdrawn and was backwards**: we work from draft proposals as a matter of course, a seat that thinks a draft is wrong implements it and reports back, and a peer's schedule is not an authorization to declare a release. `REVISION-AUTO-VERSION-EXCLUDE-PARITY` is further **reopened as DRAFT** — its SA-PY-11 doublestar pin is withdrawn for contradicting `ENTITY-CORE-PROTOCOL` §5.4, see `docs/status/AUDIT-2026-08-18-the-glob-rule-…`. · Original same-day entries follow: `DEFAULT-NAME-FORMAT-DISPATCH` opened + **folded out** same day — browser-rust's last arch blocker (B16b); pins the ordered default globs and makes the catch-all local-only, because §4.1 step 2 calls itself the primary privacy mechanism and that property lives in the defaults, not the mechanism · `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` **folded out** — all twelve deltas; the association check closes a live resolution-integrity defect in three shipping impls, and §6a.1a names the byte-server the threat model never had; REGISTRY 1.6→1.7 · `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` **folded out** — all seven deltas landed, D4 adjudicated first per its own *do not fold blind* (the phantom resolves to `EXTENSION-SUBSTITUTE` §7.2); NETWORK 1.7→1.8, TREE 4.0.2→4.1 · `DISPATCH-AUTHORIZATION-FRAME` opened + RULED same day — core-py's SA-PY-9/SA-PY-10: §5.2's resource check binds **any dispatch carrying a resource**, and a stored `dispatch_capability` is **granter**-framed; `EXTENSION-CONTINUATION` §3.5's worked example spells the losing form and taught it to a cohort fixture · `REVISION-AUTO-VERSION-EXCLUDE-PARITY` opened + RULED same day — core-py's SA-PY-8, the only one of their four that gated code: `EXTENSION-REVISION` §6.1 **contradicts itself**, its Amendment-2 rule requiring auto-version to *build* a trie while its Algorithm block *adopts* an unfiltered one · `REGISTRY-NAME-ASSOCIATION-AND-THE-HOSTILE-HOST` opened — browser-rust's two static-host findings, confirmed 3-of-3; the name check is a live resolution-integrity defect and the `ttl` bound that backstops revocation was never enforced · 2026-08-17: `STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` opened — `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` was never written and three normative sentences hand it obligations; both app tiers built against it · `DISCOVERY-RENDEZVOUS-BACKEND` opened + RULED same day — browser-rust's Q8, their #1 blocker: rendezvous is a DISCOVERY backend on the SIGNALING carrier, token `rendezvous` · `RELAY-COMPLETE-THE-MODE-SET` opened — retract two deferrals, complete the mode set, name the modes in English · 2026-08-14: `REGISTRY-SERVICE-ADVERTISEMENT` folded out, `SIGNALING-ICE-PROVISIONING-LIFETIME` opened in · 2026-08-15: `REVISION-MERGE-DELEGATION-AND-CASCADE` opened, then four folded out — `NETWORK-LIVENESS-REACTIVE-BUILDOUT`, `CONTINUATION-LOST-ERROR-MARKER-MUST`, `AVAILABILITY-DESCRIPTOR`, `NAMESPACE-CLEANUP-AND-BROWSER-LEG` · 2026-08-16: `COMPUTE-IS-ERROR-PREDICATE` opened, ruled and folded out the same day — the one arch-side item the cohort was blocked on · `SUBSTRATE-DISCHARGED-OBLIGATIONS` opened + folded from browser-rust's rig routing — **and left in `active/` until 2026-08-17**, see below)* · *validated by go / rust / py*

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

**`active/applications/` — 9** *(**2026-08-31: `WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX` opened — DRAFT, folded at authoring, eight deltas across four guides.** Filed by **`entity-browser-rust`** with a built implementation and a two-arm falsified gate, after a user-visible symptom: `GUIDE-ENTITY-WORKBENCH-APP` §8 offers a **persist** arm for per-window state and **nothing anywhere constrains `{window_id}`'s lifetime**, so both impls made it a per-session counter and the state at `windows/3/state` belongs to whatever held ordinal 3 last boot. **Ruled: `{window_id}` is a session-scoped slot address**, derived from §8's conditional arm and §3.2's path-position-invariance rather than from what the impls do (**L18**). **The roster slot already existed and no one knew**: `app/state/layout` sits in `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §5.1 and `GUIDE-SDK-PATTERNS` §2, both canonical and published, **absent from the §4.2 table §4.1.1 names as the authority**, and §4.2's consensus had already assigned arrangement to per-impl decoration — one rule, three homes, one updated (**L23**). Retired, not repurposed; `app/state/window-index` added on the **membership-is-portable, arrangement-is-not** line. **The filed "one small thing" is the largest item**: on the transitional `app/state/window` fallback every window's entity carries the *same* type name, so the filing seat's own discriminator mitigation is structurally unavailable — the easy on-ramp is more dangerous than the shape it leads to, and `content_type` becomes required (**`entity-workbench-go` already writes it**, under a kebab key that violates `STYLE-NAMING`). **Two defects found in our own text on the way**: §1 tiers the action wire shape **MUST** while §6 calls it speculative and **NOT REQUIRED** — the contradiction the filing seat's §1 argument was standing on, corrected to §6 (**L19** — §1 is an index, §6 owns the surface); and `GUIDE-SHELL-FRAMING` §7.1's `Selection` example emits `origin: {surface, window_id}`, which §5.4 rule 1 makes a **MUST NOT** — invisible to a token grep and to workbench-go's live `legacySelectionFields` gate, so an impl reading it would be **non-conformant and silently green** (**L23** second shape, **L25**). **Two of the filing seat's claims corrected without moving the conclusion**: their "cosigned MUST" forbidding a path re-key **does not say that** (§1 pins the *prefix*; §3.1 blesses two sub-shapes) — withdrawal upheld on design grounds instead; and workbench-go's *"write-only, future restore last session"* quote is on the **shell-alias** slot, keyed by a durable alias name, while the real workbench-go evidence is **stronger** — a live read at `log_model.go:143` keyed on a session ordinal, into a read-modify-write merge (**L22**, and arch moved the sentence as much as they did))* · *(**2026-08-30: `FOLLOW-THE-PATTERN-THE-SET-AND-THE-TWO-ROUTES` opened — DRAFT, first pass, and deliberately a stress test rather than a sign-off candidate.** Answers the dependency question a design audit was opened for: **following requires `EXTENSION-TREE` + `EXTENSION-NETWORK` + core, and NOT `EXTENSION-REVISION`** — `system/peer/published-root` (TREE §3.3a) already carries `root_hash`, **`seq`** (the change check *and* the rollback defense) and `predecessor` (a history chain) in one signed entity, so the cursor, the change check and the walk entry point are all there. REVISION is required exactly where it should be — `revision:fetch-diff` and DAG convergence — and neither is a requirement of the *pattern*, only of one route through it. Establishes **two routes**: **A** (live publisher, dispatch, work on the publisher, **needs a capability grant**) and **B** (static origin, walk the root, work on the reader, **publisher may be offline**), where B gets its delta free from content addressing because dedup skips held subtrees. **T5 is the result that constrains the design**: a stranger holds no `revision:fetch-diff` grant, so **public following is Route B only** — a stronger argument for B as the base case than liveness was. **Three stress tests are UNRESOLVED and say so**: **T2** — a publisher who has not republished and an origin withholding a newer root are *byte-identical* at the consumer, so "no new posts" is not a fact a follower may assert; **T3** — a followed publisher that re-keys is a *different peer* and every follower silently points at an abandoned identity, with a successor pointer signed by the old key being exactly what a compromised key must not be able to assert; **T6** — a publisher restored from backup at `seq: 0` is **permanently unfollowable**, because the rollback defense is working as designed. Six deltas, one of them (**D2**, the cursor site) the only substrate-shaped claim, left open on Q1/Q3's promotion test — *is there a second non-social consumer of the identical loop?* **Second pass same day: T3's prior art pulled in as §6a rather than re-derived** — the property we lack is `did:plc`'s *identifier-is-a-hash-of-genesis*, core has already ruled cross-form correlation belongs to identity/registry/policy and **never core**, so T3's answer is a **registry binding kind** and not a follow-list field; a genesis-hash name form **conflicts with the landed `self-certifying` pin** (`name == target_peer_id`, explicitly not a hash) but lands cheaply **beside** it under REGISTRY's ignore-unknown-kind forward-compat rule. Carries the uncomfortable one: **a genesis that is a bare keypair with no attestation has no rotation path under any design**, so for peers publishing that way T3 is structurally unavailable rather than merely open — and *publishing freezes what consumers pin*, which is the deadline. **Two new stress tests. T10 — a follower cannot scope its attention below the peer**: one root and one `seq` per peer (forking the stream is what destroys rollback detection), and the trie is HAMT-routed by `SHA-256(relative_key)` so *"structure is determined by hash bits, not by path-segment locality"* — there is no subtree to check, no per-prefix hash without already holding the bindings, and the path-keyed LocationIndex is **local and unpublished**. So *"did anything I care about change?"* costs the same as *"what changed at all?"*; three directions named, none picked, and it is flagged as **the question most likely to change the design**. **T11 — asymmetric extension adoption**: rotation continuity that lives in an extension the *consumer* may not have leaves that consumer seeing T3, the same shape the parked key-death work already met as a support-tier flag with no feature negotiation)* · *(2026-08-21: `ACQUISITION-SURFACE-PROTOTYPE-TIER` opened — the **living ledger** for browser-rust's acquisition work, which is arriving faster than proposal→ratify→fold turns)* · *validated by **workbench-go / browser-rust**, **not core-go***

**`EXTENSION-HOST-INSTALL-SEAM`** *(2026-09-01 — **T1's deliverable #0**, ahead of any extension. The corpus has three words for the party that installs a handler — §6.2 *"user-installed"*, §9.1 *"user"*, §11.6.7 *"application-owned"* — and all three mean **application code**, so the **extension installer** is named nowhere and every one of the 26 standard extensions, all under `system/*`, is refused `403 forbidden_pattern` on the wire path. D1–D9 across both repos: §6.2's reservation is scoped to the **dispatch path** (a `0.8.2.4` fourth component that changes **no peer's behaviour**), §11.6 names its two callers and forbids an SDK applying §6.2 inside the primitive, the decline becomes **declared** rather than inferred (**L17** before the fact, for the seven peers with no first-class callable), and `GUIDE-CONFORMANCE` gains a **§7d host-seam class** — driven in-process by the peer's own harness, asserted over the wire, with the reference body required to return something **no `compute/literal` can produce** (**L8's eighteenth form**, designed against rather than learned from) and **both** controls mandatory. Sizing: one contract, one gate, one keystone sweep — ~250 lines per peer in the worst tree read (`go`), zero in `runChain`. **Consumers: `entity-core-keystone` (build) + `entity-core-go` (oracle); routed in the consolidated packet, not today** — L13's fifth axis)* · `APP-CONVENTION-CHAT` *(consumers: browser-rust, workbench-go — **both asked 2026-08-17**)* · `SHARE-AS-GRANT-AND-THE-AUDIENCE-CARRIER` *(RULED; fold = author `APP-CONVENTION-SHARE`)* · `APP-CONVENTION-COMPUTE-PROGRAM` *(consumer:
workbench-go)* · `COMPUTE-LOWERING-CONTRACT` · `COMPUTE-LOWERING-TOOLKIT` *(§5 sweep built at
core-go's `compute-corpus/worked.go`; the builder/toolkit surface is what is owed)* ·
`SUB-PEER-ISOLATION-MODEL` · **`WINDOW-ID-LIFETIME-AND-THE-WINDOW-INDEX`** *(filed by browser-rust,
built there; consumers: **browser-rust + workbench-go**, both routed 2026-08-31)* ·
**`ACQUISITION-SURFACE-PROTOTYPE-TIER`** *(**append items to its §3
ledger rather than opening a proposal per item** — operator direction 2026-08-21. Carries the
**PROTOTYPE** verb tier for `meet`, and the `app/chat/room` ruling)*

**This is the tier that had never been routed to anyone who implements it.** Four of the five
have a running consumer in tree; none of those consumers was asked.

**`active/process/` — 4** *(2026-08-31: `DEVERSION-TEST-VECTOR-CORPUS` **moved to `implemented/`** — the ECF half landed with the `ENTITY-CBOR-ENCODING` Appendix E amendment (§9) its own §8.2 said was owed; `spec corpus` 1 → 0 · 2026-08-17: `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1` **re-tiered out to `core/`** — a core-protocol §6.2/§6.9a.1 defect that had been filed here, which routed it to arch-tools instead of to the three seats that implement the surface · 2026-08-17: `DOCUMENT-CLASS-HEADER-FIELD` opened — `specs/` holds normative specs, guides and architecture references with no way for a document to declare which it is, so the class lives only in a tool's filename-glob map and defaults to `canonical-spec` on silence; two files invented `Authoritative scope:` and two analyzers disagreed about the same corpus · 2026-08-17: `POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1` opened — a core-protocol contradiction filed by browser-rust (F6) and verified here: §6.2 closes the policy-path key to two forms, §6.9a.1 defines a three-form resolution order at the same path · 2026-08-15: `CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` opened — reference
proposal for the §2.4a/§5.2a/§5.2b/§5.2c rules, folded 08-09…12 with no proposal; see
`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`)* · *validated by core-go (oracle) / arch-tools (gates)*

`CONFORMANCE-COVERAGE-FAILURE-TAXONOMY` *(reference)* · `HASH-WIDTH-IS-NEVER-FIXED` *(reference)* · `CONFORMANCE-ORACLE-CONTRACT` *(the register is built and running — folding it describes what is
already there)* · `CORPUS-REFERENCE-INTEGRITY` · `DEVERSION-TEST-VECTOR-CORPUS` *(the `v767-`
prefix rename **is** the fold)* · `MATURITY-MODEL-AND-ROADMAP-COMMUNICATION`

**Nothing external blocks any of the four.** They touch no wire format and no impl behavior.

**The oldest are three weeks old and nothing has moved them.** That is not automatically wrong —
several are deliberately parked — but it is not visible anywhere else, which is why it is stated
here.

## 2. Implemented — 55 (moved this cycle)

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

## 3. Superseded — 2

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
