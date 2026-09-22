# EXPLORATION — the shareable reference: why a name and a key are not redundant, and what should actually be pinned

**Status:** Exploration (design record). Not a proposal, not normative.

**Origin: the operator's objection, 2026-09-05**, to the answer this corpus gave the day before.

The objection, stated as the four questions it raises:

1. **To share Alice via a registry, wouldn't the public key be needed as well?** And if the key is
   being shared directly, the registry is not doing anything — so why not just share the key?
2. **But can the names be trusted?** Names change.
3. **If a public key ends up tacked on anyway, naming and registries lose their point** — the keys
   alone would have done.
4. **And if the registry key must always be pinned, keys are still being juggled**, which is the
   difficulty the names were supposed to remove.

---

## §0 The result

**The objection is correct about the form the guide recommended, and the correction is not a
defence of names — it is that we were pinning the wrong thing.**

`GUIDE-RESOLUTION` §6.1a said: *for a link you intend to share, use `alice@<peer-id>` — name for reach,
key for trust.* A peer-id is a **handle key**, and a handle key **rotates** — that is what
`EXTENSION-IDENTITY` §4.3 (handoff) and §4.4 (compromise-recovery) are for. So the recommended share
form pins the one part of an identity that is designed to change, and it inherits the failure the
operator names: it is a key, wearing a name.

**The identifier that never changes already exists in this corpus and is not the key.** It is
`quorum_id` — the **content hash of the quorum entity** (`EXTENSION-QUORUM` §3.1, stored at
`system/quorum/{quorum_id_hex}`) — and `quorum-publish` (§3.3) is the K-of-N-signed, supersede-chained
record that binds it to the **current** `published_handle`.

| Layer | Object | Changes when | Job in a shared reference |
|---|---|---|---|
| **Anchor** | `quorum_id` — a content hash | **never** (a `quorum-update` changes the *members*, not the id) | **trust** |
| **Handle** | controller / identifier peer-id | rotation, compromise recovery | **dial** |
| **Name** | `alice@entity-church` | at the registry's discretion | **reach + say aloud** |

> **So: `alice@entity-church` + the anchor.** The name resolves to a handle and transports; the anchor
> says *the handle you got must chain to this quorum*. Verification is `quorum_id` → hash the quorum
> entity → `signers` + `threshold` → K-of-N over `quorum-publish` → `published_handle`. **Every step is
> content-addressed or signed, so the bytes may come from the registry, the peer, or a mirror, and the
> registry is not trusted at any step — not even the first one.**

**This answers all three halves of the objection.**

1. ***"Why not just share the key?"*** — Because the key is not the identity; it is the identity's
   current address. Sharing it is sharing a snapshot, and there is no path from a retired key to its
   successor for someone who only ever held the key.
2. ***"Names change."*** — Yes, and that is the name's job. Under an anchor, a name changing is
   *survivable* (re-resolve) and a name **lying** is *impossible* (the chain check fails). The name is
   demoted to a hint, which is the only safe thing to do with a mutable identifier.
3. ***"We're still juggling keys."*** — You are juggling **one value per person, once, and it is the
   last one you will ever need for them.** Key-only juggling is re-juggling on every rotation, forever,
   out of band. That is not a smaller cost; it is an unbounded one.

---

## §1 What a key cannot do — three things, and the third is the one that kills it

**(a) A key does not tell you where the peer is.** `peer-id → transports` is a separate layer
(`EXTENSION-NETWORK` §6.5), and NETWORK §6.5.1a D1 is explicit that self-publication is RECOMMENDED,
not required, and that *"Consumers MUST NOT assume the self-published path exists."* A binding carries
`transports` (REGISTRY §3) precisely because a bare key is not a reachable address. **This is not
theory: Nostr shipped `nprofile` for exactly this reason** — a bech32 envelope carrying the pubkey
**plus relay hints**, because `npub` alone does not tell a client where to look (NIP-19, TLV type 1:
*"a relay in which the entity … is more likely to be found"*).

**(b) A key cannot be said aloud, printed, or introduced by a third party.** The third is the
under-rated one: `alice@entity-church` works when **the person sharing it does not know Alice's key**.
A key-only ecosystem has no way for someone to point you at a person they have not personally met.

