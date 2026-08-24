# PROPOSAL — tiering the permission to put optional fields on core types

**Status:** ✅ **RATIFIED + FOLDED 2026-07-31** → `specs/SPECIFICATION-FORMAT.md` **§8.4.1**. Operator confirmed
the test at ratification **and added a third question** (the seam check — *"we need to do all our analysis
always"*), which is folded with the rest. A companion **§8.4.2** lands alongside it covering the namespace
question this proposal deliberately did not answer (§6 item 4) — see
`docs/research/explorations/ANALYSIS-CORE-NAMESPACE-CLAIMS-IMPACT.md`.
**Target:** `specs/SPECIFICATION-FORMAT.md` §8.4 (Optional Fields on Core Types) — a qualifier on an existing
permission, plus a test an author can apply. No change to core, no change to any extension, no wire effect.
**Scope:** *which* extensions may exercise §8.4's permission. It does not change the mechanism (open types),
does not revoke any existing field, and does not touch `ENTITY-CORE-PROTOCOL.md` §2.10.
**Origin:** an operator ruling, 2026-07-31 — recorded here rather than left in a transcript.

---

## 1. What §8.4 says today

> **Extensions MAY define optional fields that appear on core types** (e.g., `deliver_token` on EXECUTE). These
> are documented in the extension spec, not the core spec. The core spec's Open Types (§2.7) guarantees
> preservation.

The permission is **unqualified**: it reads as available to any extension. Backed by
`ENTITY-CORE-PROTOCOL.md` §2.10 (unknown fields MUST be preserved, MUST NOT be rejected, are covered by content
hashing), so the mechanism genuinely works for anyone who uses it.

## 2. The problem

**Core types are permanent, and open types make adding cheap while making removal impossible.**

An optional field on a core type is carried, preserved, and hashed by **every peer in the ecosystem forever** —
including every peer that will never implement the extension that defined it, and including peers built long after
the motivating technology is gone. There is no deprecation path: §2.10's preservation guarantee is exactly what
makes removal unsafe, because some peer's stored bytes and content hashes depend on it.

So the permission's cost is not paid by the extension that exercises it. It is paid by the whole ecosystem, once,
permanently. An unqualified permission prices that at zero.

**The concrete case that surfaced it.** `EXTENSION-SIGNALING` (NAT traversal and rendezvous) is landscape-specific
in a way the existing standard extensions are not. NAT traversal is an artifact of **IPv4 address exhaustion and
middlebox deployment** — a contingent property of one era's internet, not a durable property of a peer-to-peer
entity protocol. A field it planted on a core type would outlive its own reason to exist. *(As folded,
`EXTENSION-SIGNALING` v1.0 puts **no** field on any core type — this proposal is not a fix for it, it is the rule
that would have prevented the question.)*

## 3. The proposed rule

> **§8.4's permission is tiered.** An extension MAY define optional fields on core types **only when its concept
> is durable and broadly applicable** — part of the standard capability layer that peers generally participate in.
> An extension whose concern is **specific to a technology landscape, a deployment topology, or a particular
> feature set MUST NOT extend core types.** It defines its own types and composes through seams.

**The test an author applies — three questions, all three must pass. Answer all three in writing, every time;
a field that fails any one of them does not qualify, and a field that passes two is not a close call.**

1. **Durability.** *Would this field still make sense if the technology that motivated it disappeared?*
   `deliver_token` on EXECUTE describes **delivery authorization**, which any message-passing system has —
   it passes. A `nat_candidate` field describes an artifact of **middlebox behavior in the IPv4 era** — it fails.
2. **Universality.** *Would a peer that never implements this extension still reasonably carry this field on that
   type?* If the answer is "no, it is dead weight for most peers," the field belongs in the extension's own types.
3. **Seam check.** *Is composing through a seam genuinely worse here — and why, concretely?* The alternative
   always exists (§3, below), so a core-type field must be shown to be **better**, not merely convenient. State
   what the seam version would look like and what it costs. **"A seam would be awkward" is not an answer; "a seam
   cannot express this because X" is.** An author who cannot articulate the seam design has not yet established
   that they need the field.

**Why three and not two.** Durability and universality are properties of the *field*; they can both pass while the
extension still had a cheaper option it never considered. Question 3 is the only one that forces the alternative
to be designed rather than dismissed — and since §8.4's whole cost is that **core types are permanent**, the
burden belongs on the irreversible choice. It also produces a durable record: the answer to question 3 is the
rationale a future reader needs when asking why this field is on a core type.

*(Question 3 was added by operator ruling at ratification: do the full analysis every time. The two-question form
would have admitted a field on the strength of two easy passes and no examination of the alternative.)*

**Qualifying — the four that exist today, all re-swept against all three questions:** INBOX (`deliver_token` on
EXECUTE), TYPE (`constraints` on `system/type/field-spec`), CLOCK (`clock` on `system/revision/entry`), and
SYSTEM-COMPOSITION (`composition` on `system/handler`). Each names a concern that outlives its motivating
technology and that any peer plausibly has.

**Adding question 3 produced a sharper discriminator than expected, and it is worth stating as the rule of
thumb.** All four pass, and all four pass for the *same structural reason*: **the information has to be in the
bytes, not merely available at a call site.**

| Field | Why a seam cannot express it |
|---|---|
| `deliver_token` on EXECUTE | The authorization travels **on the wire to another peer**. A seam is a local call; it cannot put a field in an envelope the remote peer parses. |
| `constraints` on `system/type/field-spec` | The constraint must travel **with the type definition**, which is fetched and validated by readers who may run no validator extension at all. |
| `clock` on `system/revision/entry` | The entry is **hashed and replicated**. A stamp attached at a seam is not in the hash, so it does not survive the thing it describes. |
| `composition` on `system/handler` | Anyone loading the handler entity must see it; a seam would require every reader to call a resolver that may not be installed. |

**So question 3 has a concrete form:** *does this information have to be carried in the entity's bytes, or does it
only have to be available where the code runs?* **Bytes ⇒ a field may be justified. Call site ⇒ use a seam.**
Every connectivity extension's data — candidates, observed addresses, punch coordination — is call-site data,
which is why SIGNALING composes through `establish_live` and needs no core-type field. *This also explains why
the two-question form never rejected anything: durability and universality are properties of the concept, and a
call-site-only concern can pass both while still not needing a field.*

**Not qualifying, illustratively:** SIGNALING, WEBRTC-transport, and any future extension for a specific
transport technology, NAT/middlebox behavior, or a single deployment shape.

**The alternative always exists and costs nothing.** A non-qualifying extension composes exactly as
NETWORK / RELAY / ROUTE / SIGNALING already do — its own types plus a **seam** the substrate exposes
(`dispatch_fallback` §10.2, `establish_live` §10.3, `resolve_next_hop`). Seams are additive, removable, and
carried only by peers that install the extension. **This proposal restricts one mechanism precisely because a
better one is already in universal use.**

## 4. What this does not do

- **Does not revoke any existing field.** All four known instances pass §3's test (§6 item 3). No migration, no
  deprecation, nothing for any implementation to change.
- **Does not change the mechanism.** `ENTITY-CORE-PROTOCOL.md` §2.10 open types are untouched; unknown fields are
  still preserved, still hashed, still MUST NOT be rejected. This is a rule about **who should**, not about what
  the wire does.
- **Does not gate on a registry or an approval step.** It is a design test an author applies and a reviewer
  checks, in the same way "pin what diverges across a peer boundary" is applied.
- **Does not touch the core spec.** The qualifier lives in `SPECIFICATION-FORMAT.md`, which is where §8.4 lives.

## 5. Why a rule rather than case-by-case judgment

The distinction is easy to get wrong **in the permissive direction**, because §8.4 as written offers a genuinely
convenient mechanism and the cost is invisible at authoring time — it lands on other peers, later. The failure
shape is familiar to this corpus: *a choice that is locally correct and globally expensive, where nothing fails
loudly at the moment it is made.*

Two recent instances argue for writing it down:

- A service-declaration field was drafted onto `system/handler` (a core type) and moved off it during audit. The
  move was right, but the **stated reason was wrong** — it claimed §8.4 forbade the field, which it does not. Had
  the tiering rule existed, the correct reason would have been available at authoring time.
- The same audit had to reason from first principles about whether SIGNALING could extend core types, with no rule
  to consult.

## 6. Open items

| # | Item | Disposition |
|---|---|---|
| 1 | **Where the boundary sits for borderline extensions** — e.g. ENCRYPTION, REGISTRY, DISCOVERY | The two-question test is deliberately a judgment aid, not a classifier. If a future case genuinely splits reviewers, that case is the one that refines the rule. **Not pre-solved here.** |
| 2 | **Should §8.4 name the qualifying extensions explicitly?** | *Leaned no* — an enumeration ages badly and invites "add mine to the list" pressure. The test travels better than a roster. Illustrative examples only. |
| ~~3~~ | ~~Retroactive audit of existing core-type fields~~ | **DONE 2026-07-31 — swept, all pass, no action.** Four extension-added fields on core types exist, not three: `deliver_token` on EXECUTE (INBOX §2.3), `constraints` on `system/type/field-spec` (TYPE §251), `clock` on `system/revision/entry` (CLOCK §5.5), and **`composition` on `system/handler`** (`SYSTEM-COMPOSITION.md` §416 — found by the sweep, not previously listed). All four pass both questions: each names a concern that survives its motivating technology and that a peer plausibly has. Two near-misses checked and excluded as **core-defined, not extension-added**: `cascade_depth` on `system/bounds` and `budget_consumed` on `system/protocol/execute/response` are both in ENTITY-CORE-PROTOCOL.md already. |
| ~~4~~ | ~~**Namespace claims are a separate question this proposal does not answer.**~~ | **CLOSED 2026-08-01, after a wrong first answer.** See `ANALYSIS-CORE-NAMESPACE-CLAIMS-IMPACT.md`. **The flag as worded is not a violation** — core defines `system/protocol/inbox/*` itself and `SYSTEM-COMPOSITION.md` is a core-model spec. **But the 07-31 conclusion that this made the placement *correct* was circular and is withdrawn.** Asking the real question — *are these protocol messages?* — says no: `system/protocol/**` holds wire messages and their components, **7 of its 9 members are V7 §9.5 floor types and the only 2 that are not are the inbox pair**; the machine spec's own §3.2 "Protocol Types" excludes them; their §3.9 siblings use `system/subscription/*`. **Routed upstream** (`ROUTING-2026-08-01-inbox-types-namespace-upstream.md`) with the `result` drift folded into the same change. `system/handler/composition` passes — no action. General rule folded as **§8.4.2**, now including the **placement test** that the first pass lacked. |

## 7. Validation

A rule about authoring, so there is no wire behavior to exercise. It is "validated" by the next extension author
reaching §8.4 and getting the right answer without needing an operator ruling — which is exactly what did not
happen this week.

## 8. References

- **Target:** `specs/SPECIFICATION-FORMAT.md` §8.4.
- **Mechanism it qualifies (unchanged):** `ENTITY-CORE-PROTOCOL.md` §2.10 Open Types.
- **Precedent fields that pass the test:** `EXTENSION-INBOX.md` §2.3 (`deliver_token` on EXECUTE);
  `EXTENSION-CLOCK.md` §5.5 (`clock` on `system/revision/entry`); EXTENSION-TYPE (`constraints`).
- **The composition alternative:** `EXTENSION-NETWORK.md` §10.2 / §10.3 seams; `EXTENSION-ROUTE.md` §4.
