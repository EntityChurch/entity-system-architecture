# PROPOSAL — ~~inbox namespace rename + retire `ENTITY-CORE-MACHINE-SPEC.md`~~ **[SUPERSEDED]**

> ## → Consolidated into `PROPOSAL-NAMESPACE-CLEANUP-AND-BROWSER-LEG.md` (2026-08-02). Execute from there.
>
> Both halves carry forward unchanged — the inbox rename is its §3.2, the machine-spec retirement its §4,
> including the §1.8-relocation-before-deletion ordering. Merged because this and
> `PROPOSAL-NAMESPACE-SEGMENT-DISCIPLINE` were the same defect class found from two directions, and shipping
> them separately would send the same peers the same rename twice.
>
> Retained for its reasoning: the four-line evidence that the inbox types are not protocol messages, the
> three-implementations-mis-layered-it finding, and the record of two withdrawn conclusions (§6).

**Status:** **SUPERSEDED (2026-08-01)** — see title. The inbox-rename half landed via the namespace-cleanup reorganization; the machine-spec-retirement half was withdrawn (built on the void `ENTITY-CORE-MACHINE-SPEC.md`). Retained as a record. Checklist at §7.
**Target repos:** `entity-system-architecture` (Part A) and `entity-core-protocol` (Part B). **The architecture
team maintains both**; neither part is a cross-org request.
**Scope:** **Part A** — two type renames. **Part B** — relocate one normative section, then delete a 1307-line
derived document.
**Not in scope:** no opcode, no framing, no wire format, no V7 §9.5 Core Type Floor entry, no behavioral change
anywhere.
**Cohort impact:** a string constant and its references per implementation (Part A); ~11 citation repoints
(Part B). **Keystone: none for Part A**, one citation repoint for Part B.
**Origin:** operator, 2026-08-01 — *"are they protocol messages or not?"* (Part A) and *"get rid of entity core
machine spec, it's more a liability than any value"* (Part B).

> **The two parts are related and should land together.** Part B's only non-§1.8 dependency is
> `entity-core-go`'s two citations of machine-spec **§3.9** — which is the section defining the very types Part A
> renames.

---

## 1. The change

| Today | Proposed | Owner |
|---|---|---|
| `system/protocol/inbox/delivery` | **`system/inbox/delivery`** | `EXTENSION-INBOX.md` §2.1 |
| `system/protocol/inbox/notification` | **`system/subscription/notification`** | `EXTENSION-SUBSCRIPTION.md` (canonical) + `EXTENSION-INBOX.md` §2.2 (reproduction, already pointer-marked) |

## 2. Why — these are not protocol messages

**Where they are actually defined.** Both types are declared **only in this repo's extension specs**:

- `system/protocol/inbox/delivery` — `EXTENSION-INBOX.md:80`, and nowhere else.
- `system/protocol/inbox/notification` — `EXTENSION-SUBSCRIPTION.md:86` (canonical) and `EXTENSION-INBOX.md:94`
  (marked reproduction).

**No core specification mentions either type** — verified across every file in `entity-core-protocol/specs/`.
So this is an **extension defining types inside a namespace whose every core member is a wire message.** That is
precisely the act `SPECIFICATION-FORMAT.md` §8.4.2 now prohibits.

**What the namespace means.** The V7 §9.5 Core Type Floor contains **seven** `system/protocol/*` types, and every
one of them is a wire message or a structural component of one:

`envelope` · `execute` · `execute/response` · `error` · `resource-target` · `connect/hello` ·
`connect/authenticate`

**Neither inbox type is in the floor.** The namespace's meaning is unambiguous from its core membership, and
these two do not have the property.

**What they actually are.** `EXTENSION-INBOX.md` §137 is explicit: the `receive` operation *"accepts any typed
entity. The entity's type carries the semantic information — `system/protocol/inbox/delivery` for async
operation results, `system/protocol/inbox/notification` for subscription events, `system/protocol/execute` for
raw deferred dispatch, **or any domain-specific message type**."*

