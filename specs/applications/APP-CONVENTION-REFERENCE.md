# APP-CONVENTION-REFERENCE — the reference atom and its string form — v0.1 DRAFT

**Version**: 0.1
**Status**: Draft
**Domain:** `applications/` (fourth member — see `CHARTER.md`). **Charter class:** FORMAT-only.
**Depends:** `ENTITY-CORE-PROTOCOL.md` §1.2 (content hash, self-describing) · §1.4 (URI and path
model) · §1.5 (`PeerID`) · `APP-CONVENTION-EMBED.md` §3 (the pointer slots that adopt this atom) ·
RFC 3986 (URI generic syntax).

> **What this document is.** One shape for *"this points at that"*, and one string form for it. Every
> other member of this domain needs to point at something — a page, an image, an entry, a blob — and
> before this document each defined its own way of doing so. This convention adds **no** kernel
> feature, **no** required SDK surface and **no** wire form (charter #2, #3). It is a format, and the
> only thing it asks of the substrate is content addressing, which already exists.

> **Foundational member.** The other conventions in this domain **import** the atoms defined in §2.1
> rather than restating them. Where a member previously defined `content-hash`, `peer-id` or
> `tree-path` locally, this document is the single home.

---

## 1. What a reference carries

A reference has four terms, and only the first two are ever required:

| Term | Slot | Why it is there |
|---|---|---|
| **who** | `peer` | The publisher. **A reference is routable if and only if it names one** — bytes alone name no holder you can go and ask |
| **what** | `hash` or `path` | The identity of the thing. Which of the two it is *is the intent*, and §2.2 is about why that is not a formatting detail |
| **which part** | `at` | Optional. Absent means the whole entity |
| **where to look first** | `via` | Optional, **advisory, and droppable**. §2.3 bounds it |

### 1.1 The two intents are different questions, not two spellings of one

- A **pinned** reference names *these exact bytes*. It is self-verifying: anyone holding them can
  satisfy it, and the answer can never change.
- A **live** reference names *whatever is at this address now*. Only the publisher is authoritative
  for it, and the answer is expected to change.

**These are the two hops of the path model** (`ENTITY-CORE-PROTOCOL` §1.4: a path resolves to a hash;
a hash resolves to bytes) exposed as two entry points. A consumer that collapses them cannot express
*"the version I read"* and *"the current version"* as different things, and both are needed.

---

## 2. The atom

### 2.1 Shape

```cddl
; ---- shared atoms — THIS DOCUMENT IS THE SINGLE HOME (charter, Members) -------
content-hash = bstr   ; self-describing (format_code, digest) per ENTITY-CORE-PROTOCOL §1.2.
                      ; The leading varint is the content_hash_format and THE DIGEST LENGTH
                      ; FOLLOWS THE CODE. NOT fixed-width (charter #6). SHA-256 -> 33 B is one
                      ; instance; SHA-384, BLAKE3 and future codes are equally valid.
                      ; Unknown code -> unsupported_content_hash_format.
peer-id      = tstr   ; a PeerID in the canonical Base58 form of ENTITY-CORE-PROTOCOL §1.5.
                      ; A TEXT string, not bytes -- see §2.1.1.
tree-path    = tstr   ; absolute or peer-relative per ENTITY-CORE-PROTOCOL §1.4

; ---- the atom ----------------------------------------------------------------
entity-ref = pinned-ref / live-ref     ; TAGGED. There is no untagged form.

pinned-ref = {
  tag:   "pin",
  peer:  peer-id,                      ; WHO published it
  hash:  content-hash,                 ; WHAT it is -- identity and expectation coincide
  ? at:  anchor,                       ; which PART. Absent = the whole entity
  ? via: [* hint]                      ; ADVISORY. Ordered, descending confidence
}

live-ref = {
  tag:    "live",
  peer:   peer-id,                     ; WHO publishes there
  path:   tree-path,                   ; the address of record -- authoritative
  ? seen: content-hash,                ; what the linker saw -- an EXPECTATION only
  ? at:   anchor,
  ? via:  [* hint]
}

anchor = { field: [* tstr] }           ; a field path within the entity. See §2.4
hint   = { tag: "origin" / "mirror" / "peer" / "path", value: tstr }
```

#### 2.1.1 `peer-id` is text, and this is the one place it is stated

**A `PeerID` is defined as `Base58(varint(key_type) || varint(hash_type) || digest)`**
(`ENTITY-CORE-PROTOCOL` §1.5) — the Base58 encoding is part of the definition, not a display
convenience — and §1.4 uses it directly as a tree-path segment. **So a peer id is a text string at
every layer that names one**, and a convention that types it as `bstr` names a different thing.

**[MUST]** `peer-id` is a `tstr` carrying the canonical Base58 form. An implementation MUST NOT
substitute the raw `(key_type, hash_type, digest)` byte sequence for it.

*This is called out because it is easy to reach the wrong answer by analogy: `content-hash` is a
`bstr` and is self-describing, and a peer id is also self-describing, so the two look like the same
kind of term. They are not — one is bytes the system hashes, the other is an identifier the system
spells.*

### 2.2 The discriminator is a tag

**A reader tells a pinned reference from a live one by reading `tag`, not by observing which of
`hash` / `path` is present.**

The requirement a discriminator has to meet is that *a reader can always tell*. A tag meets it more
directly than a field-presence rule does — the discriminator is a value rather than an inference —
and it costs nothing:

| | field-presence rule | tag |
|---|---|---|
| A reader tells them apart by | observing which field is present, and rejecting an atom carrying both | reading one field |
| A malformed atom is | detectable only by a reader that implements the presence rule | a tag mismatch, at the first field |
| A third intent later | needs a third presence rule interacting with the first two | is a third tag |

**What a tag does NOT license.** It does not make the hash optional on one shape. **An atom with a
`"pin"` tag and no `hash`, or a `"live"` tag and no `path`, is malformed** and MUST be refused. Were
the hash merely optional, a reference arriving without one would be indistinguishable between *the
author wants the live version*, *the author's implementation did not populate it*, and *the author
only ever had an address* — one intent and two defects, with nothing to separate them. **The two
shapes exist to keep those apart, and the tag is how a reader reaches the right one first.**

### 2.3 `via` — advisory, ordered, and droppable or it is not a hint

> **[MUST]** A reader that ignores every `via` hint MUST reach the same answer as one that uses them,
> or fail. A hint MUST NOT be the only path to a correct resolution, MUST NOT be signed, and MUST NOT
> be treated as authority for anything.

**Why the slot exists.** A reader holding a peer id it cannot resolve to a transport has no next
step; `via` gives it a starting point. **Why it must be droppable:** a hint that becomes load-bearing
is a link that rots, because a hint records where something was, and that is exactly the term with
the shortest life. The rule above is what keeps a stale `via` a slow path rather than a broken one.

Hints are **ordered, descending confidence**. A reader MAY try them in order, MAY try none, and MUST
NOT treat exhaustion of the list as proof the referent does not exist.

**The four hint kinds all answer one question — *where might I find this?***

| `tag` | `value` | Says |
|---|---|---|
| `origin` | a content origin | where the publisher serves bytes from |
| `mirror` | a content origin | somewhere else that had them |
| `peer` | a peer id | somebody who is likely to hold them |
| `path` | a tree path | where in the publisher's tree it was placed |

> **`path` is a hint and is only ever a hint, and this is the one that is easy to get wrong.** On a
> **pinned** reference the hash is the identity, so a path can only ever say *where it was put*. A
> reader **MUST NOT** treat a `404` at a `path` hint as evidence the referent does not exist — the
> bytes validate against the hash from any source at all, so failing at one location proves nothing.
>
> **This is why a path hint belongs in `via` and not in a `path` field beside `hash`.** A slot named
> `path` sitting next to `hash` is the same field name that is *authoritative* on a live reference,
> and one field name meaning **the address of record** in one shape and **a guess** in the other is a
> discriminator a reader has to know the shape to interpret. **One field, one job:** `path` is
> authoritative only where it is the identity term.

**An unknown `hint.tag` is ignored, not an error** — the same MUST-ignore discipline the substrate
applies to unknown fields. A reader that refuses an atom because it carries a hint kind it does not
know has made an advisory term load-bearing, which is the failure this section exists to prevent.

### 2.4 `at` — the anchor slot

`? at: anchor` where `anchor = { field: [* tstr] }` — a path of field names into the referenced
entity. **Absent means the whole entity, so nothing changes for a consumer that does not use it.**

It is declared now because the alternative is minting a second atom later for *"the same thing, but
part of it"*, and because a field path is the one part of a reference that survives a change of
bytes: the hash changes, the field path does not.

**What it deliberately does not do:** it does not name a location *inside a rendered body*. That
needs a name slot the content conventions do not yet mint, and no consumer has asked for one. The tag
structure means such a rung can be added as another `anchor` variant without disturbing this one.

> **`field` is an opaque sequence of names in v0.1.** The core type system's address primitives are
> content hash, tree path, type name and peer id; **an intra-entity field path is none of them**, and
> the scope grammars that exist are tree-path patterns rather than field paths. **A slot with no
> consumer does not mint a grammar** — when one arrives, the grammar lands with it.

---

## 3. The string form

A nav target, a link in a document body, a pasted address and a scanned code are all **strings**. The
atom above is what a producer stores; this section is what a human types and what a document carries.

### 3.1 Grammar

```abnf
entity-ref-uri = "entity+ref://" peer-id path-abempty [ "?" query ] [ "#" fragment ]
```

Mapped onto RFC 3986 §3:

| RFC 3986 component | Carries |
|---|---|
| **scheme** | `entity+ref` — a valid scheme name per §3.1 (`ALPHA *( ALPHA / DIGIT / "+" / "-" / "." )`) |
| **authority** | the **peer id**, and nothing else. No userinfo, no port |
| **path** | for a live reference, the tree path. For a pinned reference, **empty** |
| **query** | the non-identity terms: `hash`, `seen`, `via` |
| **fragment** | the `at` anchor |

```
live:   entity+ref://{peer}/{tree-path}[?seen={hash}][&via=...][#{anchor}]
pinned: entity+ref://{peer}/?hash={hash}[&via=...][#{anchor}]
```

**Which component carries what is a design decision and not a formatting one: the identity terms go
in the address, the advisory terms go in the query.** A pinned reference's identity is its hash, so
its path is empty and the hash is a required query parameter. That reads oddly and it is the honest
encoding — a pinned reference genuinely has no path-shaped identity. Putting the hash in the path
instead would make pinned and live references structurally indistinguishable to a generic URI parser
and hand the discriminator back to a scan.

**The tag is recoverable from the string without a lookup table:** `hash` present ⇒ `pin`; path
non-empty ⇒ `live`.

> **[MUST]** A string carrying **both** a `hash` parameter and a non-empty path, or **neither**, is
> malformed and MUST be refused rather than resolved.

**The authority component is mandatory.** `entity+ref:` without `//` and a peer id is malformed. The
scheme is spelled with `//` because it has an authority, and the authority is the term that makes the
reference routable at all.

### 3.2 Round-tripping is normative

> **[MUST]** The string form is **lossless in both directions** for every atom expressible in §2.1.
> Parsing a conformant string and re-serializing it MUST produce a byte-identical string; serializing
> an atom and re-parsing it MUST produce an equal atom.

This is the same obligation the embed convention places on a directive string and the entity it
projects, for the same reason: a re-serializer that is not lossless silently rewrites other people's
references.

### 3.3 Normalization

Percent-encoding, case and normalization are where string forms fail in practice.

- **The scheme is case-insensitive on parse and MUST be emitted lowercase** (RFC 3986 §3.1, §6.2.2.1).
- **The authority is a peer id and is CASE-SENSITIVE.** It is a Base58 identifier, **not a DNS name**,
  so RFC 3986 §6.2.2.1's host-normalization MUST NOT be applied to it. **This is the single most
  likely implementation error**, because general-purpose URL libraries lowercase the host by default,
  and a peer id that survives such a parser names a different peer — failing as a clean `404` at a
  well-formed address rather than as a parse error.
- **Path segments are compared after percent-decoding** (§6.2.2.2). An implementation MUST NOT
  compare raw.
- **Reserved characters within a path segment MUST be percent-encoded** (§2.1). `/` is the segment
  delimiter and is never encoded when it is one.
- **No empty-versus-absent conflation:** an absent query parameter and one present with an empty
  value are different, and the second is malformed.
- **No dot-segment removal** (§6.2.2.3 is not applied). **In the absolute form, a path containing a
  `.` or `..` segment is malformed and MUST be refused, not resolved** — a tree path is not a
  filesystem path and `..` has no meaning in it. A resolver that borrows filesystem semantics here
  produces a different, well-formed, wrong address.

> **This refusal is scoped to the absolute form, and the scoping is load-bearing.** §3.4's relative
> form is directory-relative, so `..` is both meaningful and expected *there*. **Relative resolution
> consumes the dot segments; what it produces is an absolute reference in which none survive.** A
> reader that applies the refusal to an unresolved relative string rejects ordinary correct links.

### 3.4 The relative form, which is the common case

The grammar above is the absolute form. **The overwhelmingly common case is a bare string in a
document body**, and it needs a base.

> **The base is the referring entity's own location.** A link with no scheme is resolved against it:
> a leading `/` is **root-absolute within the current site**; anything else is **directory-relative to
> the current page**. A trailing `.md` or `.markdown` is stripped.
>
> Producers of application-generated links — navigation, breadcrumbs, generated index pages —
> **SHOULD** emit root-absolute form, so that a link resolves identically from whatever page it is
> rendered on.

**`site:` is the same-peer short form.** `site:{site-id}/{page}` names a page in a different site on
the **implied** authority — the current peer. It is **opaque**: no `//`, no authority component,
because it has none to carry. It is not a competitor to `entity+ref://`; **it is the rung between a
bare relative link and a fully-qualified reference**, and it exists because the same-peer case is
common enough to deserve a spelling that does not require knowing your own peer id.

**A string that cannot be parsed as any of these forms is treated as leaving the system** — an
external link — rather than guessed at. Refusal is the correct behaviour; a tolerant re-anchoring
scan produces a well-formed wrong location and cannot report that it did.

---

## 4. What a producer emits and what a consumer accepts

`entity://` is the **wire dispatch scheme** (`ENTITY-CORE-PROTOCOL` §1.4): `entity://{peer_id}/{path}`
addressed to a handler. **Following a link is not dispatching to a handler**, and a single string form
meaning both is ambiguous by construction.

- **[MUST]** A conformant producer emits `entity+ref://` in a link position.
- **[MUST NOT]** A producer that emits the atom at all MUST NOT emit `entity://` in a link position.
- **[SHOULD]** A consumer SHOULD continue to resolve `entity://` in a link position, and **SHOULD
  surface that it did so** — the same *name your own provenance* obligation a consumer has when a
  live reference resolves to something other than what the linker saw.

**The asymmetry is the mechanism**: a MUST on the producer stops the population of ambiguous strings
growing; a SHOULD on the consumer keeps every already-published document resolving. **No flag day is
declared here.** When the ambiguous form stops being accepted is a decision for the parties holding
the corpus of published documents, not for this specification.

---

## 5. Floor (charter #4)

**An atom with no `via` and no `at` is the whole floor.** A peer that resolves only pinned references,
only within its own namespace, and ignores every hint, is a **valid participant** — not a degraded
one. Capability adds:

| Capability | Adds |
|---|---|
| resolve a live reference | the address-of-record hop; needs a tree read |
| follow `via` | a starting point when a peer id resolves to no transport. **Never required** (§2.3) |
| honour `at` | addressing part of an entity instead of all of it |
| emit the string form | interoperating with anything a human types |

Nothing in this convention requires a network, a registry, or any extension.

---

## 6. Conformance

**Requirement id prefix:** `REF`

### 6.1 Requirements

| id | Requirement | Level | § |
|---|---|---|---|
| `REF-R1` | Carry a `tag` of `"pin"` or `"live"`; there is no untagged atom | MUST | §2.1 |
| `REF-R2` | Refuse an atom whose tag and required identity term disagree (`"pin"` without `hash`, `"live"` without `path`) | MUST | §2.2 |
| `REF-R3` | Encode `peer-id` as a text string in canonical Base58 form, never as the raw digest bytes | MUST | §2.1.1 |
| `REF-R4` | Encode `content-hash` as a self-describing `(format_code, digest)` byte string of no fixed width | MUST | §2.1 |
| `REF-R5` | Reach the same answer ignoring every `via` hint as when using them, or fail | MUST | §2.3 |
| `REF-R6` | Treat a `via` hint as signed, authoritative, or as the only path to a resolution | MUST NOT | §2.3 |
| `REF-R7` | Ignore a `via` hint carrying an unknown `tag` rather than refusing the atom | MUST | §2.3 |
| `REF-R8` | Round-trip atom → string → atom and string → atom → string losslessly | MUST | §3.2 |
| `REF-R9` | Refuse a string carrying both a `hash` parameter and a non-empty path | MUST | §3.1 |
| `REF-R10` | Refuse a string carrying neither a `hash` parameter nor a non-empty path | MUST | §3.1 |
| `REF-R11` | Emit the scheme lowercase; accept it case-insensitively | MUST | §3.3 |
| `REF-R12` | Apply host-style case normalization to the authority component | MUST NOT | §3.3 |
| `REF-R13` | Compare path segments after percent-decoding | MUST | §3.3 |
| `REF-R14` | Percent-encode reserved characters within a path segment | MUST | §3.3 |
| `REF-R15` | Refuse a `.` or `..` segment in an **absolute-form** path, without applying the refusal to an unresolved relative string | MUST | §3.3 |
| `REF-R16` | Distinguish an absent query parameter from one present with an empty value, and refuse the second | MUST | §3.3 |
| `REF-R17` | Emit `entity+ref://` in a link position | MUST | §4 |
| `REF-R18` | Emit `entity://` in a link position, once emitting the atom at all | MUST NOT | §4 |
| `REF-R19` | Continue resolving `entity://` in a link position, and surface that it did | SHOULD | §4 |
| `REF-R20` | Treat an unparseable reference string as external rather than re-anchoring a tolerant scan | MUST | §3.4 |
| `REF-R21` | Treat a fetch failure at a `path` hint as evidence the referent does not exist | MUST NOT | §2.3 |

**Ids are allocated once and never reused** (`SPECIFICATION-FORMAT` §8.5a). A row may be added,
reordered or retired; its number does not move.

### 6.2 Vectors — OWED, NOT SHIPPED

**This convention is authored and is NOT ratifiable until these ship** (charter #5). Class:
application-tier format vectors — example atoms plus expected bytes and expected strings. These are
not wire-oracle checks and not host-seam checks.

| # | Vector | Drives | What fails without it |
|---|---|---|---|
| `REF-V1` | a pinned reference round-trips atom → string → atom, byte-identical both ways | `REF-R8` | a re-serializer silently rewrites references |
| `REF-V2` | a live reference carrying `seen`, `via` and `at` round-trips | `REF-R8` | the query component carries three different kinds of term and is where they diverge |
| `REF-V3` | **a mixed-case peer id survives parse and re-serialize unchanged** | `REF-R12` | **the highest-value vector here.** Every general URL library lowercases the authority; the failure is a clean `404` at a well-formed address, which reads as *not found* rather than as a bug |
| `REF-V4` | a string with both `hash` and a non-empty path is refused | `REF-R9` | the discriminator degrades from a rule to a convention |
| `REF-V5` | a string with neither is refused | `REF-R10` | as `REF-V4`, the other arm |
| `REF-V6` | a path segment containing a reserved character round-trips percent-encoded | `REF-R14` | the most common real content case — a page slug with a space or a `#` |
| `REF-V7` | a reference carrying an **unknown `via` hint tag** resolves identically to the same reference with `via` absent | `REF-R5`, `REF-R7` | droppability measured directly; without it hints become load-bearing |
| `REF-V8` | a `.` or `..` segment in an absolute-form path is refused | `REF-R15` | a resolver borrowing filesystem semantics produces a well-formed wrong address |
| `REF-V9` | a relative link containing `..` resolves normally and is **not** refused | `REF-R15` | the arm that catches over-application of `REF-V8`, which would reject ordinary correct links |
| `REF-V10` | a peer id encoded as a byte string is refused | `REF-R3` | the two spellings of an identity diverge across implementations with no error at either end |
| `REF-V11` | a pinned reference whose `path` hint `404`s still resolves from a second source | `REF-R21`, `REF-R5` | a reader that gives up at the hinted location makes a hint load-bearing, which is the whole failure `via` is bounded to prevent |

**`REF-V3`, `REF-V7` and `REF-V9` are the three that would not be written by someone implementing
from the prose alone**, which is the argument for naming them here rather than leaving them to be
discovered.

### 6.3 Types installed

**None.** This convention mints no entity type. It defines an atom that other conventions carry
inside their own types, and a string projection of it.
