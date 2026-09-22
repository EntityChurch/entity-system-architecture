# PROPOSAL — pin the status on `path_required`, at the site that generalizes it

**Status:** IMPLEMENTED — ruled and folded in the same session it was written. D1–D5 all landed;
nothing was unknown, so leaving it DRAFT would have been fold debt (**L26**). The §4 oracle-check
item does **not** fold with it and is carried to `COHORT-OPEN-ITEMS` (**L9**).
**Tier:** extensions (four extension specs + the guide that generalizes the rule).
**Filed from:** `entity-system-generator`'s `ROUTING-2026-09-03-arch-content-v3-6…` **F2**, reading
CONTENT for code emission. Their framing was correct and their ask is adopted with one change: the
restatements name the authority rather than each re-pinning the status.

---

## §1 The defect

`path_required` is a **MUST-return** error code in **eight normative sites across five documents**,
and **not one of them names a status.**

| Document | Site | Operations |
|---|---|---|
| `EXTENSION-CONTENT` | §6.2 | `system/content:get` |
| `EXTENSION-CONTENT` | §6.3 | `system/content:ingest` (with an explicit v3.4 → v3.5 behaviour-change callout) |
| `EXTENSION-ATTESTATION` | §6 (`:589`) | `:create` / `:supersede` / `:revoke` |
| `EXTENSION-ATTESTATION` | §6 (`:678`, conformance list) | same three |
| `EXTENSION-QUORUM` | §6 (`:375`) | `:create` / `:update` / `:publish` |
| `EXTENSION-QUORUM` | §6 (`:507`, conformance list) | same three |
| `GUIDE-EXTENSION-DEVELOPMENT` | §171 | generalized — *"a `:create` / `:supersede` / `:revoke` style op"* |
| `GUIDE-ATTESTATION` §50 · `GUIDE-QUORUM` §50 | restatements | inherit |

**Searched and confirmed absent:** the string `path_required` does not occur anywhere in
`entity-core-protocol/specs/` — not in `ENTITY-CORE-PROTOCOL.md`'s status table (L830–L833), not in
its error registry. Sections read to establish the negative: L830 (the 400 row), L833 (the 404 row),
§4.7's table (L1909–L1944), §6.5's dispatch ladder (L3137), §5.2a (L2440). *(The negative-claim rule,
adopted from `entity-core-keystone`: a claim that the corpus is silent records which sections were
read, by number.)*

**This is L17.** A `[MUST]` naming a value, with no declared site for the value and no check that
reads it. The 2026-08-19 instance was `max_ttl` — four seats, three config keys. Here the value is a
status and the divergence surface is every peer that implements any of the three extensions.

**It has not diverged yet only because there is one implementation.** `entity-core-go`
`ext/content/handler.go:153` returns `handler.NewErrorResponse(400, "path_required", …)`. The oracle
does not constrain it: `cmd/internal/validate/content.go:155,166` asserts `status != 200` and
`code == "path_required"`, so a peer answering `404` or `422` passes today. The generator is about to
become the second implementer and asked rather than picking — which is the only reason this is being
ruled before it costs something.

---

## §2 Derivation — from the landed text, not from what go does

`ENTITY-CORE-PROTOCOL` L830 defines the 400 row:

> *"400 | Bad request — malformed, mis-addressed, or otherwise structurally invalid. Default `code` =
> **`invalid_request`**; more-specific 400 codes where one applies: `invalid_path`, `invalid_params`,
> `unexpected_params`, `chain_depth_exceeded`, `signature_path_conflict`. … `invalid_request` … is the
> generic 400 code an extension handler uses for a structurally invalid request."*

An EXECUTE that omits a required `resource` field is **structurally invalid** — the request cannot be
acted on as a request. That is the 400 class, unambiguously, and no other row fits: it is not 404
(the handler exists and the operation is implemented), not 403 (nothing has been evaluated for
permission — indeed the resource is what the cap scope reads, so there is nothing to check), and not
501 (the operation is implemented; this call is malformed).

