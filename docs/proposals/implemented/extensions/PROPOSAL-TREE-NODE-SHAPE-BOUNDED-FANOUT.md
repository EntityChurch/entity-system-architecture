# PROPOSAL v4.3 — EXTENSION-TREE node-shape: bounded-fanout content-addressed trie (IPLD HashMap algorithmic reference)

> **PULLED IN FROM LEGACY, 2026-09-06 — copied verbatim below this note, not rewritten.**
> `specs/extensions/EXTENSION-TREE.md` cites this document **three times** as the authority for the
> v4.0 substrate fork — §3.1 (`§6 / §8.4`, wire-compat boundary), §3.7.2 (`§4.7`, the five speculative
> use cases for the lost property), and **`§3` — *"why IPLD HashMap, not JMT or other alternatives"***.
> **It had never existed in this repo, in any commit**, so for the whole life of this corpus the
> alternatives analysis behind the single largest structural decision in the substrate was unreadable
> from the document that depends on it. Filed on the 2026-08-17 dangling-citation worklist (3
> citations, the `pull-in` disposition) and executed here.
>
> **The judgment the disposition rule requires was done before copying:** §3.4 and §4.7 were read
> against `EXTENSION-TREE` §3.7.2, and the spec's summary of the five use cases is faithful to §4.7.
> **One correction is owed and is NOT made here** (a pull-in is a copy): §3.4's ruling *"prolly trees:
> cross-impl non-determinism in chunker boundaries makes spec-precision hard"* is **true of prolly
> trees and false of ATProto's Merkle Search Tree**, which fixes fanout by a deterministic
> leading-zeros rule rather than a content-defined chunker and is normatively required to be
> reproducible. See `EXPLORATION-CERTIFICATE-TRANSPARENCY-THE-WITNESS-AND-THE-APPEND-ONLY-ASYMMETRY`
> §7. The *decision* is unaffected; the *stated reason* does not span the family it rules out.
>
> **Dates in this document are `cgid-*` build identifiers from the legacy tooling, not calendar
> dates.** They are preserved because a pull-in is a copy.

