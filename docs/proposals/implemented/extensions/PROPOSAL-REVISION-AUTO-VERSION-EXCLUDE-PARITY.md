# PROPOSAL — auto-version adopts a root it did not filter, and `commit` builds one it did

**Status:** **RULED and FOLDED 2026-08-18 — the matcher is pinned (four closed forms, `**` rejected at config write). SA-PY-11 closed. Ready to move to `implemented/` on the operator's word.**
D1 §6.1 Algorithm (filtered root + Amendment-2 augmentation + the empty-exclude fast path) ·
D2 §6.1 prose · D3 the emission-gate clarification · D4 §2.4 · D5 vector. **Added at fold:**
**SA-PY-11** — `glob_match` was used three times and defined nowhere, and it is hash-determining
(§2.4 now pins doublestar; *both* stdlib defaults are non-conformant) — and **SA-PY-7**, which
`entity-core-py` measured as returning **disjoint** sets, so `log.since` → `log.start_at`.
`EXTENSION-REVISION` 3.11 → 3.12.
**Tier:** extensions — `EXTENSION-REVISION` §2.4, §6.1; cross-reference `EXTENSION-TREE` §3.4.1a.
**Answers:** `entity-core-py` SA-PY-8 (`ROUTING-2026-08-17-b`, re-routed `ROUTING-2026-08-18-b` §3 —
**the only one of their four open items that gates code**).
**Read at:** arch `1782d6d` · core-go `94df9e6` · core-py `961f5fe`

---

## §0 Summary of the ruling

**SA-PY-8 is confirmed, and it is larger than filed.** The ask was *"must a tracked structural root
used as a version root be exclude-filtered (§2.4 vs §6.1), or must §6.1 re-snapshot?"* The answer is
**exclude-filtered** — and establishing it turned up that **§6.1 contradicts itself**, independent of
any implementation.

| | |
|---|---|
| **The divergence is real and is the spec's** | `handle_commit` builds `build_trie(compute_versioned_bindings(tree, prefix, version_config))` — **filtered**. `auto_version_on_write` assigns `root = current_tracked_root(prefix)` — **unfiltered**. Same tree state, two roots, whenever any exclude matches. |
| **`entity-core-go` is conformant to the text** | Applying excludes at `commit` and not at auto-version is **exactly what §2.4 and §6.1 say**. py's report that go "applies them at commit and not at auto-version" describes a peer following the spec. |
| **`entity-core-py` is non-conformant at `commit`** | Applying excludes nowhere misses §2.4's *"Exclude applies to trie building"* MUST at the `commit` path. That half is a straightforward impl fix. |
| **The version DAG is the casualty** | A version entry is `{root, parents}` and is content-addressed. Two roots for one tree state = **two version identities with no content difference** — a DAG fork that entity exchange cannot converge, on a spec whose §6.1 opening line is *"the version DAG converges naturally across peers."* |
| **§6.1's own Amendment-2 MUST is unsatisfiable as written** | Deletion-marker emission requires auto-version to **build** a trie; the algorithm **adopts** one. §2 |

---

## §1 The two paths, quoted

**`commit` — filtered.** `EXTENSION-REVISION` §4.4, `handle_commit`:

```
  ; Build trie from current tree state at prefix.
  ; Apply exclude filters from version configuration if present.
  version_config = find_version_config(prefix)
  if version_config is not null:
    trie_root_hash = build_trie(compute_versioned_bindings(tree, prefix, version_config))
  else:
    trie_root_hash = build_trie(compute_bindings(tree, prefix))
```

**auto-version — unfiltered.** §6.1, `auto_version_on_write`:

```
auto_version_on_write(event, prefix, version_config):
  if event.path matches version_config.exclude:
    return

  root = current_tracked_root(prefix)          ; from system/tree/root/{prefix}
  …
  version = { type: "system/revision/entry",
              data: { root: root, parents: … } }
```

**The exclude appears in both, doing two different jobs.** In `commit` it filters *what goes into the
root*. In auto-version it gates *whether an entry is emitted at all* — and the root the entry then
carries is taken wholesale from `system/tree/root/{prefix}`.

### The tracked root cannot be filtered, and that is settled in `EXTENSION-TREE`

