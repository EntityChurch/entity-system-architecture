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

The fixture is `conformance-vectors.cbor` (normative; loaded by impls) generated from `conformance-vectors.diag` (human-editable; CBOR diagnostic notation per RFC 8949 §8). The `.diag` is the source of truth for authors; the `.cbor` is the build artifact.

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

1. Edit `conformance-vectors.diag` — add the vector under the appropriate category, give it a fresh `<category>.<n>` id, write the description, set `kind`, populate `input` (for `encode_equal`) or `canonical` (for `decode_reject`).
2. Run the build script (see §3.2) to regenerate `.cbor` from `.diag`.
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

**Scope: a *guarded operation* is one a caller can request. An authoring invariant is not one, and has no negative half `[clarification, 2026-09-03]`.** The rule above binds a condition on an operation — something a requester supplies, and which the peer may therefore *refuse*. Where a rule instead fixes how the peer **authors** its own artifact, with no operation through which anyone can ask for the other form, **there is nothing to refuse and the negative half does not exist.** Demanding one produces a check no conformant peer can satisfy, which is §5.2b's *"a surface the suite cannot reach"* arriving from the authoring side.

**The worked example, and it is why this paragraph exists.** `ENTITY-CORE-PROTOCOL` §4.5a item **1a** makes the `system/peer` identity entity *"authored under ECFv1-SHA-256 unconditionally"*. No caller can request another peer's identity entity in a chosen format, so no peer can refuse one. The agility vector nonetheless asked for a refusal assertion, and `entity-core-keystone` reported the honest result: *"no peer refuses it,"* and satisfying it *"is a peer-behaviour change — the constructor must reject a non-floor home format — not a test edit."* **A constructor rejecting its own input is not a guard; it is an assertion about a code path no wire input reaches.** The vector's negative half is therefore **retired**, not owed, and the observable that remains — that the identity hash **is** the floor form — is what the harnesses already assert and is the whole of the property.

**Distinguish it from §4.5a item 2**, the transmission MUST NOT, which *is* a guarded surface: a peer can be sent a non-floor form and must refuse it. That one keeps its negative half. **The test is not whether the rule is normative or important; it is whether any input exists that the peer could be asked to refuse.**

> **They declined to fake it, and that is the only reason this was resolvable.** Four fixed harnesses assert the positive half and state at the site that the negative half is owed, rather than writing an assertion that would have passed. **A deferral stated at the site is the honest form of an unsatisfiable check** — the alternative is a green run that certifies nothing, which is the failure this whole section exists to end.

**This is not a new idea; it is an existing rule that was only ever written one section at a time.** `EXTENSION-REGISTRY` §6a.9.1 already has it — `policy_manual_publishes_nothing`, `policy_allowlist_unlisted_publishes_nothing` — and those are exactly the checks nobody's registry failed. **The sections where nobody wrote it are precisely where the hole appeared:** §6a.9 named `REG-REGISTER-PROOF-1` for register and nothing for `revoke`/`renew`, and **two of the three reference implementations shipped those two ops with no verification at all.** *The vector list, not the prose, is what gets implemented against.*

**The third implementation is the sharper half of the lesson.** It implemented the *prose* — it verified the proof on both ops — and the shared checks therefore **failed it**, because they asserted acceptance with no proof attached. **A correctly-guarded peer scored worse than the two unguarded ones**, and converging to green would have meant writing unauthenticated revocation into a correct implementation. **When positive-only coverage meets a correct peer, the scoreboard names the correct peer as the defect.** That is the failure mode this rule exists to end, and it is why the row below records a *refusal to converge* as the detection mechanism.

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

This is ADR-0012's *"conformance-green ≠ correct if the test asserts the wrong thing"* in its authoring-time form: §5.2a is a rule with no surface, §5.2b is a surface the suite cannot reach, **§2.4a is a surface the suite reaches and scores backwards**, §2.4b is a surface the suite reaches and scores **forwards for the wrong reason**, **§2.4c is a surface the suite reaches and cannot READ** — both answers are the same observable — and §5.2c is a surface the suite reaches only **by accident**.

### §2.4b A deny-only check MUST establish its own antecedent `[MUST]`

**§2.4a is a category that scores a correct peer badly, and a red is investigated. This is its mirror and it fails the quiet way: a check that asserts only a refusal cannot tell the refusal it is testing from a refusal it caused.**

> **The rule.** Where a check asserts that an input was **refused**, and the property under test is *why* it was refused — the declaration reads *"MUST refuse this, **even though** the input is otherwise valid"* — the check MUST establish that antecedent **inside itself**: a positive control on the **same probe**, over the **same operation, target and scope**, built from the **same material**, with the property under test as the **only** variable.

**Why this is not pedantry: a refusal is the cheapest observable a peer produces.** Any construction fault on the driving side — a malformed field, a signature over the wrong bytes, a mismatched grantee, a resource pattern that does not cover — produces **exactly** the observable the check is looking for, at **every** peer at once. The check then reports PASS across the whole cohort having measured nothing about the rule, and the more implementations that agree, the more confident the wrong conclusion becomes. **A fault in the harness arrives wearing a green three-way.**

**The discriminating shape is two arms on one probe, differing in exactly the property under test.** Where the rule says *this form of authority does not authorize what that form does*, the control is the other form: the accepting form MUST succeed — asserted on the **field that proves it landed**, not on a status alone — and the refused form MUST be refused, with the form as the only difference between them.

**Three things that look like the control and are not:**

1. **A mutation witness recorded in a comment, or in the oracle author's own unit test.** It is evidence about one artifact at one commit. It does not run at the other peers — where the same check reports the same green — and it does not fire when the construction it vouches for later drifts. §2.4a's corollary requires that a check be *makeable to fail*; this requires that its **passing be attributable**.
2. **A neighbouring check that drives a different input.** The same surface is not the same probe: a second credential, minted separately, controls for nothing about the first.
3. **An in-process validity assertion run through the same implementation's verifier.** Construction and verification from one source catches a typo and cannot catch a shared misreading — two operands, one judgement. Acceptable as a cheap floor **beneath** the control; never as the control.

**The mechanical tell, and it is worth grepping for.** A deny-only check that needs this rule almost always **says so in its own declaration**: read the declarations for *"even though"*, *"while"*, *"despite"*, *"notwithstanding"* — then ask which arm establishes the clause that follows. If none does, the check measures the clause before it and nothing else.


> **The declaration KEEPS the antecedent, and names the arm that establishes it.** The tell above has an obvious perverse discharge — **delete the *"even though"* clause and the grep goes quiet while the check is unchanged** — so the rule is stated in the direction that closes it. **The antecedent is the reason the property is not obvious**, and a declaration that drops it leaves the next reader unable to tell a paired check from a bare one without reading the body. A conformant declaration says both halves: *what MUST be refused, even though X holds* — **and** *paired with the arm that establishes X.*

### §2.4c When both arms SUCCEED, the status cannot discriminate — assert the field that names which input was acted on `[MUST]`

**§2.4a is a check that scores a correct peer backwards; §2.4b is a check that passes for the wrong reason; this is a check whose two outcomes are the same observable.** Where a probe varies *which* of several inputs a peer should act on, and **both the right answer and the wrong answer return success**, a status assertion measures nothing at all — and unlike §2.4b it does not even need a fault to go wrong. It is wrong by construction on every run.

> **The rule.** Where the property under test is **which** input the peer selected, the check MUST assert a **field of the response that is attributable to exactly one input** — a returned entity's type, an identifier, a content hash — and MUST NOT rest on the status, the count, or the absence of an error.

**The worked shape.** A request carries two candidate targets, one the caller is entitled to and one it is not, and the peer answers `200` either way. The only thing separating a conformant peer from a compromised one is **the type of the entity that came back**: one of them can only have come from the entitled path, the other only from the forbidden one. **A check that asserts `200` reports the same result for both.**

**Why this is a distinct rule and not §2.4b restated.** §2.4b is about a refusal whose *cause* is unestablished; the remedy is a positive control. Here there is no refusal to attribute — **both arms are successes and the peer is behaving; the question is which thing it did.** A positive control does not help, because the control also passes. **Only a witness field does**, and the check has to know in advance which field can only have come from one input.

**Corollary, and it is what makes this cheap to apply: pick the discriminating input for its WITNESS, not only for its authority.** A probe target chosen because it is out of scope but whose response is shaped identically to the in-scope one cannot be read; a target whose response carries a distinct type makes the same probe self-reporting. **That choice is made when the vector is designed and cannot be recovered afterwards.**

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
    --diag  core-protocol-domain/specs/test-vectors/ecf-conformance/conformance-vectors.diag \
    --out   core-protocol-domain/specs/test-vectors/ecf-conformance/conformance-vectors.cbor
```

This is mechanical translation of `.diag` to canonical-ECF-encoded `.cbor`. It is not the cross-impl gate — the gate is the cross-impl `emit-canonical` agreement under §3.1. The build script exists because impls load `.cbor`, not `.diag`.

### §3.3 The validate-peer cross-impl category

A new `validate-peer` category (`-category conformance`) consumes the per-impl emission outputs and diffs them. The category is convergence-shaped (operates over multiple peer outputs at once), not single-peer-shaped:

```
$ validate-peer -peers go:emit-go.cbor,rust:emit-rust.cbor,python:emit-python.cbor \
                -category conformance \
                -corpus conformance-vectors.cbor \
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

