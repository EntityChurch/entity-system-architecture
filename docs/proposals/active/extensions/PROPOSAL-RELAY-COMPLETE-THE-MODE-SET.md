# PROPOSAL — RELAY: the mode set, the void deferrals, and say the modes in English

> ## ⛔ THE DISPOSITION IS RETRACTED — 2026-08-17, by the operator. Read this before anything below
>
> **This proposal twice carried a ruling that aggregate leaves RELAY (disposition 1). That ruling is
> RETRACTED — not downgraded, retracted — and no lean is recorded from this seat.** The three
> dispositions are open and the proposal is back to its original scope: *the void deferrals, the mode
> naming, and the real aggregate gaps.*
>
> **Why.** The disposition rested on: *"you cannot serve a queryable view over bytes you may not
> decode."* **§9 answers it directly** — *"there are two envelopes; only one is decoded… the **relay
> envelope IS decoded**"* — and the MUST NOT is scoped to the **inner** envelope, with `aggregate`
> deliberately in that list. **An aggregator that indexes outer routing fields and never opens the
> payload is exactly conformant**, and it is what AT Protocol's Relay does — the survey's named closest
> structural analog, which keeps that role boundary *even though its data is public*.
>
> **And the prior art was never read.** `HANDOFF-2026-08-17-relay-audit` §3a named eleven documents,
> *"start here"* on the study that **produced** the four-mode model. Three arguments were written before
> any of them were opened. That study's own standing pin is
> `[[feedback_design_space_not_authority]]` — ***relay modes are design-space; don't pick a single
> canonical*** — and its §0 states the frame the disposition would have broken: **"Relay is the
> universal intermediary primitive… every form — active forward, passive store-and-poll, aggregator,
> NAT-circuit, federation server — is one configuration of this primitive. One mechanism; many modes."**
> **The unification across modes is the extension's deliberate design contribution and the thing the
> study names as genuinely new versus every surveyed system.** Evicting a mode on a coherence argument
> would have dismantled it.
>
> **What the defect actually is** (`EXPLORATION-RELAY-LANDSCAPE-AND-PRIOR-ART` §3.1): **§1's opening
> sentence, not §9 or §10.4.** *"…carries opaque, signed, capability-bearing envelopes **between two
> endpoints**"* is contradicted eight lines later by **§2's own modes table**, which types Mode A as
> `multi-publisher` / `broadcast or filtered`. **§1's definition was written from the routed modes (F, C)
> and is narrower than the design space §2 tabulates.** That is a wording defect with a small fix — but
> it is **not ruled here**; six of the eleven documents are still unread (`EXPLORATION-…-PRIOR-ART` §7).
>
> **`ANALYSIS-RELAY-MODE-A-IS-NOT-A-RELAY-MODE.md` (browser-rust @ `ca3c760`) remains a well-argued
> filing document and should still be read** — its citations are accurate; the inference from them is
> what did not survive §9.

> ## ✅ THE PRIOR ART IS NOW READ — 2026-08-17 (later). **Disposition 2**, on the record, not on coherence
>
> The read the retraction demanded is done and written up:
> **`docs/research/reviews/REVIEW-2026-08-17-the-relay-prior-art-read-and-the-mode-set-disposition.md`**,
> which lists all seven legacy documents opened, by path. **`L11` is discharged for this proposal.**
> Four things changed as a result, and the first is the one to read:
>
> **(1) `PLAN-EXTENSION-LANDSCAPE` is not the network landscape.** Both handoffs named it *"the overall
> extension architecture"* and the second ordered it **first**, *"because which extension owns this is
> exactly the Mode A question."* Opened, it is the **identity / authorization / coordination stack**, and
> its §8 explicitly excludes *"subscription, content, continuation…"* — **RELAY, NETWORK, ROUTE,
> SIGNALING, DISCOVERY and REGISTRY appear nowhere in it.** A conclusion drawn from a document's *name*:
> **L8's tenth form**, propagated through two handoffs, none of which opened it. The documents that do
> draw the network family's boundaries are the three `cgid-10-220/229` explorations.
>
> **(2) The enumeration is sound, and eviction removes the contribution.** `EXPLORATION-RELAY-AND-
> AGGREGATOR-PATTERN` §5.1 extracts **five orthogonal dimensions** from a six-system survey and says
> *"different canonical modes are different points in this 5D space"* — §5.2's four modes are **clusters,
> not a partition**. Mode A's own examples are Nostr, ATProto Relay/firehose, Mastodon relay servers and
> NNTP federation, and **not one surveyed system is single-mode** (ATProto S+A · Mastodon F+S · Nostr A+S
> · SMTP F+S). §9.3 names the cross-mode unification as *"genuinely new … **no surveyed system has this
> unification**."* **Disposition 1 deletes the extension's stated contribution.**
>
> **(3) The blocker is void on stronger text than §2 cites.** `EXTENSION-SUBSCRIPTION` carries a
> top-level **`## 6. Cross-Peer Delivery`** — §6.1 does third-party delivery (A subscribes on B,
> delivers to **C**; more indirection than Mode A needs) and §6.3's convergent-mirror recipe states
> *"**Verified cross-impl (Go / Rust / Python, fresh peers per directional pair)**."* The mechanism is
> not merely specified, it is **cross-impl verified with a measured amplification bound**.
>
> **(4) The residual is smaller and sharper than §2.1 says — except on ordering, where it is a genuine
> rule conflict.** §7.7 of the study pins aggregation semantics as a **per-deployment knob** under
> `[[feedback_design_space_not_authority]]`, uniformly with routing algorithms (F) and NAT mechanisms
> (C). So §11.3's *"semantics per-deployment"* is **not** a deferral in disguise. **But** this corpus's
> *pin-the-cross-impl-observable-surface* rule bites on the one surface that is cross-peer observable —
> **the order a consumer sees a union of N sources in.** The design-space pin and the interop rule point
> opposite ways there. **That conflict is the open question, and it is an operator call, not an arch
> ruling.** Recommended shape: pin a **determinism floor** (a total order any two conformant aggregators
> agree on), leave merge/conflict policy free — the same shape as `APP-CONVENTION-SEMANTIC-CONTENT-SITE`'s
> lexicographic ordering floor.
>
> **Disposition, therefore: 2 — narrow §1's wording, keep Mode A.** §1's *"between two endpoints"* is
> contradicted eight lines later by §2's own table; **fix the sentence, the table is right.** The
> operator's boundary is not violated: an aggregator holds **N addressed subscriptions** and merges
> locally, so every hop is point-to-point and capability-gated. `EXPLORATION-INFORMATION-TRAVEL` §5 draws
> the same line — *"Mode A … is the **pull/subscription** federation answer; gossip is the
> **push/epidemic** answer. Likely siblings"* — and its §6 map places Mode A **inside RELAY's box**.
> **The boundary separates Mode A from gossip, not from relay.**

