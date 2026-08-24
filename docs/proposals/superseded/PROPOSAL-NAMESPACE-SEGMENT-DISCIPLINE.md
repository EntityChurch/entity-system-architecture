# PROPOSAL — ~~top-level namespace discipline: one segment per owner~~ **[SUPERSEDED — rule folded]**

> ## → The rule is folded as `SPECIFICATION-FORMAT.md` §8.4.4. The renames moved to `PROPOSAL-NAMESPACE-CLEANUP-AND-BROWSER-LEG.md` (2026-08-02).
>
> Nothing here is withdrawn. The rule this proposed is **normative as of 2026-08-02**; its defect list is that
> proposal's §3.1 (the `system/nat` flag day) and §3.2 (durability, encryption). Merged with
> `PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION` because both describe one defect — a namespace naming something
> other than its owner — and the peers should receive one rename list, not two.
>
> Retained for its reasoning: the 51-segment survey establishing that one-segment-per-extension is the existing
> convention, and **§2's argument that the §8.4.1 tiering test must NOT extend to namespaces** — the part most
> likely to be re-proposed.

**Status:** **SUPERSEDED (2026-08-02)** — rule folded to `SPECIFICATION-FORMAT` §8.4.4 (one top-level segment per owner). Retained as a record. The convention question is §3; the defect list is §4.
**Target:** `specs/SPECIFICATION-FORMAT.md` §8.4.4 (new, the rule) + renames across the extension corpus.
**Origin:** operator, 2026-08-02 — *"why would these be top-level namespaces in system? They aren't top-level
system extensions, they're implementation details."*
**Method:** every `type := {` declaration in both repos' `specs/`, grouped by first path segment. 51 distinct
top-level segments across 299 types.

---

## 1. The finding, stated against the data

**There is already a convention, and it is nearly universal: one top-level `system/<name>` segment per extension,
named for the extension that owns it.** Sixteen extensions follow it exactly:

`system/revision` · `system/group` · `system/transaction` · `system/role` · `system/clock` ·
`system/continuation` · `system/history` · `system/query` · `system/network` · `system/subscription` ·
`system/content` · `system/compute` · `system/attestation` · `system/quorum` · `system/encryption` ·
`system/signaling`

**So `system/signaling` is not the anomaly** — by the corpus's own convention it is exactly as top-level as
`system/query` or `system/clock`, and singling it out would be inconsistent rather than principled.

**The anomalies are the *extra* segments** — cases where one owner holds several top-level names, or where a
hyphen is doing the job of a path separator. **That is the real defect, it is what makes the namespace look
arbitrary, and `system/nat` is one instance of it.**

## 2. Why the §8.4.1 tiering test does *not* extend to namespaces

The obvious move — *"§8.4.1 says signaling-class extensions are landscape-specific, so they shouldn't get a
top-level segment either"* — **is wrong, and the reason is worth stating so it is not re-proposed.**

**§8.4.1 exists because a field on a core type is carried, preserved and hashed by every peer forever, including
peers that never implement the extension.** The cost is unbounded and paid by everyone; that asymmetry is the
entire justification for tiering.

**A namespace segment has no such cost.** It is a string prefix on types that only peers implementing the
extension ever hold. A peer that does not implement SIGNALING carries no `system/signaling/*` entity and pays
nothing for the segment's existence. **The cost asymmetry that justifies tiering fields does not exist here**, so
importing the test would be cargo-culting a rule past its rationale.

**What a namespace is actually for:** disambiguation and ownership — *given a type, which spec defines it?* A flat
one-segment-per-owner scheme answers that in one lookup. **It is not a status ranking**, and it should not be
made into one: nesting `system/network/signaling/*` would say something about importance while making ownership
*harder* to read, and it would put SIGNALING's types inside NETWORK's namespace — which §8.4.2 prohibits
(a spec does not define types in a namespace another spec owns).

## 3. The convention, proposed as normative

> **One top-level `system/<segment>` per owning specification, named for the owner.** Everything that
> specification defines nests beneath it. A specification MUST NOT hold two top-level segments, and MUST NOT use
> a hyphenated top-level name (`system/foo-request`) where a nested path (`system/foo/request`) expresses the
> same thing.

**Corollary — hyphens separate words, `/` separates scopes.** `system/durability-request` and
`system/durability-result` are not two concepts; they are one concept's request and result, spelled with the
wrong separator, and they occupy two top-level segments as a result.

