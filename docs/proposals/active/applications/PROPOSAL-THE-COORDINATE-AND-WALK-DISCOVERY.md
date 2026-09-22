# PROPOSAL — the coordinate, and how a walk is found: discharging §3.5 for the published walk

**Status:** DRAFT (2026-09-07) — **first pass, written to be attacked. Not ready to rule.**
**Tier:** applications — app convention + SDK guidance over existing types. **§4.4 is the one clause that argues its own tier**, and it argues *up*.
**Rests on:** `PROPOSAL-THE-PUBLISHED-WALK` (the object) · `EXPLORATION-THE-INTEROP-FLOOR-…` §7 (the
obligation and the candidate) · `EXPLORATION-THE-REPLY-PACKAGE-…` (the ownership axis).

> **In one sentence.** A **coordinate** is always a `system/hash` over a canonically-encoded preimage
> that any two parties can compute without coordination; a **walk is published at a path constructed
> from it**; and that discharges a **landed core MUST** the walk proposal is currently in breach of —
> which incidentally means a reader holding only a coordinate can ask **any peer it has already
> reached**, with no index, registry or aggregator anywhere in the loop.

---

## §1 The obligation, because this is not a new idea — it is an unmet rule

**`ENTITY-CORE-PROTOCOL` §3.5, *Discovery locality — general principle* (normative, v7.45):**

> *"any entity that generic, cross-peer or cross-consumer core machinery must **discover** (not merely
> fetch by a hash it already holds) MUST be discoverable by one of two strategies — **(A)**
> observational/recorded discovery … with a fail-closed default when unresolved; or **(B)** a
> **core-general constructable path** … constructable with no extension knowledge. **It MUST NOT rely
> on path-construction against an extension-private scheme** … An extension introducing a new
> discoverable entity **MUST state which strategy it uses** and (for A) its fail-closed behavior."*

**A published walk is exactly such an entity, and `PROPOSAL-THE-PUBLISHED-WALK` declares neither
strategy.** This proposal declares **B**, and specifies what B requires: a constructable coordinate and
a constructable path.

**Strategy A is rejected for this entity, and the reason is the use case.** Observational discovery
means *the discoverer records the location as it ingests* — which works only for walks you have already
seen. **The query is "find a walk for a coordinate I hold from a peer I have not asked yet."** A is
structurally unable to answer it; that is not a preference.

---

## §2 The coordinate is always a `system/hash`

```cddl
coordinate = system/hash        ; over the canonical encoding of exactly one preimage below

coordinate-preimage = entry-coord / namespace-coord / path-coord / name-coord

entry-coord     = { kind: "entry",     ref: reference }        ; a pinned entity — OWNED
namespace-coord = { kind: "namespace", peer: peer-id }         ; everything in a peer's namespace — OWNED
path-coord      = { kind: "path",      peer: peer-id, path: tree-path }  ; a subtree — OWNED
name-coord      = { kind: "name",      scheme: tstr, value: tstr }       ; a subject — OWNERLESS
```

**Uniform type, so the path shape and the hex rules are uniform** (`ENTITY-CORE-PROTOCOL` §3.5's hex
convention: lowercase, format-code byte included, normative). **The kinds differ only in what bytes get
hashed.**

**A walk MUST carry the preimage beside the coordinate** so a reader can recompute it and confirm the
walk is about what it claims. **A coordinate is therefore checkable, like everything else here** — a
walk claiming coordinate `C` with a preimage that does not hash to `C` is refused, at no cost.

### §2.1 The three owned kinds close the single-entity limit

`PROPOSAL-THE-PUBLISHED-WALK` §2 declares `coordinate: reference`, which pins **one entity** — so
*"replies to anything I wrote"* is inexpressible and would be N walks for N entries. **`namespace-coord`
and `path-coord` fix that**, and they satisfy the governing property (§2.2a of the exploration): a
peer-id and a tree path are both computable by both parties with no coordination.

**All three owned kinds have a derivable publisher** — the coordinate names its owner —
so for these, discovery has *two* answers: the constructable path below, and *"ask the owner."*

### §2.2 `name-coord` — and the design move is to stop trying to be clever

**This is where this class of design goes wrong, so it is stated as a refusal first:**

> **We do not attempt to make "the same topic" match. A coordinate is a deterministic function of an
> exact byte string, and nothing more.**

Canonicalization is **minimal and mechanical**, and it reuses the corpus's existing house rule rather
than inventing one — `EXTENSION-REGISTRY` §6.3's local-name safety rules:

- `value` **MUST** be Unicode-normalized to **NFC**.
- `value` **MUST NOT** contain control characters U+0000–U+0020 or U+007F.
- `scheme` **MUST** be a non-empty ASCII string, and **the scheme declares the case policy**
  (`as-written` or `lowercase`) — because case-folding is a per-domain judgement and a protocol-wide
  answer would be wrong somewhere.
