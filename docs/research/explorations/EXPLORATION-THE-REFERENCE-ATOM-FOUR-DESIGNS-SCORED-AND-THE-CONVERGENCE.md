# EXPLORATION — the reference atom: four designs, scored, and the convergence

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing here is folded and no
spec text is written.**

**Operator-requested, 2026-09-06 (third pass).** *"Let's see what it converges to. Let's do our best —
maybe we do a few different designs and then we pro/con them, see how they all work out, and then we try
to converge on the one that makes the most sense."*

**Predecessors, both read and not re-derived:** `EXPLORATION-THE-LINK-AND-THE-WALK-…` (the problem: nine
shapes, five discriminators, two peer terms) · `EXPLORATION-THE-REFERENCE-PRIMITIVE-SIX-SYSTEMS-AGAINST-ONE-TUPLE`
(the reduction: `(identity, authority, hints, expectation)` + a resolution record).

---

## §0 The result

**Four designs were built and scored against nine criteria drawn from the corpus's own rules. Design C
wins, and the margin comes from one criterion the other three cannot satisfy at all.**

**The finding that decided it, and it was discovered while building Design C: `entity://` is already
taken, and it does not mean what a hyperlink means.** It is the **EXECUTE dispatch URI** —
`entity://{peer_id}/{handler-path}` — used throughout `SDK-OPERATIONS`, `EXTENSION-COMPUTE`,
`EXTENSION-SUBSCRIPTION`, `EXTENSION-CONTENT` and `ENTITY-SYSTEM-REFERENCE`, whose §228 says outright:
*"If EXECUTE → extract handler path from URI (strip `entity://peer_id/`)."*

> **So `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4's `link-ref` comment — which names `"entity://…"` as the
> absolute form of a nav link — is a layer confusion. Clicking a link in a menu is not executing a
> handler.** It is a one-line comment and it is the only guidance a renderer has, which is why D-36
> exists. **This is a second, independent reason the string form needs designing rather than assuming.**

**The convergence is not one design but a staged pair**, and the staging is what makes it cheap:

| Stage | What | Why now |
|---|---|---|
| **1** | **Design C's structured atom** — tagged, with an advisory `via` slot | it is where two seats are about to build, and the atom is under review *now* |
| **2** | **Design D's resolution record** — `D-27`, the publishable resolver view | independent, no blocker, and it is the steady state that makes stage 1's hints droppable |

**Not one or the other. Nostr ships both and says why** (`…-ONE-TUPLE` §3): the link-borne hint is the
bootstrap, the identity-borne record is the steady state, and they fail in opposite directions.

---

## §1 The criteria, fixed before the designs were scored

**Drawn from the corpus rather than invented, so the scoring is checkable.**

| # | Criterion | Source |
|---|---|---|
| **1** | **Fixes the string surface** — `link-ref` and prose links | the originating complaint; D-36 |
| **2** | **Reaches Mode A1** — carries a publisher peer-id | `NETWORK` §6.5.5; the fetch path's input |
| **3** | **Handles an unresolvable peer** | `NETWORK` §6.5.4 defers peer→transport discovery |
| **4** | **Preserves the rug-pull guarantee** — no optional-hash collapse | `FEED` §2.2.3, corroborated in four systems |
| **5** | **Consistent with the tier's discrimination discipline** | `EMBED` §3 / `SHARE` §2.2 both tag; `G-PIN-2` |
| **6** | **Low cost to the two shipping seats** | L26 — the cohort discovers by building |
| **7** | **Extends to anchors** — the D-31 / rung-1 field path | the durable-reference ladder |
| **8** | **Creates no new rot surface**, or makes it droppable | measured decay in Nostr hints and IPFS records |
| **9** | **One thing to teach** | the operator's *"clear, understandable, reduced to primitive form"* |

---

## §2 Design A — *Promote and stop*

**Lift `FEED`'s two atoms verbatim into a shared home the whole app tier imports. Change nothing else.**

```cddl
reference      = { peer: peer-id, hash: content-hash, ? path: tree-path }
live-reference = { peer: peer-id, path: tree-path, ? seen: content-hash }
```

**Pro.** Zero new design — the shapes are derived, corroborated against four systems, and already
written. Smallest possible diff. Nothing to argue about. Immediately fixes seven shapes' missing peer
term, which is the finding that actually bites.

**Con.** **Does not touch the string surface at all**, so the operator's originating complaint —
`link-ref`, prose links — is untouched (criterion 1). No hint slot, so both open fetch cases stay open
(criterion 3). **Re-freezes `path`'s double role**: advisory in `reference`, authoritative in
`live-reference`, same field name — the ambiguity the reduction identified, now blessed tier-wide.

**Verdict: the honest baseline.** If nothing else is affordable, this is strictly better than today and
should be done. It is not the answer to what was asked.

---

## §3 Design B — *Promote + `via`*

**Design A plus an advisory hint list.**

```cddl
reference = {
  peer: peer-id, hash: content-hash,
  ? path: tree-path,
  ? via:  [* hint]        ; ADVISORY. Unsigned. A reader ignoring all of them MUST reach the same answer
}
```

**Pro.** Closes the unresolvable-peer case (criterion 3) with the exact mechanism Matrix, Nostr and
BitTorrent each adopted independently. Still small. The safety property — *droppable, or it is not a
hint* — keeps the rot surface bounded (criterion 8).

**Con.** **Still no string form** (criterion 1 unaddressed). Keeps field-name discrimination, so the tier
stays split between two discrimination styles (criterion 5). `path` and `via` now overlap — `path` in
`reference` *is* a hint, so the atom has a hint field and a hint list and no rule about which wins.

**Verdict: A's problem, one gap smaller.** The `path`/`via` overlap is the tell that the shape is not yet
regular.

---

## §4 Design C — *Tagged atom + `via` + a canonical string form*

**Three changes, and the third is the one no other design makes.**

```cddl
; --- structured form: TAGGED, matching EMBED §3 and SHARE §2.2 -------------------
entity-ref = pinned-ref / live-ref                 ; TAGGED — no untagged ambiguity

