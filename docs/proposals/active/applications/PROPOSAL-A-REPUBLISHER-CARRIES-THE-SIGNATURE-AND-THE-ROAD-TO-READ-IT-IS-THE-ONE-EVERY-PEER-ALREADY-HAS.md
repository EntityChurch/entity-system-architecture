# PROPOSAL — a republisher carries the author's signature in its own view, and the road a third party reads it over is the one every peer already has

**Proposes:** `SYSTEM-DATA-EXCHANGE` v0.2 → v0.3
**Status:** DRAFT (2026-09-17) — two sentences of new normative text and two conformance rows. The
larger half of this document argues that **nothing else changes**, because the question that prompted
it is already answered by the core protocol's addressing model.
**Tier:** system composition.
**Rests on:** `ENTITY-CORE-PROTOCOL` §1.4 (URI and path model; local view; authority layers; the
cross-peer worked example) · §5.4 (canonicalization) · §5.2 (grant pattern forms) · §6.5 (inbound
dispatch canonicalization) · `SYSTEM-DATA-EXCHANGE` §2.2, §2.3.

> **In one sentence.** A republishing peer already binds the author's detached signature under the
> **author's** namespace in its own local view; a third party already reads it by naming the
> **serving** peer's tree handler and the **author's** absolute path as the resource; and the only
> thing missing from the exchange tier is that it says the first is optional and never says the
> second at all.

---

## §0 What prompted this, and why the framing needed checking before the finding

An implementation built the gathering loop and ran the third hop of `DX-C1` end to end — **A** authors
and publishes, **B** gathers and republishes, **C** has never spoken to A. The measurement:

```
LEG 1 ok      — the mirror head is committed at the derived coordinate
LEG 2 ok      — 3 entries named, every one referencing A and not B
LEG 3 ok      — B served all 3 of A's entries by hash, from its own content store
LEG 4         — 0 of 3 mirrored entries attributable by C
LEG 4 control — 3 of 3 attributable when read DIRECTLY from A
```

That is the state §2.2 says **MUST NOT** be presented as attributed, reached by an implementation
that had done everything the exchange tier asks of it. The control arm is what makes it a measurement
about mirrors rather than about a harness, and it should be copied by anyone reproducing this.

**The finding is real and the framing that arrived with it is not.** The framing was that resolving
the evidence *"needs a read operation that names serving-peer and resource-namespace separately,"*
that no road expresses this, and — generalized — that **a peer-qualified path is ambiguous in both
directions and the corpus has no way to say which is meant.**

**Every clause of that is answered in landed core text, and two of the three are named instances of a
bug class the core protocol already calls the single most-recurring one in this ecosystem.** §1
below is therefore not a courtesy citation: if the framing had been adopted, this tier would have
grown a road qualifier, a second way to locate a signature, and a second trust argument, to route
around an addressing model that was working.

---

## §1 The road exists, and it is the core's worked example

### 1.1 Reading — the handler names who you ask, the path names what you want

`ENTITY-CORE-PROTOCOL` §1.4's cross-peer worked example is this operation, written out. Step (3),
with the example's Alice as the serving peer and Bob as the author:

```
EXECUTE entity://alice_id/system/tree  operation:"get"
  resource: {targets: ["/bob_id/local/files/readme.md"]}
```

**The handler URI and the resource target are separate fields, and they carry exactly the two things
the framing said no road could name separately.** The handler URI is *who you are asking*; the
resource target is *what you are asking about*, absolute and rooted at the peer whose namespace it
belongs to. Substituting the exchange case changes nothing but the letters:

```
EXECUTE entity://{B}/system/tree  operation:"get"
  resource: {targets: ["/{A}/system/signature/{target_hash_hex}"]}
```

**This is not an inference from the addressing model. It is the model's own example, at the layer the
model defines it.** §1.4 states the semantics in prose beside it: a binding under another peer's
namespace *"is Alice's cache authority over her own view of Bob's subtree; Bob's key remains the
namespace authority for what is canonically true there."* That sentence is the whole ruling — a
reader asks the serving peer what it knows about the author's namespace, and the author's key decides
whether the answer is true.