### §3.5 Core-peer scoreboard discipline (normative for `--profile core`; V7 v7.72)

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

### §3.6 Run discipline (normative for cohort closeout; V7 v7.70)

A 23-day false-baseline drift (a harness keypair bug whose 20-test cascade was repeatedly labelled "pre-existing infrastructure" and inherited across sessions; documented in the v7.69 same-format-drift postmortem) makes these binding on any cohort closeout that reports test results:

1. **No "pre-existing" disposition without a bisect.** A failure claimed to predate the current work MUST carry the triple: the commit hash where it first appeared, the test name, and a one-line root cause. Without that triple, "pre-existing" is not an allowed disposition.
2. **Skips count; a run is `(PASS, FAIL, SKIP)`.** `N FAIL + M SKIP` is **INCOMPLETE**, not PASS. A skip means the behavior is *unknown*, never *correct*. Each skip carries an explicit, time-bounded reason (e.g. "needs 3 peers, this run had 2 — re-run with 3") and is run on the next cycle that satisfies its precondition.
3. **Same-format baseline is a hard gate before any cross-format claim.** A document making a cross-format claim MUST first show the same-format baseline is clean for the *same* setup. Cross-format numbers measured against an unclean baseline are unanalyzable (they mix the cross-format effect with the baseline bug).
4. **Closeout and memory entries carry the numbers.** Any artifact labelling work "clean/locked" carries the actual `(PASS, FAIL, SKIP)` triple, not just verbal characterizations — so a future session inherits "185/185 same-format, 13 cross-format root failures by class," not "cohort-locked."
5. **Cross-format hash-equality tests are explicit SKIPs.** Tests that compare two peers' content hashes directly (trie roots, version-determinism, xpeer-determinism) are single-address-space by design (V7 §1.2 / §1.2a). Under a **cross-home-format** pairing they MUST be explicitly SKIPped with the tracked reason "single-address-space test; cross-format pairing is experimental per V7 §1.5" — never silently failed and never silently passed. They run normally under same-format pairings.
6. **Decode the `.cbor` artifact, not its sha256.** A cohort closeout that asserts "byte-equal" by comparing the **sha256 of an emitted `.cbor`** across impls only proves the impls produced identical bytes; it does not prove the bytes are correct. The byte gate MUST also decode the artifact and assert structural invariants against the `.diag` (the source of truth): every typed-bytes field has the spec-mandated length (e.g. Ed448 `secret_seed` = 57 B per RFC 8032; the corpus's `0xAA×64` fixture pubkey = 64 B), and no normative-output field carries a placeholder string (`"TBD-…"`, `"PENDING"`, etc.). `grep TBD` on the `.diag` is not a substitute — the producer may have failed to substitute placeholders back into the `.cbor` even when the `.diag` is clean. This rule is the structural sibling of (4) above: where (4) requires the closeout *narrative* to carry the numbers, (6) requires the closeout *byte gate* to carry the decode. Reason: F16 — the crypto-agility corpus's `.cbor` sha-locked on a file whose Ed448 seeds were 58 B (off-by-one), whose experimental pubkey was 63 B (off-by-one), and whose 12 Phase-2 `expected_*` fields were still literal `"TBD-COHORT-ROUND-TRIP"` text. Documented in the F16 agility-corpus `.cbor`-regen closeout.
7. **A verdict is the check-set actually asserted — never a proxy for it.** No run may carry a prior verdict forward, or declare an oracle "current," on the strength of a value that tracks *which checks ran* rather than *what they assert*. A published number MUST be **dual-anchored**: the oracle commit **and** a `check_set_digest` over the exact assertions in the run (count + content), both required; a match on only one is not a match. This bans the whole family in one rule — a `core_gate_fingerprint` that tracks categories not assertions, a `grep` over source, a harness's default-category `Result:` line, a green Part-A probe standing in for a Part-B certification: each is a proxy, and a proxy is never a verdict. Corollary (the census rule): a peer's status is `UNMEASURED` until the *core gate itself* ran against it — a harness that silently defaulted to another category did **not** measure the gate, and "no result" is `UNMEASURED`/`FAIL`, never inherited PASS. Reason: the 0.8.1 bucket-B census — `cc1970f` and `af8a582` shared a byte-identical `core_gate_fingerprint` while one build contained none of the four bucket-B vectors and the other contained all four; `oracle-bootstrap.sh`'s fingerprint-only short-circuit would have run the stale check-set over all 43 peers and published it as current. The same census found `sql`/`ruby`/`dart`/`pd` mis-classified from source greps and harness-default lines. Fixed at `c04d04c` (dual anchor required; carry-forward-at-same-fingerprint withdrawn). This is the run-discipline sibling of the CDN-corridor meta-rule: not validated until the *asserting* check exercises it.

---

### §3.6a The outcome vocabulary — five results, and three of them are not PASS and not SKIP `[MUST]` `[added 2026-09-16]`

§3.6 rule 2 fixes the run triple `(PASS, FAIL, SKIP)` and the rule that a skip is *unknown, never
correct*. That triple is about **a check that ran or did not run**. Three situations are neither, and
a suite with only three words picks one of them **silently** — which is how the same observation
becomes PASS in one suite and FAIL in another with neither author making a mistake.

**1. An observed outcome the requirement does not enumerate is `inconclusive`, never PASS `[MUST]`.**
Where a requirement declares a closed set of conformant answers (an `accept` arm and a `refuse` arm,
say) and the peer produces something in neither, the probe **ran** and produced an observable, so it
is not a SKIP; and the requirement made no claim about that answer, so it is not a PASS and cannot
honestly be a FAIL either. It is `inconclusive`, and the suite MUST record the observed value
verbatim. **`inconclusive` counts as NOT-PASS in the run arithmetic**, exactly as a skip does.

⚠ **The distinction it preserves is an authoring signal, which is the whole reason not to collapse
it into SKIP:** a SKIP says *we could not run this*, and an `inconclusive` says *we ran it and our
requirement does not range over what came back.* The first is scheduled for a later cycle; the
second is a defect **in the requirement** and is fixed by re-authoring it. Collapsing them files a
requirement bug in the queue for environment problems, where nobody will look for it.

**2. Behaviour the specification PERMITS to be absent is `not-applicable`, and it is NOT a skip
`[MUST]`.** A peer that ships no WebSocket listener, or declines a `MAY`, has been **measured**: the
suite established that the optional surface is absent, which is a conformant state. Treating that as
a SKIP — and therefore, by the ecosystem rule that a skip counts as a failure, as a failure —
penalises a peer for exercising a permission the specification granted, and makes every optional
surface un-passable by construction. **`not-applicable` does not count against a run.**

⛔ **It MUST be declared, never inferred.** The requirement names the precondition whose absence
makes it inapplicable and cites the clause that permits the absence; the suite records **which**
precondition fired. An undeclared `not-applicable` is indistinguishable from a suite excusing a
failure it did not understand, and it would be the most attractive escape hatch in the vocabulary.

⭐ **This does not carve an exception into "a skip counts as a failure" — it removes cases that were
never skips.** The ecosystem rule is about *unmeasured* behaviour. A permitted absence is measured.

**3. A declared precondition that did not hold makes the run a SKIP, whatever the assertion did
`[MUST]`.** Where a requirement declares a setup precondition and the precondition was not
established, the observation is not evidence about the requirement, and **an assertion that happens
to hold under the wrong setup is a coincidence and MUST NOT be reported as PASS.** The suite SKIPs
with the unmet precondition named.

⚠ **This one has teeth and the worked case is why it is a MUST.** A requirement declaring a covering
grant, run against a peer where the grant did **not** cover the probe, observed the expected status
and would have been reported PASS — and the status it observed is the one `ENTITY-CORE-PROTOCOL`
`0.8.2.30` subsequently ruled **non-conformant**. Reporting PASS there records a conformant-looking
result for precisely the defect the corpus exists to catch. **The assertion holding is not evidence
that it held for the declared reason.**

**4. A requirement whose obligation quantifies over a DOMAIN declares that domain, and scores each
member as its own arm `[MUST]`.** Where the normative rule says *every request of this kind*, a probe
of one member measures one member. A requirement that probes one and reports on the rule is
over-claiming, and two suites that pick different members disagree while both are right — which is
not a divergence about the peer at all.

The requirement declares the domain it ranges over (resource present/absent, URI peer-relative or
fully qualified, and so on); each member is a scored arm; a run reports per member. **A requirement
that deliberately probes a subset says so and scopes its claim to the subset** — which is a legitimate
and often correct choice, and is only wrong when it is silent.

**5. The step and assertion vocabularies stay OPEN until two independent suites have exchanged the
format, and each suite declares its verb set with its results `[MUST]`.** Closing a vocabulary is
cheap to do and expensive to do wrongly: a verb set pinned from one suite's needs pins that suite's
internals as the contract. **This is `§1`'s Stage-4 rule one tier up** — the vectors are byproducts of
implementations running against each other and are canonicalized afterwards, and a check-set format is
the same kind of artifact. Until a second independent suite exists, an open vocabulary is an honest
record of an unsettled surface; **what makes it safe is the declaration**, which turns a silent
divergence between two readers into a visible one. **Where a step verb or SKIP condition is carried in
prose, a suite MUST reproduce it verbatim in its result** rather than paraphrasing it into its own
vocabulary.

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

### §5.1 Corpus identity `[MUST]` `[revised 2026-08-22]`

**A corpus is identified by its name, never by a version stamp.** The directory is named for what the corpus tests (`crypto-agility`, `ecf-conformance`); the artifacts are `<subject>-vectors.{diag,cbor}`. A corpus directory or artifact name **MUST NOT** carry a spec-revision stamp (`v767`, `v7.67`) or an artifact version (`-v1`, `_v2`). A corpus artifact's stem MUST name the same subject as its directory.

*Rationale, and why the previous rule is retired rather than enforced: the prior §5.1 made an integer corpus version part of the conformance citation and required a bump on every vector addition. Between the first public release and this revision the ECF corpus went 69 → 71 vectors and the crypto-agility corpus gained three matrix vectors; neither filename moved, in any of the repos carrying them, and no `-v2` was ever created. **A rule broken by every change it governs is not a weak rule, it is the wrong rule** — a corpus is a growing set of canonical vectors, vectors are never removed, so the "version" was only ever an opaque restatement of "something changed." The stamp was also the sole reason a corpus needed two committed copies, and that structure produced four distinct drift defects in one week.*

**The conformance citation is `(spec-version, corpus-name, artifact sha256)`.** The sha256 of the `.cbor` is the exact identifier: it cannot be forgotten, it is already what the corpus gates check, and it is already how vendors pin. `ADR-0012` requires oracle-pinned numbers; this makes the corpus pin the same shape.

**Change history lives in `CHANGELOG.md` beside the artifacts** — dated entries naming vectors added, any vector whose `input` or `canonical` changed together with the spec revision that changed it, and the cross-impl round that re-blessed the result. This holds strictly more information than an integer: `-v1 → -v2` says something changed; a changelog entry says what, when, why, and who confirmed it.

**Vectors are never removed.** A landed vector stays a conformance criterion. A vector that is wrong is corrected in place through §5.1d and recorded in the changelog, never deleted.

**Changing a vector's `input` or `canonical` remains a spec-changing event** and goes through the normal proposal cycle — the previous canonical bytes were *the* correct bytes for that input under the prior spec.

**One corpus, one copy.** A corpus has exactly one authoring location. A vendored copy is **byte-identical in every member**; no de-versioning, date-stripping, or citation-shortening transform is applied on the way in. Vendor-local framing belongs in the vendor's own manifest, never in the artifacts.

*Gated by `spec corpus` (`entity-system-arch-tools`): `corpus-version-stamp`, `corpus-name-mismatch`, `corpus-pair-incomplete`, `corpus-placeholder`, `corpus-fixture-width`, `corpus-pair-disagree`, and `vendor-drift` under `--vendor`.*

### §5.1a Division of labour — architecture sets fields, the encoder settles bytes `[MUST]`

**Architecture edits the `.diag` source: vector `id`s, `description`s, `kind`s, inputs, and which assertion a vector makes. Architecture does NOT hand-derive, hand-edit, or hand-inspect encoded bytes.** No byte-run measuring, no hex-literal eyeballing, no `.cbor` edits, and no byte pin transcribed from a report into a corpus by hand. **A claim about what bytes an artifact contains is made by running a tool and quoting its output, never by reading the file.** The `.cbor` is a build artifact and architecture is not its build owner.

*Rationale: hand-carrying values into a corpus is how matrix rows go stale and how a width correction reaches the artifact but never the source. Both classes were eventually found by hand-inspection, which feels like diligence and is not a process — it does not run, it does not gate, and it does not survive the reviewer's attention span.*

### §5.1b Two gates, and they assert different things `[MUST]`

Both are required. Either alone leaves the class the other catches invisible.

| Gate | Asserts | Catches |
|---|---|---|
| artifact-is-expected | decode the `.cbor`, re-derive the crypto, check structural invariants against the source | a corrupted or wrongly-valued artifact |
| source-produces-artifact | re-encode the `.diag`, compare to the committed `.cbor`, write nothing | **source and artifact having drifted apart** |

**The second gate exists because its absence cost two months.** A width correction was once applied to a `.cbor` and never swept back to its `.diag`. The first gate reported clean throughout — *correctly*, because the artifact **was** what it was expected to be. Nothing asserted that the source still produced it, so a corpus verifying green against itself carried a source that could no longer rebuild it.

**A source-produces-artifact check MUST NOT recommend rebuilding on failure.** When source and artifact disagree, **which side is right is a judgement, not a default.** In the incident above the `.cbor` was the correct side, so a blind rebuild would have destroyed the good copy and silently re-introduced the bad widths. Deciding which side is right is exactly the step a mechanical regen skips. The check writes nothing and is safe to run against a tree its runner does not own.

### §5.1c The legacy encoder proof MUST be pinned to a frozen source `[MUST]`

An encoder proof demonstrates that the encoder still reproduces a known-good historical artifact. Its input pair — a frozen `.diag` and the artifact sha it produces — **MUST NOT be re-pinned to a moving source.** Once the live `.diag` legitimately moves, re-encoding it cannot reproduce the historical artifact, and re-pointing the proof at HEAD would make it assert only that today's source produces today's output — **a tautology wearing the costume of an encoder proof.**

### §5.1d Sequence for any corpus change `[MUST]`

1. **Architecture** edits the `.diag` **fields**.
2. **The build owner** rebuilds the `.cbor`.
3. **source-produces-artifact green** (§5.1b).
4. **artifact-is-expected green** (§5.1b).
5. **Architecture commits the artifacts.** The build owner writes the files; it does not run git in a tree it does not own.
6. **The change is recorded in the corpus `CHANGELOG.md`** (§5.1). A corpus change that moves the artifact and leaves no changelog entry has deleted its own history — that is what the version stamp used to stand in for.

**Never hand-edit a `.cbor`. Never transcribe a byte pin by hand. Never re-pin the encoder proof.**

### §5.2 Impl reporting

Impls cite the artifact they ran against, per §5.1's `(spec-version, corpus-name, artifact sha256)` form:

> *passes `ecf-conformance` @ `9695b1f1d939cfdf…` under spec v1.5; full report at `<path>`.*

That means every vector in the `.cbor` at **that sha** returns pass under Appendix E §E.3 semantics, run through the impl's encoder/decoder. **Reports MUST cite the artifact sha; a report without one is not a conformance report** — the corpus grows, so "passes the corpus" is not a claim until it says *which* corpus bytes.

Passing at one sha does not carry forward to the next — an artifact moves because vectors were added or corrected, and either may have surfaced a new corner. The impl team runs the new artifact, fixes anything that surfaces, updates the report. **The corpus `CHANGELOG.md` (§5.1) is what tells them what moved and why**, which an incrementing integer never did.

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

**An assertion about a field reads the DECODED FIELD, never a substring of the response `[MUST]` `[added 2026-09-03]`.** A check that scans the response bytes — or a stringified body — for the value it expects is a **proxy**, and it passes under every shape that happens to contain that value anywhere. **Assert on the decoded key.**

*Canonical instance, and it is the cheapest one in this guide.* `entity-core-rust`'s `extensions/type-system/{validate,constraint}.rs` each declared a local `error_entity` shadowing the canonical one and wrote the code under key **`type`**, not `code` — so all 14 emit sites decoded to `code = absent` at a conformant reader, in violation of `system/protocol/error`'s own descriptor, which declares `code` non-optional. Their test for that body scanned the raw data for the **substring** `invalid_request`, which is present under `type` and under `code` alike. **It passed under both shapes**, through three revisions, and a same-week fix that landed the correct *spelling* at two of those sites was invisible on the wire because the *field* was still wrong. Found only when `entity-core-go`'s harness read the decoded key.

**The transferable form: a code is `(status, FIELD, spelling)`, and an assertion — or a census — keyed on any two of the three is blind to the third.** A substring test measures the spelling and nothing else, which is a real if coarser property; it may stay, but it does not discharge the row. **The check that discharges the row decodes `result.data.code` and compares it.**

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

### §5.3a A fold discloses the cells it touches `[MUST]`

**A proposal folding a normative change into the core protocol MUST carry a CELL DISCLOSURE: the cells of the conformance scope table its deltas touch, and, for each one, whether a check has been DRIVEN against it.** The state vocabulary is closed — `driven` · `named-vector-not-driven` · `no vector`.

**An undriven cell does NOT block the fold.** A fold may land over any number of them, and most will. **An undisclosed cell DOES block it:** a fold whose disclosure is absent, or which names fewer cells than its deltas touch, is not ready to land.

**Why disclosure rather than permission, stated as the property it protects.** An executed check set can be insensitive to a defect that every peer in a cohort shares: the requirement is uncovered before the fold and uncovered after it, the suite is green on both sides, and **nothing in any artifact records that a normative change was made in a region no check can see.** That is §5.2b's unreachable surface arriving at the fold boundary instead of at a check. Requiring the cells to be *driven green* before a fold may land is the stronger rule and is not available: while most cells carry no vector, that rule forbids every fold, and a condition that cannot be met is not a strict gate — it is an unenforced one, and the two are indistinguishable from outside. **Disclosure converts an invisible omission into a countable one, which is the move available to a party that does not run the checks.**

#### §5.3a.1 The required shape

The disclosure is a section of the proposal, titled **`Cell disclosure`**, carrying:

1. **The run it was read from** — a `Read from:` line naming the repository, the commit and the date of the scope-table summary the states were taken from. **A disclosure that cites no run is a guess with a table's formatting.**
2. **One row per cell the fold touches**, each with a state from the closed vocabulary above.

```
## Cell disclosure

**Read from:** `<repo>` `<commit>` `<date>` — scope-cell table summary

| cell | state | note |
|---|---|---|
| `<cell id>` | `driven` | |
| `<cell id>` | `no vector` | requirement on the check set |
```

#### §5.3a.2 Four rules that decide what the disclosure means

- **The drive state is read from the anchor's table, never asserted from the spec side.** The party that authors the specification does not run the checks and cannot observe coverage; the scope table and its summary are the conformance anchor's artifact, and they are the source. Verification on the specification side stops at the shape — **that a disclosed state is TRUE is verifiable only by the party that owns the table**, and this rule says so rather than implying otherwise.
- **The disclosure is taken over the FOLD, not accumulated over its deltas.** A per-delta disclosure measures each delta's own cells and asks nothing about whether the deltas covered the fold; the omission it misses is a cell no delta touched and the fold did. The control is a set difference against the fold's own normative surface.
- **A cell disclosed `no vector` is a REQUIREMENT ON THE CHECK SET, created by the fold**, recorded against the check-set author at the moment of the fold. **It is not discharged by the fold landing.**
- **A change to the scope table's summary output is a change to this rule's input.** The summary is a shared surface between implementations from the moment this rule binds, not an instrument internal to the project that maintains it.

#### §5.3a.3 It binds forward

`ENTITY-CORE-PROTOCOL` **0.8.2.32** is the last revision outside this rule; every revision folded after it carries a disclosure. **Earlier revisions are not reconstructed** — a reconstructed disclosure is a table nobody measured, which is the defect this rule exists to prevent, committed deliberately and with the authority of a landed record. Whether the earlier revisions are ever covered is a question for whoever next runs the scope table over the whole corpus, and it is answered by a census, not by back-filling proposals.

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

### §5.2d A suite declares what it BORROWED — the self-check rule, pointed at the instrument `[added 2026-09-11]`

**§5.2's self-check rule is about a check that never contacts the peer. This is the same rule one level up: a check that contacts the peer through machinery it took *from* that peer.**

A conformance suite needs a wire client — framing, a canonical-encoding codec, a signature scheme. It can write one, or it can borrow one from an implementation. **Borrowing is legitimate and is often the only affordable option. What is not legitimate is borrowing silently**, because the surfaces the suite borrowed stop being independently measured, and nothing in the report says so.

**The rule `[MUST]`.** A suite **MUST** declare the implementations it borrows from and the surfaces it borrows. **A requirement whose subject is a surface the suite borrowed is reported against the lending peer as a `[self]` check, never as a peer result.**

**The test is §5.2's, unchanged:** *could a sibling implementation ship none of this rule and the row not move?* A suite using peer P's ECF codec to score peer P's canonical encoding will score `PASS` whatever P does, because both sides of the comparison are P. The row is about the suite, not the peer.

| Suite consumes | What stops being independently measured |
|---|---|
| a peer's client library wholesale | framing, canonical form, handshake — measured **through one implementation's reading of the wire**, so a shared bug is invisible |
| a peer's codec only | the encoding requirements become partial self-checks against that peer |
| nothing — written from scratch | nothing, and it costs the most |

**None of the three is wrong. The undeclared one is.** A from-scratch suite is the strongest and the most expensive; a borrowing suite is cheaper and narrower, and *a reader can only tell which they are holding if it is written down.*

**Where this becomes load-bearing rather than pedantic:** once more than one suite exists, **the difference between two suites is itself the measurement.** Two suites that borrow from the same peer agree for reasons that have nothing to do with the specification, and their agreement will read as convergence. Declaring the borrow is what keeps that distinguishable from the real thing.

*Raised by `entity-system-conformance` while deciding how to build its first suite, and generalised here because it binds any suite, including the reference oracle. Its authority is §5.2's `[MUST; ruled 2026-08-09]` — this states the borrowed-substrate case that ruling's wording did not enumerate. **If a reader takes it as a new obligation rather than a restatement, it wants a proposal and this clause should be cut back to a pointer.***

---

## §6 Vendoring into keystone

The keystone repo (`entity-core-keystone/`) vendors a byte-identical copy of the canonical fixture for its language-binding generator. The discipline:

1. Arch commits `conformance-vectors.cbor` (+ `.diag`) at the canonical path: `core-protocol-domain/specs/test-vectors/ecf-conformance/`.
2. Keystone copies the `.cbor` to `protocol-generator/shared/test-vectors/{spec-version}/conformance-vectors.cbor`, and verifies it by **sha256**, never by filename.
3. Keystone records the SHA-256 of the vendored fixture in its `MANIFEST.md` alongside the spec-data snapshot manifest.
4. Keystone's CI verifies on every run that the vendored `.cbor` SHA-256 matches the canonical-path file's SHA-256. Drift fails the build.
5. When arch bumps the corpus, keystone re-vendors in a single commit; the manifest update is the audit trail.

This is the only cross-repo flow. Arch writes to its own repo; keystone reads from arch's repo and writes to its own. No cross-repo write authority either direction.

---

## §7 The conformance-surface map (the full picture)

Wire conformance (§§1–6) is one of several surfaces. A peer's conformance claim is `(spec-version, V7 §2.11 level, extension set)` — what it must clear scopes to that.

### §7.0 Four different things are called a "vector" — say which one

**The word is overloaded, including inside the oracle's own source comments, and asking for the wrong
one routes work to a seat that does not author it.** Before writing *"needs a vector,"* pick a row:

| Say | What it is | Lives in | Authored by |
|---|---|---|---|
| **`validate-peer` check** (or *behavioral vector*) | an assertion driven **over the wire against a running peer**, grouped into a **category** (`capability`, `entity_native`, `connectivity`, …) | `entity-core-go/cmd/internal/validate/*.go` | **`entity-core-go`**, per the §9 register's cross-impl authoring convention — *not* whichever seat found the defect |
| **fixture corpus** (or *test-vector corpus*) | static byte-level data — `.diag` source + canonical `.cbor` — for ECF / crypto-agility | `entity-core-protocol/specs/test-vectors/` | **architecture** (this team); vendored downstream per §6 |
| **host-seam check** | an assertion about a peer's **in-process API** — `register_handler` and what it binds — **driven** by the peer's own harness and **asserted** over the wire. The one class `validate-peer` structurally cannot drive alone, because it speaks TCP and the surface under test is a function call | harness: generated per peer · assertion: the `validate-peer` transport | **architecture** authors the reference handler and its expected observable; the generator builds the harness; `entity-core-go` carries the transport. **See §7d** |
| **unit / property / fuzz test** | impl-internal correctness, not cross-impl and not conformance | each repo's own test files | that repo, its own concern |

**`entity-core-keystone` authors none of these.** It **consumes** the oracle: it pins a version
(`tools/oracle-pin.env`), runs `--profile core` across the peer matrix, and publishes
`CONFORMANCE-MATRIX.md`. A new behavioral check therefore reaches keystone **only** as an oracle re-pin —
`check_set_digest` moves, and the cohort census re-runs. *Asking keystone for a vector asks the scorer to
write the exam.*

**So the routing rule:** a spec revision that needs behavioral proof files the check against
**`entity-core-go` as oracle author**, appends it to the §9 register, and lists keystone's re-census as a
**consequence**, not an ask. (Recorded 2026-08-17 after three arch documents asked keystone for four
checks that belong in `capability.go`.)

The surfaces:

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
- **params**: `{ target: text` (pattern to invoke **at the caller**, e.g. `system/validate/echo`)`, operation: text, value: <any>, reentry_capability, reentry_granters, reentry_cap_signatures, deadline_ms?: uint }` — **`deadline_ms` is OPTIONAL and independent of the `reentry_*` triple's all-or-none rule below**; it composes with the presented arm and the ambient arm alike. See §7a.1b for what it obliges. — the last three are the caller-minted authority for the reentry direction (this peer → caller). **`reentry_granters` and `reentry_cap_signatures` are PLURAL carriers `[0.8.2.19]`** — arrays, and the single-granter case is an array of one. **They were singular, and that made one normative rule ungateable:** `ENTITY-CORE-PROTOCOL` §1.4's multi-signature root rule needs a K-of-2 root to drive it, which requires **two** granter identities and **two** signatures; a single-credential carrier cannot express the input, so every seat drove it in-process only. **The set is all-or-none:** supplying the three selects the presented arm, omitting all three selects the ambient arm, and a partial set is `400 invalid_params` — a partial credential is malformed, not ambient.
- **behavior**: originate **exactly one** outbound EXECUTE via the §6.11 reentry sender → `operation` on `target`, **back to the caller over the same inbound connection** (see §7a.2a) → await the response.
- **result**: `{ status: uint, result: <downstream result entity> }`

These are **behavioral contracts**, satisfied by a **native handler each impl supplies** (validator drives them black-box over the wire). They are *not* a declarative body vocabulary the peer interprets — that would re-open the body-evaluator question and re-couple to compute. The guide specifies *what EXECUTE must do*; the impl provides *how*.

> ### ⛔ `dispatch-outbound`'s handler grant MUST be NARROW, and this is a scaffold-contract requirement
>
> **The `system/validate/dispatch-outbound` handler's own grant MUST be scoped to a fixed, declared
> operation set — not the peer's bootstrap grant.** A conforming scaffold declares that set; the
> obvious minimum is the `echo` operation on `system/validate/echo`, which is the only operation the
> reentry contract needs.
>
> **Why this is a requirement and not an implementation detail.** `ENTITY-CORE-PROTOCOL` §1.4's
> outbound gate admits **two** authority contributions — the executing handler's grant (Dimensions
> 1–3, always) and a target-minted credential (Dimension 4 only). A check set MUST discriminate a
> **compose** from a **bypass**, and the discriminating vector is *a valid credential presented to a
> handler whose own grant does not cover the request → MUST refuse*. **That vector is unconstructible
> against a wide grant:** if the handler's grant already covers every operation, the composed reading
> and the bypassed reading return the same answer for every input the probe can send. **The probe
> would sit exactly at the point where the two readings agree** — which is how a cohort-wide
> confused-deputy bypass passed two green vectors across three reference trees and 46 generated peers.
>
> **The discriminating probe, therefore:** drive `dispatch-outbound` to sub-dispatch an operation
> **outside** the handler's declared set, **while presenting a credential that does cover it**. A
> conformant peer refuses; a peer whose credential path bypasses the handler grant succeeds. Pair it
> with the existing reentry probe, which must still succeed — **one arm alone does not discriminate**,
> and a refusal raised *upstream* of the gate is indistinguishable from the refusal under test unless
> the check is validated against a peer that restores the bypass.
>
> **This generalizes and is the reason it is stated here rather than in one vector's prose: an
> authorization gate admitting N authority sources needs N compose-vs-bypass discriminators, and the
> two obvious vectors — all sources agree → allow, no source at all → refuse — are precisely the two
> that are blind to a bypass.**

