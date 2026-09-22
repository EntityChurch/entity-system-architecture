# EXPLORATION — the verification ladder: what a publisher signs, what a reader can check, and where it lives

**Status:** Exploration (design record). Not a proposal, not normative. **Proposal-ready — §7 names the
artifact class and the delta.**

**Operator-driven, 2026-09-06.** Three demands, and this document exists to answer them together
rather than as three findings:

> **What went wrong in the client renderer?** A thread could not be followed, a peer could not be
> located, and the signatures could not be verified. Account for it.
>
> *"Maybe I'm pulling from peer A, and it pulled in the reply from peer B and peer C. **What do I need
> so I don't need to go to peer B and peer C?** … Maybe I got some stale data. But I shouldn't be
> questioning the integrity of it."*
>
> *"If I'm publishing, **I need to know what I need to sign.** I need to know what I need to prove.
> Because if someone else is going to consume it later and they're missing a signature on something —
> then what? The content hash pans out, but that's all we know. **That's not adequate.**"*

---

## §0 The result, in one table

**There is a resolution ladder and there is no verification ladder.** `GUIDE-RESOLUTION` §3 answers
*"what do I have → where do I enter"* for getting **to** an entity. **Nothing states what you can
establish once you have one**, and that absence is the layer the operator keeps reaching for.

**The ladder has five rungs. Every instrument for rungs 0–3 already exists and is landed; rung 4 is
impossible and should be stated as impossible.**

| Rung | The reader's question | Instrument | Cost to check | **Can a MIRROR answer it?** |
|---|---|---|---|---|
| **0** | Are these the bytes I asked for? | **content hash** | free | **yes** — anyone |
| **1** | **Who wrote them?** | **detached `system/signature`** at `/{author}/system/signature/{hex(H)}` | **1 entity** | **yes** — this is what makes mirroring work |
| **2** | Did the author actually **publish** it? | **inclusion proof** vs their signed root — *or* a **binding assertion** (§4) | log₃₂(n) nodes + root — *or* **1 entity** | **yes**, if it carries them |
| **3** | Is it **current**? | signed root **`seq`** + rollback-reject | a root fetch | **NO — only the authority** |
| **4** | Is this **everything**? | **none. Unattainable without a gatekeeper** | — | **no — and neither can anyone** |

> **That table is the answer to the mirroring question.** *"What do I need so I don't have to go to B
> and C?"* — **rungs 0, 1 and 2, all of which peer A can hand you.** Rung 3 you cannot get from a
> mirror by definition, because currency is a fact about the author's tree right now. Rung 4 nobody
> can give you. **So the operator's acceptance criterion — *"maybe I got stale data, but I shouldn't be
> questioning the integrity"* — is exactly rungs 0–2 without rung 3, and that is a reachable state.**

**And the corpus is closer than any of this session's earlier documents implied.** `FEED` §4 already
specifies the mirror with these properties, including the sentence that answers the question directly:
***"each republished entry travels with its author's detached `system/signature` entity — without it a
mirror carries integrity but not authorship, and a reader can confirm the bytes match the hash while
having no way to learn who wrote them."***

---

## §1 Which part is not an entity — the precise answer

`[operator: "What do you mean a tree binding? Which part isn't an entity? We do have the links, those
are entities, look at the bootstraps. So what are we missing? The CAS part that bundles the path and
the hash into a unit that then someone can sign? Is that what we're missing?"]`

**Yes — that is exactly and only what is missing, and both halves already exist as types.**

| Piece | Exists? | Where |
|---|---|---|
| the **path** | **yes — a bootstrap type** | `system/tree/path`, one of the fourteen bootstrap types (`ENTITY-NATIVE-TYPE-SYSTEM` §4.6) |
| the **hash** | **yes — a bootstrap type** | `system/hash` |
| **an entity pairing them** | **no** | — |

**So nothing is missing from the type system.** `system/tree/path` is a first-class type and has been
since bootstrap. What does not exist is an **entity type whose `data` is `{path, content_hash}`** — and
because a trie edge is not an entity, that pair has no content hash and therefore no signature slot.
**That is a composition nobody has minted, not a capability the system lacks.**

**The registry is the same composition, one coordinate over** — which is the operator's long-standing
instinct, and it is correct:

> *"I've always kind of felt like the registry seems like a more general pattern."*

`system/registry/binding` is `{name, target_peer_id, transports, issued_at, ttl, supersedes, …}` — **a
signed assertion binding a mutable coordinate to a target**, stored content-addressed at
`system/registry/binding/{binding_hash}` and signed at the invariant pointer. **Change the coordinate
from a name to a path and the target from a peer-id to a content hash and you have the operator's
shape.** §4 takes that generalization seriously.