`system/tree/root/{P}` is produced by the **structural summary consumer** from a
`system/tree/tracking-config` (`EXTENSION-TREE` §3.4.1a). That config's fields are the tree
extension's own; **it has no knowledge of any revision `exclude` or `exclude_types`.** TREE §3.4.1a
is explicit that the binding is *"a direct pointer: its value is the content_hash of the root trie
node entity."* There is no filtering stage between the tree's trie and revision's adoption of it.

So `current_tracked_root(prefix)` is definitionally a trie over **every** binding under the prefix.
§2.4's *"Exclude applies to trie building"* and §6.1's use of the tracked root are not two readings of
one rule — they are two different roots.

### What the divergence costs, concretely

Take `exclude: ["ephemeral/**"]` (§2.4's own worked example) on prefix `project/`:

| event | `commit` path | auto-version path |
|---|---|---|
| write `project/src/a` | root over `{src/a}` | root over `{src/a}` — **agree** |
| write `project/ephemeral/tmp` | *(no commit)* | **suppressed** — no entry emitted |
| write `project/src/b` | root over `{src/a, src/b}` | root over `{src/a, src/b, ephemeral/tmp}` — **diverge** |

The second row is the trap: suppressing the *entry* does not suppress the excluded path's
*contribution to the tracked root*, so the very next non-excluded write emits a version whose root
commits to data the config says is not versioned. **The exclude is not merely inconsistent across
paths — on the auto-version path it does not work at all.** A peer running auto-version over a config
with excludes is versioning the excluded paths, one write late.

---

## §2 §6.1 contradicts itself, and this is the part that settles the ruling

§6.1, **Deletion-marker emission at commit (v3.1, Amendment 2)**:

> When auto-version (**or** explicit `commit`) emits a new version, the new version's trie MUST
> include explicit entries for every path that was bound in the parent version's trie. … For paths
> bound in parent but unbound in current live state … the entry binds the path to
> `CANONICAL_DELETION_MARKER_HASH`.

**An algorithm that assigns `root = current_tracked_root(prefix)` cannot satisfy this.** It never
constructs a trie, so there is no step at which marker-augmentation can happen; the tracked root is
whatever the tree extension last wrote, and the tree extension knows nothing about the parent
version's key set. The Amendment-2 paragraph and the Algorithm block in the same section describe
two incompatible auto-version implementations.

The v3.3 D3 refinement makes it sharper still — it requires marker-augmentation to run **before** the
dedup-against-prior-head check, and the dedup step it names (`if current_version.data.root == root:
return`) is present in the Algorithm block while the augmentation step it must precede is not.

**Conclusion: the Algorithm block is the stale half.** Amendment 2 (v3.1) and D3 (v3.3) both presume
auto-version builds a trie; the pseudocode was never updated to match. Ruling for the filtered root
is therefore not a new constraint — it is the reading the rest of §6.1 already requires.

---

## §3 The ruling

**A version entry's `root` MUST be the exclude-filtered trie over the tracked prefix, on every path
that emits one — auto-version and `commit` alike.** The tracked structural root is an *input* to that
computation, not a substitute for it.

**The fast path is preserved exactly, and this is why the ruling costs nothing in the common case.**
When a config's `exclude` and `exclude_types` are both empty or absent, the filtered trie and the
tracked structural root are **definitionally the same trie over the same binding set**, so an
implementation MAY adopt `current_tracked_root(prefix)` directly. The O(1) adoption that motivated
the current pseudocode remains available for every config that does not filter — which §2.4 notes is
the common application-prefix case, *"self-excluding by construction."*

**Where excludes are non-empty**, the implementation MUST compute the filtered trie. §6.1's own
*Efficient diff path* already supplies the mechanism and the cost bound — trie-diff against the
parent version root, *"O(changes × depth), not O(total paths in scope)"* — so the filtered path is
not a re-scan of the prefix either.

**Rejected alternative: make `commit` stop filtering, so both paths adopt the tracked root.** This
would give parity at the cost of deleting the exclude feature — `ephemeral/**`, `**/*.cache`,
`exclude_types`, and the §6.1 Reentrancy MUST list (`system/revision/**`, `system/tree/root/**`,
`system/history/**`, `system/clock/**`) all exist to keep paths *out of versioned bindings*. Under
universal-tree versioning (prefix `"/"`) that alternative reinstates the self-feeding loop TREE
§3.4.1a and §2.4's *"Trie root exclude"* were both written to prevent. Parity must be reached by
filtering both, never by filtering neither.

---

## §4 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-REVISION.md` | §6.1, Algorithm block | Replace `root = current_tracked_root(prefix)` with the filtered computation, mirroring `handle_commit`: build the versioned bindings under `version_config` and `build_trie` them, then run the Amendment-2 marker-augmentation, **then** the existing dedup-against-head check (preserving the v3.3 D3 ordering). Add the fast-path clause: when `version_config.exclude` and `.exclude_types` are both empty/absent, the result is identical to `current_tracked_root(prefix)` and an implementation MAY adopt it directly. |
| **D2** | `EXTENSION-REVISION.md` | §6.1, prose above the Algorithm | One paragraph stating the invariant plainly: **a version entry's `root` is exclude-filtered regardless of which path emitted it**, because the version DAG's identity is its roots and a root that depends on the arrival path forks the DAG with no content difference. Name the failure: two peers over identical tree state and identical config produce different `system/revision/entry` hashes, and entity exchange cannot converge them. |
| **D3** | `EXTENSION-REVISION.md` | §6.1, `if event.path matches version_config.exclude: return` | Keep the emission gate, and add the clarifying sentence that it is **not** the filter: it suppresses emitting a version *for an excluded write*, and the filtering of the emitted root is D1's computation. As written the two read as one mechanism, which is the mis-reading that produced this divergence. |
| **D4** | `EXTENSION-REVISION.md` | §2.4, *"Exclude applies to trie building"* | Broaden the sentence, which currently scopes itself to `commit` by name (*"The `compute_versioned_bindings` call during `commit` applies the exclude filters"*). It applies to every version-root computation. Cross-reference §6.1. |
| **D5** | `EXTENSION-REVISION.md` | §11 conformance | One vector: **`REV-AUTOVERSION-EXCLUDE-PARITY-1`** — a prefix configured with a non-empty `exclude`; write an excluded path, then a non-excluded path; the emitted auto-version entry's `root` MUST equal the `root` an explicit `commit` produces over the same live state. This is the two-peer divergence reduced to a single-peer check, and **no current vector reaches it** — which is why three implementations disagreed silently. |

**Not in scope:** the `exclude_types` semantics · `EXTENSION-TREE`'s tracking-config (unchanged — the
tracked root stays unfiltered, and that is correct for its own consumers) · nested-prefix
coordination · the Reentrancy MUST list · any change to version-entry structure.

