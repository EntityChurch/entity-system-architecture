# PROPOSAL — the handler frame is the owning handler, and the corpus states it three ways in thirteen places

**Status:** DRAFT 2026-09-13 *(census corrected after cross-implementation review, same day)*
**Proposes:** corrections to the `check_path_permission` call sites in `EXTENSION-QUERY`,
`EXTENSION-COMPUTE`, `EXTENSION-TRANSACTION` and `EXTENSION-REVISION`, folding the `handler_pattern`
rule stated in `ENTITY-CORE-PROTOCOL` §6.3.
**Target:** `specs/extensions/EXTENSION-QUERY.md` · `specs/extensions/EXTENSION-COMPUTE.md` ·
`specs/extensions/EXTENSION-TRANSACTION.md` · `specs/extensions/EXTENSION-REVISION.md`
**Depends:** the core-protocol revision that states the rule in §6.3.

**This document does not rule.** The rule and its derivation belong to the core protocol, which owns
`check_path_permission`. This is the extension-tier fold checklist: the call sites in this corpus that
state the rule wrongly, or do not state it at all. Where the two disagree, the core specification
wins.

---

## 1. The rule being folded

> **`[MUST]` The handler frame is the handler that OWNS the operation being authorized, not the
> handler performing the check.** When a handler authorizes an access whose operation belongs to
> another handler's surface — a tree read, a tree write — it passes **that** handler's pattern.
>
> **`[MUST]` `handler_pattern` is REQUIRED.** An absent, null or empty value MUST NOT be read as
> *"match all handlers."* A call site that cannot name its frame is a defect at that call site.

`handler_pattern` filters the authority's grants **before** their `resources` scope is read
(`ENTITY-CORE-PROTOCOL` §6.3). A wrong frame therefore discards the correct grant *unread*, and the
check answers DENY with no dimension to attribute the refusal to. An omitted frame does the reverse:
it considers every grant, and an authority scoped to some unrelated handler authorizes the access.

**Why the owning handler is the right answer.** The frame answers *whose authority is being spent*,
and the answer is the handler whose namespace the access lands in. A caller granted `system/tree:
get` has been granted a tree read; whether that read is reached through the history, compute,
subscription or query surface is the caller's route, not a second authority they must separately
hold.

⭐ **This is already stated in the imperative in a landed specification.** `EXTENSION-SUBSCRIPTION`
§2.3 is normative today:

> *"the subscribe handler **MUST** additionally verify the caller's capability covers the tree read —
> `check_path_permission("get", resource_path, caller_capability, "system/tree", local_peer_id)`"*

…for a handler that is **not** the tree handler, with the matching conformance row in that document's
§9. **The fold below generalizes a rule this corpus has already ratified on one path**, rather than
introducing one.

## 2. The census — 13 call sites in 7 documents

Counted with a scan that joins continuation lines, because the call wraps at four of these sites and
a per-line scan mis-reads it.

**Nine sites carry a frame:**

| site | frame passed | checking handler | discriminating? | states |
|---|---|---|---|---|
| `EXTENSION-TREE` §1083 | `system/tree` | tree | no — owner == runner | ✅ |
| `EXTENSION-TREE` §1671 | `system/tree` | tree | no — owner == runner | ✅ |
| `EXTENSION-CONTENT` §1030 | **`system/content`** | content | no — owner == runner | ✅ |
| `EXTENSION-HISTORY` §419 | `system/tree` | **history** | **yes** | ✅ owning |
| `EXTENSION-COMPUTE` §847 | `system/tree` | **compute** | **yes** | ✅ owning |
| `EXTENSION-COMPUTE` §1391 | `system/tree` | **compute** | **yes** | ✅ owning |
| `EXTENSION-SUBSCRIPTION` §420 | `system/tree` | **subscription** | **yes** | ✅ owning |
| `EXTENSION-QUERY` §601 | `system/query` | **query** | **yes** | ⛔ **`E1`** |
| `EXTENSION-QUERY` §607 | `system/query` | **query** | **yes** | ⛔ **`E1`** |

**Four omit it** — `EXTENSION-COMPUTE` §812, §827, §831 and `EXTENSION-TRANSACTION` §324 — ⛔ **`E2`**,
plus four normative prose statements carrying the same short form (§4).

**On the discriminating rows — where the checking handler is not the owning handler — the corpus is
4 for the rule and 2 against**, three handlers to one.

> ⚠ **`EXTENSION-CONTENT` §1030 passes `system/content` and is correct.** The access lands in the
> content handler's own namespace, so owner and runner coincide. It is listed because every other
> compliant row reads `system/tree`, and a reader sweeping for deviations will otherwise conclude
> that `system/tree` is the only legal frame — which the rule does not say, and which would make a
> conformant site look like a defect indefinitely.

**The six ✅ rows are listed on purpose.** They were verified correct, not left unexamined. A sweep
that lists only its edits cannot be read as complete.

---

## 3. `E1` — `EXTENSION-QUERY` §5.2 passes the query handler for a tree read

**Today**, §5.2 step 6b, in both the tree-scope and content-store arms:

```
      if not check_path_permission("get", candidate.path, ctx.capability,
                                    "system/query", ctx.local_peer_id): continue
```