pinned-ref = {
  tag:   "pin",
  peer:  peer-id,                                  ; authority — Mode A1's input
  hash:  content-hash,                             ; identity AND expectation (they coincide)
  ? at:  anchor,                                   ; sub-entity address; absent = the whole entity
  ? via: [* hint]                                  ; ADVISORY, ordered by descending confidence
}

live-ref = {
  tag:    "live",
  peer:   peer-id,
  path:   tree-path,                               ; identity — the address of record
  ? seen: content-hash,                            ; expectation only
  ? at:   anchor,
  ? via:  [* hint]
}

anchor = { field: [* tstr] }                       ; rung 1 — a field path. Rungs 2/3 are later tags
hint   = { tag: "origin"/"mirror"/"peer", value: tstr }   ; ranked; unsigned; droppable
```

**And a string projection, losslessly convertible, under a scheme that is NOT `entity://`.**

**Pro.**

- **The only design that fixes criterion 1.** A nav target, a prose markdown link, a pasted link, a QR
  code and a shared URL all need a *string*. **Every system surveyed has one** — `at://`, `nostr:`,
  `magnet:`, `matrix:`, `ipfs://`, Nix store paths. We have string projections already (`::embed{ref=}`,
  `link-ref`) and they are ad hoc, which is precisely D-36.
- **Uniform with the tier's majority discipline** (criterion 5). `EMBED` and `SHARE` both tag; only
  `FEED` discriminates by field name. Tagging is also what makes a **third** intent addable later
  without touching the first two.
- **`at:` gives the anchor ladder a home now, at zero cost** (criterion 7). Absent means whole-entity, so
  nothing changes for anyone who does not use it, and D-31's rung 1 (a field path — *"the hash changes,
  the field path does not"*) has a declared slot rather than needing a new atom later.
- **Kills the `path`/`via` overlap.** `path` appears only in `live-ref`, where it is authoritative;
  every advisory locator is a `via` hint. **One field, one job** — which is the reduction the operator
  asked for.
- **One thing to teach** (criterion 9): *a reference names who, what, optionally where-to-look-first, and
  optionally which part.*

**Con.**