**The one question worth asking is whether `path_required` is a forbidden synonym.** L1913, normative
at `0.8.2.4`, says: *"Extension specifications use this code for the same class and **MUST NOT mint a
synonym**."* The answer is no, and the row itself draws the line: it **sanctions more-specific 400
codes where one applies**, and names `invalid_params` and `unexpected_params` among them — codes of
exactly this genus, naming *which* structural thing is wrong. A forbidden synonym is a differently
spelled **generic** (`entity-core-py`'s retired `bad_request` is the worked example). `path_required`
names a specific, remediable defect, and §4.7 states the governing principle outright: **the code
selects the remedy.** *Supply a resource* is a different remedy from *fix your request*.

**So: `400`, code `path_required`, as a more-specific 400 under the L830 row.**

**Corroboration, cited last and verified in the tree (L18).** `entity-core-go` at `ad6cef1` returns
`400 path_required` at `ext/content/handler.go:153` — the only implementation. It is the same answer.
**Apply L18's operative test:** if go had returned `404`, the derivation above would be unchanged and
go would be non-conformant. The citation is therefore not load-bearing and is here only because a
reader will otherwise wonder whether it agrees. It does.

---

## §3 The delta

The generator asked us to pin it at the `GUIDE-EXTENSION-DEVELOPMENT` §171 site *"so all four
extensions inherit it."* **Adopted, with L23's fourth shape applied:** inheritance is invisible from
the restatement, so a reader of `EXTENSION-QUORUM` §6 still cannot see the status. **A restatement
names its authority**, which makes the next sweep a grep instead of a search.

| # | Document | Edit |
|---|---|---|
| **D1** | `GUIDE-EXTENSION-DEVELOPMENT` §171 | Pin the status: *"…MUST return **`400 path_required`** — a more-specific 400 under `ENTITY-CORE-PROTOCOL` §L830's row, not a synonym for `invalid_request`."* **This section is the authority for the rule.** |
| **D2** | `EXTENSION-CONTENT` §6.2, §6.3 | Append to each MUST: *"`400`; `GUIDE-EXTENSION-DEVELOPMENT` §171 is the authority for this code."* |
| **D3** | `EXTENSION-ATTESTATION` §6 (both sites) | Same. |
| **D4** | `EXTENSION-QUORUM` §6 (both sites) | Same. |
| **D5** | `GUIDE-ATTESTATION` §50 · `GUIDE-QUORUM` §50 | Same — these already read as restatements and should say so. |

**Not proposed: adding `path_required` to `ENTITY-CORE-PROTOCOL`'s status table.** The L830 row is a
*pattern* — it enumerates five examples and licenses more — and every extension-specific 400 code
would have to be listed for the table to be an authority rather than a sample. The core spec is the
right home for the *class*; the generalizing guide is the right home for this *member*. Raising it to
core is available later if a second extension-family code needs the same treatment, and would then be
a rule about where extension codes live, not a one-row addition.

---

## §4 What must be true before this lands (L17's second half)

**(i) A declared site a peer can carry.** Satisfied by construction — the value is a status on an
error response, which every handler already emits.

**(ii) A check that reads it.** **Not satisfied, and this is the owed half.** The existing oracle
assertion at `cmd/internal/validate/content.go:155,166` deliberately accepts any non-200. Tightening
it to `status == 400` is a one-line change in a **`validate-peer` behavioural check**
(`GUIDE-CONFORMANCE` §7.0's first class — *not* a fixture-corpus artifact, and not an impl-internal
test). **Satisfaction mode:** directly constructible by any conformance client — issue
`system/content:get` with no `resource` field and read the status. No harness capability is needed
and no state must be reached, which is why this is landable rather than pinned unconstructible.

**Ownership, per the routing model:** the check is `entity-core-go`'s harness and the tightening is
theirs to make, on their own schedule. **Arch does not author the check and is not asking for a
sweep** — the three extensions have one implementation between them today, so there is nothing to
converge yet. Route the pin; the check follows when a second implementation exists to disagree with.

---

## §5 Sequencing

**Nothing waits on this.** `entity-system-generator` said they will emit `400` and record it as a
greppable assumption in the generated module header. That is the right posture and this proposal
confirms it rather than unblocking it — build 1 proceeds either way, and if the ruling had gone the
other way their regeneration is mechanical.

**Is it a flag day (L21's third shape)?** No. `path_required` is currently emitted by one peer at
`400`; the ruling makes that spelling normative rather than changing it. Nothing that any peer
currently accepts becomes refused, and no peer becomes unreachable.
