# PROPOSAL — `published-root`: land the type, carry its prefix, and pin the republish obligation

**Status:** **RATIFIED + FOLDED 2026-08-08.** Cohort review happened — `entity-core-go` (lead implementation)
adopted **D1–D7 as written** and answered all four §9 questions, `entity-core-rust` raised the defect, and both
corrections it produced touched evidence rather than any delta. Per `AGENTS.md` (*a well-reviewed proposal lands
as a versioned spec file — don't manufacture extra review cycles*), **all seven deltas are folded**:

| Delta | Landed in |
|---|---|
| D1 published-root type + `prefix` | `EXTENSION-TREE.md` **§3.3a** (new) |
| D2 absolute-prefix trim + 3-case table | `EXTENSION-TREE.md` §3.3 |
| D3 universal-case no-op clarification | `EXTENSION-TREE.md` §3.1 |
| D4 dangling-pointer fix | `EXTENSION-NETWORK.md` §6.5.3 (×2) |
| D5 bounded-convergence republish MUST | `EXTENSION-NETWORK.md` §6.5.6 |
| D6 prefix-scoped closure extent | `EXTENSION-NETWORK.md` §6.5.6 (A10) |
| D7 conformance assertions | `EXTENSION-TREE.md` §12.1 |

**D5's open value is ruled, not deferred: maximum convergence delay = 30 s**, derived from the 10 s
measured-achievable figure across all three impls with 3× headroom (§5). It is **falsifiable on measured
evidence** and fixes in place if load data contradicts it — arch is not holding the fold for a number that the
cohort can correct cheaply. *(Earlier draft gated the whole fold on this; that was wrong — Q2 touches D5 alone
and six deltas were blocked on an unrelated question while `entity-core-go` could not build §6.1/§6.2 without
D1/D2.)*

This file is now the **design record**; the specs are source of truth.
**Raised by:** `entity-core-rust` (2026-08-08) → filed by `entity-core-go` @ `8a25d90`+
(`docs/validation/spec-issues/2026-08-08-published-root-does-not-carry-the-prefix-its-trie-keys-are-relative-to.md`),
measured at `e77272f`, cohort closeout `eebc7c0`.
**Specs touched:** `EXTENSION-TREE.md` §3.1/§3.3/§3.4.1/§3.4.1a · `EXTENSION-NETWORK.md` §6.5.3/§6.5.6 (A10)
**Blocks:** nothing implementation-side is owed until this lands; it is the **only** open arch item from the
2026-08-08 cohort closeout (all three impls 0 F / 0 S).

---

## §0. Summary

Three conformant implementations publish trie roots whose keys are **mutually unreadable**, and every one of
them is defensible under the text as written. The cohort cannot adjudicate it because no test can: the divergence
is in a value the entity does not carry.

Three deltas, in dependency order:

1. **Land `system/peer/published-root` in this corpus** — it is referenced by landed normative text and defined
   only in a legacy archive (§2). This is a provenance defect, and it is the reason the other two went unnoticed.
2. **Add a `prefix` field** to the entity, and fix §3.3's trim formula, which is ill-defined for the universal
   case (§3, §4). This resolves Q1 and Q2 **without any implementation changing its key convention.**
3. **Pin the republish obligation** (§5) — currently unspecified, which silently voids the 2026-08-07
   closure-timing MUST that arch folded on the strength of it.

---

## §1. What was found, and what it is not

`entity-core-go` built `published_root.v8_trie_key_convention` — it mirrors the served trie over the wire from
`published-root.root_hash`, collects every `(relative_key → value_hash)` pair, rebuilds locally, and asserts the
rebuilt root equals the served one. Measured on fresh peers, one per impl:

| Impl | Observed keys | Bindings | Nodes | Structural equality |
|---|---|---:|---:|---|
| Go | `attestation`, `capability`, `peer/published-root/…` | 401 | 34 | **PASS** |
| rust | `system/attestation`, `local/files` | 415 | 35 | **PASS** |
| python | `/2K3vawud…/system/signature/0083fd…` | 490 | 35 | **PASS** |

**The good half is a genuine first, and it is worth more than the defect.** The trie *algorithm* now has a
three-way byte-identical result: routing by `SHA-256(canonical-normalize(key))`, `bitWidth=5`, `bucketSize=3`,
the §3.1 canonical-form invariant and the bucket-sort invariant all agree. That had never been measured.

**The `v8` PASS must be read narrowly, and this constrains the fix.** The rebuild takes its keys *from* the trie,
so it is self-consistent by construction with respect to key *form* — an impl keying by absolute path rebuilds to
its own root perfectly and passes. **Equality proves the routing algorithm; only the printed classification
surfaces a wrong key form.** Inferring key correctness from a green `v8` is exactly what produced core-go's
withdrawn 2026-07-31 closure (their `published_root` category read 7·0·0·0 while v2/v5 verified only a
*signature* over the root hash and v7 checked only an entity *type* — no vector in any category walked a path
from a published root). §6 turns that constraint into a conformance requirement.

**This is not three impls disagreeing about one thing.** Reconstructed against §3.4.1's own prefix table, the
three are publishing **three different subtrees**, each with a prefix that section already blesses:

| Impl | Implied config `prefix` | §3.4.1 row | Keys are relative to |
|---|---|---|---|
| Go | `system/` | peer-relative subtree (`project/`) | `/{peer_id}/system/` |
| rust | `/{peer_id}/` | peer-qualified (`/alice/data/`) | `/{peer_id}/` |
| python | `/` | universal tree | nothing — absolute |

The binding-count spread (401 / 415 / 490) is that difference, not a key bug: Go's `system/` prefix genuinely
excludes `local/files`, so its **served closure differs in extent** from rust's. All three are conformant *for
their own prefix*. **The defect is that a consumer cannot learn which prefix it is holding** — so it can neither
reconstruct a single absolute path nor know what the closure was supposed to contain.

## §2. The provenance defect underneath it (new — not in the filed issue)

`system/peer/published-root` **is not defined anywhere in this repo's spec surface.** It is *referenced* by
landed normative text three times — `EXTENSION-NETWORK` §6.5.3 (`signed_pointer: "system/peer/published-root"`),
§6.5.3's `MANIFEST_GET` body MUST, and §6.5.6 Amendment 10's `published-root.root_hash` — and each defers to
`PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE`, which §6.5.6 marks **"(planned)."** `entity-core-protocol` does not
mention it either.

It is not planned. It exists, it is **`NORMATIVE-LOCKED cgid-10-219`**, and it lives at
the internal legacy corpus (read-only) — a
**pre-split legacy archive**. Its §4 defines exactly the shape all three impls built:

```
type: "system/peer/published-root"
data: {
  peer_id:      <Base58 peer-id per V7 §1.5>
  root_hash:    <system/hash, BARE>
  seq:          <int>                  ; monotonic freshness
  published_at: <ms-since-epoch>
  predecessor?: <system/hash, BARE>    ; prior published-root content_hash | absent
}
```

**So the cohort is not improvising** — the impls built to a locked definition. What failed is that the definition
never crossed the split into the corpus that now carries its consumers, so **the field list has had no arch review
in this repo, and the missing `prefix` was never examined against §3.3's trim rule.** Landing it here is the
first delta because the other two are edits to it.

*This is the second dangling-proposal-pointer instance* — `EXTENSION-SIGNALING` §7.3 pointed at
`PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` the same way, and the cohort hit it as a blocker without being told the
document existed. **Recurring shape: a landed MUST citing a document that is not in the corpus.** A corpus-wide
sweep for cited-but-absent documents belongs in the ratchet (§9 Q4).

## §3. Q1 — what prefix are a published root's keys relative to?

**Proposed: the entity carries it.** Add a required `prefix` field to `system/peer/published-root`:

```
  prefix: <system/tree/path>   ; the absolute prefix these trie keys are relative to.
                               ; MUST end with "/". "/" designates the universal tree.
                               ; Full path reconstruction: prefix + relative_key.
```

**Why a field rather than pinning one normative prefix.** §3.4.1a already admits three prefix shapes and
§3.4.1's table blesses all three; pinning one would make two of the three impls non-conformant for a choice the
spec invited, and would forbid publishing a scoped subtree — which is the *point* of `serve_scope`'s
`published-set`. The field is also the self-describing shape the manifest already uses for `hints`, and
`published-root` is re-signed independently of the manifest (legacy §6), so the cost is one field on an entity
that is already re-minted per publish.

**This resolves the §3.3 hole at its root.** §3.3 says the prefix *"is an operational parameter to `snapshot`,
`extract`, and `merge` — not stored in the snapshot entity. Full paths are reconstructed by the consumer:
`prefix + relative_key`."* That is sound for the three named operations, where the prefix rides the request. **A
published root has no request** — it is fetched by a consumer who was not present at publish time — so the
consumer holds `relative_key` and not the operand. The rule was written for the ops and inherited by an entity
that cannot satisfy its precondition.

**No implementation changes its key convention.** Each declares the prefix it already uses.

## §4. Q2 — the universal case, and the malformed trim

§3.3's formula is:

```
relative_key = trim_prefix(path, "/" + peer_id + "/" + operation_prefix)
```

**Two defects, both confirmed:**

1. **It is ill-defined for `prefix = "/"`** — it yields `"/" + peer_id + "/" + "/"`, i.e. `/{peer}//`. rust
   reports hitting exactly this and silently tracking nothing. §3.4.1a's *"the universal case collapses to the
   empty string"* governs the **storage path** in §3.4.1's substitution table, not this formula. Verified against
   both sections.
2. **`peer_id` is not a single value for the universal tree.** The universal tree spans every peer's namespace
   (`ENTITY-CORE-PROTOCOL` §1.4 — a peer's root is the set of peer-ids it holds). Trimming the local peer's id
   leaves every other peer's keys fully qualified: a mixed key space, which is worse than either uniform answer.

**Proposed ruling.** Re-express the trim against the **absolute prefix**, not a peer-id-plus-operation-prefix
concatenation:

```
relative_key = trim_prefix(path, absolute_prefix)
```

where `absolute_prefix` is the entity's `prefix` field resolved to absolute form, and the three cases are stated:

| `prefix` | `absolute_prefix` | `relative_key` |
|---|---|---|
| `"system/"` (peer-relative) | `/{peer_id}/system/` | `attestation` |
| `"/{peer_id}/"` (peer-qualified) | `/{peer_id}/` | `system/attestation` |
| `"/"` (universal) | *(empty — trim is a no-op)* | `/{peer_id}/system/signature/…` |

**For the universal tree, keys are fully qualified and the trim is a no-op** — python's reading. It is the only
form that does not produce a mixed key space, and it makes the formula total instead of yielding `/{peer}//`.

**Consequence, stated plainly: no implementation is defective.** Go is correct at `prefix: "system/"`, rust at
`prefix: "/{peer_id}/"`, python at `prefix: "/"`. `entity-core-go` was right to decline filing python's absolute
keys as a defect, and right that no test could adjudicate it.

**§3.1's conformance fixture #2 keeps its teeth.** *"The SHA-256 input is the relative_key, not the absolute
path"* stays true — under `prefix: "/"` the relative_key **is** the absolute path, because the trim is a no-op,
not because the rule was skipped. The fixture (single binding at `relative_key = ""`) is unaffected.

## §5. The third gap — nothing requires a publisher to ever republish

**Not in the filed issue, and it voids a MUST arch folded yesterday.**

On 2026-08-07 arch ruled `EXTENSION-NETWORK` §6.5.6 A10 closure timing **(a) recompute-on-root-change (MUST)**,
with the precision that *"the trigger is the root republishing, not the bare `tree:put`."*

**All three implementations republish today `[measured live 2026-08-08 — corrected]`:**

| Peer | Observed | Landed in |
|---|---|---|
| Go | republishes within seconds of a bind | — |
| **rust** `f561493` | **republishes** — `seq 35→37`, `44→46` across runs | `8879e06` *"republish the signed root on every tree-root change"* |
| **python** `8174d11`+wt | **republishes** — `seq 14→18`, `32→36` | `c896e66` *"…and the republish underneath"* |

`seq` counters in the 30s–40s on a fresh peer: continuous republication, not a single mint.

> **Retraction — this section's first draft asserted the opposite, and it was arch's error.** It carried
> `entity-core-go`'s `2026-08-07-f` measurement (rust/py mint once, `seq: 0`, no `predecessor`) forward as
> *current* state. That report was accurate on 08-07 and was **superseded within a day**: rust `8879e06` is an
> **ancestor of `f561493`**, and python `c896e66` an ancestor of `8174d11` — i.e. **both siblings already had
> republish at the very commits the 08-08 closeout measured.** One `git log --since` per sibling repo — the check
> `AGENTS-STANDARD.md` already requires before claiming an absence — would have surfaced a commit literally
> titled *"republish the signed root on every tree-root change."* Corrected by `entity-core-go`
> (`2026-08-08-b-review-…`); durable fix recorded in `AGENTS.md` §*Verify build state before you assert it*.

**D5 still lands, and the reason is what matters — it was never "the siblings don't republish."** The gap is that
**nothing requires anyone to.** A publisher that mints once and freezes is conformant today, so the 08-07
closure-timing MUST stays voidable by a route the ruling did not close. That argument is untouched by what the
cohort voluntarily does: a spec hole no current implementation exploits is still a spec hole. **Three impls
already doing the right thing is the cheapest possible moment to pin it — nobody owes an implementation change
at fold.**

**Nothing in any landed spec, or in the locked legacy §4, requires a republish.** The legacy proposal says only
that the root *can* update without re-signing the manifest, and that `seq` is monotonic. So:

> **A publisher that mints `published-root` once at startup and never again is conformant today — and its served
> closure is frozen forever.** That is functionally the evaluate-once reading arch *rejected*, reached by a route
> the ruling did not close. The MUST governs what the closure tracks; nothing governs whether the thing it
> tracks ever moves.

**Proposed: a bounded-convergence MUST**, in `EXTENSION-NETWORK` §6.5.6 alongside the timing rule:

- A publisher advertising a `signed_pointer` **MUST** republish its signed root after a change to the tracked
  prefix, such that the published root **converges** to the tracked root within a bounded interval.
- **Coalescing/debouncing is explicitly permitted** — a signature per `tree:put` is write-amplifying and is not
  the intent. The requirement is convergence, not immediacy.
- `seq` **MUST** increase monotonically across republishes, and `predecessor` **MUST** carry the prior
  `published-root` content hash once one exists (the chain is the freshness/rollback defense; a peer stuck at
  `seq: 0` with no `predecessor` is the observable signature of a non-republishing publisher).
- **Not offered ⇒ not owed:** a publisher that does not advertise `signed_pointer` has no republish obligation
  (consistent with A10's existing carve-out for the path-bound content-only mirror).

**Route this as new spec, not as a sibling bug.** rust and python are not violating anything today. The
`--publish-root` help text asserting *"on every tree-root change · honored by all three impls"* was ahead of the
spec — the same stale-capability-note shape core-go corrected twice this cycle.

**Open for review (§9 Q2):** whether the bound is a pinned value or publisher-advertised.

## §6. What the conformance surface must assert (from the `v8` caveat)

The fix is not validated by a green `v8`, by construction. Landing this needs, in the cohort's oracle:

1. **Key-form assertion, not just equality** — a vector that takes an absolute path known to be in the published
   subtree, derives `relative_key` from the entity's declared `prefix`, and asserts the trie resolves **that**
   key. This is the check that no impl currently has and that would have caught the divergence years earlier
   than a printed classification line.
2. **Reconstruction round-trip** — `prefix + relative_key` MUST reproduce the absolute path the publisher bound.
3. **Republish convergence** — `entity-core-go`'s `seed_republished` is the right vector and **already passes
   three ways on genuine `seq` advances**; nothing in the oracle is owed for it. *(The first draft said it
   "currently SKIPs on rust/py" — same stale-evidence error as §5, corrected by `entity-core-go`.)*
   **One disclosure that belongs here:** that vector carried a false-PASS branch — a failed baseline manifest
   fetch was the first disjunct of the pass condition, so the first successful fetch returned PASS asserting a
   `seq` advance it had never observed, and a mint-once-and-freeze peer would have passed on a transiently
   unavailable manifest. `entity-core-go` found it **by auditing the gate instead of trusting its green**, once
   §6.3 proposed to make it the gate for a MUST, and fixed it (failed baseline is adopted as reference; SKIP as
   UNEXERCISED otherwise). Honestly scoped by them as **latent, not active** — observed advances are `44→46`,
   not `0→46`, proving the branch was never taken — fixed by inspection, non-regression verified live, and
   explicitly **not claimed as a caught bug** since they have no harness that can fail a baseline fetch then
   succeed. It is recorded because **a MUST whose gate can pass without observing the thing it gates is worth
   less than no gate**, which is a general rule this proposal should not have needed telling.

Per the load-bearing meta-rule: **a claim about what a path resolves to is not validated until a cross-impl
conformance test exercises it.** Prose review did not catch this one and will not catch its fix.

## §7. Spec deltas (executable form — the edits this proposal authorizes at fold)

| # | File | Section | Edit |
|---|---|---|---|
| D1 | `EXTENSION-TREE.md` | new §3.3a | Land `system/peer/published-root` normatively: the legacy §4 field list **plus** required `prefix`. Cite the legacy proposal as provenance and mark it superseded-in-place-of. |
| D2 | `EXTENSION-TREE.md` | §3.3 | Re-express the trim against `absolute_prefix`; add the three-case table (§4); state the universal-case no-op; delete the `"/" + peer_id + "/" + operation_prefix` concatenation. |
| D3 | `EXTENSION-TREE.md` | §3.1 | One clarifying sentence: under `prefix: "/"` the relative_key **is** the absolute path (trim is a no-op) — fixture #2 unaffected. |
| D4 | `EXTENSION-NETWORK.md` | §6.5.3 | Replace the `(planned)` pointer to `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` with the §3.3a citation. Same at the `MANIFEST_GET` body MUST. |
| D5 | `EXTENSION-NETWORK.md` | §6.5.6 | Add the **bounded-convergence republish MUST** (§5) adjacent to the 2026-08-07 evaluation-timing MUST, with the not-advertised carve-out. |
| D6 | `EXTENSION-NETWORK.md` | §6.5.6 A10 | One sentence: the closure's **extent is prefix-scoped** — what a signed-root closure must cover is determined by the published root's `prefix`, which is why two conformant publishers legitimately serve different sets. |
| D7 | `EXTENSION-TREE.md` | §12 conformance | Add the three assertions in §6. |

## §8. Cohort impact

| Impl | Owed at fold |
|---|---|
| **Go** | Emit `prefix: "system/"`. Republish already conformant. |
| **rust** | Emit `prefix: "/{peer_id}/"`. **Republish already conformant** (`8879e06`). Fix the `/{peer}//` universal-trim path they reported. |
| **python** | Emit `prefix: "/"`. **Republish already conformant** (`c896e66`, with cascade-coalescing already built — the shape D5 permits). |
| **oracle** | §6.1 key-form + §6.2 round-trip only. **§6.3 needs nothing** — `seed_republished` already passes three ways. |

**D5 costs no implementation anything.** All three already republish; the MUST pins a hole none of them exploits,
which is why now is the moment to land it.

**No impl re-keys its trie.** That is the property the `prefix`-field shape was chosen for.

## §9. Open for review

**All four answered by `entity-core-go` as lead implementation (`2026-08-08-b-review-…`); arch adopts all four.**

- **Q1 — RESOLVED: `prefix` is REQUIRED.** Their argument is better than the original one and replaces it: any
  default **has to be one of §4's three shapes**, which silently promotes one impl's convention to "the answer
  you get for saying nothing" — re-creating the asymmetry this proposal closes, in a form that is *harder to
  see*. No installed base, and the entity is re-minted per publish, so there is no migration cost.
- **Q2 — RESOLVED against arch's lean: pin a normative MAXIMUM CONVERGENCE DELAY; publishers MAY advertise only
  a tighter value.** core-go pushed back and is right. A purely advertised bound is **not cross-impl testable**:
  the conformance vector needs a deadline to wait against, and a peer advertising 24 h would be conformant and
  **unexercisable in any CI run** — the MUST becomes unfalsifiable at exactly the peer boundary the standard says
  to lean MUST at. It also forces a consumer to fetch publisher-specific config before it can know whether what
  it holds is stale, which is the **same out-of-band-parameter shape §3 removes from the prefix**. The
  CDN-origin-vs-laptop concern is real and the asymmetric form serves it: a laptop meets the ceiling, a CDN
  advertises better, consumers may rely on the tighter value. One number is cross-impl observable; the other is
  an optimization. **Naming pinned at their request: "maximum convergence delay"** — not "floor" or "bound".
  rust spent part of this cycle on a genuine floor/ceiling collision where two readings shared one key, so the
  vocabulary is load-bearing. *(Open sub-item for fold: the value itself.)*
- **Q3 — RESOLVED: on the root, not the manifest.** Their added reason is the decisive one: **the root is the
  signed artifact and `prefix` is what makes its contents interpretable.** Putting the key convention in a
  separately-signed entity means a consumer that has verified the root's signature still cannot read it without
  verifying a second chain — and **the two can disagree, with no rule for which wins.** Keep the interpretation
  of a signed artifact inside the signed artifact.
- **Q4 — ENDORSED, with their scoping correction.** The sweep is not only for *absent* documents but for
  **present-in-a-stale-repo** ones: the pre-split monorepo is the pre-split tree and returns
  plausible-looking outdated hits (core-go's own `AGENTS.md` carries a standing warning about it). **A
  resolvability check that passes because the document exists *somewhere on disk* would have reported §2 as
  fine** — the check must resolve within the current corpus, and flag legacy-tree hits as failures.