### 1.2 The reading is not ambiguous, and §6.5 is what makes it unambiguous

The claim that `/{A}/x` means both *"ask A for x"* and *"my copy of A's x"* with no way to
disambiguate is **false, and the disambiguator is not subtle**: it is which peer the EXECUTE's
handler URI names.

| The operation | Handler URI | Resource target | What resolves |
|---|---|---|---|
| A third party reads the republisher's view | `entity://{B}/system/tree` | `/{A}/…` | **B's local view of A's namespace** |
| A peer's own handler wants the authoritative copy | (internal sub-dispatch) | `/{A}/…` | **routed outbound to A** (§1.4, *Internal dispatch*) |

**An inbound EXECUTE is never re-routed.** §6.5 canonicalizes at step 3, before handler resolution and
before `check_permission`, and the core notes that on this path `target_peer == local_peer_id`
*"always holds"* by the time the peer-scope dimension runs. A request arriving at B is answered by B
out of B's own view, definitionally. The second row of that table is a **locally-originated outbound
request**, which the core distinguishes by name — it is B deciding to go ask A, not C's read being
redirected.

⇒ **The two readings belong to two different operations and the corpus does say which is meant.**

### 1.3 The write direction is the same example, step (1) — and the error message is the tell

The same implementation hit the write half the same afternoon: binding obtained bytes under the
author's namespace dispatched to **the author**, who refused with `403 capability_denied` — *"a
sentence indistinguishable from `that publisher revoked us`,"* with the author not involved at all.

§1.4's worked example opens on precisely this and says it is unremarkable:

> `; (1) WRITE — Alice caches one of Bob's entities into her local view.`
> `;     The handler URI targets Alice (her local peer); the *resource target* is`
> `;     an absolute path in Bob's namespace. Cap-check authorizes; nothing about`
> `;     the path being foreign is special (§1.4 layer-1 local tree control).`

The implementation's own repair — pin the handler local and let the peer-qualified path travel as the
resource — **is that comment**. Reaching it independently is a good sign about the model; it should
not have required reaching.

### 1.4 The prepend bug is named in the core, and it has a conformance category

The reported cause on the read side was that the fetch layer *prepends the serving peer*, so the only
path it can express is one under the server. §1.4 states the rule that forbids this and predicts it:

> (1) **Form-agnostic input** — a peer-relative path (no leading `/`) is qualified to
> `/{local_peer_id}/...`; an **already-absolute** path (leading `/`, e.g. a cached remote namespace
> `/{other_peer_id}/...`) **passes through unchanged**. A layer MUST NOT re-qualify an already-absolute
> path — doing so yields the corrupt `/{local_id}//{other_id}/...` (the prepend-local double-segment
> bug). … **This is the single most-recurring cross-impl bug class.** The durable guard is the
> cross-impl `universal_address_space` conformance category.

And the rule is scoped, in the same paragraph, to *"wherever a path parameter is accepted: listing
render, scope filters, namespace enumeration, capability canonicalization, tree-walk, subscription
matching, and any handler-internal function that takes a path."* A signature-locating helper is such
a function.

⇒ **The read-side defect is an instance of a documented class with an existing conformance category,
not a gap at the exchange tier.** It is also why the sibling defect — deriving the pointer under *the
peer being read from* rather than under the **signer** — is the same mistake twice: both re-qualify a
path that was already absolute.

### 1.5 The grant shape is expressible today, and the leading slash is the whole difference

The remaining leg was that a grant naming `system/signature/*` canonicalizes to the granting peer's
own namespace, so B cannot express *"my copy of A's signatures."* **It canonicalizes that way because
the pattern is peer-relative, which is what the core specifies it to mean**, and §5.2's summary of
grant pattern forms states the alternative in one line:

> Bare `*` is peer-relative — it matches all paths under the local peer only. To match all peers, use
> `/*/*`. **The leading `/` always signals universal tree scope.**

So B grants C either of:

