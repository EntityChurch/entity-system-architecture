# EXPLORATION — the local-view problem: who answered, and why our namespace differs

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-raised, 2026-09-06 (fifth pass), and it found something the previous four missed:**

> *"The peer in the path tells you who the authority is, but my local tree can have data on a peer path.
> It's only as good as your ability to contact that peer live… another peer that's not the one you're
> looking for can theoretically answer, but you don't know if it's the most up to date without contacting
> the actual peer with the live ability to sign, holding the key. Our addressing and our universal
> namespace has slightly different properties than you see in some of these other tools."*

**The claim of difference is correct, and it is sharper than "slightly."** §2 states it.

---

## §0 The result

**Three findings, and the second is the one no previous document in this series reached.**

**1 · The corpus has already solved the freshness half, precisely, and better than the surveyed field.**
`EXTENSION-TREE` §3.3a: *"`published_at` is a signed lower bound on the artifact's age, and MUST NOT be
read as evidence of origin freshness… a publisher that has not republished and an origin withholding a
newer root produce byte-identical results at the consumer — same signature, same `seq`, same
`published_at`. What a verified root establishes is **authenticity and non-rollback as of the root
received**."* `GUIDE-SERVING-MODE` §8 turns that into three UI states and forbids a bare *"Verified"*
without a date. **This is a stronger and more honest statement than Nostr, which resolves competing
replaceable events by `created_at` — an author-controlled clock — and says nothing about the limit.**

**2 · But that machinery covers a publisher's own signed root. It does not cover the case the operator
named, and that case is architecturally sanctioned.** `EXTENSION-NETWORK` §6.5.6, verbatim:

> *"A serving peer publishes a scoped projection of its **local view of the universal tree**: paths under
> `/{any_pid}/…` it holds — **its own authoritative namespace and any cached/mirrored remote
> namespaces**."*

**So a third peer really can answer `TREE_GET /{alice}/blog/post`.** And per Amendment 6 that route
returns *"the bound content hash as a `system/hash` value"* — **a bare two-key entity, unsigned.**

> **A tree-path binding resolved through a third party's local view arrives, ON THAT ROUTE, as an
> unsigned claim by that third party about someone else's namespace.** The bytes it eventually leads to
> are self-verifying against the hash; **the binding as delivered is not.** Bob cannot forge Alice's
> *content*, but nothing stops Bob's local view from binding `/{alice}/blog/post` to a *different* hash
> of Alice's — an older post, a retracted one, or any other entity he holds.

**Say "as delivered," not "unsigned" — the distinction is the whole finding** (§0.3). Alice's signature
over that binding is constructible, canonically addressed, and in the common case **already sitting in
the served set**; the `TREE_GET` leaf route simply does not hand it to you and nothing tells you to go
and get it.

**3 · The gap is CARRIAGE, not signature. The signing primitive is fully general and canonically
addressed; the one route that serves bindings to strangers carries neither the signature nor an
obligation to fetch one.** `[corrected 2026-09-06 — see §8. The first version of this finding said the
binding was "signed only in bulk, at the root," and that was false three ways.]`

`ENTITY-CORE-PROTOCOL` §3.5 defines `system/signature` as `{target, signer, algorithm, signature}` where
**`target` is any content hash and `signer` is any peer** — *"multiple peers signing the same content
produce signatures at parallel paths."* Any peer may sign anything. It is reachable at the **invariant
pointer `/{signer}/system/signature/{target_hex}`**, which §3.5's *discovery-locality* rule names as
strategy (B), *"constructable with no extension knowledge."* And §3.5's property **(3) Forwardable** is
this document's case verbatim:

> *"when Carol asks Alice for Bob's data, **Alice can include Bob's signatures. Carol can verify Bob's
> authority without contacting Bob.**"*

**Carriage is the envelope** — `{root, included}` plus target-matching — and it is already used for
exactly this: `EXTENSION-TREE` §3.3a puts the `published-root`'s signature entity *in the `MANIFEST_GET`
envelope's `included`*, and `EXTENSION-NETWORK` §6.5.6's Amendment-10 closure MUST makes a
`signed_pointer` publisher **serve** that signature at the invariant pointer along with every trie node.

**"In bulk" was also wrong about the root signature.** `EXTENSION-TREE` §3.7's math contract: *"each
binding the receiver gets is verifiable via **inclusion proof against the source snapshot root**
(per-binding math, preserved)."* The signature is produced once over the root; **verification is
per-binding.** One signature plus a trie path is a per-binding authority claim.