They occupy **the same structural slot as an application's own message types.** A chat message goes in that
slot. The slot does not make a chat message a protocol type. **The wire message is the EXECUTE that carries the
payload; the payload is cargo.**

**And `notification` is a subscription event** — its fields are `subscription_id`, `event`, `uri`, `hash`,
`previous_hash`, and its canonical home is already `EXTENSION-SUBSCRIPTION.md`. It sits under an inbox-shaped
protocol prefix while being neither.

## 3. The evidence that the name is actively causing harm

**All three reference implementations put these types in their *core* layer:**

| Implementation | File |
|---|---|
| entity-core-go | `core/types/delivery.go` |
| entity-core-rust | `core/types/src/core_types.rs` |
| entity-core-py | `entity_core/protocol/delivery.py` |

**Three independent teams mis-layered an extension type into core, and the only thing telling them to was the
prefix.** This is not a hypothetical readability argument — the name has already produced the same wrong
decision three times, in three languages, in the layer where layering matters most.

**The trap that makes it convincing.** `delivery` carries `{original_request_id, status, result}` — nearly the
shape of `system/protocol/execute/response`, which genuinely *is* a wire message. **It mirrors a protocol
message without being one:** `execute/response` is framed by the transport; `delivery` is payload inside an
EXECUTE. Same shape, one layer apart, and the name asserts the resemblance while hiding the difference.

**Likely origin:** `entity-core-py` carries a `system/callback/* → system/protocol/inbox/*` rename note at v7.8 —
a rename that fixed a vague name (`callback`) and does not appear to have re-examined the placement.

## 4. Naming rationale

- **`notification` → `system/subscription/notification`.** High confidence: the fields are subscription fields,
  the producer is SUBSCRIPTION, and its canonical definition already lives there. The inbox is only its delivery
  route — naming it after its transport rather than its producer is the same category error at smaller scale.
- **`delivery` → `system/inbox/delivery`.** INBOX defines it and owns the concept.
  *(A conflict with the mailbox tree paths `system/inbox/network` / `system/inbox/local` was considered and
  **dismissed**: `system/tree` is simultaneously a handler path (`"system/tree"`) and a type prefix
  (`system/tree/path`, `system/tree/snapshot`), so the corpus shares these spaces throughout.)*

## 5. Cost, and the case against

**Cost:** a string constant plus its references in each of the three implementations, and their inbox /
subscription call sites and tests. **Keystone: zero** — both types report `absent / not-a-FAIL-if-absent` across
all 22 generated peers, so nothing generated has adopted them. **No floor entry, no wire change.**
*(Arch's cost estimates have been corrected five times this arc against zero ownership rulings overturned —
**the cohort sizes this, not arch.**)*

**The honest case against:** the rename buys **no runtime behavior.** It buys a namespace whose meaning holds.
The counterweight is §3 — the name has already caused three implementations to file an extension type under
core, and it will keep doing that.

**If the churn is judged not worth it:** state the exception in `EXTENSION-INBOX.md` explicitly — *"these are not
wire messages despite the prefix; the placement is historical"* — so the next reader is not misled. **Silence is
the one clearly wrong option**, because the prefix keeps making its claim either way. **Do this rather than
nothing.**

**Timing:** cheaper now than at any later point — keystone has adopted neither type and Python has not built the
call sites that would multiply the references. The population that must migrate only grows.

---

## Part B — retire `ENTITY-CORE-MACHINE-SPEC.md`

**Decision (operator, 2026-08-01): retire it.** *"More a liability than any value; it's never really been used
in practice and it is causing us to get confused, so I don't see the argument to keep it. The idea is right — we
may want to refine the spec more — but this is not the way to do it."*

**We own `entity-core-protocol`.** This is an ordinary edit in a repo the architecture team maintains, not a
cross-org request.

### B.1 The case, in numbers