> ### ⛔ The multi-signature-root vector the plural carrier unblocks is DENY-ONLY, and §2.4b binds it
>
> **The rule it drives is `ENTITY-CORE-PROTOCOL` §1.4's: a multi-signature root MUST NOT relax
> Dimension 4 *even though chain verification accepts the root*** — an antecedent in the property
> itself. A check that mints a K-of-N-rooted credential, presents it, and asserts only that the
> sub-dispatch was refused **does not measure that rule.** A credential invalid for any unrelated
> reason — a malformed multi-granter encoding, a signature over the wrong bytes, a grantee mismatch,
> a resource that does not cover — is refused by every conformant peer for that reason, and the check
> goes green cohort-wide having measured nothing.
>
> **The required control is the single-granter arm, on the same probe:** same operation, same target,
> same scope, same grantee, **the granter form the only variable.** The single-signature credential
> MUST succeed and reach the sub-dispatched handler exactly once; the K-of-N credential over the
> identical request MUST be refused and MUST NOT reach it. That is the compose-vs-bypass shape above,
> rotated onto the authority-form axis, and it is the reason the plural carrier was worth a breaking
> params change: **the carrier makes the input expressible, and only the paired arms make it
> measurable.**
>
> **The existing reentry probe does not substitute** — it is a different check driving a separately
> minted credential, which controls for nothing about this one (§2.4b, item 2).
>
> **A control that establishes the credential is valid does NOT establish that the peer can VERIFY
> that credential form.** A peer whose chain walk rejects every multi-granter root passes this vector
> for a reason that has nothing to do with §1.4 — *fail-closed by absence*, §2.4b's own concern
> arriving at the peer rather than at the check, and the single-signature control cannot see it
> because the control is single-signature. **The discriminating input is a multi-signature root in a
> context where the rule does NOT forbid acceptance**, which does not exist inside this vector —
> §1.4's prohibition is unconditional — so it lives in the capability-chain category as an ordinary
> **accept** of a valid K-of-N root. **Read that row before reading this one at a new seat:** a seat
> that cannot accept a valid multi-signature root anywhere has not been measured here.