- **No confusable folding, no stemming, no synonym mapping, no stripping.** Ever, at this layer.

**Two people who type slightly different names get different coordinates. That is the correct
outcome**, and it is only survivable because of the next clause.

### §2.3 Aliasing is a walk, not a normalization problem

**"These two coordinates are the same subject" is an ordinary signed claim**, published like anything
else and discoverable at **both** coordinates' paths. Readers union them and apply their own policy.

- **No canonical namer exists, and none is needed.** The system already has no gatekeeper; a naming
  authority would be the first one.
- **It merges by union**, so partial alias knowledge composes exactly like partial walk knowledge.
- **It is checkable in the only sense that matters**: an alias is a *pointer*, so a wrong one costs one
  wasted fetch — the same argument as a participant claim or a locator.

> **This is the whole trick: identity is exact, and agreement about identity is a first-class mergeable
> artifact.** Every system that instead tried to make naming *canonical* had to appoint someone to
> canonicalize it.

---

## §3 The path — Strategy B

### §3.1 Shape

```
/{walker}/system/walk/{hex(coordinate)}
```

Mirroring the invariant pointer `/{signer}/system/signature/{target_hex}` — the pattern §3.5 names as
its own example of a core-general constructable path. **Lowercase hex, format-code byte included.**

**The leaf holds a bare `system/hash` naming the walk body**, not the body itself — the two-layer
storage pattern `EXTENSION-REGISTRY` §6.3 already uses (immutable content-addressed body, mutable tree
pointer). So republishing a grown walk moves one pointer.

**What this buys, stated as the mechanism rather than the benefit:** *given a coordinate and any peer
you can already reach, the address of that peer's walk is computable.* **No lookup service is consulted
at any point.** The askable set is your follow graph plus everyone their citations introduced you to
(§3.1), and the answers merge by union.

### §3.2 Absence is could-not-look — unless the walker declares the prefix

**A 404 at the constructed path means "this peer has no walk for C **at an origin that answered**."**
It is **not** evidence that no walk exists, and it is **not** evidence this peer has none — which is
the **absent-vs-withheld collapse** the discipline charter has now recorded **seven** times.

**But there is a fix already specified and it costs one declaration.** If the walker declares
`system/walk/` as a **tracked prefix** under its signed root (`EXTENSION-TREE` §3.3a), then
`EXTENSION-REGISTRY` §6a.3a applies verbatim: *"a hostile origin … cannot omit a node from the walk
without the walk failing. Silently hidden becomes visibly incomplete."*

| Walker published `system/walk/` as a tracked prefix? | What a 404 means |
|---|---|
| **no** | **could-not-look.** No inference permitted |
| **yes** | **a verifiable negative** — *this peer has no walk for C*, provable against their signed root |

**`[SHOULD]` declare it.** This is the one clause that converts *"ask everyone and hope"* into a
bounded search, and it is the reason the fail-closed question §3.5 asks about Strategy A has a sane
answer here: **absence yields no conclusion unless the publisher paid for one to be drawn.**

### §3.3 Volume

**Per-coordinate paths are the right unit and the cost is a trie, which is what the substrate is.**
A walker covering 10,000 coordinates holds 10,000 leaves; `EXTENSION-REVISION` §8.7.2 records that
snapshot cost is `O(changed_paths × depth)` **regardless of prefix size**, so growth is in what changed,
not in what is held.

**The cost that is real is on the READER: N peers × M coordinates is N×M fetches.** Three existing
mitigations, none new: a walker's **own** walk-index is a `namespace-coord` walk over itself (one
fetch, then targeted); `subsumes` means a reader holding a cited walk fetches only the delta;
and the tracked-prefix declaration above turns a miss into a *decided* miss rather than a retry.
**Whether that is enough is an open question and it is §6's first row.**

---

## §4 What this deliberately does not decide

### §4.1 Acceptance policy is the reader's, and the corpus already has the primitive

A walk is a set of claims. **What a reader does with them — accept all, require K independent
agreeing parties, weight by peer — is policy, and this proposal specifies none of it.**
`EXTENSION-QUORUM` is *"K-of-N consensus over an entity — a primitive authorization pattern, not an
identity-specific feature"*, and it is the right instrument where a reader wants one.

**And that is where this pattern and the build case turn out to be one pattern.**
`PROPOSAL-EXTENSION-PACKAGE` §2 defines `build-provenance` with
`attesting: <peer_hash | quorum_hash>` and states in §7 that *"independent `build-provenance`
attestations over one `attested` hash from distinct `attesting` peers = independent confirmation the
build is reproducible… reproducibility is **empirical, not structural**."*

