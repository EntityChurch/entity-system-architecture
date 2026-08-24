# PROPOSAL — namespace cleanup across the extensions, and the browser leg back on track

**Status:** **IMPLEMENTED — every spec edit this proposal owed is landed. Closed 2026-08-15.** The
one-package framing was **over-merged** and was split A+B / C / D on 2026-08-02; **all four halves are now
folded**, verified section-by-section at close rather than taken from this header:

| Workstream | Spec-side state, verified 2026-08-15 |
|---|---|
| **B — §3.1 signaling flag day** | **Folded.** `system/nat/*` → `system/signaling/*` across `EXTENSION-SIGNALING` (§6.1 types, §3.1 derivation, §12 table; §13 open-item-1 resolved) |
| **C — §3.2 general renames** | **Folded and ruled twice.** `system/inbox/delivery` + `system/subscription/notification` ratified (`EXTENSION-INBOX` §2.1, `EXTENSION-SUBSCRIPTION` §2.2, one round / two strings), `durability/{request,result}` re-homed, **ENCRYPTION deliberately reverted** (§8.4.4 exception), and the coordinated-cut `[MUST]` pinned at `SPECIFICATION-FORMAT` §8.4.4 |
| **D — S2, the WebRTC fold** | **Folded**, and all three of its settle-at-the-fold items are in the text: the `system/signaling/webrtc/*` namespace (`EXTENSION-SIGNALING` §6.5), **DTLS fingerprint pinning as a security `MUST`** (§6.5 channel-identity binding), and the reachability-class answer — **no new class; a punch-substrate class at the §10.3 seam** (`EXTENSION-NETWORK`) |
| **Cross-repo — §1.8** | **Additive half complete.** Relocated verbatim to `ENTITY-CBOR-ENCODING.md` §5.4, which now carries the canonical-home marker; the machine-spec copy is marked derived and scheduled |

**What remains is execution by other seats, and arch does not track it as owed work** (`AGENTS.md`, spec
lane — a proposal records the spec delta and the owning team, never an impl checklist):

- **S0 completion** — rust's eight strings plus a cross-impl meet. **core-rust.**
- **S1 · S3 · S4 · S5** — the wasm probe, the WebRTC transport crate, browser consumption, and the
  browser↔native gate. **core-rust · browser-rust · cohort.**
- **The `ENTITY-CORE-MACHINE-SPEC` §1.8 deletion** — blocked on the cohort repointing ~9 citations and on
  nothing else. The relocation already retired the ordering hazard permanently, so the deletion is hygiene,
  not a correctness gate.

> **Why this closes rather than staying open on the residue.** Every item above is a *build*, and this
> proposal's own §1 says the defect it exists to fix is a **naming** defect. Holding an `active/` slot open
> for four seats' execution is what made twenty-two proposals read as in-flight when nothing was owed here.
> **The routing is recorded; the tracking belongs on the board, not in a DRAFT.**

**Supersedes:** `PROPOSAL-INBOX-TYPE-NAMESPACE-CORRECTION` and `PROPOSAL-NAMESPACE-SEGMENT-DISCIPLINE`. Both
were the same defect class found from two directions; carrying them separately would send the same peers the
same rename twice.
**Absorbs:** `PLAN-2026-08-02-browser-leg-territory-and-drive` (§5 here is its operative half).
**Targets:** `entity-system-architecture` (spec renames), `entity-core-protocol` (retire one document),
`entity-core-rust` + `entity-core-go` (constant renames), `entity-browser-rust` (consume, later).
**Scope:** type-name renames, one document deletion, and a sequencing plan. **No behavioral change, no wire
format change, no opcode, no V7 §9.5 Core Type Floor entry.**
**One wire-visible consequence, in §3.1 only** — read it before scheduling anything.

---

## 1. What this is

Four separate findings over three days turned out to be **one defect with one cause**: *a namespace segment
naming something other than the specification that owns it.*

| Found as | Actually |
|---|---|
| "INBOX claims a core namespace" | `system/protocol/inbox/*` names a **shape** (protocol-message) instead of its owner |
| "why is signaling top-level" | `system/nat/*` names a **problem domain** instead of its owner |
| durability / encryption sprawl | hyphens standing in for `/`, manufacturing extra top-level segments |
| the machine spec's "13 drifted types" | a **derived document** treated as a source |