### §7a.1a How a REFUSED reentry sub-dispatch surfaces `[MUST]`

**The status SHAPE is not pinned; the CODE is, and it always was.** Two shapes are conformant and both
are already in the corpus — **relayed** (the handler propagates the refusal as its own outer `403`) and
**wrapped** (outer `200`, the refusal carried as the inner status). §7a.2's ambient arm accepts both,
deliberately, and this section does not narrow that.

> **What a refusal MUST NOT do is change the CODE.** When `system/validate/dispatch-outbound`'s own
> outbound sub-dispatch is refused by the §1.4 gate — whether on the handler's grant (Dimensions 1–3)
> or on Dimension 4 — the surfaced code is the authorization domain's code, **`capability_denied`** by
> default or a more-specific defined authorization code. **A transport- or gateway-class code is
> non-conformant**, and this is not a new rule: `ENTITY-CORE-PROTOCOL` §3.3's authorization-path code
> discipline already says *implementations MUST NOT surface a generic catch-all default on an
> authorization path; the catch-all is a sign that an authorization failure escaped its defined code.*
> The same document normalizes the code **regardless of which pipeline layer detects the rejection**,
> for exactly this reason: a caller must not be able to read a peer's internal layering off its error
> codes.

**Why a scaffold section restates a core rule instead of pointing at it.** The scaffold is where the
refusal is *caught and re-emitted* — the handler learns its own sub-dispatch was refused and chooses
what to return — and that re-emission is a code path the core rule's authors were not describing. **A
handler that wraps every unsuccessful sub-dispatch in one generic failure code launders an
authorization verdict into a transport fault**, and the resulting observable is indistinguishable from
*"the route was broken"*. That is the same unattributability §2.4b refuses in a check, one layer over:
the property may hold perfectly and the wire cannot say so.

