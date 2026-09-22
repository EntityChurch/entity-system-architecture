# PROPOSAL — a wildcard is a statement about scope, not consent to audit the machinery

**Status:** **IMPLEMENTED (2026-09-08)** — folded as `EXTENSION-HISTORY` v1.10 the same session. **§6's open question is NOT closed by the fold** and is deliberately still open: this proposal is correct under either answer, which is why it could land before the question was settled.
**Target:** `EXTENSION-HISTORY` §2.2 (a `pattern_exclude` field) and §6.3 (the worked config that
everyone copies). **Not** §3.2, and §4 says why that is a reversal of the previously recorded lean.
**Raised by:** a seat building a peer from the specs, which measured the effect on a real loopback.

---

## 1. The finding

`EXTENSION-HISTORY` §6.3 shows, as *the* worked configuration for a peer that wants history:

```
pattern: "*"              ; match everything
enabled: true
max_depth: 10000
```

**On a real loopback, one `system/tree:put` produced three consumer events.** Two of them were the
peer's own signature bookkeeping — writes the peer performs *while serving the request*, not writes
the caller asked for. Both fall inside `"*"`, and both are recorded as ordinary transitions.

**Nothing is corrupted.** Head pointers are per-path, and the measuring seat reported zero
regressions across 756 checks over two rounds. **What accrues is a transition per served request,
forever** — in a store where retention is a *floor* and pruning is collection-side, so nothing in the
specification obliges anyone to remove them.

**Scope of the measurement, and it is not to be generalized away:** three-events-per-request is
**one engine's number**. Two other implementations perform the same signature binding, so the shape
is expected to hold — **but it was measured on one.**

---

## 2. What the wildcard actually means, which is the whole argument

**`pattern: "*"` is a scope statement: *every path*. It is not a statement of intent to audit the
protocol's own machinery.** A peer's signature binding is not something the peer *did*; it is *how
it did* the thing the caller asked for. Recording it produces an audit log in which the mechanism
outnumbers the events two to one.

**But it is not always noise, and that is why the fix is not a prohibition.** A deployment auditing
capability issuance has an excellent reason to want grant writes in history. **The distinction is
consent:** a wildcard did not ask for them; an explicit pattern did.

So the rule this proposal wants is:

> **A wildcard covers what the peer was asked to do. Naming a machinery path explicitly is how you
> ask for the rest.**

---

## 3. The design space, and the constraint that eliminates two of the three

### 3.1 Widen §3.2's recursion guard — **rejected**

§3.2 makes writing to the local peer's `system/history/head*` paths a `MUST NOT` record. Widening
it to *"local, engine-written protocol paths, set enumerated"* is one small edit and changes no
config surface. **Two objections, and the second is decisive:**

- **It conflates two rationales.** §3.2 exists to stop *infinite recursion* — history recording its
  own head pointers re-triggering the recorder. Signature writes create no recursion whatsoever.
  They create **volume**. Putting a volume rule inside a clause whose stated reason is recursion
  means the next person to read the clause cannot tell which constraint they are allowed to relax.
- **It is a hard `MUST NOT`, so it forecloses the audit a deployment may legitimately want.** §2's
  whole point is that these writes are not noise *by nature*, only noise *by default*.

### 3.2 A normative default exclusion set that `*` does not cover — **rejected, and the reason is not obvious**

Attractive: it fixes the problem for everyone with no new config surface, and leaves an explicit
pattern able to reach the excluded paths.

**It cannot be written correctly, because the paths are not ours to pin.** `ENTITY-CORE-PROTOCOL`
§1.9 states, in terms, that the system path locations — `system/capability/grants/*`,
`system/peer/*`, `system/transport/*`, `system/connection/*` — are a **recommended convention and
not a structural requirement**; peers that diverge remain conformant and communicate their actual
paths through grants. **A normative default exclusion list keyed on those prefixes silently pins a
convention the core deliberately left open**, and it is wrong on exactly the conformant peer that
took the core at its word.

**This is the load-bearing constraint in the whole question, and it is easy to miss** — it lives in
a core section about path organization, four documents away from the extension that would have
pinned it.

### 3.3 An emit-site marker — **right, and not proposable from here**

The correct long-run shape: the writer marks the write. A tree mutation performed by the peer's own
protocol machinery is flagged as such at the emit site, and history matches on the flag rather than
on a path. **No convention is pinned, no deployment has to re-derive a list, and every extension
consuming the emit pathway gets the distinction for free.**