**And the corpus has already ruled this exact threat, as a MUST, one extension over.**
`EXTENSION-SUBSTITUTE` §7.2 on a `path_index` served by a third party: *"the `path_index` is an authority
claim about the publisher's tree; **anyone serving the URL can forge it**, so hash-verifying the manifest
against its own hash proves nothing about authorship"* — remedied by §4's **Signature MUST**: every entry
carries the source peer's signature at `system/signature/{hex(source.content_hash)}`, unsigned entries
*"are rejected at consultation time,"* with **no transitive trust** (*"peer B serving peer A's content
without A's signature is invalid"*). That is D-42's fix, already written, for a different route.

**So the departure from the field is narrower and more interesting than "ours is unsigned."** Nostr,
ATProto and SSB bind the signature to the *unit* — event, commit, log entry — so it cannot be separated
from it. We bind signatures to **content hashes at a parallel address**, which is strictly more general
(any peer, any target, multi-party, cacheable, forwardable) and costs one thing: **the signature is a
separate object, so every route has to decide whether to carry it — and one route decided not to.**

**The consequence for the reference atom, which is what this series is for:**

| | resolved via the authority | resolved via a third party's local view |
|---|---|---|
| **pinned `{peer, hash}`** | fine | **fine — the hash settles it.** A cache is exactly as good as the origin |
| **live `{peer, path}`** | fine — and it is the only way to get *current* | **an unsigned third-party claim**, unless you walk the authority's signed root |

> **That table is the answer to the operator's question, and it is also the argument the pin has been
> missing.** The pin is not merely *safer against a rug-pull* — **it is the only form that is safe to
> obtain from a stranger at all.** A live reference carries a resolution requirement that a pin does not,
> and nothing currently says so.

---

## §1 The four honest states, where the corpus names three

`GUIDE-SERVING-MODE` §8's three states — **Never checked · Failed · Verified as of {published_at}** — are
about following a `signed_pointer` to a publisher's own root. **Resolving a live reference has a fourth
state that sits outside them**, and it is the ordinary case when you did not get the answer from the
authority:

| # | How you got the binding | What is proved | What is not |
|---|---|---|---|
| **1** | **the authority, live** | it is current | — (this is the state) |
| **2** | the authority's **signed root**, from anywhere | **authenticity + non-rollback** as of that root | that it is newest — §3.3a's byte-identical case |
| **3** | **a third party's local view**, no signed root walked | **nothing about the binding.** The bytes hash-match whatever hash you were handed | that the authority ever bound that path to that hash |
| **4** | a pinned hash, from anywhere | **the content, fully** | nothing about *where* it sat — and it does not need to |

**State 3 is unnamed, reachable by default, and looks identical to state 2 at the consumer** unless the
consumer deliberately walks a signed root. **That is the finding.** It is L8's eighteenth form on a new
noun — *two mechanisms, one observable* — where the two mechanisms are *"the authority told me"* and
*"someone's cache told me"*, and the observable is a `system/hash` with no signature either way.

**Two mitigations already exist and neither is a rule.** `serve_scope` defaults to **`published-set`**
(SHOULD), which is namespace-scoped, and **`whole-store` is default-off and marked *"security-defective"*
for multi-party use** (`EXTENSION-CONTENT` §6.4.1). So the *common* deployment does not serve foreign
namespaces. **But the local view explicitly may**, §6.5.6 says so, and a consumer has no way to tell
which kind of peer it just asked.

---

## §2 Why the universal namespace makes this ours specifically

**In the surveyed systems, "who may answer" is a role.** A Nostr relay is a relay; an ATProto PDS is
authoritative and an appview is derived; the roles are distinct, named, and a client knows which it is
talking to.

**Here there is no role, because every peer is both.** `GUIDE-RESOLUTION` §1 (P2): *"A peer's tree holds
`/{its_own_id}/…` (authoritative) and `/{other_id}/…` (its cache of others)."* **Authority is therefore
not a property of the peer you are talking to — it is a property of the `(peer, path)` pair you are
asking about.** The same peer is authoritative for one path and a cache for the next.

> **So the reference names the authority, the fetch chooses the answerer, and nothing in either says
> whether they were the same party.** In a system with roles that question is answered by *which endpoint
> you dialled*. In ours it cannot be, by construction.

**This is not a defect of the universal namespace — it is the cost of its best property.** One address
space with local views is what makes caching, mirroring and offline reading fall out for free, and what
makes `EXTENSION-CONTENT`'s dedup work across peers. **The bill is that "authoritative" stops being
observable from the endpoint.**

**And it is the same distinction one more time.** `GUIDE-RESOLUTION` §5's split — self-verifying vs
authority-scoped — is *why* the pin is immune and the live reference is not. **Hop 2 (`hash → bytes`) is
self-verifying, so the answerer is irrelevant. Hop 1 (`path → hash`) is authority-scoped, so the answerer
is the whole question.** Fourth appearance of this distinction in five documents; it is the load-bearing
idea of the whole series.

---

## §3 What the field does, and what we actually lack

**Nothing on the mechanism axis. We have the strictly more general form of what every one of these
ships.** `[rewritten 2026-09-06 — the first version's "ours" column was scored against a claim §0.3
now retracts]`

| System | Third-party answer is trustworthy because | Ours |
|---|---|---|
| Nostr | **every event is individually signed** by its author | ✓ **and detachable** — `system/signature` targets any content hash, from any signer, at a canonical parallel path (V7 §3.5) |
| ATProto | records sit under a **signed repo commit** with a monotonic `rev` | ✓ `published-root` + `seq`, **plus per-binding inclusion proof** against the root (TREE §3.7) |
| SSB | append-only **signed log** with sequence numbers | ✓ analogue: `seq` + `predecessor`, rollback-reject MUST |
| IPNS | mutable pointer with **signed sequence number + validity window** | ✓ analogue, same shape |
| Nix | store paths are **immutable**; the `narinfo` is signed | ✓ — this is our pin, exactly |

**The one real difference is a design choice, and it cuts both ways.** In all four signed-unit systems
the signature is **inseparable from the unit** — you cannot serve a Nostr event without its `sig`. Ours
is a **separate entity at a parallel address**, which buys multi-party signing, caching, dedup, and
third-party forwarding (§3.5's four properties) — and costs the one thing inseparability gives for free:
**every route must decide to carry it, and a route that forgets is silently unauthenticated.**

**So what we lack is not a mechanism and not a counter. It is two smaller things:**

1. **Carriage on one route.** Three of our four binding-bearing surfaces carry the authority's signature;
   the fourth does not, **by normative design**:

   | Surface | Carries the authority's signature? |
   |---|---|
   | EXECUTE / protocol envelope | ✓ `included` + target-matching (V7 §3.5) |
   | `MANIFEST_GET` | ✓ signature entity in the envelope, and in the served closure (TREE §3.3a; NETWORK Amendment 10) |
   | `SUBSTITUTE` `path_index` | ✓ **MUST**, verify-or-reject, no transitive trust (§4, §7.2) |
   | **`TREE_GET` leaf** | ✗ **bare 2-key `system/hash`** (NETWORK §6.5.3.1, Amendment 6) |

   **And the omission has a reason, which is why this is a design question and not an oversight.**
   §6.5.3.1 minimised that body deliberately: returning a fuller entity *"materialises a separate copy of
   the entity at every tree path bound to `H`, defeating the content-store dedup invariant."* **The
   thinnest possible object was the point.** Any carriage design has to respect that.

2. **A consumer obligation.** Under Amendment 10 everything needed is **already served** — the
   `published-root`, its signature at the invariant pointer, and every interior trie node. A consumer
   *can* verify a third party's binding today with no spec change at all. **Nothing says it must, and
   nothing tells it whether the peer that answered was the authority.**

---

## §3a Four carriage designs, scored — sketches, not proposals

**The operator's framing is the right one: this is a bundling question.** *"A lot of the time you may
be asking for the signature for specific caches from the peer, and a lot of times you want that to
travel with whatever it's signing."* The signed-root walk covers the bulk case well; it is not the only
case, and it is not a form a stranger can hand you in one object.

| | Sketch | New wire surface | Round trips (cold) | Respects §6.5.3.1 dedup | Write cost |
|---|---|---|---|---|---|
| **(a)** | **Proof suffix** — a third member of the existing append-one/strip-one suffix bijection, `{path}{tree_proof_suffix}` ⇒ an envelope `{root: the binding, included: {published-root, its signature, the trie path nodes}}` | one suffix, no new type | **1** | ✓ leaf body untouched; the proof is a separate object key | none — precomputable at publish |
| **(b)** | **Consumer obligation only** — resolving from a non-authority MUST walk that authority's signed root, or surface the result as state 3 | **none** | 1 + manifest + ~log₂(n) interior nodes, all `immutable`-cacheable | ✓ | none |
| **(c)** | **Per-binding signed entity** — publisher signs `(path, hash, seq)` individually | new type | 2 | ✓ | **a signature per `tree:put`** |
| **(d)** | **Fatten the leaf body** to carry the signature inline | changes a MUST | 1 | ✗ | none |

**(d) is foreclosed** — §6.5.3.1 rejects a fatter leaf body by name, for dedup, and that reasoning is
sound. **(c) is very likely foreclosed too, by a ruling already on the books**: §6.5.6's
bounded-convergence MUST explicitly permits coalescing because *"a signature per `tree:put` is
write-amplifying and is not the intent."* (c) reintroduces exactly that cost. It is worth keeping only
if a case appears where a binding must be authenticated **without** the publisher having republished.

**(a) and (b) compose, and that is probably the answer: (b) is the rule, (a) is the fast path.** (b) is
free and available today — everything it needs is already served under Amendment 10 — so it can be
stated without waiting for anything. (a) then collapses its cost from *"a manifest plus a trie walk"* to
one GET, which is what makes the rule cheap enough to be honoured rather than skipped. **(a) also fits
the existing URL grammar with no new concepts**: §6.5.3.1 already defines a total suffix bijection over
any name, so a third suffix costs one field in the profile and one route in the demux.

**The open question inside (a)** is whether the proof object is the trie path or the whole envelope, and
that is a size question that should be measured rather than argued. **Not answered here.**

---

## §3b Why NO bundler in this system finds a signature — the structural cause

`[operator-raised 2026-09-06: "you pass the signatures around with all of the hashes, so you're bundling
it together. Does it become a specific type of handler extension that bundles the signature and that
together?" — the answer is that one already exists, for one entity class, and the reason it had to be
written specially is the finding.]`

**The signature model is directional, and the direction is the cause.** V7 §3.5: *"signatures point TO
the content they sign. **The signed entity does NOT reference the signature.**"* Content-addressing means
every bundler in this system assembles a transfer set by **walking references out of the root**. A
signature is never on that path — nothing points at it — so **a closure walk structurally cannot find
one.** Not "forgot to include"; *cannot reach.*

**Measured across all three bundlers in the corpus, and it holds in all three:**

| Bundler | What it walks | Signatures included? |
|---|---|---|
| **`tree:extract`** (TREE §6.2) | trie nodes reachable from the snapshot root + data entities at leaf bindings | **✗ none** — the algorithm has no third loop |
| **`.entsite`** (`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §6) | closure-complete over pages, embeds, assets, chunk closures | **✗ per-entity none** — carries a **`pin`**, the signed *site root*, instead |
| **cross-peer dispatch** (`EXTENSION-CONTINUATION` §4.3) | the EXECUTE's `data` refs | **✓ — but only because a special MUST was written for it** |

**CONTINUATION §4.3 states the cause outright, having hit it:**

> *"§4.3's general rule guarantees only the leaf cap, because parent/granter caps **and all per-link
> signature entities** are referenced from *within* the cap entities, not from the EXECUTE's `data`
> fields; cross-peer dispatch MUST additionally bundle the transitive chain and its signatures… **The
> safe default is to bundle the whole chain — content-addressing makes over-inclusion free** (any entity
> B already holds dedups by hash on ingest)."*

**That is the operator's HTTP instinct, already ruled, with the same rationale — and scoped to
capability chains only.** *"Over-inclusion is free because dedup"* is the general argument for bulk
transfer in a content-addressed system, and nothing about it is specific to capabilities.

**The consequence, and it is why this cannot be automatic.** V7 §3.5 names the only general way to find
a detached signature: *"the invariant pointer path constructed from `(signer, target)`."* **Construction
needs a `signer` — you must already know whose signature you expect.** So:

> **Bundling content is a CLOSURE operation and needs nothing but the root. Bundling signatures is an
> EXPECTATION operation and needs a set of expected signers.** They are different kinds of computation,
> which is why one falls out of content-addressing for free and the other never will.

**This is the architectural justification for the operator's read that it belongs above the substrate**
— *"it's kind of an operational choice at a higher level"* — and it is stronger than a preference. A
substrate bundler has no expectation to work from; the layer that holds one is the layer that knows what
it is reading and why. **`system/attestation`'s graph operations are the shape of the answer** (§5.4
`find_attestations_targeting` already does target-keyed lookup), not a new closure rule in TREE.

**And it explains the "who cares about signatures" mode without excusing it.** The operator's *"I just
want the content, I don't care who it came from"* is not merely tolerated by this design — **it is the
default that falls out of it**, because the unsigned path is the one that requires no expectation. That
is the right default for a substrate and the wrong default for a client reading strangers' data, which
is exactly the split §3's two lacks describe.

---

## §3c The line is MUTABILITY — and `PROPOSAL-APP-CONVENTION-FEED` §1.1 is the AUTHORITY for it

> **⚠ `PROPOSAL-APP-CONVENTION-FEED` §1.1 is the authority for this rule. What follows restates it and
> generalises it; the restatement is not independent evidence.** `[This section was derived fresh on
> 2026-09-06 from the operator's framing, and only then found already-derived, in an active proposal in
> this repo. **L7 — the toolkit was not searched before building**; `grep -rn "invariant pointer"
> docs/proposals/` returns it. Recorded rather than quietly absorbed, and the section rewritten to cite
> rather than restate — **L23's fourth shape: a restatement names its authority so the next sweep is a
> grep.**]`

**FEED §1.1 rules it normatively, and reaches a sharper formulation than this document did:**

> ***"An entry MUST be individually signed"*** *— a separate `system/signature` entity at the invariant
> pointer `/{author}/system/signature/{hex(entry_hash)}`… "The required one is the signature, and the
> argument is the same one that produced §1.2: **the unit of addressing should also be the unit of
> verification.** An entry that travels alone should verify alone, in `O(1)` extra objects, without its
> author's tree, without their origin, and without a root that may be many publishes stale."*

It carries the same two-mechanism split this section had independently reconstructed — **detached
signature (required) vs inclusion proof (optional)**, *"one small content-addressed entity, per entry"*
against *"a chain of trie nodes, proportional to tree depth, per entry"* — and it makes a point §3a
missed: **the inclusion proof is not redundant.** It answers the **anti-omission** question — *was this
in their published tree as of sequence N?* — which a signature cannot, so a mirror asserting *"and here
is proof they published it"* is making a stronger and different claim.

**It also carries the measured cost, from `entity-browser-rust` `62c6d62`, and this is the fact that
makes it a programme rather than an edit:**

> *"**Nothing signs individual entities today: a publish signs exactly one thing, the root.** So the
> detached per-entry signature is a **new obligation on the composer**, and a mirror cannot supply it —
> a mirror can only carry a signature the author already minted."*

**So the operator's *"you're going to need to sign your entries"* is already ruled — for feed entries,
by name, and nowhere else.** The two claims are not substitutes and neither subsumes the other:
authorship says nothing about *where* an entity sits or whether it is current; a root signature attests
that Alice's tree bound a path, **not that Alice authored the bytes it points at**. What this document
adds is the scope claim below.

### §3c.0 The scope finding — one convention has the rule and the other three have nothing

**Measured 2026-09-06. Region searched, named so the negative is reviewable: all four files in
`specs/applications/` (`EMBED`, `SHARE`, `SEMANTIC-CONTENT-SITE`, `CHARTER`) plus
`docs/proposals/active/applications/PROPOSAL-APP-CONVENTION-FEED.md`.**

| Convention | Status | Per-entity authorship signature | Anchors on |
|---|---|---|---|
| `PROPOSAL-APP-CONVENTION-FEED` §1.1 | **draft** | **MUST**, at the invariant pointer | the entry itself |
| `APP-CONVENTION-SEMANTIC-CONTENT-SITE` | **landed** | **none** | the signed `site-root` **pin** (§2), and `.entsite` carries `pin` + closure |
| `APP-CONVENTION-SHARE` | **landed** | **none** | `published-root` as *"the verification anchor"* (§ on singularity) |
| `APP-CONVENTION-EMBED` | **landed** | **none** | *"content-hash verify + manifest signature"* (§7 item 7) |

**`system/signature` occurs ZERO times in all three landed conventions.** So the rule the operator wants
is stated exactly once, in the one document that has not landed, and **every convention that has landed
anchors on a root or manifest signature** — the form §3c argues is the wrong instrument for immutable
authored content.

**This is `L21`'s fifth shape waiting to happen, on the same document class it fired on before.** A
vocabulary/obligation derived in one app convention, with no mechanism by which the seats reading the
other three would learn of it: an `APP-CONVENTION-*` has no version bump the cohort watches, no §9.1,
and no diff a peer can read. **When FEED §1.1 lands, the fold audience is every convention that anchors
on a root** — and by the fifth shape's enforcement point, that audience is found by grepping the cohort
for what those seats *currently emit*, not by what the fold says they should.

**So the line the operator asked for is mutability, not size and not convenience:**

- **Immutable authored content** — a forum post, a reply, a comment, a feed entry, a wiki revision.
  **Authorship is the claim that matters, it is the cheap one, and it is eternal.** Sign the entity.
  Once signed it forwards through any number of intermediaries at the cost of one extra entity, verifies
  with no trie walk, and **outlives the publisher** — the same property that makes the pin outlive it.
- **Mutable pointers** — a profile head, a site root, a feed head, "the current version of this page."
  **The claim is irreducibly about tree state at a time**, so it needs the signed root, the `seq` and
  the rollback rule. **The bulk form is not a shortcut here; it is the only correct shape**, because
  there is no per-entry fact that answers *"is this the latest?"*

> **Read that way, the bulk root is not "too broad" — it is being asked to do a job it was never the
> right instrument for.** It is doing double duty as an authorship proof because nothing else in the app
> tier provides one. **The fix is not to shrink it; it is to stop borrowing it.** Once entries carry
> their own signatures, the root goes back to being what the operator described — *"this is where my
> state was, these are the entries I had"* — a **completeness and currency** statement over a set whose
> members are independently authenticated. Both, composed, and each doing only its own job.

**And this is why the transfer cost inverts the way the operator noticed.** Under root-only, forwarding
one entry drags the publisher's whole-tree state; under per-entry signatures, forwarding the *whole set*
costs one signature per entry and each one is independently checkable by whoever ends up with it. **The
form that looks like the bulk optimisation is the one that forces bulk transfer.**

### §3c.1 Countersignature — what it asserts depends on which hash you target

`[operator: "I can have someone else say, I'll sign that they signed it too… I don't know what it
exactly tells you, but there's some stuff there that's useful."]`

**Because signatures are themselves content-addressed entities, there are two distinct moves here and
the corpus gets the second one for free:**

- **Bob signs the same `target` Alice signed** ⇒ a second, independent authorship/endorsement claim over
  **the content**. This is V7 §3.5's property (4) — *"multiple peers signing the same content produce
  signatures at parallel paths"* — already specified, already multi-party.
- **Bob signs `content_hash(Alice's signature entity)`** ⇒ a claim about **Alice's signature**, not about
  the content: *"this signature existed, and I saw it."* Different target, different proposition. **That
  is a witness/timestamp primitive**, and it is well-formed today with no new type, because a signature
  entity has a content hash like anything else.

**The second is the one worth naming**, because it is what gives an *ordering* story without a clock —
the gap TREE §3.3a is honest about (`published_at` is publisher-controlled) and that Nostr does not
close (`created_at` is author-controlled). A witness set is third-party evidence about *when* something
existed. **Not proposed here** — flagged because it is reachable at zero substrate cost and
`EXTENSION-ATTESTATION`'s supersedes/liveness graph is already the right home for it.

---

## §3d The root cause, and it is one sentence: YOU CAN ONLY SIGN AN ENTITY

**Everything above converges here.** `system/signature.target` is a **content hash**. A content hash is
the identity of an **entity** — `(type, data)`. Therefore **a thing is signable exactly when it is an
entity**, and nothing else in the system can be signed at all.

**A tree binding is not an entity.** `path → hash` is an *edge in a trie* (`ENTITY-CORE-PROTOCOL` §1.7).
It has no `(type, data)` of its own and therefore **no content hash and no signature slot.** That is not
a gap in `TREE_GET`'s route design — the route could not carry a signature over the binding, because
**there is no such signature to carry.**

**And this predicts, correctly, every workaround already in the corpus.** Each is the same move: *make
the relation an entity, then sign the entity.*

| Where a relation had to be authenticated | The entity that was minted for it | Signed at |
|---|---|---|
| a whole tree's path set | **`system/peer/published-root`** (commits to `root_hash`, `seq`, `prefix`) | the invariant pointer |
| a name → target binding | **`system/registry/binding`** | `system/signature/{hex(binding_hash)}` |
| a path set served by a third party | SUBSTITUTE's **manifest / `path_index`** | `system/signature/{hex(manifest_hash)}` |
| a site's root | the **`site-root` pin** | the standard signature ref |
| an entry's authorship | **the entry itself** (already an entity) | the invariant pointer, per FEED §1.1 |

**Five instances, one pattern, and nobody had named it.** The registry row is the sharpest: **a registry
binding is exactly a `name → target` relation, and it IS individually signed** — because REGISTRY made
it a first-class entity. **The tree binding is the same shape of relation and is the one that never
got an entity.**

> **So D-42 restates one final time, and this is the form to carry:** the tree is the system's only
> load-bearing relation that is *not* an entity, so it is the only one that cannot be signed in place.
> **Everything else about the local-view problem follows from that**, including why the signed root
> exists at all — *the root is the entity we mint so that bindings become signable in bulk, because
> individually they are not signable at all.*

**Which reframes §3a's four sketches.** They are not four ways to carry a signature; **they are two
ways to mint an entity and two ways to avoid needing one.** (a) the proof suffix and (b) the consumer
obligation both work by pointing at the **root** entity that already exists — no new entity, which is
why they compose and why they are cheap. (c) *is* the mint-an-entity option (a signed per-binding
entity), and its cost is exactly the cost of entity-hood: one object and one signature per binding,
which is the write amplification §6.5.6 already declined. **The design space is smaller and better
understood than "four sketches" suggested, and the question is now a single one:** *is a tree binding
worth making an entity, or is pointing at the root sufficient?* **§3c's answer is that it is not
worth it, because for immutable content the entity to sign already exists — it is the content — and for
mutable pointers the root is the correct instrument anyway.**

---

## §4 What this changes for the design

**Design C survives unchanged; two things are added and one earlier claim is qualified.**

1. **The `via` hint needs a kind, and now it has a derivation rather than an instinct.** The bake-off
   sketched `hint = {tag: "origin"/"mirror"/"peer", …}` by analogy to BitTorrent's `xs`/`as`/`ws`. **§0's
   table is the reason it is required**: for a pinned reference the hint kind is irrelevant (any source
   will do), and for a live reference **a mirror hint is nearly useless while an authority hint is the
   whole point.** One atom, two intents, and the hint semantics differ by intent.
2. **A resolution result needs to carry *who answered*.** Not a term of the reference — a term of the
   *result*, alongside the existing `FEED` §2.2.4 obligation that a reader be told when a resolved hash
   differed from `seen`. **Same rule, one step earlier**: *a view that names its own provenance is
   debuggable; one that does not is indistinguishable from a bug.*
3. **Qualified from the previous document:** it said *"anyone can answer a self-verifying reference, so it
   outlives its publisher"* — **true** — and, by symmetry, implied the live case merely *dies* with its
   publisher. **It is worse than dying: it can be answered wrongly by a third party, silently.** Dying is
   a `404` and is honest. This is not.

---

## §5 What this puts on the docket

**Four items — D-42 plus three from the 2026-09-06 bundling pass.** Everything else folds into D-37.

- **`D-43` — NO BUNDLER IN THIS SYSTEM FINDS A SIGNATURE, and it is structural, not an omission.**
  Signatures point *to* content and content never points *to* signatures (V7 §3.5), so every
  closure/ref-walk assembler is signature-blind **by construction**. Measured in all three: `tree:extract`
  (TREE §6.2 — trie nodes + leaf data entities, no third loop), `.entsite`
  (`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §6 — closure-complete over pages/embeds/assets, carries a root
  `pin` instead), and **cross-peer dispatch, which is the exception only because
  `EXTENSION-CONTINUATION` §4.3 had to write a special MUST for it** — and that rule states the cause
  outright, plus the general argument for bulk transfer: ***"the safe default is to bundle the whole
  chain — content-addressing makes over-inclusion free."*** **This is the answer to the operator's
  "does it become a handler extension that bundles?"** — the precedent is a **helper**
  (`collect_authority_chain`) **plus a per-surface MUST**, not a new extension, and the general form is
  a `collect_signatures(targets, expected_signers)` analogue. **The reason it cannot be automatic:
  bundling content is a CLOSURE operation needing only a root; bundling signatures is an EXPECTATION
  operation needing a set of expected signers** (§3.5's constructed path is keyed on `(signer, target)`).
  That is the architectural case for it living above the substrate.
- **`D-44` — the per-entry authorship rule exists in ONE unlanded draft and in none of the three landed
  app conventions.** `system/signature` occurs **zero times** in `EMBED`, `SHARE` and
  `SEMANTIC-CONTENT-SITE`; all three anchor on a root or manifest signature.
  `PROPOSAL-APP-CONVENTION-FEED` §1.1 is the sole home of the MUST (§3c.0 has the measurement and the
  region searched). **Two halves:** whether the rule generalises past feed entries by the §3c mutability
  line, and — when it lands — **an L21 fifth-shape fold audience**, since an `APP-CONVENTION-*` carries
  no diff a peer can read.
- **`D-45` — a signature over a SIGNATURE is well-formed today and asserts something different from a
  co-signature.** Targeting the content hash = a second authorship/endorsement claim (V7 §3.5 property 4,
  already specified). Targeting `content_hash(the signature entity)` = **witness**: *"this signature
  existed and I saw it"* — a timestamp/transparency primitive that closes the ordering gap TREE §3.3a is
  honest about and Nostr's `created_at` does not. **Zero substrate cost; `EXTENSION-ATTESTATION`'s
  supersedes/liveness graph is the home.** Not designed, flagged.

- **`D-42` — the `TREE_GET` leaf route delivers a binding without the authority's signature, and nothing
  obliges the consumer to go and get it.** `[restated 2026-09-06 — the first version said "is an
  unsigned claim," which was false; see §8]` §6.5.6 sanctions serving cached foreign namespaces and
  Amendment 6 fixes the leaf body as a bare 2-key `system/hash`, **while V7 §3.5's signature primitive,
  the invariant pointer, envelope carriage, TREE §3.7's per-binding inclusion proof and Amendment 10's
  served closure all already exist.** `GUIDE-SERVING-MODE` §8's three states do not cover the result.
  **The fix is carriage plus an obligation, not a mechanism** — §3a scores four sketches; **(a)+(b)
  compose and (c)/(d) look foreclosed by rulings already on the books.**

---

## §6 Sources

**In-corpus, opened this session by section:** `EXTENSION-NETWORK` §6.5.3 (Amendment 6, the two-hop
`TREE_GET` pointer), §6.5.3.1 (the `MANIFEST_GET` freshness bound), §6.5.6 (**`serve_scope` exposes the
local view** — the load-bearing quote), the `whole-store` bullet · `EXTENSION-TREE` §3.3a (`seq`,
`published_at`, and the byte-identical paragraph) · `GUIDE-SERVING-MODE` §8 (the three honest states) ·
`PROPOSAL-APP-CONVENTION-FEED` §5 (monotone vs authoritative, and the *quiet publisher vs withholding
origin* hole) · `GUIDE-RESOLUTION` §1 (P2), §5 · `EXTENSION-CONTENT` §6.4.1 via NETWORK's citation.

**Opened for the 2026-09-06 correction pass (§8), and these are the ones the first pass should have
read:** `ENTITY-CORE-PROTOCOL` **§3.5** — the `system/signature` type definition, the signature model,
the invariant-pointer path, the four properties, and the discovery-locality principle · `EXTENSION-TREE`
**§3.3a** signature carriage (`MANIFEST_GET` envelope `included`), **§3.7** + §6's math contract
(per-binding inclusion proof against snapshot root) · `EXTENSION-NETWORK` **§6.5.6** Amendment-10
signed-root closure MUST, **§6.5.3** publish-side closure obligation · `EXTENSION-SUBSTITUTE` **§2.1**,
**§4** trust contract (Signature MUST, no transitive trust), **§7.2** `verify_manifest`.

**External:** [NIP-01 — replaceable/addressable events](https://github.com/nostr-protocol/nips/blob/master/01.md).
The ATProto `rev`, SSB and IPNS rows in §3 are **from the predecessors' reads and general knowledge, not
opened for this document** — see §7.

## §7 What is unread

- **§3's table is the weakest thing here.** Only the Nostr row was opened this session. **ATProto's
  `rev`/commit semantics, SSB's log, and IPNS's record validity were not read at the source** for this
  document — they are stated at a level that is standard knowledge, and **none of them should be cited
  normatively until opened.** The finding does not depend on them; they are corroboration and are
  labelled as such (L18).
- **`EXTENSION-CONTENT` §6.4.1/§6.4.2** — the namespace-scoped vs single-trust-domain topologies, and
  Hash Tree Presence. Cited through `EXTENSION-NETWORK`'s references to them; **not opened.** D-42 should
  open them, because the presence rule is what actually gates a third-party `CONTENT_GET`.
- **Whether any implementation actually serves foreign namespaces today.** §0's finding is a claim about
  the *spec's permission*, not about deployed behaviour. **Both app-tier trees remain unopened — fifth
  document to say so** — and here it matters more than usual: if every seat serves `published-set` only,
  D-42 is latent rather than live, and that changes its priority.

---

## §8 Correction, 2026-09-06 — what this document got wrong and why it was reachable

**Operator-raised, same day, on the sentence this document led with:** *"I don't know why you're saying
we only sign the path-hash binding and only in bulk at the root. The whole invariant signatures are
designed so any peer anywhere can sign anything, and there's a canonical address for all of them."*
**Correct on every clause, and the corpus says so verbatim in `ENTITY-CORE-PROTOCOL` §3.5.**

**The retracted sentence:** *"Ours serves a `path → hash` binding that is signed only in bulk, at the
root, and only when the publisher advertises `signed_pointer`. So a third party's answer is
self-authenticating everywhere else and is not here."* **False three ways:**

1. **The primitive is general, not root-scoped.** `system/signature.target` is *any* content hash and
   `signer` is *any* peer — *"multiple peers signing the same content produce signatures at parallel
   paths."*
2. **There is a canonical address, built for this exact case.** The §3.5 invariant pointer
   `/{signer}/system/signature/{target_hex}`, whose stated property (3) is **Forwardable**: *"when Carol
   asks Alice for Bob's data, Alice can include Bob's signatures. Carol can verify Bob's authority
   without contacting Bob."*
3. **"In bulk" misreads the root signature.** TREE §3.7: each binding is verifiable by **inclusion proof
   against the snapshot root**. Signed once, verifiable per binding.

**And the same threat is already ruled, as a MUST, in `EXTENSION-SUBSTITUTE`** (§4, §7.2) — *"anyone
serving the URL can forge it"* → source-peer signature at the invariant pointer, reject on failure, no
transitive trust. **This document said no mechanism existed while the corpus carried the mechanism, the
address, the carriage, the per-binding math, and a worked precedent.**

**Why it was reachable, and it is a rule that already exists rather than a new one.** **L16's corpus
axis**: *a claim that this corpus does not describe something is discharged by naming the region
searched.* §6 lists what was opened — `NETWORK`, `TREE` §3.3a, `GUIDE-SERVING-MODE`, `GUIDE-RESOLUTION`
— **and `ENTITY-CORE-PROTOCOL` is not among them**, on a claim whose entire subject is the core
signature model. `grep -rn "system/signature" specs/` returns `EXTENSION-SUBSTITUTE`'s Signature MUST in
the first few hits. **The rule was not wrong; it was not run**, which is the same finding as the
2026-09-01 note in `AGENTS.md` — *the catalog was in context and did not function.* **No new letter.**

**The transferable half is about the shape of the error, not the miss.** The false sentence was a
**comparative** claim — *"every surveyed system does X and we don't"* — and comparative claims about our
own corpus are the most dangerous negatives we write, because the effort visibly goes into the foreign
column. Four external systems were characterised carefully; **our own column was filled in from the two
documents already open.** The tell was available: §7 of the *predecessor* had already flagged that
external rows were unread, so scepticism was pointed outward while the unchecked claim was inward.

**What survived the correction, and it is why the rule is *re-derive* rather than *delete*.** Chasing
the retraction produced a sharper finding than the false one: the carriage table in §3, which shows
**three of four binding-bearing surfaces carry the signature and the fourth was minimised deliberately
for dedup** — a real design tension with four scored resolutions (§3a), where the original said only
*"we lack a rule."* **The first version was pointing at something real and describing it wrongly.**
