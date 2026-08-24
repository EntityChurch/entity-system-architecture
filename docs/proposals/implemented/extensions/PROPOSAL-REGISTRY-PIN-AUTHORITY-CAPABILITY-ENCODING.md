# PROPOSAL — pin authority needs an operation name, because a grant cannot say "the same write, but different"

**Status:** **IMPLEMENTED — folded 2026-08-20, `EXTENSION-REGISTRY` 1.19 → 1.20.** All five §4 deltas
verified landed against the tree before this marker was set (L3). Ruling requested by `entity-core-go`
(spec-issue `2026-08-20-a`, their `a34c61f`), independently reached by `entity-core-rust` and
`entity-core-py`.

> **Closed three-way on the wire before the fold, which is unusual and worth recording.**
> `entity-core-go` built fresh `entity-core-rust` (`951f0ce`) and `entity-core-py` (`5d9fdd1`) peers
> and drove its own row-8 harness against both from one seat: **`registry 18/18 · 0F · 0S` at all
> three.** That is a live cross-impl run, not three seats reporting their own gates — the distinction
> `AGENTS-STANDARD` draws between cohort-consistent and independently measured. **And the check
> discriminates:** a fail-open peer answers row 8a (a pin change under a configure-only cap) with
> `200` and is scored FAIL; both siblings returned `403 not_entitled` and wrote nothing, and row 8c
> rules out the opposite over-strict error.
> **Honestly scoped by them, and adopted here unchanged:** clause 4's *ordering* in its sharpest form
> — a config that both changes pins and violates §4.1 step 2 — is **in-tree teeth at each seat, not
> proven on the wire by that run.** Recorded as such rather than folded into the 18/18.
**Tier:** extensions — `EXTENSION-REGISTRY` §4.3 (the pin-delta clause) + §5 (the capability table).
**Read at:** arch this commit · go `483418a` · rust `cb4bb97` · py `5d31511` · browser-rust `7519064`
· workbench-go `2d79fa2`.
**Release scope:** **`EXTENSION-REGISTRY` v1.** This unblocks `REG-DISPATCH-CONFIG-REFUSED-1` row 8,
which is on §0a's IN list — so it gates the release and is deliberately small.

---

## §1 The question

`EXTENSION-REGISTRY` §4.3 `[MUST, v1.19]`: *"a write that changes `pinned_bindings` additionally
requires `system/capability/registry-pin` … absent the latter, refuse `403 not_entitled` and write
nothing."*

**The behavior is settled, implemented three-way, and not in question.** What is unspecified is **how
a capability expresses "pin authority"** — and until that is fixed, a portable conformance vector
cannot mint the capability the MUST requires, because it does not know what to mint.

---

## §2 The derivation — the clause refutes itself in its own sentence

**§4.3's pin-delta paragraph diagnoses the defect it then reproduces.** It reads:

> *"A capability whose operations column is a bare tree-write is a name with no enforcement point: a
> raw write cannot refuse selectively, cannot carry a qualifier, and **cannot be distinguished from
> any other write to the same entity**. This clause is that enforcement point."*

**It fixes the first two and not the third.** Moving the check from a tree-write to an operation gives
the handler somewhere to refuse selectively and something to carry a qualifier. But the clause then
makes `registry-pin` and `registry-configure` name **the same operation** — `set-resolver-config` —
so pin authority **still cannot be distinguished from any other write to the same entity**, by the
one artifact that has to distinguish it: **a grant.**

The enforcement point ended up inside the handler. **The authority stayed indistinguishable.**

### §2.1 Why a grant cannot express the difference — from V7, not from any implementation

A capability's authority is a `system/capability/grant-entry` (V7 §5), and it scopes along exactly
two axes:

- **path-scope** — `resources`, matched by §5.4's `matches_pattern` (canonicalizing, peer-wildcard).
- **id-scope** — `operations` and `peers`, matched as a **literal string** with exactly two wildcard
  forms, per the grammar pinned at V7 0.8.1 F40.

**The path axis is not portable, and V7 says so outright** (§ *"Convention vs structure"*):