---

## §2 What the publisher must sign — the composer contract

`[operator: "If I'm publishing I need to know what I need to sign."]` **This is the ladder read
backwards, and it is the half that has never been written down.** A reader can only climb a rung the
publisher paid for.

| To let a reader reach… | The publisher MUST… | Cost | Landed? |
|---|---|---|---|
| **rung 0** | nothing — content-addressing is automatic | zero | ✓ |
| **rung 1** | **sign each entry** — mint a `system/signature` at `/{me}/system/signature/{hex(entry_hash)}` | **one signature per entry** | **RULED, unlanded** — `FEED` §1.1 |
| **rung 2** | publish a **signed root** covering the path, and serve its closure — *or* mint a **binding assertion** per portable binding | one signature per republish — *or* one per binding | ✓ `TREE` §3.3a + `NETWORK` Amendment 10 |
| **rung 3** | **republish on change**, within the 30 s convergence ceiling, `seq` monotone | bounded, coalescing permitted | ✓ `NETWORK` §6.5.6 |
| **rung 4** | — | — | impossible |

**The single most important line in this document:** ***nothing signs individual entities today.***
Measured independently in both app-tier trees this session — `entity-browser-rust` `d9cc645` and
`entity-workbench-go` `ff268ac` sign **published roots, registry bindings, attestations and
capabilities**, and **no content entity**. `FEED` §1.1 records the same fact from the implementation
side (`entity-browser-rust` `62c6d62`): *"a publish signs exactly one thing, the root."*

> **So the operator's *"that's not adequate"* is measurably the current state.** A reader receiving a
> post today gets rung 0 and **cannot reach rung 1 at all** — not because verification is hard, but
> because **the signature was never minted.** Rung 1 is one signature per entry and it is the whole
> difference between *"the content hash pans out and that's all we know"* and *"I know who wrote
> this."*

---

## §3 The thread walk — the L5 algorithm, stated once

`[operator: "What's the application-level algorithm that tells me this is the post this person made,
this is the reply that connects to that, that's authorized, and I can trust it?"]`

**It composes rungs; it introduces nothing.** Given a thread root reference and a mirror from peer A:

```
verify_thread(root_ref, source):
  1. FETCH the mirror record; verify gatherer's signature over it   → who assembled this view
  2. for each entry reference in mirror.entries:
       a. fetch bytes; hash them; require hash == reference         → RUNG 0  (integrity)
       b. locate /{entry.author}/system/signature/{hex(entry_hash)} → RUNG 1  (authorship)
          in the envelope's `included`, or by constructed path
          require sig.target == entry_hash AND sig.signer == entry.author
          FAIL-CLOSED if absent: render as UNATTRIBUTED, never as authored
       c. require entry.author == the namespace it was published under  (FEED §1.1)
       d. OPTIONAL: verify inclusion proof or binding assertion    → RUNG 2  (publication)
  3. STRUCTURE: each reply's parent field is a PIN (content hash), so the
     edge is self-verifying — a reply cannot be re-parented without changing its hash
  4. ORDERING: do NOT trust created_at (FEED §2.3.1 — author-controlled clock).
     Use the version DAG / prev commitments where present
  5. CURRENCY: unreachable from a mirror. Surface the view's age; do not claim freshness  → RUNG 3
  6. COMPLETENESS: unattainable. Render as "what was gathered", never "the thread"        → RUNG 4
```

**Three properties worth stating because they are what make it cheap:**

- **Step 2b is the whole mirroring answer.** Because the signature is *detached* and *content-addressed*,
  peer A carries B's and C's signatures alongside their bytes and **the reader contacts neither.** This
  is `ENTITY-CORE-PROTOCOL` §3.5 property (3) — *"Carol can verify Bob's authority without contacting
  Bob"* — arriving at L5 exactly as written two layers down.
- **Step 3 is free.** Thread structure needs no signatures at all: a reply names its parent by hash, so
  the edge is self-verifying and cannot be forged or re-pointed. **The graph is secured by
  content-addressing, not by cryptography.**
- **Fail-closed at 2b is the rule that makes the ladder mean anything.** *"Missing signature ⇒ render
  as unattributed"* is what turns the operator's *"maybe it's real, maybe it's right"* into a defined
  state. **The unverified case must be visible, not silently equal to the verified one** — the same
  discipline `GUIDE-SERVING-MODE` §8 already applies to freshness.

---

## §4 The binding assertion — generalizing the registry

**The operator's shape, taken seriously as the general pattern it appears to be.**