**It is not proposable from here today, for a reason worth stating rather than hiding:** it needs a
new execution-context field, which is `SYSTEM-COMPOSITION`'s to define and reaches every emit
consumer — and **arch cannot establish that the distinction is even available at the emit site.**
§5.1 records `handler_pattern` and `operation` from the execution context, and a signature write
performed inside a `system/tree:put` cascade plausibly inherits the caller's context verbatim, in
which case there is nothing to read and the flag has to be *set* by the machinery rather than
*derived*. **That is an implementation fact, not a specification fact.** See §6.

### 3.4 `pattern_exclude` on the config — **proposed**

```
pattern_exclude: {array_of: {type_ref: "system/tree/path"}, optional: true}
             ; Paths matching any of these are not recorded, even when `pattern`
             ; matches. Same core §5.4 pattern syntax as `pattern`.
             ; Absent = no exclusions.
```

- **It pins nothing.** A deployment that stores its signatures somewhere else writes its own
  excludes, and the specification never had to guess.
- **The shape already exists in this corpus.** `EXTENSION-REVISION` carries `exclude` and
  `exclude_types` on its configuration for the same reason. This is not a new idea being introduced
  to solve one bug.
- **It uses core §5.4 patterns**, not a domain-local matcher — worth saying explicitly, because the
  one other extension with an `exclude` field defines its own glob for it and **declares that
  divergence in terms.** This field does not diverge, so it does not get a matcher.
- **Evaluation order is stated, because otherwise two conformant readings exist:** exclusion is
  checked **after** the most-specific matching config is selected and **before** the event-type
  filter. A path excluded by the selected config is not recorded, **and does not fall through to a
  less specific config** — an exclusion is a decision, not a failure to match.

---

## 4. This reverses the previously recorded lean, and the reason is the constraint in §3.2 above

The lean on file was **widen the §3.2 guard** — on the reasoning that §3.2 is already the
self-exclusion clause, already draws the narrow/wide line deliberately, and that
`pattern_exclude` *"asks every deployment to re-derive a set the spec knows."*

**The last clause is the part that turned out to be false. The spec does not know the set** — core
§1.9 says the paths are convention and conformant peers may differ. A rule that assumes a set the
core declined to fix is wrong on precisely the peer that exercised the freedom the core granted.

**The rest of the lean's reasoning stands and is why §3.4 above is scoped as narrowly as it is:** a
new config surface is a real cost, so it gets one optional field, the same pattern syntax, and a
stated evaluation order — and no default value, because a default is where the convention would
sneak back in.

---

## 5. The edits

**§2.2** — the `pattern_exclude` field, with the evaluation-order sentence.

**§6.3 — the worked configuration, which is the half that actually fixes anything.** No default set
is normative, but §6.3 is what people copy, and today what they copy is the configuration that
produced the finding. It becomes:

```
system/history/config/everything := {
  type: "system/history/config"
  data: {
    pattern: "*"
    pattern_exclude: [
      "system/signature/*",          ; signature bindings written while serving
      "system/capability/grants/*",  ; grant storage
      "system/peer/*"                ; peer and connection status
    ]
    enabled: true
    max_depth: 10000
  }
}
```

— **shown as an example for the recommended convention and labelled as one**, with the reason
stated: these are paths the peer writes as a side effect of serving, a wildcard did not ask for
them, and a deployment using different locations substitutes its own.

**§5.1's `emit_entity`** gains the exclusion check in the one place the config is already
consulted, immediately after `find_history_config`.

---

## 6. The open question this proposal cannot answer, and it is the one that decides §3.3

**Can a peer distinguish, at the emit site, a write it performed as its own protocol bookkeeping
from a write it performed on a caller's behalf — without consulting a path list?**

- **If yes**, §3.3's emit-site marker is the right long-run answer and this proposal is the
  interim, which is a fine thing for it to be.
- **If no**, a marker has to be *set* by every piece of machinery that writes during serving, which
  is a much larger change and probably not worth it — and `pattern_exclude` is the answer, not a
  stepping stone to one.

**Arch cannot answer this and should not guess.** It is a fact about how the emit pathway is
actually wired in a running peer, and the honest form of that sentence is *"three seats know and we
do not."* **This proposal is deliberately correct under either answer**, which is why it can land
before the question is settled.