**The tell, and it is worth checking against your own handler before a run:** if the refusal branch for
an **ambient** sub-dispatch surfaces `capability_denied` and the branch for a **credential-presented**
sub-dispatch surfaces a generic failure, **the two branches disagree about what the same gate decided.**
Both are the §1.4 outbound check returning DENY.

**Shape clarification (post §7b matrix — per the concurrency-gate §7b matrix rulings):** `echo` returns the params **entity** (`{value: X}`), not a bare scalar — `result.value == params.value`. `dispatch-outbound` is a **generic relay**: its `result` field carries the downstream handler's **result entity verbatim**, with **no unwrapping** (it relays arbitrary `target`/`operation` and cannot assume the downstream's shape). So for the echo round-trip the value rides as `result.value` (an entity), and a probe MUST assert `result.value == sent`, **not** `result == sent`. A relay that unwraps, or a probe that expects a bare scalar, is the non-conformant party — not a peer that returns the entity.

**Value-passthrough (pin, post 6-peer matrix):** the relayed `value` IS the params data, passed through as `primitive/any` — **`dispatch-outbound` MUST NOT re-wrap it** (e.g. `{value: value}` so `result.value` comes back a *map*). All six generated peers independently re-wrapped, latent under the value-blind §7a single-call probe and surfaced only by §7b's stricter assertion — *cohort agreement ≠ conformance*. The relay passes `value` through unchanged; the downstream handler's result entity is returned as `result` verbatim. (Note: arch's ruling-#2 prediction "cohort already conformant, no ×6 change" was **wrong** — there *was* a cohort-wide ×6 re-wrap bug; the §7a *principle* held, the cohort-conformance *prediction* did not. Don't predict cohort conformance from the reference impls.)

### §7a.1b The per-call deadline `[MUST]` — `ENTITY-CORE-PROTOCOL` §6.11(c)'s only expressible input

**§6.11(c) is a MUST with no way to drive it.** It requires per-request deadlines to be enforced at
the request layer rather than with connection-wide primitives *"that would race across concurrent
in-flight requests on the same connection"* — a behaviour whose two failure directions are
**shortening** (one request's expiry fails another that had longer) and **extending** (a later
deadline overwrites an earlier one, so the first does not expire when it should). **One probe exposes
both: stagger two deadlines and answer the second between them.** Staggering requires setting them,
and until this section nothing in any handler contract let a prober set one.

> **When `deadline_ms` is present, the peer MUST apply it as the per-request deadline on this
> invocation's outbound reentry EXECUTE, enforced at the request layer per §6.11(c) `[MUST]`.**
> Absent, the peer applies whatever it applies today — so a peer shipping the handler before this
> section landed remains conformant until a probe sends the parameter. A value of `0`, or a
> non-integer, is `400 invalid_params`: a zero deadline is degenerate, not a request to disable one.

#### ⛔ No silent substitution — this is the half the negative control depends on

