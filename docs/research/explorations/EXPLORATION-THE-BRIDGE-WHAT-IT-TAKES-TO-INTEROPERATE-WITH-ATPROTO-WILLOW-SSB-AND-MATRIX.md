# EXPLORATION — the bridge: the two axes, the hub argument, and what each system costs

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested, and named as a goal rather than a curiosity.** **Rewritten 2026-09-04 after an
operator correction — the first draft conflated two orthogonal things and got lost in the weeds.**

> `[operator, 2026-09-04, paraphrased]` *"There are two different types. Say I want to participate in
> ATProto — internally I manage my stuff, and from ATProto's perspective I'm just another ATProto
> peer; I do the translation outgoing. The other one, HTTP, is easier to understand: I build an HTTP
> client, it goes out, pulls HTTP, translates it, stores it in the entity tree. That's pretty easy.
> And then — people speak HTTP, so we make an HTTP server that on the surface looks like it's just
> serving HTTP, and on the inside it translates into the entity language, stores the data,
> retranslates it back out, sends it over. Those are the types of bridges.*
>
> *Speaking the protocol is one thing. The data representation is another. Being a participant in
> their networks is another. And then there's bridge-to-bridge: maybe I'm a participant in ATProto,
> I pull stuff in, translate it into the entity protocol, then retranslate back out into Mastodon and
> push into Mastodon. **My internal representation — the entity system — is the bridge of bridges.
> It's the lingua franca that connects all these systems.** That's the goal: bridging and
> composability, by translating everything into the entity system so we have a standard
> representation, which can then be translated out by speaking their protocol, as a client or a
> server, understanding how to translate the data model.*
>
> *I don't know what you're doing saying we can't do it or it doesn't map. What do you mean, man? We
> have context for address data. Does the bridge extension maybe need to speak a different bit of code
> to interpret their security cryptography, their data model, the way they update it? Yeah — you've
> got to learn their language, you've got to speak it. But once you're back into the entity system,
> those translation parts are at the boundaries and the edges. Once you're in the entity system you're
> using all the tools the entity system has."*

**What the first draft got wrong, stated plainly, because it is the useful part:** it presented
*"carry the bytes / sign the translation / participate"* as **three modes of bridging.** Those are not
three modes. **The first two are what happens to the foreign integrity proof; the third is a
direction.** They are orthogonal, and collapsing them produced a taxonomy that could not express *"an
HTTP server that is entities underneath"* at all. **§1 is the corrected mode model and it is the
operator's; §2 is the integrity disposition, which is a separate axis.**

**Sources opened:** ATProto **Repository** + **AT-URI** · Willow **Data Model**, **Meadowcap**,
**Confidential Sync** · SSB **Protocol Guide** · Matrix **client-server** spec (partially — §8).

---

## §0 The hub argument, which is the actual thesis

**This is the sentence the first draft never wrote, and it is the reason the whole track exists.**

Point-to-point integration between **N** systems costs **N²** translators, and every one of them has
to understand two foreign models at once. **Translating everything into one internal representation
costs N bridges** — each one understands exactly one foreign model plus the one it already knows.

> **The entity system is the hub. A bridge is a spoke. And the value of the hub is not that it is
> better than any spoke — it is that a spoke author only has to learn one foreign language, and
> anything that reaches the hub can reach everything else.**

**Concretely:** pull from ATProto → hold as entities → push to Mastodon. The ATProto bridge author
never learns ActivityPub; the Mastodon bridge author never learns ATProto. **Neither of them writes
an ATProto↔Mastodon translator, and that translator is the thing that does not exist in the world
today.**

**What the hub has to be good at for this to work** — and all four are already true:

| Requirement | Why | Where it lives |
|---|---|---|
| **hold anything, byte-exact, forever** | a foreign object is *entirely* fields we did not model | open types + preserve-unknown-fields + hash-over-everything (`ENTITY-CORE-PROTOCOL` §2.10, `ENTITY-CBOR-ENCODING` §5.4) |
| **name things stably** | a bridge must be able to say *"this entity is that foreign object"* and have it survive | `path → hash` under a signed root |
| **say who asserted what** | a translation is somebody's claim, not the original author's | detached `system/signature` + `EXTENSION-ATTESTATION` |
| **let a component speak a foreign language at the edge** | their crypto, their canonicalization, their update rules | **wherever the bridge author puts it — §1a.** A handler is one packaging and a good default; a pure translator function, a fronted process and a proxy peer are others. **What matters is that the foreign competence is localized at the edge, not which shape holds it** |

