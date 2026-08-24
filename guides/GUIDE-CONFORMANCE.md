# GUIDE-CONFORMANCE — How conformance is verified across implementations

**Status**: Draft

> **If we ever split this:** the natural cut is `GUIDE-CORE-CONFORMANCE` (this file's job — core protocol: wire + connect/auth + security floor) vs a future `GUIDE-EXTENSIONS-CONFORMANCE` (per-extension behavioral suites). Single file for now; no rename needed yet.

**Audience:** implementers wiring up the conformance gate; architecture/peer leads running the cross-impl publication round; the Go team updating `validate-peer`; keystone vendoring fixtures into the language-binding generator; any new-peer author asking "what do I have to pass, and what's not finalized."

**Scope (wire, §§1–6).** *How* you verify wire conformance. The *what* — which categories exist, what canonical bytes must look like, the fixture format, the harness contract — is normative in Appendix E. §§1–6 cover the operational loop: who authors what, how the validate-peer harness drives the cross-impl run, how divergences are triaged, how the corpus is versioned, how fixtures land in keystone.

**The wire floor is necessary but not sufficient.** Appendix E + §§1–6 cover **wire conformance** — the canonical encoding, content-addressing, identity, and envelope surfaces every peer shares — the universal floor every peer clears before anything else is meaningful. A conforming peer must *also* clear the **connect/auth** surface (handshake PoP, capability verification, auth-status boundary — V7 §4/§5) and, for each extension it ships, that extension's **behavioral** surface. §7 maps all of them and ties them to V7 §2.11 conformance levels; the per-extension *what* is each extension's own concern.

---

## §1 The conformance loop

The conformance gate exists because two impl-time encoder bugs (Rust receive-side re-encode 2026-05; Python's cbor2 float16 minimization 2026-05) made it through to production cross-impl matrix runs before being caught. The gate flips that: encoder corners are auditable contracts at impl-time, not artifacts discovered when the first float-carrying entity lights them up months later. The keystone work (W4) raised it from "good practice" to "blocking" — language-binding generators cannot bootstrap without a fixture every impl agrees on.

The loop runs once per corpus version:

```
┌───────────────────────────────────────────────────────────────────────────┐
│  1. AUTHOR        arch authors .diag source (inputs + ids; canonical      │
│                   field left blank for encode_equal vectors)              │
│                   ↓                                                       │
│  2. EMIT          each impl loads .diag/.cbor, runs every input through   │
│                   its canonical encoder, emits (id → bytes) per vector    │
│                   ↓                                                       │
│  3. DIFF          arch (or the harness) diffs emitted bytes across impls  │
│                   ↓                                                       │
│  4. TRIAGE        for each divergence: impl bug OR spec ambiguity?        │
│                   - impl bug → fix in impl, re-emit                       │
│                   - spec ambiguity → fix spec, regenerate inputs, re-emit │
│                   ↓                                                       │
│  5. LOCK          every impl agrees byte-for-byte → fill canonical field, │
│                   build .cbor, commit at canonical path, tag corpus       │
│                   version, keystone vendors                               │
└───────────────────────────────────────────────────────────────────────────┘
```

**Spec arbitrates, not majority.** If two impls produce one byte sequence and one produces another, the answer is not "two out of three wins." The answer is "what does §§3–9 of `ENTITY-CBOR-ENCODING.md` say the bytes are." If the spec is unambiguous, the minority impl has a bug. If the spec is ambiguous, the spec gets tightened in the same pass — and *all three* impls likely have to change.

---

## §2 Vector authoring

### §2.1 The `.diag` source

The fixture is `conformance-vectors-v{N}.cbor` (normative; loaded by impls) generated from `conformance-vectors-v{N}.diag` (human-editable; CBOR diagnostic notation per RFC 8949 §8). The `.diag` is the source of truth for authors; the `.cbor` is the build artifact.

A vector during authoring (canonical field empty):

```cbor-diag
{
  "id":          "float.7",
  "description": "max f16 finite — MUST minimize to f16",
  "kind":        "encode_equal",
  "input":       65504.0,
  "canonical":   h''
}
```

After cross-bless, the `canonical` field is filled with the converged bytes:

```cbor-diag
{
  "id":          "float.7",
  "description": "max f16 finite — MUST minimize to f16",
  "kind":        "encode_equal",
  "input":       65504.0,
  "canonical":   h'f97bff'
}
```

`decode_reject` vectors are authored directly with `canonical` populated — the bytes are the intentionally non-canonical wire input the decoder must reject:

```cbor-diag
{
  "id":          "tag_reject.1",
  "description": "Tag 0 (date/time) wrapping text in a data field — MUST reject",
  "kind":        "decode_reject",
  "canonical":   h'c074323032362d30362d30365431323a30303a30305a'
}
```

### §2.2 Categories

See `ENTITY-CBOR-ENCODING.md` Appendix E §E.1 for the normative category list. Two classes:

- **Class A — canonical encoding.** Pure ECF correctness over arbitrary canonical inputs (`float`, `int`, `map_keys`, `length`, `primitive`, `nested`, `tag_reject`).
- **Class B — protocol-surface conformance.** Compositions of canonical encoding with hash, signature, and envelope rules (`content_hash`, `peer_id`, `signature`, `envelope`). Added v1.5 in response to keystone F1.

Class B vectors compose Class A — a Class A failure for an impl typically cascades to the Class B vectors that exercise the same subordinate value. The categories localize the failure; they do not isolate it.

### §2.3 Adding a vector

1. Edit `conformance-vectors-v{N}.diag` — add the vector under the appropriate category, give it a fresh `<category>.<n>` id, write the description, set `kind`, populate `input` (for `encode_equal`) or `canonical` (for `decode_reject`).
2. Run the build script (see §3.1) to regenerate `.cbor` from `.diag`.
3. Run the cross-impl loop (§§1, 4) — for `encode_equal`, the canonical field gets filled by the converged bytes; for `decode_reject`, every impl's decoder must reject.
4. Commit the updated `.diag` and `.cbor` together.

### §2.4 Deterministic Ed25519 seeds (signature vectors)

The `signature` category uses fixed Ed25519 seeds named in the `.diag` as 32-byte hex literals. Ed25519 (RFC 8032) signing is deterministic — given a seed and a message, the signature is fixed — so vectors are reproducible across impls without per-impl key generation. Seeds are arbitrary; they exist only to make the test reproducible. Do not reuse seeds across vector ids — give each `signature.N` vector its own seed so a failure points cleanly at one input.

### §2.4a Every check of a guarded operation MUST assert the negative half `[MUST] [added 2026-08-11]`

**A check that asserts an operation was *accepted*, and nothing about the authority or the encoding behind it, certifies the absence of the guard.** A peer that skips the work scores at least as well as one that does it — and better, because the work can only cost it a failure.

> **The rule.** Where a normative rule says an operation is permitted **only** under some condition — a signature, a capability, a policy admission, a format — the vector set MUST contain **both halves**:
>
> 1. **the positive half** — satisfying the condition is accepted; and
> 2. **the negative half** — *failing* the condition is refused **and produces no state change and no publication.**
>
> A category that has only positive checks is **not** partial coverage of that rule. It is coverage of the *unguarded* behavior, and a correctly-guarded peer **fails** it.

**This is not a new idea; it is an existing rule that was only ever written one section at a time.** `EXTENSION-REGISTRY` §6a.9.1 already has it — `policy_manual_publishes_nothing`, `policy_allowlist_unlisted_publishes_nothing` — and those are exactly the checks nobody's registry failed. **The sections where nobody wrote it are precisely where all three implementations shipped the same hole:** §6a.9 named `REG-REGISTER-PROOF-1` for register and nothing for `revoke`/`renew`, and go, rust and py all three shipped those two ops with no verification at all. *The vector list, not the prose, is what gets implemented against.*

**Four defects in the 2026-08-11 cycle were the same shape, and every one was found by a peer rather than by the tooling:**

| Defect | How it was found |
|---|---|
| go's unauthenticated `revoke`/`renew` — its own checks asserted `200`/`202` with no proof, so a correct peer *failed* them | **core-py refused to converge** and reported instead of matching to 16/16 |
| go's containment boundary (`/srv/peerroot` ⊃ `/srv/peerroot-backup`) | auditing what **rust's report pointed at** — not the report |
| rust's second symlink escape (wrong-inode `lstat`) | rust's own audit; its note: *"our vector can't see that one; it would survive a green run in all three impls"* |
| rust's and py's round-trip tests passing a **deliberate mutation** | both sides shared an encoder, so the test compared a thing to itself |

**Two corollaries worth stating as rules.**

- **A sibling's refusal to converge is signal, not friction.** py's two red checks were worth more this cycle than rust's sixteen green ones. An implementation that reports a disagreement instead of matching the cohort is doing the job the cohort exists to do; **treat a lone red as a finding to investigate before treating it as a defect to fix.**
- **A round-trip check whose two sides share an encoder measures the encoder against itself.** It must be gated by a deliberate mutation that it is required to *catch* — a check that cannot be made to fail has not been shown to measure anything (§5.2b's twin, one layer up).

**The negative half MUST probe the state, not the response `[MUST] [added 2026-08-11-b]`.** §2.4a's three conjuncts are *refused* **and** *no state change* **and** *no publication*. **Asserting the status and the code satisfies only the first.** For any guarded operation that writes, a check MUST additionally read back the surface the operation would have written and assert it is absent or unchanged — a peer that answers `403` and performs the write anyway satisfies a status-only assertion completely.

> **Where this concentrates, and it is not per-vector forgetfulness.** It pools in **shared deny helpers**. core-go's 2026-08-11 audit of its own suite found two — `sendAndExpectAuthzDeny` (8 callers, asserts `status == 403` and nothing else) and the `extractStatusAndCode` path — and because `sendScopeTest` tail-calls the first, **every scope check inherits status-only assertion regardless of which operation it drives.** Four of those instances drive writes (`system/tree:put`, `subscription:subscribe`, `capability:request`, `role:delegate`). One helper, eight callers, four real holes, and nothing in any individual vector is wrong.
>
> **So a new vector family built on a fresh `expect_X_refused(...)` helper reproduces this hole by construction, even with both halves nominally present** — the helper asserts the code and stops. When authoring a matrix for a *new* security predicate, write the state probe into the helper first, before it has callers. It is the cheapest moment it will ever be.

**Corollary — do not audit for this by grepping declaration strings.** The declaration is not the check, and name-grepping scores it wrong in *both* directions: core-go's `authz` category scores **0/11** on a refusal-word grep while *being* the negative halves, and `encryption`'s `sender_auth_peer` reads as a single positive declaration while carrying two tamper vectors. **Read the implementation.** An audit that reports coverage from category names has measured its own vocabulary.

This is ADR-0012's *"conformance-green ≠ correct if the test asserts the wrong thing"* in its authoring-time form: §5.2a is a rule with no surface, §5.2b is a surface the suite cannot reach, **§2.4a is a surface the suite reaches and scores backwards**, and §5.2c is a surface the suite reaches only **by accident**.

---

## §3 The harness

The conformance loop runs through Go's existing `cmd/validate-peer` infrastructure. Each impl produces canonical bytes through its own encoder; validate-peer orchestrates the cross-impl run and diffs results. No new peer-driven harness is needed.

### §3.1 What each impl provides

Each conformant impl ships a small mode in its existing test/conformance binary (Go's `cmd/internal/wire-conformance`, Rust's existing wire-conformance harness, Python's equivalent) with the same contract:

```
$ <impl-harness> emit-canonical --diag <path/to/diag> --out <path/to/output.cbor>
```

The harness:
1. Loads the `.diag` (or, equivalently, an already-built `.cbor` carrying the same vectors).
2. For each `encode_equal` vector, runs the impl's canonical ECF encoder on `input`.
3. Emits a CBOR map `{ vector_id (tstr) → canonical_bytes (bstr) }` at the output path.
4. For each `decode_reject` vector, runs the impl's decoder on `canonical`; emits `{ vector_id (tstr) → rejected (bool) }` in a sibling map or as an additional output map keyed `decode_results`.

The output is a `system/protocol/conformance-emission/v1`-shaped entity (subject to the normal canonical-ECF rules). The shape is named so the diff harness can load it as data and not have to know about the impl's exit codes or log format.

### §3.2 The Go-side build script

The `.cbor` build artifact is generated from `.diag` by a small script in `entity-core-go/cmd/internal/wire-conformance/`:

```
$ go run ./cmd/internal/wire-conformance build-fixture \
    --diag  core-protocol-domain/specs/test-vectors/ecf-conformance/conformance-vectors-v1.diag \
    --out   core-protocol-domain/specs/test-vectors/ecf-conformance/conformance-vectors-v1.cbor
```

This is mechanical translation of `.diag` to canonical-ECF-encoded `.cbor`. It is not the cross-impl gate — the gate is the cross-impl `emit-canonical` agreement under §3.1. The build script exists because impls load `.cbor`, not `.diag`.

### §3.3 The validate-peer cross-impl category

A new `validate-peer` category (`-category conformance`) consumes the per-impl emission outputs and diffs them. The category is convergence-shaped (operates over multiple peer outputs at once), not single-peer-shaped:

```
$ validate-peer -peers go:emit-go.cbor,rust:emit-rust.cbor,python:emit-python.cbor \
                -category conformance \
                -corpus conformance-vectors-v1.cbor \
                -json-out reports/conformance-v1.json
```

The category:
1. Loads `corpus` to know which vector ids exist and what `kind` each one is.
2. Loads each impl's emission file.
3. For `encode_equal` vectors, compares emitted bytes across all impls. A vector passes iff every impl emitted byte-identical output. A vector fails iff at least one impl disagrees; the report enumerates which impl emitted what.
4. For `decode_reject` vectors, compares each impl's `rejected: bool` decision. Pass iff every impl rejected.
5. Emits a per-vector, per-category, per-impl pass/fail table in the report.

The `-peers` flag accepts `<label>:<path>` pairs rather than network addresses; the convergence-mode plumbing handles arbitrary peer labels. (validate-peer's other convergence categories use network addresses because they exercise live peers; conformance is an offline run over emitted artifacts.)

### §3.4 Why this and not a generator-driven flow

The earlier proposal text named Go's `core/ecf.go` as the reference encoder and assigned it to produce canonical bytes that other impls would verify against. **That framing is retracted as of Appendix E v1.5.** Reasons:

1. **No impl is privileged.** ECF is defined by §§3–9, not by any impl's behavior. If Go's encoder has a bug at some boundary (it has not, but it could), cross-blessing against Go propagates the bug.
2. **The loop reveals more.** Generator-driven flow produces bytes once; the other impls report agree/disagree. Cross-impl flow produces bytes three times; if two agree and one disagrees, the question becomes "what does the spec say?" — and either the minority impl is wrong (it has a bug) or the spec is ambiguous (and is the work product of the round). Generator-driven flow flattens that second case.
3. **It scales to new impls.** A fourth impl (next: C# via keystone; eventually a fifth) joins by emitting and being part of the diff — no privileged-encoder dance.

Generator-driven flow is fine for *bootstrap* — keystone's interim oracle wraps `core/ecf.Encode` in a container and that's how F5 was caught on day one. But the v1 publication gate is cross-impl agreement, not generator output.

### §3.0 Core-peer scoreboard discipline (normative for `--profile core`; V7 v7.72)

Until the validate-peer oracle ships `--profile {core|full}` (cohort target ~2 days from v7.72 ratification), a core peer is scored by a per-category hand-maintained scoreboard against a documented exclusion list — NOT by a blind full-suite verdict. The keystone-generated C# core peer is the worked example (the keystone ARCH-ASK-CORE-PEER stewardship doc, §1 scoreboard). Discipline:

1. **Run by category** (`-category` flag), not full-suite. The full-suite verdict for a core peer is uninformative (~758 fails under v7.72 = absence of core verdict, not 758 bugs).
2. **Document every excluded "fail" as an over-demand with a spec citation.** Not relaxed; not silently dropped. Every per-check carve-out names the targeting extension (`system/subscription` → SUBSCRIPTION; `system/role` → ROLE; etc.) and the V7 §9.0 core-profile entry that excludes it.
3. **Core-real check categories under v7.72 §9.0:** `connectivity, encoding, universal_address_space, peer_canonicalization, format_agility, crypto_agility, negotiation, multisig, type_system, handlers, tree_operations, capability, authz, security` (14 total).
4. **Per-category check subsets under v7.72:**
   - `type_system` — scored against the 53-type floor (V7 §9.5).
   - `handlers` — `tree` handler op-set is `{"get","put"}` only; EXTENSION-TREE §9 ops matched-if-present.
   - `tree_operations` — six `CORE-TREE-*` vectors (V7 §9.5a); §9 ops dropped.
   - `security` — `handler_scope_denied` skipped (targets `system/subscription`); rerouted-core variant runs instead when authored.
   - `authz` — `authz_delegate_grant_1` skipped (targets `system/role`); `authz_revoked_1` split. **Core variant (`authz_revoked_core_1`) accepts EITHER `(403, capability_revoked)` (preferred when the verifier knows the cap was revoked — same principle as v7.71's "no catch-all when a defined code applies") OR `(403, capability_denied)` (legitimate fallback when the impl genuinely doesn't track the specific reason).** Both are conformant per V7 §3.3 line 900 — `capability_revoked` is enumerated as a core defined authorization code; the ROLE-scoped element is the **401 status carve-out** for the in-flight cascade race (§5.5), not the code itself. **ROLE variant (`authz_revoked_1`) asserts `(401, capability_revoked)` under `--profile full` with ROLE installed** — the cascade fail-fast surface specifically. Per the Class-C 403 `capability_revoked` core ruling.
5. **Once `--profile core` ships in Go**, the scoreboard becomes a clean machine-checkable PASS/FAIL; this discipline applies to interim runs only.

### §3.1 Run discipline (normative for cohort closeout; V7 v7.70)

A 23-day false-baseline drift (a harness keypair bug whose 20-test cascade was repeatedly labelled "pre-existing infrastructure" and inherited across sessions; documented in the v7.69 same-format-drift postmortem) makes these binding on any cohort closeout that reports test results:

1. **No "pre-existing" disposition without a bisect.** A failure claimed to predate the current work MUST carry the triple: the commit hash where it first appeared, the test name, and a one-line root cause. Without that triple, "pre-existing" is not an allowed disposition.
2. **Skips count; a run is `(PASS, FAIL, SKIP)`.** `N FAIL + M SKIP` is **INCOMPLETE**, not PASS. A skip means the behavior is *unknown*, never *correct*. Each skip carries an explicit, time-bounded reason (e.g. "needs 3 peers, this run had 2 — re-run with 3") and is run on the next cycle that satisfies its precondition.
3. **Same-format baseline is a hard gate before any cross-format claim.** A document making a cross-format claim MUST first show the same-format baseline is clean for the *same* setup. Cross-format numbers measured against an unclean baseline are unanalyzable (they mix the cross-format effect with the baseline bug).
4. **Closeout and memory entries carry the numbers.** Any artifact labelling work "clean/locked" carries the actual `(PASS, FAIL, SKIP)` triple, not just verbal characterizations — so a future session inherits "185/185 same-format, 13 cross-format root failures by class," not "cohort-locked."
5. **Cross-format hash-equality tests are explicit SKIPs.** Tests that compare two peers' content hashes directly (trie roots, version-determinism, xpeer-determinism) are single-address-space by design (V7 §1.2 / §1.2a). Under a **cross-home-format** pairing they MUST be explicitly SKIPped with the tracked reason "single-address-space test; cross-format pairing is experimental per V7 §1.5" — never silently failed and never silently passed. They run normally under same-format pairings.
6. **Decode the `.cbor` artifact, not its sha256.** A cohort closeout that asserts "byte-equal" by comparing the **sha256 of an emitted `.cbor`** across impls only proves the impls produced identical bytes; it does not prove the bytes are correct. The byte gate MUST also decode the artifact and assert structural invariants against the `.diag` (the source of truth): every typed-bytes field has the spec-mandated length (e.g. Ed448 `secret_seed` = 57 B per RFC 8032; the corpus's `0xAA×64` fixture pubkey = 64 B), and no normative-output field carries a placeholder string (`"TBD-…"`, `"PENDING"`, etc.). `grep TBD` on the `.diag` is not a substitute — the producer may have failed to substitute placeholders back into the `.cbor` even when the `.diag` is clean. This rule is the structural sibling of (4) above: where (4) requires the closeout *narrative* to carry the numbers, (6) requires the closeout *byte gate* to carry the decode. Reason: F16 — the v767 cohort's `.cbor` sha-locked on a file whose Ed448 seeds were 58 B (off-by-one), whose experimental pubkey was 63 B (off-by-one), and whose 12 Phase-2 `expected_*` fields were still literal `"TBD-COHORT-ROUND-TRIP"` text. Documented in the F16 agility-corpus `.cbor`-regen closeout.
7. **A verdict is the check-set actually asserted — never a proxy for it.** No run may carry a prior verdict forward, or declare an oracle "current," on the strength of a value that tracks *which checks ran* rather than *what they assert*. A published number MUST be **dual-anchored**: the oracle commit **and** a `check_set_digest` over the exact assertions in the run (count + content), both required; a match on only one is not a match. This bans the whole family in one rule — a `core_gate_fingerprint` that tracks categories not assertions, a `grep` over source, a harness's default-category `Result:` line, a green Part-A probe standing in for a Part-B certification: each is a proxy, and a proxy is never a verdict. Corollary (the census rule): a peer's status is `UNMEASURED` until the *core gate itself* ran against it — a harness that silently defaulted to another category did **not** measure the gate, and "no result" is `UNMEASURED`/`FAIL`, never inherited PASS. Reason: the 0.8.1 bucket-B census — `cc1970f` and `af8a582` shared a byte-identical `core_gate_fingerprint` while one build contained none of the four bucket-B vectors and the other contained all four; `oracle-bootstrap.sh`'s fingerprint-only short-circuit would have run the stale check-set over all 43 peers and published it as current. The same census found `sql`/`ruby`/`dart`/`pd` mis-classified from source greps and harness-default lines. Fixed at `c04d04c` (dual anchor required; carry-forward-at-same-fingerprint withdrawn). This is the run-discipline sibling of the CDN-corridor meta-rule: not validated until the *asserting* check exercises it.

---

## §4 Divergence handling

When the validate-peer diff reports disagreement on a vector, the response is structured:

| Pattern | Likely cause | Resolution |
|---|---|---|
| One impl differs from the other two | Impl bug at the named encoder corner | Open a bug against that impl; fix; re-emit; re-run diff. Vector stays in the corpus. |
| All three impls differ from each other | Spec ambiguity — none knows what the canonical bytes are | Tighten §§3–9 in the same pass; re-author the vector if needed; re-emit all impls. |
| Two-and-two split *(four+ impls only)* | Either spec ambiguity or split bug | Same as "all differ" — spec arbitrates; do not vote. |
| Every impl agrees but the bytes do not match what the spec says | Spec-encoder divergence (the encoder is wrong in the same direction across all impls — typically a shared upstream library defect) | Spec is canonical; fix the underlying library or shim every impl until it agrees with the spec. The fact that "everyone agrees on wrong bytes" is exactly the W2 latency pattern; the fixture is the audit trail. |

**No partial v1 publication.** v1 is locked only when every vector in the corpus is byte-identical across every conformant impl. A vector with unresolved divergence holds up publication. This is a feature — landing a "v1 except this one vector" fixture means the conformance gate has a hole at exactly the point an encoder corner was hardest to nail down.

**Bug attribution lives in the impl's repo.** Arch reports the divergence; the impl team triages the bug, lands the fix, signals back. Arch re-runs the diff. No cross-repo writes from architecture.

---

## §5 Versioning and growth

### §5.1 Corpus versioning

The corpus carries an integer version `v{N}` in the fixture filename. The `(spec-version, corpus-version)` pair is the conformance citation. Spec v1.5 + corpus v1 is the first pair; subsequent corpus versions are additive only.

| Change | Bumps corpus version? |
|---|---|
| Adding a new vector to an existing category | Yes (v{N} → v{N+1}; additive) |
| Adding a new category | Yes (additive normative scope) |
| Fixing a vector's `description` field | No (informative; can be a corpus-version errata or rolled into the next bump) |
| Removing a vector | No — vectors do not get removed; they stay in the corpus and continue to be conformance criteria |
| Changing a vector's `input` or `canonical` | Yes — but this is a spec-changing event (the previous canonical bytes were *the* correct bytes for that input under the prior spec; if they're now wrong, the spec changed) and goes through the normal proposal cycle |

### §5.2 Impl reporting

Impls cite the corpus version they pass:

> *core-go passes ecf-conformance-vectors-v3 under spec v1.5; full report at `<path>`.*

A report of "passes v3" means every vector in `conformance-vectors-v3.cbor` returns pass under Appendix E §E.3 semantics, run through the impl's encoder/decoder. Reports MUST cite a specific version; reports without a version are not conformance reports.

An impl passing v{N} does not automatically pass v{N+1} — new vectors may have surfaced new corners. The impl team runs the new vectors, fixes anything that surfaces, updates the report.

**A check that never contacts the peer is a `[self]` check, and a per-peer figure that hides one overclaims `[MUST; ruled 2026-08-09]`.** Some conformance checks take no client at all — they exercise the running implementation against the spec locally (a resolver, an ordering rule, a local-store invariant). They are legitimate and they stay in the suite; what is not legitimate is a **peer's** row scoring a PASS for behavior that peer was never asked about. A sibling could ship none of the rule and the row would not move.

- Every client-free check MUST be **declared** as such by the suite, rendered distinguishably (`[self]`), and carried as a machine-readable flag in structured output.
- Self-checks **stay in the suite total.** They are not removed. Removing them would delete real coverage from the record to make a labelling point, and would break every published comparison at once.
- **Both numbers are published.** The conformance citation form (`N·0F @ <oracle-commit>` with the P/W/F/S breakdown, per `AGENTS-STANDARD`) gains a **peer-attributable count** whenever the figure is stated per-peer: `1549 (1520 peer-attributable) · 0 F / 0 S @ <oracle>`. A bare total against a named sibling is no longer a complete report.
- The change is a **reporting** change, so it lands in **all impls in the same cycle** — a cohort where one repo reports peer-attributable counts and two report bare totals is worse than either convention applied uniformly.

*(Surfaced by `entity-core-go` 2026-08-09: 29 client-free checks across five categories, 9 of 20 in `encryption` alone, found by diffing the whole suite rather than fixing the one instance that bit them. This is the sharpest form of the "three-way green is not independence" class — not three implementations agreeing, but **one implementation counted three times**, on a rule whose entire justification is that implementations must not diverge.)*

### §5.2a Rules with no peer-observable surface

A normative rule can be genuinely unprobeable: it governs what a peer does **before** it emits anything, and no operation exists to ask it. §4.4 of `EXTENSION-ENCRYPTION` is the worked example — sender-side key resolution, with no peer-facing encrypt operation to interrogate.

**A `[cross-peer seam — MUST]` with no peer-observable surface MUST be gated by a pinned-input vector, in the same change that lands the MUST `[MUST]`.** Prose alone leaves it unenforceable in exactly the place the "cross-peer" label claims it matters, and the divergence stays invisible until a second implementation builds it. The vector is a shared row file of authored inputs and expected outputs that **each implementation runs in its own suite** — the crossing is the shared file, not a live peer. `EXTENSION-ENCRYPTION` §16.6 defines the required properties (coverage, order-independence, negative control, declared exclusions); reuse that shape.

Corollary, now general: **a check that cannot be made to fail has not been shown to measure anything.** Every guard added to such a vector needs an injected-fault control, including — especially — the guards that can never fire against a correct implementation.

### §5.2b Rules the suite has no way to *reach* `[added 2026-08-10]`

§5.2a is about a rule with **no observable surface**. This is its opposite and it is more dangerous: the surface is **built, correct, flagged, and documented** — and the harness has no way to turn it on. The absence of coverage is then invisible, and the scoreboard reads *covered*.

**Three instances surfaced in a single 2026-08-10 cycle, in three different repos:**

| Surface | Built | Why nothing reached it |
|---|---|---|
| `content_hash_format = SHA-384` (V7 §1.2) | `--hash-type sha384` shipped in all three CLIs, honored, documented | `validate-complete.sh` had no way to pass it. **Every conformance number this cohort has ever published was measured under exactly one `content_hash_format`.** |
| `EXTENSION-REGISTRY` §6a.9 live registration | three ops, three policy modes, `403 not_entitled`, `202 pending_review` | `peer-manager` has no `--issuer-policy-mode` passthrough; the surface cannot be started through the documented tooling |
| `DOMAIN-LOCAL-FILES` §8.3 containment | all six callsites defended in go/rust/py | the shared V4 probe drives **`read` only** — never `list` through a symlinked directory (the literal original defect), nor `write`, nor `delete` |

**The rule `[MUST]`.** A normative rule that enumerates **N callsites, N operations, or N values of a configuration axis** is covered only when the suite exercises it **per callsite, per operation, per value**. One probe against one arm of an enumerated MUST is a **sample, not coverage**, and MUST NOT be recorded as closing it. `DOMAIN-LOCAL-FILES` §8.3 is the canonical shape: *"All path-resolving callsites MUST apply both defenses — `read`, `write`, `list`, `delete`, the watcher's ingest, and the reverse-write / reverse-delete handlers"* — six callsites named in the spec, one probed.

**The audit, and it is nearly free.** *Every peer flag with no harness counterpart is an uncovered axis.* Diff the peer binary's flag set against what the harness can pass:

```bash
# the cheapest audit available — a flag the suite cannot set is a surface it cannot reach
comm -23 <(peer-flags | sort) <(harness-passthrough-flags | sort)
```

**This SHOULD be a test in each implementation's suite**, not a discipline anyone remembers to run — a discipline is what failed three times here. Where a flag is deliberately unreachable, record it as a declared exclusion (§5.2a's shape), so the gap is *stated* rather than *absent*.

**Why per-check review cannot find these.** Every component is individually correct: the flag is real and honored, the peers start, the handler works, the defense is implemented. The failure exists **only in the combination**, and nothing in any single check is wrong. No amount of reading finds it — **only turning the knob does.** That makes this the conformance-side twin of the `SPECIFICATION-FORMAT.md` §8.4.5 width lock (invisible while one value ships) and of §11.5.1's loopback-blindness (green for a peer that cannot traverse). Same shape, three layers.

#### §5.2b.1 The second sub-shape — *no knob exists* `[added 2026-08-12]`

The three instances above are all *"the harness cannot turn a knob that exists."* The harder sub-shape is **no knob exists at the wire**: the state a rule describes is not constructible by a conformance client at all, whatever the harness passes.

**Canonical instance.** `EXTENSION-NETWORK` §5.4a's negative half requires a peer *unbound yet still `connected`* — reachable only from the paths §A1's scope deliberately excludes from demoting, all of which need an optional extension deployed. **The scope pin is exactly why the state is unreachable**, so the rule and the obstacle have one cause.

**The authoring rule `[MUST]`.** **Before a vector is pinned MUST, the state it requires MUST be shown constructible by a conformance client.** Where it is not, the spec **states the satisfaction mode at the point of the MUST** — typically: wire-driven where the enabling surface is installed, in-process with a **declared exclusion + the mutation it was verified against** otherwise. A vector pinned without this check is a rule that cannot be discharged, which the next implementer discovers instead of the author.

*This one was authored by architecture and caught by an implementation the same cycle — §5.4a invoked the "not validated until a cross-impl vector exercises it" meta-rule and then pinned a vector that could not exist.*

**The mutation MUST be executed and dated, not described `[MUST]` `[strengthened 2026-08-12]`.** A declared exclusion names the mutation the in-process test was verified against. **Naming it is not enough: run it, and record the date and result.** *A declared exclusion whose mutation is only described is an honest zero that nobody has checked is still zero* — it decays exactly like a build-state claim, silently, the first time a refactor makes the test pass with the guard removed.

*Adopted from the implementation that drew the distinction, and their reason is the argument: in the prior cycle their control **and its mutation test** both ran against a malformed probe, so **neither could have failed**. A mutation that is asserted rather than run is the same class of evidence as a conformance number carried forward across a commit change — plausible, previously true, and unverified.*

**Rejecting the proxy is usually right `[MUST]`.** The tempting fix is a nearby reachable state. **A proxy that cannot fail the way the real case fails MUST NOT be recorded as covering it.** In the §5.4a instance the proxy — a live idle counterpart — stays *bound*, so it exercises a timer-driven escalation and never the scope-pin defect the vector exists to catch. **It reads as coverage while missing the case, which is worse than a declared exclusion**, because §5.2b's entire failure mode is a scoreboard reading *covered* over an unreached surface. **A declared exclusion is an honest zero; a proxy is a false one.**

#### §5.2b.2 Audit the extractor, not only the checks `[added 2026-08-12]`

**An assertion is only as reachable as the plumbing that feeds it.** A harness helper that harvests a value under an assumption the spec does not share makes every downstream assertion **unpassable against any peer, however conformant** — and the failure presents as a sibling bug, so the hunt starts in the wrong tree.

**Instance.** A cohort validator's response extractor harvested an error `code` only when `status >= 400`, encoding *"codes ride failures."* `EXTENSION-REGISTRY` §6a.9 pins a code on a **2xx** row (`202 pending_review`), so that code was silently dropped to `""` and the assertion could not have passed. **Moving the gate to `>= 300` did not fix it** — 202 is still excluded. **The status class was never the right discriminator; the result's *type* is.** Gate on the response body being error-shaped, keeping the status class only as a fallback trigger so a `>= 400` response with a non-error body still reports something rather than nothing.

**The rule `[MUST]`.** When auditing checks against a pinned table, **audit the extractor that feeds them in the same pass.** Anyone who audits assertions without auditing their plumbing will write assertions that cannot pass and then go hunting a peer bug that is not there. **Both the original error and the insufficient first fix belong in the source comment**, not quietly corrected — the near-miss is the part that transfers.

**Run the audit against the extractor's *shape*, not against the one bug you know `[MUST]` `[extended 2026-08-12]`.** When this rule was first applied cohort-wide, **not one of three implementations came back empty, and no two had the same mechanism**: one gated on **status class**, one gated on **field name** (harvesting `result.data.code` only, so its own 2xx-carried status value recorded as absent), and one **dropped the result body entirely on one transport** at the handshake — thirteen sites across seven files, so every coded refusal a responder built arrived at the dialer as a bare number. **The class is "the extractor's assumption is narrower than the spec's surface"; the mechanism varies per tree.** An audit that greps for the sibling's specific bug will pass while the local variant survives.

**And audit the paths that BYPASS the extractor `[MUST]` `[added 2026-08-12]`.** This rule as first written said *audit the extractor that feeds your checks* — which **cannot see a check that routes around it.** A check driving a raw socket, a hand-rolled transport, or a bespoke decode path has its own extraction inline, unaudited and invisible to exactly the sweep prescribed above. *(Named by an implementation whose one check asserting a handshake refusal code drove a raw socket for that reason.)* **Enumerate the checks that do not use the common extractor and audit each one's inline extraction individually.** A bypass is not an exception to the rule — it is an instance of it with no shared code to fix.

**Where the doctrine points an agent, the blindness compounds.** In one tree the blind extractor was also the tool that repo's `AGENTS.md` sends agents to **first** on a cross-impl failure. **An instrument named in a doctrine is load-bearing beyond its own checks** — it shapes where every future investigation starts, so a gap in it sends the next several investigations down the wrong path before anyone questions the tool. Audit doctrine-referenced instruments first.

### §5.3 Growth triggers

Per Appendix E §E.5, two things trigger growth:

1. **A new spec feature introducing a new CBOR construct** (a future-whitelisted tag, a big int, a new container) lands with corresponding vectors in the same change. Mandatory.
2. **A new cross-impl divergence found in production** — i.e., a wire bug surfaced by live cross-impl matrix runs that the fixture didn't catch — MUST be reduced to a vector and added to the corpus. The vector is the artifact-of-record; the bug report points at it. Mandatory.

Discretionary growth (covering corners we've thought of but haven't hit) is fine and encouraged; it's the conformance discipline that prevents the W2 pattern.

### §5.2c A flaky check is a surface reached *by accident* `[added 2026-08-12]`

The fourth member of the family, and the one that disguises itself best. §5.2a is a rule with **no** observable surface; §5.2b is a surface the suite **cannot reach**; §2.4a is a surface the suite reaches and **scores backwards**. This is a surface the suite reaches **only sometimes, for reasons unrelated to the rule** — and the intermittency is the *only* thing that draws a human's attention to it.

**The rule `[MUST]`.** **A flaky conformance check is not noise to be stabilised. It is a defect whose reachability is accidental.** Before a flaky check is quieted, widened, retried, or marked known-flaky, two questions MUST be answered in writing:

1. **What makes this only sometimes-reachable?** A mechanism, not a hypothesis. *A jitter theory predicts a spread; it does not predict 6 s or never.*
2. **Is the same state deterministic somewhere else in the cohort?** If it is, the flake was never the defect — it was the instrument.

**A flake is closed by explaining it, never by making it stop.** Widening a margin converts a real conformance failure into a permanently invisible one, and the widened check will then certify the defect for every implementation that has it.

**The inversion, which is the reason this section exists.** One implementation's §5.4a escalation defect surfaced as a **1-in-4 flake** because two code paths raced there. A sibling had **no race** — its transport seam always won — so the identical spec defect was unreachable **100 % of the time**: deterministic, silent, and green. **The flakiness was the lucky version: a check that fails sometimes gets investigated; a check that never fails gets believed.**

Two corollaries follow, and both cut against ordinary triage:

- **Flake rate measures the local implementation's accidental structure, not the defect's severity.** The same defect is 25 % visible in one tree and 0 % visible in another. Ranking work by flake rate ranks it by luck.
- **A green sibling is not evidence that the flaking implementation is uniquely broken.** It may be evidence that the sibling *cannot observe* the defect it also has. **"Two impls green, one flaky" is equally consistent with three defective implementations and one accidental instrument** — which is exactly what it turned out to be. Investigate before converging, and treat the flaking tree as the one holding the evidence.

*This is `ADR-0012`'s "conformance-green ≠ correct" in its sharpest form: not a test asserting the wrong thing, but a correct test whose ability to fail is an artifact of one implementation's internals. Raised by an implementation, not by review — the cohort's own §5.2b twin, one layer down.*

---

## §6 Vendoring into keystone

The keystone repo (`entity-core-keystone/`) vendors a byte-identical copy of the canonical fixture for its language-binding generator. The discipline:

1. Arch commits `conformance-vectors-v{N}.cbor` (+ `.diag`) at the canonical path: `core-protocol-domain/specs/test-vectors/ecf-conformance/`.
2. Keystone copies the `.cbor` to `protocol-generator/shared/test-vectors/v{N}.{spec-version}/conformance-vectors-v{N}.cbor`.
3. Keystone records the SHA-256 of the vendored fixture in its `MANIFEST.md` alongside the spec-data snapshot manifest.
4. Keystone's CI verifies on every run that the vendored `.cbor` SHA-256 matches the canonical-path file's SHA-256. Drift fails the build.
5. When arch bumps the corpus, keystone re-vendors in a single commit; the manifest update is the audit trail.

This is the only cross-repo flow. Arch writes to its own repo; keystone reads from arch's repo and writes to its own. No cross-repo write authority either direction.

---

## §7 The conformance-surface map (the full picture)

Wire conformance (§§1–6) is one of several surfaces. A peer's conformance claim is `(spec-version, V7 §2.11 level, extension set)` — what it must clear scopes to that. The surfaces:

| Surface | What it proves | Normative home | Verified by | State |
|---|---|---|---|---|
| **Wire** | canonical ECF bytes, content-hash, peer_id, signature, envelope | `ENTITY-CBOR-ENCODING.md` App E | the offline fixture corpus (§§1–6) | ✅ canonical, cross-blessed, vendored to keystone |
| **Connect / auth** | handshake PoP (nonce/sig/identity-binding), auth-status boundary, capability verification, chain/attenuation, multisig | V7 §4, §5 (incl. v7.60 multisig, v7.61 PoP) | `validate-peer` `connectivity` + `security` + `capability` + `multisig` categories | spec ✅ landed; **oracle Go-coupled — §9 roadmap** |
| **Per-extension behavioral** | TREE history/snapshot-hash, NETWORK routing/framing, ROLE grant chains, REVISION metadata ordering, etc. | each extension spec | `validate-peer` per-extension categories | grounded but **self-skip + Go-convention defects — §9** |
| **Live peer matrix** | integration bugs (handler routing, identity resolution, cross-peer convergence) fixtures can't reach | the specs under test | `validate-peer -peers <addrs>` live | complementary to fixtures, not a substitute |
| **Extensibility hooks** | `register`/`unregister` behavioral presence (§6.13(a)); handler-initiated outbound seam (§6.13(b)/§6.11) | V7 §6.13; **attestation home = §7a here** | register-contract = wire (`validate-peer`); body-dispatch + outbound = `system/validate/*` conformance handlers OR code-attestation | **§7a — resolves A-011 (compute-coupling) + A-013 (no wire trigger)** |
| **Impl-internal** | non-wire-visible correctness | n/a (impl's own) | each impl's unit/property/fuzz | impl's own concern; wire conformance doesn't exempt it |

**Conformance levels (V7 §2.11).** A Level-0 peer publishes no extension types and must clear only the wire floor + the core connect/auth surface; a peer at higher levels additionally clears the behavioral surface of each extension it declares. The oracle must scope its assertions to the declared level + extension set rather than demanding the Go reference peer's full type/extension union as "core" (the §9 S5 defect). The genuine core type set (~30 entries) is arch-owed (§9).

**The meta-rule (why a surface isn't done until a test exercises it).** A normative claim about what bytes a route returns / what a handshake rejects / what a chain denies is not *validated* until a cross-impl conformance test exercises it. Prose review does not catch these — it's exactly what let hello-negotiation (§4.5) sit unimplemented across all three impls, and the Python identity-binding gap ship. Every landed surface above earns its place only when a `validate-peer` category (or a fixture vector) drives it hostilely.

---

## §7a The conformance test-handler surface (`system/validate/*`)

**The problem this solves (the A-011 + A-013 root).** Two extensibility hooks — `register`/`unregister` behavioral presence (V7 §6.13(a)) and the handler-initiated outbound seam (V7 §6.13(b)/§6.11) — are core-floor MUSTs, but **wire-testing them requires installing a minimal handler and driving it**, and core *deliberately fixes no handler-body vocabulary* (V7 §6.6; SDK-OPERATIONS §11.6: binding a callable body is "the one thing the protocol cannot specify"). That one gap is what made Go's §10.1 gate assume `compute/literal` (the **A-011** compute-coupling) and left §10.2 with no *obvious* wire-reachable trigger (**A-013** — every fresh-dial origination driver, continuation/subscription/compute, is an extension; the non-obvious surface that *does* work is §6.11 reentry-to-caller, §7a.2a). This section is the home for the resolution. **No core-protocol change; `compute/literal` is removed from the gate** (the compute extension is too specific to pull into a core gate).

### §7a.1 The two test handlers (behavioral contracts, not wire formats)

Both use `primitive/any` params/results, ECF. Mechanism, reentry model, and contracts proven cross-impl by keystone over real two-peer TCP (C#/TS/OCaml) — per the keystone conformance-handlers-mechanism handoff.

**`system/validate/echo` — operation `echo`** — proves §6.13(a) resolve→dispatch (replaces the A-011 `compute/literal` round-trip with a native, compute-free body).
- **params**: `{ value: <any> }`
- **result**: the params entity **verbatim** (`result.value == params.value`)

**`system/validate/dispatch-outbound` — operation `dispatch`** — proves §6.13(b)/§6.11: the target **originates**, not just responds. No continuation/INSTALL/subscription/compute.
- **params**: `{ target: text` (pattern to invoke **at the caller**, e.g. `system/validate/echo`)`, operation: text, value: <any>, reentry_capability, reentry_granter, reentry_cap_signature }` — the last three are the caller-minted authority for the reentry direction (this peer → caller); carriage convention is the one open Go-ruling item (§7a.2a).
- **behavior**: originate **exactly one** outbound EXECUTE via the §6.11 reentry sender → `operation` on `target`, **back to the caller over the same inbound connection** (see §7a.2a) → await the response.
- **result**: `{ status: uint, result: <downstream result entity> }`

These are **behavioral contracts**, satisfied by a **native handler each impl supplies** (validator drives them black-box over the wire). They are *not* a declarative body vocabulary the peer interprets — that would re-open the body-evaluator question and re-couple to compute. The guide specifies *what EXECUTE must do*; the impl provides *how*.

**Shape clarification (post §7b matrix — per the concurrency-gate §7b matrix rulings):** `echo` returns the params **entity** (`{value: X}`), not a bare scalar — `result.value == params.value`. `dispatch-outbound` is a **generic relay**: its `result` field carries the downstream handler's **result entity verbatim**, with **no unwrapping** (it relays arbitrary `target`/`operation` and cannot assume the downstream's shape). So for the echo round-trip the value rides as `result.value` (an entity), and a probe MUST assert `result.value == sent`, **not** `result == sent`. A relay that unwraps, or a probe that expects a bare scalar, is the non-conformant party — not a peer that returns the entity.

**Value-passthrough (pin, post 6-peer matrix):** the relayed `value` IS the params data, passed through as `primitive/any` — **`dispatch-outbound` MUST NOT re-wrap it** (e.g. `{value: value}` so `result.value` comes back a *map*). All six generated peers independently re-wrapped, latent under the value-blind §7a single-call probe and surfaced only by §7b's stricter assertion — *cohort agreement ≠ conformance*. The relay passes `value` through unchanged; the downstream handler's result entity is returned as `result` verbatim. (Note: arch's ruling-#2 prediction "cohort already conformant, no ×6 change" was **wrong** — there *was* a cohort-wide ×6 re-wrap bug; the §7a *principle* held, the cohort-conformance *prediction* did not. Don't predict cohort conformance from the reference impls.)

### §7a.2 Load mechanism — DECIDED (keystone's call, cross-impl)

**A runtime builder/constructor opt-in, surfaced as a host `--validate` switch, OFF by default.** Not a build-tag/feature matrix.

- **Library/SDK surface — exactly one new public thing: the opt-in.** C# `new Peer(conformanceHandlers: true)`; cohort `peer.WithConformanceHandlers()` / functional-option / builder method; OCaml `?conformance` on `create`. The handler bodies stay **internal**; the §6.13(b) outbound closure they call already exists in core (F2). No public handler-registration API is required — the opt-in calls the same internal native-handler install path the bootstrap handlers use (the five §6.2 writes).
- **Host/CLI surface:** `--validate` on the runnable host sets the opt-in. The harness flips one flag — no separate build artifact, no build matrix. (An impl that *additionally* wants physical exclusion MAY gate the bodies behind a native build flag *under* the opt-in — optional hardening, not the contract.)
- **Why runtime opt-in, not a build flag:** a build-tag/feature matrix is the *least* consistent option across C#/TS/OCaml/Go/Rust (tags vs features vs dune profiles vs bundle splits) and forces a second artifact through the pipeline. A runtime opt-in is one code path, identical everywhere.
- **Why OFF by default:** `system/validate/dispatch-outbound` originates outbound — it must never be a live standing dialer in production. A default peer 404s `system/validate/echo` (proven: `ConformanceHandlers_AreOffByDefault`). `echo` is dead surface in production.
- **Absence is honest:** a peer run without the opt-in → the wire gate **SKIPs** (falls back to the §7a.4 code-attestation floor). No false FAIL.

### §7a.2a The reentry surface + the one open Go-ruling item

**The substantive finding (keystone, proven over real TCP):** the wire-reachable origination surface in a core-only peer is the **§6.11 reentry seam**. While servicing an inbound EXECUTE, `dispatch-outbound` originates an EXECUTE **back to the caller over the same connection**, and the caller — running its own dispatcher on that bidirectional connection — services it. **The validator plays B-role on that same connection** (its `system/validate/echo` answers the reentrant call). This is genuine A-role origination (target builds, signs, sends, consumes the response) and it is exactly §6.13(b). 

This corrects the A-013 bounce-back's "no wire-reachable outbound surface" conclusion: a **fresh dial to an arbitrary third peer** genuinely has no core surface (Go was right about *that*) — but **reentry to the caller does**. So `dispatch-outbound` is a **real black-box wire gate**, not the code-attestation fallback. No V7 change — §6.11 reentry was always the surface; this names it as the attestation path.

**The one thing still to ratify (Go's call — Go builds the validator side):** how the caller hands the reentry capability to `dispatch-outbound`. The reentry direction can only be authorized by the caller (a cap valid *at the caller*). Two shapes: **(a) in-band, nested in params** (`reentry_capability`/`reentry_granter`/`reentry_cap_signature`) — self-contained, transport-agnostic, no reliance on an included-set convention; or **(b) the envelope `included` set** (how caps normally travel) with a hash reference in params. **All six-impl-leaning + arch-leaning: (a) in-band params** — all three keystone peers already implement it, and it doesn't depend on the high-level session API exposing the `included` set. Go ratifies (or flags (b) — then it's one uniform ×N change, not divergent rework). Secondary confirm: the validator drives `target` as **itself** (B-role on the same connection), not a third-peer dial.

### §7a.3 What stays a pure wire test (no body needed) — the register *contract*

The register/unregister *contract* (§6.13(a)) is wire-tested **independently and needs no body**: the validator calls `register` then `unregister` and asserts the five normative writes appear at their spec paths (manifest, types, grant, **grant-signature at `system/signature/{grant_hash}`** per §3.5, interface) and that unregister removes them with writer symmetry. This is the actual "not 501-stubbed" gate and is fully portable. It is **separate** from the §7a.1 `echo` dispatch test — registering a handler over the wire proves the *declarative* side; `system/validate/echo` being dispatchable proves the *resolve→dispatch* side. The old compound "register a wire-supplied body then dispatch it" step (which forced compute) is **dropped** — its two halves are now tested separately and portably.

### §7a.4 Two attestation tiers

| Tier | Mechanism | When |
|---|---|---|
| **Floor — code-attestation** | impl ships its own seam-presence unit test (the `OutboundDispatchTests.cs` shape); conformance matrix row marks the hook "code-attested," with a pointer to the in-impl test. Wire gate SKIPs. | always available; the answer when no conformance build / native handler is present |
| **Wire-attestation (primary)** | validator EXECUTEs `system/validate/echo` and `system/validate/dispatch-outbound` against the `--validate` peer, playing **B-role on the same connection** for the reentry leg (§7a.2a); asserts the contracts | when the peer is run with the opt-in |

Wire-attestation is now a **real black-box gate for both hooks** (the §7a.2a reentry finding makes outbound wire-testable, not just code-attested). It is the only black-box signal for core-only peers — they ship no origination extension, so reentry is the sole surface. Code-attestation drops to the **floor** for peers run without the opt-in. Per the GUIDE-INSPECTABILITY §1 precedent ("why a guide and not an extension"): conformance scaffolding is an operational concern, not a protocol-mandatory feature — it lives here, not in V7 core and not as an extension.

### §7a.5 Ownership split (so this doesn't ping-pong)

| Layer | Owns |
|---|---|
| **Core protocol (V7)** | `register`/`unregister` (§6.13(a)), dispatch (§6.6), outbound seam (§6.13(b)/§6.11) — all already mandated. Body vocabulary deliberately open. **No change.** |
| **GUIDE-CONFORMANCE (here)** | the `system/validate/{echo,dispatch-outbound}` contracts + the two-tier attestation + the §7a.3 register-contract split. The single source of truth. |
| **SDK domain** | `register_handler(spec, body)` binds the native body — the portable "dynamic handler registration" pattern. The two contracts are reference bodies bound this way. |
| **Each impl + keystone generator** | the two native handlers behind the `--validate`/builder opt-in (§7a.2). **Keystone: DONE — C#/TS/OCaml all GREEN, off by default, `--profile core` 568/0F unchanged.** Cohort (Go/Rust/Py peers) owe the two handlers. |
| **`validate-peer` (Go)** | cap-passing ruled **(a) in-band params** (`af81897`/Go `9c624aa`); EXECUTEs the two handlers when present, driving `dispatch-outbound` as B-role-on-same-connection (SKIP when absent); dropped the `compute/literal` round-trip from §10.1. **A-013 disposition: wire-attestable via reentry — code-attestation no longer the fallback.** ✅ landed + 3-peer GREEN. |

### §7a.6 Coverage model — one seam, two resolution paths (RULED)

Go's close raised the right question: `dispatch-outbound` exercises **caller-directed** origination (the caller's params name the target), which resolves via the §6.11 inbound-reentry path. **Handler-autonomous** origination (a handler dispatches from its *own* state — subscription `deliver_to`, continuation advance, timer, configured fanout) uses the *same* `hctx.Execute` seam but resolves via transport-profile lookup + fresh dial. Should `--profile core` gate the autonomous path too?

**Ruling: no — "one seam, the core gate attests it via reentry" is the intended model. The split-by-profile is correct; pinning it here makes it explicit.** Reasoning (first principles, not convenience):

1. **§6.13(b) mandates a *seam*, and the seam is one mechanism.** `hctx.Execute` (build → sign → send → await by `request_id`) is identical for both patterns. Once the reentry gate is GREEN, the seam is verified. The branch is in *connection resolution* (`getRemoteConnection`), not in the seam.
2. **Autonomous origination has no core *trigger*.** Every autonomous example needs an extension to fire it — subscription (`deliver_to`), continuation (advance), timer/fanout (config). Under `--profile core` there is no trigger to originate from autonomously, so there is nothing coherent to gate. A `--profile core` autonomous contract would have to invent a trigger, which is exactly the compute/continuation coupling §7a exists to avoid.
3. **The fresh-dial *resolution* path needs transport-profile resolution (NETWORK §6.5) — legitimately `--profile full`.** Reentry-to-caller needs no resolution (the connection already exists); that's why it's the core-reachable path. Fresh-dial-to-an-arbitrary-peer drags in the network machinery, which lives under full.

So: **core gates the seam via the resolution-free (reentry) path; `--profile full` gates the fresh-dial resolution path + autonomous triggers, where the extensions that provide them live** (`checkRemoteExecute` + cohort interop). No second core conformance contract. The **niche middle case** Go flagged (autonomous origination *on* an inbound connection — e.g. subscription `deliver_to` to a dialed-in subscriber) is covered without new work: its *resolution mechanics* (the §6.11 inbound-reentry fallback) are already proven by the core gate, and its *trigger* (subscription) is exercised by the subscription extension's own `--profile full` conformance. It belongs to subscription-extension conformance, not a core contract.

---

## §7b Concurrency conformance (§6.11) — LANDED 3-way GREEN

The conformance gate proved wire-correctness but never drove the **§6.11 concurrency MUSTs** (no per-connection serialization; out-of-order reply tolerance; concurrent outbound on a pooled connection). Those were validated by per-impl unit tests, never by the cross-impl oracle — so a generated peer could pass everything else and still serialize/deadlock/mis-demux. Closed via the conformance-concurrency-gap finding and the concurrency-conformance-contract draft. New `concurrency` category in `validate-peer` (`cmd/internal/validate/concurrency.go`, ~480 LOC). **Go/Rust/Py all 5/5 PASS** (Go `0623bec`+`bcc52a6`). No bug surfaced — the reference impls were already correct; the point is every *future* generated peer (keystone C#/TS/OCaml, Elixir, OCaml-FFI) is gated **before ship**.

**Two design principles (held):**
1. **Zero impl work if architected right** — a correct §6.11 peer passes unchanged; the work was the validator's multiplexed client (already in tree from the C# leg-3 desync fix — so the predicted "~10 Send* call sites" was a *stale audit note*; real Phase-0 was an atomic `requestSeq` + a doc-comment). A FAIL = a real bug caught pre-ship.
2. **Ratios/invariants, not absolute numbers** — a slow language fails only for being *wrong*, never slow. Speed stays Tier 3 (out of floor).

**The checks (5, tunable consts at the top of `concurrency.go`):**
| Check | Drives | Catches | Knob |
|---|---|---|---|
| **T1.1** demux | N=16 concurrent EXECUTEs / one conn | **demux correlation = hard gate**; **ratio = informational/WARN** (per `RULINGS-CONCURRENCY-GATE-7b-...` #3 — parallel *speedup* is NOT a §6.11 MUST; a single-threaded cooperative-async runtime that demuxes + passes T1.3 is conformant). The §6.11 serialization MUST is enforced by T1.3 + demux, not the ratio. | N=16, ratio 0.7× (WARN-only), 50 ms noise floor |
| **T1.2** reentry-under-load | M=8 concurrent reentrant `dispatch-outbound` | bidirectional-reentry deadlock (the F-WB28 class), cross-impl | M=8 (raise for sharper stress) |
| **T1.3** head-of-line | slow+fast concurrent | fast gated behind slow (§6.11(a)) | trials=10, headroom 4× |
| **T2.1** sustained load | 16 workers × 10k reqs | drops, crashes, latency runaway | p50 ratio 4× |
| **T2.2** connection churn | 100× connect→1 req→close | per-connection resource leak (black-box) | 100 cycles (→1000 if late-cycle degradation) |

T1.2 rides `--validate` (reuses the §7a `dispatch-outbound` handler — honest-SKIP without it); the other four use core ops → run on **every** peer. **T2.3** (internal-leak via inspect surface) stays optional/deferred.

**6-peer outcome: all six generated peers GREEN (573/0F).** The gate paid off exactly as designed — it caught real **fall-over** bugs the functional suite cannot see:
- **Store concurrency-safety = a §6.11 requirement, NOT an optimization.** Zig and CL independently accessed their content/tree stores from per-request dispatch threads with **no synchronization** → CL raced `gethash` (500s), Zig raced its HashMaps (**double-free panic**, masked by the slow thread-safe allocator). A bare-map store **passes functional conformance but fails §7b**. Conformant fix: serialize / RwLock / actor-mailbox (Elixir's GenServer gets it free; C#/TS/OCaml were already safe). **Generator-menu requirement**: the store MUST be data-race-safe under concurrent per-request dispatch. **NORMATIVE as of v7.75 — V7 §4.8 store-safety paragraph + §9.1 floor** (a data race = crash = falls over); the T2.1/T2.2 fall-over outcomes are now the V7 §4.9 resilience floor. See the V7.75 nonfunctional-substrate-floor proposal.
- **TCP_NODELAY belongs in the transport profile-menu** for every raw-socket peer. Zig's raw sockets defaulted Nagle-on → ~40 ms delayed-ACK per small frame → churn 343 ms/cycle, concurrency 62 s. `setNoDelay()` → 1.9 s. A request/response protocol with small frames MUST disable Nagle; managed-runtime peers happened to avoid it, the next raw-socket peer (manual-C, Zig-class) would re-hit it.
- **Blocking syscalls MUST NOT run on a bounded structured-concurrency cooperative pool** (Swift peer #7). Swift's `async`/await scheduler runs on a small fixed cooperative thread pool; putting blocking `read()`/`accept()` on it starved the pool under load → 60 s, fixed to 3.0 s by moving the socket I/O to dedicated OS threads. Same *outcome* class as the §4.9 resilience floor (a starved pool stops staying responsive = falls over), same *shape* as the Zig Nagle trap (a language-default runtime behavior the generator must steer around). **Generator-menu requirement** for any cooperative-scheduler / green-thread runtime (Swift structured concurrency, and watch for it in async-Rust/Node-class peers): blocking socket calls go on OS threads (or use the runtime's non-blocking I/O), never on the cooperative pool. The §7b T2 checks catch it (Swift hit 573/0F only after the fix). *Corroboration of §4.8 store-safety:* Swift's actor-isolated store made the Zig/CL store-race a **compile error** under Swift 6 strict concurrency — actor/mailbox is one of the §4.8-listed safe mechanisms, working as intended.

**Follow-ons:**
- **§9.0 fold (v7.75): DONE.** `concurrency` is now enumerated in V7 §9.0's `coreProfileCategories`. Go flips its oracle carve-out → enumerated in this cohort cycle (paired with the V7 edit per the §9-alignment standing audit, so the §10.3 drift gate stays GREEN). Per the V7.75 nonfunctional-substrate-floor proposal.
- **`resource_bounds` probes (v7.75 §4.10): LANDED 3-way GREEN; §4.10(a)+(b) folded as floor.** Keystone's cross-impl audit found 6/6 peers already enforce max-payload (16 MiB), 4/6 enforce max-chain-depth (64), 0/6 cap connections — so (a)+(b) **ratified observed convergence** (now §9.1 floor MUSTs) and (c) is **SHOULD** with an external-layer carve-out (admission = systemd/proxy/OS). Go shipped the probe (documented in the V7.75 resource-bounds cohort-close validation report): over-`max_payload` → `413 payload_too_large` + keeps-serving (PASS); over-`max_chain_depth` → **`400 chain_depth_exceeded`** (NOT 403 — that conflated too-deep with unauthorized) + keeps-serving (PASS); connection flood → **SHOULD, WARN-not-FAIL** (PASS-with-Warn). Go/Rust/Py all GREEN; `--profile core` 0 FAIL. `resource_bounds` is in V7 §9.0 `coreProfileCategories`. Probe-authoring note: error-code extraction must fall back from strict ErrorData → generic CBOR map lookup (mirroring `format_agility.go::extractStatusAndCode`) so a `primitive/any`-wrapped error result still reads its `code` — same lenient-extraction lesson as the §7a value-blind probe. **Keystone re-vendors + re-runs the 6-peer matrix next** (Zig/CL each owe a ~5-line code-mapping).
- **T1.4 cross-peer concurrency — DEFERRED.** N peers hammering one peer is a *different* bug class (accept-loop / per-connection-state isolation, **not** §6.11 demux). Pick up if a future impl surfaces that shape.

---

## §7c Compute conformance: the differential corpus (guidance to converge — UNBUILT)

The **third conformance modality**. Wire ECF (§§1–6) is enumerable golden fixtures; concurrency (§7b) is
behavioral probes; **compute is *differential* over a combinatorial input space.** This is the conformance
artifact for **AE-1** (`PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION` — an alternate execution strategy is conformant
iff materialized-boundary-equivalent). **Status: no portable cross-impl compute corpus exists.** Today the
entire cross-impl compute-evidence base is the single `compute/scope` hash-equality point
(`EXTENSION-COMPUTE §447`) plus Go-internal differential tests — so "Rust eval == Go eval" is *assumed, not
verified*. This section is arch's guidance to the core peers to build and **converge** it; it is high-level by
intent — the vector shape is pinned, the construction is left to the cohort.

### §7c.1 Why a different modality (the §3.4 reconciliation)

Compute's inputs are arbitrary well-formed IR graphs — **not** enumerable, so hand-authored golden fixtures
(§2) cannot cover them. The corpus is therefore **generator-seeded, then frozen**: a deterministic seeded
generator produces the coverage; its output is captured as frozen `(IR, inputs, budget) → boundary-hash |
error` vectors that every impl runs. This does **not** contradict §3.4 ("generator-driven is for bootstrap;
the gate is cross-impl agreement") — the generator bootstraps the corpus **once**, and the **frozen vectors
are the cross-impl gate**, arbitrated by the spec exactly as §4 prescribes. Generator for coverage; freeze +
cross-bless for the gate.

### §7c.2 What already exists — build ON it, don't reinvent (surveyed 2026-07-21)

- **The Axis-1 differential harness** — `entity-workbench-go/entitysdk/axis1_equivalence_test.go`
  (`TestAxis1Equivalence_Differential`). A working **seeded generator** (`exprGen`: `seed=20260716`,
  `cases=300`, a *typed* recursive grammar over `ComputeBuilder` — signed+unsigned literals, arithmetic,
  `if`/`let`, `index`/`length`, cast, compare/logic, arrays with `map`/`filter` lambdas capturing an enclosing
  binding, error paths *on purpose*; every case wrapped in `Construct(...)` to force the boundary), a
  **boundary comparison** (both engines reduced through Stage-1's own `CaptureScope` → entity-kind = content
  hash, value-kind = canonical-CBOR hex, error = `code+message`), and **five anti-vacuity guards**. It is
  Go-internal and Axis-1-vs-Stage-1 (*intra*-impl). **This is the thing to port cross-impl.**
- **The wire-conformance pipeline** — `entity-core-go/cmd/internal/wire-conformance`
  (`.diag → .cbor → per-impl emit → cross-bless`, + the MANIFEST hash-pin + the S5 authoring split). The
  scaffolding and discipline are directly reusable; it needs **one new stage — evaluate + materialize + hash**
  (today it only does static encode/decode; a compute vector's "expected" requires *running an evaluator*).
- **The boundary in code** — `entity-core-go/ext/compute/{eval_construct.go::materialize, scope.go::CaptureScope}`.
  Already exactly the AE-1 boundary set; the corpus pins the **same hashes the harness already compares**.

### §7c.3 The vector shape (high-level — the only thing pinned here)

A compute conformance vector is `(IR entity, root bindings, budget) → { boundary-hash | error{code} }`,
where the **boundary** is the AE-1 set: the **materialized bare-entity content hash** (a `construct`, byte-
identical to the hand-built entity — V7 §1.4), **value-kind canonical-CBOR bytes**, and the **error `code`**
(compared strictly; `message`/`at`/`expression` are in-flight diagnostics, **not** part of the materialized
error boundary — `EXTENSION-COMPUTE §2.4`, so the gate's `code`-comparison *is* the materialized-error-hash
comparison). Portable by construction — IR + inputs + hash, no Go. Leave the exact serialization
(`.diag`-analog vs. a compute-native form) to the cohort; the schema above is the contract.

**Two build profiles; one locks.** An `inproc` profile matches the `(IR, root bindings, budget)` shape above
exactly and exercises the evaluator in isolation — it is the **normative, hash-pinned artifact** and is the one
that can satisfy §7c.4(2)'s alternate-engine-each-side guard. A `wire` profile inlines root bindings so the
corpus can be driven against a live peer (§3.2 evaluates from an empty scope); it exercises the handler path,
records `engine: "wire:<addr>"`, and **cannot** satisfy the alternate-engine guard — it is an auxiliary driver,
not the locked corpus. The two are different artifacts with different SHAs; **`inproc` is the one vendored and
locked.**

### §7c.4 Guidance to the core peers — converge the set

1. **Port the generator, not a static dump.** A frozen vector dump alone freezes *Go's* coverage. Each impl
   SHOULD be able to run the seeded generator (identical seed → identical shapes) so coverage grows cross-impl
   — but the **published artifact is the seeded frozen output**, so every impl runs identical bytes. **The
   generator's PRNG MUST be portable — SplitMix64 with a 64-bit seed** (corpus v1: seed `20260716`), never a
   language-runtime RNG (Go's `math/rand` is not reproducible cross-language, so "seed S" would mean a different
   corpus per impl). Identical seed → byte-identical shapes on every impl is the contract.
2. **The anti-vacuity guards travel WITH the corpus — as gates, not decoration.** Port the five verbatim:
   **≥25% value outcomes** (else agreement is mostly errors — vacuous); **≥1 error-as-value** (the code path is
   part of the contract); **≥1 closure built** (else `map`/`filter` / the live-frame path never ran); **0
   fallbacks** (a fallback compares an engine to itself — circular); **≥3 distinct error codes**. A corpus that
   cannot meet them is vacuous — the ECF-F30 / F-D3 "green-can-be-empty" lesson. **Add one guard the intra-Go
   harness didn't need:** every cross-impl vector MUST exercise the *alternate* engine on each side, never a
   fallback to the reference — else the cross-impl agreement is circular too.
3. **Cross-impl agreement is the gate; the spec arbitrates (§3.4 / §4 unchanged).** *Within* Go, Stage-1 is
   reference-by-construction for Axis-1. For the **cross-impl corpus, no impl is privileged**: the frozen
   vector's boundary-hash is fixed by the spec (AE-1 + the canonical encoding + the v3.19 value model), not by
   Go. Disagreement routes through §4 — one impl differs = its bug; all differ = spec ambiguity (the round's
   work product); **no voting**.
4. **Do not mistake the intra-Go green for cross-impl evidence (the honesty gate).** The differential harness
   proves Axis-1 == Stage-1 *within Go*; the corpus must prove **Go == Rust == Py** at the boundary. Report
   them as different claims (the AE-1 §7 ledger) — cohort-consistent ≠ independent convergence.
5. **The convergence sequence** (the F29/F30 cross-bless loop, extended for evaluation):
   (a) port `exprGen` → a seeded frozen `(IR, inputs, budget)` set — **start with the nine lowering-toolkit
   worked lowerings** (`PROPOSAL-COMPUTE-LOWERING-TOOLKIT §5`), then the random sweep;
   (b) run through core-go's evaluate+materialize+hash → candidate boundary-hashes (the new emit stage; Go is
   the fixture-builder, **not** the oracle — Appendix E.4 "spec arbitrates the bytes");
   (c) each impl (rust/py) evaluates the frozen `(IR, inputs)` → emits its own boundary-hash + error codes;
   (d) cross-bless — **lock only when byte-identical**; divergence → §4;
   (e) hash-pin the frozen corpus in a MANIFEST (like the ECF corpus SHA); vendor into keystone; the
   **compute-bearing peers re-run** at fold (the CDN-corridor meta-rule — not validated until the cross-impl
   cohort exercises it).
6. **Reproducibility discipline (§3.1 applies).** The corpus carries `(seed, case-count, generator-version)`;
   a failure reproduces from `(seed, case-index)`. Decode the artifact, not just its SHA (§3.1(6)); skips count
   (§3.1(2)); no "pre-existing" without a bisect (§3.1(1)).

### §7c.5 Scope, ownership & what it gates

**Gates:** AE-1 admission (fast interpreter *and* compiled handler), the lowering-toolkit vectors (first
tranche), W-BUDGET preemption determinism (BP-1), W-HOSTING transferable compute. Highest-leverage cohort
build (`ROUTING-2026-07-21` Wave 0→1). **`budget_exhausted` is a gated, cross-impl-deterministic outcome** —
the observable `operations` ceiling charges `evaluate()` steps only (`EXTENSION-COMPUTE §4.2`, ruled from this
corpus's first run), so budget-edge vectors (a workload deliberately partway through its budget) **stay in the
equality gate**: two conformant impls given the same `(IR, inputs, budget)` MUST reach `budget_exhausted` at
the same step. Impl-private scan-based DoS guards surface out of band (a distinct resource condition), never as
a shifted in-band `budget_exhausted`, so they do not perturb the gate. **Ownership:** **arch authors this guidance + the vector shape + the
discipline (here);** the **cohort** builds the generator port + the evaluate-emit stage + cross-blesses; the
frozen corpus + MANIFEST live in `entity-core-protocol/specs/test-vectors/compute-conformance/` (new dir),
vendored into keystone — the same split as the ECF corpus. Per the working pattern, arch **guides**; the impl
work lands in the cohort repos, not isolated into `entity-core-protocol`.

## §8 `validate-peer` remediation roadmap (the Go handoff)

`validate-peer` is a valuable, mostly spec-grounded oracle, but the comprehensive validate-peer audit found **five systemic defects, all tracing to one root: it encodes the Go reference peer's shape as the conformance contract.** That was invisible while only Go-cohort impls ran against it; the C# first-peer exposed it. This is the work to make the oracle spec-ordained and language-neutral — so a new peer validates against the *spec*, not against "what Go does." Owner: Go (oracle owner); arch supplies the spec-side inputs noted. These are **oracle fixes, not spec defects.**

| # | Defect | Fix | Priority |
|---|---|---|---|
| **S4** | `security` category blind to the chain/attenuation surface (where the F1–F6 cross-impl divergences live) | add the eight chain/attenuation probes (security register §3); pins F1–F6 cross-impl | **non-deferrable (security)** |
| **S1** | skip logic keys on Go's grant-coverage, not extension *presence* → spurious FAILs (not SKIPs) for a peer with a different extension subset | uniform `handler_manifest_present` 404 → SkipCheck for optional extensions; gate behavioral categories on their structural category passing. Copy `durability.go` (the model citizen) | high |
| **S3** | `ecf_key_ordering` validates against Go's `ecf.Encode`, not the spec (privileged-encoder anti-pattern §3.4 retracts) | `encoding` defers to the fixture corpus (§§1–6 path), not Go's encoder | high |
| **S5** | no conformance-level / `--core` mode; demands Go's ~85-type extension union as "core" | **✅ ARCH SIDE DONE (V7 v7.72)** — V7 §9.0 defines `--profile {core|full}`; §9.5 the 53-type floor; §9.5a six new `CORE-TREE-*` vectors. **Go owes oracle wire-up ~2 days** per the V7.72 core-profile cohort impl-team-alignment handoff — top-level flag at suite.go + per-category guard + three per-check carve-outs (security.go:735, authz.go:115, authz.go:408) + profile-aware `expectedHandlers["tree"].coreOps = {"get","put"}` + 53-type floor against §9.5. | **high — cohort target ~2 days** |
| **S2** | strict write-then-read; `request_id` generated but never demuxed → desyncs on any pipelining/reorder/unsolicited frame (caused the C# leg-3 "failure") | match V7 §6.11 request_id routing with out-of-order tolerance | medium |
| — | concrete bugs | session required/optional inversion (checks OPTIONAL `minted_capability`, never REQUIRED `held_capability`); revision/auto_version gate-path mismatch; `content.go` library-self-tests split out; Go-convention thresholds (75% chunk-reuse, error-substring) → WARN; spec-ref hygiene (V4→V7, proposal-§→landed-§); `decode_reject` assert the error code; `conformance_passthrough` reject → FAIL not WARN | rolling |
| — | the oracle itself was never security- or spec-reviewed | do both (F12 + S4 prove it has blind spots) | with S4 |

**Longer arc (the de-Go-coupling):** core-gate vs per-extension vectors (§7 map); **declarative expectations** (handshake legs, status tables, grant shapes) so the oracle is data the harness drives, not Go code asserting Go assumptions; language-neutral so any peer emits into it (the conformance-corpus model of §§1–6 already proves this shape for wire — generalize it).

**Sequencing for this round.** After the v7.60/v7.61 spec amendments land and peers update: Go updates `validate-peer` (new multisig assertions already exist; add the connect-auth/PoP probes per the handshake catch-up §6 and the S4 security probes), runs the cross-impl matrix, the cohort goes green, sign-off. **Capability handler landed V7 v7.62**; its conformance (validate-peer + this guide) updates in turn — `validate-peer` vectors for the new ops + status codes per §9; Rust + Python land `is_revoked` + `capability_path_for` impls as cross-impl follow-on.

## §9 Open questions / pending register

The single place "what's not finalized" is tracked, so nothing is lost while it settles. Each item names its disposition and where it lands when ruled.

| Item | Status | Disposition |
|---|---|---|
| **v7.73 closeout + Amendment 1 — validator vectors + §PR-8 plumbing across three surfaces** | **Resolved (V7 v7.73 + Amendment 1; 6-way PASS on all three §PR-8 surfaces)** | Proposals: the V7.73 closeout validator-and-OCaml-probes proposal (head) + the V7.73 Amendment 1 PR-8 chain-attenuation-and-handler-recheck proposal. Three v7.73-head validate-peer vectors + three v7.73-Amendment-1 V1'-family vectors landed cohort-wide and exercised on the keystone peers. **v7.73 head vectors**: **V1 `chain_mid_link_expiry_denied`** — security_chain.go; expect `403 capability_denied` on a chain with mid-link expiry; gates §5.5 per-link temporal walk; 6-way PASS. **V2(a) `captok_form_dispatch_minted_pl_presented_xpeer`** — load-bearing; mint a cap with peer-local resource pattern (bare `*`), present cross-peer; pre-fix expected behavior was 200 (under-enforcement, latent under same-peer-dominant test paths because granter byte-collapses on self-issued caps); post-fix expected 403 `capability_denied`; **all six reference impls FAILed identically pre-fix** (Go/Rust/Py/C#/TS/OCaml); cohort + keystone landed Rust kernel-only dispatch-boundary fix shape (resolve granter once at dispatch, thread into `check_resource_scope`; grant resources only — request target / operations / handlers / peers stay local); 6-way PASS post-fix. **V2(b) `captok_form_dispatch_minted_xpeer_presented_pl`** — reverse direction; informative branch (200 OR 403 both count); 6-way PASS. **V3 `int.15/16/17`** (ECF corpus) — large-uint boundary vectors; closes F7. **§5.2a verdict-to-status enumeration** (folded from the F14 auth-vs-authz status-boundary ruling) pins 20 request-time auth/authz rows that `validate-peer`'s `security` and `authz` categories enforce. **Amendment 1 V1'-family vectors** (security category): **`AUTHZ-ATTENUATION-FOREIGN-GRANTER-1`** — V1' standard 3-link chain (root R, mid A foreign with bare `*`, leaf B foreign with explicit cross-peer `/{verifier}/system/type/system/peer`); leaf admits at dispatch unconditionally so chain-walk subset-check is the deciding gate; expect 403 `capability_denied`; pre-fix all 5 non-Go impls FAILed at 200 (§3.1 canon-against-wrong-frame for Rust/C#/TS/OCaml; §3.2 no-canon-before-wildcard-shortcircuit for Py); post-fix 6-way PASS. **`AUTHZ-ATTENUATION-FOREIGN-GRANTER-DEEP`** — 4-link chain with TWO foreign mids; rules out first/last-link special-casing in the plumbing; same pre-/post-fix arc, 6-way PASS. **`AUTHZ-ATTENUATION-FOREIGN-GRANTER-WILDCARD-LEAF`** — 3-link, leaf is `/{verifier}/system/*` (wildcard); stresses wildcard-vs-wildcard subset arithmetic across peer namespaces; same arc, 6-way PASS. Per-impl plumbing: Rust threads `child_granter_peer_id` + `parent_granter_peer_id` through `is_attenuated`/`grant_subset`/`scope_subset_path` (only resources canonicalize per-side); Py canonicalizes resource includes per-side BEFORE `scope_includes_subset` (defeats bare-`*` short-circuit); keystone mirrors Rust shape with **hard-fail on identity-resolution failure** (preferred over cohort's silent fallback to `local_peer_id` — pre-empts the v7.74 follow-up `AUTHZ-MALFORMED-GRANTER-IDENTITY-1` at keystone layer). **V2 (`AUTHZ-CONFUSED-DEPUTY-PARAMS-PATH-1`) DROPPED** from vector set per cohort empirical finding — architecturally non-fireable in iterate-all-grants architectures (the only architecture in the 6-impl cohort); handler-internal re-check surface named in §5.5a spec text only, gated indirectly by V2(a) at dispatch. v7.74 backlog: `AUTHZ-MALFORMED-GRANTER-IDENTITY-1` (silent-fallback hardening), `AUTHZ-MINT-FOREIGN-GRANTER-SUBSET-1` (§6.2 mint-time subset check on foreign-granted caller cap — keystone surfaced this as the next per-link frame question; no vector gates it yet), future `AUTHZ-CONFUSED-DEPUTY-PARAMS-PATH-N` (gated on a grant-indexed re-check architecture appearing). 6-way commits: Go `87e6eaa`; Rust working tree; Py working tree; C# `b123808`; TS `c1a3153`; OCaml `1554a17`. **All three §PR-8 surfaces closed 6-way at the wire level.** |
| **Spec/oracle alignment** (standing audit, v7.72+) | **standing — runs every cohort closeout** | A v7.x amendment that touches V7 §9.5 (Core Type Floor Manifest) or §9.5a (Core Tree Operations Vector Set) MUST be paired with the corresponding entity-core-go `validate-peer` update in the same cohort cycle, or the cohort cycle does not close. Conversely: an oracle PR that touches `typesystem.go`'s core-floor list, the `CORE-TREE-*` vectors in `tree_operations.go`, or the `expectedHandlers` profile fields (`coreOps`, `coreProfile`) without a corresponding V7 §9.5/§9.5a edit is rejected with reference to this row. Audit mechanism: `git log --oneline` against V7 §9 sections and against the oracle's named files; mismatch = blocker. Surfaced by Go's review of v7.72 (downstream-drift concern); landed via v7.72 Amendment 1. Keystone vendoring (`MANIFEST.md` SHA verification) is the third independent observer. |
| **Core Conformance Profile + Type Floor** (the Keystone-unblock bundle) | **Resolved (V7 v7.72; Amendment 1 same day)** | Proposal at `proposals/implemented/PROPOSAL-V7-V7.72-CORE-CONFORMANCE-PROFILE-AND-TYPE-FLOOR.md`. V7 §9.0 defines `--profile core` / `--profile full`; §9.5 the 53-type Core Type Floor Manifest (cross-verified against Go/Rust/Py — zero re-classifications); §9.5a six normative `CORE-TREE-*` vectors (`PUT-1`, `PUT-CAS-1`, `PUT-CAS-2`, `DELETE-1`, `LISTING-1`, `PATH-FLEX-1`); §6.7 informative resolution-first information-leak paragraph. F13 + F14 closed via no-spec-change ruling memos (already pinned in v7.63 / v7.71). F17 + F18 + F19 closed by §9.0 / §9.5 / §6.7. Cohort handoff to Go via the V7.72 core-profile cohort impl-team-alignment doc; ~2 day target for `--profile core` wire-up in validate-peer. Keystone unblocked to run peer #2 hands-off once Go ships. **Extensibility-spike gate (keystone ARCH-ASK §7) sequences AFTER peer #2** — recommendation: two independent core peers gives the spike a stronger evidence base. |
| **Capability handler amendment** (standalone handler, baseline policy surface) | **Landed (V7 v7.62)** | Proposal at `proposals/implemented/PROPOSAL-V7-CAPABILITY-HANDLER-AMENDMENT.md`. Spec-side ratification complete; conformance update + validate-peer test vector run follows (see new items below). |
| **Capability handler — `validate-peer` test vector update** | **NEW pending (post-v7.62; expanded by the v7.62 cross-impl closeout)** | New `validate-peer` vectors needed: (a) `request` subset-validation (caller asks within bounds → 200; exceeds caller's cap → 403 `scope_exceeds_authority`; exceeds policy entry → 403); (b) `revoke` happy path (marker written at `system/capability/revocations/{hex}`; subsequent `is_revoked` returns true); (c) revoke authz (caller without revoke-cap → 403); (d) `configure` writes a `policy-entry` at `system/capability/policy/{peer_pattern}` (exact + `default` lookups — v7.63 F8 renamed the fallback segment from `*` to `default`); (e) §4.4 handshake union (peer with policy entry for caller gets floor + policy grants; peer without entry gets floor only); (f) `501 unsupported_operation` on registered-but-not-implemented op (distinguishable from 404 / 403). **Added by the v7.62 closeout (see the v7.62 capability-handler cross-impl closeout review):** (g) **501-before-403 dispatcher ordering** — call a registered-but-unimplemented capability op with an authz-deficient cap; expected `501 unsupported_operation`. The dispatcher MUST check manifest-op existence (→ 501) **before** cap-authz (→ 403). V7 §6.2 line 2724 already binds this normatively ("the caller's authority is irrelevant"); this vector pins it as a conformance item. Go discovered the masking trap as a real bug while authoring v7.62 vectors (fixed in Go cycle); other impls plausibly have the same ordering bug. (h) **`delegate-zero-parent` → `400 bad_request`** — zero-hash `parent` field on `delegate-request` is structural input defect (SEC-18 zero-hash + §3.6 M3 normalization precedent), NOT a "not found" lookup miss. Three impls split: Go = 400, Rust + Python = 404. Pin 400. (i) **wire-only cap revocation distinguishing** — request a cap (inline-returned, no tree path) → revoke it → re-present in EXECUTE. Expected: rejection via marker-check in `verify_request`. Surfaces whether each impl actually wires `is_revoked` through the marker path or only catches bound-path revocation. (j) **cross-peer `delegate` rejection** — remote caller A invokes `delegate` on B. Expected: `501 unsupported_operation` (V7 v7.63 F1 narrows `delegate` to same-peer-only for v1). Pair vector: same-peer self-attenuation happy path — local-peer-as-caller invokes `delegate`; child cap chain-verifies under §5.5. **Added by V7 v7.71: authorization-denial status+code matrix** — pins the §3.3-403 / §5.2 verdict-to-status / `result.data.code` contract that the Rust SHA-384 cohort's `verification_failed` residual exposed. The matrix is the enforcement half of the v7.71 §3.3 normative MUST (catch-all defaults outlawed on authorization paths). (k) `AUTHZ-DELEGATE-GRANT-1` — delegate without `system/role:delegate` grant (PR-8.2) → `403 capability_denied`. (l) `AUTHZ-DENY-DEFAULT-1` — generic `verify_request` DENY (e.g. cap doesn't cover op) → `403 capability_denied`. (m) `AUTHZ-SCOPE-EXCEEDS-1` — §6.2-dispatch-level `request` exceeds policy/caller authority → `403 scope_exceeds_authority`. **NOTE**: overlaps with (a); (a) tests the cap-handler `request` op specifically, (m) tests the dispatch-layer general case — cohort SHOULD reuse the fixture, treating (a) as the specialization. (n) `AUTHZ-GRANTEE-1` — unresolvable grantee (§5.2 single 401 carve-out / PR-3) → `401 unresolvable_grantee`. (o) `AUTHZ-REVOKED-1` — revoked cap on use (EXTENSION-ROLE §5.5 in-flight cascade) → `401 capability_revoked`. Complements (i) which verifies the marker mechanism wires through; (o) verifies the surfaced status+code. (p) `AUTHZ-NO-CATCHALL-1` — granter identity unresolvable during chain verify (§5.5 `granter is null → DENY`) → `403 capability_denied`, **NOT** `verification_failed`. The exact regression pin for the Rust residual. (q) `AUTHZ-EXPIRED-1` — capability used after `expires_at` (§5.6 / §5.2 validity check) → `403 capability_denied` (NOT a separate `capability_expired` string — pins that expiry uses the default code). Authoring owner: Go per the cross-impl test-vector convention. |
| **`:delegate` shape** (D-DEL) | **Resolved (V7 v7.62 §3.6, §6.2)** | Self-attenuation only — `delegate-request` carries `parent + grants + ttl_ms` (no `grantee` field); handler enforces direct-hold `parent.grantee == caller's authenticated identity`. Third-party delegation deferred to a future amendment with explicit grantee auth model. |
| **Revocation** (D-VOC: `is_revoked` ↔ `:revoke` unwired) | **Resolved (V7 v7.62 §5.1, §6.2)** | `is_revoked` extended with explicit marker check at `system/capability/revocations/{root_hash_hex}`; `unknown_root_policy()` indirection eliminated. `:revoke` is path-agnostic (tree-unbind + marker for path-bound caps; marker-only for wire-only). Rust + Python land `is_revoked` + `capability_path_for` impls as cross-impl follow-on (~1 week each); Go has existing infrastructure. `validate-peer` negative probe per the row above. |
| **Hello negotiation** (D-NEG: §4.5 intersection a stated MUST, 3/3 non-compliant) | deferred, **document don't pin** | No clear answer yet + no consumer; document each impl's behavior + what §4.5 says; revisit. An unimplemented MUST is worse than an honest OPTIONAL. |
| **Protocol version string** | informational for now | V7 §8.4 = `entity-core/1.0` (Go/Rust correct). Python sends `entity-core/7.0` — a divergence that's latent only because nobody intersects; **left un-pinned normatively** until we have stable versioning. Fix Python when negotiation lands. |
| **Leg-3 reverse-authenticate mechanism** (M-1 hello flag vs M-2 initiator-requested) | deferred | No impl sends leg 3; the reachability-gated clarification landed (V7 §4.1). Mechanism pinned when a serving-initiator use case surfaces. |
| **Storage-contract / transport technical review** | pending | The core is transport-agnostic (NETWORK extension owns transport specifics). Some deferred connect/auth items may migrate to the extension once we verify the flow under different storage contracts and movement. Why these are parked in the guide rather than hard-pinned in core now. |
| **Connect-surface conformance vectors** | recommended, not yet authored | short/empty nonce, wrong-length pubkey, unknown key_type, double-hello, authenticate-before-hello, hello/authenticate peer_id mismatch, cross-connection replay, impersonation-by-different-key, peer_id↔pubkey mismatch (the probe that would have caught the Python gap). Until authored, the PoP MUSTs (v7.61 §4.6) are prose-only — see the meta-rule (§7). The **`unknown key_type → 400 unsupported_key_type`** vector here is the UN-b/F48 gate (`PROPOSAL-KEYSTONE-CROSS-SUBSTRATE-HARDENING` B2). |
| **F38 — `created_at` mint-timestamp precision** (keystone) | NEW pending; **arguably normative, not just a guide note** | `created_at` second-truncation makes same-scope same-second re-mints byte-identical → hash-alias → 403-cascade (isolated category runs stay green; Pure Data A-PD-016). Disposition: **`created_at` MUST be ms-precision from a real-time clock at every mint site** — belongs in normative token-shape text (§3.6/§5.5), carried as a conformance note because the failure is timing-dependent (no deterministic vector). Fixed keystone-side. Route the normative half into a core token-shape amendment. |
| **F39 — bare `["*"]` resource is granter-local** (keystone) | NEW pending (low; mostly resolved keystone-side) | An open/debug seed with `resources:["*"]` can't cover foreign namespaces — §5.5a makes bare `*` granter-local; `universal_address_space` writes silently SKIP; a debug seed needs `["*","/*/*"]`. Disposition: **one sentence on the dual resource form** (SDK-OPERATIONS §2.1 / this guide); optionally name the missing absolute form in the oracle skip message. |
| **F41 — authority-as-derivation appendix** (keystone) | NEW pending; **high-leverage, no wire/verdict change** | Specifying the §5/§6.6 authority verdict **as a monotone derivation** makes fail-closed + the §5.5a within-grant conjunction **structural invariants**, not silently-violable MUST-prose (Datalog A-DL-013). Disposition: **add an authority-as-derivation informative appendix** (this guide or a core informative appendix); §5.10 already formalizes much of the monotone-core/non-monotone-guard split, so this consolidates it. The single most-leveraged guide write of the keystone pull-in. |
| **Keystone corpus gaps F34/F35/F44/scalar-`data`** | NEW pending → routed | Vacuous-green closures (tampered granter-sig; §7a reentry-echo verifies nothing; caveat accept-path untested; scalar-`data` accept). Specified in `entity-core-protocol/docs/proposals/PROPOSAL-KEYSTONE-CORPUS-GAPS.md`; ride the F29/F30 add-vector/cross-bless loop + the `security`/`authz`/`concurrency` categories (F35 is a `conformance_handlers.go` probe fix). |

## §10 Cross-references

- **`ENTITY-CBOR-ENCODING.md` Appendix E** — normative conformance contract (categories, fixture format, harness semantics, growth policy).
- **`ENTITY-CBOR-ENCODING.md` §5.4 (Entity Fidelity)** — the property-vs-mechanism framing the conformance gate makes testable. *(Repointed 2026-08-10 from `ENTITY-CORE-MACHINE-SPEC.md` §1.8, which is now a marked derived copy; §5.4 is the canonical home and sits in the same document as the Appendix E vectors that verify it.)*
- **`proposals/implemented/PROPOSAL-WIRE-ENCODING-CONFORMANCE-VECTORS.md`** — history. The proposal that ratified the appendix framework. Some §5 framing (Go-as-named-reference) is superseded by Appendix E v1.5; see §3.4 above.
- **`entity-core-go/cmd/validate-peer/`** — the harness driver. The conformance category extends this binary.
- **`entity-core-go/cmd/internal/wire-conformance/`** — the build-script home for `.diag` → `.cbor` translation; promoted from the original one-shot fixture writer.
- The keystone F1-ECF-fixture request to arch — the keystone request that prompted v1.5.
- The Rosetta status tracker — F1/F2 disposition tracking.
- The comprehensive validate-peer audit — the full oracle audit (S1–S5; per-category map; the multisig spec-behind-impls finding) the §8 roadmap is drawn from.
- The handshake-PoP-and-auth-status catch-up proposal — the connect/auth catch-up (PoP, status boundary, leg-3); §9 ruling dispositions; partially landed V7 v7.61.
- **`proposals/implemented/PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`** — multisig, landed V7 v7.60; `validate-peer multisig` category is its conformance pin.
- **`proposals/implemented/PROPOSAL-V7-CAPABILITY-HANDLER-AMENDMENT.md`** — the capability-handler amendment, **landed V7 v7.62**. The §9 register's D-DEL + D-VOC items resolve here; the validate-peer test vector update + Rust/Python `is_revoked` + `capability_path_for` impls are the follow-on work.
- The advisory on Python peer-id binding — the independent Python identity-binding security fix (the §4.6 step-3 gap; fixed impl-side).
- **`docs/proposals/PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION.md`** (AE-1) — the admission contract §7c is the conformance artifact for; **`PROPOSAL-COMPUTE-LOWERING-TOOLKIT.md §5`** — the worked-lowering vectors that seed the compute corpus; **`ROUTING-2026-07-21-cross-team-sequencing.md`** — where the corpus sits on the cross-team clock (the critical path). Grounding survey: `entity-workbench-go/entitysdk/axis1_equivalence_test.go` + `entity-workbench-go/docs/architecture/reviews/COMPUTE-AXIS1-ORACLE-GAPS-2026-07-16.md`.