**The rules are already folded** (`SPECIFICATION-FORMAT.md`, 2026-08-01/02) — §8.4.2 ownership and placement,
§8.4.3 derived documents are never authoritative, §8.4.4 one top-level segment per owner. **This proposal is the
backlog those rules produce**, nothing more.

## 2. What is NOT a defect — checked and closed, do not re-open

Recorded because each of these looked like a finding and cost time:

- **`system/signaling` at top level is correct.** Sixteen extensions each hold one top-level segment named for
  themselves (`revision`, `group`, `transaction`, `role`, `clock`, `continuation`, `history`, `query`,
  `network`, `subscription`, `content`, `compute`, `attestation`, `quorum`, `encryption`, `signaling`).
  Signaling follows the convention; only its **second** segment does not.
- **The §8.4.1 tiering test does not extend to namespaces.** It exists because a core-type field is carried and
  hashed by every peer forever; a namespace segment costs nothing to peers that do not implement the extension.
  Reasoning recorded in §8.4.4 so it is not re-proposed.
- **`compute/*` vs `system/compute/*` is intentional** — the expression language an author writes vs the handler
  surface a peer invokes. Documented as an exception, not fixed.
- **Lowercase bare roots are licensed** by `SPECIFICATION-FORMAT.md` §2.5 (supplementary structures):
  `governance`, `binding`, `budget`, `get-request`, `grant-entry`. Not defects.
- **The `envelope` "trio" is two types and an example.** `ENTITY-CORE-PROTOCOL.md:41` is an illustrative wire
  snippet; `system/protocol/envelope` and `system/envelope` are structurally identical with a **documented
  semantic distinction**. Core-owned and deliberate.
- **`system/handler/composition`** — `system/handler` holds handler-describing types and `composition` describes
  a handler. Correct.

## 3. The renames

### 3.1 `system/nat/*` → `system/signaling/*` — **the flag day** `[do first; blocks WebRTC]`

| Today | Becomes |
|---|---|
| `system/nat/connect-request` | `system/signaling/connect-request` |
| `system/nat/connect-response` | `system/signaling/connect-response` |
| `system/nat/punch-sync` | `system/signaling/punch-sync` |
| `system/nat/rendezvous-key` *(type string, hashed)* | `system/signaling/rendezvous-key` |

> ### ⚠ This one is wire-visible. Read before scheduling.
>
> The rendezvous key is derived by hashing the **literal type string**:
> `rendezvous_key = varint(0x00) ‖ SHA-256( ecf_for_hash( "system/nat/rendezvous-key", cbor_bstr(payload) ) )`
>
> **Renaming it changes every derived key.** Two peers on different versions compute different keys and
> **silently never meet** — no error, no diagnostic, just an empty bucket. **All implementations and any
> deployed lobby must change together.** Our no-backward-compatibility policy makes a flag day acceptable; it
> does not make it free.
>
> **It is cheapest now and gets worse weekly:** the punch is cross-impl green but **not deployed**, WebRTC has
> not yet added a third segment, and no keystone peer implements any of it.

**Why it can't wait:** the WebRTC fold introduces `system/webrtc/*`, which would give one extension **three**
top-level segments for one carrier's messages. Renaming after that is three renames instead of one.

### 3.2 Same class, no flag day `[opportunistic]`

| Today | Becomes | Owner |
|---|---|---|
| `system/protocol/inbox/delivery` | **`system/inbox/delivery`** | EXTENSION-INBOX |
| `system/protocol/inbox/notification` | **`system/subscription/notification`** | EXTENSION-SUBSCRIPTION (canonical) |
| `system/durability-request` | `system/durability/request` | EXTENSION-DURABILITY |
| `system/durability-result` | `system/durability/result` | EXTENSION-DURABILITY |
| `system/encrypted` | `system/encryption/encrypted` | EXTENSION-ENCRYPTION |
| `system/encryption-pubkey` | `system/encryption/pubkey` | EXTENSION-ENCRYPTION |

