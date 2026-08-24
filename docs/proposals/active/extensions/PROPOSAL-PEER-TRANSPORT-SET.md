# PROPOSAL — a peer's statement of where it is: signed by the peer, servable by anyone

**Status:** DRAFT — **fourth pass, refined on `entity-browser-rust`'s answers (`8cd3010`).**
Stress-tested, measured, then **corrected by the operator on the third pass's central premise**, then
**corrected again by browser-rust on T11's reason.** All five open questions stand as decided; **R4 is
restored**; and T11 generalizes — **one gap, three substrates**, which reframes what this record is
(§6 T11). Ready for a cohort round; not ruled.

> **What this record is, in one sentence, now that T11 has generalized it.** Not *"the A-record gap"*
> — **the peer-id-keyed answer for every way a peer can be reached**: dial it directly, meet it at a
> rendezvous, or leave mail with its relay. All three are unanswered from a `peer_id` alone today, and
> all three are answered by one signed, self-authenticating record that anyone may serve.
**Tier:** extensions — `EXTENSION-NETWORK` §6.5 / §6.7 (the record and its precondition);
`EXTENSION-REGISTRY` §2.3 / §3 / §12 (scoping corrections only)
**Origin:** operator, 2026-08-20, rejecting this repo's own scoping — twice. Derivation in
`docs/research/explorations/EXPLORATION-NAMING-LANDSCAPE-AND-THE-DNS-MAPPING` §4.6–§4.10.
**Read at:** arch `68f13a6` · go `5020e62` · rust `302b7f4` · py `5205895` · browser-rust `7bc1ccf` ·
workbench-go `7492fe8`
**Release scope:** **v2. Touches nothing on `EXTENSION-REGISTRY` v1's IN list.**

---

## §0 What the second pass changed, and why

Three operator corrections, each of which moved the design rather than the wording:

1. **"Anyone can serve it if it's signed. The data, the hash, and the cryptographic proof is what
   makes it valid."** — this is the governing principle and it **dissolves two of the five open
   questions** (§1).
2. **"Published-root is a convenience thing. I could sign every single entity, hash it, sign the
   hash."** — correct, and it reframes the whole set-vs-member question: **signing is a primitive;
   aggregation is the design choice** (§2).
3. **"Are these either-ors? Are they exclusionary? You're acting like we can only pick one."** —
   they were not, and the first pass framed three separate questions as forced choices. All three
   are layers (§1, §2, §5).

And one question re-opened deliberately: *maybe it is a different record type.* **It is. It is not an
A record — it is an SRV RRset** (§3), and that reframing brings 20 years of DNSSEC operational
experience directly onto the set-vs-member question with an unambiguous answer.

### §0.1 What the third pass changed — `entity-browser-rust` measured it and two things moved

`ROUTING-2026-08-20-d` (their `5c84154`), answering §10's two assignments to that seat. **Both
answers changed the proposal.**

1. **§9.2 is `inline`, and one of the two justifications the second pass gave for hashes is
   self-refuting** — the cross-peer dedup it half-rests on is **zero by construction**, and
   **this proposal's own consumer rule 6 is what makes it zero**. Re-verified here, in their tree,
   before acceptance: the emitted profile is **exactly 615 bytes** and the peer-id
   `2KEb7Hgmo…PTtNA` occurs **three times** in it — once as the field, twice inside the URL
   prefixes built from it. Two peers' profiles can never be byte-identical, so they can never share a
   content hash. **Decided: inline.** §9.2, and it deletes a consumer rule (§4).
2. **R4 is withdrawn as discharged, and that is a design finding rather than a caveat.** The second
   pass read R4 — *"publishable by a peer with no tree"* — as a **storage** question. It is an
   **expressibility** question: a browser peer has no listener, so *"being reachable is an activity
   it performs, not an address it has,"* and **an honest transport-set for that deployment has zero
   profiles.** The record cannot say how such a peer is reached. §5's table no longer shows ✅, and
   §11 names the successor. **↑ The premise in item 2 is false — see §0.2, which supersedes it.**

### §0.2 What the fourth pass changed — the premise under item 2 is false

> **Operator, 2026-08-20:** *"Just because a browser doesn't have a listener doesn't mean it can't
> communicate. It can work through a proxy, through a WebSocket, through a relay. Listening isn't the
> property that determines your interactivity — it's your ability to communicate through some sort of
> shared intermediate exchange."*

**Correct, and the corpus already says so in three places the third pass did not open.**

1. **`EXTENSION-NETWORK` §6.5.2c, on the `websocket` profile:** it *"additionally pushes to a
   **non-listening** target (e.g. a browser page) **down that target's own outbound socket**."* The
   spec names listening as *not* the property, in the section defining the browser's transport.
2. **§6.5.2d, `system/peer/transport/webrtc`:** a landed, durable transport profile carrying **no
   `endpoint` at all** — *"the profile declares the capability + negotiation parameters, not an
   address."* **"Reachable by activity, not by address" is not something the record cannot express.
   It is a profile shape that has been landed since 2026-08-02.**
3. **§6.5.1b's duplex table** lists `websocket` push *"incl. to a non-listening client (browser)"* and
   `webrtc` as *"NAT-traversing"*, both as one connection reused both ways.

**So the third pass's conclusion — *"an honest transport-set for a browser peer has zero profiles"* —
is refuted by the spec this proposal amends.** The consequence is not cosmetic: on that reading
**every browser peer in the ecosystem publishes a signed statement that it is not reachable, while
running a daemon that makes it reachable.** Consumer rule 8 would then be load-bearing in the most
common case *and wrong there* — protecting a false statement rather than an honest one. The empty set
is for a peer that is genuinely unreachable, and that is not the browser.

**The exact question was asked and answered eighteen days earlier, in this repo, in a folded
proposal.** `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` §6 **open item #1** is *"Does a browser peer publish
a `webrtc` profile at all?"* — leaned **no**, reasoning that *"a browser is a permanent initiator… its
reachability is rendezvous/session-scoped."* **L7, seventh instance:** the corpus held the answer and
was not searched. `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE` was not opened either
(**L11**).