**(c) A key cannot be rotated, and this is where key-only systems actually break.** Rotation is not
hypothetical hygiene — `EXTENSION-IDENTITY` §9.6 is the case: *"If the identifier's key (four-key) or
controller's key (three-key default) is compromised, the attacker can sign new agent certs claiming
arbitrary peers as authorized."* The remedy is a quorum-signed recovery. **A holder who has only the
old key has no way to learn that**, because the only thing that would tell them is signed by a set they
never pinned.

> **Measured on the deployed system that took the key-only branch: Nostr has no key-rotation or
> identity-recovery NIP at all.** `41.md` is absent from `nostr-protocol/nips` (HTTP 404), the index
> lists no NIP for migration or recovery, and its one delegation mechanism is struck through as
> ***"unrecommended: adds unnecessary burden for little gain"*** (NIP-26). **That is the honest price of
> "just share the key," paid by a network with real users.**

---

## §2 What a name cannot do

**A name is an assertion by an authority, and every authority can lie or be replaced.** That is not a
criticism of registries; it is the definition of a nickname in Stiegler's sense
(`GUIDE-RESOLUTION` §6.0): global + memorable, **not** securely unique.

Concretely, with a name and nothing else:

- **Substitution at the origin** is closed by REGISTRY §3 step 2a (`binding.name == norm`) — that one is
  handled, and it is handled at the body layer where it belongs.
- **Reassignment** is not. If `alice@entity-church` is later bound to a different key, a stored
  name-only reference silently resolves to a different person. Nothing is forged and no check fires,
  because the registry is doing exactly what a registry does.
- **Withholding a revocation** is not closable at all (`GUIDE-RESOLUTION` §7a): your bound is the TTL.

**So the name and the key fail in opposite directions**, which is the actual reason they are not
redundant: the key is the strongest introduction and the weakest long-term reference; the name is the
weakest introduction and the strongest long-term reference. A form carrying both is not belt-and-braces
— it is two different jobs that happen to travel in one string.

---

## §3 The word "anchor" is already taken, and by the other party

**REGISTRY has `trust_anchor` and `accepted_trust_anchors` (§2.4, §4), and they name the *issuer* — the
authority that says the thing. There is no field anywhere naming the *subject* — the identity the
binding is about.**

That asymmetry is the whole gap in one sentence. Resolution was designed to answer *"whom do I believe
when they tell me about a name?"*, and it answers it well. The operator's question is a different one:
***"what is the durable identity of the person on the other end, independent of anyone's say-so?"*** —
and the resolution layer has never had a slot for it.

`system/registry/binding` (§3) carries `name`, `kind`, `target_peer_id`, `transports`, `issued_at`,
`ttl`, `supersedes`, `issuer_attestation`, `metadata`. **`target_peer_id` is the handle, and §3 says so
in terms: *"the identity, NOT a content-hash."*** At the time that pin was written it was closing a
different confusion (self-certifying names are Base58 peer-ids, not `hex()` of a hash) and it is correct
for that. But it is also, read today, the exact statement of the gap: **the subject of a binding is a
rotating key and never a durable id.**

---

## §4 The landscape — two of the three neighbours built precisely this, and one did not

**This is the strongest external evidence in the analysis, and it was checked in the primary specs.**

| System | Durable anchor | Rotating operational key | Mutable name |
|---|---|---|---|
| **ATProto / `did:plc`** | the **DID**, *"derived from the hash of the first operation in the log, called the 'genesis' operation"* | `rotationKeys` — *"Control over a `did:plc` identity rests in a set of reconfigurable rotation key pairs"* | the handle (a domain) |
| **Matrix** | the **master cross-signing key** — *"used to identify themselves and to sign their other cross-signing keys"* | self-signing / user-signing — *"intended to be easily replaceable if they are compromised by re-issuing a new key signed by the user's master key"* | `@alice:server` |
| **Nostr** | — | — | NIP-05 (`alice@example.com`) |
| **Entity (this corpus)** | `quorum_id` — content hash of the quorum entity | controller / identifier / agent keys | `alice@entity-church` |

**`did:plc`'s anchor is structurally identical to ours and reached independently: a content hash of a
genesis record, with a reconfigurable signer set.** The differences run in our favour and are worth
being precise about rather than smug:

- **Ours is K-of-N; theirs is an ordered list of individual keys.** A single master key is exactly the
  object that cannot be recovered when it is lost — which is the failure `EXTENSION-IDENTITY` §7.1's
  cold-custody quorum exists to survive, and it is Matrix's known sharp edge too.