**Status:** DRAFT (2026-08-17)
**Target:** `specs/extensions/EXTENSION-RELAY.md` §1, §2, §3.3, §3.4, §4, §11.1, §11.1a, §11.3 ·
`specs/extensions/EXTENSION-REGISTRY.md` §8.2 (the inherited deferral) · `EXTENSION-SUBSCRIPTION` §8
(cross-reference only, no normative change)
**Tier:** `extensions/` — validated by go · rust · py. **Not keystone** (§6 — corrected 2026-08-17)
**Provenance:** the operator, 2026-08-17, disputing §11.1a out loud. Recorded in
`docs/status/TRIAGE-2026-08-17-the-backlog-against-the-friday-re-release.md` §1.
**Scope:** the *disposition* of RELAY's four modes and what they are called. **No change** to the two
load-bearing arch rulings (*static IS relay*; *relay is transport, not authority*), to Mode F/S
normative behaviour, or to V7. **No V7 amendment.**

---

## §0 Why this is a proposal and not a correction

The Mode A deferral is wrong on its stated grounds, which would ordinarily make it a cohort-style
fix-in-place. It is not, for two reasons:

1. **Retracting a deferral is a normative widening.** It moves two modes from *named, not
   normatively specified* to *specified*, adds operations to a handler surface three
   implementations ship, and changes what a conformant relay may advertise. That is L1 territory.
2. **The mode identifiers are values on the wire.** §3.x's advertise entity carries
   `modes: [<mode_id: "F"|"S">]`, exchanged between peers, so go/rust/py must agree on the strings.
   That makes it cross-impl coordination. **It is not a type-schema change** — see §6, where an
   earlier claim in this proposal is withdrawn.

---

## §1 Where the deferral came from, and why review did not catch it

**It was never authored in this repo.** `EXTENSION-RELAY.md` has three commits here: the
**v0.8.0 initial public release** import (`70c52b4`), a hash-width ruling (`38f3bfd`), and a
standards-gate cleanup (`1937e37`). The spec arrived whole.

**Its rationale document is not in this repo either.** `EXTENSION-REGISTRY` §8.2 and this spec both
cite `PROPOSAL-EXTENSION-RELAY.md §11.1a`. That file exists at exactly one path in this checkout —
`entity-lab-legacy-meta/entity-core-architecture/docs/architecture/v7.0-core-revision/proposals/implemented/PROPOSAL-EXTENSION-RELAY.md`
— **a legacy tree, not this corpus.** Every reader who tried to follow the citation to check the
reasoning hit a dead reference. This is `PROPOSAL-CORPUS-REFERENCE-INTEGRITY`'s thesis exactly (*a
reference nothing can follow is not a citation*) and the **C12** shape: a landed spec citing a
proposal absent from every post-split checkout.

**Authorship:** *"Authors: Architecture team"*, revision series `cgid-10-217` → `cgid-10-228`. No
individual seat, and no history in this repo to attribute it to.

### §1a The dead citation is not an accident — it is unfinished cleanup, and it was already tracked