**And the lean has since expired — voided by the filing seat's own build.** Open item #1 rested on one
counter-argument: *"the alternative needs a story for how the two meet at a rendezvous when the browser
is offline (they can't; rendezvous needs both present)."*

| | Landed | What it is |
|---|---|---|
| open item #1 leans "publishes nothing" | **2026-08-02** | On "nobody is at the bucket" |
| `entity-browser-rust` `bb27e64` `src/rendezvous.rs` | **2026-08-13** | `tag`/`secret`/`lobby` reach the app tier |
| `entity-browser-rust` `c5e16ef` `src/reach_keeper.rs` | **2026-08-16** | *"the serving side attempts too, **so a peer that only serves is reachable**"* |

**The commit title is the refutation.** `reach_keeper` is a standing presence daemon — their own doc
calls it *"the accept loop, spelled as presence at a rendezvous, because that is the only spelling the
substrate offers."* **A peer standing at a bucket is waiting exactly as a listener waits; the
difference is where the waiting happens, not whether it happens** — and a durable standing activity is
a publishable, signable fact. What made "publish nothing" right in early August is that the
advertisement would have been a lie. **`reach_keeper` is what makes it true, and it landed four days
before the third pass cited the lean's conclusion as a design limit.**

**L9, fifth instance, and the worst-placed one:** *a deferral is a build-state claim and expires like
one.* The four prior instances were `EXTENSION-RELAY`'s deferral list. This one was voided **by the
seat whose shipping code was the evidence for it** — and neither they nor arch re-opened it, because
nothing re-reads a resolved open item in a folded proposal. Their `reach_keeper` doc still opens *"a
browser has no listener"* and reasons forward from it, in the module that disproves the inference.

**What changes:** R4 is **restored** (§5) · §9.3's *"primary case / the whole record for every browser
peer"* framing is **withdrawn** — the empty set stays valid and stops being the browser's expected
shape · §11 stops being *"a new MX-shaped record type"* and becomes **two additions to mechanisms that
already exist** · §6 gains **T11**, which is the one thing genuinely missing · §8 gains D10–D11.

---

## §1 The governing principle, and what it dissolves

> **A signature makes the data valid. Serving is not authority.**

This is already the substrate's posture everywhere else — content-addressed bytes verified by hash,
`published-root` verified against `peer_id`, `serve_scope` gating *which* entities a host answers for
and never *whether they are true*. Applied here:

**Dissolved — "who may serve it?"** *(first pass Q2)* **Anyone. There is no rule to write.** A
registry, a relay, a CDN, a peer that met P once, a QR code on a business card. Serving a signed
record confers nothing on the server and requires nothing from it. The only rules are consumer-side
verification rules, and they are in §4.

**Dissolved — "which lookup shape?"** *(first pass Q4: a NETWORK op, a REGISTRY backend, or both)*
**All of them, and they are not alternatives.** Once the record is self-authenticating, **none of
the retrieval paths is a trust boundary**, so there is nothing to arbitrate between them. A consumer
takes the first answer that verifies. The first pass treated this as an architecture decision; it is
a plumbing decision, and the reason it is cheap is precisely the signature.

**This inverts the ordering of the work.** The first pass presented the record and the lookup as two
halves. They are not equals: **the signature is what makes the lookup problem easy.** Without it you
need a *trusted* index — which means an authority, a trust anchor, configuration, and a revocation
story, for a question that needs none of them. With it you need *any* index, and "ask whoever you are
already talking to" becomes a legitimate implementation.

---

## §2 Signing is a primitive; aggregation is the design choice

V7 §5.2 gives **any** entity a signature at the invariant pointer
`system/signature/{hex(content_hash)}`. So "sign the profile **or** sign the set" was never the
question — both are always available, neither excludes the other, and `published-root` is not a
distinct mechanism but **an aggregation convenience over exactly this primitive**: sign one root
instead of validating every entity under it.

The real question is: **what does aggregating buy, that per-item signing does not?**

| | A signed **member** proves | A signed **set** proves |
|---|---|---|
| | *"P authored this profile"* | *"these are **all** of P's transports, as of `seq` N"* |
| Detects a forged member | ✅ | ✅ (via hash cover — the set's signature covers the member hashes, and content-addressing does the rest) |
| Detects a **dropped** member | ❌ | ✅ |
| Detects a **stale** answer | ❌ | ✅ (`seq`) |

**The set's signature already authenticates the members transitively**, because it signs their
hashes and a hash is the member. So per-member signatures are **redundant for the common path** and
buy exactly one thing: a profile can be quoted **in isolation**, with its own provenance, by a party
that does not carry the set. That is a real case — `binding.transports` hands a consumer bare profile
hashes today — so member signatures are a **MAY**, not a competing design.

**The set is the load-bearing one, and the reason is the failure this corpus keeps meeting.** Without
a signed set, a party serving P's transports can hand you two of three profiles and you cannot tell
"P has two" from "I was given two." That is the **absent-vs-withheld collapse**, which the discipline
charter has now recorded **six times** — the short trie walk, the revocation `404`, the withheld
interior node, `coverage`, the publish that reported success and wrote nothing, and now this. **Every
time it has appeared, the fix has been the same: make the complete set the signed unit.**

### §2.1 DNSSEC made this exact choice, and it is the strongest external evidence available

**RRSIG signs an RRset — never an individual record.** That is not an implementation detail; it is
the design decision, and the reason is verbatim ours: without it a resolver cannot detect that a
record was dropped from a set. DNS has run that decision in production for two decades across every
zone on the internet.

It also brings the second half: **NSEC exists because a signed set still cannot prove the absence of
a set.** Our analog is already landed — `binding-manifest` `coverage: "complete"`, which the spec
already cites *"DNSSEC-NSEC / TUF"* by name. The pattern is consistent and we are already inside it.

**Conclusion, and it is not a compromise between the two options:** sign the set (MUST, if you want
completeness), sign members (MAY, for isolated quotation). They compose.

---

## §3 It is not an A record. It is an SRV RRset — and we have been building SRV without saying so

The operator's question — *maybe it is a different record type* — is the right one, and the first
pass answered it loosely by calling this "the A-record gap."

**A DNS `A` record is `name → address`. A leaf. Ours is not that.** A transport profile carries
`endpoint` **plus** `priority`, `supported_ops`, `freshness`, `nonce_required` and `cap_flow` — a
*way to reach a service*, with a preference, not an address.

**That is SRV**, and our own text has said so without drawing the conclusion:

- §6.5.1a: *"`priority` — lower value = more preferred **(DNS-SRV semantics)**"*, and *"Weight-based
  load-balance among equal-priority mirrors — **SRV `weight`** — is NOT v1."*
- `supported_ops` is the thing SRV expresses by having a **separate `_service._proto` name per
  service** (`_sip._udp`, `_xmpp-client._tcp`). We fold it into a field instead — the same
  information, one indirection cheaper.
- `cap_flow: egress | ingress | both` has **no DNS analog at all** and is a genuine addition. Nostr
  reached the same distinction independently in NIP-65 (read relays vs write relays), which is the
  kind of convergence worth noticing.

**So the mapping is:**

| | DNS | Ours |
|---|---|---|
| one record | `SRV` | one `system/peer/transport/{peer_id}/{profile-id}` |
| the set of them for a name | the **SRV RRset** | **missing — this proposal** |
| the signature over that set | **RRSIG** | **missing — this proposal** |
| proof there is no set | NSEC / NSEC3 | `coverage: "complete"` (landed, deferred impl) |
| the address the target resolves to | `A` / `AAAA` | the `endpoint` **inside** the profile — we inline it, DNS indirects |

**We inline what DNS indirects** (SRV → target name → A), which removes a round trip and a whole
class of dangling-target bugs. That is a real simplification and it is defensible. **What we skipped
is the RRset and its signature**, which is the one part DNS treats as load-bearing.

---

## §4 The record

```
system/peer/transport-set := {
  fields: {
    peer_id:      {type_ref: "system/peer-id"}    ; whose. the signature verifies against THIS
    profiles:     {array_of: {type_ref: "system/peer/transport"}}
                                                  ; INLINE profile entities (§9.2 — decided).
                                                  ; MAY be empty; an empty set is the signed
                                                  ; statement "I am not directly dialable" (§9.3).
                                                  ; NOT the browser's expected shape — a browser
                                                  ; peer publishes an endpoint-less §6.5.2d-shaped
                                                  ; profile (§0.2, §11). Empty is for a peer that
                                                  ; is genuinely unreachable, which is rarer.
                                                  ; array order is NOT significant — §6.5.1a D1's
                                                  ; (priority asc, profile-id lex) stays the only
                                                  ; selection rule, so no second ordering is invented
    seq:          {type_ref: "primitive/int"}     ; monotonic per peer_id; MUST increase per publish
    published_at: {type_ref: "primitive/int"}     ; ms since epoch, UTC
    expires_at:   {type_ref: "primitive/int"}     ; REQUIRED — see §6 T2
    predecessor:  {type_ref: "system/hash", optional: true}   ; BARE; prior set's content_hash
  }
}
```

*(`system/peer/transport` as a member type-ref is the family; the concrete member is a
`system/peer/transport/<transport_type>` entity, and §6.5.1a D5's suffix-match rule is what
identifies which. If the type system cannot express the family as a member type, the member is
carried as an opaque entity and D5 is the decoder's discriminator — an authoring detail for the
fold, not a design choice.)*

**Signature carriage** — `system/signature/{hex(transport_set.content_hash)}`, V7 §5.2
target-matching, verified against `peer_id`'s key derived locally from the Base58 form (V7 §1.5).
**Identical to `published-root`'s carriage on purpose:** one convention for *a peer's signed
statement about itself*, not two.

**Consumer rules (MUST):**

1. **Verify** the signature against `peer_id`. Failure ⇒ discard entire. Never partially honored.
2. **Reject `seq` lower** than one already accepted for that `peer_id`.
3. **Honor `expires_at`.** An expired set is not a weaker answer; it is **not an answer**.
4. **`published_at` is a signed lower bound on age and MUST NOT be read as freshness** — §3.3a's
   sentence applies verbatim: a peer that has not republished and an origin withholding a newer set
   are byte-identical at the consumer.
5. ~~**A profile hash that does not resolve leaves the set incomplete…**~~ **DELETED by the inline
   decision (§9.2), and deleting it is one of the four arguments for inlining.** With members
   carried in the set there is no hash that can fail to resolve, so incompleteness is
   **unrepresentable** — no rule to state, no consumer that can skip it, and **no seventh instance
   of the absent-vs-withheld collapse.**
6. **A member whose inner `peer_id` ≠ the set's `peer_id` makes the set malformed and it is rejected
   entire.** *(Under the hash design this rejected one member; inline makes it a well-formedness
   property of a single signed artifact, which is stronger and simpler.)*
7. **The set is the peer's belief, not ground truth** (§6 T1). Consumers try profiles in
   §6.5.1a D1 order and fall through on failure, exactly as today.
8. **An empty `profiles` array is a valid, meaningful answer** — *"this peer is not directly
   dialable"* — and is **distinguishable from having no set at all** (§9.3). A consumer MUST NOT
   report an empty set as *"no record"* or as *"peer down."*

**Optional:** a peer MAY **additionally** publish any member as its own addressable, separately
signed entity (V7 §5.2 permits it and §2's isolated-quotation case wants it). That is a publishing
choice with no consumer obligation and **no indirection on the read path** — the recommendation is
`entity-browser-rust`'s and it keeps §2's MAY without paying §9.2's cost.

### §4.1 Relationship to `published-root` — not exclusive, and there is only one `seq` stream

The first pass listed "a second `seq` stream per peer" as an objection, citing `entity-browser-rust`'s
warning. **That warning was about *forking* one stream — two roots for one tree, which destroys
rollback detection. Two records answering two different questions are not a fork.** But they can be
played against each other, and that is a real hazard, so:

**Bind the transport-set into the tree at `{peer_id}/system/peer/transport-set`.**

- A peer **with** a published root: the root covers the transport-set, so the root's `seq` is the
  freshness authority and there is **one stream**. No fixed point — the set does not commit to
  `root_hash`, so binding it changes the root without changing the set.
- A peer **without** a tree (§5 R4 — the browser peer): the set stands alone and its own `seq`
  carries the freshness.
- **Same bytes, same signature, two retrieval paths.** A consumer walking a root and a consumer
  handed a bare record get the identical artifact.

That answers first-pass Q1 without choosing: **not "a separate record *or* walk the root" — one
record, reachable both ways.**

---

## §5 Requirements, derived before the shape (unchanged from the first pass, now with the stress results)

| # | Requirement | Source | Stress result |
|---|---|---|---|
| R1 | Self-authenticating against `peer_id` alone | The querier holds only P | ✅ §4 |
| R2 | Servable by any party | §1's principle | ✅ and it is the *reason* §1 dissolves two questions |
| R3 | Rollback-defended and expiring | `published-root`'s lesson | ✅ `seq` + `expires_at` **required** (§6 T2) |
| R4 | **Reachable-by-record for a peer with no listener** | §6.5.1a admits *"a browser peer that cannot self-host its tree"* | ✅ **RESTORED 4th pass (§0.2).** The storage half was always fine (§4.1's standalone path). The expressibility half is **discharged by §6.5.2d** — an endpoint-less profile declaring reachability-by-negotiation is a landed shape. *(The 3rd pass withdrew this on "no listener ⇒ no address ⇒ empty set." Listening is not the property — §6.5.2c pushes to a non-listening target by name.)* **The residue is not R4; it is T11** |
| R5 | No IDENTITY, no REGISTRY, no name | §1 of the first pass | ✅ — with one honest exception, §6 T4 |
| R6 | Does not weaken §2.3's fallback loop | The name carries the revocable binding | ✅ untouched |
| **R7** | **The peer must be able to know what to sign** | **New — §6 T1** | 🟡 **a rule, not a dependency.** §6.7.2 `check-reachability` is the confirmation and it is **built** (go `5020e62`, py `5205895` responders; rust `302b7f4` client) — measured today, not quoted from the spec's stale note |

---

## §6 Stress tests

**T1 — A NAT'd peer does not know its own address, and our own spec already forbids the shortcut.**
Amendment 13 (§6.7) is explicit: *"the observed address is **never** persisted to
`system/connection.address` or a transport profile (it is a responder-side fact and every durable
address field in this spec is dialer-side dialable-endpoint state — the cheap fix corrupts §10
dispatch for every other reader)."*

So a peer behind NAT **cannot honestly populate a transport-set from reflection alone.** It needs a
confirmation step, and that is `§6.7.2 check-reachability` (dial-back).

**This is the evolution lesson, and it is exact.** libp2p's AutoNAT and its signed peer records are
contemporaneous *because they are the same problem*: **you cannot sign "reach me here" until
something tells you that you are reachable there.** Signing an unconfirmed address does not merely
fail — it publishes a signed, durable, widely-cached wrong answer, which is worse than the unsigned
guess it replaces.

> **The first draft of this test said `check-reachability` is "unbuilt in every seat," on the
> strength of `EXTENSION-NETWORK` Amendment 13's own build-state note. That note is dated
> **2026-08-07** and it is stale.** Measured today instead of quoted:
>
> | Seat | Commit | State |
> |---|---|---|
> | `entity-core-go` | `5020e62` | **responder built** — `ext/network/reachability.go` `handleCheckReachability`, with the §6.7.2 rate-limit (`429`) and the live-accepted-connection guard, plus `reachability_test.go` |
> | `entity-core-py` | `5205895` | **responder built** — `entity_handlers/reachability.py` `handle_check_reachability`, same two guards |
> | `entity-core-rust` | `302b7f4` | **client side built** — `core/peer/src/srflx.rs` dials a responder and checks the result type against §6.7.2. Responder side not confirmed by this search |
>
> **This is exactly the failure mode `AGENTS.md` opens with, and I walked into it from inside our own
> spec:** a dated measurement embedded in normative text, carried forward across thirteen days and
> three trees, and about to become a **sequencing constraint on a design**. The spec sentence was
> true when written. *A build-state claim in a spec is re-read as a rule* — and I re-read it as one.

**Corrected, R7 is much weaker and the proposal is stronger for it.** The confirmation mechanism
exists in the cohort. The remaining constraint is a **rule**, not a dependency: **a peer SHOULD NOT
publish a transport profile in a signed set for an address it has not confirmed dialable**, and
§6.7.2 is the confirmation. A publicly-dialable peer needs nothing. **`EXTENSION-NETWORK`
Amendment 13's build-state note should also be removed from the spec** — see D5.

**T2 — Withholding, and why `expires_at` is REQUIRED rather than optional.** `seq` catches a rollback
below a floor you already hold. It catches **nothing** on first contact — a hostile server hands a
never-seen consumer an old, validly-signed set and nothing detects it. `published-root` has this
exact hole and states it honestly. **The only bound that works on first contact is expiry**, so
`expires_at` is required here where `published-root` has none. *(First-pass Q5, closed.)*

**T3 — Partial serving, and the win over the status quo.** A server hands you the set and withholds
one member entity. **The set's signature covers the hash list, so you can see one is missing** — and
consumer rule 5 forbids treating that as complete. Under today's design, the same withholding is
**invisible**: you get a shorter `binding.transports` list and no way to know. **Seventh instance of
absent-vs-withheld, and the first one where the fix is already in the artifact rather than added
after.**

> **Corroboration, cited last and labelled as such (L18) — and it is measured, which is rare for
> this seam.** `entity-browser-rust` `ed36713` (2026-08-20, landed while this pass was being
> written) built signed-root enumeration for the registry browse surface and rejected the obvious
> implementation for precisely this reason: a walk that skips a missing `Entry::Link` *"returns a
> BTreeMap with nowhere to report the miss — **measured next door as 1 name of 24 hidden with the
> root hash still verifying**… the browse and the resolve disagree and neither complains."* Theirs
> fails closed with `IncompleteWalk`.
> **One name in twenty-four, under a signature that verified.** That is the same failure, one layer
> over, and it is evidence the concern is not theoretical.
>
> **They then sharpened the L18 handling and were right to:** their walk fails closed *"because we
> chose to write our own rather than call upstream `collect_all_bindings` — that is an implementation
> choosing to satisfy a rule, not evidence the rule is right."* Correct. It corroborates that the
> failure mode is real; it says nothing about whether our rule is the right response. **The argument
> is DNSSEC's RRSIG-over-RRset and the corpus's own six instances, exactly as §2 has it.**
>
> **Third pass: the rule this note defended no longer exists.** Consumer rule 5 is deleted by the
> inline decision (§9.2), which makes incompleteness unrepresentable rather than forbidden. **The
> failure mode is designed out instead of ruled against** — which is the better outcome and is one of
> the four reasons inline won.

**T4 — Key compromise, and the one place IDENTITY genuinely touches this.** An attacker holding P's
key publishes `seq: 9999` pointing at themselves. `seq` does not help — they have the key. **That is
compromise recovery, which is EXTENSION-IDENTITY's job.** Stated precisely because it is the strongest
argument *against* this proposal's scoping and it does not survive contact: **IDENTITY is the recovery
mechanism for everything a peer's key signs, and that does not make everything a peer's key signs
IDENTITY's.** The same argument would move `published-root` into IDENTITY.

**T5 — Rotation, and the two-step that validates the layering.** `peer_id` **is** the key, so a
rotation produces a different `peer_id` and therefore a different transport-set by construction.
"Where is Alice now" is then honestly two lookups: `durable identity → current peer_id` (IDENTITY) then
`peer_id → transports` (this). **The two-step is real, each step is owned by the layer that has the
data, and the legacy plan collapsed them** — which is how the record ended up in the identity
namespace in the first place.

**T6 — Query privacy: correcting an overclaim from the first pass.** I wrote that an id-keyed lookup
*"cannot violate the privacy MUST."* True of the **letter** — §4.1 step 2 is about *names*, and no name
is transmitted. **False of the spirit:** asking a third party for P's transports discloses *that you
are looking for P*. DNS has the identical problem and it is why ODoH exists. **The honest statement:
this lookup is outside the v1 privacy MUST by construction, and it is not privacy-free.** Not a
blocker; a thing not to oversell, and it argues for preferring "ask a peer you are already talking
to" over "ask an index" where both work.

**T7 — What the record does not solve: the first hop.** You still need *somebody* to ask. The record
does not answer that and must not be sold as if it does. **What it does is make asking *anyone* safe**
— which is the operator's principle, and which is why the lookup can then be the cheapest available
mechanism instead of a trusted one.

**T8 — Cost of republication.** A mobile peer changing networks republishes: one small entity, one
signature, one `seq` bump. libp2p does exactly this at exactly this cadence. ✅

**T9 — Graceful degradation / no flag day.** A peer that never publishes a transport-set is reachable
exactly as it is today: by name (registry glue) or by prior contact. Nothing breaks, nothing is
required, and adoption is per-peer. ✅

**T10 — Does this make `binding.transports` redundant?** No, and it should not be removed. Glue is
how you reach a peer you resolved **by name**, and §6a.3's non-empty `[MUST]` is still right for a
static publisher with no live surface. **What changes is that glue stops being the only authenticated
path:** once a set exists, a consumer can check the registry's claimed profile hashes against the
peer's own signed set — the registry stops being the party that vouches for where P is.

**T11 — the thing that is actually missing: rendezvous cannot introduce a *stranger* to a *specific*
peer, and that is the same gap this proposal exists to close.** `[fourth pass]`

A negotiated peer is reached by meeting it at a rendezvous key. `EXTENSION-SIGNALING` §3.2 pins four
modes, and **not one of them can be derived by a party holding only P's `peer_id`:**

| mode | input | why it does not reach P |
|---|---|---|
| `pair` | **both** peer-ids, sorted | **the stranger CAN derive the key** — see below. **P cannot be standing at it** |
| `tag` | an agreed public label | there is no agreed label — that is what "stranger" means |
| `secret` | an agreed high-entropy string | same, plus it is an admission gate by design |
| `lobby` | a fixed constant, `lobby:default` | untargeted — *"connect me to anyone here"*, not *"connect me to P"* |

> **The `pair` row's reason is corrected, and the correction strengthens the case.**
> `[entity-browser-rust`, `8cd3010`, measured against all three upstream establishers]` The fourth
> pass wrote *"P must already know the stranger to be standing at that bucket"* — which reads as a
> **derivation** failure. It is not: `pair_key` **sorts its two arguments**, and §3.2 says outright
> *"public — **identity *is* the key**."* A stranger holding P's id holds **both** inputs and derives
> the bucket correctly. **Nothing is secret and nothing fails on the querier's side.**
> **The real deficiency is responder-side enumerability:** P would have to stand at **one bucket per
> possible counterpart**, and that set is unbounded. **That is precisely the property a listening
> socket has and a rendezvous bucket lacks** — a listener accepts from *anyone* at *one* address;
> presence accepts from *one* counterpart per bucket. **So the peer-id-keyed fifth mode is the
> minimal fix rather than a convenient one**: it is the single bucket that restores the one-address
> property, and no combination of the existing four gets there.
> **Why the wrong reason mattered:** a derivation failure would have argued for a *key-exchange*
> answer (get them an agreed label somehow). An enumerability failure argues for a *bucket-identity*
> answer. **Same gap, and it would have sent the fix in the wrong direction.**

**So `reach_keeper`-style presence is real, but scoped to counterparts already known.** Their own
module says it: *"so peers that **have met** can actually connect."*

**This is R1 wearing the other substrate's hat.** R1 is *"self-authenticating against `peer_id` alone
— the querier holds only P."* This proposal answers it for **dialable** peers. **Rendezvous does not
answer it for negotiated peers**, and the two are one gap on two substrates — which is why the answer
belongs in this record rather than in a second record type (§11).

**Second half — and it is not the rendezvous's problem, it is every substrate's.**
`[generalized from `entity-browser-rust` `8cd3010`, who found the third instance]`

§3.4 carries a `MUST — same provider`: *"Both peers of a handshake meet at the same provider… A
provider mismatch is a silent never-meet."* The pool comes from `EXTENSION-REGISTRY` §3b.1's
`services` field — **which arrives on a `ResolutionResult`, i.e. only if you resolved a *name*.**

**browser-rust then found the same hole in RELAY Mode S** — *"the querier must learn which relay we
poll, exactly as they must learn which signaling pool we use. One field shape, two extensions."*
**It is three, and `EXTENSION-RELAY` §3.5 states its own instance outright:**

> *"A peer with no registry presence and no prior relationship is **reachable by a stranger only via a
> cached copy** — honest parity with a mail server that has no DNS entry."*

**The parity is not honest, and that sentence is where to see it.** A mail server with no DNS entry
has **no identity you can hold either** — there is nothing to look up. Here you hold the `peer_id`, a
complete cryptographic identity that self-certifies every answer, **and there is still no lookup.**

| Substrate | The intermediary | How you learn it today | From a `peer_id` alone? |
|---|---|---|---|
| **Live direct** — dial a profile | none | `system/peer/transport/*` from prior contact, or `binding.transports` by **name** | ❌ **R-22 — this proposal** |
| **Live mediated** — rendezvous / WebRTC | the **signaling pool** | REGISTRY §3b.1 `services`, on a `ResolutionResult` | ❌ **name-keyed** |
| **Store-and-forward** — RELAY Mode S | the **inbox-relay** | REGISTRY, *"the A+MX-in-one-zone pattern"*; else a cached copy | ❌ **§3.5 says so** |

**One gap, three substrates, and every one of them is answered by a peer-id-keyed, self-authenticating
record that anyone may serve — which is precisely what §§1–4 specify.** That reframes this proposal:
it is not *"the A-record gap."* **It is the peer-id-keyed answer for every way a peer can be reached**,
and the transport-set is where the locators belong because it is the only artifact keyed the right way.

**Neither half needs a new record type.** One is a fifth rendezvous mode; the rest are fields. §11.

---

## §7 The extension boundary, shown rather than asserted

The operator's challenge — *"even the `system/peer/transport`, you're saying it's a network"* — is
fair, and the first pass asserted it. By elimination:

| Candidate | Verdict |
|---|---|
| **V7 core** | No. Transports are pluggable and extension-shaped; the wire core is locked and does not grow a transport registry |
| **EXTENSION-REGISTRY** | No. REGISTRY's subject is *names*. This has no name in it — that is the entire point |
| **EXTENSION-DISCOVERY** | No. Its subject is *peers you do not know exist*; here you know exactly which peer you want |
| **EXTENSION-IDENTITY** | No — §6 T4/T5 |
| **A new extension** | Possible, and rejected for now: there is no second consumer and no second concern. One record does not earn a namespace |
| **EXTENSION-NETWORK** | **Yes.** It already defines `system/peer/transport/*`, §6.5.1a already governs selection over exactly this set, and §10's dispatcher is the consumer |

**And REGISTRY exposing it is orthogonal to NETWORK defining it.** That split already exists and is
not being invented here: REGISTRY does not define what goes in `binding.transports` — **NETWORK
does** — and REGISTRY carries the lookup machinery. Same division, one field over.

---

## §8 Deltas

> **D8 / D8a / D8b are FOLDED — `EXTENSION-REGISTRY` 1.20 → 1.21, 2026-08-21.** They were cut ahead
> of the rest of this proposal because they close a **live cross-impl interop break** (a Go peer
> could not decode a single binding in the cohort's only federation), not because the transport-set
> design settled. **The remaining deltas are unaffected and still DRAFT.** Two conformance vectors
> landed with them — `REG-BINDING-TRANSPORTS-SHAPE-1` and `REG-PUBLISH-CLOSURE-1` (§11.1) — so the
> MUSTs ship with instruments that read them, per L17.

| # | File | § | Delta |
|---|---|---|---|
| D1 | `EXTENSION-NETWORK` | §6.5.1c (new) | `system/peer/transport-set` per §4 — type, signature carriage, seven consumer MUSTs, the optional member-signature MAY |
| D2 | `EXTENSION-NETWORK` | §6.5.1a | State the **positional-authority** rule: a profile read **at its path in the peer's namespace**, **walked from that peer's signed root**, or **covered by a signed transport-set** carries the peer's authority; obtained any other way it carries none. Today the inner `peer_id` reads as authoritative and is not |
| D3 | `EXTENSION-NETWORK` | §6.5.1a | *"Absence falls through to other discovery"* falls through to nothing. Name the successor or state that a consumer holding only an id fails closed |
| D4 | `EXTENSION-NETWORK` | §6.5.4 | Withdraw *"Post-corridor: the EXTENSION-REGISTRY.md extension will define peer-ID → endpoint-set resolution."* NETWORK defines the record |
| D5 | `EXTENSION-NETWORK` | §6.7 | Two things. **(a)** Record **R7** as a rule: a peer **SHOULD NOT** publish, in a signed set, a profile for an address it has not confirmed dialable; §6.7.2 is the confirmation. **(b) Delete Amendment 13's build-state paragraph** — *"`check-reachability` remains unbuilt"*, the peer-reported 07-31/08-07 measurements, the retraction note. **All of it is dated cohort state inside normative text, it is now stale (§6 T1 measured it), and it nearly became a sequencing constraint on this design.** Nineteen references of exactly this class came out of `EXTENSION-REGISTRY` on 2026-08-17; NETWORK's `impl-team-ref` count is **20**, the highest in the corpus |
| D6 | `EXTENSION-REGISTRY` | §12 | Withdraw *"Lives in EXTENSION-IDENTITY amendment, separately authored."* No such amendment exists and IDENTITY is not the owner |
| D7 | `EXTENSION-REGISTRY` | §2.3 | Narrow to what it argues — the **fallback loop** re-resolves the original name. Delete the widening to *"`:resolve` is name-keyed by contract"* and the scoping clause, which cites §12 while §12 cites it |
| D8 **FOLDED v1.21** | `EXTENSION-REGISTRY` | §3 | **R-24 — `transports` is `[system/hash]`, bare, naming `system/peer/transport/*` profile entities.** Two edits, and the second is the one that was missing: **(i)** replace the prose type `[<endpoint per NETWORK §6.5>]`, which resolves three ways in this corpus — a §6.5.1 field holding a transport-specific object, a §3b URI string, and this; **(ii)** add `transports` to §3's **Hash-field shape** enumeration, which today lists `supersedes`, `issuer_attestation` and `revocation.revokes` and **omits the field in dispute**. *(ii) is not tidying — it is the sentence a careful reader used to conclude the opposite,* and leaving it unfixed reproduces the divergence one revision later. **Derivation** (cohort artifacts cited last, L18): REGISTRY does not define transport shapes, NETWORK does (§7 above); the inline form is **type-stripped**, which deletes the entity-type that §6.5.1a **D5 makes authoritative** over `transport_type` and requires decoders to fail closed on; §6.3 calls `transports` *"a cached hint, not the binding's substance"*; and content-addressing dedups one profile across N names. **Conformance (L19/L17):** a **`validate-peer` behavioral check** — a registry peer serving a `peer-issued` binding, driven by an oracle-authored resolve; the observable is the decoded `transports` element's CBOR major type (byte string, not map). Arch authors the case, the cohort builds the peer side, oracle cross-blesses. **Ruled 2026-08-21** on `entity-workbench-go`'s cross-impl measurement; see `COHORT-OPEN-ITEMS` §1f for the four-seat table and the argument against |
| D8a **FOLDED v1.21** | `EXTENSION-REGISTRY` | §3 / §6a.3 | **A registry MUST serve the closure its bindings reference `[MUST]`** — the `system/peer/transport/*` profile entities named by `transports`, and the §5.2 `system/signature/{hex(binding_hash)}` invariant pointers. **D8 is not landable without this.** A bare hash names an entity the consumer must fetch, and nothing in §3 or §6a says the registry holds it; §6a.3's MUST exists precisely because *"profile discovery is out-of-band in v1, so for a statically published peer there is nothing to find"* — so a `transports` array of unfetchable references satisfies the MUST in letter and delivers nothing. **`EXTENSION-NETWORK` §6.5.2 already states the publisher's form of this obligation** (*"MUST upload … the transitive hash-linked closure … plus the `published-root` entity itself and its `system/signature` entity"*); REGISTRY has no counterpart, and a grep of the file for `closure` / `MUST serve` / `MUST publish` returns nothing. **This is L17's ask discharged at ruling time, not after:** the value now has a site a peer can carry *and* a way for a peer to reach it |
| D8b **FOLDED v1.21** | `EXTENSION-REGISTRY` | §6a.3a | **The recommended publishing prefix is too narrow and a registry that follows it does not work.** §6a.3a says a browsable registry **SHOULD** publish at `prefix: "system/registry/binding/by-name/"`. A binding's signature lives at **`system/signature/{hex(binding_hash)}`** — V7 §5.2's invariant pointer, which §3 requires and §6a.4 step 3 fetches — and that path is **outside every `system/registry/…` prefix**. Follow the SHOULD and enumeration succeeds while **every resolve 404s at step 3**, which at a consumer is byte-identical to a withholding origin — the absent-vs-withheld collapse this charter has now recorded seven times. Widen to a prefix covering both, or state the pair explicitly. **Filed and measured by `entity-workbench-go`** (`publish/registry_roundtrip_test.go::TestNarrowRegistryPrefixOmitsTheSignatures`, with a control) and **confirmed by arch against §6a.3a's own text**. It is D8a's gap arriving one path over, which is why they land together |
| D9 | `EXTENSION-TREE` | §3.3a | Cross-reference: `transport-set` uses the same signature carriage and the same `published_at`-is-not-freshness rule. One convention, stated once, cited twice |

| D10 | `EXTENSION-NETWORK` | §6.5.2d | **The *"Who publishes it"* bullet carries an expired lean and should stop.** It records *"whether a **browser** peer publishes one… is open"* and points at `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` open item #1. That lean rested on *"they can't meet at a rendezvous when the browser is offline"*, and `entity-browser-rust` shipped the standing-presence half on **2026-08-16** (`c5e16ef`). **A browser peer MAY publish a `webrtc` profile**, and the residual precondition is T11's, not the profile's. Same `impl-team-ref` class as D5 — a dated cohort lean inside normative text, re-read as a rule (§0.2) |
| **D12** | `EXTENSION-NETWORK` | §6.5.3 / §6.5.4 | **R-28 — a consumer MUST NOT synthesize a `content_layout` it was not given, and a surface whose inputs cannot build a URL refuses rather than guesses.** §6.5.3 pins `content_layout` as a closed enum (`flat` / `sharded-2-flat` / `sharded-2-4` / `sharded-2-2`) and §6.5.4 makes profile discovery **out-of-band in v1** — so a consumer holding only `(origin, peer_id)` has **no conformant basis to guess**, and the corpus never says so. **Measured cross-impl 2026-08-20:** `entity-core-go`'s live origin serves **flat** `content/{hex}`; `entity-browser-rust`'s `name pin <pid> <origin>` fallback assumes **`sharded-2-4`**; **every blob 404s** before the walk reaches the closure — **both seats conformant, both values legal.** The refusal clause is the load-bearing half: **a wrong layout guess and a withholding origin produce byte-identical observations at the consumer**, so a guess does not merely fail, it **fails as the wrong diagnosis** — the third time this corpus has paid for that indistinguishability (the phantom manifest; §6.5.6's quiet-publisher-vs-withholding-origin pair). *Filed by `entity-browser-rust` (`059f0e1` §3b) explicitly as **"not an ask"**; arch is taking it as one, because a `MAY` whose two readings diverge across a peer boundary is the class this corpus leans MUST on, and a seat scoping its own finding to the half it owns is how a cross-impl divergence gets filed as one seat's UI gap.* **Third consumer of this proposal's gap, not new architecture** — it joins §11's rendezvous / signaling-pool / inbox-relay set as a fourth thing a `peer_id` alone cannot reach |
| D11 | `EXTENSION-NETWORK` | §6.5.1a (D1, Self-publication) | **Split the two properties the sentence conflates.** *"A peer may be reachable only via REGISTRY / manifest / out-of-band (e.g. a browser peer that cannot self-host its tree)"* runs a **storage** fact and a **reachability** fact together in one parenthetical, and that conflation is exactly what the third pass reproduced. Tree-hosting and dialability are independent axes; a browser peer holds a full tree (open item #1 says so explicitly) and its reachability question is about **negotiation vs. dial**, not about hosting |

**D6–D8 correct text that is wrong or loose today; D10–D11 correct text that actively teaches the
false inference §0.2 walked into.** **D1–D5 and D9 are the design.**

---

## §9 The five questions — all decided in the third pass

1. **R7 as a `SHOULD` or a `MUST` — DECIDED: `SHOULD`, satisfaction mode in-process per seat.**
   ~~Sequencing against §6.7.2~~ closed by measurement (§6 T1): the confirmation is built. `MUST NOT
   publish an unconfirmed address` is unenforceable by a conformance client — it cannot see what the
   publisher confirmed — which by `GUIDE-CONFORMANCE` §5.2b.1 and **L19** makes it a `SHOULD` with a
   declared satisfaction mode. **`entity-browser-rust` already implements exactly this shape and it
   is the confirmation the derivation wanted:** their `connect_peer` publishes a transport profile
   **on the 200, never on issue** — *"an address that never connected must not become a route the
   ladder spends a dial timeout on."* In-process, invisible to any conformance client, and correct.
2. **`profiles` by hash, or inline? — DECIDED: INLINE.** `entity-browser-rust` measured it and the
   second pass's own framing does not survive:
   - **The cross-peer dedup is zero by construction, and rule 6 is what makes it zero.** A profile
     carries its `peer_id` three times — the field, plus both URL prefixes built from it — so two
     peers' profiles can never be byte-identical and can never share a content hash. **Re-verified
     here in their tree: 615 bytes, peer-id present three times.** *The proposal forbade the dedup
     the hash option was half-justified by.* The dedup that **does** survive — the same peer's
     unchanged profile across republishes — **inline gets too**, because the set's own content hash
     is unchanged.
   - **Hashes do not permit lazy fetching, which is the argument that ought to have saved them.**
     D1 orders on `priority` and `profile-id`, and **both live inside the members**, so a consumer
     cannot order the set without fetching all of it. *N extra round trips deferring nothing.* The
     only escape — hoisting `priority` and `profile-id` beside each hash — inlines the record in all
     but name while keeping the fetch.
   - **Measured cost:** three-profile set ≈ **1.8 KB in one fetch** vs ≈ **370 bytes + 3 round
     trips**, on the record defined by being read *before you can talk to anyone*. In a browser that
     is three connections to an origin not yet spoken to.
   - **It deletes consumer rule 5 and an entire failure mode** (§4).
   - **And DNS is the third data point — our own analogy correcting itself.** SRV is the *indirected*
     design and its follow-up A/AAAA lookup is its best-known operational cost; **RFC 9460's
     SVCB/HTTPS `ipv4hint`/`ipv6hint` exist precisely to inline the address and remove that round
     trip.** Three of three, and the third is not a design preference — it is **a documented IETF
     correction of the indirected design, on this exact record type, for this exact reason.**
3. **Does a set with zero profiles mean anything? — DECIDED: YES. `[fourth pass: valid, but NOT the
   primary case — that half is withdrawn]`** ~~for every browser peer in the ecosystem it is **the
   whole record**~~ — **false, and §0.2 is why.** A browser peer publishes an endpoint-less
   §6.5.2d-shaped profile; **an empty set from a peer running a presence daemon is a signed false
   statement.** The empty set is for a peer that is genuinely unreachable — real, and rarer than the
   third pass claimed. The *decision* is unchanged and the *frequency argument under it* is deleted.
   A signed empty set means *"I am not directly dialable, and here is that fact signed by me."* It is
   strictly better than the status quo, where a bare peer-id fails as *"no transport profile for
   peer"* and a user reads that as *"that peer is down"* — **the absent-vs-unreachable distinction,
   which is this corpus's recurring seam wearing one more hat.** Consumer rule 8.
4. **Unsolicited / gossiped sets — DECIDED: permitted, with one rule that is not optional.** By §1
   there is nothing to protect: a signed record is valid whoever hands it over. **But the `seq` floor
   MUST be keyed on `peer_id` alone, never on `(peer_id, source)`** — otherwise a gossiping party
   gets a fresh floor of zero simply by being a second source, and the rollback defense evaporates at
   exactly the moment gossip makes it matter. *(`entity-browser-rust` already ships this discipline
   for publishers — "keyed on the publisher, never on `(publisher, origin)`" — and raised it here.)*
5. **`predecessor` — DECIDED: keep, as a non-normative convenience, and do not let it stand in for
   the property.** `entity-browser-rust` shipped a bug this would not have caught: every CLI publish
   emitted `seq 0` forever, because the emitter built a fresh in-memory publisher per invocation and
   had no prior head to chain from — **two different trees, same key, both `seq 0`, every signature
   and hash verifying.** A `predecessor` would have been *absent* in both, which is a legal first
   publish, so the chain would have looked fine while the floor was inert. **The fix was
   publisher-side state, not a field.** Continuity is a publisher discipline no consumer can check
   from one record, and the field must not be documented as though it were the check.

---

## §11 The residue, re-scoped — two additions, not a new record type

**The third pass filed this as *"mediated reachability has no record — it is MX-shaped, copy
`system/peer/inbox-relay`."* All three parts of that are wrong, and the correction makes the work
much smaller.**

**(a) `inbox-relay` is the wrong template, and it is wrong in the direction the spec has already
paid to fix.** `EXTENSION-RELAY` §3.5 defines it as where to deliver *"when the peer is
**unreachable**"* — the **offline store-and-forward mailbox**, RELAY Mode-S. A browser peer standing
at a rendezvous is **online and live-reachable through an intermediary**, which is a different rung.
`EXTENSION-NETWORK` **Amendment 14** exists precisely because *"a NAT'd peer… falls straight to the
store-and-forward terminal **even when it is reachable right now** by traversal,"* and it makes the
ordering a **MUST — live first, store-and-forward last.** **Modelling a live browser peer on the MX
record is the exact defect Amendment 14 was written to eliminate.**

**(b) The live-mediated shape is not missing.** §6.5.2d is a durable, endpoint-less profile that
advertises reachability-by-negotiation, and §10.3's `establish_live` seam is the consumer. §0.2.

**(c) A separate record would re-open the seam the inline decision just closed.** Two sets means a
consumer cannot tell *"P has no mediated record"* from *"I was handed only one of the two"* — the
absent-vs-withheld collapse, **seventh instance**, reintroduced one field over from where §9.2
designed it out. The DNS `A`-vs-`MX` precedent does not carry, because those are *different
services*; direct and mediated reachability are **one service, two substrates**, and DNS puts those in
one RRset.

**What is genuinely owed (T11), in full:**

| | The gap | The shape | Cost |
|---|---|---|---|
| **1** | **No rendezvous bucket exists that P can stand at for *anyone*.** Not a derivation failure — `pair_key` sorts, so a stranger derives it fine; P cannot be at one bucket per possible counterpart (T11) | A **fifth `EXTENSION-SIGNALING` §3.2 mode** — `peer`, `canonical(mode_input)` = P's canonical Base58 `system/peer-id` (§3.2's third sub-pin applies verbatim). **The single bucket that restores a listener's one-address property.** §3 licenses it: *"adding a mode is picking an input, **not adding a mechanism**"* | one table row + one vector |
| **2** | **The intermediary locator, for all three substrates** — signaling pool (§3.4's same-provider MUST), inbox-relay (RELAY §3.5), and any future `via` | **One field shape on the transport-set**, carrying the locator per substrate. Not three designs: the record is already the peer-id-keyed, self-authenticating, anyone-may-serve artifact, and each locator is a small typed member | one field, three consumers |

**All of it composes into *this* record, which is the point.** A browser peer's honest transport-set is
then **non-empty**: an endpoint-less §6.5.2d-shaped profile, plus the locators, plus the `peer`-mode
declaration — signed, publishable, and **derivable by a stranger holding only the peer-id, which is
R1.** One record, one signature, one completeness claim.

**Design constraint, stated now because it is the trap:** the locators are **where to find an
intermediary**, never a claim about what that intermediary will do. §1's principle holds — an
intermediary carries bytes and gets no authority — and a locator that grows into a capability, a
credential, or a policy has left this record's job. (`EXTENSION-REGISTRY` §3b.0a already records the
credential-channel gap for `data_relay`; **it stays there and does not migrate here.**)

**Three things to carry into that work rather than discover in it:**

- **It MUST be opt-in.** A peer that does not want to be stranger-reachable publishes no declaration
  and stands at no bucket. Same posture as T9 — nothing breaks, adoption is per-peer.
- **Presence at a peer-id-keyed bucket discloses that P is online** to anyone holding P's peer-id.
  That is T6 pointed the other way and it is the honest cost of being stranger-reachable. Name it; do
  not sell the mode as free. (`pair` already has this property once the counterpart is known.)
- **The bucket is public, so it is a DoS surface** — anyone may deposit. §4's service limits and §5's
  bucket semantics are the existing bound; confirm they hold at this shape rather than assuming.
- **It stays inside §1's principle:** the key introduces, it never authorizes (§1.2/§3.4). An
  intermediary carries bytes and gets no authority — `nonce_required`, the §6.6 held capability, and
  end-to-end envelope signatures are unchanged. **This is why a mediated endpoint needs no new trust
  machinery**, and it is the same argument that dissolved *"who may serve it?"* in §1.

**R-10 stays next, and now for the right reason.** The third pass put
`PROPOSAL-DISCOVERY-RENDEZVOUS-BACKEND` on the critical path because *"the content is likely 'my
rendezvous is node N'."* The real reason is item 2: **the pool locator is the rendezvous backend's
subject**, and `entity-workbench-go` is blocked on the fold today. Right conclusion, wrong derivation
— recorded because a conclusion that survives its own justification being wrong is the hardest kind to
notice (**L8's fourteenth form**).

**R-25 is rewritten to this scope**, and **R4 is no longer part of it.**

## §10 Validation this needs before it is ruled

**This is a design that has been stress-tested, not one that has been shown to work.**

- ~~**`entity-browser-rust`**~~ — **ANSWERED, `5c84154`.** §9.2 inline, measured; R4 withdrawn as
  not-discharged (§11); §9.1/§9.4/§9.5 answered from shipped implementations. Both §10 assignments
  discharged and both changed the proposal. Their gates at that commit: `make test` **1192/0/4**
  across 15 binaries · `make lint` clean · `make wasm` fresh · `make e2e-worker` **19/0** ·
  `make federation` 5/5 `--verify` clean. **They are ready to implement when it is ruled.**
  **`[fourth pass — the caveat is withdrawn]`** This bullet said *"for their own peer the set will be
  empty until mediated reachability exists."* **It will not be empty** (§0.2): their peer publishes an
  endpoint-less §6.5.2d-shaped profile. **Two questions go back to them, and they are the ones that
  changed:** (i) does `reach_keeper`'s presence hold long enough to make a *published* advertisement
  honest under R7 — i.e. what is the publish/withdraw trigger, given their `connect_peer` already
  publishes on the 200 and never on issue; (ii) confirm the T11 table against their establishers —
  **is there any way today for a party holding only your peer-id to reach you**, or is `pair`-mode's
  both-ids requirement the hard floor we read it as? **Their finding stands and it improved the
  design; the inference drawn from it here was arch's, not theirs** (L10).
  **— ANSWERED `8cd3010`, and both answers changed the proposal.** *Q2:* confirmed, **reason
  corrected** — the failure is responder-side enumerability, not derivation (T11's note). *Q1:*
  exposed a live bug in their tree — `ReachKeeper::forget` had **zero production callers**, so a
  dismissed peer drew a probe every ~30s forever; fixed remote-scoped as `forget_remote`. **Their
  framing is the durable half: *"presence is this peer form's substitute for a listening socket — an
  assertion to a third party — so an unwithdrawable one is a standing invitation to somebody the user
  forgot."*** That is R7's withdraw side, and it belongs in the fold: **a presence-backed
  advertisement needs a withdraw path, and the other three teardowns are bookkeeping.**
  **And one finding nobody asked for, which is the best thing in the packet:** RELAY Mode S fails the
  stranger on the *same* missing piece, which is what generalized T11 to three substrates (§6 T11).
- **The core tier — OPEN, and this is the round to run.** T1/R7 is theirs: whether a NAT'd engine
  peer can populate a set honestly today via §6.7.2, and what the `SHOULD` costs in practice. Also
  the inline decision's wire consequence — a member type-ref inside a set (§4's parenthetical) is a
  type-system question go/rust/py answer better than arch does.
- **`entity-workbench-go` — OPEN.** They hold the only non-browser consumer of foreign published
  state and would be the first to fetch one of these from a third party. **R-10's rendezvous fold is
  theirs-adjacent and is now on §11's critical path.**

If any seat has already built something in this shape, **it is corroboration and it is cited last**
(L18) — §§1–3 stand or fall on the corpus and the prior art, not on a cohort implementation.