**On the inbox pair specifically** — the evidence that it matters rather than merely offends: **all three
implementations file these types in their *core* layer** (`core/types/delivery.go`,
`core/types/src/core_types.rs`, `entity_core/protocol/delivery.py`). Three teams mis-layered an extension type,
and the only thing telling them to was the `protocol` prefix. They are payloads carried *inside* an EXECUTE, in
the same slot `EXTENSION-INBOX.md` §137 gives to *"any domain-specific message type"* — a chat message goes in
that slot. **Neither is in the V7 §9.5 floor; keystone has adopted neither.**

**Left alone deliberately:** `system/peer-id`, `system/delivery-spec`, `system/resource-limits` are **core-owned**
and hyphenated. Same pattern, core's call, not bundled here.

## 4. Retire `ENTITY-CORE-MACHINE-SPEC.md`

**1307 lines. Its §3 re-declares 38 types owned by other specs. It has already cost one full audit cycle in
false findings** — diffing specs against it produced "13 drifted types, three interoperation-breaking," every
one an artifact of a stale copy. **`EXTENSION-COMPUTE.md` §2.4 has called it derived and downstream the whole
time.**

**The entire dependency surface, from an ecosystem-wide scan — exactly two sections:**

| Section | Citations |
|---|---|
| **§1.8 Entity Fidelity** | **9** — go (2 validator declarations), rust (`core/protocol/src/verify.rs`), keystone, formalization, protocol, plus this repo's `AGENTS.md` and `GUIDE-CONFORMANCE.md` |
| **§3.9 Inbox & Subscription Types** | **2** — go only |

**§1.8 is not derived content.** It is the receive/forward byte-preservation contract — one of the five
load-bearing invariants `AGENTS.md` declares, and two implementations' conformance validators cite it by name.
**Deleting the file without relocating §1.8 orphans a normative contract that live code depends on.**