**Corrected 2026-08-17 by the operator, twice over. The finding stands; both of my explanations of it
were wrong.**

**It was intentional.** The split deliberately did not carry `proposals/`, `reviews/` and
`explorations/` into the published corpus, because those are process artifacts and the published
specs are for implementers who have never heard of this cohort. **The plan was to strip the
references as part of making the specs publish-clean — and that pass was never run.** So these are
not lost records to reconstruct. They are **leaks to remove**, and reconstructing them would be
work in the wrong direction.

**It was already tracked, with the counts already taken.** The legacy repo carries
`STATUS-V8-PUBLISH-CLEANUP.md`, which defines the two axes explicitly — *conformance maturity*
(M0–M6) versus *publish-clean*, the latter meaning **"no dates, no cgids, no references to
unpublished `proposals`/`reviews`/`explorations` docs."** Its §3 records
**`entity-system-architecture`: 73 files, 58 errors, 982 warnings**, with `extensions/` at
*"208 dates + 134 leaks"*. Its §4 names the instrument and says to use it rather than hand-grep:

> **`spec address <repo> --gate`** — classifies citations **Forward / Stale / Leak**; *"leak" = a
> normative spec cites a process artifact.* **The unpublished-doc-leak detector.**

**So the gate existed, was purpose-built for exactly this, and had never been run.** Run today it
reports **1180 findings, 585 of them `dangling`** — the class whose own docstring reads *"a citation
whose target is not a file in the corpus. Judgment: Forward / Stale / Leak (§11.4)"*, configured at
observation severity precisely because the cleanup was pending.

> **My own second error, recorded because it is the more embarrassing one.** I measured this by hand
> at 14, then built a new `citations` gate for it (arch-tools `a870408`) and reported 134. **Both
> numbers were narrower than the tool that already existed**, and a second citation counter would
> have guaranteed the failure this repo has already written down — `W-CORPUS` Q7a records three
> plausible counts describing one corpus, where the paydown plan was correct for only one of them.
> **Reverted in full at arch-tools `50fec87`.** `address` flags the same bare-filename proposal
> citations mine did (`EXTENSION-TREE.md` lines 137 and 554 appear in both reports) *and* the
> `DOC.md §N.M` forms mine never read.
>
> **The lesson is one line: I searched the corpus for the defect and never searched the toolkit for
> the instrument.** `AGENTS.md` mentions the address gate in passing; the legacy plan names it
> outright. **Check what exists before extending it** — the same discipline as reading a peer's own
> document rather than a summary, pointed at our own tools.

**What this means for RELAY specifically.** Its four cited documents are legacy-only, including the
two the header calls *"load-bearing and unchanged"* — `reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md`
(*static IS relay*) and `reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md` (*relay is transport, not
authority*). **This proposal must not disturb those two rulings, and it cannot read them from this
repo**, so they are the one case here where the reasoning has to be carried across before the fold —
not reconstructed as a document, but restated in this proposal where the rule can live.

**And one is worse than a leak:** `explorations/EXPLORATION-TRANSPORT-COMPOSITION-AND-FALLBACK.md`
**exists nowhere** — not here, not in the legacy tree. §11.1a gates **Mode F's routing-on-failure
behaviour** on it. That is not an unstripped reference to a real document; it is a normative gate on
a document that was never written (§7 item 8).

**How it survived scrutiny — this is the reusable part.** The `cgid-10-228` revision was a **full
pre-implementation review against the landed state of every dependency** (INBOX v5.9, CONTINUATION
v1.20, NETWORK Amdt 10, REGISTRY v1.0, DISCOVERY v1.0). It re-verified twelve things and changed
eleven of them — dropped `correlation_ref`, reframed INBOX composition, remapped error codes,
retyped peer-ids to Base58, collapsed the inner envelope, pinned namespaces and timestamps. Its own
closing line:

> ***"No change** to the two arch rulings, the four-mode model, the Mode A/C deferrals, or the
> no-V7-amendment conclusion."*

**A review that re-derived everything around the deferrals passed over the deferrals themselves**,
because a scope cut reads as a decision already made rather than as a claim to be checked. That is
`AGENTS.md`'s recurring shape stated in its own words — *the stale claim is always plausible, and
plausibility is exactly why review does not catch it* — and it is the argument for **L4 inward**:
a claim in our own corpus is checked by opening the thing it cites.

---

## §2.0 **PROVISIONAL — downgraded from RULED, 2026-08-17, same day.** Mode A is mis-filed; *which* disposition is not settled