- **Ours needs no directory.** `did:plc`'s README calls its resolution *"a central directory server"* and
  says they *"expect to evolve … into something less centralized."* A `quorum_id` resolves against any
  store that has the bytes, because it is a content hash.
- **Nostr's row is empty in two columns, and that is the finding**, not an omission in the table.

**Per L18 this is corroboration and is cited after the derivation, not as it.** §0's result stands on
`EXTENSION-QUORUM` §3.1/§3.3 and `EXTENSION-IDENTITY` §4.3/§4.4/§9.4 alone; if all three neighbours had
done the opposite, nothing above §4 would change.

---

## §5 The seam — what is actually missing, stated so it can be built

**Everything the anchor needs exists. Nothing connects it to resolution.** Three specific absences:

1. **The binding has no subject-anchor field.** REGISTRY §3's shape has `metadata: <opaque object |
   null>`, and putting an anchor there would be exactly the *"`hints`-shaped opacity"* L17 forbids.
   A declared key or nothing.
2. **`ResolutionResult` carries no anchor**, so a resolver that wanted to check one has nothing to check
   it against and no place to report that it did.
3. **Nothing requires a contact introduced by name to cache `quorum-publish`.** `EXTENSION-IDENTITY`
   §9.4 is **fail-closed**: *"If no `quorum-publish` is cached … the recovery rotation MUST be
   rejected."* So a relationship formed through a registry — the day-one path, the one the demo runs —
   **acquires no ability to follow a recovery**, and the failure only becomes visible on the day
   recovery is needed, which is the day nothing can be fixed out of band.

**REGISTRY's own §12 already flags this and its answer is narrower than the question.** Q3 asks whether
existing bindings stay valid when a publisher rotates, answers *yes* on cert-chain grounds, and closes
with *"worth cross-checking against EXTENSION-IDENTITY in cross-impl review."* **That is a question
about the binding's validity. The operator's question is about the receiver's ability to follow**, and
those come apart: a binding can remain perfectly valid while every holder of it is stranded.

**And two normative sentences are already wrong on the rotation axis, in the two specs that restate
identity's model without pointing at it** — the L23-fourth-shape signature exactly:

- **REGISTRY §6.6:** *"When the target identity rotates per EXTENSION-IDENTITY §4.3/4.4, the local-name
  remains valid (it points at the stable `Public_X` identifier)."* §4.3/§4.4 **are** the handle-rotation
  kinds; they are the case in which the pointed-at key stops being the handle. The sentence is true of
  *agent* rotation and cites the sections for the other one.
- **NETWORK §6.4:** cites `EXTENSION-IDENTITY §5.6 rotate_operator` and `§5.9 revoke_peer`. **Neither
  exists** — IDENTITY §5 is *Path conventions*, and `GUIDE-IDENTITY` records that *"there is no
  dedicated `rotate_operator`"* in v3's unified attestation model.
- **`Public_X` is retired vocabulary.** `GUIDE-IDENTITY`'s own history says the guide was rewritten to
  lead with the three-key default, *"op = contact-face; **no separate `Public_alice`**."* It survives in
  two normative specs (REGISTRY §6.6, NETWORK §6.4) and throughout `GUIDE-GROUP`.

---

## §6 The connection nobody had drawn: this is D-11 on the time axis

**`D-11` (multi-device) is on the board as *"branch 3 (per-device ids) fractures other people's
published follow lists and no later fix repairs it."*** Rotation is the same fracture with time
substituted for space.

`PROPOSAL-APP-CONVENTION-FEED` §2.3 pins `author: peer-id, ; MUST equal the authoring namespace`, and
its reference atom (§2.2) carries `peer: peer-id`. **So every stored reference, every follow subject,
and every durable `(peer, path)` in
`EXPLORATION-THE-DURABLE-REFERENCE-…` inherits the stability of whatever peer-id publishes.** That
exploration's §6 open list does not contain rotation, and its whole point is that
*"the hash of the link stays the same — it's this peer, this tree path."* **The durability of the path
half was designed; the durability of the peer half was assumed.**

> **One question therefore governs three items: which identifier does a person publish under, and is it
> allowed to change?** D-11 asks it across devices, this asks it across time, and the reference atom
> spends the answer. **They should be ruled together or not at all** — and note that the anchor makes
> D-11's branch 3 survivable for the first time, because per-device ids that all chain to one quorum are
> no longer a fracture, they are a fan-out under a stable root.

---

## §6a How this fits identity — the operator's two objections, both answered by the same fact

**Objection 1: *"I was always under the impression that identity, through the controller, comes back to
a single public key."* — That is correct, and §0 does not change it.** `EXTENSION-IDENTITY` §2.3:
*"In the three-key default, the controller also serves as the identifier; **the controller's key IS the
contact-side handle**."* Day to day, an identity **is** one public key. The quorum is not a second
identifier competing with it — **it is the thing that says which set of keys may name a new controller
key**, and you touch it only when the controller must be replaced.

**Objection 2: *"A content hash is just a content hash. That doesn't tell you anything."* — Correct,
and it is the same thing that is true of a peer-id.** A V7 §1.5 peer-id is an identity-multihash **of
the pubkey**: it tells you nothing either, until you hold the preimage — and then it certifies it with
no trust in whoever handed it to you. **`quorum_id` is to a signer set what a peer-id is to a pubkey.**
The preimage is small and boring, which is the point:

```
system/quorum := { signers: [<peer_hash>, ...], threshold: <K>, signer_resolution?, name?, metadata? }
```

That is the whole entity (`EXTENSION-QUORUM` §3.1), it is **structural and not itself signed**, and
`quorum_id` is its content hash. Fetch it from anywhere, hash it, and you know exactly which N keys and
which K — nothing else in the system had to be trusted to tell you.

### §6a.1 The correction: `quorum_id` is stable across membership change, and that is what makes the upgrade path work

**A `quorum-update` does not mint a new quorum.** The entity is never rewritten; updates are
attestations stored **under** the original id at `system/quorum/{quorum_id_hex}/event/{hash_hex}`, and
§4's `current_signer_set(quorum_id)` reads the entity's `signers`/`threshold` and then walks the
supersedes chain of updates to the live head. **So constituents can be added, removed, and K raised,
for the life of the identity, with `quorum_id` unmoved** — which is precisely `did:plc`'s
genesis-operation property (§4), reached here through a different door.

### §6a.2 The upgrade path exists, is peer-id-stable, and still shifts the handle

**This section was drafted claiming the V7-only → identity upgrade changes your published identifier.
Three documents say otherwise and the draft was wrong.** `SDK-IDENTITY-INFRASTRUCTURE` §8,
`SDK-OPERATIONS` and `GUIDE-PERSISTENCE` all state that `BootstrapFromExistingKeypair` leaves
*"the peer's keypair unchanged (**peer_id stable**)."* **True, and it is about the runtime peer.**

**The two sentences answer different questions and both are correct:**

| | Says | Subject |
|---|---|---|
| §8.1 helper contract | *"Reuses existing peer keypair as **the agent** … generates fresh quorum + controller"* | the **key graph** |
| §8 migration prose | *"the peer's keypair is unchanged (peer_id stable)"* | the **runtime peer** |

**Put together: your peer-id does not move, and it stops being your handle.** Before the upgrade,
V7-only means *"peer's keypair = peer's identity"* (§1.1 rung 1). After it, that key is an **agent** —
never handle-bearing under §4.2's derivation table — and the handle is a freshly generated controller,
which is what `quorum-publish` is seeded with as `published_handle` (§8.1's `BootstrapNewIdentity`).

**The corpus states the consequence in four words, in three documents, with no elaboration anywhere:
*"No retroactive migration of contacts."*** Nothing breaks on upgrade day — the old key is live as an
agent and still answers. **The failure is deferred to the first agent retirement**, which is the
*typical* flow and not an incident (§4.5, §9.5): every pre-upgrade contact holds a reference to a
device key, never learned the controller or the quorum, and has no path to the replacement.

> **So the operator's requirement — *"fall back and additively upgrade to identity"* — is satisfied for
> the upgrader and not for the people who already reference them.** The recommendation that follows is
> cheap and mechanical: **if you expect to be referenced, start at rung 2 (1-of-1 quorum), not rung 1.**
> It costs one keypair, it gives you a `quorum_id` from first publication, and by §6a.1 you can grow it
> into full K-of-N recovery later without moving anything anyone else holds. **Rung 2 → rung 3+ is
> genuinely additive; rung 1 → rung 2 is the only lossy step in the progression, and it is lossy for
> other people rather than for you** — which is exactly why it is easy to miss at the moment you choose
> where to start, and why `GUIDE-IDENTITY` §2.1 currently says *"most users start here"* about rung 1
> without the consequence attached.

**The alternative was available and is not recorded anywhere.** Nothing in `EXTENSION-IDENTITY` requires
a controller key to be freshly generated — §2.3 establishes the function by cert, not by provenance — so
`BootstrapFromExistingKeypair` **could** have promoted the existing key to *controller* and preserved the
handle outright. There is a real argument against it (a key that has been a hot runtime cap-signer sits
uneasily with §9.2's confinement of controller signatures to internal flows), and that is the point:
**a decision with cross-peer consequences was made in a one-line SDK helper comment, with no rationale
and no alternative recorded.** Preferring rung 2 avoids needing to reopen it.

---

## §7 The honest costs of the anchor

**Stated because a recommendation with no cost column is a sales pitch.**

1. **Not every identity has one.** `EXTENSION-IDENTITY` §11.1 (V7-only) has no quorum at all, and §11.2
   (1-of-1) has one with no recovery. **This is a consistency, not a hole:** you can only *need* an
   anchor if you can rotate, and you can only rotate if you have a quorum. The anchor's availability
   tracks rotation's availability exactly.
2. **`quorum-publish` is optional** (§9.4 records the opt-out). Opting out of publication is opting out
   of being followed across a recovery — already true today, and the anchor does not add the cost, it
   makes it legible.
3. **Publication discloses `signers` and `threshold`** (QUORUM §3.3's shape). That names the exact key
   set an attacker must compromise and how many of them. This is the real reason the opt-out exists.
4. **An anchor is a perfect correlator, by construction** — the same property that makes it durable
   makes one identity linkable across every context it appears in. **Zooko again, on a third axis.** The
   answer is per-persona quorums, which §11.5 (multi-binding) already permits; the answer is *not* to
   weaken the anchor.
5. **It is not shorter.** A content hash is ~44 Base58 characters, the same as a key. **The gain is not
   brevity, it is finality.**

---

## §8 What this leaves open

1. **The share form's semantics are unwritten, and that is what produced the confusion.** `GUIDE-RESOLUTION`
   §6.3 lands `name@X` and REGISTRY §4.1 step 1a decodes `X`, but **nothing says whether a pin is a
   permanent equality, a first-contact assertion, or a chain-root check** — and those are three
   different protocols with the same syntax. **This is the smallest, highest-value item here.**
2. **Where the anchor lives** — a declared field on the binding, a field on `ResolutionResult`, or purely
   receiver-side in the link. Receiver-side is the cheapest and needs no registry change; the binding
   field is what lets a registry *offer* the anchor to someone who has only a name. They are not
   exclusive.
3. **Whether the resolve path should acquire `quorum-publish`.** §5's third absence. This is the one with
   a real cost (an extra fetch on first contact) and a real payoff (registry-independent thereafter).
4. **The three stale sentences in §5** are not open questions — they are corrections, owed in the session
   that lands anything here, per L9.
5. **Nobody has asked for the anchor.** No seat has hit it; the day-one demo does not need it. **L26 cuts
   toward letting the seats discover by building.** What argues for moving on item 1 now is that the
   reference atom is under review *right now* and two implementations are about to ship against it.

---

## §9 Sources

Primary, read this session:
[did:plc specification v0.1](https://web.plc.directory/spec/v0.1/did-plc) ·
[ATProto — DID specification](https://atproto.com/specs/did) ·
[Matrix — MSC1756, cross-signing](https://github.com/matrix-org/matrix-spec-proposals/blob/main/proposals/1756-cross-signing.md) ·
[Nostr NIP-19 — bech32 entities (`nprofile`)](https://github.com/nostr-protocol/nips/blob/master/19.md) ·
[Nostr NIP index — no NIP-41; NIP-26 struck as unrecommended](https://github.com/nostr-protocol/nips)

In-corpus: `EXTENSION-QUORUM` §3.1, §3.3 · `EXTENSION-IDENTITY` §4.2, §4.3, §4.4, §7.1, §7.2, §9.4,
§9.6, §11.1–§11.5 · `EXTENSION-REGISTRY` §3, §6.6, §12 Q3 · `EXTENSION-NETWORK` §6.4, §6.5.1a ·
`GUIDE-RESOLUTION` §6.0, §6.1a, §6.3, §7a · `PROPOSAL-APP-CONVENTION-FEED` §2.2, §2.3 ·
`EXPLORATION-THE-DURABLE-REFERENCE-THE-ANCHOR-AND-THE-THREE-WAY-READ` §0, §3, §6.