**Ordered:** relocate §1.8 **verbatim** into `ENTITY-CBOR-ENCODING.md` (its own conformance already routes to
that document's Appendix E) → repoint 11 citations → delete → drop the `spec-tool/config.default.toml` entry →
sweep prose references.

**Not claimed:** that a condensed implementation view is a bad idea. It is a good idea *generated from the
specs as part of a build* and marked derived per §8.4.3. An unmaintained hand-copied one is worse than none,
because nobody can tell it from a source.

## 5. Getting Rust back on track — territory and order

### 5.1 Territory, decided by a fact rather than a preference

**`entity-core-rust` already cross-compiles its peer stack to the browser** — `make wasm` builds `entity-peer`
for `wasm32-unknown-unknown` in CI, and `core/crypto`, `extensions/{encryption,history,discovery}` already carry
`[target.'cfg(target_arch = "wasm32")'.dependencies]`.

> **Protocol is core-rust's, including its wasm32 target. `entity-browser-rust` is the application host that
> consumes it. A transport is protocol — so the WebRTC data channel belongs in core-rust behind
> `cfg(target_arch = "wasm32")`, the shape `core/crypto` already uses.**

The alternative — browser-rust implements WebRTC itself — produces a **second partial protocol implementation
inside an app host**, diverging from core-rust with no conformance gate over it.

**Two assumptions this corrects:**

1. **browser-rust is not disconnected.** It reaches native peers over `ws://` today. The "❌ column" framing is
   about **direct punch**, not connectivity — now pinned in `EXTENSION-SIGNALING.md` §7.3.1. **WebRTC is an
   optimization; no browser product capability is gated on it.**
2. **browser-rust is an app host** — 196 files of `app_host/`, `apps/`, `dom/`, `content_site/`, consuming
   core-rust's primitive crates. No signaling, no webrtc, no peer stack. It should not grow one.

### 5.2 The order

| | Step | Owner | Blocked on |
|---|---|---|---|
| **S1** | **wasm probe** — add `signaling` (+ `network`) to `WASM_FEATURES` | core-rust | nothing |
| **S0** | **the flag day** (§3.1) | arch + core-rust + core-go | nothing |
| **S2** | fold WebRTC as `system/signaling/webrtc/*` | arch | S0 |
| **S3** | WebRTC transport crate, wasm32-gated | core-rust | S2 |
| **S4** | consume it; publish `system/peer/transport/webrtc` | browser-rust | S3 |
| **S5** | the gate — browser peer ↔ native peer, real data channel, real signaling node | cohort | S4 |

**S1 first, even though S0 is bigger.** The signaling crate was **written to be wasm-portable and never put in
the wasm lane** — it depends on `web-time` (the wasm-safe clock, chosen over `std::time::Instant`, which panics
in the browser) and its `core.rs` takes no entity-system dependencies. `WASM_FEATURES` today is
`inbox, continuation, subscription, clock, revision, query, history, compute, handlers, identity, role,
registry, discovery, type-system, content` — no `signaling`, no `network`.

**Adding one Makefile line converts the whole browser-leg estimate from guess to fact:** either the carrier half
already runs in the browser and only the transport is missing, or it returns a concrete portability list. **Do
it before writing any WebRTC code**, so S2 is folded against known portability.

**Settle at the S2 fold, not during the run:**
- **`system/signaling/webrtc/*`** as the namespace (closes the WebRTC proposal's open item #2).
- **DTLS fingerprint pinning as a MUST** (its open item #3) — the receiver verifies the negotiated fingerprint
  matches the signed offer; without it a signaling MITM substitutes the channel. A security MUST should not
  wait for a cross-impl run to find it.
- **Reachability class** (open item #1) — confirm the leaned answer: no new class; it is a punch substrate at
  the §10.3 seam.

**Deferred, explicitly:** native WebRTC (a native peer publishing a `webrtc` profile — §7.3.1's **MAY**) needs a
`webrtc-rs`-class stack and buys nothing S0–S5 does not; **G4** (two real networks) is operator-scheduled.

## 6. Execution checklists

**Arch — unblocked now:**

- [ ] Rename §3.1 across `EXTENSION-SIGNALING.md` (types, the §3 key-derivation string, §12 type table, all
      cross-references).
- [ ] Rename §3.2 across `EXTENSION-INBOX.md`, `EXTENSION-SUBSCRIPTION.md`, `EXTENSION-DURABILITY.md`,
      `EXTENSION-ENCRYPTION.md` + corpus cross-references.
- [ ] Relocate machine-spec §1.8 → `ENTITY-CBOR-ENCODING.md`; repoint `AGENTS.md` + `GUIDE-CONFORMANCE.md`.
- [ ] Delete `ENTITY-CORE-MACHINE-SPEC.md`; drop its `spec-tool/config.default.toml` entry.
- [ ] Fold WebRTC (S2) once S0 lands.

**core-rust:**

- [ ] **S1 — add `signaling` (+ `network`) to `WASM_FEATURES`; report what breaks.** Cheapest high-information
      step available; **do this first.**
- [ ] S0 — rename the `system/nat/*` constants **and the rendezvous-key derivation string**, in lockstep with go.
- [ ] Repoint machine-spec §1.8 citation in `core/protocol/src/verify.rs`.
- [ ] S3 — the wasm32-gated WebRTC transport, after S2.

**core-go:**

- [ ] S0 — same rename, same window as rust. **Neither ships alone.**
- [ ] Repoint 2 machine-spec §1.8 citations (validator declarations) and 2 §3.9 citations.
- [ ] §3.2 inbox constants — and **reconsider whether the type still belongs in `core/types/` once the prefix no
      longer says `protocol`** (§3.2).

**browser-rust:** nothing until S4. **Continue L5 app-host work; the browser leg is not blocked on you.**

**keystone / py:** expected zero for §3.1 and §3.2 — verify rather than trust; adoption converts a spec edit
into a migration.

## 7. Risk

- **The flag day is the only real risk**, and its failure mode is silent (peers never meet). **Mitigation:
  land go and rust in one window**, and confirm with a cross-impl meet before either is deployed.
- **Everything else is a string rename** with no wire consequence.
- **Sizing is the cohort's.** Arch's cost estimates have been corrected five times this arc against zero
  ownership rulings overturned; every effort claim here is pending an implementer's check.