- **1307 lines.** Its §3 Type Registry alone re-declares **38 types owned by other specs**.
- **It has already cost a full audit cycle in false findings.** Diffing specs against it produced "13 drifted
  types, three interoperation-breaking" — every one an artifact of a stale copy. A derived document is
  indistinguishable from a source at a glance, which is exactly what makes it a liability rather than merely
  redundant.
- **`EXTENSION-COMPUTE.md` §2.4 already calls it out** — *"a derived condensed summary and is downstream;
  regenerate-or-retire is the protocol maintainer's hygiene"* — so the ecosystem has known and worked around
  this rather than fixing it.

### B.2 What actually depends on it — the whole dependency surface

An ecosystem-wide scan for citations of specific sections (all repos, `.md`/`.go`/`.rs`/`.py`/`.toml`) returns
**exactly two sections**:

| Section | Citations | Where |
|---|---|---|
| **§1.8 Entity Fidelity** | **9** | `entity-core-go` (2 validator declarations), `entity-core-rust` (`core/protocol/src/verify.rs`), `entity-core-keystone`, `entity-core-formalization`, `entity-core-protocol` — plus this repo's `AGENTS.md` ("prefer byte preservation (§1.8)") and `GUIDE-CONFORMANCE.md` |
| **§3.9 Inbox & Subscription Types** | **2** | `entity-core-go` |

**Every other line of the document is cited by nobody.** That is the argument for retirement stated precisely —
not "unused," but *"two sections carry the entire load and one of them is original normative content filed in a
derived document."*

### B.3 The one thing a naive delete would break