> *"Interoperability depends on shared type vocabulary — the `type` field strings in entity data —
> **not on where entities are stored in the tree**… All other path locations are communicated through
> the initial capability grant: the grant's `resources` patterns tell the connecting peer where to
> find system data on this peer. **Peers that diverge remain conformant.**"*

So a resource-path discriminator cannot be assumed by a party that did not receive the grant — which
is precisely a conformance harness minting a capability for a foreign peer. **A path split is
unavailable by construction**, before any argument about whether it is tidy.

*(It is also unavailable in fact: `pinned_bindings` is a **field of** `system/registry/resolver-config`,
not a path under it, so there is no narrower resource to name.)*

**The operation axis is portable**, and it is the only one that is: operation names are shared
vocabulary, the spec names them, and their matching grammar is normatively pinned.

**Therefore:**

> **The only cross-impl-portable discriminator a capability has is its operation name. Two
> capabilities that name the same operation and the same resource are the same capability to every
> conformant grant. `registry-configure` and `registry-pin` currently are.**

That is the whole argument, and it does not depend on what any seat built.

### §2.2 §5's own maintained rule was satisfied in letter and not in substance

The table carries a standing rule, added because three rows had already failed it:

> *"**Every row's operations column names an operation, and that is the property this table is
> maintained for.** … A new row is not landable without naming the operation the check runs at."*

**`registry-pin`'s column does not name an operation.** It names *"the §4.3 `set-resolver-config`
write whose submitted `pinned_bindings` differ from the stored ones"* — **a condition on another
row's operation.** That reads as compliance and is not: the rule exists so that authority has a name
a grant can carry, and a runtime data condition is not something a grant can carry.

**This is arch's defect, not the cohort's.** The rule was written one incident too narrow — against
*"a bare tree-write"* — so it caught the absence of an enforcement point and missed the absence of a
**distinguishable** one.

### §2.3 It is cross-peer observable, which is what makes it a MUST rather than a convention

§5.2 grants the local peer all seven caps at install, so today these are mostly held locally. **But
V7's `delegate` operation exists**, and a peer that delegates registry authority to another peer hands
over a grant whose `operations` the recipient must match. `AGENTS.md` is unambiguous about this class:

> *"Pin the cross-impl-observable surface, leave internals to converge. A `MAY`/`SHOULD` whose two
> conformant readings diverge across a peer boundary is a latent interop bug — lean MUST."*

**A capability encoding is a cross-peer observable.** Leaving it implementation-defined is the
`0`-is-dropped and `max_ttl` failure shape again: four faithful implementations, four keys, and an
operator whose grant means something different at every seat.

---

## §3 The ruling

**Pin authority is expressed as authority over the operation `pin-bindings`.**

1. **`system/capability/registry-pin` is the authority to invoke the operation `pin-bindings` on the
   registry handler `[MUST]`.** That is the name a grant carries and the name a harness mints.
2. **`pin-bindings` is a capability-check discriminator and MUST NOT be dispatchable `[MUST]`.**
   Nothing routes to it; it appears in no operation table; a peer MUST NOT accept it as an EXECUTE
   operation. **This is pinned because it is the one place a seat could diverge into a new wire
   surface** — an operation name that is checkable but not callable is unusual, and a seat that
   exposed it would add an undeclared operation to the registry handler.
