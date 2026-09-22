# PROPOSAL — ENCRYPTION: namespace ownership at Tier C, and §4.4 recipient resolution

**Status:** **DRAFT — reference proposal, written after the fold. See §0.**
**Target:** `EXTENSION-ENCRYPTION.md` §4.1, §4.2.a, §4.2.b, §4.4, §5.1, §16.1, §16.6;
`EXTENSION-IDENTITY.md` §4.2; `SPECIFICATION-FORMAT.md` §8.4.2/§8.4.4.
**Tier:** `extensions/` — validated by go · rust · py.
**Scope:** which extension owns the paths an encryption certificate lives at, and how a sender picks
a recipient key. **Not** the cipher suite, the tier ladder's existence, or the wire.
**Source:** `entity-core-go`, 2026-08-09, building `ENC-CERT-LIFECYCLE-1` and `ENC-RESOLVE-ORDER`.
**Cohort review:** ruled in-cycle. **rust and py have built neither half** (§6) — this is the arc
with the largest gap between ruled and built.

---

## 0. Process deviation — recorded, not hidden

**Folded across three commits (`a19234e`, `f855793`, `dbb4faf`) with no proposal.** It includes a
corpus-wide namespace-ownership rule and several new `[MUST]`s on sender-side behaviour.

Written now per the reconstruction ledger (arc D). Not back-dated. **Diligence pass run 2026-08-15
against core-go's own spec-issues — see §8.** One item from the filing document was found unanswered
and is now §5.3; everything else checked out.

## 1. Namespace ownership — six sites, one rule

core-go built `ENC-CERT-LIFECYCLE-1` at Tier C, **got a 404**, and traced it to IDENTITY's
fail-closed unbind: **ENCRYPTION named `system/identity/public/encryption/` for publication and
discovery, and IDENTITY defines no such subtree.** Two conformant peers miss each other and Tier C
resolves as `403 encryption_recipient_unknown` **with nothing erroring.**

**The framing correction matters more than the fix, and it is why this is not a dependency change.**
§1 already states ENCRYPTION does not require IDENTITY, and the Tier A/B/C ladder is a deliberate
**optional**-dependency design that stands unchanged. **The defect is namespace ownership *inside*
the optional tier.**

**Ruled: ENCRYPTION MUST NOT define, name, or write to any path segment inside
`system/identity/`** (`SPECIFICATION-FORMAT` §8.4.2/§8.4.4). At Tier C the cert is created through
`identity:create_attestation`, so **IDENTITY's canonical path function decides placement.** IDENTITY's
shape is `system/identity/{audience}/cert/{h}` routed by `properties.mode`; **`function` is a property
on the cert and never a path segment**, so an encryption cert is found by *filtering*
`properties.function == "encryption"`.

**core-go filed two sites; arch found six** by diffing every `system/identity/*` string against
IDENTITY's actual paths. **Only `internal/cert/{h}` was ever correct.** Four of the six live in
**unbuilt** features, which is why only two had bitten. **Folded as one general rule, not six
patches** — state-it-once, never-patch-instances.

Tier-C key-backup/key-share **collapse to the ENCRYPTION-owned A/B path** — arch ruled *against*
core-go's suggestion here, because those are ENCRYPTION's own artifacts and IDENTITY never reads them.

**Registration follow-on:** `EXTENSION-IDENTITY` §4.2 registers `function="encryption"` with
ENCRYPTION as sole registrant. **This was owed from 08-09, routed once, and never re-listed** — it
existed on disk in exactly three places, all of them ephemeral. Landed at `dbb4faf` with the
open-vocabulary hazard stated: **two extensions can adopt one string and nothing detects it**, which
is why registrants are now recorded in a table rather than a routing note.

## 2. §4.4 recipient resolution — three unpinned inputs and no observer

core-go built `ENC-RESOLVE-ORDER` to gate the tie-break ruling, and **building it surfaced that the
rule had no observer and three of its inputs were unpinned.**

**All three answers were already in the corpus, in surfaces ENCRYPTION does not own and had not been
read against.** That is the finding, and it recurs across this cycle.

**Q1 — the candidate is the inner pubkey entity at every tier.** `system/attestation` has **no
`created` field** (ATTESTATION §3.1) and `identity-cert` adds none — so **ordering carriers is
unimplementable**, and ENCRYPTION cannot mint the field: *the same §8.4.2 rule that decided the
namespace question, applied twelve hours later to a field instead of a path.* Filtering stays
carrier-scoped; ordering projects to the attested pubkey and **dedups by `content_hash`**.