> ### ⚠ Read this before acting on anything below. The ruling is withdrawn as a ruling.
>
> §2.0 was published as **RULED — disposition 1 (aggregate leaves RELAY)**. **It is downgraded to
> PROVISIONAL and nothing may be folded on it.** Three things surfaced within hours of publishing it,
> two from peers and one from finally opening the right document:
>
> **1. `entity-core-go` showed the "origin, not transport" argument contradicts landed text.**
> RELAY Q4 and `REGISTRY` §8.2 both say *"the aggregator does NOT re-sign … Aggregator is
> transport."* So the ruling's own second row read as a mistake against the spec. **Their fix is
> better than the argument it repairs:** an aggregator is **transport for each payload** (items keep
> their original signatures — the spec is right) and **origin for the aggregate artifact** — the
> membership set, the ordering, the merge. *That artifact* has no upstream signature, and *that* is
> what is absent from F/S/C. **This names the real content of the "one genuinely new spec": how the
> aggregate artifact is attested.** `REGISTRY` §8.3's unsigned `MAY` conflict-annotation is that
> surface, already present and already unattested.
>
> **2. §9 permits more than the ruling assumed.** *"There are **two** envelopes; only one is
> decoded."* The **relay envelope IS decoded** — `destination`, `next_hop`, `namespace`, `ttl_hops`.
> Only the **inner** payload is opaque, and §9's stated rationale is **re-encoding hazard**
> (bit-fidelity + content-store dedup), not a prohibition on reading. **So an aggregator that indexes
> outer fields, holds inners opaque, and re-serves them verbatim is coherent** — which is close to
> what ATProto actually does. **Disposition 2 (narrow §1's "queryable view", keep the mode) is
> therefore live, and may be stronger than disposition 1.** The contradiction is real; it may be a
> **wording** defect in §1 rather than a category error in the enumeration.
>
> **3. The landscape study that produced the four-mode model was never read.** RELAY §1 cites
> *"the relay-and-aggregator pattern study (Nostr / AT Protocol / ActivityPub / libp2p circuit-relay
> / SMTP-NNTP → four canonical modes)"* — it is
> `explorations/EXPLORATION-RELAY-AND-AGGREGATOR-PATTERN-cgid-10-217.md`, **in the legacy tree**, and
> it maps Mode A onto Nostr relays, ATProto's *"crawls PDSes; aggregates a firehose of **signed
> records**"*, and Mastodon relay servers. **Every real system the enumeration was derived from calls
> the aggregator a relay.** That is not decisive, but **ruling an enumeration incoherent without
> reading the study that produced it is L7 for the third time in two days.**
>
> **What survives, and it is the useful part:** *something* is wrong at §1 / §9 / §11.3 —
> `entity-browser-rust` proved that by citation and `entity-core-go` confirmed it independently, plus
> a third witness they supplied: **Go's `core/types/relay.go` refuses to stub Mode A** — *"DO NOT add
> these types, they'd silently look implemented"* — because an implementer building to the spec found
> no coherent slot. **Three seats agree the mode is mis-filed. None of us has established which of
> the three dispositions is right, and the evidence for that is in eleven unread legacy documents.**
> **Full relay audit handed off:** `HANDOFF-2026-08-17-relay-audit-and-the-extension-landscape.md`.

**Filed by `entity-browser-rust` @ `ca3c760`, from their operator's objection:** *"transmitting
messages across the network on a forward to another peer to try and get it to a destination is not
the same as subscription."* **Accepted. The argument is quotation, not opinion, and every citation
was verified in the landed text before ruling.**

**RELAY's own definition excludes Mode A on all three of its properties.** §1 ¶1: *"A relay is an
intermediary that carries **opaque**, signed, capability-bearing envelopes **between two endpoints**.
… **the intermediary is transport**."*

| | F · S · C | **A** |
|---|---|---|
| **opaque payload** | ✅ | ❌ — §1 has it serve *"a unified stream or **queryable view**"* |
| **two endpoints** | ✅ addressed to someone | ❌ no destination; consumers subscribe to *it*, unbidden |
| **intermediary is transport** | ✅ signature passes through | ❌ it is an **origin** — re-serves under its own authority |

**The first failure is a live contradiction inside one spec, and it is normative on both sides.**
§10.4 MUST NOT, verified verbatim: *"Decode the inner envelope (addressed by the
`data.envelope_inner` hash) in the forward / store / **aggregate** / terminal-delivery path (§9)."*
**You cannot serve a queryable view over bytes you are forbidden to decode.** §1 and §10.4 are both
normative; only one is implementable.

**The other two failures explain §2.1.3.** Verification-through-aggregation is *"the actual hard part
of federation"* precisely because the aggregator is **not** transport — the data is re-served from a
third party's tree under that party's authority. **That problem cannot arise in F/S/C**, where the
bytes and the origin's signature travel together untouched. A hard problem appearing in exactly one
member of an enumeration is a signal about the enumeration.

### §2.0a Correction to this proposal's own §2 — my citation was weaker than I stated

§2 below argues the mechanism *"was specified the whole time"* and calls
`EXTENSION-SUBSCRIPTION` §8.1–§8.2 *"the dissemination tree, **specified normatively**, with a
fan-out table."* **Checked: `## 8. Dissemination Trees (Informative)`.** The section is
**Informative**, and describes itself as *"the convention"* that bounded fan-out *"naturally
produces."*

