# PROPOSAL — the `put` admission predicate, and the two handlers whose codes were never a set

**Status:** IMPLEMENTED — folded at `ENTITY-CORE-PROTOCOL` **0.8.2.11** · `EXTENSION-TREE` **v4.5** ·
`EXTENSION-CONTENT` **v3.7** · `DOMAIN-LOCAL-FILES` **v1.4** · `ENTITY-NATIVE-TYPE-SYSTEM` (no version
field) · `SDK-OPERATIONS` §3.2. All of D1–D9 landed.
**Tier:** core + extensions — a normative delta across both repos this team owns.
**Answers:** `entity-core-go` `ROUTING-2026-09-06` §§1a–1e and §2, filed with rust and py data merged.

> **What this proposes, in one sentence.** `EXTENSION-TREE` Appendix A v4.4 mandated `400
> invalid_request` when a submitted entity *"does not decode"* and never said what decoding is — and
> the answer was already in the corpus, because `put-request.entity` is typed **`core/entity`**, whose
> three fields are all required; naming that one type answers four of the six routed items at once,
> and the fifth is a table that was never written.

**The filing seat asked for five sentences and one ruling. Four of the five are one rule** — *admit
the value as a `core/entity` before comparing its hash* — **and the sixth item was scoped by a
contradiction rather than by a gap.**

**Measured at** `entity-core-go` **`cb1737b`** · `entity-core-py` **`89d3514`** · `entity-core-rust`
**`06404cb`** · `entity-system-generator` **`0a5350c`**, by opening every file cited below. **py and
rust have both moved since the filing seat measured them** (`ca62657` → `89d3514`, `27a1dc3` →
`06404cb`); every impl claim here is re-taken at the commits above, not carried forward.

---

## §1 The predicate was never missing — it was written in a type nobody cited

`ENTITY-CORE-PROTOCOL` §3.9 declares:

```
system/tree/put-request := {
  fields: {
    entity: {type_ref: "core/entity", optional: true}
    ...
```

and `ENTITY-NATIVE-TYPE-SYSTEM` §8.1 declares `core/entity` with **three fields and no `optional`
marker on any of them**, stating the consequence in prose one line down: *"raw CBOR values that don't
match the three-field structure are rejected."* `ENTITY-CORE-PROTOCOL` §1.1 states the same rule from
the other end — *"All entities MUST include `content_hash`"* — and names **entity embedding in params**
as one of the boundaries where it binds. §3.4 states it a third time for `params` and `result`
themselves: *"they contain materialized entities `{type, data, content_hash}`, **not raw data
values**."*

**So *"does not decode"* has a definition, and it is: this value is not a `core/entity`.** That
resolves §1a, and it resolves §1c and §1d as a consequence rather than as separate rulings.

### §1.1 One home said otherwise, and it is the only one describing the wire

`ENTITY-NATIVE-TYPE-SYSTEM` **§2.8** — *Value Fields vs Entity Fields (**Wire Shape**)* — carried:

| `type_ref` form | Wire shape |
|---|---|
| `type_ref: "core/entity"` | Entity envelope — `{type, data, content_hash?}` |

**Six homes state the three-field form; that one carried a `?`.** Enumerated by subject across both
repos: §1.1, §3.4 and the §1.1 note in `ENTITY-CORE-PROTOCOL`; §166, §244, §1010 and §8.1's type
definition in `ENTITY-NATIVE-TYPE-SYSTEM`; plus four restatements in `EXTENSION-INBOX` and
`EXTENSION-TYPE`. **Every one of them is the strong form. §2.8 is alone.**

It is also the one that matters most, and that is not a coincidence: **§2.8 is the only table in the
corpus that says what is *on the wire* at a `core/entity` slot**, which is exactly the surface
`put-request.entity` occupies. A reader deciding what a peer must accept from the wire reads §2.8, and
§2.8 said the hash was optional.

**This is L23's fourth shape and it is why the fix is two edits, not one.** A restatement that does not
name its authority is invisible from the authority — nobody reading §8.1 can see that §2.8 disagrees.
So §2.8 now carries the corrected shape **and** the sentence *"§8.1 is the authority for `core/entity`;
the row below is a restatement of its shape, not a second definition of it."*