| Pattern | Meaning |
|---|---|
| `/{A}/system/signature/*` | my copy of one named author's signatures |
| `/*/system/signature/*` | my copy of every author's signatures |

**No new grant shape, no road qualifier, no scope extension.** A pattern that was read as inexpressible
differs from the expressible one by one character, which is a good argument for §3's conformance row
and a poor argument for new normative text.

---

## §2 What is actually missing, and it is on this tier

§1 disposes of the road. Two things the exchange tier genuinely does not say remain, and the
measurement above is the instance of both.

### 2.1 §2.3 rule 2 makes carrying the evidence optional

Landed text:

> 2. **[MUST]** **A republished entry travels with its author's detached signature** (§2.2). A
>    republishing peer **MAY** carry one and **MUST NOT** supply one.

**The bolded sentence and the sentence under it disagree**, and the permissive one is the operative
word for an implementer. `MUST NOT supply` is right and stays — a republisher does not hold the
author's key and cannot author in the author's name. **`MAY carry` is wrong**: a republisher that
declines to carry the evidence emits a view that §2.3 rule 3 then obliges every reader to render
**unattributed**, which is a correct, verifiable, complete publication in which nobody wrote anything
— the exact failure §2.4 says is invisible because the bytes check out.

**Obtaining the signature costs a republisher nothing it has not already done.** It read the entry
through a verifying consumer, which means it resolved the signature to verify it. Carrying it is
binding bytes already in hand.

⇒ **`MAY` → `MUST`, with `MUST NOT supply` unchanged.** The two are not in tension: *carry what the
author signed; never sign in their name.*

### 2.2 Nothing says where the republisher binds it, or where a reader looks

§2.2 names the pointer — `/{signer_peer_id}/system/signature/{target_hash_hex}` — and stops. It does
not say that a republisher binds it **at that same absolute path in its own local view**, and it does
not say that a third party **resolves it against the serving peer**. Both follow from §1; neither is
written anywhere a reader of this document will look, and the measured consequence is that an
implementation derived the pointer under the serving peer instead and concluded the corpus was silent.

**This is the cheaper half of the ruling.** One paragraph here removes the need for anyone to re-derive
§1 from the core's addressing model — which two implementations will otherwise each do, at the moment
they first read a mirror over a live transport.

---

## §3 The proposed text

### 3.1 `§2.2` — append

> **[MUST]** A republishing peer **MUST** bind the evidence at the pointer's own absolute path —
> `/{signer_peer_id}/system/signature/{target_hash_hex}` — **in its own local view**, unchanged. The
> path is rooted at the **signer**, never at the republisher: it is the same path the evidence
> occupies on the author's own peer, and it is the same path on every peer that carries it.
>
> **[MUST]** A reader obtaining a republished object from a third party **MUST** resolve that pointer
> **against the peer serving the object**, by naming that peer's tree handler and the signer-rooted
> path as the resource. A reader **MUST NOT** derive the pointer under the serving peer's identifier,
> and an implementation **MUST NOT** re-qualify the already-absolute pointer to any other namespace.

**Why it is two sentences and not a mechanism.** Both restate `ENTITY-CORE-PROTOCOL` §1.4 at the tier
that needs them. They introduce no operation, no field, no pattern and no road; an implementation that
already honors §1.4's form-agnostic-input rule satisfies both without a code change.

> **Restatement provenance.** Both sentences name `ENTITY-CORE-PROTOCOL` §1.4 as their authority in
> the folded text, so this pair is a declared pointer and is checkable against its source rather than
> free-standing.

### 3.2 `§2.3` rule 2 — replace

> 2. **[MUST]** **A republished entry travels with its author's detached signature** (§2.2). A
>    republishing peer **MUST** carry one when the entry it republishes has one, and **MUST NOT**
>    supply one: it does not hold the author's key and cannot author in the author's name.

