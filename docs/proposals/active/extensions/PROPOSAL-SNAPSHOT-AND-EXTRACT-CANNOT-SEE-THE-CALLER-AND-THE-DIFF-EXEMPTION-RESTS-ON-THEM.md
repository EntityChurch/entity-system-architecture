# PROPOSAL — `snapshot` and `extract` cannot see the caller, and the `diff` exemption rests on them

**Status:** **DRAFT 2026-09-12.** Target: `specs/extensions/EXTENSION-TREE.md` v4.9 → **v4.10**.
Sections: §3.3 (`compute_snapshot`), §6.2 (`execute_extract`), §8.2, §11, §12.1.

**Companion:** the core-protocol proposal *the arm we scored clean is the one with the hole, and the
authority table flattened an intersection* — its `J2` is the rule this proposal applies at the
extension tier. The two are one change and should land together.

## Provenance

Three independent implementations of this specification reported, within one week:

1. `snapshot` commits to unfiltered bindings, so `snapshot` + `diff`-against-empty discloses the
   excluded keys and their content hashes through the operation §11 exempts from path checks.
2. `extract` returns the excluded binding's entity in its envelope — both under a full-prefix request
   and under one naming the excluded path explicitly in `paths[]`.
3. A reader citing §8.2 for the default-tree listing filter concludes the obligation does not bind a
   peer that does not implement view trees.

**Each of those implementations then fixed its own code against text that does not ask for it.** That
is the finding. The reports frame item 1 as *"worth an explicit sentence."* Measured at the line, the
landed algorithms **specify the unfiltered behaviour**, and no sentence is being omitted — two of them
are being contradicted.

---

## 1. The signatures are the tell

The three bulk operations, as landed:

```
execute_merge(params, target_tree, capability, local_peer_id)      ; §5.4
compute_snapshot(tree, prefix)                                    ; §3.3
execute_extract(tree, content_store, prefix, paths)               ; §6.2
```

**`merge` takes a capability. `snapshot` and `extract` do not.** An operation with no access to the
caller's authority cannot filter by it, and neither algorithm tries: `compute_snapshot` iterates
`for (path, hash) in tree: if path starts with prefix`, and `execute_extract` does the same in its
full-prefix branch and reads each named path directly in its `paths[]` branch. There is no
authorization step in either, anywhere.

`merge`, by contrast, carries the per-path check in its body:

```
    if check_path_permission("put", target_path, capability, "system/tree", local_peer_id) == DENY:
      return error("capability_denied", 403)
```

**So the write side of this extension filters per path and the read side does not** — which is the
read carve-out `ENTITY-CORE-PROTOCOL` §6.8 ruled closed four days ago, still in force in the two
algorithms that embody it.

## 2. §11's table is the instruction implementers followed

| Operation | Path Permission | Scope |
|---|---|---|
| `snapshot` | `get` | **Snapshot prefix** |
| `extract` | `get` | **Extract prefix** |
| `diff` | — | Operates on stored snapshots; no path-level check |

*"Snapshot prefix"* is a complete and unambiguous instruction to check **one** path — the prefix — and
return everything under it. Two independent implementations built exactly that. **This is not an
implementation gap against landed text; it is conformance to landed text.**

## 3. The composition, which is why it is not merely an over-broad read

§11 exempts `diff` from path checks, and §4.2's prose repeats it: *"Diff operates on stored snapshots
— no path-level authorization."* That exemption is sound **only** while a snapshot cannot commit to
bindings the caller may not see. It cannot be evaluated locally, in `diff`'s own section, because its
soundness is a property of a **different operation**:

```
snapshot(prefix)  under a cap excluding prefix/secret   -> snapshot commits to prefix/secret
diff(that, empty)                                       -> `added` names prefix/secret + content hash
```

Both steps are authorized. Neither operation is individually wrong under its own section. **The
excluded key and its content hash cross to the caller through the operation the spec exempted from
checking.**

**The general shape, and it is the durable half of this proposal:** *a path-check exemption is a claim
about its upstream producer.* An exemption granted because *"this operation touches no tree"* is
sound exactly as far as the thing it consumes was filtered — and the two live in different sections,
so neither section can see the composition. **Every exemption in this corpus should name the producer
it depends on.**

## 4. §8.2 is the wrong citation and it is a reasonable mistake

`EXTENSION-TREE` §8.2's read pseudocode is the corpus's most legible statement of per-entry
filtering, and its §8.5 `scope` is compiled from the request capability — so it is **not** the
vacuous reading, and a seat that cites it has not misunderstood the security model.

What it is, is **scoped**. §12.2 lists *"View trees for handler isolation (§8)"* under **SHOULD**, with
the information-hiding rule a MUST **only when view trees are implemented**. A peer that does not
implement view trees reads §8 as not binding on it — correctly — and concludes it owes no listing
filter. That peer is the leaking peer, and it got there by reading the document properly.

**The unconditional home already exists and is core §6.3's Listing filter** — three MUSTs, no view-tree
predicate, and `count` and pagination pinned with it. §8.2 needs one sentence pointing at it.