**What is not claimed:** that §2.8's `?` caused any implementation's behaviour. Nobody said it did, and
we did not ask. It is a contradiction in the corpus that reads as licence, and it is enough that a
conformant implementer could have arrived at the lenient reading honestly.

---

## §2 Ordering (§1b) is a data dependency, not a convention

The filing seat is right that the two 400 rows are individually satisfiable and jointly ambiguous, and
right that **no per-row vector can discriminate them** — a row-1 vector carries no hash to disagree
with, so a hash-first peer falls through to the structural branch and passes.

But the order does not need to be chosen. **Step 2's inputs are exactly what step 1 establishes:** you
cannot compare a carried `content_hash` against `content_hash({type, data})` until you know there *is*
a `content_hash`, a `type` and a `data`. Structure-before-hash is what the dependency already is; the
spec's only failure was not writing it down.

**And the corpus already runs this discipline, one section over.** §9.1 pins, for §4.6's
proof-of-possession ladder: *"for an input failing more than one numbered step, the emitted code is the
**lowest-numbered** failing step's."* The `put` rule is that rule on a two-step ladder, and it is
written to look like it.

---

## §3 §1c — the dichotomy is false, and the authoring step is the SDK's

The seat framed this as *receipt path (§1.8) **or** authoring path (§4.5a)*, and asked arch to pick.

**§4.5a is not an authoring arm.** It governs *which `content_hash_format` an author uses*, not
*whether a receiver may author*. It presupposes an author and says nothing about `put`.

**Both paths exist. They are at different layers, and the corpus says where.**
`SDK-OPERATIONS` §3.2 specifies:

```
put(path: string, type: string, data: any) → hash
```

An unhashed payload in, a content hash out. **The SDK cannot return that hash without computing it**,
which means the SDK constructs the `core/entity` — and that construction *is* the authoring step. The
wire operation it dispatches to is a receipt path (§1.8 item 1), and the two compose exactly.

So the ruling is not *"go's reading or the other two's."* It is:

| Layer | Rule | Measured state |
|---|---|---|
| **Peer** (`system/tree:put`) | Receipt. Validate the carried hash; **never** author one | **go** conformant in policy, **wrong in code** (below). **rust, py** author on absence — non-conformant |
| **SDK** (`put(path, type, data) → hash`) | Author. Construct the entity, compute the hash | **rust `06404cb`, py `89d3514`** both send `{type, data}` — non-conformant |

### §3.1 Every seat is non-conformant here, and no seat's own tests could see it

- **go `cb1737b`** — `core/entity/entity.go:121-133` checks empty `type` and empty `data` first
  (correct order), then `hash.Validate` (`core/hash/hash.go:161-170`) recomputes under
  `claimed.Algorithm`, which for an absent hash is the zero `Hash{Algorithm: 0x00}` → mismatch →
  **`400 hash_mismatch`**. The refusal is right and **the code is wrong**: an absent required field is
  a structural defect, so it is row 1, `invalid_request`. go's own step-ordering rule convicts it —
  the hash check is step 2 and the missing field is step 1.
- **py `89d3514`** — `packages/entity-sdk/src/entity_sdk/client.py:373`:
  `payload = {"entity": {"type": type, "data": data}}`. Two keys. The SDK never constructs.
- **rust `06404cb`** — `bindings/sdk/src/sdk.rs:4292-4299`, `build_put_params`: the caller hands in an
  `Entity` whose `content_hash: Hash` is **not optional**, and the function builds a two-key map from
  its `type` and `data`. **It has the hash and drops it.**

**The reason six weeks of green tests missed this is structural and worth keeping.** At rust and py the
lenient peer and the stripping SDK are **in the same tree**, so each seat's round-trip works
perfectly — the peer authors exactly the hash the SDK declined to. **The defect is only observable
across a seat boundary**, which is why it surfaced the day py drove a live go peer and not before. A
compensating pair inside one repo is invisible to that repo's entire test suite, by construction.

### §3.2 The predecessor fold said this was not a flag day, and it was