Revocation was a **false binary** in the ask and now evaluates at **both granularities**:
pubkey-targeted kills the candidate outright; carrier-targeted kills only that carrier, and the
candidate survives iff a live carrier still attests it. **Revoking a per-relationship cert MUST NOT
retire the device's key.**

Two previously-unwritten MUSTs fell out: **"live" is three filters** (carrier validity, revocation at
both granularities, the key's own expiry), and **retrievability is part of resolution** — a candidate
whose inner entity cannot be fetched is *dropped and the walk continues*, because *publishes nothing*
/ *only revoked* / *cannot read it* are one fact at the sender.

**Q2 — the full 33-byte `format‖digest` form**, pinned as **byte-for-byte the value bound as
`recipient_key`** so it stops being a choice; unsigned lexicographic, shorter-prefix-first for
totality. *(The Q2 text originally read "33 bytes under SHA-256" — itself an instance of the
hash-width class the 08-10 sweep then ruled and gated. See `PROPOSAL-HASH-WIDTH-IS-NEVER-FIXED`.)*

**Q3 — `expires` participates in sender-side selection ONLY.** A receiver **MUST NOT** refuse to
decrypt on expiry: data at rest outlives the window and in-flight material must not become
undecryptable. **Expiry retires a key from selection; revocation retires it from use.**

## 3. §16.6 — a MUST with no probe, and why that is a property of the rule

`ENC-RESOLVE-ORDER-1` is added to §16.1 (floor, all tiers) and defined in new **§16.6** as a
**pinned-input differential, explicitly not a probe**. Resolution is sender-side and **ENCRYPTION
defines no peer-facing encrypt operation, so no probe exists to write** — a property of the rule, not
a validator gap.

Required properties: coverage, order-independence, **a negative control as a MUST** (each wrong
resolver caught by a *named* row; any guard that cannot fire against a correct impl needs its own
injected fault), and **declared exclusions in the file**.

Generalized into `GUIDE-CONFORMANCE` **§5.2a** — *a `[cross-peer seam — MUST]` with no peer-observable
surface MUST be gated by a pinned-input vector **in the same change that lands the MUST*** — written
against arch's own failure, since arch had landed the tie-break as prose two days running. That rule
now lives in `PROPOSAL-CONFORMANCE-COVERAGE-FAILURE-TAXONOMY`.

## 4. The sharpest finding in the arc

**Not three impls agreeing — one impl counted three times.** §16.6's own instance: go's resolver,
called by go's own validator, scoring rows against peers it never contacted, **on the one rule whose
entire justification is that implementations must not diverge.**

That is why the self-check accounting changed in the same cycle: 29 client-free checks MUST be
declared and flagged but are **not removed** — for a rule like §4.4 the self-check *is* the only
possible gate — and per-peer figures now carry a **peer-attributable count** alongside the total.

## 5. Open questions for the cohort

1. **`expires` at Tier C** — ATTESTATION §4.3 already enforces carrier expiry with a vector and
   IDENTITY §9.3 already makes it a MUST, so the "we ignore `expires`" gap was **wider and older**
   than flagged. Has any seat audited supersession as an entry point?
2. **The open-vocabulary hazard on `function`** has a table and no gate. Two extensions adopting one
   string is undetectable today.
3. ~~**§4.4 step 1 cannot reach a `per-relationship` or `embedded` Tier-C cert**~~ — **RULED AND
   FOLDED 2026-08-15.** See §5.3a for the ruling; the statement of the gap is kept below verbatim
   because it is the record of what was filed and how long it stood. `[filed by go 2026-08-09;
   never answered; found unanswered by review 2026-08-15; ruled 2026-08-15]`

   **This was in the filing document and neither the ruling nor the first-pass proposal carries it.**
   go's spec-issue closes with *"two smaller points that travel with it"*; point 1 (register
   `function="encryption"` in IDENTITY §4.2) was folded at `dbb4faf`. **Point 2 was not:**

   > *"Per-relationship and embedded Tier-C certs are unaddressed. §4.2.c speaks only of the public
   > handle. A `mode="per-relationship"` encryption cert is expressible and would land under
   > `system/identity/relationships/{contact_id}/cert/` — where a sender walking only the public
   > subtree never sees it. **Either say those modes are out of scope for encryption certs, or say how
   > discovery reaches them.**"*

   **Confirmed still open in the landed text**, which now states both halves and never joins them:

   - §4.2 note: IDENTITY's shape is routed by `properties.mode` — `internal` / `public` /
     `per-relationship`; **`embedded` has no tree path.**
   - §4.3: *"For **per-relationship hygiene** … the relationships path applies (Tier C only)"* — so
     publishing one is explicitly conformant.
   - §4.4 step 1: enumerates **only** *"`system/identity/public/cert/` for the discoverable case."*

   **So a conformant per-relationship encryption cert is unreachable by the resolution walk**, and the
   sender falls through Tier C → B → A to `403 encryption_recipient_unknown` — **with nothing
   erroring, which is verbatim the failure this arc was opened to fix** (§1). §4.4's own preamble
   promises the opposite: *"the sender doesn't need to know the recipient's tier in advance — the
   sender walks the namespace and finds whichever publication shape the recipient uses."* An
   `embedded` cert is unreachable by construction, having no path at all.

   **This wants a ruling, not a patch, and the two conformant readings diverge across a peer
   boundary** — one implementer scopes encryption certs to `mode=public` and refuses to publish
   otherwise; another publishes per-relationship and expects senders to find it. Per `AGENTS.md`,
   lean MUST. **Recommendation: state the scope** — encryption certs are `mode=public` for
   third-party discovery, and a per-relationship cert is resolvable **only** by the contact whose
   `contact_id` names the path, as an explicit step 1a rather than an inference.