> **A peer that cannot honour the requested `deadline_ms` MUST refuse the invocation with
> `400 invalid_params`. It MUST NOT apply a different value `[MUST]`.**

A check for this rule needs a **mandatory anti-vacuity control** — one call at deadline `D`, never
answered, which MUST time out near `D` — because without it a peer that ignores the supplied deadline
passes one half of the real arm and fails the other for a reason that has nothing to do with §6.11(c).
**That control's verdict is SKIP (*deadline input not honoured*) and not FAIL**, and the two are only
distinguishable if a peer that will not honour the value **says so**. A refusal is a fact the prober
can read; a silently clamped value is indistinguishable from a broken deadline, and it turns a SKIP
into a false FAIL against a peer whose §6.11(c) behaviour may be perfect.

**A ceiling is still allowed. Announcing it by refusing is what is required**; nothing here obliges a
peer to honour an arbitrarily large deadline.

#### What an expired sub-dispatch surfaces — the code is pinned, the shape is not

**Exactly §7a.1a's treatment, applied to the neighbouring outcome.** When the reentry sub-dispatch
expires on its deadline rather than being refused, **the surfaced code is `recv_timeout`
(`ENTITY-CORE-PROTOCOL` §6.12, status `503`)**. The status **shape** is not pinned — **relayed** (the
handler propagates `503` as its own outer status) and **wrapped** (outer `200`, the timeout carried as
the inner status) are both conformant, as §7a.1a already holds for refusals. **A generic or
transport-catch-all code is non-conformant.**

**Why a scaffold section pins a code §6.12 already owns.** §7a.1a's answer transfers verbatim: the
scaffold is where the outcome is **caught and re-emitted**, and that re-emission is a code path
§6.12's authors were not describing. **A handler that wraps every unsuccessful sub-dispatch in one
generic failure launders a deadline expiry into a transport fault** — and the resulting observable is
indistinguishable from *"the route was broken"*, which is the `connection_broken` row sitting one line
below `recv_timeout` in §6.12's own table. ⭐ **The consequence for a check set is that the code stops
being witness-only here.** In-process, §9.4 makes the representation implementation-defined; at the
scaffold's wire boundary it is this code, so an arm may score it.

#### ⛔ The handler does NOT report elapsed time, deliberately

**The validator is the counterparty and already holds both endpoints of the interval** — it sends the
invocation and receives the response the handler returns when the deadline fires, so time-to-outcome
is measured **externally, on one clock, by the party that would be misled if the peer were wrong**,
with the check's clock-tolerance axis absorbing the transit terms.

**A self-reported duration would be the peer grading its own timing on the one axis under test**, and
a peer whose request-layer deadline is broken is exactly the peer whose self-report cannot be trusted.
The `result` shape is therefore unchanged: `{ status: uint, result: <downstream result entity> }`.

#### ⭐ Why N concurrent invocations are a valid driver for §6.11(a), and which rule makes them one

**§6.11(a)'s subject is the connection, not the invocation** — *"multiple outbound EXECUTE calls on
the same pooled connection MUST be able to proceed concurrently"* — and §7a.2a pins every reentry
origination to the caller's own inbound connection. So N concurrent invocations of `dispatch-outbound`
place N outbound EXECUTEs on one pooled connection, which is the clause's own wording. **A fan-out
form — N originations from one invocation — is not needed to measure it.**

**The premise that needs holding up is in §4.8, not §6.11:** if a peer processed inbound EXECUTEs on
one connection one at a time, the N invocations would be serialized before reaching the handler and
the probe would measure nothing while the peer looked conformant. **§4.8 forbids that at MUST level**
— while a handler is processing a frame, the peer MUST be able to read and dispatch further frames on
that same connection — and its own reconciling sentence closes the apparent escape its permissions
open: bounding concurrency is allowed, but *"inbound frame processing [must] not block on outbound
dispatch from the same connection"*, and **a per-connection concurrency cap of one blocks inbound
dispatch on a handler that is itself awaiting an outbound response.**

⚠ **What N invocations cannot do is attribute a failure.** A peer serializing its **outbound** dispatch
(§6.11(a)) and a peer serializing its **inbound** dispatch (§4.8) produce the **same observable** — the
originations arrive one after another. The check correctly fails either way; **naming which rule broke
is Stage 2 triage and needs a different input**, which is what a fan-out form would provide. It is not
minted here: the evidence for it is a real run where the two cannot be told apart.

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