- **Largest diff of the four.** Both shipping seats change; `FEED` is re-cut before landing.
- **A string form is a real surface with real hazards** — escaping, normalization, case, percent-encoding,
  and the round-trip obligation (`EMBED` §10's *"the directive and the child entity must be the same
  thing"* is the precedent, and it is a conformance-vector cost).
- **A new scheme is a naming decision with no obvious right answer**, and `entity://` — the one that
  looks right — is taken by dispatch (§0).
- **Tagging is churn against a landed derivation.** `FEED` §2.2.3 chose field-name discrimination
  deliberately and its argument is sound. **The counter is that its argument is against an *optional
  field*, not against a tag** — a tag satisfies §2.2.3's requirement (a reader can always tell) more
  explicitly than a field-name xor rule, and matches what the tier already does twice.

**Verdict: the most work and the only one that answers the question asked.**

---

## §5 Design D — *Thin reference + resolution records*

**Strip the reference to `(identity, authority)`. Put every locator in a published, maintained record —
the Nix `narinfo` / ATProto DID-document / NIP-65 model.**

```cddl
entity-ref = { peer: peer-id, hash: content-hash }        ; and nothing else
; locators live in the peer's own published record, resolved on demand
```

**Pro.** **Hints cannot rot in links, because there are none in links** (criterion 8, perfectly). One
place to update. It is the mature end-state of three of the six surveyed systems, and **the record half
is already on our board as D-27** with an independent motivation. Smallest possible reference.

**Con.** **It does not solve the case that is actually open.** A resolution record must be *resolved*,
which requires resolving the peer — and *"I cannot resolve this peer"* is precisely gap 1. **D is the
steady state and cannot be the bootstrap.** This is not a guess: Nostr deployed NIP-65 and **kept** the
NIP-19 link hints, because the two fail in opposite directions.

**Verdict: correct, necessary, and not sufficient alone.** It is stage 2, not an alternative.

---

## §6 The scorecard

**✓ = satisfied · ~ = partial · ✗ = not addressed.**

| Criterion | **A** promote | **B** +`via` | **C** tagged+string | **D** records |
|---|:--:|:--:|:--:|:--:|
| 1 · fixes the string surface | ✗ | ✗ | **✓** | ✗ |
| 2 · reaches Mode A1 | ✓ | ✓ | **✓** | ✓ |
| 3 · unresolvable peer | ✗ | ✓ | **✓** | ✗ |
| 4 · rug-pull preserved | ✓ | ✓ | **✓** | ~ |
| 5 · tier discrimination discipline | ~ | ~ | **✓** | ~ |
| 6 · low cost to seats | **✓** | ✓ | ~ | ✓ |
| 7 · extends to anchors | ✗ | ✗ | **✓** | ✗ |
| 8 · no new rot surface | **✓** | ~ | ~ | **✓** |
| 9 · one thing to teach | ~ | ~ | **✓** | ✓ |

**Criterion 1 is the discriminator and it is not a tie-break — it is the original complaint.** Three of
the four designs leave a nav link as `tstr` resolved by *"the renderer's classifier."* **A design that
standardizes the wire form and leaves the human-facing form ad hoc has standardized the half that was
already nearly consistent.**

**Where C is weakest, and it should be said plainly:** criterion 6 (seat cost) and criterion 8 (a string
form and a hint list are both new surface). **Both are real and both are bounded** — 6 by doing it before
`FEED` lands rather than after, 8 by the droppability rule and by having no signed hint.

---

## §7 The convergence

**Design C for the atom, Design D for the record, staged — and A is the fallback if C is judged too
expensive.**

**Stage 1 — the atom (D-37).** Design C's structured shape. **Sequenced first because `FEED` is under
review and two seats are about to build against it**, and the difference between changing an atom now and
changing it after two implementations ship is the whole cost of the decision.

**Stage 2 — the record (D-27).** Independent, unblocked, already on the board, and it is what lets stage
1's `via` hints stay genuinely optional rather than quietly becoming load-bearing.

**Three things the convergence deliberately does NOT do:**

1. **No content routing.** Both predecessors reached this and the evidence is the same: the convergent
   endpoint of every P2P discovery design surveyed is *"ask a well-known HTTP server,"* which is our
   origin tier. **A publisher who is gone stays a genuine dead end**, and that is the honest `404`.
2. **No collapse of the two intents into one shape.** `FEED` §2.2.3's refutation stands; C changes *how*
   they are told apart, never *whether*.
3. **No anchor semantics beyond a declared slot.** `at:` reserves the shape; rung 2 (a named node in a
   body) still needs `EMBED` to mint a name slot, and no consumer has asked (L26).

### §7.1 The open question the convergence does not settle, and it is the operator's call

**What the string scheme is called.** `entity://` is taken by dispatch (§0), and reusing it would put a
hyperlink and an EXECUTE target in one namespace — the exact word-overloading
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §11 already forbids (*"each word in the system carries one
meaning"*). Three candidates, none obviously right:

| Candidate | For | Against |
|---|---|---|
| **`ent:`** | short, unclaimed, reads as a scheme not a location | collides with nothing here and everything elsewhere — it is a very common abbreviation |
| **`entity+ref://`** | unambiguous, structurally related to the existing scheme | ugly, and plus-schemes are poorly handled by real link detectors |
| **a reserved first segment**, as `sites` is at the §6.5.6 demux | **no new scheme at all** — reuses machinery that already exists and has a registration hook | only works where there is a base URL; does not help a bare pasted string |

**This is a naming decision with a real consequence and it is not arch's to make alone.** It is also the
only part of the convergence that is blocked on anything.

---

## §8 A documentation finding, since it was the other half of the ask

> *"Some of these explorations are really — they almost become canonical references, so when people ask
> 'why did you do it this way,' we can say this is what we studied."*

**They already are.** `CANONICAL-DOCS.toml` line 121 declares **`docs/research/explorations`** as a
directory — **all 60 publish**, including the three written today. The instinct is right and the state
already matches it.

**Two consequences that follow from that and are worth knowing rather than guessing:**

- **Measured: 16 of 60 published explorations cite internal-only document families** (`HANDOFF-*`,
  `BEARINGS-*`, `CHECKPOINT-*`, `ROUTING-*`, `WORKSTREAMS`, `docs/status/`) that a public reader cannot
  open. **This is covered by a stated convention, not a defect** — `docs/research/INDEX.md` (itself
  canonical) carries the note enumerating those families and saying a citation to one *"records where the
  reasoning happened, not a document you can open here."* **Working as designed.**
- **Not covered by that convention: docket IDs.** `D-27`, `D-37` and friends live in
  `docs/status/WORKSTREAMS.md`, which never publishes, **and they have already been renumbered once.**
  Four published explorations use them — **including two of today's three, heavily.** A public reader
  gets a bare token with no referent. **This is small, real, and worth a convention rather than a
  cleanup**: either spell the item out at first use, or say once per document that `D-*` are internal
  tracking IDs.

**And a gate-scope observation, filed rather than acted on.** The `standards` rules that hold
`specs/` and `guides/` to the published-surface discipline — `impl-team-ref`, `date-in-body`,
`proposal-citation` — **do not scan `docs/research/` at all**, though it is declared canonical. So 60
published documents are graded by nothing on the rules that exist precisely to keep internal process
narrative out of published text. **This is `L8`'s eleventh form on the scope axis** (*a clean scope and
an unread scope produce identical output*), and it is arch-tools work this team owns.

**Deliberately not fixed in this session, and the reason is the corpus's own:** expanding the scope
would light up 60 documents at once, and `.spec-baseline.json` only ever lowers — so it would gate
immediately. **A gate that is red on day one teaches people to skip it**, which is the reasoning that
already keeps `sdksync`'s 47 unpinned blocks at warn. **The right shape is: expand the scope, capture
the existing count into the baseline in the same commit, and let it ratchet.** That is a real task with
a real design and it is not a five-minute edit, which is why it is named here rather than done badly.

---

## §9 Sources

**In-corpus, opened this session by section:** `CANONICAL-DOCS.toml` (the declaration list, in full) ·
`ENTITY-SYSTEM-REFERENCE` §228 and the `entity://` dispatch examples · `SDK-OPERATIONS`,
`EXTENSION-COMPUTE`, `EXTENSION-SUBSCRIPTION`, `EXTENSION-CONTENT` (`entity://` usage, by grep and by
line) · `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4 `link-ref`, §11 the word-overloading rule ·
`APP-CONVENTION-EMBED` §3 (the tagged-union discipline and `G-PIN-2`) · `APP-CONVENTION-SHARE` §2.2 ·
`PROPOSAL-APP-CONVENTION-FEED` §2.2–§2.2.4.

**Carried from the two predecessors**, sourced there: NIP-19 · NIP-21 · NIP-65 · ATProto AT-URI ·
Matrix appendices (`via`) · IPFS delegated routing · Nix store paths and `narinfo` · magnet parameters.

**Measurements taken this session:** the 16/60 and 4/60 internal-citation counts, and the
`standards`-does-not-scan-`docs/research` result, are `grep` and gate-output over this tree at the
current commit.

## §10 What is unread, named so nobody assumes coverage

- **`entity://`'s grammar has no single normative home that was opened.** The scheme is used in at least
  seven documents and §0's claim about its meaning rests on `ENTITY-SYSTEM-REFERENCE` §228 plus the usage
  pattern. **Before a sibling scheme is minted, find or write the authority for the existing one** —
  and if there is none, that absence is itself the finding.
- **RFC 3986 / WHATWG URL normalization.** Design C's string form waves at escaping, case and
  percent-encoding without having opened either. **A string-form proposal must cite one of them
  properly.**
- **Both app-tier trees, still unopened.** Criterion 6 (seat cost) is scored from the specs alone. **A
  seat may already have built a link classifier**, which would move C's cost either direction. This is
  the third document to say so and it remains the cheapest unexecuted check on the board.
- **`EXTENSION-TYPE`'s pattern-matching rules**, which govern how a `field` path in `anchor` would be
  validated. Not opened; the `anchor = {field: [* tstr]}` sketch is illustrative and may collide with an
  existing path grammar.