```
; sketch, not proposed
type: "system/binding-assertion"      ; name TBD
data: {
  coordinate:    <system/tree/path>,  ; the mutable address being asserted
  target:        <system/hash>,       ; what is bound there
  asserted_at:   <ms-since-epoch>,
  seq:           <uint>,              ; monotone PER COORDINATE — see the cost below
  supersedes:    <system/hash | null>
}
; signed at /{authority}/system/signature/{hex(assertion_hash)}
; the authority is DERIVABLE: it is the first segment of `coordinate`
```

**What it buys:** a rung-2 instrument that is **two objects and self-describing** — a verifier holding
only the assertion reads the authority off the path (§1.4), constructs the invariant pointer, and
checks. No root, no trie walk, no prior knowledge of the publisher. Against an inclusion proof
(log₃₂(n) nodes plus the root plus a trie implementation) that is a real reduction.

**What it costs, honestly:**

- **`seq` per coordinate is the expensive field.** Monotonicity is what separates *"the author asserted
  this"* from *"this is replayable"* — and a counter **per path** is per-path state the tree does not
  keep today, where the root keeps exactly one counter for everything. **This is the trade: N counters
  and O(1) proofs, or 1 counter and O(log n) proofs.**
- **Without `seq` it is strictly a rung-2 instrument and never rung 3.** That is fine — rungs are
  separate on purpose — but it must not be sold as freshness.

**Where it is clearly right:** where **the path IS the claim** — a site's page map (*"the canonical
`/about` is `H`"*), a registry name, a pinned release. **Where it is redundant:** immutable authored
content, where the entity to sign already exists and rung 1 covers authorship completely.

> **Withdrawn from the previous session: *"the social tier does not need it."*** That was stated with
> more confidence than the evidence carried, and the operator's objection is correct — **we have not
> built the client, so we do not know what the reader will demand.** What survives is narrower and
> defensible: **rung 1 is required for the social tier and is unbuilt; rung 2 is optional there and
> mandatory wherever a path is the claim.** Whether readers demand rung 2 on feeds is an empirical
> question **the client answers, not this document.**

---

## §5 Landscape — how the field ranks these rungs

**Every surveyed system pays for rungs 0–1 and differs almost entirely on rung 2.**

| System | Rung 1 (authorship) | Rung 2 (publication) | Rung 3 (currency) | Rung 4 |
|---|---|---|---|---|
| **Nostr** | inseparable per-event `sig` | — none; a relay's holding is not a claim | `created_at`, **author-controlled** | no |
| **ATProto** | records under a **signed commit** | **the commit IS rung 2**, with `rev` | `rev` monotone | no |
| **SSB** | signed log entry | **the log chain** — position is proof | sequence number | **within one feed, yes** |
| **IPNS** | — | — | signed **`seq` + validity window** | no |
| **Nix** | signed `narinfo` | store path is immutable | — | no |
| **Ours** | detached `system/signature` — **separable, multi-party, forwardable** | **two options** (inclusion proof · binding assertion) | root `seq` | no |

**Two observations that matter for the design:**