`PROPOSAL-THE-400-SLOT-FALLBACK-AND-THE-TREE-PUT-ROWS` §7 recorded *"Emit-side only. **Not a flag day,
no divergence unit**"* and §8 recorded *"**None from this fold.**"* Both were wrong, and the second was
wrong the moment anyone implemented the first: **v4.4 tabulated the codes for an operation nobody had
ever driven**, and driving it produced five under-specifications, a phantom operation, and a live
cohort break in which **no rust or py SDK can `put` to a go peer at all**.

**This is L21's third shape on the row axis rather than the field axis** — *a ruling that makes a
previously-unenforced check load-bearing names its divergence unit* — and it is not a new rule. The
divergence unit is stated here, in §5, because it should have been stated there.

**The tell was available:** a fold that tabulates error rows for an operation with no existing
conformance check is not describing behaviour, it is **commissioning a first measurement**, and the
first measurement of anything finds something. *"No open items"* is not a claim a fold like that can
make about itself.

---

## §4 §1d — `type: ""`, and §1e — `set` does not exist

**`type: ""`.** Refused, `400 invalid_request`, folded into the §1 predicate. The derivation is
§2.7's — *"the name is the interop contract"* — plus the precedent the corpus set at **0.8.2.4**: a
`hello` whose `protocols` is **absent or empty** is `400 invalid_request`, *"a malformed request, not a
version incompatibility."* Present-but-empty is malformed, and it is malformed for the same reason: a
required field carrying no value has not been supplied. An empty interop contract is not a contract.

This also closes the case the seat flagged in passing — a **non-string** `type` (py stored an entity
typed integer `42` before this arc, under a hash no typed peer can decode). `core/entity.type` is
`primitive/string`, and §2.4 says type checking is **strict**; the predicate says *text-string*, so the
class is closed by rule rather than per seat.