*(Note the residue: core §6.3's listing filter covers `system/tree:get` on a trailing-slash path. It
does **not** cover `snapshot` or `extract`, which are this extension's operations. So §1's gap has no
unconditional home anywhere, in either document — which is why §5's delta puts it in the algorithms.)*

---

## 5. Deltas

**5.1 §3.3 `compute_snapshot`** — the signature takes the authority, and the binding loop filters:

```
compute_snapshot(tree, prefix, authority, local_peer_id):
  ...
  for (path, hash) in tree:
    if path starts with prefix:
      ; A snapshot commits only to bindings the caller is authorized to `get`
      ; (ENTITY-CORE-PROTOCOL §6.3). The binding is DERIVED — the caller named
      ; the prefix, not the key — and its existence and content hash reach the
      ; caller, so the caller's capability bounds it (§6.8 row 1).
      if check_path_permission("get", path, authority, "system/tree", local_peer_id) == DENY:
        continue
      relative = path[len(prefix):]
      bindings.append((relative, hash))
```

**Omitted, never refused.** An unauthorized binding is absent from the snapshot, exactly as an
out-of-scope read is `not_found` (§8.2's information-hiding rule) — a `403` on a prefix snapshot would
itself disclose that something hidden exists under the prefix.

**5.2 §6.2 `execute_extract`** — both branches:

- full-prefix branch: the same `check_path_permission` filter as §3.3, `continue` on DENY.
- `paths[]` branch: after the existing per-entry `is_valid_relative_path` admission, each resolved
  `prefix + path` is checked. **A caller naming an excluded path explicitly is still denied it** —
  and it is **omitted, not `403`**, for the reason above and because §6.2 already specifies that *"a
  well-formed path that binds nothing is ABSENT and is silently omitted."* Unauthorized and unbound
  are indistinguishable to the caller by design.

**The completeness MUST is unaffected and this is worth stating**, because it looks like it should be:
§6.2's *"an extract's envelope is complete against its own root"* holds because `build_trie` is built
over **precisely the selected bindings**. Authorization filtering selects a smaller binding set before
`build_trie`, which is the same operation a `paths` filter performs. A filtered extract remains a
**smaller trie, not a partial view of a larger one**.

**5.3 §11's table:**

| Operation | Path Permission | Scope |
|---|---|---|
| `snapshot` | `get` | **Each binding captured** — the prefix alone is not the subject |
| `extract` | `get` | **Each binding included** — including paths the caller named explicitly in `paths[]` |
| `diff` | — | Operates on stored snapshots; no path-level check. **The exemption depends on §3.3: a snapshot commits only to bindings the caller may `get`.** An unfiltered snapshot moves the disclosure into this operation |
| `merge` | `put` | Each target path (after remapping) |

**5.4 §8.2** — one sentence after the read pseudocode:

> **This section governs compiled view trees only.** The listing filter for the **default** tree is the
> unconditional rule at `ENTITY-CORE-PROTOCOL` §6.3 — it binds every peer, including one that does not
> implement view trees (§12.2). The pseudocode here is the same filter expressed against a compiled
> scope; it is not the authority for it.

**5.5 §12.1 MUST** — three rows:

- **Bulk-read filtering (§3.3, §6.2)** — `snapshot` and `extract` capture only bindings the caller is
  authorized to `get`. An unauthorized binding is **omitted**, never refused.
- **The `diff` exemption is conditional (§11)** — `diff`'s freedom from path checks is sound only
  against a filtered snapshot. A peer that exempts `diff` and does not filter `snapshot` is
  non-conformant on §3.3, and the observable is `diff`.
- **Default-tree listing filter (§8.2)** — the obligation is `ENTITY-CORE-PROTOCOL` §6.3 and binds
  independently of view-tree support.

---

## 6. Check-set requirements

Stated as requirements, not vectors (`GUIDE-EXTENSION-DEVELOPMENT` §7 Stage 0). **Every one needs a
narrow cap** — a cap scoped to a subtree or carrying an exclude. The existing `tree_operations` arms
drive the broad connection cap, under which the filtered and unfiltered answers are identical, which
is why three green categories never witnessed any of this.

1. **`snapshot` + `diff`-against-empty** — snapshot a prefix under a cap excluding one child; diff
   against an empty snapshot. The excluded key **and its content hash** MUST NOT appear in `added`.
   This is the composition vector; a snapshot-only assertion cannot reach it, because the leak is in
   the exempt operation.
2. **`extract` per-entry, two arms** — full-prefix, and `paths[]` naming the excluded binding
   **explicitly**. Neither envelope may carry the entity. The second arm is the one that fails on an
   implementation that filters by prefix containment.
3. **Listing `count` under `limit: 1`** — the excluded child MUST be omitted and `count` MUST be the
   filtered count. Drive `limit: 1` specifically: the excluded child was measured returning *under*
   `limit: 1`, so the leak is ordered adversely rather than being a tail effect.
4. **A control arm on each** — the same operation under a cap that grants the child, asserting it
   **is** returned. Without it, an implementation that returns nothing scores green on all three.

**All four are unobserved at the moment of landing, and none may be recorded as covered on the
strength of the three implementations agreeing.** Independent implementations converging is evidence
of attention, not of correctness — a set of peers that all pass one author's reading has demonstrated
consistency with each other, which is a weaker claim than a driven check.