**§1.8 "Entity Fidelity" is not derived.** It is original normative content — the receive/forward byte-preservation
contract, the property-vs-mechanism clarification, and the conditions under which an implementation MAY re-encode
canonically instead of storing raw bytes. **It is one of this repo's five declared load-bearing invariants**
(`AGENTS.md`: *"re-encoding on receive/forward is THE interop hazard — prefer byte preservation (§1.8); a
re-encoder MUST produce bit-identical canonical ECF"*), and **two implementations' conformance validators declare
against it by name.**

**Deleting the file without relocating §1.8 would orphan a normative contract that live conformance code cites.**

### B.4 Plan

1. **Relocate §1.8 to `ENTITY-CBOR-ENCODING.md`** — the natural home: §1.8's own conformance already routes there
   (*"verified by the test-vector appendix in `ENTITY-CBOR-ENCODING.md` Appendix E"*), and the content is an
   encoding/byte-fidelity contract. Move it **verbatim**; this is a relocation, not a revision.
2. **Repoint the 9 §1.8 citations** — go (2), rust (1), keystone, formalization, protocol, plus this repo's
   `AGENTS.md` and `GUIDE-CONFORMANCE.md`. Mechanical.
3. **Repoint go's 2 §3.9 citations** to `EXTENSION-INBOX.md` / `EXTENSION-SUBSCRIPTION.md` — **Part A renames
   those types anyway**, so the two changes should land together.
4. **Delete `ENTITY-CORE-MACHINE-SPEC.md`.**
5. **Drop its entry from `entity-system-arch-tools/spec-tool/config.default.toml`** (currently classified
   `"guide"`).
6. **Sweep the remaining prose references** — 15 files in this repo, 6 in `entity-core-protocol`, plus cohort
   status docs. Cohort docs are historical records and can keep their citations; **this repo's `specs/` and
   `guides/` must not cite a deleted document.**

**Sequencing:** step 1 before step 4, or the invariant is orphaned. Steps 3 and Part A land together.

### B.5 What is explicitly *not* being claimed

**The idea behind the document is sound** — a single condensed implementation-facing view has real value, and
this is not an argument against ever having one. It is an argument that **an unmaintained, unsourced, manually
copied one is worse than none**, because readers and auditors cannot tell it from a source. If a condensed view
is wanted later it should be **generated from the specs as part of a build**, so it cannot drift, and marked
derived per `SPECIFICATION-FORMAT.md` §8.4.3.

*(The one item from the withdrawn audit that was genuinely in this repo's lane is already **fixed**:
`SDK-OPERATIONS.md` restated the core-owned `system/type/field-spec` informally; it now carries a canonical
pointer to `ENTITY-CORE-PROTOCOL.md` §2.2 and is marked an abridged sketch.)*

## 6. Withdrawn: the "13 drifted types" finding

An earlier revision carried a Part B reporting 13 type definitions as drifted between
`ENTITY-CORE-MACHINE-SPEC.md` and the owning extension specs, three of them "interoperation-breaking."

**Withdrawn in full.** Every comparison was against the machine spec. A stale derived copy differing from its
source is the expected state of a stale copy — not a defect in the specs, and **no implementation was ever at
risk.** Retained as `ANALYSIS-CORE-CATALOGUE-DUPLICATION-AND-DRIFT.md` (marked VOID) because the methodological
failure is worth keeping: **an audit is only as good as its premise about which documents are authoritative, and
that premise was never checked.** One grep asking *"who sources from this?"* would have cost nothing and saved
the cycle. That premise-check is now normative — `SPECIFICATION-FORMAT.md` §8.4.3.

## 7. Execution checklist

**Not urgent; blocks no current build. Wanted before the next release.** Ordered — B1 must precede B4.

**Arch (this repo + `entity-core-protocol`), unblocked:**

- [ ] **A1.** Confirm the two target names (§1). `delivery`'s is the less settled of the pair.
- [ ] **A2.** Rename in `EXTENSION-INBOX.md` §2.1/§2.2, `EXTENSION-SUBSCRIPTION.md`, and every corpus
      cross-reference.
- [ ] **B1.** Move machine-spec **§1.8 Entity Fidelity verbatim** into `ENTITY-CBOR-ENCODING.md`. **Do this
      first** — it is original normative content and one of the five declared load-bearing invariants.
- [ ] **B2.** Repoint this repo's `AGENTS.md` and `GUIDE-CONFORMANCE.md` §1.8 citations.
- [ ] **B3.** Repoint `entity-core-protocol`'s own 6 references.
- [ ] **B4.** Delete `ENTITY-CORE-MACHINE-SPEC.md`.
- [ ] **B5.** Drop its entry from `entity-system-arch-tools/spec-tool/config.default.toml`.
- [ ] **B6.** Verify no `specs/` or `guides/` file in any repo cites the deleted document.

**Cohort, after the above lands:**

- [ ] **C1.** Rename the constant and its references in go / rust / py. **Reconsider whether the type belongs in
      your core layer once the prefix no longer says `protocol`** — §3 suggests it does not.
- [ ] **C2.** Repoint §1.8 citations: `entity-core-go` (2 validator declarations), `entity-core-rust`
      (`core/protocol/src/verify.rs`), `entity-core-keystone`, `entity-core-formalization`.
- [ ] **C3.** `entity-core-go` only: repoint the 2 machine-spec **§3.9** citations to the renamed types.
- [ ] **C4.** **Keystone: expected zero for Part A** (both types absent in all 22 generated peers). Verify
      rather than trust; adoption would convert a spec edit into a migration.

*(Sizing is the cohort's. Arch's cost estimates have been corrected five times this arc against zero ownership
rulings overturned.)*

## 8. References

- **Rules this applies:** `specs/SPECIFICATION-FORMAT.md` **§8.4.2** (namespace placement + ownership) and
  **§8.4.3** (derived documents are never authoritative) — both folded 2026-08-01.
- **Analysis:** `docs/research/explorations/ANALYSIS-CORE-NAMESPACE-CLAIMS-IMPACT.md`.
- **The withdrawn audit, kept as a methodological record:**
  `docs/research/explorations/ANALYSIS-CORE-CATALOGUE-DUPLICATION-AND-DRIFT.md` (VOID).
