# PROPOSAL — the published walk: one object for every "who contributed to this" question

**Status:** DRAFT — 2026-09-07. First pass, and it is a **generalization of something already
specified**, not a new mechanism. **Not ratified, not folded.**
**Tier:** open — see §7. It is either an extension or an SDK-level convention; the derivation does not
decide it; the choice is open.
**Derived in:** `EXPLORATION-THE-TWO-LEVEL-INDEX-…` (read §2, §4, §5b first).
**Generalizes:** `PROPOSAL-APP-CONVENTION-FEED` §4's `app/feed/mirror`.

---

## §1 What this is, in one paragraph

**Every application in this system has the same unanswered question**: *N writers each own only their
own namespace and contribute to a subject nobody owns — how does a reader holding the subject find the
contributions?* Thread, repo, wiki page, listing, room, document: **one problem, six costumes.**

**The base case needs nothing** — a contribution is an entry in the contributor's own stream, so
following them delivers it, and every entry names its outbound edges so the graph walks forward for
free. **This proposal is only for the residue: contributors you do not follow.**

**And it is one object.** A peer that walked the graph publishes what it found, **signed, in its own
namespace, as the same kind of entity as everything else it publishes.** That last clause is the whole
design and §3 is about why.

## §2 The object

```cddl
published-walk = {
  type: "system/walk",
  data: {
    coordinate:   reference,          ; the subject this walk is about
    participants: [* peer-id],        ; WHO contributed. The cheap rung; may stand alone
    entry_hashes: [* system/hash],    ; OPTIONAL — WHAT they contributed, by hash
    entries:      [* reference],      ; OPTIONAL — the bytes, republished UNMODIFIED
    subsumes:     [* system/hash],    ; OPTIONAL — walks this one folds in (§4)
    walked_at:    uint,
    walked_by:    peer-id
  }
}
```

**Three rungs, one object, one knob.** A publisher includes as much of its walk as it chooses:

| Published | Cost | A reader gets |
|---|---|---|
| `participants` | ~32 B each | who to go ask |
| `+ entry_hashes` | +32 B each | what to ask for; dedups against what it holds |
| `+ entries` | full bytes | no second fetch — **this rung is `app/feed/mirror`** |

**There is no separate index format and no aggregator protocol.** `app/feed/mirror`
(`{subject, entries, gathered_at, gathered_by}`) is this object at the top rung with a feed-shaped
coordinate; **it should become an instance of this rather than a parallel design.**

### §2.1 The rules it inherits, unchanged