---

## §5 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-py`** | **The `commit`-path filter** (§2.4's existing MUST — applying excludes nowhere is non-conformant today, independent of this proposal) **and D1.** This is the item they flagged as gating code; both halves are now specified. |
| **`entity-core-go`** | **D1 only.** Their `commit` path is already conformant and their auto-version path already follows the current text — **no fault, and the change is a spec revision reaching them, not a defect report.** |
| **`entity-core-rust`** | Unmeasured on this surface. Read both paths and report before implementing. |
| **oracle / keystone** | D5's vector. **No re-pin** — adds a case, changes no encoding. |

---

## §6 What this proposal does NOT claim

- **Not** that any implementation is currently forking a DAG in production. The divergence is
  structural and reachable; whether any deployed config carries a non-empty `exclude` under
  auto-version is unmeasured and is not asserted.
- **Not** that `entity-core-rust` has either half. Unmeasured; routed as a read-and-report.
- **Not** that the tracked structural root is wrong. It is correct for TREE's own consumers; the
  defect is revision adopting it as a version root without the filter its own §2.4 requires.


---

## 9. SA-PY-11 withdrawn — the doublestar pin contradicted core §5.4

**Withdrawn at the point of claim, 2026-08-18, by operator ruling.** §2.4's normative matcher table —
*"`*` stays inside a segment, `**` crosses"*, pinned to doublestar — has been removed from
`EXTENSION-REVISION`. It was wrong, and this proposal is reopened to settle what replaces it.

### 9.1 What the pin got wrong