**The ruling in §2 survives** — it never rested on §8. Cross-peer subscription is permitted by the
capability model (§1's three slots, §2.2's mirror recipe, §5's delivery token) and §3.1 step 3a's
303-at-capacity is normative operation flow. **But "specified normatively" was an overstatement, and
it strengthens browser-rust's case rather than mine:** they caught that my own citation pointed at
*another extension*, which is evidence for **relocation**, not for retracting a deferral in place.

> **A second intra-spec contradiction, found while checking the first.** §8.2 — inside the
> **Informative** section — contains: *"**The normative requirement is:** notification data from the
> upstream peer ends up in the local tree, local emit fires, and downstream subscriptions trigger."*
> **A self-declared normative requirement inside a section labelled Informative**, and it is the
> load-bearing sentence for *both* Mode A and `entity-browser-rust`'s Follow. Two implementations may
> read §8 differently and conformance cannot see it, because Informative text is not gated — the
> cross-peer-seam hazard `AGENTS.md` names, in its purest form.

### §2.0b What the ruling obliges

1. **RELAY drops the aggregate mode from the enumeration** — §1, §2's table, §3.3, §4's op list, §9
   retention, §11.1a, §11.3 — **and the `aggregate` token comes out of §10.4's path list.** RELAY
   ends coherent: **three modes, all transport, all opaque, all addressed.**
2. **SUBSCRIPTION owns the replication story, and the Informative/normative defect is a *pattern*,
   not one section.** `entity-core-go`: besides §8.2, **`EXTENSION-SUBSCRIPTION` §7 carries the same
   buried *"the normative requirement is…"* MUST inside prose.** **Scope the fix as a grep-sweep
   across SUBSCRIPTION** — every self-declared normative sentence sitting in an ungated section —
   not as "promote §8." Two consumers are already building against one of them.
3. **Fan-in and multi-source merge semantics are the genuine new work.** §8 is explicitly
   **fan-out** — *one* source, many subscribers, a tree for scale. Aggregation is **fan-in** — many
   publishers, one view. **Ordering and conflict disposition over N sources is specified nowhere**,
   which is §2.1.1's first residual and is a *replication* question RELAY has no vocabulary for.
   That is why §11.3 could only manage *"semantics per-deployment."*
4. **`REGISTRY` §8.2's federation deferral retires by re-pointing at SUBSCRIPTION**, not by waiting
   on RELAY. Five citation sites.
5. **§2.2's verification answer survives and travels with it** — `published-root` is the anchor for
   a consumer reading an entity out of a third party's tree, which is a replication concern.
6. **Name the third thing so it is not folded into either:** the **control plane** — how peers learn
   a relay exists, its advertised limits, its route table (`:advertise`, `EXTENSION-ROUTE`).

> **The process lesson, and it is this proposal's own.** §4 asks to rule that *"the mode set is
> specified or the mode is not in the enumeration."* **I wrote that ask and did not apply it to Mode
> A itself** — I was completing an enumeration rather than auditing it, which is how a category error
> survives being worked on. **Settle the categorisation before completing the enumeration.**
> Nothing is built for Mode A in any tree, so re-filing costs nothing today and will not stay free.

---

## §2 Ruling asked: the Mode A blocker is void — a category error

**The claim, verbatim from §11.1a:**

> *"Mode A's 'aggregator subscribes to N publisher peers' subtrees' requires **cross-peer
> subscription initiation** that does not exist in current substrate — subscription engines (the
> landed EXTENSION-SUBSCRIPTION and the cohort impls) are **local-tree-only**; cross-peer
> subscription is new choreography layered above."*

**The defect is a conflation of two different things:**

| | |
|---|---|
| *An engine fires on **local** mutations.* | **True, and universal.** The subscribed-to peer's engine watches its own tree. It is how subscription must work, not a limitation of it. |
| *Therefore a subscription can only be **initiated** locally.* | **False.** Who may initiate is a capability question, and `EXTENSION-SUBSCRIPTION` answers it cross-peer throughout. |

**The mechanism was specified the whole time**, in the spec §11.1a names as its evidence:

- **§1** — the three subscribe capabilities, *"each … rooted at a different peer in **cross-peer
  scenarios**"*, an explicit instance of `ENTITY-CORE-PROTOCOL` §5.2's three-slot model.
- **§2.2** — the **cross-peer mirror recipe**, single-hop, with `include_payload` bundling the bytes
  so the subscriber needs no follow-up cross-peer GET.
- **§5** — *"the delivery-token mechanism applies to **cross-peer delivery** where subscriber and
  server are distinct principals."*
- **§8.1–§8.2** — the **dissemination tree**, with a K=10 fan-out table, and the intermediate peer's
  requirements enumerated: *(1) a subscription on its upstream peer, (2) a mechanism to write
  received notification data to the local tree, (3) capacity to accept downstream subscriptions* —
  and the normative requirement stated: *"notification data from the upstream peer ends up in the
  local tree, local emit fires, and downstream subscriptions trigger."*

**§8.2 is the aggregator's requirement list.** An intermediate peer holding upstream subscriptions,
materializing into its local tree, and serving downstream subscribers **is** Mode A.

**The operator's framing is the load-bearing one and is stronger than "someone has since built
it":** bilateral capability exchange is what determines which exchanges are permitted. A peer
subscribing across a boundary is an ordinary EXECUTE under a grant the target issued — §5.2's three
slots. **Nothing had to be added for it to be possible**, so there was never a substrate absence to
sequence behind.

> **Corroborating, at lower weight, and deliberately not the basis.** `entity-core-rust`
> `bindings/sdk/src/follow.rs` (live at `f23fb3b`, clean) packages the composition and its module
> doc says it mirrors `entity-workbench-go`'s follow chain. **An SDK convenience does not create a
> capability.** Citing it as the unblocker would repeat §11.1a's error pointing the other way —
> inferring the substrate from what happens to be built on it.

### §2.1 What is *not* thereby specified — the honest residual

Retracting the blocker does **not** make Mode A complete. Three things are genuinely unspecified and
this proposal owes them:

1. **Aggregation semantics.** §11.3 currently reads *"mechanism exposed; semantics per-deployment"* —
   which is a deferral wearing a delegation's clothes. Union of N sources needs an ordering and a
   conflict disposition, and per-deployment answers diverge at a consumer seam.
2. **Retention.** §9 gives Mode A *"operator-configured retention; default unlimited."* Unlimited is
   not a default, it is an unbounded accumulation — the same finding browser-rust filed against
   CONTENT (`ROUTING-2026-08-16-f`).
3. **Verification through aggregation — the real one, and `entity-browser-rust` has supplied the
   answer (`ROUTING-2026-08-17-comprehensive` §2.1.3).** §11.1's own **Q4** already rules that *the
   aggregator does not re-sign; receivers verify against the original publisher's signature.* But
   §8.2's mechanism materializes upstream data **into the aggregator's local tree** via ordinary
   tree writes. **How a downstream consumer verifies the original publisher's signature over an
   entity that now sits in a third party's tree is not specified anywhere.** This is the actual hard
   part of federation and it was hidden behind a blocker about subscription plumbing.

> ### §2.2 The verification answer, and a **third** stale deferral it exposes
>
> `entity-browser-rust` hits §2.1.3 one layer up — a followed prefix materializes a remote peer's
> subtree into *their* tree, so anyone reading it from them is exactly the third-party-tree consumer
> §2.1.3 describes — and proposes the mechanism: **`system/peer/published-root`**
> (`EXTENSION-TREE` §3.3a).
>
> **Checked, and it is not merely plausible — it is the corpus's own answer to this threat model.**
> §3.3a, verbatim: *"A published root is a **signed, mutable pointer to a trie root** that a publisher
> commits to serving. It is the anchor of the **walk-from-signed-root threat model**: a consumer
> fetches it, verifies the signature, and walks the hash-chain from `root_hash` — **never trusting
> paths the host claims outside that chain**."* An aggregator is precisely such a host.
>
> **So the proposed disposition is:** an aggregate relay serves the upstream's `published-root`
> alongside the materialized entities, and the consumer walks from the signed root. That answers
> Mode A's verification question **and** what a Follow must carry to be verifiable — one ruling,
> two consumers.
>
> **And it voids RELAY §7.** §7 declares a **tracked blocker**: *"For Mode S to serve 'current state
> of publisher's tree' use cases, a signed mutable pointer at a named location (the IPNS-equivalent)
> is required. **This lives in CONTENT, not RELAY** … the 'give me the publisher's latest' cases are
> **gated on the CONTENT signed-mutable-pointer work** and are **not deliverable in RELAY v1**."*
> **That pointer exists** — landed 2026-08-08 in `EXTENSION-TREE` §3.3a. **Pin caveat, raised by
> `entity-core-go` against this proposal:** *"built three-way"* was **repeated from §3.3a's own
> provenance note, not measured** — the spec's claim about implementations propagated as arch's
> verified build-state, which is the precise failure `AGENTS.md` forbids and **AP-12**, which arch
> invoked on them, governs. Re-checked at file level 2026-08-17: go `6fc0af3`
> (`ext/publishedroot` + `core/types/published_root.go`), rust `f23fb3b`
> (`core/peer/src/published_root.rs`), py `1d951f6` (`peer/published_root.py` + unit test).
> **`entity-core-go` signs only the Go leg on semantics** (signed mutable pointer, `Current()` =
> latest, signature-target tests green). **rust/py carry the file; their §7 semantics are
> unverified by anyone.** File presence is not a conclusion about behaviour — **L8** — so the
> three-void-deferrals finding rests on **one measured leg and two unmeasured ones** until each is
> pinned to a diff that carries it. It is in TREE, not CONTENT, which is why a reader looking where §7
> points finds nothing.
>
> **That is the third of RELAY's deferrals to be void on landed text** — Mode A (§2), Mode C's fired
> condition (§3), and now Mode S's latest-pointer. **Three of three examined.** The pattern is not
> three coincidences: `EXTENSION-RELAY` is at v1.2 and its scope cuts were written against a corpus
> that has since landed what they wait for, with **nothing in the process obliged to re-check them.**
> §4's ask — stop deferring spec text — is the structural fix; §7 item 8's dangling
> `EXPLORATION-TRANSPORT-COMPOSITION-AND-FALLBACK` is the fourth candidate and the only one whose
> target does not exist at all.
>
> *(§3.3a's own provenance note is the same story one level down: the type was `NORMATIVE-LOCKED` in
> a legacy proposal that **never crossed into this corpus**, while three landed `EXTENSION-NETWORK`
> MUSTs cited it as "(planned)" — and all three impls built to it anyway.)*

**Ask:** rule the blocker void, and open §2.1's three items as the real Mode A work.

---

## §3 Ruling asked: Mode C's deferral condition has already fired

§11.1 defers circuit relay with a **condition attached**:

> *"Defer to a follow-on proposal **when a driver materializes**; the static-host Mode S pattern
> covers most NAT cases via Mode S inbox addressing (§6.2.1)."*

**A driver has materialized, and it is the release product.**

- `entity-browser-rust` is a **listener-less** peer whose only path to reachability is rendezvous
  (`ROUTING-2026-08-16-e`). Mode S inbox addressing does not cover it — Mode S is store-and-poll for
  *envelopes*, not a bidirectional circuit for a media/data channel.
- **Q18** established there is **no TURN credential mechanism anywhere in `specs/` or `guides/`**,
  which is why `EXTENSION-REGISTRY` §3b.0a had to declare `policy: open` the only interoperable
  data-relay deployment and `members`/`metered` reserved shape.
- **Mode C is the TURN-equivalent** — §1 names libp2p circuit-relay-v2 / TURN as its models.

So the corpus has a **relay-shaped hole being worked around in three places at once**: Mode C
deferred, `data_relay` advertised in REGISTRY §3b with no way to authenticate to it, and a browser
peer that cannot be reached without one. **These are one gap.**

**Ask:** rule Mode C's condition met, and scope Mode C together with Q18's credential channel rather
than separately — the reservation discipline and the credential are the same conversation.

---

## §4 Ruling asked: stop deferring *spec text*

**This repo has no installed base.** `AGENTS.md`: *"No backward compatibility. No installed base —
write every change as the only design; no legacy paths, migration windows, dual-kind acceptance, or
mirror-writes."*

A **"v1 scope cut"** that names a mode, reserves its entity types and operations *"for
forward-compatibility"*, and defers their normative text **is a migration window in disguise.** It
buys compatibility with a future we have not designed, at the cost of shipping an enumeration whose
members are half real.

**Deferral is an implementation-scheduling instrument, and it belongs to the impl teams.** An impl
shipping forward+store now and aggregate later is ordinary sequencing. **A specification deferring
its own text is just unfinished**, and it is worse than unfinished because the deferral is *load
bearing* — other specs inherit it. `EXTENSION-REGISTRY` §8.2's federation deferral exists only
because RELAY §11.1a said so, and it is cited at five sites in REGISTRY alone.

**The measured cost in this one spec: 27 occurrences of deferral language, two of four enumerated
modes with no normative text, one dead citation, and one inherited cross-spec deferral that closed
registry federation for the whole ecosystem.**

**Ask:** rule that RELAY carries **no deferred modes** — the mode set is specified or the mode is not
in the enumeration. Then state implementation staging as what it is: a note about what implementations
ship first, carrying no normative weight and reserving nothing.

---

## §5 Ruling asked: say the modes in English

**`Mode A` is a lookup-table label**, and the words it looks up are already in the spec: §1 defines
*Mode F — **Forward***, *Mode S — **Store-and-poll***, *Mode A — **Aggregate***, *Mode C —
**Circuit***. The letters carry no information the words do not.

This is the same defect class `AGENTS.md`'s terminology rules already govern — *"extension" not
"actualizer"*, *"core" not "kernel"*, *"bootstrap"* reserved for real bootstrapping. **A reader
should not need a decoder ring to read a normative sentence**, and every prose sentence citing
"Mode A" costs a lookup. The letters also actively hid this defect: *"Mode A is deferred on a
cross-peer subscription dependency"* reads as a technical fact, where *"aggregating relays are
deferred because peers cannot subscribe to each other"* reads as the false claim it is.

**Ask:** drop the letters from prose and normative text; use `forward`, `store-and-poll`,
`aggregate`, `circuit`.

**The wire half is separable and must be sequenced, not bundled.** §3.x's advertise entity carries
`modes: [<mode_id: "F"|"S">]`. Per `STYLE-NAMING-CONVENTIONS.md` these are enum string values and so
should be **kebab** — `forward`, `store-and-poll`, `aggregate`, `circuit`. That is cross-impl
observable — three peers must agree on the strings — but it changes no type definition (§6). **Prose
now; wire values with the three core peers** (§6).

---

## §6 Build state, read live 2026-08-17 — who this coordinates with

Pinned and observed by reading the trees, not reported:

| Seat | HEAD | Relay surface |
|---|---|---|
| `entity-core-go` | `de8f807` | `ext/relay/` + `core/types/relay.go` + `cmd/relay-fixtures`. Operations present: **`forward`, `put`, `poll`** — exhaustive over the op-name literals. No `subscribe` |
| `entity-core-rust` | `f23fb3b` | `extensions/relay/src/` — `forwarder.rs` + `store.rs` + `handler.rs` + `resolver.rs`. Same two modes by structure |
| `entity-core-py` | `1d951f6` | `tests/integration/test_relay.py` + `test_relay_conformance.py` |
| `entity-core-keystone` | `6a11cfc` | **Not involved. Carries no relay handler.** See the correction below |

**All three ground-up impls built exactly the two specified modes.** The deferral did not merely sit
in a document; it propagated into three implementations. **Nothing here is an impl defect** — every
seat implemented the spec as written.

> **Correction — an earlier draft of this proposal claimed keystone was in scope. It is not, and the
> error is instructive.** The claim rested on finding
> `protocol-generator/asm-*/reference/typestore/system_type_system_relay_advertise.bin` in keystone's
> tree and reading *file exists* as *implements the extension*. Two things are wrong with that:
>
> 1. **Keystone carries no relay handler.** Those files are the protocol-generator's reference
>    typestore for the **whole system type registry** — type definitions, not an implementation of
>    `system/relay`.
> 2. **The mode identifiers are not in any type definition.** Read the bytes: the advertise typestore
>    declares `modes` as `array_of → primitive/string`. `"F"` and `"S"` are **free string values**,
>    nowhere in the schema. Renaming them regenerates nothing.
>
> **So the wire half is a three-peer agreement, not a typestore migration** — and the caution about
> not renaming under the in-flight 44-peer census was wrong too: that census runs `--profile core`,
> and relay is an optional extension. *Found by the operator; the failure was inferring a build state
> from a filename instead of opening the artifact, which is the inward form of the rule `AGENTS.md`
> already carries.*

**Coordination this needs, and it is why nothing folds this week:**

- **Application tier** (`entity-browser-rust`, `entity-workbench-go`) — the aggregate mode is what
  their federation and share/follow work stands on (`ROUTING-2026-08-16-g` Q1/Q2), and the circuit
  mode is browser reachability. Route §2 and §3 to both.
- **Core peers** (go/rust/py) — §5's wire values and any new operations.
- **Not keystone, and not gated on the census** — both claims withdrawn above.

---

## §7 What "complete on relay" means — the checklist this proposal closes against

1. Four modes, four normative specifications. No mode named without text.
2. Aggregation semantics ruled, not delegated per-deployment (§2.1.1).
3. Mode A retention bounded (§2.1.2).
4. **Signature verification through an aggregator specified** (§2.1.3) — the one that matters.
5. Circuit reservation discipline + bandwidth accounting, scoped with Q18's credential channel (§3).
6. `EXTENSION-REGISTRY` §8.2 federation deferral retracted, and its five citation sites corrected.
7. **Mode S "publisher's latest" — the blocker is already void (§2.2).** `EXTENSION-TREE` §3.3a's
   `system/peer/published-root` **is** the signed mutable pointer §7 waits on; it landed 2026-08-08
   and all three impls built to it. §7 must be rewritten to cite it, not to gate on it. *(Separately,
   `ROUTING-2026-08-16-f`'s CONTENT reclaim ruling is still owed — that is an eviction verb, a
   different gap.)*
8. Mode F routing-on-failure: §11.1a gates it on
   `EXPLORATION-TRANSPORT-COMPOSITION-AND-FALLBACK.md`, **which exists nowhere** (§1a). Either the
   fallback loop gets specified here or the gate comes out — a normative gate on a nonexistent
   document is not a deferral, it is a dangling pointer.
9. Modes in English; wire values kebab, with the cohort.
10. RELAY's four citations resolved — `PROPOSAL-EXTENSION-RELAY.md` and the two load-bearing arch
    rulings reconstructed into this repo, or the citations retargeted. **A landed spec must not cite
    a document outside the corpus.** Corpus-wide this is 14 citations across 9 specs (§1a) and is
    `CORPUS-REFERENCE-INTEGRITY`'s work, not RELAY's; RELAY's four are in scope here because the
    rulings they name are the ones this proposal must not disturb.

---

## §8 Sequencing

**Nothing in this proposal is Friday-release work** and none of it should be folded before the
re-release. Item 7 is the exception worth noting: the CONTENT pass that rules browser-rust's reclaim
gap is pre-Friday (**F1** in the triage) and it touches the same signed-mutable-pointer area, so
**author it so RELAY §7 can be un-gated later without a second CONTENT revision.**

Post-release order: §2 and §3 rulings (they are one conversation with Q18) → §4's disposition rule →
§2.1's three real gaps → §5's wire half with the three core peers.