**That last row is the operator's correction in structural form.** *"Does the bridge extension need to
speak a different bit of code to interpret their cryptography?"* — **yes, and that is where it goes.**
The entity system does not need to verify an SSB signature. **A bridge extension that speaks SSB
does**, and it hands the entity system a verified artifact plus its own attestation. **Nothing is
impossible; the foreign competence is localized at the edge.**

---

## §1 The mode model — two axes, four cells

**Axis 1 — direction.** Which way is data moving?
**Axis 2 — surface role.** Do we reach out, or do we present a surface they connect to?

| | **Client** (we initiate) | **Server** (we present a surface) |
|---|---|---|
| **Ingress** — foreign → entities | **pull and translate.** An HTTP client that fetches, translates, and binds into the tree. An ATProto firehose consumer. An SSB `createHistoryStream` reader. **The easiest cell, and the right first build** | **accept and translate.** They push to us: an ActivityPub `/inbox`, an SMTP MX, a webhook receiver. We look like a normal destination to them |
| **Egress** — entities → foreign | **translate and push.** Post to a PDS, deliver to Mastodon inboxes, publish to Nostr relays. **This is "being a participant"** — from their side we are just another peer | **translate and serve.** An HTTP server that looks like plain HTTP, a PDS-shaped endpoint, a relay. **Entities underneath, native on the surface** |

**Three things this framing gets right that the first draft could not say:**

1. **"Participation" is not a separate kind of bridge.** It is the **egress** column executed well
   enough that the foreign network cannot tell. *"From ATProto's perspective, I'm just another ATProto
   peer."* Whether that is client-egress (we push to their relay) or server-egress (they crawl our
   PDS-shaped endpoint) is **their protocol's choice, not ours.**