**`set`.** Dropped from the rows. Three independent corpus sites define the tree's operation set as
exactly two: `ENTITY-CORE-PROTOCOL` §6.3 (*"All tree reads and writes go through this handler via **two
operations**"*), `EXTENSION-TREE` §2.2 (*"The **Two** Index Operations"*), and §4198's `--profile core`
expectation (`{"get", "put"}`). **The word does live in this corpus** — `EXTENSION-REVISION` spells a
change record's `action` as `"set" | "delete"` — and that is worth recording so nobody re-imports it: a
value in an adjacent vocabulary, not an operation.

---

## §5 §2 — the content vocabulary, and why the framing needed correcting first

The seat filed this as *"two spec files contradict each other on `system/content:ingest`'s rule."*
**They do not, and the difference matters for the remedy.** `EXTENSION-CONTENT` §6.3 governs
`system/content:ingest`'s `envelope`/`entity` modes; `DOMAIN-LOCAL-FILES` §4.3 governs
`local/files:write`'s `bytes`/`content` modes. **Two handlers, two operations, one *shape*** — which
LOCAL-FILES says outright (*"the shape parallels CONTENT §6.3's envelope-or-entity input-mode
discipline"*) and then spells differently. It is not a contradiction to adjudicate; it is **one
condition with two vocabularies**, and the fix is convergence, not precedence.

**The ruling: the token is the `code`, never a label in the `message`.** Derived, not counted:

1. §3.3 types `message` as **optional** and human-readable, and `code` as *"the programmatic
   identifier."* A discriminator a caller must branch on cannot live in a field a conformant peer may
   omit entirely.
2. §3.3's own slot rule (0.8.2.9) names the remedy for a needed specific: *"the peer holds it as a
   **named divergence and routes it** for a **table row** rather than minting a site."* A parenthetical
   in the message is precisely the mint-a-site workaround that rule forecloses.
3. The unit of conformance is **the code slot**. A label in `message` is not in the slot, so a peer
   emitting it satisfies nothing and a check reading `result.data.code` cannot see it.

So `EXTENSION-CONTENT` §908/§910 were right and `DOMAIN-LOCAL-FILES` §4.3 moves.

### §5.1 Censusing the slot found six more codes than the packet named

The ask was for two codes. **Grouping by emit site rather than by token** (L20) found that
`EXTENSION-CONTENT` has **ten `error(...)` sites carrying eight distinct codes, six of which name no
status at all**, and that `DOMAIN-LOCAL-FILES` spells one condition two ways in one document:

| Finding | Detail |
|---|---|
| CONTENT `403 forbidden` (§6.4) | **Defined nowhere.** §3.3's 403 default is `capability_denied`, and 0.8.2.9 says an undefined spelling falls back to the default — so the spec's own pseudocode emitted a non-conformant code. Corrected |
| CONTENT `blob_pending_sync` (§3.4) | Carried no status. It is **retryable** — the blob is present, a chunk has not arrived — which is a different remedy from `blob_not_found`. Pinned at **503**, distinguished from 404 by exactly the argument v4.4 used to split `hash_mismatch` 400 from 409 |
| CONTENT `missing_chunk` / `empty_chunk` / `size_mismatch` (§3.3) | Returns of `verify_content`, an **internal predicate over locally stored state** — not wire codes, which is why they were written without a status. Recorded as such rather than assigned one; where such a failure reaches the wire it is **500 `storage_error`** |
| LOCAL-FILES `no_root_mapping` ×4 vs `root_mapping_not_found` ×1 | **One condition, one document, two codes.** Invisible to any check keyed on either spelling. Converged |

**The 404-naming rule that came out of that last row is the part that generalizes.** Both spellings
were defensible and neither was derivable, so rather than pick, the convergent form is stated as a
rule: **`{noun}_not_found`**, which is what the corpus already does everywhere else it names a 404 —
`handler_not_found` (§3.3), `tree_not_found` / `snapshot_not_found` / `capability_not_found`
(`EXTENSION-TREE` Appendix A), `blob_not_found` (`EXTENSION-CONTENT` Appendix A). The next 404 code in
any extension is now named by a rule instead of by taste, and does not need a ruling.

### §5.2 This one is time-critical, and that is a fact about a repo that had not been asked

`entity-system-generator` shipped **its first extension the day before this fold** — CONTENT ×
TypeScript, `0a5350c` — and it emits `ambiguous_input` / `missing_input` / `hash_mismatch`,
**following CONTENT's pseudocode exactly**, with its own gate asserting those codes
(`extensions/content/typescript/test/handler.test.ts:323-336`).

**It is right, and it was right before this ruling** — which is the point, not a vindication. A
generator's output scope is its input scope; had this been ruled the other way, its first extension and
its first gate would both have been wrong, and **every substrate generated after it would have
inherited the wrong vocabulary uniformly, from the first commit.** The window in which that is a
one-file fix rather than a cohort-wide sweep is measured in days, and it opened on 2026-09-05.

**What this is *not* is evidence for the ruling.** The derivation above stands on §3.3's own text and
would be identical had the generator emitted the other spelling — L18's operative test, applied and
passed. It is recorded because it changes the *urgency*, never the answer.

---

## §6 Delta table

| # | Document | Section | Change |
|---|---|---|---|
| **D1** | `ENTITY-NATIVE-TYPE-SYSTEM.md` | §2.8 | `{type, data, content_hash?}` → `{type, data, content_hash}`, *"all three keys are required"*, plus the authority sentence naming §8.1 |
| **D2** | `ENTITY-CORE-PROTOCOL.md` | §6.3 | The `put` admission rule — receipt not authoring; the two ordered steps; the predicate; the `unsupported_content_hash_format` carve-out; structural ≠ semantic; and where the authoring step lives |
| **D3** | `ENTITY-CORE-PROTOCOL.md` | §9.1 | One conformance row — the admission ladder, its three codes, and the note that the ordering's discriminating input carries both faults |
| **D4** | `ENTITY-CORE-PROTOCOL.md` | line 3 | Version → **0.8.2.11** |
| **D5** | `EXTENSION-TREE.md` | Appendix A | Predicate stated on row 1; ordering stated on row 2; `unsupported_content_hash_format` row restated from §4.7 row 5 with its authority named; **`set` dropped from all rows** |
| **D6** | `EXTENSION-TREE.md` | §2.2 | *"Does not validate"* scoped to semantics, so the paragraph stops reading as licence to skip admission |
| **D7** | `EXTENSION-CONTENT.md` | Appendix A (new), §6.4, line 3 | The handler's code table; `403 forbidden` → `capability_denied`; `blob_pending_sync` at 503; the internal-predicate note; the token-is-the-code rule; **v3.7** |
| **D8** | `DOMAIN-LOCAL-FILES.md` | §4.3, §4 pseudocode, Appendix A (new), line 3 | `invalid_params` + message-label → `ambiguous_input` / `missing_input` citing CONTENT as authority; `no_root_mapping` ×4 → `root_mapping_not_found`; the code table; the `{noun}_not_found` rule; **v1.4** |
| **D9** | `SDK-OPERATIONS.md` | §3.2 | The SDK constructs the entity and computes the hash; the MAY branch MUST NOT drop a hash it was given |

**Class and satisfaction mode (`GUIDE-CONFORMANCE` §7.0, §5.2b.1).** Every new normative row here is a
**`validate-peer` behavioral check** — driven over the wire against a live peer, oracle-authored — and
every one is **constructible**: a `put-request` carrying a two-key entity, an empty `type`, a
non-string `type`, a mis-sized `content_hash`, or a both-faults input is an ordinary EXECUTE. The
ordering row is the only one needing care: its input must carry **both** faults, which is exactly the
input a row-scoped author does not write, so it is named in D3 rather than left to be inferred.

## §7 Cohort impact — **this is a flag day on `put`, and the divergence unit is named**

**During adoption, a seat that has landed the strict predicate refuses a two-key `put` from a seat
whose SDK has not landed construction.** That refusal already exists today in one direction — go
refuses rust and py — so landing go's code fix first *widens* nothing, but landing a peer fix at rust
or py **before** their own SDK fix breaks their own round-trip. **Order within each seat: SDK
construction first, peer strictness second.** Across seats the order does not matter.

| Seat | Owed |
|---|---|
| `entity-core-go` | **1 site**: the absent/mis-sized `content_hash` arm moves `hash_mismatch` → `invalid_request`. `core/entity/entity.go:121-133` already checks structure first — add the `content_hash` presence and length clauses to that block, above `hash.Validate`. Empty `type` is already right. Then add the both-faults ordering vector to `tree_put_error_codes.go` |
| `entity-core-rust` | **SDK first** — `build_put_params` (`bindings/sdk/src/sdk.rs:4292`) drops a hash it holds; emit all three keys. **Then peer** — stop authoring on absence, refuse `400 invalid_request`; same for empty / non-string `type`. Separately: `system/tree/put/params` is used at 4 sites where go and py use the §3.9 type name **`system/tree/put-request`** — one of the two is wrong and it is not the two seats that agree |
| `entity-core-py` | **SDK first** — `client.py:373` builds `{"type", "data"}`; construct the entity and send its hash. **Then peer** — refuse rather than author. The row-1 work landed at `ca62657` already covers non-map / missing-`type` / missing-`data` |
| `entity-core-keystone` | Vendor step: **four** pinned documents moved, three of them core. Their snapshot is at **0.8.2.3**; this is 0.8.2.11 |
| `entity-system-generator` | **CONTENT × TypeScript is already conformant to the ruled vocabulary** and needs no change. `blob_pending_sync` at 503 and `capability_denied` at 403 are new inputs if `get` is generated |
| `entity-browser-rust`, `entity-workbench-go` | Any SDK-layer `put` wrapper: confirm it constructs rather than forwards `{type, data}` |

## §8 Open items

**Three, and none of them blocks the fold.**

1. **`system/tree/put/params` vs `system/tree/put-request`** (rust, 4 sites). Not ruled here because it
   was not routed here and rust has not been asked; §3.9 names the type and two seats already use it,
   so the expected answer is convergence, but that is rust's to confirm in their tree.
2. **rust's `put_cas` doc comment** (`sdk.rs`, above `put_cas`) states their tree handler *"does not
   yet support native CAS via `expected_hash`"* and that the SDK does get-compare-put. The filing seat
   reports rust **passing** the 409 CAS-race check on the wire at `27a1dc3`. Both cannot be current.
   **A doc comment is an artifact (L8) and neither this note nor that comment is evidence** — routed to
   rust to say which is stale.
3. **OP-3 remains open.** Three of the extension family's error-code tables now exist (`TREE`, `CONTENT`,
   `LOCAL-FILES`) plus the three landed 2026-09-04. This is not a cohort sweep and MUST NOT be routed as
   one; the per-extension tables land as each surface is driven, which is how these two were found.