1. **SSB is the only one that gets partial rung 4, and it pays for it with a total order per feed** — an
   append-only log where a gap is detectable. **We deliberately do not have that** (`FEED` §1.3, *the
   set only grows*; §3a's collection is bounded and complete but per-author). **Rung 4 across authors is
   not a thing anyone has**, and `FEED` §4.2 says so plainly: *"completeness is unattainable without a
   gatekeeper and this document does not pretend otherwise."*
2. **Ours is the only one where rung 1 is DETACHABLE**, and that is the property the whole mirror model
   rests on. Nostr cannot serve an event without its signature — a strength — but also cannot let a
   third party attest, countersign, or witness one. **Detachment is what buys multi-party signing,
   dedup across mirrors, and `S-11`'s witness form; the cost is that a route can forget to carry it,
   which is D-42 and D-43.**

---

## §6 So what actually failed in the renderer — the concrete list

**Mapping the operator's *"what went wrong"* to rungs, because that is the test of whether this ladder
is real:**

| Symptom | Rung | Status today |
|---|---|---|
| *"Couldn't verify the signatures"* | **1** | **the signature does not exist** — nothing signs content entities in any tree |
| *"Couldn't follow a thread"* | 3 (structure) | **works** — parent pins are self-verifying, no signature needed |
| *"Couldn't locate a peer"* | resolution, not verification | `GUIDE-RESOLUTION`; D-27/D-37 |
| *"Couldn't find the authorization"* | 1/2 | **carriage** — D-43: no bundler collects signatures, so even a minted one may not travel |
| *"Maybe it's real, maybe it's right"* | **fail-closed rule** | **unstated** — §3 step 2b |

**Three of the five are one missing signature and one missing carriage rule.** That is the reduced
design the operator asked for, and it is smaller than the five documents preceding it suggested.

---

## §7 What this is — the artifact class, decided

`[operator: "Is this an extension? Is it SDK, guide, usability pattern? Are there shared normative
algorithms? Seems like we're getting to some kind of shared normative algorithm at the L5 app layer."]`

**Their read is right, and the negative half is the load-bearing one.**

- **NOT a new extension.** Every instrument for rungs 0–3 is landed: `system/signature` + the invariant
  pointer (§3.5), envelope carriage (§3.5), inclusion proof (`TREE` §3.7), root + `seq` (`TREE` §3.3a),
  pins. **Minting an extension here would be inventing a mechanism we already have.**
- **NOT core.** The substrate is correct as-is; the operator's *"the core protocol is a substrate, it
  says this is what it is"* is the right boundary.
- **One optional new TYPE** — the §4 binding assertion — and only if rung 2 needs an O(1) form. It is
  **REGISTRY's binding generalized**, so its natural home is beside it, not in a new extension.
- **The real deliverable is a SHARED NORMATIVE ALGORITHM at L5**, stated once and referenced by every
  convention, exactly as the operator guessed. Two halves, and both must be normative or neither works:
  **the reader's ladder** (§3) and **the composer contract** (§2). A convention then declares only
  *its floor* — *"a conformant feed reader reaches rung 1; rung 2 is optional"*.

**Precedent for the shape, so this is not a new artifact kind:** `EXTENSION-ATTESTATION` §4 carries
normative validation helpers and `GUIDE-ATTESTATION` teaches them; `GUIDE-CAPABILITIES` documents
`collect_authority_chain` while V7 §5.2 binds it. **Normative algorithm in the spec, pedagogy in the
guide** — the ladder should follow that split and not invent a third.

**Proposed delta (for a proposal, not done here):**

1. **Land `FEED` §1.1's rung-1 MUST.** It is the single highest-value unlanded thing in the corpus and
   nothing signs entries without it.
2. **State the ladder once**, normatively, with the fail-closed rule at rung 1 and the explicit
   *"rung 4 is unattainable"*.
3. **State the composer contract** (§2) as the publisher-side dual.
4. **Route it to all four app conventions** — three landed ones anchor on a root and have no rung-1
   story (D-44), and this is an L21 fifth-shape fold.
5. **Decide the binding assertion** (§4) on whether a page map wants O(1) rung 2 — **and let the client
   answer it**, not this document.

---

## §8 Sources

**Opened this session, by section.** `ENTITY-CORE-PROTOCOL` §3.5 (signature model, invariant pointer,
four properties), §1.4, §1.7 · `ENTITY-NATIVE-TYPE-SYSTEM` §4.6 (`system/tree/path`), §1 (the fourteen
bootstrap types) · `EXTENSION-TREE` §3.3a, §3.7, §6.2 (extract algorithm) · `EXTENSION-NETWORK` §6.5.3,
§6.5.3.1, §6.5.6 · `EXTENSION-REGISTRY` §3 (binding entity) · `EXTENSION-CONTINUATION` §4.3 ·
`EXTENSION-SUBSTITUTE` §4, §7.2 · `PROPOSAL-APP-CONVENTION-FEED` §1.1, §2.3.1, §4, §4.1, §4.2 ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §6 · `GUIDE-RESOLUTION` §3, §5 · `guides/` index (35 files —
**no verification guide exists**).

**Measured.** `entity-browser-rust` `d9cc645`, `entity-workbench-go` `ff268ac`, `entity-core-go`
`78db4a9`.

**External:** the §5 rows for ATProto, SSB, IPNS and Nix are **from prior reads and general knowledge,
not opened at source for this document** (L18 — corroboration, labelled). The Nostr row was opened in
the predecessor. **None should be cited normatively until opened.**

## §9 What is unread and what is unknown

- **`EXTENSION-CONTENT` §6.4.1/§6.4.2** — still unopened, fourth document to say so. It gates a
  third-party `CONTENT_GET` and therefore bears on whether a mirror can serve rung 0 at all.
- **`EXTENSION-REVISION`'s version DAG as the rung-3 instrument for edited content** — §3 step 4 waves
  at it. `REVISION` §5.4/§7.2 are the sections that own it and were not opened here.
- **Whether readers demand rung 2 on feeds is empirically unknown**, and the operator is right that it
  stays unknown until the client exists. **This document must not be read as settling it.**