3. **The pin-delta check runs `[MUST]`** on `set-resolver-config` when the submitted `pinned_bindings`
   **bytes** differ from the stored config's; absent `pin-bindings` authority, refuse **`403
   not_entitled`** and write nothing. *(Byte comparison, not structural — see §5.)*
4. **Capability checks precede config validation `[MUST]`.** A `set-resolver-config` that both changes
   pins and violates §4.1 step 2, submitted with configure-only authority and no acknowledgement,
   returns **`403 not_entitled`** — never `403 policy_rejected`. **Two reasons, and the second is the
   load-bearing one:** the order is otherwise cross-impl-observable and unruled; and §4.3's violation
   response is deliberately verbose (*"every violation, not the first"*), so validating first hands a
   **config-shaped disclosure to a caller who lacks the authority to change it.** Authorize, then
   validate.

**What is NOT ruled:** how a peer stores or names the capability internally, whether `pin-bindings`
appears as a constant, and every other in-process detail. **The pinned surface is the operation string
a grant carries.**

---

## §4 Deltas

| # | File | § | Delta |
|---|---|---|---|
| D1 | `EXTENSION-REGISTRY` | §5 table | `registry-pin`'s **Operation(s)** column becomes **`pin-bindings`** (non-dispatchable; checked at §4.3's pin-delta). Today it names a condition on another row's operation |
| D2 | `EXTENSION-REGISTRY` | §4.3 | State §3.1–§3.2: the check is authority over `pin-bindings`, and that operation **MUST NOT** be dispatchable |
| D3 | `EXTENSION-REGISTRY` | §4.3 | State §3.4: **capability checks precede config validation**, with the disclosure-leak reason |
| D4 | `EXTENSION-REGISTRY` | §5 | Strengthen the maintained rule: *"names an operation"* means **an operation name a grant can carry** — a condition on another row's operation does not satisfy it. This is the rule that let the row land |
| D5 | `EXTENSION-REGISTRY` | §11 | `REG-DISPATCH-CONFIG-REFUSED-1` **row 8 becomes constructible.** Declare its class — **`validate-peer` check** — and its satisfaction mode per `GUIDE-CONFORMANCE` §7.0 / §5.2b.1 |

**D1–D3 are the ruling. D4 is the defect underneath it. D5 is what the ruling was for.**

---

## §5 One thing the seats settled that this proposal adopts rather than rules

**The comparison is over raw bytes, not a decode round-trip**, and §4.3 already says *"byte-identical."*
`entity-core-rust`'s review of `entity-core-go` found go's compare was **fail-open**: a typed-struct
decode silently drops any §4.2 forward-compat key a pin carries, so a config that adds or strips one
read as *"no change"* and could rewrite `pinned_bindings` under `registry-configure` alone. Fixed at
go `f44ed4d` with a mutation-verified test; py decodes to a native mapping and had no field-drop;
rust compared raw bytes already.

**This proposal does not re-rule that** — §4.3's word was already *"byte-identical"* and rust read it
literally, which is the correct reading. It is recorded because **D1's operation name and this byte
rule are the two halves of one check**, and a fold that lands the first without restating the second
leaves the enforcement point precise about *who* and vague about *what changed*.

---

## §6 Corroboration, cited last and labelled as such (L18)

**All three core seats independently chose the operation axis and independently chose the string
`pin-bindings`** — go `pin-bindings` (spec-issue `2026-08-20-a`), rust `OP_PIN_BINDINGS` at
`extensions/registry/src/resolver.rs`, py `_OP_PIN_BINDINGS` at `entity_handlers/registry.py`, whose
comment independently records both *"**Not a dispatchable operation**"* and *"**Why the operation axis
and not a resource path**."*

**That is corroboration and it is not the argument.** L18: *three seats agreeing is three seats
agreeing, and unanimous agreement on a wrong reason is the failure mode that looks most like
validation.* The operative test — **if all three had chosen a resource path, would §2 change?** It
would not: V7's *"peers that diverge remain conformant"* on path locations makes a path discriminator
unportable regardless of who implemented it.

**What the convergence legitimately buys:** evidence the reading is implementable, cheap, and that
three practitioners under delivery pressure reached it unprompted — including the non-obvious
non-dispatchable half. **It is why this proposal pins `pin-bindings` rather than inventing a name**:
where the derivation fixes the *axis* and not the *string*, the string that three seats already ship
is the one to take.

---

## §7 Why not the other option

The spec-issue offered a second: **declare the encoding implementation-defined and scope row 8 to
in-process per seat.** Rejected, and the reason is not that it fails today.

It would work now, because §5.2 grants everything locally and nobody delegates registry authority yet.
**It fails the moment anyone does** — and the failure is a grant that means something different at
each seat, which is exactly the class §3b.0 already cost this extension an entire subsection to fix
one field over. **The cost of option 1 is one string; the cost of option 2 is discovering the
divergence from a `403` on a valid grant.**

**It would also make row 8 permanently un-green as cohort coverage** while three seats each report
in-tree teeth — the *"3-way green"* ambiguity `GUIDE-CONFORMANCE` §7.0 exists to forbid.