**Date:** cgid-10-215 (v4.3 — four-impl stress-test pass returned all concur; errata absorbed; one new use case added to §4.7 per user's "nested subtree / sync walk" intuition)
**Author:** entity-core-architecture
**Status:** **LANDED cgid-10-215.** Amendment text merged to EXTENSION-TREE.md v3.15 → v4.0 (substrate fork) and EXTENSION-REVISION.md v3.4 → v3.5 (co-landing note, no normative change). Proposal moved to `proposals/implemented/`. SIGNED OFF by core-go (REVIEW-V4.3-SIGNOFF) + core-py (42f53db); four-impl CONCUR from v4.2 stress-test pass (workbench-go 7724eec, core-go REVIEW-V4.2, core-rust RESPONSE-STAGE-7-V4-2, core-py fb64838). K=32 pinned (was CANDIDATE in v4.2); K-sweep moved to post-amendment parametric optimization, not closure gate. Wide-flat throughput validation moved to post-amendment empirical confirmation. Closure criteria 1-2 satisfied; criteria 3-7 are now impl-level work. **v3 WITHDRAWN.** v4.1 superseded after Q6 attestation framing was identified as math/trust conflation. v4.2 reframed around math contract. v4.3 absorbs five textual errata + adds incremental-sync use case to §4.7. Direction unchanged across all revisions since v3 (IPLD HashMap algorithm + ECF wire).
**Audience:** entity-core-go, entity-core-rust, entity-core-py, entity-workbench-go.
**Reviews absorbed (verified primary-source by each impl):**
- core-go: `REVIEW-PROPOSAL-TREE-NODE-SHAPE-V3-IPLD-HASHMAP-cgid-10-215.md` (7 amendment-text issues, none structural)
- core-rust: `RESPONSE-STAGE-7-V3-IPLD-HASHMAP-cgid-10-215.md` (4 pushbacks, all primary-source-verified)
- core-py: pushed at `2a92b2d` (6 issues + process note, all WebFetch-verified against IPLD spec / go-hamt-ipld GitHub tags / Filecoin specs-actors post-mortem)
- workbench-go: `455fc8e` (2 CRITICAL + 4 HIGH + 2 MEDIUM)

**Direction status:** all three impls + workbench-go **CONCUR on IPLD HashMap structural adoption.** Twelve+ precision items routed for v4. None structural; all amendment-text.

---

## §0 v3 → v4 → v4.1 → v4.2 → v4.3 changelog

### §0.-2 v4.2 → v4.3 (four-impl stress-test errata absorbed)

All four impls returned CONCUR on the v4.2 math contract + touch-surface scope. No missed use cases surfaced in any impl's codebase audit. Five small textual errata flagged across the four reviews + one substantive use case addition per user direction:

| # | Source | Fix | Where |
|---|---|---|---|
| H1 | core-rust + visual audit | §4.5 had two effectively-duplicate rows ("Each binding provably in S" + "Per-binding inclusion proof against snapshot root"). Consolidated. | §4.5 |
| H2 | core-rust | §4.6 reconstruction math depends on identical canonical-normalize semantics across peers; add cross-reference to §F1 relative-key pinning so impls don't independently re-derive | §4.6 |
| H3 | core-go | §4.7 framing "wide-flat cliff and deep-nesting cliff that motivated this entire proposal" overstates deep-nesting as motivation. Original trigger was wide-flat; deep-nesting is a bonus the structure delivers. Reword. | §4.7 |
| H4 | core-go | §8.2 + §10.1 impl-effort estimates should explicitly note that the `trie_merge.go` algorithm shift is the load-bearing item within the 1.5-2.5 week per-impl estimate. Prevents budgeting for `core/tree/*` only and undershooting. | §8.2, §10.1 |
| H5 | workbench-go | §10.1 touch-surface table claimed "SDK / shell / app code: no change at contract layer" — true for core-go/rust/py, but workbench-go's `entitysdk/revision.go:598-639 collectMissingLeafHashes` walks `nd.Binding/nd.Entries` directly from SDK layer. Same walker shape as core-go's `transfer.go/pull.go`; <1 day mechanical. Honest add. | §10.1 |
| H6 | **user direction** | §4.7 use case enumeration missed: incremental-sync short-circuit via cached prefix-subtree hash (today's path-keyed lets a sync source short-circuit if the cached subtree hash at X matches; under hash-keyed this requires per-prefix index maintenance). Add as use case #5 with workbench's audit verdict (their production sync already retired tree:extract recipe in favor of revision:fetch-diff → tree:merge; uses version-DAG diff, not subtree-hash-equality). | §4.7 |

**Three further changes locked in v4.3 (post-errata, per user direction):**

| # | Change | Rationale |
|---|---|---|
| L1 | **K=32 pinned in §4.1**, not CANDIDATE. K-sweep removed from closure criteria. | K is a parameter; post-amendment parametric optimization handles it if measurement ever surfaces a workload-specific reason. The structural decision is what's load-bearing; don't gate the structural close on parameter validation that takes minutes to re-run. |
| L2 | **§4.7 tree-only-vs-tree+revision qualifier added** to use case #5 verdict. | Workbench's audit verdict (version-DAG supersedes tree:extract recipe) presumed revision-extension availability. Honest framing: tree-only consumers don't have version-DAG as an alternative, but they also don't have a code path that verifies prefix-subset completeness today. Same trust posture either way. |
| L3 | **Closure gates trimmed** — wide-flat throughput validation moved from gate to post-amendment confirmation. | Same principle as K-sweep: throughput is empirical confirmation, not a structural gate. |

**Nothing structural changed.** Math contract, touch surface, algorithm, wire format, test vectors, breaking-change framing, reversibility story — all preserved from v4.2.

**All four impls have concurred** on the math contract and touch-surface scope. The use-case audit returned zero hits in any impl. The errata are textual; no impl has flagged a structural concern. v4.3 ready for amendment-text drafting — one impl peer can do quick errata sign-off, then merge.

### §0.-1 v4.1 → v4.2 (Q6 dissolved, touch-surface stated honestly, "what we lose" deep-dived)

Working through Q6 with the three impl positions on the table (workbench: cap-layer attestation; core-go: scoped attestation; core-py: tree-layer attestation), surfaced that **all three were solving a non-problem.** A sender signature does not recover a lost mathematical property; it only adds an accountability layer. The actual question is: **is the lost mathematical property load-bearing?** Per Explore audit of every trie consumer across all three impls: no current operation uses it as a verification step. Q6 disposition: **drop attestation entirely; reframe as a math-properties contract; trust stays in its existing layer.**

| # | Change | Where |
|---|---|---|
| G1 | **Q6 dissolved.** Attestation primitive dropped from proposal. §4.5 reframed as math-properties table (what was true under path-keyed; what's true under hash-keyed; what's preserved, gained, lost). §9 Q6 reframed from "pick a/b/c" to "convergence check: do reviewers see a use case I missed?" | §4.5, §9 |
| G2 | **§4.6 new — what you CAN still compute.** Subtree-hash-by-reconstruction: build a fresh HAMT over filtered bindings under prefix X, get a deterministic root. CHAMP canonicalization guarantees two peers building the same set produce the same root. Subtree-hash equality survives — it's *constructed*, not *extracted as a structural subtree*. | new §4.6 |
| G3 | **§4.7 new — the genuine loss, honestly framed + future extension story.** The single property genuinely lost: the original-snapshot-root no longer cryptographically commits to "the set under prefix X is exactly H_X" as a derivable property. Currently unused — verified by consumer audit. Possible future use cases enumerated + the explicit extension-pluggability path for restoring this property via a parallel path-prefix Merkle index (sidecar entity) if any future use case demands it. We are not foreclosing; we are not building it into the substrate today. | new §4.7 |
| G4 | **Touch surface stated honestly per Explore consumer audit.** v4.1 §10.1 said "hard substrate fork; pre-1.0" (true but understated). v4.2 expands: rewrites `core/tree/*` (algorithm) + `ext/revision/*` (walker rewrites + merge algorithm shift). Other extensions (subscription, capability, identity, content, history) untouched per grep. Wire protocol unchanged. SDK/shell/apps speak preserved protocol contracts. The existing test corpus is the abstraction-correctness validation. | §10.1 expanded |
| G5 | **Closure criteria simplified.** v4.1's Q6 disposition gate is removed (Q6 no longer has dispositions to pick from); replaced with "convergence check on math properties." | §10.2 |

**Posture for v4.2 routing:** this is a stress-test pass before amendment text lands. Reviewers verify the math contract; explicitly look for use cases that might depend on the lost property; check the honest touch surface against their own codebases. If anything red-flags, we have a real architectural conversation; if everything concurs, amendment text lands.

**Reversibility note:** Stage 7 is a hard substrate fork pre-1.0. If post-amendment cross-impl work surfaces a use case that genuinely needs the lost cryptographic property, we can either (a) add it as an extension layer per §4.7's pluggability story, or (b) roll back the substrate change. Both are viable pre-1.0. We are not painting ourselves into a corner.

### §0.0 v4 → v4.1 (fresh-eyes critical pass, prior revision)

Four straight-up textual fixes + one new open question routed to peers:

| # | Fix | Where |
|---|---|---|
| F1 | **Pin SHA-256 input as `relative_key`, not "path".** EXTENSION-TREE §3.3 line 163 already binds trie keys to **prefix-relative** keys ("These are not paths — they are structural keys within a subtree"). Ambiguous "path" in v4 §4.1 was a silent-divergence trap of the exact class this proposal exists to prevent. | §4.1 |
| F2 | **Pull back IPLD wire-coherence framing.** v4 still left a tension between "IPLD HashMap reference" and "our own ECF wire." Sharpened throughout: we adopt the **algorithm + parameters + canonical-form rules** from IPLD HashMap. Wire is ours. No interop with go-hamt-ipld bytes; no tracking of IPLD spec evolution; bridge technology if anyone ever needs to exchange trees with IPFS. Field-name choice `{map, data}` retained for terseness, NOT for cross-tool wire interop. | §3.4, §6, §8.4 reframed |
| F3 | **Add 1-binding literal-hex test vector** alongside empty-root vector. Two impls reading the SHA-256-input rule differently produce different bytes for this vector — catches §F1's ambiguity class at fuzzer-touch time, not at cross-impl-divergence time. | §4.1 |
| F4 | **Breaking-change framing made explicit.** Stage 7 is a hard substrate fork — every existing snapshot and revision history root hash changes. Pre-1.0, no migration path, three impls + workbench land simultaneously. | new §10.1 |
| Q6 | **Cross-peer extract under hash-keyed trie — abstraction-layer question routed to reviewers.** Today's path-keyed trie gives cryptographic exhaustiveness for free (subtree-hash = "all entries under prefix X"). Under hash-keyed HAMT, exhaustiveness is asserted by sender, not derivable from snapshot root. The user-direction framing: **extract returns its own independent HAMT commitment over the filtered binding set** — the bigger tree can always be reduced; the patterns still work, but they need re-translation through one more abstraction layer. Surfaced as Q6 in §9 with three dispositions for peer review. | §4.5 expanded, §9 Q6 |

**Not changed from v4:** the 15 v3-absorbed corrections (below); the structural decision (IPLD HashMap algorithm + Filecoin parameters); the K=32 candidate framing; the closure criteria. v4.1 is a precision pass; structurally it is v4.

### §0.1 v3 → v4 changelog (absorbed in v4; preserved here for trail)

#### §0.1.1 Five CRITICAL fixes (all three impls converged)

| # | v3 error | v4 correction | Source |
|---|---|---|---|
| 1 | Quoted BOTH CHAMP arithmetic invariant AND IPLD form simultaneously | **Adopt IPLD form only:** "no non-root node may contain less than `bucketSize + 1` entries" | All 3 impls; matches go-hamt-ipld + js-ipld-hamt reference; operationally equivalent to CHAMP for IPLD params but textually distinct |
| 2 | Specified `binding: optional<hash>` field on every node | **DROP `binding` field entirely.** Under full-path hashing every value lives in a leaf bucket; no "binding at internal node" semantics. Root-prefix-binding handled via snapshot-envelope field, separately | All 3 impls (workbench, core-go, core-rust); vestigial from path-segment v1/v2 design |
| 3 | Cited Filecoin Jan-2021 post-mortem as evidence of "chain head mismatch from canonical-on-delete bug" | **REMOVE the citation.** Actual post-mortem (verified by core-py via WebFetch): Go map iteration nondeterminism + buggy sort, NOT CHAMP/HAMT. CHAMP-on-delete is still defensible on principle (see §4.4 retained motivation); just remove the wrong example | All 3 impls verified independently against [specs-actors/post-mortem-jan-21.md](https://github.com/filecoin-project/specs-actors/blob/master/docs/post-mortem-jan-21.md) |
| 4 | Pinned K=32 normatively in §4.1 before workbench K-sweep measures | **K=32 is CANDIDATE** pending workbench K-sweep validation; spec amendment text on K gated on measurement | core-py: pins-before-measurement inverts ground-in-measurement discipline I saved in same-day memory pin (`feedback_check_existing_specs_before_first_principles`) |
| 5 | "go-hamt-ipld v4" / "four major versions / v1 → v4" | **go-hamt-ipld v3.4.1 (May 2025), stable on v3.x since Jan 2021.** Two major versions (v2 → v3), not four. pkg.go.dev /v4 is pseudo-version dev build | All 3 impls; core-py verified via GitHub tags WebFetch |

#### §0.1.2 Six HIGH fixes

| # | v3 issue | v4 correction |
|---|---|---|
| 6 | Field names `{bitmap, entries, binding}` vs IPLD spec `{map, data}` | **Adopt IPLD spec names `{map, data}`** for spec-reading clarity + lineage signaling. Even with name alignment we are NOT byte-wire-compat with IPLD HashMap (see #7); names are about clarity, not interop |
| 7 | Implied "IPLD HashMap reference" meant byte-wire-compat | **Explicitly scoped:** IPLD HashMap is **algorithm reference + parameter source + reference-impl pattern**. Our wire encoding follows ECF (not dag-cbor); we use `system/hash` (not multihash); we do not use CBOR tag 42 for hash references. Cross-impl byte-identical-output verified by our own fuzzer, not interop with go-hamt-ipld bytes |
| 8 | "Pin CBOR encoding precisely" but no literal-hex examples | **Add literal-hex empty-root-node encoding** in same style as current §3.1 (`A1 67 656E7472696573 A0` precedent). Bucket-vs-link entry discriminator pinned by CBOR major type. Bitmap byte/bit order pinned big-endian |
| 9 | Impl burden "~20-50 lines per language" | **Re-priced: ~1-2 weeks per impl** for CHAMP-maintaining delete (walk down + remove + propagate collapse on ascent + inline when sub-node falls below invariant). Filecoin's go-hamt-ipld v2→v3 migration is the existence proof. Fuzzer authored alongside, not after — adds 2-3 days |
| 10 | Closure criteria conflated CBOR-simulation K-sweep with cross-impl conformance | **§10 distinguished:** CBOR simulation = K validation (workbench owns). Cross-impl fuzzer running same probe across all 3 impls + comparing root hashes = byte-identical-output conformance (impls own; gating spec close). Different artifacts |
| 11 | `SHA-256(path)` not precisely defined | **Pin: `SHA-256(UTF-8-bytes(canonical-normalize(path)))`** per existing EXTENSION-TREE §5.4 canonical normalization rules |

#### §0.1.3 Four MEDIUM fixes

| # | v3 issue | v4 correction |
|---|---|---|
| 12 | Jumped to K=32 citing Filecoin authority without engaging Python's K=16 argument | **Engage Python's argument in §3.2.** Under per-segment routing (v2 framing), Python argued K=16 minimized per-Put encoded-byte cost. Under full-path-hashing (v3+ framing), store-Put count dominates encoding cost ~10× per core-go's measurement — favors deeper-fanout K=32 or K=64. The discipline is acknowledging the argument shifted with the framing change, not citing authority |
| 13 | JMT-vs-IPLD rationale overstated | **Reframe (§3):** real reasons for IPLD HashMap over JMT are spec-precision + Go reference impl available + Filecoin-validated parameter choices. Dropped: "JMT is for RocksDB" (JMT's data structure is backend-agnostic; only version-key optimization is LSM-tuned). Dropped: "IPLD HashMap is multi-writer" (both data structures are sequential under update; convergence happens via byte-identical-output) |
| 14 | tree:diff / extract output ordering not specified under hash-keyed trie | **Pin sort step §4.3:** outputs of tree:diff, tree:extract, and revision:fetch-diff are sorted lex by path string at output time, independent of trie's internal hash-keyed traversal order |
| 15 | Cross-peer extract semantics under hash-keyed trie not enumerated | **§4.5 clarifying note:** `extract(prefix)` under hash-keyed trie goes through `LocationIndex.List(prefix)` (unchanged from v3). The trie's role is Merkle convergence; LocationIndex's role is path-keyed prefix scan. Hash-keyed routing does not break prefix-scan-by-LocationIndex |

#### §0.1.4 Process discipline lessons (new memory pin)

Per core-py's process note: Stage 7 went v1 → v2 → v3 → v4 in 24 hours, driven by:
- v1 → v2: cross-impl review caught 4 v1 errors (sidecar, compression, K-aesthetics, E-scope)
- v2 → v3: user pushback ("why isn't HAMT+CHAMP in literature?") forced IPLD HashMap discovery
- v3 → v4: three-impl primary-source verification caught 15+ v3 defects

**Each iteration produced a correct improvement.** The convergence on IPLD HashMap is right. **But the v1 → v2 → v3 process was reactive, not systematic.** I did not score alternatives (Ethereum MPT, JMT, IPLD HashMap, Prolly trees, etc.) against fixed criteria; I picked candidates serendipitously and got corrected.

Memory pin saved (`feedback_systematic_alternatives_matrix_before_recommendation`): for substrate primitive choices, author a systematic alternatives matrix (axes: domain match, production maturity, cross-impl coordination cost, asymptotic properties, security model) BEFORE recommending. Reactive course-correction is expensive and creates false impressions of arbitrariness.

The IPLD HashMap choice in v3 was correct. The process to it was serendipitous. v4 absorbs this discipline going forward, not retroactively (the substantive choice is fine).

---

## §1 TL;DR

**Adopt the IPLD HashMap algorithm and Filecoin's parameter choices for our trie node shape.** Adapt the wire encoding to ECF (not dag-cbor) and our `system/hash` (not multihash). The trie's role is Merkle convergence — hash-keyed routing internally; LocationIndex handles all prefix-scoped reads.

Specifically:

1. **EXTENSION-TREE §3.3 node format** follows IPLD HashMap structurally: `{map, data}` with bitmap-compressed sparse encoding
2. **Algorithm:** routes by bit-slices of `SHA-256(UTF-8(canonical(path)))`, full-path-hashed
3. **Parameters (candidate, pending workbench K-sweep validation):**
   - `bitWidth = 5` → K = 32 (matching Filecoin's `go-hamt-ipld` v3.x frozen choice)
   - `bucketSize = 3` (matching Filecoin + IPLD default)
   - Hash function: SHA-256 (matching IPLD default + our existing usage)
4. **Canonical form (normative MUST per IPLD spec):** no non-root node may contain, directly or via links, fewer than `bucketSize + 1` reachable entries; buckets sorted lex by key
5. **Wire encoding:** ECF (not dag-cbor); `system/hash` (not multihash); no CBOR tag 42

**Cross-impl coordination cost:** moderate. Algorithm reference + impl pattern from [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1); wire encoding is our own (ECF). Cross-impl byte-identical-output verified by our own fuzzer.

**Per-impl effort (realistic, with fuzzer authored alongside):**
- core-go: ~1 week
- core-rust: 1.5-2.5 weeks
- core-py: 1.5-2.5 weeks (revised up from "3-5 days" per Python's pushback — CHAMP-on-delete bugs are silent without fuzzer)

**K-sweep status:** CBOR simulation (workbench owns) validates K=32 candidate. Spec amendment text on K gated on K-sweep result.

---

## §2 What IPLD HashMap is (primary-source-verified)

### §2.1 The IPLD HashMap spec

[ipld.io/specs/advanced-data-layouts/hamt/spec/](https://ipld.io/specs/advanced-data-layouts/hamt/spec/). Direct normative quotes (verified by all three impls):

> *"no non-root node may contain, either directly or via links through child nodes, less than `bucketSize + 1` entries. […] for any given set of keys and their values, a consistent IPLD HashMap configuration and block encoding, the root node should always produce the same content identifier (CID)."*

This is the canonical-form invariant we adopt. It is operationally equivalent to CHAMP's arithmetic form (`branchSize ≥ 2 × nodeArity + payloadArity`) for IPLD parameters but textually distinct. **We quote the IPLD form**, matching go-hamt-ipld and js-ipld-hamt reference implementations.

> *"We must keep buckets sorted by `key` during both insertion and deletion operations."*

Bucket-sort invariant — applies to our trie node bucket entries.

> *"The default supported hash algorithm for writing IPLD HashMaps is SHA2-256 (identified by the multihash multicodec code `0x12`)."*

SHA-256 is the spec default. Matches our existing usage.

Default parameters: `bitWidth = 8` (K=256); Filecoin deviates to `bitWidth = 5` (K=32).

### §2.2 What Filecoin froze (verified)

go-hamt-ipld at v3.4.1 (May 2025; stable v3.x since January 2021):
- `bitWidth = 5` (K=32)
- `bucketSize = 3`
- SHA-256 hash

From [Filecoin's own commit message](https://github.com/filecoin-project/specs-actors/) on the bitWidth choice (verified by core-py):

> *"This value has been empirically chosen, but the optimal value for maps with different mutation profiles may differ, in which case we can expose it for configuration."*

**Filecoin admits the choice is empirical, not principled.** This matters: it's evidence FOR running our own K-sweep to validate, not assume the choice transfers.

### §2.3 What CHAMP gives us (motivation, not normative)

[Steindorfer & Vinju, OOPSLA 2015](https://michael.steindorfer.name/publications/oopsla15.pdf) §4 — the academic basis for the canonicalization invariant family. Direct quote:

> *"Clojure's HAMT implementations do not compact on delete at all... Bagwell's original version of insert is enough to keep the tree canonical for that operation. All other operations having an effect on the shape of the trie nodes can be expressed using insertion and deletion."*

**The load-bearing property for us:** byte-identical-output for the same key set regardless of insertion-and-deletion history. Without this, two peers building "the same" tree by different histories produce different root hashes; `convergent_mirror` breaks.

CHAMP gives us the *theory*; the IPLD HashMap form gives us the *normative spec text* (and the reference impl). We use IPLD form throughout the proposal.

---

## §3 Why IPLD HashMap not JMT (reframed)

### §3.1 What's NOT the reason

Per core-go pushback on v3 overstated rationale:

- **NOT "JMT is for RocksDB."** JMT's data structure is backend-agnostic. Only its version-prefixed-keys optimization is LSM-tuned. We could adopt JMT's structure on any backend.
- **NOT "IPLD HashMap is multi-writer."** Both data structures are sequential under update. Cross-peer convergence happens via byte-identical-output, not via concurrency primitives.

### §3.2 What IS the reason

Three substantive reasons:

1. **Spec precision.** IPLD HashMap has a written spec with normative MUST clauses, multiple implementations to compare against, and 5 years of operational lessons (parameter migrations documented across go-hamt-ipld v2 → v3.4.1). JMT has a paper plus the Aptos Move impl plus Penumbra's Rust crate; less documented operational history.

2. **Reference impl available.** [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1) is a mature, MIT-licensed Go reference implementation. Core-go can adapt it directly (algorithm-port, not byte-wire-compat). JMT's Penumbra `jmt` crate is similar but lineage-different.

3. **Filecoin-validated parameter choices.** `bitWidth=5`, `bucketSize=3`, SHA-256 have been running in Filecoin Lotus since ~2019. The empirical evidence for these parameters in a content-addressed-storage context is strong (per Filecoin commit messages it's empirical-not-principled, but 5 years of production at scale is real evidence).

### §3.3 Engaging Python's K=16 argument

Python's v1 review argued K=16 minimized per-Put encoded-byte cost under per-segment routing (v2 framing). Under the v3 → v4 reframe (full-path hashing), the argument changes:

- Under **per-segment** routing: each level's bucket-node holds 1 entry (for path-segment-shared workloads); encoding-byte cost dominates. K=16 wins.
- Under **full-path** routing: each Put walks log_K(N) levels; each level's bucket-node may hold up to `bucketSize` entries before recursing. Store-Put cost (one per modified node) dominates. Deeper-fanout (K=32 or K=64) means shallower tree → fewer Puts/Put.

Per core-go's measurement: store-Put cost > encoding cost ~10× in any realistic backend. **K=32 or K=64 should win under full-path routing.** K-sweep validates which.

This is not citing-Filecoin-authority; it's the math changing with the framing change. Python's K=16 conclusion was correct under v2's per-segment math; the framing changed in v3+; K choice re-evaluates.

### §3.4 What's NOT in scope

- **Verkle / vector-commitment trees:** ruled out per exploration §3.1 (Ethereum walked away).
- **JMT:** structurally similar to IPLD HashMap; lineage-different. We adopt IPLD HashMap for the three reasons above. Future could revisit if specific JMT features become valuable.
- **Prolly trees:** cross-impl non-determinism in chunker boundaries makes spec-precision hard. Future consideration if range-iteration becomes a requirement.
- **IPLD HashMap wire-level coherence:** explicitly out of scope. We adopt the **algorithm, parameters, and canonical-form rules** from IPLD HashMap. We do NOT adopt: dag-cbor, multihash, CBOR tag 42, the `representation tuple` encoding, the `HashMapRoot` parameter envelope, or any commitment to tracking IPLD spec evolution. Our trie nodes are not interoperable with go-hamt-ipld or any IPLD reader. If anyone ever needs to exchange trees with IPFS, that is **bridge technology** (a translator at the application layer), not a substrate concern. Wire format is precision territory we can refine post-1.0; the structural decision (algorithm + canonicalization) is the thing this proposal pins. core-rust's v3 review suggested "Option A: full IPLD wire-compat" for cross-validation against go-hamt-ipld as a test oracle; we declined because (1) the IPLD-tooling cross-validation benefit is small when our cross-impl fuzzer is the stronger oracle, (2) switching trie nodes to dag-cbor + multihash creates a wire-format boundary mid-ECF that violates V7 §1.11 boundary-conformance, and (3) we don't want to track an external project's spec evolution as a substrate dependency.

---

## §4 The proposed spec text (v4 amendment shape)

### §4.1 EXTENSION-TREE §3.3 — node format (REWRITTEN)

> **Node format.** Trie nodes follow the IPLD HashMap node structure adapted to ECF encoding:
>
> ```
> system/tree/snapshot/node := {
>   map:  bytes(K/8),         ; popcount-friendly bitmap of occupied positions (see "Bitmap convention" below)
>   data: [Entry, ...]        ; dense array; length = popcount(map)
> }
> ```
>
> **Bitmap convention (MUST, pinned to match IPLD HashMap / go-hamt-ipld).** The bitmap encodes a K-bit unsigned integer where **position `p` is bit `p` of the integer** (LSB-indexed: position 0 corresponds to the value 1; position 31 corresponds to the value 2^31). The integer is serialized as `K/8` bytes **big-endian** (most-significant byte first). For K=32, position 28 set → integer value `0x10000000` → 4-byte serialization `10 00 00 00`. Three impls reading this rule differently produce different root hashes for any non-trivial trie; this is normative MUST and the single-binding test vector below catches deviations at fuzzer-touch time.
>
> Where `K = 2^bitWidth` and:
>
> - **`bitWidth = 5`** (K = 32) — PINNED. Matches Filecoin's frozen choice (5-year production validation). Determines bitmap size = 4 bytes; determines logical fanout = 32 buckets per node. K is a parameter; if post-amendment workload measurement surfaces a decisively better K for our use cases, that's a parametric optimization not a structural change — would be addressed by a future K-amendment without re-opening the substrate decision.
> - **`bucketSize = 3`** — maximum entries per occupied position before recursing to a sub-node.
> - **Hash function: SHA-256.** Input MUST be `UTF-8-bytes(canonical-normalize(relative_key))` where `relative_key` is the prefix-relative trie key per EXTENSION-TREE §3.3 line 163 ("These are not paths — they are structural keys within a subtree produced by trimming a known prefix from the stored path"). The hash input MUST NOT include the prefix or peer-id — those are operational parameters to `snapshot`/`extract`/`merge`, not part of the trie key. Canonicalization rules follow existing EXTENSION-TREE §5.4. **Two impls reading "key" two different ways = silently divergent root hashes;** the relative-key binding is normative MUST and is also the input to the 1-binding test vector below.
>
> Each `Entry` is discriminated by CBOR major type:
>
> - **CBOR major type 4 (array)** → Bucket entry: array of [key, value_hash] tuples, length ≤ `bucketSize`, sorted lex by key (UTF-8 lex order)
> - **CBOR major type 2 (byte string)** → Link entry: 33-byte `system/hash` of a sub-node entity
>
> **Canonical form (MUST) per IPLD HashMap spec:** no non-root node may contain, either directly or via links through child nodes, less than `bucketSize + 1` reachable entries. Buckets sorted lex by key. The root node MAY contain fewer.
>
> **Parameters MUST NOT be exposed as per-tree configuration.** The wire format does NOT carry parameter values. Drift between implementations on bitWidth, bucketSize, or hash function produces silently divergent root hashes.
>
> **No `binding` field.** Under full-path hashing, every binding is a leaf-bucket `[key, value_hash]` entry, found by traversing `SHA-256(UTF-8(canonical(path)))` bit-slices. The "binding at path P" and "prefix for path P/..." semantics are independent lookups — preserved by LocationIndex (path-keyed), not by trie structure.
>
> **Literal CBOR encoding of empty-root node** (K=32, no bindings):
>
> ```
> A2 63 6D6170 44 00000000 64 64617461 80
> ```
>
> Where `A2` = map(2), `63 6D6170` = "map", `44 00000000` = 4-byte zero bitmap, `64 64617461` = "data", `80` = array(0). `content_hash = SHA-256("system/tree/snapshot/node" + <those 17 bytes>)`. **All implementations MUST produce this exact byte sequence for the empty-root node.** Any deviation breaks cross-peer trie root comparison.
>
> **Field naming note (v4.1):** the field names `map` and `data` are borrowed from the IPLD HashMap spec for terseness and lineage clarity in spec text — NOT for wire-level interop with go-hamt-ipld or any IPLD reader. IPLD uses CBOR `representation tuple` (positional 2-element array); we use a CBOR map per ECF. Even with the matching strings, our trie node bytes are not consumable by IPLD tooling and we explicitly do not intend to track IPLD spec evolution (see §3.4, §6, §8.4).
>
> **Literal CBOR encoding of single-binding root node** (K=32, one binding at relative_key `""` with value_hash `H`):
>
> Worked from `SHA-256(UTF-8-bytes(""))` = `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` (well-known empty-string SHA-256). First 5 bits of the hash (MSB-first bit-extraction from byte 0 = `0xe3` = `0b11100011`) = `0b11100` = position 28. Per the bitmap convention above, position 28 set → integer `0x10000000` → 4-byte big-endian = `10 00 00 00`. Data array length 1; entry is a bucket with one `[key, value_hash]` tuple: bucket is CBOR array of 1 element which is itself CBOR array `["", H]`.
>
> Full encoding (where `<H>` is the 33-byte `system/hash` of the value):
>
> ```
> A2 63 6D6170 44 10000000 64 64617461 81 81 82 60 58 21 <H>
> ```
>
> Breakdown: `A2` = map(2); `63 6D6170` = "map"; `44 10000000` = byte string len 4, bitmap with bit 28 set; `64 64617461` = "data"; `81` = array(1) (data has one entry); `81` = array(1) (entry is bucket with one tuple); `82` = array(2) ([key, value_hash]); `60` = empty text string ""; `58 21` = byte string of length 0x21 = 33; `<H>` = 33 bytes of value-hash. **Three impls MUST produce this exact byte sequence for a single binding at `relative_key=""` with the same value_hash.** Two impls producing different bytes for this vector = silent root divergence at the §F1 ambiguity line. This vector is the canonical fuzzer seed for §B (SHA-256 input pinning).

### §4.2 EXTENSION-TREE §3.4.2 — incremental update algorithm

> **Algorithm:** the IPLD HashMap insertion / deletion algorithm with canonical-form maintenance. Per-Put:
>
> 1. Compute `hash_bytes = SHA-256(UTF-8-bytes(canonical-normalize(path)))` — produces 32 bytes
> 2. Walk levels by consuming `bitWidth` bits from `hash_bytes` (big-endian per byte, MSB first)
> 3. At each level, position `p = next bitWidth bits`; check `map` bitmap at bit `p`:
>    - If clear: set bit `p`; insert `[key, value_hash]` tuple into a new bucket at the appropriate `data` position; done
>    - If set: locate existing entry at position `popcount(map & ((1 << p) - 1))` in `data`
>      - If existing entry is a bucket and `len(bucket) < bucketSize`: insert tuple into bucket (maintain lex sort); done
>      - If existing entry is a bucket and `len(bucket) == bucketSize`: convert to sub-node; recurse all `bucketSize + 1` entries (existing + new) into the sub-node
>      - If existing entry is a link: descend into sub-node; recurse
> 4. On ascent, rehash modified nodes; produce new root
>
> Per-Delete:
>
> 1. Walk via `SHA-256(...)` bits to the binding location
> 2. Remove the tuple from the bucket
> 3. On ascent, enforce canonical-form invariant:
>    - If a sub-node now has `branchSize < bucketSize + 1`: collapse it; re-inline contents into parent's bucket (maintain lex sort)
>    - If parent's bucket now has `len(bucket) == 0`: clear bit in parent's `map`; remove from `data`
>
> Reference implementation: [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1) (algorithm-reference; wire encoding differs per §4.1).
>
> **Implementations MAY use any algorithm** that produces the same trie structure as a canonical-form IPLD HashMap built from the full updated binding set. Byte-identical-output is the cross-impl invariant; algorithm choice is implementation freedom.
>
> **What "same trie structure" means precisely (per core-py sign-off, cgid-10-215):** byte-identical CBOR encoding of every node. The §4.1 empty-root and 1-binding test vectors are the byte-level conformance anchors for the simplest cases; the cross-impl byte-identical-output fuzzer extends this to arbitrary binding sets. If two impls produce different bytes for the same binding set under the same parameters, one of them is non-conformant — the test vectors plus fuzzer narrow down which.

### §4.3 EXTENSION-TREE §4.3 — diff algorithm

> **Algorithm:** walk both trie roots in parallel by `map` bit-position; early-exit on hash equality (entire subtree unchanged); recurse into bit positions where the two trees' entries differ; report (added, removed, changed) at leaf positions.
>
> The previous §4.3 compression-mismatch machinery (`resolveMismatch`, `LongestCommonPrefix`, `ResolveAtDivergence`) is REMOVED — IPLD HashMap nodes have fixed-position structure indexed by `map` bitmap position; there is no compressed-key divergence to resolve.
>
> **Output ordering (MUST):** the diff output's `added`, `removed`, and `changed` arrays are sorted lex by path string. The trie's internal traversal order is hash-keyed (random with respect to paths); the output is explicitly re-sorted before return. This makes diff outputs comparable across peers regardless of internal traversal order.

### §4.4 EXTENSION-TREE §5.4 — merge algorithm

> Same simplification as §4.3. Walk-then-apply by bit-position; canonical-form invariant maintained throughout. Output sort same as diff.

### §4.5 EXTENSION-TREE §6 — extract under hash-keyed routing (MATH PROPERTIES CONTRACT)

> **Math, not trust.** This section describes what the new structure mathematically gives the receiver. Trust concerns (was the sender honest about what they sent?) are unchanged from today and stay in their existing layer (peer identity, cap chains, signed protocol envelopes). The earlier framing of this section as a "sender attestation" was conflating math with trust; v4.2 reframes around math properties only.
>
> **Local prefix-scoped reads — unchanged.**
>
> `extract(prefix)` continues to use `LocationIndex.List(prefix)` (B-tree range scan or BTreeMap equivalent) → O(prefix-subtree) bindings; the trie is not descended for prefix narrowing. This is the existing pattern across all three reference implementations.
>
> Similarly, `revision:fetch-diff(prefix, base)` computes diff via §4.3 (full-trie diff, bounded by actual differences via hash-equality early-exit at every node) then filters output by `path.starts_with(prefix)` (post-hoc filter on path strings). Filter cost is constant per diff entry; total cost bounded by changes, not by tree size.
>
> **Cross-peer extract — math properties before / after.**
>
> | Property | Path-keyed (today) | Hash-keyed (Stage 7) |
> |---|---|---|
> | Per-binding inclusion proof against snapshot root (each binding in the bundle provably in S) | YES (walk subtree, hash check) | **YES** (walk SHA-256 bits, hash check) |
> | Deterministic root over a given binding set (build → compare) | YES | **YES** (CHAMP canonical: same set → same root) |
> | Two peers building the same set produce the same root | YES | **YES** (CHAMP canonical; **gained:** holds under arbitrary insert/delete history) |
> | Original snapshot root cryptographically commits to "the set under X is exactly H_X" as a derivable property | YES (subtree hash, structural) | **NO** — see §4.7 for whether this matters |
> | Receiver can construct a deterministic hash for the subset under X | YES (read subtree hash from snapshot directly) | YES (build fresh HAMT from filtered set; see §4.6) |
>
> The only red row is the snapshot-root → prefix-subset commitment. Everything else is preserved or strengthened. **The single thing genuinely lost is the structural mapping from path prefix to original-snapshot subtree.** §4.6 covers what you can still compute under hash-keyed; §4.7 enumerates what could potentially depend on the lost property and the extension-pluggability path if any future use case ever needs it.
>
> **Trust posture — unchanged from today.** Today, a sender choosing what to extract is trusted by the receiver (peer identity, cap chain, protocol context). Tomorrow, same. The receiver verifies each binding mathematically; the receiver does not get a free "sender gave me everything they have" guarantee from the trie shape today either (a malicious sender today could extract a sub-subtree and ship that; the receiver verifies its hash matches a node in the snapshot but cannot tell from math alone whether the sender chose to ship a smaller scope than asked). Trust is its own layer; the trie shape change does not weaken or strengthen it.

### §4.6 What you CAN still compute under hash-keyed routing (subtree-hash by reconstruction)

> **Deterministic hash for an arbitrary subset survives.** Given any subset S' of bindings (e.g., "all bindings under prefix X"), an implementation can build a fresh HAMT over S' and obtain a deterministic root hash R_{S'}. Two impls building the same S' produce byte-identical R_{S'} (this is the CHAMP canonicalization invariant, the entire reason for adopting it).
>
> **Precondition — identical canonical-normalize across peers.** Reconstruction determinism depends on every peer applying the same canonical-normalize semantics to path strings when filtering "all bindings under prefix X." Two peers using different Unicode normalization upstream, or different prefix-stripping semantics, would filter to different sets and reconstruct to different roots even with identical CHAMP canonicalization. **§F1 (v4.1) pins this:** the SHA-256 input is `UTF-8-bytes(canonical-normalize(relative_key))` per EXTENSION-TREE §3.3 line 163; the same canonical-normalize rule applies to prefix-filter operations.
>
> So:
> - Want a stable hash for the set under prefix X? Filter via LocationIndex → build fresh HAMT → R_X. Cost: O(|set under X| × log_K |set under X|).
> - Want to compare "the set under X" between two snapshots? Compute R_X on each side; compare hashes.
> - Want to ship "the set under X" to a peer and let them verify it builds to the same hash? Send the bindings; receiver builds locally; receiver compares against your claimed R_X. If they match, both peers have the same set under X.
>
> **Subtree-hash equality survives — it's constructed, not extracted as a structural subtree.** Cost shifts from O(1) read of an existing subtree hash to O(|subset|) compute of a fresh HAMT. For most prefix-scoped operations the subset is small and the compute is cheap; LocationIndex filtering keeps the constant factor low.

### §4.7 The one property genuinely lost — when it might matter, and the extension path if it ever does

> **The lost property, precisely stated.** Under path-keyed routing, the snapshot root commits — cryptographically and derivably — to "what's at structural position X is exactly hash H_X." Under hash-keyed routing, the snapshot root commits only to the full binding set. There is no structural mapping from path prefix to a position in the trie; bindings under prefix X are scattered across hash-determined HAMT positions.
>
> **Audit of current consumers (Explore swept all three impls + the spec):** no current operation uses the path-prefix → structural-subtree-hash mapping as a verification step. Walkers (transfer.go, pull.go, fetch_diff.go and their rust/python equivalents) traverse the trie to collect entities; they do not anchor at "subtree hash at path X = expected H_X." Merge applies bindings; it does not verify subtree equivalence at path X. Subscription, capability, identity, content, history extensions do not consume trie internals at all.
>
> **So under hash-keyed routing, nothing breaks.** The property was a side-effect of the data structure; no operation depends on it.
>
> **What could potentially use the lost property — speculative use cases.** Worth enumerating honestly so reviewers can stress-test:
>
> 1. **Cryptographically-verifiable prefix-bounded subscription scope.** "Prove to me that this subscription is receiving exactly the events under prefix X in your snapshot, no more, no less." Today's path-keyed could in principle give this for free (witness via subtree hash equality at X in the snapshot at each subscription delivery point). Today's implementation does not actually do this; subscription delivery is per-event, not snapshot-anchored.
> 2. **Light-client prefix proofs.** "Give me a compact proof that these are all the bindings under X in snapshot S, without sending me the whole snapshot." Today's path-keyed could give this via a Merkle inclusion proof of the subtree hash. We do not currently have light clients; if we ever do, they'd need an alternative mechanism under hash-keyed (per-binding inclusion proofs, or the extension layer below).
> 3. **External audit / regulatory commitment** to subset-under-prefix without revealing other bindings. Hash-keyed makes this harder; you'd ship inclusion proofs per binding (with the disclosed value) plus a non-membership proof for any other position in the subtree under X, which is more complex than path-keyed's single subtree hash.
> 4. **Operational debugging / tooling** that thinks of "the subtree under data/users/" as a structural object with its own hash. This is operational mental-model, not protocol correctness — adapts to "the subset under data/users/" via LocationIndex + fresh HAMT build per §4.6.
> 5. **Incremental-sync short-circuit via cached prefix-subtree hash.** A sync optimization where the source caches `expected_hash_at_prefix_X` from a prior sync and short-circuits when the current snapshot's subtree-at-X-hash matches (no diff needed). Today's path-keyed makes this an O(1) lookup of the subtree hash. Under hash-keyed, the source would have to either maintain per-prefix Merkle indices separately (§4.7's parallel-sidecar extension) or compute the prefix-subset hash by reconstruction (O(|subset|), cheaper than full diff but not O(1)). **Workbench-go's audit verdict on this use case:** their production sync recipe (`cmd_revision_follow.go:58`) was already retired in favor of `revision:fetch-diff → tree:merge` (uses version-DAG diff, not subtree-hash-equality), so this optimization is not currently in use. **This is the strongest "potentially future-useful" case in the list** — if incremental sync optimization ever becomes a hot path and the version-DAG diff isn't enough, the parallel path-prefix Merkle sidecar (§4.7's first extension shape) restores it. Not building today; substrate doesn't foreclose.
>
> **None of (1)-(5) are real use cases for us today.** We do not ship light clients, we do not have subscription cryptographic scope proofs, we do not have external regulatory audit requirements. (4) is an operational adaptation, real but bounded. (5) is a retired sync recipe that the version-DAG approach already supersedes.
>
> **Subtlety on tree-only vs tree+revision composition.** The workbench audit verdict on (5) ("the version-DAG approach already supersedes the tree:extract-by-subtree recipe") is conditional on having the revision extension available. A consumer using ONLY the tree extension (without revision) would have tree:extract as their primary sync mechanism — they don't have version-DAG diff as an alternative. **But:** even in that tree-only mode, the receiver does not today verify "this bundle is the complete subtree under X in snapshot S" as a code path. Today's tree:extract receivers walk bundles and trust the sender for "you sent me what I asked for"; under hash-keyed, same trust posture, with the added math benefit of per-binding inclusion proofs against snapshot root (which today's receivers don't compute either). So tree-only consumers don't suddenly start needing the lost property; they have the same guarantee shape as today (each binding mathematically verifiable; bundle completeness sender-trusted). If a future tree-only consumer truly needs cryptographic prefix-subset completeness without depending on revision, that's a §4.7 extension-sidecar use case — sidecar path-prefix Merkle index, opt-in. Substrate doesn't foreclose.
>
> **The extension-pluggability story — if any of (1)-(3) ever becomes a real need.** Nothing about the v4.2 substrate forecloses a future extension that restores cryptographic prefix-subset commitments. Possible shapes:
>
> - **Parallel path-prefix Merkle index** as a sidecar entity. A separate `system/tree/prefix-index` entity type that maintains a path-keyed Merkle tree alongside the canonical hash-keyed HAMT. The HAMT is the source of truth (canonical, convergent under reorder); the prefix-index is a derived second commitment over the same binding set, optimized for prefix-bounded queries. Cost: O(N) storage overhead + O(log N) per write. Optional opt-in per tree.
> - **Application-layer Merkle accumulator** over (path, value_hash) pairs maintained by an extension when prefix-bounded cryptographic commitments are needed in a specific domain.
> - **Light-client proofs via per-binding inclusion + completeness oracle** at the cap layer (signed by the sender peer — accountability, not verification, but acceptable in a trust-anchor model).
>
> **We are not building any of these in v4.2.** We are saying: the substrate does not foreclose them; if any of (1)-(3) becomes load-bearing, propose the extension layer then. **YAGNI applied to a property we genuinely aren't using.**
>
> **What we lose by NOT building it today:** small. Operational mental-model adaptation (prefix-subset is "constructed" not "extracted"). Tools that want a stable subset hash compute it per §4.6.
>
> **What we'd lose by trying to bake it into the substrate today:** big. Either we keep path-keyed routing (and the wide-flat cliff that motivated this entire proposal — plus the deep-nesting cliff that hash-keyed resolves as a bonus); or we maintain two parallel commitments at the substrate layer (doubling the per-write cost on EVERY write, not just writes that touch prefix-bounded queries); or we invent novel cryptographic structures with their own correctness risks and zero production validation. The substrate choice is the wrong layer to push this complexity into.
>
> **If the community converges on a content-addressed P2P Merkle structure that preserves path-prefix locality at scale, we can revisit.** Today the production-validated structures (IPLD HashMap, JMT, MPT, Verkle's MPT predecessor) all give up path locality. We are aligning with the production-validated trade.

---

## §5 Algorithm impact per operation

| Op | Effect | Output semantics |
|---|---|---|
| `snapshot` (build_trie) | Walk by hash bit-position; canonical-form invariant | Same byte-identical-output property |
| `diff` | **SIMPLIFIES** — fixed-position structure removes compression-mismatch surface | Same (added, removed, changed) set; output sorted lex |
| `merge` | Same walk-then-apply structure; canonical-form invariant maintained | Same |
| `extract` | LocationIndex range scan (unchanged) | Same |
| `revision:fetch-diff` | Calls simplified `compute_trie_diff`; post-hoc path-prefix filter | Same |
| Subscription delivery | No impact | Same |
| Capability scope checks | No impact | Same |

---

## §6 What we carry from IPLD HashMap (and what we don't)

**We carry:** algorithm (hash-keyed bit-slice routing), parameters (`bitWidth=5` candidate, `bucketSize=3`, SHA-256), canonical-form rules (`bucketSize+1` invariant for non-root nodes; bucket-sort by key), reference-impl pattern (`go-hamt-ipld v3.4.1` as algorithm reference for impl authors).

**We do NOT carry:** dag-cbor encoding, multihash hash format, CBOR tag 42 for sub-node links, `representation tuple` positional encoding, the `HashMapRoot` parameter wrapper, or any commitment to tracking IPLD spec evolution. Our wire format is ECF + `system/hash` + our own `system/tree/snapshot` envelope, full stop.

**Implication:** the IPLD HashMap is our **algorithm and parameter reference; nothing more.** Cross-impl byte-identical-output is verified by our own three-impl fuzzer, not by interop with go-hamt-ipld bytes. If anyone ever needs to round-trip trees with IPFS, that is bridge technology at the application layer, not a substrate concern. This keeps us free to refine the wire format post-1.0 without coordinating with an external project.

### §6.1 What IPFS production scars tell us

From the exploration (§5 of `EXPLORATION-AUTHENTICATED-DATA-STRUCTURES`):

| Scar | How v4 addresses |
|---|---|
| Murmur3 unsafe with attacker-chosen keys | We use SHA-256 (matches IPLD spec default; matches our existing) |
| dag-pb HAMT (NOT CHAMP) → silent divergence | We adopt CHAMP-equivalent canonical-form invariant; normative MUST |
| Threshold off-by-one (`>=` vs `>`) between Kubo and JS | No threshold — always-HAMT structure; no flat-vs-HAMT transition |
| Per-instance pluggable parameters | We freeze in spec (§4.1); no wire-format parameter encoding |
| DOS via attacker-controlled fanout | Parameters in spec text; impossible via wire |
| Mutation paths half-aware of HAMT | One canonical walker; impl reference |
| "Experimental for 4 years" | Ship on by default from day one |

---

## §7 Workbench-go K-sweep — clarifications still answered

Per v3 §7 (workbench's 5 clarifications); all decisions stand:

| Workbench ask | Decision (unchanged) |
|---|---|
| Receiver-side cost on matrix | YES — add (H-G3 dominance) |
| Mixed-workload sanity check | YES — 70/30 wide-flat/hierarchical, single cell per K |
| Recovery curve against K | YES — single (K, N=2000) cell per K |
| Tiebreaker rule pinned | YES — wide-flat ratio > hierarchical regression > receiver-side > encoded bytes > K=32 wins ties matching Filecoin |
| Spoke fan-out | YES — M=4 baseline + M=32 cross-check |
| CBOR-simulation alternative | YES — validation not discovery |

### §7.1 K=32 as candidate

v4 lists K=32 as CANDIDATE in §4.1. Final K commitment for amendment text gated on workbench K-sweep result. **Spec amendment text on K cannot land until K-sweep validates.** This restores the ground-in-measurement discipline.

Expected outcome (per §3.3 reframing): K=32 wins or ties on producer-side; receiver-side may differ; tiebreaker rule resolves. If a different K wins decisively (>20% headline metric), we adjust.

### §7.2 Distinguishing simulation from cross-impl fuzzer

Per core-go: CBOR-encode-and-hash simulation discovers the K choice. **Cross-impl byte-identical-output fuzzer running the same probe across all three impls is a SEPARATE artifact** — it proves conformance, not parameter choice.

§10 closure criteria distinguish:

- K-sweep simulation → K candidate validation → amendment text on K
- Cross-impl fuzzer (impls own, running against published probe) → byte-identical-output conformance → cycle closure

Both are required; neither substitutes for the other.

---

## §8 Cross-impl coordination

### §8.1 Reference impl pointers

- **Algorithm reference:** [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1) (Go, MIT license). 5+ years production at Filecoin Lotus.
- **Spec:** [IPLD HashMap](https://ipld.io/specs/advanced-data-layouts/hamt/spec/) — language-agnostic.
- **Other implementations** (algorithm references, not byte-wire-compat with us either):
  - [fvm_ipld_hamt](https://docs.rs/fvm_ipld_hamt/) (Rust)
  - [ipld_hamt](https://docs.rs/ipld_hamt/) (Rust)
  - [ipld-hashmap](https://github.com/rvagg/iamap) (JavaScript)
- **CHAMP theory paper:** [Steindorfer & Vinju OOPSLA 2015](https://michael.steindorfer.name/publications/oopsla15.pdf)

### §8.2 Per-impl effort (realistic, with fuzzer alongside)

Per core-go's repricing and core-py's pushback on optimistic estimates:

| Impl | v3 estimate | v4 realistic estimate | Why |
|---|---|---|---|
| core-go | days–1 week | **~1 week** | Can adapt go-hamt-ipld v3.4.1 algorithm directly + ECF wire wrapper + fuzzer alongside. Estimate includes `ext/revision/trie_merge.go` three-way merge algorithm rewrite (segment-aware merge → filter+rebuild pattern per §4.6); this is the load-bearing single item within the estimate, not core/tree/* alone. |
| core-rust | 1–2 weeks | **1.5–2.5 weeks** | Port algorithm from spec; existing Rust HAMT crates use dag-cbor; we use ECF; fuzzer alongside. Estimate includes `extensions/revision/src/lib.rs` walker rewrites (direct `nd.binding`/`nd.entries` inspection) + trie_merge equivalent. |
| core-py | 3–5 days | **1.5–2.5 weeks** | Per Python pushback: CHAMP-on-delete bugs are silent without fuzzer; impl + fuzzer authored alongside, not after. Estimate includes `revision.py` walker rewrites (`_collect_missing_pull_hashes` direct field access). |

**Common load-bearing work across all three impls:**
- CHAMP-maintaining delete (walk down + remove + propagate collapse on ascent + inline)
- Cross-impl byte-identical-output fuzzer (portable probe; ports between impls; common test fixtures)
- Empty-root + 1-binding literal-hex regression tests (catches any deviation from §4.1 normative encoding)

**LoC sanity check (per core-go sign-off, cgid-10-215):** core-go's current trie internals (`trie_incremental.go` + `trie_diff.go` + `trie_merge.go`) are ~1500 lines. Under HAMT estimate is ~1270 lines because the compression-mismatch machinery (`LongestCommonPrefix`, `ResolveAtDivergence`, `DecomposeEntries`) goes away with fixed-position structure. **Net code reduction, not addition.** Stage 7 simplifies the trie codepath even as it changes the algorithm — the wide-flat cliff fix comes with codebase cleanup, not new complexity surface. Similar reduction expected in core-rust and core-py.

### §8.3 Cross-impl conformance discipline

The cross-impl byte-identical-output fuzzer is **NON-NEGOTIABLE.** Per CHAMP paper §4 and Python's emphasis: CHAMP-on-delete bugs are silent under insert-only tests; only surface as cross-peer divergence.

Fuzzer shape (proposed):
- Random sequence of N Put/Delete operations against a fresh trie
- Compare resulting root hash across all three impls
- Repeat for M random seeds
- Bug found ⇒ extract reproducer test fixture; pin in all three impls

Reference fixture pattern: core-go's `TestTriePutRemove_Fuzz` (from OP-1 work); generalizes to v4 algorithm.

### §8.4 What's NOT byte-wire-compat

Worth restating, since v3 reviews showed this is a recurring framing question:

| Layer | Our position |
|---|---|
| Algorithm (bit-slice routing, CHAMP-equivalent canonicalization) | **Identical to IPLD HashMap** |
| Parameter values (`bitWidth`, `bucketSize`, hash function) | **Identical to Filecoin's frozen choice** (pending K-sweep) |
| Canonical-form invariant text | **Adopt IPLD spec's `bucketSize+1` form verbatim** |
| Wire encoding (CBOR shape, field-name vs positional, hash format) | **Ours** — ECF, `system/hash`, CBOR map with `{map, data}` keys. NOT dag-cbor; NOT multihash; NOT tag 42; NOT `representation tuple` |
| `HashMapRoot` parameter envelope | **Not adopted** — parameters live in spec text, not on wire |
| go-hamt-ipld bytes ↔ our bytes | **Incompatible by design.** Bridge technology if anyone ever needs to exchange trees with IPFS |
| Tracking IPLD spec evolution | **Not committed.** Spec evolution is our own; we lifted a snapshot of the algorithm as a reference, full stop |
| Cross-impl test oracle | **Our own three-impl fuzzer.** NOT go-hamt-ipld interop |

This is deliberate scope-keeping. The structural decision (algorithm + canonicalization) is what this proposal pins; wire-format details are refinable post-1.0 within the entity-core protocol without external coordination.

---

## §9 Open questions for v4.1 review (one more pass)

1. **Field naming alignment** — v4.1 adopts IPLD `{map, data}` strings (NOT wire shape). Per §3.4 / §6 / §8.4, we explicitly decline IPLD wire-coherence. Confirm or push back on the field-name choice; if anyone prefers `{bitmap, entries}` for clarity-from-our-vocabulary, that's a one-line spec change with no downstream consequence (wire is ours either way).
2. **K=32 as candidate** — confirm the framing (candidate pending K-sweep, not pinned in normative).
3. **Empty-root + 1-binding literal-hex encodings** — v4.1 added the 1-binding vector to catch the §F1 SHA-256-input ambiguity at fuzzer-touch time. Verify both byte sequences in §4.1; if any impl computes different bytes from the same algorithm, flag now (this is the canonical fuzzer seed for the relative-key pinning).
4. **No-`binding`-field semantics** — confirm "binding at path P + prefix for P/..." continues to work via LocationIndex without trie-level support.
5. **Cross-impl fuzzer probe shape** — anyone want to propose specific properties beyond "random Put/Delete + compare root"? (e.g., specific stress shapes: 1M random + 1M deletes; bucket-collision stress; etc.)
6. **Math-properties contract — convergence check (REFRAMED in v4.2).** v4.1 routed this as Q6 with three disposition options for sender attestation. Working through the three impls' positions surfaced that attestation conflates math and trust layers — a signature does not recover the lost mathematical property; it only adds accountability. v4.2 drops attestation entirely; §4.5 reframes around math properties; §4.6 explains what survives via reconstruction; §4.7 deep-dives the one property genuinely lost and the extension-pluggability path if any future use case ever needs it.
   - **The convergence check we're asking reviewers to run:** read §4.5 (math properties table) + §4.6 (subtree-hash by reconstruction) + §4.7 (genuine loss + extension path). Then ask of your own codebase: **does any operation currently depend on "the snapshot root cryptographically commits to the subset under prefix X is exactly H_X" as a verification step?** If you can find a real use case where that derivation is load-bearing, raise it — we have an architectural conversation. If not, concur on the math contract and we land amendment text.
   - **The Explore audit found no current use case.** Walkers traverse the trie; they do not anchor on path-prefix-subtree-hash equality. Merge applies bindings; it does not verify subtree equivalence at a path. Subscription / capability / identity / content / history do not consume trie internals.
   - **What we're checking for:** missed use cases. Use cases buried in app code or planned-but-not-yet-built features. Anything where "the original snapshot root commits to subset under X" is the verification primitive, not just an incidental property.
   - **If anyone finds one:** see §4.7's extension-pluggability story. The substrate change does not foreclose restoring the property via a parallel path-prefix Merkle sidecar entity if/when needed. We choose not to build it today (YAGNI on a property we aren't using) but we are not painting ourselves into a corner. Pre-1.0; reversible.

---

## §10 Closure and rollout

### §10.1 Breaking-change framing + honest touch-surface scope

Stage 7 is a **hard substrate fork.** Every existing snapshot root hash, every trie node content hash, and every revision history root hash changes under this proposal. No on-wire migration; peers running pre-Stage-7 trie code cannot interoperate with post-Stage-7 peers. We accept this pre-1.0; all three impls + workbench land simultaneously.

**Honest touch surface — per Explore consumer audit (NOT estimates; actual grep across all three impls):**

| Surface | Impact | Why |
|---|---|---|
| `core/tree/*` (Go, Rust, Py) | **Full rewrite** | Algorithm change; this is the proposal |
| `ext/revision/transfer.{go,rs,py}` | **Walker rewrite** | Directly inspects `nd.binding` / `nd.entries`; rewrites to walk new `{map, data}` node shape. Contract preserved (visit all entities reachable from snapshot root); implementation new. |
| `ext/revision/pull.{go,rs,py}` | **Walker rewrite** | Same pattern — `_collect_missing_pull_hashes`-shaped walkers |
| `ext/revision/fetch_diff.{go,rs,py}` | API-call-site only | Calls `Collect*` functions whose APIs preserve; internals shift transparently to callers |
| `ext/revision/trie_merge.{go,rs,py}` | **Algorithm shift** | Segment-aware merge (current) → "filter by prefix via LocationIndex + build fresh HAMT" (§4.6 pattern). Same end-state; different mechanism. **This is the load-bearing single item in the per-impl effort estimate** — bigger than the core/tree/* rewrite by line count and conceptual change. |
| `entity-workbench-go/entitysdk/revision.go:598-639 collectMissingLeafHashes` | **Walker rewrite (workbench-only)** | Workbench-go's SDK has an additional walker that inspects `nd.Binding/nd.Entries` directly from SDK layer (per workbench's audit; flagged H5). Same shape as `ext/revision` walkers; mechanical rewrite, <1 day. **Note: this is workbench-specific.** core-go, core-rust, and core-py SDKs do NOT touch trie internals (verified by each impl's independent grep). |
| `ext/revision/snapshot.{go,rs,py}` + `deletion_markers.{go,rs,py}` | API-call-site only | Use `CollectAllBindings`; API preserved |
| `root_tracker.{py,go,rs}` | API-call-site only | Uses `trie_put` / `trie_remove`; APIs preserved |
| Test fixtures w/ hardcoded trie root hashes | **Re-pin values** | Mechanical; every test that pins a root hash needs a new value |
| `ext/subscription/*` | **No change** | Does not consume trie internals (audited by grep) |
| `ext/capability/*` | **No change** | Pattern-matches on paths; not a trie consumer |
| `ext/identity/*` | **No change** | Separate surface |
| `ext/content/*` | **No change** | Content store is hash-addressed; trie shape is opaque to it |
| `ext/history/*` | **No change** | Records hashes; trie shape opaque |
| Wire / transport / ECF | **No change** | Trie nodes are entities like any other; wire shape unchanged |
| Cap-chain validation / quorum / identity protocols | **No change** | Separate surfaces |
| SDK / shell / app code | **No change at contract layer** | Speaks protocol operations (`EXECUTE system/tree operation:snapshot`); trie internals invisible above revision-extension boundary |
| Conformance vectors for non-tree surfaces | **No change** | Stage 7 produces two NEW conformance vectors (empty-root + 1-binding, per §4.1); does not invalidate existing |

**The "slotted out / slotted in" question.** The slot is: `core/tree/*` (full rewrite) + `ext/revision/*` walker rewrites + merge-algorithm shift + test re-pins. Everything else is preserved across the contract layer. Other extensions, the wire protocol, the SDK, shell, and app code are not touched.

This is bigger than "trie internals only" (revision extension's internal walkers do change) but materially smaller than "cascades through the system." Two surfaces per impl, both contained, public contracts and wire shapes preserved at the boundary.

**Validation strategy — the existing test corpus IS the abstraction-correctness check.** If the trie rewrite + the revision-extension walker rewrites preserve the public contracts, all existing tests pass (modulo re-pinned root hash values). If anything else breaks, the abstraction wasn't clean and we have a real problem. Specific tests that prove the abstraction held:

1. Workbench cross-impl matrix (multi-peer convergence) passes
2. Conformance vectors for non-tree extensions pass without change
3. SDK / shell / app integration tests pass without change
4. Subscription delivery + capability checks + identity flows work end-to-end
5. The new empty-root + 1-binding test vectors (§4.1) match byte-for-byte across all three impls

**Reversibility cascade (per core-go sign-off, cgid-10-215) — "what if we find out we need the lost property":**

1. **First check whether reconstruction is fast enough.** §4.6 build-fresh-HAMT-from-filtered-set is O(|subset| × log_K |subset|). For typical workloads (filesystem subdir = 100-1000 entries; subscription prefix = thousands), reconstruction is µs to low-ms. Probably fast enough for whatever the use case is. **If yes, no action needed.**
2. **If reconstruction is too slow at the use case's frequency,** propose the §4.7 parallel path-prefix Merkle sidecar extension. Self-contained extension entity (`system/tree/prefix-index`), opt-in per tree, O(N) storage overhead + O(log N) per write. Restores O(1) prefix-subtree hash lookup. Same pattern Cosmos IAVL+, IPFS tile reads, Ethereum stateless clients all use. Production-validated approach. **Substrate stays unchanged.**
3. **If the sidecar isn't expressive enough** (light-client-class use case requiring proofs we haven't anticipated), real architectural conversation with real requirements on the table — not a hypothetical. May surface a novel extension shape.
4. **If nothing else works,** §10.1 reversibility says substrate rollback is viable pre-1.0. Worst case escape hatch.

All four paths open. We are not painting ourselves into a corner.

### §10.2 Stage 7 closure criteria (revised)

Stage 7 closes when:

1. **v4.3 errata sign-off absorbed** — at least one impl team confirms the v4.2 → v4.3 errata (5 textual + 1 use-case addition) don't change their concur position from the v4.2 stress-test pass. If clean: amendment text lands.
2. **Amendment text lands** in EXTENSION-TREE §3.3 / §3.4.2 / §4.3 / §5.4 + new §6 (math contract from §4.5-4.7) with §4 byte-level precision (including both literal-hex test vectors from §4.1 as conformance fixtures)
3. **Three impls ship** the new node shape + the §10.1 revision-extension walker + merge-algorithm rewrites (+ workbench-go's SDK walker rewrite per H5)
4. **Cross-impl byte-identical-output fuzzer green** across all three impls. The single-binding test vector is the canonical seed for SHA-256-input pinning (§F1).
5. **Existing test corpus passes** with re-pinned root hash values (per §10.1 validation strategy): workbench cross-impl matrix, non-tree conformance vectors unchanged, SDK / shell / app integration tests, subscription / capability / identity end-to-end

**K-sweep is NOT a closure gate.** K=32 is pinned per §4.1. Workbench-go's CBOR K-sweep is post-amendment parametric optimization — runs if and when measurement surfaces a workload-specific reason to revisit. If it does, a future K-amendment adjusts the parameter without re-opening the substrate decision. Not blocking Stage 7 closure.

**Wide-flat throughput target validation** (≤3× ratio) is post-amendment empirical confirmation — not a closure gate either. The structural change is what's load-bearing; throughput measurement validates the math held at runtime.

Note distinguishing:
- K-sweep (criterion 2): parameter validation; CBOR simulation acceptable
- Cross-impl fuzzer (criterion 5): byte-identical-output conformance; requires real impls

---

## §11 Pointers

- v3 (withdrawn): see Git history
- Three-impl v3 reviews:
  - `entity-core-go/docs/reviews/REVIEW-PROPOSAL-TREE-NODE-SHAPE-V3-IPLD-HASHMAP-cgid-10-215.md`
  - `entity-core-rust/docs/reviews/RESPONSE-STAGE-7-V3-IPLD-HASHMAP-cgid-10-215.md`
  - core-py at commit `2a92b2d`
  - workbench-go at commit `455fc8e`
- IPLD HashMap spec: [ipld.io/specs/advanced-data-layouts/hamt/spec/](https://ipld.io/specs/advanced-data-layouts/hamt/spec/)
- go-hamt-ipld v3.4.1: [github.com/filecoin-project/go-hamt-ipld](https://github.com/filecoin-project/go-hamt-ipld)
- CHAMP paper: [Steindorfer & Vinju OOPSLA 2015](https://michael.steindorfer.name/publications/oopsla15.pdf); text in `/tmp/papers/champ.txt`
- Exploration doc (background): `explorations/EXPLORATION-AUTHENTICATED-DATA-STRUCTURES-cgid-10-215.md` (with §9 addendum)
- Hot paths audit: `explorations/EXPLORATION-SYSTEM-HOT-PATHS-AUDIT-cgid-10-215.md`
- Memory pins:
  - `feedback_check_existing_specs_before_first_principles` (saved cgid-10-215 with v3)
  - `feedback_systematic_alternatives_matrix_before_recommendation` (saved cgid-10-215 with v4)

---

## §12 Sign-off trail

| Cycle | Impls | Disposition | Artifact |
|---|---|---|---|
| v3 stress-test | core-go, core-rust, core-py, workbench-go | CONCUR on direction; 15 amendment-text issues flagged | three-impl reviews cgid-10-215 |
| v4 absorption | (arch authored, all four reviewed) | 15 issues absorbed | v4 published |
| v4.1 fresh-eyes pass | (arch) | F1-F4 precision fixes + Q6 question added | v4.1 published |
| v4.2 Q6 dissolve | (arch + user) | Q6 reframed as math contract; touch surface stated honestly | v4.2 published |
| v4.2 stress-test | core-go, core-rust, core-py, workbench-go | CONCUR; 5 textual errata + 1 use case addition | workbench-go 7724eec; core-go REVIEW-V4.2; core-rust RESPONSE-STAGE-7-V4-2; core-py fb64838 |
| v4.3 errata + locks | (arch + user direction L1/L2/L3) | All errata absorbed; K=32 pinned; tree-only qualifier added; closure gates trimmed | v4.3 published |
| **v4.3 sign-off** | **core-go (REVIEW-V4.3-SIGNOFF), core-py (42f53db)** | **SIGNED OFF — ship it** | both confirmed errata landed cleanly + L1/L2/L3 absorbed correctly + algorithm walk-through verified + reversibility cascade documented |

**Three minor refinements pulled in from sign-off reviews (post-sign-off, non-structural):**
- §4.2 — "same trie structure" precision clarification (per core-py amendment-text observation)
- §10.1 — explicit 4-step reversibility cascade (per core-go)
- §8.2 — LoC sanity check (~1500 → ~1270, net code reduction) (per core-go)

None of the three refinements change the math contract, the touch surface, the algorithm, or any signed-off claim. They make the proposal sharper for amendment-text authors and impl planners.

---

*v4 absorbed three-impl v3 reviews (five CRITICAL + six HIGH + four MEDIUM). v4.1 added F1-F4 + Q6 attestation question. v4.2 dissolved Q6 via math/trust separation. v4.3 absorbed four-impl stress-test errata (5 textual + 1 use-case addition for incremental-sync) + L1/L2/L3 user direction calls. **v4.3 SIGNED OFF by core-go + core-py on cgid-10-215.** Three sign-off refinements absorbed (non-structural). Ready for amendment-text drafting in EXTENSION-TREE.md §3.3 / §3.4.2 / §4.3 / §5.4 + new §6 (math contract from §4.5-4.7). Closure criteria per §10.2.*