**All four of `FEED` §4.1's rules apply verbatim and are not restated here** — republish original bytes
(a re-encoded entry no longer verifies), **omit but never substitute**, *a walk is not authorship* (each
entry travels with its author's detached signature; attribution follows `entry.author`), and unknown
fields survive because bytes are republished rather than re-serialized.

**One rule is added, and it is what makes the cheap rung safe:**

> **A `participants` claim is a POINTER, never evidence.** A reader MUST verify by fetching the
> contributor's own entry and confirming its signature and that it names the coordinate. **A fabricated
> participant costs one wasted fetch and nothing else** — which is why this rung needs no trust in the
> walker at all.

### §2.2 Merge is set union, and that is the entire reconciliation story

Two walks over the same coordinate merge by **union of `participants`**, and separately of
`entry_hashes` and `entries`. Associative, commutative, idempotent; **no ordering, no conflict, no
resolution policy.** *Two partial answers compose into a better one with no coordination and no trust
in either source* — which is why "which walk is right" is never a question anyone has to answer.

## §3 Why the type must be the same as everything else — the load-bearing clause

**This is the reason to write a proposal at all rather than let each application invent one.**

> **A walk is published as an ordinary signed entity in the walker's own namespace, and it MUST NOT be
> exposed as a service endpoint, an API, or a query interface.**

**Because then a walk is itself walkable.** You subscribe to a peer that publishes walks exactly as you
subscribe to a person; walks can cover walks; a client merges a walk and an original stream with the
same code, because they are the same type.

**The counterfactual is the whole field.** An ATProto AppView's output is an **API**. A Nostr relay's is
a **query response**. An ActivityPub instance's is **its own database**. **None is the same type as its
input, so none can be consumed by the next layer** — and *that*, not a policy choice, is why each of
those ecosystems has an aggregation tier that concentrated. **Type identity is the anti-centralization
property, and it is one sentence.**

## §4 `subsumes` — the answer to redundancy, and it is the sharpest open problem this closes

**Half of it is already free:** entries are content-addressed, so N walks carrying the same entry cost
one copy at any reader. **The duplication that is real is the walk records themselves.**

**`subsumes` makes each additional walk cost only its delta.** A walker cites the walks it folds in —
**including other peers' walks** — and publishes only what it adds:

- **The publisher's cost is proportional to what it *added*, not to what it knew.**
- **A reader holding a cited walk fetches only the delta**, so union becomes incremental instead of
  N-way.
- **Redundancy becomes self-limiting**: the more coverage already exists, the smaller each new walk is,
  and a walker with nothing to add publishes nothing.

**Precedent, and it is the ordinary shape of this problem:** a git commit cites its parents and you
fetch what you lack; BitTorrent's bitfield says what you have so the peer sends the rest. **And the
corpus already uses the pattern one noun over** — `published-root.predecessor` and
`registry/binding.supersedes`. **What is new here is only that the citation may point at a walk
somebody else published**, which is exactly what makes it dedupe *across* peers rather than within one.

## §5 Announce — an optimization, never a requirement

A walker may poll. A contributor may also **tell a walker directly** — a contentless
`(contributor, coordinate)` pair the walker verifies by fetching, exactly as Webmention's receiver
*"MUST perform an HTTP GET on source… to confirm that it actually mentions the target."*

**This is deliberately aimed at a walker and not at the contribution's author.** Announcing to an author
would need an **open delivery grant** — a spam surface to bound. **Announcing to a peer that publishes
walks needs nothing**: receiving these is what it is doing, it verifies before believing, and it may
rate-limit, require registration, or ignore announces and just poll. **Nobody's inbox is opened.**

**It stays optional in both directions.** A walker that only polls is conformant and slower; a
contributor that never announces is still found. **That keeps the offline tier whole** — announcing
needs a live recipient, and the recipient is a party that chose to be live.

## §6 What this does NOT do

- **It does not create a required role.** A client walking its own follow list is the degenerate case;
  publishing is optional; **a read-only peer that never publishes is fully participating as a reader.**
  *(But see §7.4 — "not required" is not "does not exist".)*
- **It does not offer global search.** The split answers **coordinate-anchored** queries — *what
  contributes to C*. **Content-anchored queries — *what contains X* — have no anchor and are
  whole-corpus by the nature of the question.** The top rung is a search corpus at the honest price,
  scoped to what that peer chose; **we should say plainly that global search is not on offer** rather
  than imply the object covers it.
- **It does not claim completeness, ever.** There is no `complete` field and there will not be one.
  Coverage improves monotonically by union and is never certified — **which is the same law that closes
  rung 4 of the verification ladder** (completeness needs a single writer at some scope; a subject
  nobody owns has none).

## §7 Open questions — and §7.1 is the real one

1. **What SHOULD a walk contain?** This proposal specifies a container and says almost nothing about
   what is worth putting in it. A walk could be raw participants, a digest, or a pre-computed
   navigational structure. **A walk could be
   raw participants, or a digest, or a pre-computed navigational structure**, and *"how do you design
   them so they create ways to navigate that are more efficient"* is unanswered here. **This is the
   next piece of design work and it is bigger than the container.**
2. **Discovery — and this proposal is currently in breach of a landed core MUST.**
   **`ENTITY-CORE-PROTOCOL` §3.5** (*Discovery locality — normative, v7.45*): *"any entity that
   generic, cross-peer or cross-consumer core machinery must **discover** … MUST be discoverable by one
   of two strategies — (A) observational/recorded … or (B) a core-general constructable path … **It
   MUST NOT rely on path-construction against an extension-private scheme** … An extension introducing
   a new discoverable entity **MUST state which strategy it uses**."* **A walk is exactly such an
   entity and this document states neither strategy.** That is not a gap in the reasoning — it is a
   rule that already existed, in the core spec, and the fix is a declaration plus the path
   convention it implies. **Strategy B is the candidate**, and
   `EXPLORATION-THE-INTEROP-FLOOR-…` §7.1 argues it dissolves the ownerless-coordinate problem: a walk
   at a path constructed from the coordinate means **any peer you already reached can be asked about
   any coordinate you hold**, with no index and no aggregator. **Two things must be settled first:
   name→coordinate normalization, and per-coordinate path volume.**
   **HALF ANSWERED, and it was not registry-adjacent.**
   `EXPLORATION-THE-REPLY-PACKAGE-…` §3 splits this row. **Reaching a peer you hold a coordinate for
   needs no registry**: the coordinate names the peer, and the citation that gave you the coordinate
   carries the peer's reachability record (§2's *"anyone may serve it"* in
   `PROPOSAL-PEER-TRANSPORT-SET`). **What is still open is finding a walk for a coordinate whose
   participants you have never heard of** — and §7.3's rule narrows even that, because a walk over an
   **owned** coordinate is at a location the coordinate itself names.
3. **When to publish — ANSWERED as a rule rather than a heuristic.** `EXPLORATION-THE-REPLY-PACKAGE-…`
   §4: ***publish the walk anchored at a coordinate you own.*** The owner is derivable from the
   coordinate, has a standing interest in every contribution, and is the only party whose scope is not
   arbitrary — so **K repliers produce exactly one natural republisher**, and `subsumes` is left to
   bound only the ownerless case. **The cost is that the owner's walk is the least neutral one**
   (`FEED` §4.1's *omit but never substitute* is the protection; union with any other walker is the
   corrective), and **ownerless coordinates keep the whole problem.**
3a. **A gap this exposed.** `coordinate: reference` pins **one entity**, so *"replies to anything in my
   namespace"* — the natural subscription unit for the owner-publishes case — **is not expressible.**
   Widening it is bounded by the property that makes the split work at all (a coordinate must be
   computable by both parties with no coordination): an entry pin, a **peer-id**, and a **tree path**
   all qualify and are all owned; a topic string qualifies and is not. **Decide this together with
   §7.1, not before it.**
4. **Tier, and whether a dedicated role should be named.** **Correct, and this proposal deliberately does not name one** — but there is at least one
   job only an always-on party can do: **metering a non-publishing client's data into the tree**, which
   is a hosted-tier concern and is out of scope here. **Whether this is `EXTENSION-*` or an SDK
   convention is undecided.**
5. **The offline-to-offline case is untested.** Two peers that are never live exchanging encrypted
   traffic through static publication is claimed to work and **has not been demonstrated**. It also
   carries a risk worth stating: **publishing ciphertext means a key compromise is retroactive over
   everything published**, so key rotation and forward secrecy matter more here than in a live
   transport. **Owed as a measurement, not an argument.**
6. **Economics, recorded as a risk rather than a design input.** This model's scalability rests on
   cheap CDN-backed object storage. Free tiers are cross-subsidised, and broad
   adoption of exactly this pattern could change that. **Speculative, out of scope, and worth having
   written down** — it would still be the lowest-cost option at a higher price.