**RATIFIED — shape (a), in-band params.** *(The validator seat's call, as this section reserved it, and now exercised on the wire by both `0.8.2.19` discriminators rather than only leaned toward.)* **The reentry authority travels nested in the `dispatch-outbound` params** — `reentry_capability` + `reentry_granters` + `reentry_cap_signatures` — **not** in the envelope `included` set: self-contained, transport-agnostic, and it does not depend on a session API exposing `included`. The record of how it was decided follows.

**How it was decided (Go's call — Go builds the validator side):** how the caller hands the reentry capability to `dispatch-outbound`. The reentry direction can only be authorized by the caller (a cap valid *at the caller*). Two shapes: **(a) in-band, nested in params** (`reentry_capability`/`reentry_granters`/`reentry_cap_signatures`, plural since `0.8.2.19` — see the params contract in §7a.1) — self-contained, transport-agnostic, no reliance on an included-set convention; or **(b) the envelope `included` set** (how caps normally travel) with a hash reference in params. **All six-impl-leaning + arch-leaning: (a) in-band params** — all three keystone peers already implement it, and it doesn't depend on the high-level session API exposing the `included` set. Go ratifies (or flags (b) — then it's one uniform ×N change, not divergent rework). Secondary confirm: the validator drives `target` as **itself** (B-role on the same connection), not a third-peer dial.

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

## §7c Compute conformance: the differential corpus (BUILT and cross-blessed; **not frozen, not vendored**)

The **third conformance modality**. Wire ECF (§§1–6) is enumerable golden fixtures; concurrency (§7b) is
behavioral probes; **compute is *differential* over a combinatorial input space.** This is the conformance
artifact for **AE-1** (`PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION` — an alternate execution strategy is conformant
iff materialized-boundary-equivalent). This section is arch's guidance to the core peers to build and
**converge** it; it is high-level by intent — the vector shape is pinned, the construction is left to the
cohort.

> **STATUS CORRECTED 2026-08-20 — this section said "UNBUILT" for four weeks after the corpus locked.**
>
> It read: *"Status: no portable cross-impl compute corpus exists … so 'Rust eval == Go eval' is
> **assumed, not verified**."* **All three clauses were false**, and had been since **2026-07-23** —
> **two days** after the 07-21 survey this section was written from.
>
> **What actually exists, verified in all three trees rather than inferred from one:**
>
> | | |
> |---|---|
> | Generator + emit + cross-bless | `entity-core-go/cmd/internal/compute-corpus/` — `gen.go`, `evaluate.go`, `crossbless.go`, `guards.go`, `peeremit.go`, and a **portable `prng.go`** |
> | Pin | **`seed 20260716` · SplitMix64 · corpus-SHA `d0fdd757`** — the exact `(seed, generator-version, SHA)` triple §7c.4(6) asks for |
> | Result | **330/330 byte-identical, three-way** (`EXTENSION-COMPUTE` v3.21 header) |
> | Corroboration in each seat | go `docs/validation/reports/2026-07-22-compute-corpus-first-crossimpl-run.md` · rust `ROUTING-2026-07-23-compute-corpus-f1-q1-rust.md` · py `HANDOFF-2026-07-23-compute-corpus-f3-included-resolution-python.md` — **one per impl, each recording its own F-finding** (F-1 rust cast-to-uint, F-2 the §9.1 out-of-range index, F-3 py tier-1 `included`) |
>
> **F-2 is why this mattered rather than being filing hygiene.** That ruling — *an out-of-bounds magnitude
> is `index_out_of_range`, not `type_mismatch`* — came **out of this corpus's first cross-impl run**, and
> on 2026-08-20 `EXTENSION-COMPUTE` v3.24 re-litigated it in the losing direction and a seat implemented
> the contradiction. **A guide saying the instrument does not exist is a guide nobody consults when
> deciding whether a question was already settled.**
>
> **What is genuinely still open, and it is now the only unmet step:** §7c.4(5)**(e)** — *freeze, MANIFEST,
> vendor*. There is **no frozen vector dump and no MANIFEST anywhere in the checkout**, and
> `entity-core-protocol/specs/test-vectors/compute-conformance/` — the canonical home §7c.5 declares — is
> **absent at `89e4525`**. The corpus today is a **regenerable artifact in one seat's tree**, reproducible
> only by running go's generator at the pinned seed.
>
> **So the honest state is "built, cross-blessed, and homeless," and the risk is specific**: a regenerable
> corpus is one refactor of `gen.go` away from silently changing what 330/330 means, and the SHA that would
> catch it is quoted in a spec header rather than pinned in a MANIFEST beside the bytes.

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
6. **Reproducibility discipline (§3.6 applies).** The corpus carries `(seed, case-count, generator-version)`;
   a failure reproduces from `(seed, case-index)`. Decode the artifact, not just its SHA (§3.6(6)); skips count
   (§3.6(2)); no "pre-existing" without a bisect (§3.6(1)).

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

## §7c.6 The v3.25 corner vectors — arch-authored worked vectors `[2026-08-20]`

**Six worked vectors for `EXTENSION-COMPUTE` v3.25's four corner rulings (C-1…C-4).** Authored here
because §7c.5 puts *"the guidance + the vector shape + the discipline"* on arch and the **emit + cross-bless
stages on the cohort** — the same split as §7c.4(5)(a)'s nine lowering-toolkit worked lowerings, which is
the precedent these follow.

**They are not a corpus and must not be run as one.** They are six cases to seed into the existing corpus
at the next regeneration. §7c.4(2)'s anti-vacuity guards are properties of the **whole** corpus: on their
own these six carry only **two** distinct error codes against the `≥3` guard, so a run of just these would
be vacuous by the corpus's own rule. **Seed them in; do not score them separately.**

**Arch supplies the IR and the expected outcome. Arch does not supply boundary hashes** — those are the
emit stage's output (§7c.4(5)(b)), and a hash arch computed by hand would make arch the oracle, which
§7c.4(3) forbids: *"for the cross-impl corpus, no impl is privileged … the frozen vector's boundary-hash is
fixed by the spec, not by Go."*

| # | Ruling | IR (`(IR entity, root bindings, budget)`) | Expected outcome |
|---|---|---|---|
| **CV-1** | C-1 · `group-by` result shape | `apply(builtins/group-by, {collection: [1,2,3,4], fn: λx. mod(x,2)})` | **value** — `[ group{key:1, members:[1,3]}, group{key:0, members:[2,4]} ]`. Two `system/compute/group` entities; **group order is first-appearance** (`1` first, because element `1` yields key `1`), **member order is input order**. Boundary = the materialized bare-entity hash |
| **CV-2** | C-2 · `assoc` out-of-range | `apply(builtins/assoc, {collection: [10,20,30], index: -1, value: 99})` **and** the same with `index: 3` | **`error{code: "index_out_of_range"}`**, both arms. **Both arms are required** — negative and `≥ length` are the two magnitudes §2.2's F-2 ruling names, and an impl that special-cases only one passes a single-arm vector |
| **CV-3** | C-3 · `range` domain | `apply(builtins/range, {n: -1})` · `apply(builtins/range, {n: 0})` | `error{code: "count_out_of_range"}` · **value** `[]`. The `n: 0` arm is the **anti-vacuity partner**: without it a peer that errors on *every* `range` scores green on CV-3 |
| **CV-4** | C-4 · **the discriminator** — control vs data | (a) `apply(builtins/assoc, {collection:[1,2,3], index: 1, value: <compute/error E>})` · (b) `apply(builtins/assoc, {collection:[1,2,3], index: <compute/error E>, value: 9})` | (a) **value** `[1, <E>, 3]` — the error is **contained**, it is a write payload, not a consumed operand · (b) **`<E>` itself** — short-circuit. **This pair is the whole point of C-4:** one expression type, two positions, opposite outcomes, so **an implementation that treats all positions uniformly fails exactly one arm whichever way it picks.** A uniform-short-circuit peer fails (a); a uniform-contain peer fails (b) |
| **CV-5** | C-4 · `concat` type-transparency | `apply(builtins/concat, {collections: [[1,2], [<compute/error E>]]})` | **value** `[1, 2, <E>]` — **not** `type_mismatch`. A `compute/error` element does not participate in the element-type match (§1.5's NaN model). Discriminates the reading that inspects elements for errors |
| **CV-6** | C-4 · `group-by` error key | `apply(builtins/group-by, {collection: [1,2], fn: λx. <compute/error E>})` | **`<E>`** — short-circuit. The derived key is **consumed** (compared to assign a group). Guards the tempting containment reading that C-1 makes newly arguable, since the key now has an output position |

**Two properties of this set worth keeping when it is ported:**

1. **CV-4 is the only one that cannot be satisfied by guessing.** CV-1/2/3/5/6 each admit a single wrong
   uniform rule that passes them. CV-4's two arms are the same operation and demand opposite behaviour, so
   it is the arm to port first and the one whose failure is most diagnostic.
2. **Every vector's outcome is `code`-only where it is an error** — per §7c.3, `message`/`at`/`expression`
   are in-flight diagnostics and **not** part of the materialized error boundary, so the gate compares
   `code` strictly and nothing else. An impl whose diagnostics differ is still conformant.

**Satisfaction mode (§5.2b.1): constructible today at every seat with no new surface** — these are
pure-function checks over an evaluator, needing no harness, no live peer, and no capability. **Class
(§7.0): compute differential-corpus vectors (§7c), not the ECF/crypto-agility fixture corpus** — see the
correction note below.

> **Class correction, recorded against arch `[2026-08-20]`.** `PROPOSAL-COMPUTE-V324-CORNERS` §5 and the
> first `COHORT-OPEN-ITEMS` C-5 row declared these **"fixture-corpus vectors (static `.diag` + canonical
> `.cbor`, arch-authored)"**, citing §7.0. **Wrong row of the right table.** §7.0's *fixture corpus* is
> byte-level data **for ECF / crypto-agility**, living in
> `entity-core-protocol/specs/test-vectors/`; an evaluator-behaviour check is not that, and no amount of
> `.diag` would have made it one. The correct home is **§7c**, whose shape is
> `(IR, root bindings, budget) → { boundary-hash | error{code} }` and whose ownership splits
> arch-guides / cohort-builds.
>
> **This is L19's own failure, committed in the packet that invoked L19** — the rule is *say which kind of
> vector, **and check the corpus before inventing the taxonomy***, and arch named a class from §7.0's table
> without reading §7c, which is the section `EXTENSION-COMPUTE` §11.6 points at by name. **Declaring a
> class is not the discipline; opening the section that owns it is.** The cost had it shipped: four
> vectors routed to the wrong authoring split, into a directory built for crypto agility, against a corpus
> that already existed and would have absorbed them for free.

## §7d The extension-host seam (`register_handler`) — the class the wire oracle cannot drive

**Class:** host-seam check. **Driven** in-process by the peer's own harness; **asserted** over the wire
by the oracle. **Authored** by architecture (the reference handler and its expected observable);
**built** per-peer by the generator; **run** by the existing `validate-peer` transport.

### §7d.1 Why this is a fourth class and not a `validate-peer` category

`validate-peer` drives a peer over TCP. **It cannot call an in-process API**, so there is no way to
write a wire check that asserts *"this peer's `register_handler` works."* That is not an oversight in
the oracle; it is the shape of the problem, and it is why the seam went unmeasured for the whole life
of the cohort while every peer reported green.

The mechanism to close it already exists. Every generated peer boots into a conformance mode that
installs handlers a normal run does not have (§7a's `system/validate/*` surface). **The host-seam check
is that hook with one difference: the handlers MUST be installed through the public primitive, not
compiled into bootstrap.** A peer that satisfies §7a by hardcoding its validate handlers into its
bootstrap path satisfies nothing here.

### §7d.2 The harness contract

The peer's host-seam harness performs the following, in order, using **only public API**:

| # | Harness action | Oracle asserts over the wire |
|---|---|---|
| 1 | `register_handler` a reference handler at `app/validate/host-seam/echo` with a **language-native** body | the three `SDK-OPERATIONS` §11.6.1 tree artifacts exist at their paths |
| 2 | — | dispatch to the pattern returns the **native body's** result (§7d.3) |
| 3 | `register_handler` the **same pattern** again; publish the outcome at `app/validate/host-seam/collision` | that entity records `409` |
| 4 | close the handle | dispatch → `404`; all three tree artifacts are gone |
| 5 | `register_handler` at `system/validate/host-seam/ext` — a `system/*` install path | the three artifacts exist |

**Row 5 asserts only that the SDK primitive does not carry a hardcoded namespace refusal of its own**
(`SDK-OPERATIONS` §11.6). Every standard extension owns a `system/{ext}/…` namespace, so a primitive that
refuses `system/*` by prefix cannot install one, and that is the defect this row catches.

> **It is NOT a check that a peer permits `system/*` installation as a matter of policy.**
> `ENTITY-CORE-PROTOCOL` 0.8.2.13 **withdrew** the `system/*` prefix reservation, and withdrawing a
> prohibition does not create an obligation in the other direction: a deployment that declines to install
> extensions at `system/*` after startup, or that issues no grant covering those paths, is conformant.
> **This row runs against the peer's own conformance harness composing its own peer, which is the one
> context where the policy question does not arise.** An earlier draft of this row asserted that the peer
> *"did not refuse `403 forbidden_pattern`"* — that would have made a local policy decision a conformance
> failure, which is exactly the overreach the withdrawal removed.

### §7d.3 The reference body MUST be inattributable to any other mechanism

**The whole class turns on row 2.** The oracle already carries a `core_register_body_binding` check
whose message reads *"entity-native echo body bound at …/expr"* — it binds a `compute/literal` at an
`expression_path`, because that is the body kind a wire oracle can install. **Two body mechanisms reach
one observable — a `200` from a registered pattern — and a check that does not separate them attributes
the result to whichever it can drive.** So:

> The reference body **MUST** return a value that no `compute/literal` can produce — specifically, a
> value derived from **both** a field of the request params **and** state captured at registration time
> (for example `f(params.n, captured_salt)`). A `compute/literal` returns a fixed entity, and a
> compute-expression body cannot see registration-time host state at all.

Without that property the check passes on a peer that implements nothing new, which is the state every
peer in the cohort is in today.

### §7d.4 Both controls are mandatory

A gate is validated in **both** directions before it is trusted, and both runs are recorded in the
check's own message — the `core_register_reserved_*` family already does this:

- **Negative.** A harness mutated to skip §11.6.1's step 4 — tree writes performed, dispatch index not
  bound — **MUST go RED**. Without this control the check passes on a peer that only writes the tree.
- **Positive.** An unmutated harness **MUST go GREEN**, and a peer that declares the class **declined**
  (`SDK-OPERATIONS` §11.6) **MUST report `declined`, not WARN and not FAIL**. A population where most
  peers score inconclusive is the tell that the reference answer is wrong, and this class starts against
  a cohort in which several peers genuinely have no first-class callable to bind.

**`declined` is a conformance outcome, not a gap.** A peer that composes a fixed extension set at build
time rather than at runtime is a legitimate peer; what is not legitimate is being unable to tell that
peer apart from one that simply never implemented the seam.

### §7d.5 Consumer ordering — the one row this class owes that is not about `register_handler`

`SYSTEM-COMPOSITION` §2.2 fixes the order in which emit consumers run, and states its own failure in
**wire-observable** terms: reversing the subscription and revision positions produces *"subscribers
seeing a change without a version entry."* **Nothing in any conformance category tests consumer
ordering**, and the ordering is set by registration order in peer-owner wiring code — the same
hand-written composition this seam exists to let a generator replace.

One row, and it is drivable with no new surface:

> Install REVISION and SUBSCRIPTION on a peer, subscribe under a tracked prefix, write to a path under
> that prefix, and assert the **version entry is present in the notification the subscriber receives**.
> A peer whose auto-version consumer runs after subscription notification fails this and is otherwise
> indistinguishable from a conformant one.

This row is filed here rather than under a per-extension category because the property is a fact about
the **composition**, not about either extension — §2.4 is explicit that the ordering applies to whichever
consumers are present, so no single extension's spec owns it.

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
| **Capability handler — `validate-peer` test vector update** | **NEW pending (post-v7.62; expanded by the v7.62 cross-impl closeout)** | New `validate-peer` vectors needed: (a) `request` subset-validation (caller asks within bounds → 200; exceeds caller's cap → 403 `scope_exceeds_authority`; exceeds policy entry → 403); (b) `revoke` happy path (marker written at `system/capability/revocations/{hex}`; subsequent `is_revoked` returns true); (c) revoke authz (caller without revoke-cap → 403); (d) `configure` writes a `policy-entry` at `system/capability/policy/{peer_pattern}` (exact + `default` lookups — v7.63 F8 renamed the fallback segment from `*` to `default`); (e) §4.4 handshake union (peer with policy entry for caller gets floor + policy grants; peer without entry gets floor only); (f) `501 unsupported_operation` on registered-but-not-implemented op (distinguishable from 404 / 403). **Added by the v7.62 closeout (see the v7.62 capability-handler cross-impl closeout review):** (g) **501-before-403 dispatcher ordering** — call a registered-but-unimplemented capability op with an authz-deficient cap; expected `501 unsupported_operation`. The dispatcher MUST check manifest-op existence (→ 501) **before** cap-authz (→ 403). V7 §6.2 line 2724 already binds this normatively ("the caller's authority is irrelevant"); this vector pins it as a conformance item. Go discovered the masking trap as a real bug while authoring v7.62 vectors (fixed in Go cycle); other impls plausibly have the same ordering bug. (h) **`delegate-zero-parent` → `400 bad_request`** — zero-hash `parent` field on `delegate-request` is structural input defect (SEC-18 zero-hash + §3.6 M3 normalization precedent), NOT a "not found" lookup miss. Three impls split: Go = 400, Rust + Python = 404. Pin 400. (i) **wire-only cap revocation distinguishing** — request a cap (inline-returned, no tree path) → revoke it → re-present in EXECUTE. Expected: rejection via marker-check in `verify_request`. Surfaces whether each impl actually wires `is_revoked` through the marker path or only catches bound-path revocation. (j) **cross-peer `delegate` rejection** — remote caller A invokes `delegate` on B. Expected: `501 unsupported_operation` (V7 v7.63 F1 narrows `delegate` to same-peer-only for v1). Pair vector: same-peer self-attenuation happy path — local-peer-as-caller invokes `delegate`; child cap chain-verifies under §5.5. **Added by V7 v7.71: authorization-denial status+code matrix** — pins the §3.3-403 / §5.2 verdict-to-status / `result.data.code` contract that the Rust SHA-384 cohort's `verification_failed` residual exposed. The matrix is the enforcement half of the v7.71 §3.3 normative MUST (catch-all defaults outlawed on authorization paths). (k) `AUTHZ-DELEGATE-GRANT-1` — delegate without `system/role:delegate` grant (PR-8.2) → `403 capability_denied`. (l) `AUTHZ-DENY-DEFAULT-1` — generic `verify_request` DENY (e.g. cap doesn't cover op) → `403 capability_denied`. (m) `AUTHZ-SCOPE-EXCEEDS-1` — §6.2-dispatch-level `request` exceeds policy/caller authority → `403 scope_exceeds_authority`. **NOTE**: overlaps with (a); (a) tests the cap-handler `request` op specifically, (m) tests the dispatch-layer general case — cohort SHOULD reuse the fixture, treating (a) as the specialization. (n) `AUTHZ-GRANTEE-1` — unresolvable grantee (§5.2 single 401 carve-out / PR-3) → `401 unresolvable_grantee`. (o) `AUTHZ-REVOKED-1` — revoked cap on use (EXTENSION-ROLE §5.5 in-flight cascade) → `401 capability_revoked`. Complements (i) which verifies the marker mechanism wires through; (o) verifies the surfaced status+code. (p) `AUTHZ-NO-CATCHALL-1` — granter identity unresolvable during chain verify (§5.5 `granter is null → DENY`) → `403 capability_denied`, **NOT** `verification_failed`. The exact regression pin for the Rust residual. (q) `AUTHZ-EXPIRED-1` — capability used after `expires_at` (§5.6 / §5.2 validity check) → `403 capability_denied` (NOT a separate `capability_expired` string — pins that expiry uses the default code). Authoring owner: Go per the cross-impl test-vector convention. **Added by the 0.8.1 CAP-1..CAP-7 fold (core-protocol `30ca731`; arch proposals `PROPOSAL-CAPABILITY-EMPTY-GRANTS-AND-POLICY-WITHDRAWAL`, `PROPOSAL-CAPABILITY-MINT-TEMPORAL-CEILING-AND-THE-WITHDRAWAL-BOUND`, `PROPOSAL-POLICY-PATH-KEY-FORMS-6-2-VS-6-9A-1`):** (r) **`configure` accepts `grants: []`** — write succeeds, entry is readable, and an exact-match empty entry **suppresses the `default` fallback** (assert a subsequent `request` from that peer gets `403 scope_exceeds_authority` rather than `default`'s scope). Pin CAP-2/CAP-3. (s) **entity-native dispatch with an empty handler grant** (`entity_native` category, not `capability`) — a pure-functional expression handler whose grant at `system/capability/grants/{pattern}` has `grants: []` **dispatches, expect `200`, NOT `403 permission_denied`**. Pin CAP-1. This is the check whose absence let §6.1 and §6.8 contradict each other unnoticed. (t) **the `request` mint temporal ceiling** — caller cap expiring at `T`, `request` with `ttl_ms` far beyond `T`. **Assert `200` AND `minted.expires_at == MIN_DEFINED(...)` exactly.** A `<= T` assertion is **non-conformant as a check**: it scores clamp-and-mint and reject-outright identically, which is the precise way this defect hid in two impls at once. Pin CAP-5. (u) **`ttl_ms: 0` and overflow** — `0` yields a token expired at every observable instant (expiry is an **exclusive** bound); an unrepresentable `created_at + ttl_ms` contributes **no ceiling** (term absent — MUST NOT wrap, MUST NOT saturate). Pin CAP-6. Note the encoded `expires_at` differs between absent and saturated, so this is wire-observable. (v) **policy-path key-form resolution order** — hex, then Base58 peer-id, then `default`; partial prefixes rejected in **either** encoding. Pin CAP-7. Authoring owner: Go, same convention. |
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
- **`docs/proposals/implemented/extensions/PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION.md`** (AE-1) — the admission contract §7c is the conformance artifact for; **`PROPOSAL-COMPUTE-LOWERING-TOOLKIT.md §5`** — the worked-lowering vectors that seed the compute corpus; **`ROUTING-2026-07-21-cross-team-sequencing.md`** — where the corpus sits on the cross-team clock (the critical path). Grounding survey: `entity-workbench-go/entitysdk/axis1_equivalence_test.go` + `entity-workbench-go/docs/architecture/reviews/COMPUTE-AXIS1-ORACLE-GAPS-2026-07-16.md`.