**The protocol already had a pattern rule.** `ENTITY-CORE-PROTOCOL.md` §5.4 `matches_pattern`:

```
; Subtree: pattern/* — prefix match
if pattern ends with "/*":
  prefix = pattern without trailing "*"
  return path starts with prefix
```

`path starts with prefix` — **`*` crosses `/`**. §5.4's own pattern table reads `pattern/*` →
*"local peer, **subtree**"*, and the only segment-scoped wildcard in the core is the `/*/` peer-id
strip. The core defines `**` nowhere. `entity-core-go` independently confirmed the same `HasPrefix`
rule in a second core matcher, subscription's engine — two core matchers plus §5.4 on one side.

**The stated justification for `**` was circular.** The pin argued that §6.1's Reentrancy MUST needs to
reach `system/revision/head/{prefix_hash}/…` several segments deep, which `path.Match` cannot do. Under
§5.4, `system/revision/*` reaches it — it is a subtree match. **The Reentrancy MUST was satisfiable with
one star and always has been.** `**` became necessary only after a foreign matcher was adopted; it was
buying back what §5.4's `*` already did.

**The authority cited was outside the system.** The pin was argued from Go's `path.Match` and Python's
`fnmatch`. Entity paths are not a filesystem and the tree is not POSIX. No outside standard governs here.

### 9.2 What survives

`entity-core-py`'s underlying finding stands and is still the reason this section exists: **`glob_match`
is used three times in `EXTENSION-REVISION` and defined nowhere**, and it is **hash-determining** — the
matcher decides trie membership, membership decides the version `root`, and `root` is the version
entry's identity. Two peers on different readings produce different version hashes for identical content
with no error anywhere. That defect is real and unfixed. Only the *resolution* was wrong.

`entity-core-py`'s `fnmatch` exposure is likewise still real, but its severity is re-stated: under §5.4
a `*` that crosses `/` is **correct behavior**, not the over-reach the withdrawn pin called it. What
`fnmatch` actually gets wrong against §5.4 is narrower and needs its own measurement.

### 9.3 The open question this proposal now carries

§5.4's vocabulary — exact · `pattern/*` subtree · bare `*` · `/*/` peer strip — **cannot express matching
by filename or extension at arbitrary depth**, in any notation. `exclude`'s `**/*.cache` form wants
exactly that, and it is a narrow, revision-local need: `exclude` is an ignore-list over a versioned
prefix, not a capability scope.

**Operator ruling on the shape of the answer:** an extension MAY define a matcher for its own
domain-specific need. It MAY NOT present that matcher as the protocol's pattern rule, and it MAY NOT give
an existing core token a second meaning. So:

1. **Settled, no longer in scope for this proposal.** The general glob rule is §5.4. Every `*` in the
   corpus outside REVISION's two exclude fields is §5.4's `*`. The prefix-style excludes
   (`system/revision/**`, `system/tree/root/**`, `system/history/**`, `system/clock/**`,
   `system/inbox/**`, and the rest) are **`pattern/*` under §5.4** and the `**` spelling adds nothing —
   they are pure redundancy and should be respelled.
2. **Open — D6.** Whether the depth-suffix form (`**/*.cache`) gets a spelling that **cannot be confused
   with §5.4 syntax**, and what that spelling is. Reusing `*`/`**` gives one syntax two meanings in one
   system, on a hash-determining surface. A distinct token or a typed field (e.g. an explicit
   `exclude_suffixes` list) avoids the collision entirely and is the shape to price first.
3. **Open — D7.** The corpus sweep. `**` tokens entered at `70c52b4` (v0.8.0 initial public release) in
   `EXTENSION-TYPE` (*"`*` matches any single path segment, `**` matches zero or more"* — a direct
   contradiction of §5.4), `EXTENSION-REGISTRY` (*"POSIX shell-glob"*), and REVISION's examples. Seven
   spec files carry `/**` patterns. Each needs classifying as redundant-respell or genuine-domain-need.

### 9.4 Cohort impact

**All three seats: stop.** No implementation work on the exclude matcher until D6 lands. Nothing built
against the withdrawn §2.4 table is conformant — including a doublestar dependency added on its
authority. `entity-core-py`'s `fnmatch` fix is **not** to be re-targeted at doublestar; hold at the
current behavior and await D6.

