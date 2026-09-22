# PROPOSAL — the `system/peer` hashable basis is `(public_key, key_type)`, and five sites still carry the retired shape

**Status:** **IMPLEMENTED (2026-09-09)** — folded at `ENTITY-CORE-PROTOCOL` **0.8.2.15** and
`EXTENSION-ROLE` **v2.1**. Kept as the reference proposal.

> **The fold found a SIXTH home, and the enumeration below says five.** The post-fold sweep —
> `grep -rn "peer_id, public_key, key_type"` across both corpora — returned
> `guides/GUIDE-RESTART-AND-PERSISTENCE.md` §*What survives a restart*, which describes the peer's
> own identity entity as *"deterministic from `{peer_id, public_key, key_type}`"*. It shares none of
> §3's vocabulary — no `basis`, no `preimage`, no `content_hash` — and it is in a **guide**, so
> neither the subject enumeration nor a scan of normative homes reached it. Corrected in the same
> pass. **The transferable half: enumerate by subject before the fold, then re-grep the literal
> string after it.** The enumeration finds the homes that argue about the rule; the grep finds the
> ones that merely use it, and this corpus's guides are full of the second kind.
**Target:** `ENTITY-CORE-PROTOCOL.md` §4.5a item 1a and §4.6's `peer_entity` pseudocode — correct the
data shape to the one §3.5's type system already MUST-NOTs its way to. Consequential edits at three
further sites that restate the same shape: `ENTITY-CORE-PROTOCOL`'s `specs/test-vectors/crypto-agility/SEEDS.md`,
`SPECIFICATION-FORMAT.md` §8.4.6's second worked example, and **`EXTENSION-ROLE.md` §3.6's `grantee`
encoding rule**, which states the preimage explicitly.
**Raised by:** conformance measurement of the connect handshake across a population of
independently built peers, which found §4.6's pseudocode and §3.5's type definition giving
different answers for the same entity.

---

## 1. The contradiction — one entity, two hashable bases, both normative

**`ENTITY-CORE-PROTOCOL` §3.5 defines the type, and forbids the field in the basis:**

```
system/peer := {                                  ; v7.65 — peer_id is NOT a hashable field
  fields: {
    public_key: {type_ref: "primitive/bytes"}
    key_type:   {type_ref: "primitive/string"}
  }
}

; P×I primitive discipline: peer_id MUST NOT appear in
; the system/peer entity's hashable basis. content_hash(system/peer) is a pure
; function of (public_key, key_type) and is invariant under wire-form peer_id
; choice.
```

**§1's ECF envelope example agrees, and annotates the change that made it so:**

```
type: "system/peer",
data: { public_key, key_type },              ; v7.65: peer_id exits hashable basis
```

**§3.5's prose agrees, and names the consumers that inherit the property:**

> **Under v7.65, `content_hash(system/peer)` is a pure function of `(public_key, key_type)` and is
> invariant under wire-form `peer_id` choice** — caps, signatures, attestations, and
> identity-bindings inherit this invariance.

**§4.6's pseudocode disagrees.** It is the block an implementer builds the handshake from:

```
peer_entity = Entity {
  type: "system/peer",
  data: { peer_id, public_key, key_type }
}
peer_hash = content_hash(peer_entity)
```

**§4.5a item 1a disagrees, in the clause that justifies pinning the entity to the floor:**

> It is the one entity on this surface with **no author-chosen content**: its data is
> `{peer_id, public_key, key_type}`, wholly recoverable from the public peer-id…

## 2. Why this is not an editorial mismatch

`peer_hash` from §4.6 is what goes into `signature.signer`. The same value is a capability's
`grantee` and `granter` (§3.6), and the `{peer_id_hex}` path segment (`SPECIFICATION-FORMAT` §8.4.6).
**Two implementers, each conformant to a section they read, compute different bytes for one
identity.** §5.2's `signer == author` and `grantee == author` equalities are byte-wise (§5.3), so
they fail across that pair — on a correct signature, with a correct key, at handshake.

**The retired shape also defeats the property the sections that carry it are asserting.** §4.5a
item 1a pins the entity to the ECFv1-SHA-256 floor so that the identity hash is *"the same bytes on
every connection in the network."* A basis containing `peer_id` cannot have that property: §4.5's
wire-acceptance carve-out lets one key be presented in more than one `peer_id` wire form
(identity-multihash, or the SHA-256 form at `hash_type=0x01`), so the two forms hash to two
identities — the second-form manufacture that §4.5a **item 4** prohibits in the same section. Item
1a's argument is sound and its field list contradicts its conclusion; only the field list is wrong.

**The direction is not a new decision.** §3.5 carries a `MUST NOT`, §1 and §3.5's prose both cite
the revision that moved the field out, and no later revision moves it back. The two disagreeing
sites are pre-v7.65 text that survived the change. This proposal restores the document's own rule;
it does not choose between two live designs.

> **The reverse reading, stated so it is reviewable.** §4.5a item 1a is marked v7.77 and the type
> system is v7.65, so *newer text* carries the retired shape. That ordering is why this is a
> proposal rather than a wording sweep. It does not survive contact with the sections: item 1a's
> subject is the **format** the entity is authored under, not its fields; its field list is
> incidental to its rule; and taking it as authoritative would silently repeal a `MUST NOT` in the
> type system, invalidate §3.5's invariance sentence and §1's annotated example, and — as above —
> falsify item 1a's own conclusion. A repeal of a wire-core `MUST NOT` does not arrive as a
> subordinate clause in a paragraph about hash formats.