### 5.3a The ruling — Tier C is a two-step walk, most-specific first `[2026-08-15]`

**Ruled wider than the recommendation §5.3 carried, and in the opposite direction on one point.**
§5.3 recommended *"state the scope — encryption certs are `mode=public` for third-party discovery,
and a per-relationship cert is resolvable only by the contact whose `contact_id` names the path."*
The scope half is ruled as recommended. **The "only by that contact" half was under-specified**: it
says who *may* resolve one and still never says the sender walks that path, which is the sentence
whose absence was the entire defect.

**Folded into `EXTENSION-ENCRYPTION` §4.4 step 1, split into 1a/1b, plus §4.3 and §16.6:**

- **1a — the per-relationship subtree**, keyed by the **sending peer's own** `system/peer` identity
  hash, in IDENTITY §5.1's `[derive-to-meet]` floor-pinned form. **1b — the public handle.**
- **They are separate steps, not one merged candidate set `[MUST]`**, with the tier ladder's own
  semantics reused verbatim: most-specific wins, and a **wholly-dead** per-relationship subtree
  **falls through** rather than terminating.
- **An absent or unreadable relationships subtree is not an error `[MUST]`** — it is the ordinary
  state for a sender with no bilateral relationship.
- **`internal` and `embedded` are outside the walk `[MUST]`.** A sender reads two paths and only
  two. `embedded`'s out-of-scope status is IDENTITY §4.2's registration row to state, and it already
  does — referenced, not restated.

**Why a sub-ladder rather than a union, which is the part that needed deciding.** Merged into one
candidate set, §4.4's total order picks between a dedicated per-relationship key and the public
handle by `created` — so **a later-minted public key silently shadows the dedicated one**, and the
publisher cannot observe it: it published a key for this contact and this contact encrypted to
another. That is this arc's own failure mode a third time, in the resolver instead of the path. The
sub-ladder reuses an existing shape and introduces no concept.

**The determinism objection, answered in the text because it will otherwise be re-derived.** Two
senders resolving the same recipient may now bind different keys. That does **not** weaken the
cross-peer order this arc pinned: each sender sees its own relationships subtree and no other's, so
the requirement is per **(sender, recipient)** pair, and both ends of one pair compute the same set
from the same bytes. Without that sentence the rule reads as a contradiction of §2's convergence
point.

**Observer.** §16.6's coverage list gains the sub-walk, **with the discriminating direction pinned**
— a live per-relationship candidate selected over a **newer** live public one, so a merged-union
resolver **fails** the row instead of passing it by coincidence. That is §3b.3.1's
"a vector every candidate construction passes tests nothing" applied here.

**Disposition:** cohort finding, fixed in place, **no rev bump** — the same disposition the rest of
this arc took. **The obligation it creates is new**: no seat has a §4.4 sub-walk, and go's resolver
is the only §4.4 resolver in the cohort at all (§6).

## 6. Build state — the largest ruled-but-unbuilt gap in the corpus

**Source-read 2026-08-15 — and then RE-TAKEN the same day, because all three HEADs moved while
§5.3a was being ruled** (go `2df96f8`→`2f4f6a2`, rust `462f2c2`→`0b4e0cd`, py `f33526f`→`c0de5c2`).
**Both readings agree; the rows below are the second one**, at go `2f4f6a2` *(worktree dirty, 3
files)* · rust `0b4e0cd` clean · py `c0de5c2` clean. The sibling movement was `reflection_endpoints`
and the methodology overlay, neither of which touches encryption — **which is knowable only after
looking, not before:**

- **§4.4 resolver: go only** (`ext/encryption/tier_resolver.go`). **rust and py have no §4.4 resolver
  and no Tier C.**