The two non-glob items from this fold are **unaffected and stand**: the D1–D5 filtered-root deltas, and
the `log.since` → `log.start_at` rename (SA-PY-7).


---

## 10. Operator ruling — `**` is not adopted, and the corpus is reconciled

**Ruled 2026-08-18. Folded the same session. This closes D7 and re-scopes D6.**

### 10.1 The ruling

> `**` was never adopted by this protocol. It is not in the core, it was never reserved, and it is not
> ours. The **only** place it exists is `EXTENSION-REVISION`'s exclusions — **a domain-specific matcher for
> a domain-specific application** — and that stays. Everything else that picked it up gets cleaned. If a
> use case ever proves the general need, it gets adopted deliberately; that has not happened.

The load-bearing distinction: **an extension MAY define a matcher for its own domain need. It MAY NOT
adopt, on the corpus's behalf, a token the protocol does not have.** REVISION's `exclude` is the first;
the eight documents below were the second.

### 10.2 What was found — `**` had spread to eight documents outside REVISION

`GUIDE-CAPABILITIES` §455 is the proof it carried no meaning: one sentence writes
`system/capability/**` and `system/capability/*` **for the same thing**. It was a synonym for the subtree
form, not a distinct semantic — which is exactly what a token with no definition becomes.

| Document | Occurrences | Disposition |
|---|---|---|
| `GUIDE-CAPABILITIES` | 6 | → `/*` |
| `EXTENSION-INBOX` | 4 | → `/*` |
| `EXTENSION-CONTINUATION` | 4 | → `/*` |
| `EXTENSION-TREE` | 4 | → `/*` |
| `GUIDE-INSPECTABILITY` | 3 | → `/*` |
| `SPECIFICATION-FORMAT` | 2 | → `/*` |
| `SYSTEM-COMPOSITION` | 1 | → `/*` |
| `EXTENSION-SUBSCRIPTION` | 1 | → `/*` |
| **Total** | **25** | **all now the §5.4 subtree form** |

Verified zero `/**` remain outside `EXTENSION-REVISION` and its two guides.

### 10.3 The three definition sites, corrected