**Deliberate exceptions, which stay:** `primitive/*` (core scalar primitives, not owned by any extension) and
`compute/*` (the **expression language** node types — `literal`, `lookup`, `apply`, `if`, `lambda` — as distinct
from `system/compute/*`, the handler's operational surface). **That split is principled and should be documented
rather than 'fixed':** the language is a data format an expression author writes, the handler surface is an
operation set a peer invokes. Both belong to EXTENSION-COMPUTE and the two-namespace shape is intentional.

## 4. The defect list

### 4.1 Owners holding more than one top-level segment

| Owner | Segments held | Should be |
|---|---|---|
| **EXTENSION-SIGNALING** | `system/signaling` + **`system/nat`** (3 types: `connect-request`, `connect-response`, `punch-sync`, plus the `rendezvous-key` type string) | all under `system/signaling/*` |
| **EXTENSION-ENCRYPTION** | `system/encryption` + `system/encrypted` + `system/encryption-pubkey` (+ shares `system/signature`, `system/attestation`, `system/identity`) | `system/encryption/*` for the ones it owns outright |
| **EXTENSION-DURABILITY** | `system/durability-request` + `system/durability-result` — **two top-level segments, no `system/durability` at all** | `system/durability/{request,result}` |

**`system/nat` is the worst of the three, and it is the one in the way right now.** It names a **problem domain**
(NAT traversal), not an owner — the same category error as `system/protocol/inbox/*` naming a *shape* rather than
an owner. It is also about to become a three-way problem: the WebRTC proposal introduces `system/webrtc/*`, which
would give one extension **three** top-level segments for one carrier's coordination messages.

### 4.2 Hyphenated top-level names that should nest

`system/durability-request` · `system/durability-result` · `system/encryption-pubkey` · `system/peer-id`
(alongside an existing `system/peer/*`) · `system/delivery-spec` · `system/resource-limits`

### 4.3 Bare non-`system/` roots that look accidental

`get-request` (CONTENT) · `governance` (GROUP) · `binding`, `budget` (COMPUTE) · `envelope` (core, alongside
`system/envelope` **and** `system/protocol/envelope`)

**Each needs a one-line check:** deliberate bare type, or a declaration that lost its prefix? The `envelope` /
`system/envelope` / `system/protocol/envelope` trio is the one to look at first — three names, and at most one
should exist.

## 5. Cost — and one genuine flag day

**Most of §4 is cheap:** these types are extension-owned, none is in the V7 §9.5 Core Type Floor, and the
rename is a string constant plus references per implementation.

**One is not cheap, and it must be understood before anyone starts.** The rendezvous key is derived by hashing
the literal type string:

```
rendezvous_key = varint(0x00) ‖ SHA-256( ecf_for_hash( "system/nat/rendezvous-key", cbor_bstr(payload) ) )
```

**Renaming it changes every derived key.** Two peers on different versions compute different keys and can
**never meet** — a silent failure, not an error. This is a **hard flag day**: all implementations and any
deployed lobby must change together. Our no-backward-compatibility policy makes that acceptable, but it must be
deliberate, simultaneous, and it invalidates in-flight rendezvous state.

**It is strictly cheapest now.** The punch is cross-impl green but not deployed; WebRTC has not shipped a third
namespace; no keystone peer implements any of it. **Every week this waits, the flag day gets more expensive.**

## 6. Recommendation

1. **Fold the rule** as `SPECIFICATION-FORMAT.md` §8.4.4 (§3 above). It is the convention the corpus already
   follows; writing it down stops the next extension inventing a second segment.
2. **Do `system/nat/*` → `system/signaling/*` now, as one flag day, before WebRTC folds.** This is the item
   blocking the connectivity track, and folding WebRTC first would triple it.
3. **WebRTC lands as `system/signaling/webrtc/*`**, not `system/webrtc/*` — one owner, one segment,
   per-substrate nesting beneath it. This closes the proposal's own open item #2.
4. **Sweep §4.2 and §4.3 opportunistically** — same class, no flag day, no urgency. Bundle with the next touch
   of each spec rather than as a campaign.
5. **Document the `compute/*` vs `system/compute/*` split** as intentional so it stops reading as a defect.

**Sequencing with the other open proposals:** this and `PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION` are the same
class of fix (a namespace naming something other than its owner) and touch different specs, so they can land
independently. Both want to be in before the next release.

## 7. What this does not do

- **No behavioral change**, no wire format change, no opcode, no core floor entry. Type names only.
- **Does not tier the namespace by importance** — see §2 for why that would be a mistake.
- **Does not touch `system/protocol/*`, `system/handler/*`, `system/tree/*`, `system/capability/*`** or the other
  core-owned segments. Those are core's, correctly.