The subject is a path in the entity tree and the operation being authorized is a tree read. Under §1
the frame is `system/tree`.

**Fold:** `"system/query"` → `"system/tree"`, both call sites.

**This is a live divergence between conformant implementations**, and it is invisible to the existing
check set because the arm that exercises it names both handlers in a single grant — a capability that
cannot distinguish a correct frame from a wrong one.

> ⚠ **§5.5.2's prose has been read as settling this, and it does not.** The sentence is *"Capability
> filtering uses `check_path_permission` per result — same pattern as tree listing."* That is a
> statement about the filtering **approach**; it reads as normative about the *argument* only because
> the word *pattern* collides with the parameter name `handler_pattern`. **Fold:** reword to
> *"filtered per result, as tree listing is"*, removing the collision.

---

## 4. `E2` — eight sites omit the frame, and half of them are normative prose

| site | kind |
|---|---|
| `EXTENSION-COMPUTE` §812, §827, §831 | pseudocode |
| `EXTENSION-TRANSACTION` §324 | pseudocode |
| `EXTENSION-COMPUTE` §258 | normative prose |
| `EXTENSION-COMPUTE` §2178 | normative prose — **`MUST`** |
| `EXTENSION-COMPUTE` §2196 | normative prose — **`MUST`** |
| `EXTENSION-REVISION` §3966 | normative prose — **`MUST`** |

**Fold:** all eight gain `"system/tree", ctx.local_peer_id`.

⛔ **Three of these are `MUST` sentences, and the moment the rule folds they become MUSTs that mandate
a call the rule forbids.** Verbatim, `EXTENSION-COMPUTE` §2178: *"The compute handler **MUST** verify
the caller's capability … using `check_path_permission("put", path, capability)`"* — three arguments,
no frame. Folding the rule without these leaves the corpus self-contradicting at `MUST` level in two
documents.

**The omission is not cosmetic, and its cost has been measured in a shipped implementation.** An
implementer reconciling two shapes in one section must decide what the shorter one means, and
*"no handler filter"* is the reading that makes it work. Under that reading a grant scoped to any
handler at all authorizes a compute tree read. A specification that writes the same call two ways has
already made the choice for the reader.

Where an authority genuinely grants handler access with no path component, the `{include: []}`
construction in `ENTITY-CORE-PROTOCOL` §5.2 states that **at the grant**, where it is auditable —
not at the call site, where it is invisible.

> ⭐ **Why the first version of this document found three of these eight, and the general lesson.**
> The first sweep looked for the call as a **code block** and found every pseudocode site; it missed
> every **prose** site, including the three MUSTs. The same obligation is written as pseudocode
> *where it is implemented* and as prose *where it is obliged*, and a fold that corrects one leaves
> the other contradicting it. **A corpus sweep must run over both moods.**

---

## 5. Enforcement point

Neither the conformance-inventory check nor the citation checker reaches pseudocode arguments, so
this rule needs its own check.

> ⚠ **An earlier version of this document proposed a one-line grep here and it does not work.** It
> was measured at **13 hits against a claimed 2**, for three independent reasons, and all three are
> worth recording because each is a general trap: it scanned **per line** over a construct that
> **wraps** (four correct sites would have fired forever, including after this fold); it ran over one
> repository while its stated postcondition described two; and it hard-coded `system/tree` as the
> only legal frame, which §2 shows is false — it would have accused `EXTENSION-CONTENT` §1030
> permanently. **A check whose stated output is 2 and whose real output is 9 gets switched off.**
>
> *Its noise was signal, though: the false positives were exactly the set of sites this document was
> missing. The broken check found §4's omissions before a human did.*

**The replacement is a linter rule, not a grep.** It joins continuation lines and recovers the line
number from the match offset; classifies each hit as *definition · parameterized call · concrete call
· prose statement* and scores the last two; and for a concrete call compares the frame against the
**owning handler of the operation** rather than a literal, so the owner-equals-runner case is
conformant by construction rather than by exemption. It scans both the extension corpus and the core
specification, since the definition and the call sites are in different documents. Every silence case
is asserted in its tests rather than left as an untested exemption.

**The enumerated site list in §2 and §4 is generated by that rule, not transcribed** — a count
published without the surface it ranges over cannot be checked by inspection, which is how the first
version of this document got three of its four counts wrong.

---

## 6. Not in this document

- **The `included` map-key binding** — a separate, security-class correction to the core protocol
  with no extension-tier component.
- **Which `system/revision` operations are path-authorized.** A confirmed gap: `EXTENSION-REVISION`
  carries one `check_path_permission` sentence and never enumerates its nineteen operations.
  **Separate proposal.** *(This document does touch that sentence — §4 adds the missing frame
  argument to it — but does not enumerate the operations.)*
- **Which handlers must bind the per-entry listing filter.** Ruled by a core-protocol revision
  already in draft.
- **Test vectors.** Per `GUIDE-EXTENSION-DEVELOPMENT` §7, test vectors are an output of
  implementations converging, not an input this corpus authors. The load-bearing requirement is that
  an arm needs a **split capability** — one grant naming the dispatching handler, a second naming the
  owning handler — driven at **every** site in §2 and §4, not a named subset: the first version of
  this document named four, which omitted two sites including one of the frame-omitting ones.