> **A build-attestation set and a walk are the same object with a different predicate: a grow-only set
> of independently-signed claims about one coordinate, merged by union, with the reader applying an
> acceptance policy.** For replies the policy is *show me all*; for a build it is *K distinct
> attesters*; for a converging article it is *the input set I chose*. **The information flow
> generalizes; what is in the claim does not have to.** *(This also supplies the "importance
> dimension" a composition set needs and a redundancy set does not — it is the reader's, not
> the set's.)*

### §4.2 It does not make coverage verifiable

Unchanged and unchangeable: `L-8`'s law. §3.2 makes **one peer's** answer verifiable. **Nothing makes
the union complete**, and no `complete` field will ever exist.

### §4.3 It does not replace the registry

Names with human meaning still resolve through REGISTRY, which is the authority for *name → peer* and
the cold-start entry point. **`name-coord` is not a name-resolution mechanism** — it is a
subject identifier that happens to be spelled with characters, and it confers no authority over
anything.

### §4.4 The one clause that argues its own tier — and it argues up

**§3.5 forbids path-construction against an *extension-private* scheme.** So §3.1's path convention
**cannot** live in a document only some clients read; a client that does not know it cannot construct
the address, which is definitionally the defect class §3.5 names.

> **So this document is filed at the applications tier, and §3.1 specifically wants to sit at the
> interoperability floor** — the layer at which a convention must live for every client to construct
> paths against it. **Flagged, not decided.**

---

## §5 Deltas

| # | Target | Change |
|---|---|---|
| **D1** | `PROPOSAL-THE-PUBLISHED-WALK` §2 | `coordinate: reference` → `coordinate: system/hash` + required `coordinate_preimage`. Closes `F-35` |
| **D2** | `PROPOSAL-THE-PUBLISHED-WALK` §7.2 | replace the open question with the §3.5 declaration: **Strategy B**, with §3.2's absence semantics as the fail-closed statement |
| **D3** | this document → an `APP-CONVENTION-*` on fold | §2's preimages, §2.2's canonicalization, §3.1's path, §3.2's `[SHOULD]` |
| **D4** | `EXTENSION-TREE` §3.3a *(pointer only, no normative change)* | note `system/walk/` as a per-purpose tracked prefix instance |
| **D5** | none to `EXTENSION-REGISTRY`, `-QUORUM`, `-ATTESTATION` | §4.1 uses them as-is; **this proposal defines no new entity type** |

---

## §6 Open — and the first two are the ones to attack

1. **Is §3.3's reader cost acceptable — `N peers × M coordinates`?**
   **CORRECTED 2026-09-07, same day, before any seat acted on it.** This row first read *"no cost model
   exists,"* which is **false**:
   `EXPLORATION-WHAT-CONVERGES-WHAT-CANNOT-AND-THE-ORDER-TO-BUILD-IT` **§6** models the **follow loop**
   — a signed-root comparison measured at *"roughly ten kilobytes and a handful of nodes even while the
   publisher rewrote most of their estate"*, unchanged meaning **no further work at all**, a linear
   `N follows = N origins` term, *"a thousand follows is ordinary reader behaviour… a hundred thousand
   is not a client's job"*, and the fan-in mirror as the answer at that scale.
   **What survives is narrower and is the real question: that models `N`, and walk discovery is
   `N × M`.** A second multiplication over a working set of coordinates is a different term, it is
   genuinely unmeasured, and §3.3's three mitigations are untested against it. *(The error was L22's
   shape — a true claim about one subject re-pointed at another — and the corrected ask is better than
   the wrong one.)*
2. **Is `scheme` enough to keep `name-coord` from becoming a squatting surface?** Nothing stops two
   communities from using one scheme incompatibly. **Alias walks (§2.3) are the stated answer and they
   have not been stress-tested against an adversary**, only against honest disagreement.
3. **Should the walk pointer be one leaf or a `(coordinate, walker)` pair index?** §3.1 assumes the
   walker's own namespace is the index. A reader-side inversion may be better and is unexamined.
4. **`name-coord` case policy per scheme** — declared where? A scheme registry is a gatekeeper and we
   do not want one; a free-text scheme with an inline policy field may be sufficient and is untested.
5. **Nothing here is measured.** Every claim is structural. **§7 is the ask.**

---

## §7 What would test this

**The claims here are structural and none is measured.** Three that a client implementation would
settle quickly:

1. **Reader cost.** §3.3's `N peers × M coordinates` is a second multiplication over the follow loop's
   `N`, and it is unmeasured. If §3.3's mitigations do not collapse it, §3.1's per-coordinate path is
   the wrong unit.
2. **Path construction in situ.** Does §3.1's path construct from state a loader already holds at the
   point it would need to, or does it require state carried elsewhere?
3. **Whether §3.2's `[SHOULD]` is real.** If publishers would not in practice declare `system/walk/` as
   a tracked prefix, the verifiable-negative it buys is decorative and the clause should be demoted
   rather than shipped as a `SHOULD` nobody follows.

**And one schema question that only an emitting implementation can answer:** are §2's four preimage
kinds sufficient, or is there a fifth — a coordinate that is neither an entry, a namespace, a path, nor
a name?