1. **`EXTENSION-TYPE`** — its glob paragraph *defined* segment semantics (*"`*` matches any single path
   segment, `**` matches zero or more"*) and its example asserted `system/capability/*` does **not** match
   `system/capability/path-scope/foo`. Under §5.4 it does. Now defers to `matches_pattern` explicitly, and
   states the one thing genuinely lost: **a depth-limited match must enumerate its types**, because §5.4
   carries no such form and approximating it with a wildcard is what produced this.
2. **`EXTENSION-REGISTRY`** — its *"POSIX shell-glob"* claim cited `BRIDGE-HTTP §4-RES.2` as the precedent
   that "already chose" that grammar. **`EXTENSION-BRIDGE-HTTP` exists in none of the 14 repos.** The POSIX
   framing is removed. What replaces it is narrower and true: this matches a **name string**, not a path,
   so it is a registry-local matcher with no segment structure to bind §5.4's forms to. Explicitly **not**
   §5.4, and carrying no `**`.
3. **`GUIDE-EXTENSION-DEVELOPMENT` §4.9 Rule 2** — the origin. Its reserved-token table listed `**` as
   *"Globstar (future use)"* attributed to *"V7 §1.4 / Document History v7.18"*. **§1.4 has no
   reserved-token table, and neither `v7.18` nor "globstar" appears anywhere in the core spec or in any of
   the 14 repos.** Its two sibling rows (`*`, `./`) are real, which is precisely why the invented one read
   as authoritative. Row removed; the table now cites §5.4/§6.6 correctly, and a note records that
   REVISION's use is domain-scoped and not to be generalized from.

### 10.4 Still open — D6, narrowed

**The exact semantics of `**` and `*` *inside* REVISION's `exclude` / `exclude_types` remain unpinned, and
this is not cosmetic:** the matcher decides trie membership → membership decides the version `root` →
`root` is the version entry's identity. Two peers reading these patterns differently produce **different
version hashes for identical content, with nothing failing anywhere.** That was `entity-core-py`'s original
SA-PY-11 finding and it is still correct; only the proposed resolution (adopt doublestar corpus-wide) was
wrong.

**Not ruled here, and not to be ruled at speed.** The domain is now one spec and two fields, which is the
right size for the question. **No implementation should change its matcher until it lands.**


---

## 11. D6 RULED — the matcher is four closed forms, and `**` is rejected

**Operator ruling. `**` does not justify itself anywhere, including here.**

### 11.1 The measurement that decided it

Every `*`-bearing pattern in `EXTENSION-REVISION` and its two guides — 40 of them — classified against the
question *"what does `exclude` actually need?"*:

| Need | Count | Expressible as |
|---|---|---|
| **Prefix** (`system/revision/**`, `build/**`, `ephemeral/**`, `system/clock/**`, …) | **26** | §5.4's `pattern/*` — **already crosses `/` at any depth** |
| **Suffix by extension** (`**/*.cache`, `**/*.o`, `**/*.tmp`) | **3** | `*.cache` — a trailing string match |
| Already correct (`drafts/*`, `tags/*`, `system/protocol/*`, …) | 9 | unchanged |
| **Infix** (`**/cache/**`, `**/cursor/**`) | **2** | *nothing* — and both are **guide examples**, not requirements |

**26 of 40 needed no new token at all**, because §5.4's subtree form already does what `**` was imported to
do. **3 needed one thing: a string match at the end.** **2 wanted infix, and neither was a requirement.**

So `**` bought exactly nothing. §6.1's Reentrancy MUST — the single argument ever offered for it — is
satisfied by `system/revision/*`.

### 11.2 The ruling

**Four forms, closed, tested in order** (§2.4): `*` match-all · `<literal>/*` subtree prefix, identical to
§5.4 · `*<literal>` **trailing literal**, a byte-suffix over the whole subject with `/` not special ·
`<literal>` exact.

**Form 3 is the only addition to §5.4's vocabulary in the entire corpus.** It is scoped to `exclude` and
`exclude_types`, it exists because an ignore-list wants match-by-extension, and it confers no reading on
`*` anywhere else.

**No infix form.** Excluding "anything containing a `cache` segment" is not expressible and is deliberately
unsupported — put ephemeral state under a prefix, or adopt a suffix. The two guide examples were rewritten
(`**/cache/**` → `*.cache`; `**/cursor/**` → `your-app/cursors/*`). If a real case ever needs infix it gets
added deliberately, with vectors.

**`**` is gone from the corpus.** Not undefined — **rejected.** §4.4.17 **V6** rejects any pattern outside
the four forms with `400 config/invalid-exclude-pattern`; a valid pattern carries at most one `*`, as the
whole pattern, the final character after `/`, or the first character. `**`, `a/**/b`, `a*b`, `*a*` all fail.

### 11.3 Why rejection, not omission — the part that closes the divergence

`entity-core-py`'s SA-PY-11 hazard was real: the matcher decides trie membership → membership decides the
version `root` → `root` is the version entry's identity, so two peers reading a pattern differently produce
different version hashes for identical content with nothing erroring.

**A spec that merely omits `**` and a spec that rejects it are indistinguishable until a config carries
one.** Omission leaves each implementation to do something reasonable, and "reasonable" is where the
divergence lives — it is how `path.Match` and `fnmatch` ended up in two impls in the first place. **Only
write-time rejection closes it**, because an unrepresentable config cannot be stored, so no two conformant
peers can hold patterns they evaluate differently. `REV-GLOB-REJECT-1` is the vector that enforces it.

**A prior draft of this proposal cited the hash-determinism as a reason to defer the ruling. That was
backwards** — the divergence is *caused* by the surface being unpinned, so severity is the argument for
ruling now. Recorded because it is a reasoning error, not a typo: *when a gap's consequence is severe, that
raises the priority of closing it and lowers the acceptability of leaving it open.*

### 11.4 Cohort

**All three seats — implement §2.4's four forms and §4.4.17 V6, and drop any doublestar dependency.** Nine
vectors ship with the ruling (`REV-GLOB-*`). Neither stdlib default is conformant now for a reason simpler
than the withdrawn pin claimed: **both accept patterns this grammar rejects.**