- **§5.2 self-check labelling: go only.** 29 client-free checks are in all three published totals;
  only go labels them. Arch ruled keep-in-total-and-publish-both, **landing in all three in the same
  cycle** — 1-of-3 is the state the ruling itself named as worse than either convention applied
  uniformly.

**Both rows are tracked as W-CORPUS C6.** *Method note: a name-search for "Tier C" in rust hits
shell-verb tier vocabulary in `bindings/shell/`, an unrelated namespace — the
named-search-hits-the-name shape. The rows above were confirmed by opening the encryption tree.*

## 7. Fold plan

**Already folded.** Ratification means the namespace rule and the §4.4 pins stand. The **§16.6 vector
is owed from the oracle in rust and py**, and until it exists §4.4 is a MUST that one implementation
checks against itself. ~~**§5.3 is a spec gap the fold left open and needs a ruling before it
folds.**~~ **§5.3 was ruled and folded 2026-08-15 — §5.3a.**

**Why this stays `active/` with every arch-side edit landed.** Nothing here is waiting on us. It is
waiting on the one thing this arc has never had: **cohort review as a package** (queue Q14). §0
claims *"if review rejects a shape, the spec edit comes back out"* — untested on this proposal and on
the other six. **Closing the slot now would retire that claim by declaring it rather than exercising
it**, and §5.3a is the newest and least-reviewed ruling in the file. It is not held open for builds:
the §16.6 vector and the rust/py resolvers are the cohort's, recorded in §6 and routed, and **arch
does not track what implementations owe.**

## 8. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8.

**Sources opened (go's own documents, not arch's commit messages):**
`docs/validation/spec-issues/2026-08-09-encryption-tier-c-names-an-identity-subtree-identity-does-not-define.md`
(including its `RULED — 2026-08-09` banner, which is arch's answer recorded *in go's tree*) ·
`.../2026-08-09-b-the-4-4-total-order-is-a-cross-peer-must-with-three-unpinned-inputs.md` (status
**FULLY RULED — closed 2026-08-11**; its §6 lists **four** asks). Build state read live in
`entity-core-rust` `extensions/encryption/src/` and `entity-core-py`
`packages/entity-handlers/src/entity_handlers/encryption/`.

**Build state re-pinned 2026-08-15:** go `2df96f8`, rust `462f2c2`, py `f33526f`, all clean.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **D1** | A | **An item from the filing document was never answered and never recorded** — go's second "smaller point": per-relationship / embedded Tier-C cert discovery. Confirmed still open in the landed text, and it reproduces this arc's own failure mode in a different publication mode | **New §5.3** |

**This is the same shape as `REVISION` v3.11's two-of-seven**, at a smaller scale and with a
difference worth stating: **here the filing seat's document *was* read** — the ruling banner in it
answers three corrections and the performance question — **so the miss was not "we never opened it."
It was that the items after the main finding are the ones that get dropped.** go's own phrasing
("two smaller points that *travel with it*") is a warning label. **Count the asks, including the
ones the filer calls small.**

### What the review confirmed as correct

- **§1's "core-go filed two sites; arch found six"** — go's own banner says exactly this, and draws
  the same lesson (*"when a defect is a category error, diff the whole category"*). Correctly credited.
- **§1's framing correction** — go's banner records *"our framing was wrong … the defect was namespace
  ownership inside the optional tier"* as **their** concession. The proposal states it as the arc's
  finding without over-claiming it.
- **§1's rejected suggestion** — go proposed Tier-C key-backup / key-share keep a Tier-C variant; arch
  ruled they collapse to the ENCRYPTION-owned A/B path. **The proposal records arch ruling against the
  filer, which is exactly what §4 of the reconstruction plan asks for** (do not invent consensus).
- **Completeness on §4.4** — go's §6 asks **four** things (Q1, Q2, Q3, Structural). All four are
  covered: Q1–Q3 in §2, Structural as §16.6 in §3. **Counted, not assumed.**
- **§6's build state, re-verified at source rather than carried:** `extensions/encryption/src/` in rust
  holds fifteen modules and **no Tier-C or §4.4 resolver** (no `identity-cert`, `tier_c`, or
  recipient-resolution symbol anywhere in it); py's encryption package likewise — its `identity-cert`
  handling is in `identity.py`, the IDENTITY handler, not a resolver. **C6 stands.**

### Not covered by this pass

- **§5.1's supersession question** (open question 1) was not audited against any seat's tree — it asks
  whether a seat has audited supersession as an expiry entry point, and no seat has answered.
- **The §16.6 vector's required properties were not checked against go's implementation** of
  `ENC-RESOLVE-ORDER-1` — only that rust and py have none.