2. **The HTTP pair is the clearest teaching example and it is one system in two cells.** Ingress-client
   (*"pull HTTP, translate, store"* — easy) and egress-server (*"an HTTP server that on the surface
   looks like HTTP, and inside is entities"*). **Same foreign protocol, opposite cells, and the second
   is the one that lets people who only speak HTTP use what we built.**
3. **A full bridge is usually two cells, not one**, and they can be built and shipped separately. **The
   ingress cell is almost always cheaper**, which is why it is the right first artifact for any system.

**And the four cells compose into the chain the operator described:**

```
  ATProto  --[ingress: client]-->  ENTITY SYSTEM  --[egress: client]-->  Mastodon
                                   (canonical form,
                                    stored, addressed,
                                    signed, curated)
```

**Each half is an ordinary bridge. Neither half knows the other exists.** That is the whole
composability argument, and it only works because the middle is a real representation rather than a
transcoding buffer.

---

## §1a What a bridge IS — one seat's decomposition, and the space it does not cover

**This document's §0 assumed *"a handler"* without checking and named that as the open question
(`[X-4]`). It is now PARTLY answered — an application seat opened their own source and produced a real
decomposition — and §1a.2 is the half that decomposition does not settle.**

> **`entity-browser-rust` already ships two ingress bridges and never called them that.**
> `content_site::ingest::ingest_path` reads a genuinely foreign data model — the content team's
> `render/` JSON schema — and writes entities. `apps::ingest` does the same. `crossimpl_go` reads
> another implementation. **None of the three is a handler, an extension, or a capability holder.
> They are plain functions writing into the tree.**

**So the structural answer is three parts, and the proportions are the useful half:**

| Part | What it is | Depends on the substrate? |
|---|---|---|
| **The translator** | a **pure function**, foreign bytes ↔ entities. **This is where their crypto, their canonicalization and their update rules live** | **no — none at all** |
| **A transport role** | one of §1's four cells | yes |
| **Tree I/O** | bind the results | yes |

**The consequence is the cheapest finding in the whole bridge track: the translator is the bulk of the
work and it needs nothing from us, so a bridge can be written and fully tested before deciding where it
runs.** That inverts the natural build order — the instinct is to settle the hosting question first,
and the hosting question is the small end.

**And this is where the seat's answer stops and the design space starts.** Their packet says *"a bridge
is a handler"* holds for one of four cells, because a foreign-server cell receives bytes **at a socket,
before anything entity-shaped exists** — no path to dispatch at, no grant to check. **That is true of
the socket-facing edge and it is not a claim about the whole bridge.**

> **The handler abstraction is opaque on its far side, and that is the point of it.** A handler is an
> operation the system dispatches; **it does not say what happens behind it and does not need to.**
> Installing a handler that **starts and manages a long-running parallel process** is a legitimate and
> probably common bridge shape — the operator's example is a Bitcoin node: reading as a client is easy,
> **being a full node is a lot harder and gets encapsulated**, and the natural packaging is *install a
> handler that is a service*, with the process living beside it and connecting back in, or not.
> **So "not a handler" is a true statement about three specific functions in one tree, and a false
> generalization about bridges.**

### §1a.2 There is no single bridge design, and there should not be

**One seat, one implementation, one perspective — and the three artifacts they read are all
ingress-client, in a browser.** That is the cheapest cell in the cheapest position, and its shape does
not generalize to a cell that holds a socket, runs a daemon, or maintains consensus state.

**Two axes the design space runs along, neither of which has a right answer:**

1. **What the bridge IS** — a pure function called inline · a handler · a handler fronting a supervised
   process · a separate peer acting as a proxy · a standalone service that speaks both sides. **All are
   constructible and they suit different foreign protocols.**
2. **How deep it integrates.** A bridge can be **fully encapsulated on the core protocol** — no
   extensions, its own everything — or it can **pull in revision, identity and content** and be an
   integrated citizen of the standard extension stack. *The first is portable and self-contained; the
   second inherits versioning, merge, attribution and naming for free and is coupled to them.*

**And nothing makes one implementation per foreign protocol correct.** There may well be five ATProto
bridges with five designs, the same way there are many HTTP clients. **The core protocol is what binds;
everything above it is an extension choice, and everything above the core extensions is whatever
somebody builds.** This document should be read as *the axes and their costs*, never as a
recommendation of one shape.

> **The honest framing for the whole bridge track: there is no community to inherit an answer from, so
> we make the calls ourselves and try to get close enough that an arriving community adopts rather than
> rebuilds.** That is a different standard from *"correct"* and it is the one that applies.

`[entity-browser-rust` `62c6d62`, from source; folded 2026-09-05.]`

---

## §2 The integrity disposition — a separate axis, and it is where honesty lives

**Orthogonal to §1.** Whichever cell you are in, one question has to be answered for every object that
crosses: **what happens to the proof?**

| Disposition | What is stored | What you may claim | Cost |
|---|---|---|---|
| **Carried** | the foreign bytes, verbatim, native signature inside them, wrapped in an entity | *"these are exactly the bytes X published"* — **and a component that speaks their crypto can verify X's signature, now or in ten years** | near zero |
| **Re-authored** | a translation into our vocabulary, **signed by the bridge** | *"the bridge asserts this content appeared at that address"* — an **attestation**, worth what the bridge is worth to you | the native proof is not in the artifact |
| **Both** | the translation **plus a reference to the carried original** | both of the above, and a reader can escalate from convenience to proof | one extra binding |

**"Both" is the right default and nobody in the field does it.** The translation is what makes the
object usable by every existing entity-system tool; the carried original is what makes the claim
checkable. **They are cheap together because a reference is `{peer, hash, path?}` and the original is
already an entity.**

**The failure this axis exists to prevent already has a conformance vector.** **FEED-9:** *"a mirrored
entry with its signature stripped MUST NOT be presented as attributed — integrity without authorship
is the failure mode the signature exists to prevent."* **A bridge that re-authors an ATProto post and
presents it as authored by that person is FEED-9 at ecosystem scale, and it is the standard bridge bug
in the field.** Matrix's ecosystem arrived at the honest form operationally: **a bridge is a service
that holds a named identity on both sides**, not an anonymous transcoder.

**Note how this maps onto the three acts** (`PROPOSAL-UNKNOWN-FIELD-PRESERVATION-…` §2a): **carried =
relay** (bytes unchanged, hash unchanged, not our claim); **re-authored = transform** (new bytes, new
hash, we sign it, we own it). **The corpus already has the vocabulary; the bridge is its second
consumer.**

---

## §3 What actually has to be decided per system — three questions, no impossibilities

**The first draft wrote *"bridges in but not out"* and *"does not bridge."* That was wrong as stated.**
Everything is representable; what differs is **cost** and **what is preserved versus re-authored at
the boundary.** Three questions, asked once per system:

1. **Identity** — how does their identifier become ours, and back? *(Usually easy: four of five
   systems are an ed25519 key wearing an encoding — §6.1.)*
2. **Integrity** — is their proof **carried** (we store their bytes and a bridge extension can verify
   them) or **re-authored** (we sign our translation)? *(§2. Never "lost" — that is a choice, not a
   fact.)*
3. **Update and ordering semantics** — how is *"this came after that"* and *"this replaced that"*
   represented on our side, and does the mapping **invent** a guarantee we do not have? *(§6.3 — this
   is the one that bites.)*

**Everything else is encoding, and encoding is work rather than a wall.**

---

## §4 ATProto — the closest fit in the field, by a distance

### §4.1 The structural match is almost line-for-line

**Their commit:** `{ did, version: 3, data: <CID → MST root>, rev: <TID logical clock, monotonic>,
prev: <CID|null, "virtually always null">, sig }`

| ATProto | Us | |
|---|---|---|
| `did` | `peer` (key-derived) | identifier form differs, **root is the same key** |
| `data` → MST root CID | `root_hash` → trie root | ✓ |
| `rev` (monotonic logical clock) | root sequence number | ✓ |
| `prev` — **nullable, "virtually always null"** | `prev` — **opt-in** | ✓ **third independent verdict on §2.3.2** |
| `sig` | detached `system/signature` | ✓ (embedded vs detached) |
| record path `<NSID>/<rkey>` | tree path | ✓ |
| DAG-CBOR records | canonical CBOR (ECF) | close, not identical — §8 |
| `strongRef {uri, cid}` | `reference {peer, hash, path?}` | ✓ |

**And the MST is our trie:** *"content-addressed"*, *"deterministic"*, with *"deterministic shaping
based on current contents, regardless of insertion/deletion history"*, deleting *"without leaving a
trace or 'tombstone'"*. **That is our result 4, independently derived.**

### §4.2 Per cell

| Cell | Cost |
|---|---|
| **ingress-client** (consume the firehose / fetch repos) | **low.** Public by design; *"a new PDS is visible immediately."* No permission needed |
| **ingress-server** | n/a — ATProto has no push-to-you model |
| **egress-client** (publish into ATProto) | **moderate, and it is governance not code**: a DID, a PDS to write to, lexicon conformance |
| **egress-server** (be a PDS) | **the real "participant" build** — serve `com.atproto.sync.*`, emit a firehose, hold a DID. **Genuinely plausible, and the only system here where it is** |

**The asymmetry generalizes and is the single most useful operational fact in this document:
importing is an engineering problem; exporting is a governance problem.** Reading ATProto is
permissionless. Writing means holding a DID and conforming to lexicons you do not control.

---

## §5 Willow, SSB, Matrix, Nostr — cost, not possibility

### §5.1 Willow

**Mapping:** `subspace_id ↔ peer` ✓ · `path ↔ path` ✓ · `payload_digest ↔ content hash` ✓ ·
`namespace_id` **has no home** (their top-level partition has its own keypair; ours is the peer) ·
`timestamp` — **the trap below.**

**Ingress:** easy. **Egress-client:** needs a Meadowcap capability and their encodings. **Egress-server
(be a Willow peer):** expensive — 3D RBSR, Meadowcap, LCMUX, PIO. *Possible, unmotivated today.*

> **The one dangerous line in this document.** Mapping our `created_at` onto Willow's `timestamp`
> **promotes a display hint into an overwrite authority** — Willow resolves overwrite at a
> `(subspace, path)` by newer-timestamp-wins, so a bridge that maps the two hands an author the
> ability to **backdate over history in a Willow namespace.** *(our reading; should be checked against
> Meadowcap's timestamp-range restrictions, which may already bound it.)*

### §5.2 SSB

**Message:** `{previous, author, sequence, timestamp, hash, content}` + `signature`, signed over
**whitespace-exact canonical JSON** (two-space indent, LF, no trailing newline, one space after
colons). Message id = sha256 **including** the signature. Feed id = `@<base64 ed25519>.ed25519`.

**Ingress: easy and high fidelity.** Blobs (`&<base64 sha256>.sha256`) map **straight onto our content
store**; `sequence`/`previous` maps **straight onto our opt-in `prev`** — the cleanest use that field
has. A bridge extension implementing their canonical JSON can **verify their signatures on carried
bytes**, so ingress can be *carried*, not merely re-authored.

> **And the negative half, which this section did not state and which a bridge author needs more than
> the positive one: carrying is not an optimization, it is the only way the message keeps its name.**
> Because the message id hashes the signature over a byte form we do not emit, **no amount of faithful
> field mapping reproduces `%<hash>`** — so a bridge that translates without carrying the original
> bytes has not merely lowered fidelity, it has **silently converted a *carried* integrity disposition
> into a *re-authored* one** (§2), while every field still looks right. *Carrying is possible* and *not
> carrying destroys the identity irrecoverably* are different sentences and only the first was here.
> `EXPLORATION-THE-FALSIFICATION-TEST-…` §8.1.

**Egress: entirely possible, and it was wrong to imply otherwise.** A bridge holds an SSB identity,
serializes to their canonical JSON, signs with **its own** SSB key. **The cost is that the bridge
becomes the author on that side** — which is `§2`'s re-authored disposition, stated honestly, and is
exactly how every Matrix bridge already works.

**Two real constraints, neither an impossibility:** joining SSB is **socially gated** (you are
replicated only if followed, or via a Pub), and **SSB cannot delete** — so **anything pushed into SSB
is permanent.** *That is a consent question to settle before anyone builds an egress bridge, not an
engineering one.*

### §5.3 Matrix

**Timeline events** (`m.room.message`) map onto entries cleanly, both directions.

**Room state** — membership, power levels, bans — is where the cost is, and the first draft's *"does
not bridge"* was too strong. **It is representable**: the event DAG, the auth chain and the resolved
state are all data, and a bridge extension could hold them as entities. **What it costs is that
somebody has to run state resolution**, because a Matrix room's current state is not a stored fact but
a *computed* one, and it is the computation Matrix has been repairing since 2016 (v2.1 shipped in room
version 12 in **2025**).

**So the honest statement is a recommendation rather than a limit:** *bridge the messages first;
treat room state as a snapshot with a stated as-of, and only implement state resolution if a consumer
actually needs authoritative membership.* **A snapshot is a legitimate representation — it just has to
say it is one**, which is capstone §5's *a view declares what produced it*.

### §5.4 Nostr

**Cheapest in the field.** Plain JSON events, one signature scheme, no repository, no MST, no state.
**Format mapping trivial; the semantic mapping is the work**, because kinds are opaque integers with
per-kind tag vocabularies. **The inverse of SSB**, where the format is the work.

---

## §6 Cross-cutting

### §6.1 The keys are the interop surface, not the identifiers

SSB `@…=.ed25519` · ATProto DID (signing key in the DID document) · Willow `UserPublicKey` · Nostr
`npub…` (secp256k1) · us (key-derived peer id). **Four of five are a public key wearing an encoding**,
so identity bridging is mostly re-encoding — plus the question of whether the *same human* is behind
two keys, which no protocol answers and which **is an attestation.** `EXTENSION-ATTESTATION` already
carries the shape, and this is its third use (after moderation labels and scores).

### §6.2 The fidelity contract is what makes the hub possible

**Carrying a foreign object requires holding content we did not model, exactly, forever, through
arbitrary forwarding — and a foreign object is *entirely* content we did not model.** Under
`ENTITY-CBOR-ENCODING` §5.4's current **SHOULD**, a conformant peer may drop it; the carried artifact
then no longer verifies under the foreign signature, **with a green conformance run.** This is
`PROPOSAL-UNKNOWN-FIELD-PRESERVATION-…` §3a and it is that proposal's most concrete argument.

### §6.3 A bridge may demote an ordering guarantee, never promote one

Every system has *some* ordering — SSB `sequence`+`previous`, ATProto `rev`, Willow wall-clock
`timestamp`, Matrix DAG. **Importing collapses cleanly** onto `prev` or a root sequence. **Exporting
can invent a guarantee we never made** — §5.1 is the live instance. *Candidate rule, one incident:
honor it, do not claim it generalizes.*

### §6.4 Deletion is where the systems genuinely differ

We and ATProto delete **without a tombstone** from a history-independent structure. Willow deletes by
prefix-prune and it **propagates**. Matrix **redacts**, keeping a shell. **SSB cannot delete at all**,
which makes any egress-to-SSB bridge a **one-way ratchet** and therefore a consent decision (§5.2).

### §6.5 The vocabulary problem is the bridge problem, one layer up

`app/feed/entry` ↔ `app.bsky.feed.post` ↔ SSB `type: post` ↔ `m.room.message` ↔ Nostr kind 1 is **five
names for a thing everyone agrees exists.** `EXPLORATION-EVOLVABLE-SCHEMAS-…` §4 governs.

**And the pressure runs the right way:** *a bridge is the second non-social consumer the promotion
ladder asks for.* **If our four shapes cannot absorb an ATProto post, an SSB message and a Matrix
timeline event, the taxonomy floor is wrong** — a cheap, concrete falsification test, runnable on
paper. **§7.1.**

---

## §7 What this opens

1. **Run the falsification test on paper.** One real ATProto post, one SSB message, one Matrix
   timeline event, one Nostr kind-1, each mapped onto the four shapes. **Cheapest high-value item on
   this track** and it tests the taxonomy floor against data nobody designed for us.
2. **The bridge use case belongs in the fidelity proposal** — **done**, §3a.
3. **"A bridge may demote an ordering guarantee, never promote one"** — candidate, one incident.
4. **Bridged content is an attestation, not authorship** (§2). First normative sentence of any future
   bridge convention; `EXTENSION-ATTESTATION` carries the shape.
5. **An egress-to-SSB bridge is a consent decision** (§5.2, §6.4). Name it before anyone builds one.
6. **What is a bridge, structurally, in our own terms?** A handler? An extension? A peer that holds two
   identities? **Nothing in the corpus says**, and §0's fourth row assumes "a handler" without checking.
   *This is the first thing a real bridge proposal has to answer and it is not answered here.*
7. **Nobody has asked for a bridge yet.** L26: the seats discover by building. **This is a map, not a
   design.**

## §8 Sources, and what is unread

**Opened:** [ATProto Repository](https://atproto.com/specs/repository) ·
[AT-URI](https://atproto.com/specs/at-uri-scheme) ·
[Willow Data Model](https://willowprotocol.org/specs/data-model/index.html) ·
[Meadowcap](https://willowprotocol.org/specs/meadowcap/index.html) ·
[Willow Confidential Sync](https://willowprotocol.org/specs/sync/index.html) ·
[Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/) ·
[Matrix client-server](https://spec.matrix.org/latest/client-server-api/) ·
[Matrix Room Versions](https://spec.matrix.org/latest/rooms/)

**Unread, and each would change a claim above:**

- **Matrix's event format and redaction algorithm were NOT fully extracted** — the fetch returned
  headings without the field list or the redaction-survival rules. **§5.3 is structural reasoning, not
  a read.** Weakest sourcing in the document, named rather than papered over.
- **ActivityPub / Mastodon** — the operator's own chaining example ends in Mastodon and **this
  document has no ActivityPub section at all.** The axis reference covers it; a bridge-grade read does
  not exist. **Highest-value gap on this track.**
- **ATProto's DID methods** (`did:plc` operation log, `did:web`) — §4.2 calls identity "governance"
  without having read how hard.
- **Willow's encodings spec** — what an egress bridge would encode to.
- **DRISL / DAG-CBOR's exact divergence from our ECF** — §4.1 says *"close, not identical"* and does
  not say how. **A real bridge starts by measuring that delta**, and it is a bounded, cheap
  measurement.