## 3. Every home of the rule — five, and two are outside the core spec

The finding named two. Enumerated by the rule's **subject** rather than by the section that raised
it, the retired shape is stated at five sites in two corpora:

| # | Site | What it says | Effect of leaving it |
|---|---|---|---|
| 1 | `ENTITY-CORE-PROTOCOL` §4.5a item 1a | *"its data is `{peer_id, public_key, key_type}`"* | The justification for the floor pin contradicts the pin's own conclusion |
| 2 | `ENTITY-CORE-PROTOCOL` §4.6 | `data: { peer_id, public_key, key_type }` in `peer_entity` | **The handshake implementer's copy.** Wrong `signature.signer` at connect |
| 3 | `ENTITY-CORE-PROTOCOL` `specs/test-vectors/crypto-agility/SEEDS.md` | restates item 1a's basis verbatim | A vector seed carrying the retired preimage |
| 4 | `SPECIFICATION-FORMAT` §8.4.6, second worked example | *"`system/peer`'s data is `{peer_id, public_key, key_type}` — every field of it is recoverable from the peer-id"* | The derive-to-meet ruling's stated basis |
| 5 | **`EXTENSION-ROLE` §3.6, `grantee` field encoding (normative)** | *"the peer-entity content_hash is `0x00 \|\| SHA256(ECF({type: "system/peer", data: {peer_id, public_key, key_type}}))`. Different preimages, different hashes."* | **An explicit normative preimage.** A role implementation computes `grantee` from this and never matches a core-derived one |

**Site 5 is the one a fix confined to the core spec would leave standing**, and it is the most
directly actionable of the five: it spells the preimage out byte-wise, in a normative paragraph,
for exactly the field whose equality breaks.

**Sites 4 and 5's surrounding arguments are unaffected and stay.** Both turn on *"every field is
recoverable from the public peer-id"* — which holds under the correct basis as well, since a
peer-id decodes to `(public_key, key_type, hash_type)` (§7.4). The derive-to-meet ruling, the
`{peer_id_hex}` floor pin and the `grantee`-is-not-the-peer-id-digest distinction all survive
verbatim. Only the field list changes.

## 4. The edits

**`ENTITY-CORE-PROTOCOL.md` — §4.5a item 1a.** Replace *"its data is `{peer_id, public_key,
key_type}`"* with *"its data is `{public_key, key_type}` (§3.5)"*, and keep the recoverability
clause, which holds unchanged.

**`ENTITY-CORE-PROTOCOL.md` — §4.6.** In the `peer_entity` construction only:

```
peer_entity = Entity {
  type: "system/peer",
  data: { public_key, key_type }      ; §3.5 — peer_id is not in the hashable basis
}
```

**The `authenticate_entity` block immediately above it is a different type and is NOT touched.**
`system/protocol/connect/authenticate` legitimately carries `{peer_id, public_key, key_type, nonce}`
— it is the challenge payload, not the identity entity, and `peer_id` there is a signed assertion of
the wire form being presented. A sweep that strips the field from both blocks is the failure mode
this note exists to prevent.

**`specs/test-vectors/crypto-agility/SEEDS.md`** — same substitution in the restatement of item 1a.

**`SPECIFICATION-FORMAT.md` §8.4.6** — same substitution in the second worked example; the ruling,
its date marker and its reasoning are unchanged.

**`EXTENSION-ROLE.md` §3.6** — correct the preimage to
`0x00 || SHA256(ECF({type: "system/peer", data: {public_key, key_type}}))`. The paragraph's point —
that this is a different preimage from the peer-id digest `0x00 || SHA256(public_key)` — is
**unchanged and still true**, and is worth keeping precisely because the two are now closer
together and easier to confuse.

## 5. Conformance

**No new requirement is created.** §3.5's `MUST NOT` is the requirement and it is already landed;
what changes is that four consumers of it stop contradicting it. Existing conformance rows are
unaffected in wording.

**What an implementation must check.** A peer that built its handshake from §4.6's pseudocode
computes `content_hash({peer_id, public_key, key_type})` for `signature.signer`. Against a peer built
from §3.5 the `signer == author` equality fails and the connection is refused at authenticate — so
the divergence is **visible at connect, not silent**, which bounds the exposure. The check is a
single derivation compared against a known vector, and it belongs in the connect category of the
cross-implementation check set.

**This is the class of claim prose review does not settle.** Two conformant readings of one document
producing different bytes at a peer boundary is not established as fixed until an implementation pair
exercises it. A check set is produced by implementations running against each other; this proposal
names the case such a check must discriminate, rather than the bytes it must assert.

## 6. Open items

1. **Which reading did each implementation build?** Unmeasured here — this proposal is written from
   the specification alone, and no tree was read for it. The correction stands either way, because
   one of the two sites is wrong whatever the field contains; but the *cost* of the fold depends on
   the answer, and it is one grep of the `peer_entity` construction in each implementation.
2. **A version bump is warranted and is the arch-managed fourth component**, since core text moves
   and consumers need a visible signal.