**The conditional is load-bearing and is not a softening.** §2.3 rule 3 and `DX-C5` already require
that an entry whose signature is genuinely unobtainable be **carried and rendered unattributed** —
not dropped. A flat `MUST carry` would oblige a republisher to drop such an entry, contradicting a
landed check. The obligation is on evidence the republisher *can* obtain, which — per §2.1 above —
is every entry it verified.

### 3.3 `§3.1` — two requirement rows

| id | Requirement | Level | § |
|---|---|---|---|
| `DX-R18` | Bind a republished entry's authorship evidence at the signer-rooted invariant pointer in the republisher's own local view, and carry it whenever it is obtainable | MUST | §2.2, §2.3 |
| `DX-R19` | Derive or re-qualify that pointer under the serving peer's identifier rather than the signer's | MUST NOT | §2.2 |

### 3.4 `§3.2` — one check row, and a sharpened arm on `DX-C1`

| id | The check | Drives | What fails without it |
|---|---|---|---|
| `DX-C9` | The third hop of `DX-C1`, **with the consumer reading only from the serving peer and never contacting the author** — assert the serving peer *holds* the evidence at the signer-rooted path, and that the consumer *resolves it there*. Run a **control arm** reading the same entries directly from the author | `DX-R10`, `DX-R18`, `DX-R19` | ⭐⭐ **a mirror that carries integrity without authorship** — every hash matches, every reference is right, the view is addressable and complete, and nothing in it is attributable |

**`DX-C1`'s row gains one clause**, because it is the arm that was assumed rather than run: after
*"and C's consumer is the same code path it uses for a direct read,"* add — ***"with C's reads
directed at B; a check in which C can reach A does not measure this."***

⚠ **The anti-vacuity arm is the control**, and it is the same discipline `DX-C2a` exists for: a
third-hop measurement with no direct-read arm is a statement about the harness. A consumer that can
still reach the author passes `DX-C1` while measuring nothing about mirrors — **which is how the
third hop went unrun while the check was counted as covered.**

### 3.5 Version

`SYSTEM-DATA-EXCHANGE` **v0.2 → v0.3**, with a Document History entry. Additive: no landed
requirement is renumbered, and `DX-R10`–`DX-R14` keep their ids and meanings.

---

## §4 What this proposal deliberately does NOT do

| Not doing | Why |
|---|---|
| **Adding a road qualifier** to the `DX-C1`-adjacent requirements | §1. The distinction the qualifier would carry is already carried by the handler URI, which is a separate field of every EXECUTE |
| **Putting the resolution rule in `EXTENSION-NETWORK`** | It is not a transport property. It is true on a live dispatch, a static projection and a local read, because it is a property of the address |
| **Editing `ENTITY-CORE-PROTOCOL`** | Nothing there is wrong. §3.1's sentences restate it and name it as their authority |
| **Defining a second way to locate a signature** | A second locator is a second trust argument. The implementation that found this declined to patch it for that reason, and was right |
| **Requiring the republisher's signed root to commit to the carried evidence** | Measured and correctly ruled out at the source: a detached signature is read **outside** the committed set by design — a canonical peer id is a commitment to its public key, so the signature is self-verifying. Committing to it would re-couple the evidence to a root, which is the coupling §2.2 exists to break |

---

## §5 What would falsify this

**The load-bearing claim is §1.2: that an inbound EXECUTE naming the serving peer's tree handler,
with a resource under another peer, reads the serving peer's local view and is never re-routed.**
If an implementation can show a landed core rule under which that request is instead dispatched
outbound to the author, §3.1's second sentence is wrong and the framing that prompted this proposal
is right — a road qualifier, or something like it, would then be owed.

**The evidence offered against that, and it should be checked rather than taken:** §1.4's worked
example step (3) performs the read and labels the result a cache; §1.4's *Internal dispatch*
paragraph scopes outbound routing to a **locally-originated** sub-request; §6.5 canonicalizes inbound
before handler resolution, with `target_peer == local_peer_id` holding thereafter.

**A second, cheaper falsifier:** if a grant carrying `/{A}/system/signature/*` is refused by a
conformant peer, §1.5 is wrong. That is a one-check experiment and it is worth running before
anything here is built against.
