# PROPOSAL — the extension host install seam: naming the third installer, and making a generated peer a thing you can build an extension into

**Status:** IMPLEMENTED (ratified and folded 2026-09-08; drafted 2026-09-01)

> ## Fold record — what landed, and the two things the fold changed about this document
>
> **Landed as:** `ENTITY-CORE-PROTOCOL` **0.8.2.12** (D1, D2, D3) · `SDK-OPERATIONS` **v1.12** (D4, D5,
> D6) · `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 (D7) · `GUIDE-CONFORMANCE` §7.0 + new **§7d** (D8, D9,
> D12) · `SPECIFICATION-FORMAT` **v1.2** §4.1 (D10). D11 was already withdrawn (§2.5a).
>
> **1. D3's version target was stale, and the fold corrected it.** This document targets
> `0.8.2.3 → 0.8.2.4`. The core specification was at **0.8.2.11** when the fold ran — eight
> fourth-component revisions had landed in the intervening week — so the fold went to **0.8.2.12**. The
> §6.2 and §9.1 target text was unchanged and was re-read before editing; only its position in the file
> had moved. **A version number written into a proposal is a claim about build state and expires like
> one.**
>
> **2. The homes list in §7 was incomplete, and the miss was in the dangerous direction.** §7 enumerated
> the documents that restate this rule and cleared `EXTENSION-COMPUTE`'s builtin override prohibition as
> *"no edit needed — scoped to bootstrap-registered builtins, stays true under D1."* **That judgment was
> wrong.** The compute prohibition described itself as *"a subset of"* the core reservation and told
> implementers that enforcing the core rule needed *"no separate compute-specific guard."* Once the core
> rule covers only the **dispatch** path, that subset claim inverts: the compute prohibition stops being
> redundant and becomes the **only** thing standing between an in-process extension installer and a body
> bound at `system/compute/builtins/arithmetic` — a cross-peer determinism break. **D1 would have opened
> a hole by narrowing a rule another specification was leaning on.** Folded as `EXTENSION-COMPUTE`
> **v3.28**: the prohibition now binds both installation paths and states that §6.2 alone does not
> satisfy it.
>
> A second unlisted home: `EXTENSION-TREE` §8.7 carried the retired *"user-installed handlers"* wording
> as an unmarked restatement. Folded as **v4.7**.
>
> **Both were found by searching the corpus for the rule's *subject* rather than for its wording**, which
> is what §7's own list did not do exhaustively. **The transferable form: when a fold narrows a rule, the
> sweep is not for restatements of the rule — it is for documents that were relying on the part being
> removed.** Those do not share the rule's vocabulary. `EXTENSION-COMPUTE` was found through
> `forbidden_pattern`, not through `system/*`, and its dependence was stated in a parenthetical.
>
> **Not folded, and carried forward deliberately:** §9's four open items, unchanged. Item 4 — TREE and
> TYPE as core-profile deltas, plus the two bootstrap type IDs — remains owed a proposal of its own and
> is the next piece of authoring work on this track.
>
> **Still owed on the build side:** §5.1 items 4–7 (the generation phase contract, the profile's host
> declaration block, the host-seam harness, and one regeneration sweep) and item 8 (the §7d transport in
> the conformance oracle, with both controls). Delivered to the seats that own them rather than recorded
> here.
| `entity-core-{rust,py}` | none from this proposal. D1 changes no behaviour. |
| `entity-browser-rust` · `entity-workbench-go` | none normative. D4's *"SDK MUST NOT apply §6.2 inside the primitive"* is worth checking against their own `register_handler` — if either applies the guard, they cannot host a standard extension either. |
| `entity-system-generator` | the whole of §3 and §6 is its charter. |

**Routing:** consolidated end-of-day packet only. `ROUTING-2026-09-01-a` (keystone) and `-b` (core-go)
already went out today; a second same-day packet to either seat is **L13's fifth axis**.

---

## §9 Open items

1. **`Handle` across a `declined` profile.** A peer that declines runtime registration still needs a
   *build-time* composition path if it is to host an extension at all (§11.3's static registration).
   Is a build-time-only extension host a supported posture, or does declining mean "core peer, no
   extensions"? **Leaning: supported** — `asm-x86_64` composing a fixed extension set at assembly
   time is a legitimate peer — but it needs its own class in §7d and it is not on the critical path.
   *Carried per L9: this item does not fold with the proposal.*
2. **§11.6.9 service-owning under the generator.** Deferred with `NETWORK`/`SIGNALING`/`REGISTRY`;
   nothing in §2 depends on it.
3. **V2 (L1 capability-checked registration).** §11.6's stated target. Out of scope; D4 is written so
   it does not foreclose it.
4. **TREE and TYPE as core-profile deltas.** Owed a separate proposal, including the two bootstrap
   type IDs `EXTENSION-TREE` §9 names.
