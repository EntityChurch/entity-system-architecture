# PROPOSAL — CONTINUATION: define the `deliver_token`, and say what Level 2 consults

**Status:** **DRAFT — reference proposal, written after the fold. See §0.** Folded as v1.22.
**Target:** `EXTENSION-CONTINUATION.md` §3.6a (new), §3.1b, §4.2.
**Tier:** `extensions/` — validated by go · rust · py.
**Scope:** the internal delivery token named by §3.6's pseudocode, and the authority a continuation
advance runs under. **Not** the standing-continuation model, which is closed.
**Source:** `entity-core-go`, 2026-08-14 — three asks about continuation advance authority — **plus a
fourth, independent ask from `entity-core-rust`** (§4.3's chain bundle, "the rexec seam"), which the
first pass did not credit and which is the whole of §3's third bullet. See §7.
**Cohort review:** ruled in-cycle; **built and measured in go the same day** (§4).

---

## 0. Process deviation — recorded, not hidden

**Folded at `808f4a8` with no proposal.** It defines a wire-adjacent value and pins an authority rule.
Written now per the reconstruction ledger (arc H). Not back-dated. **First-pass reconstruction.**

## 1. The most useful thing in this arc: two of three were already ruled

Three asks arrived framed as spec silence. **Two of them the corpus already answered**, and that is
the finding worth keeping — **the implementations had diverged from a spec that already said the
answer, not from a spec that was silent.**

| The ask | The landed answer |
|---|---|
| What does Level 2 (`check_path_permission`) consult? | **V7 §6.8 is not silent** — *"caller-specified paths… the caller's capability is the authorization"*, and *"propagated caller capability is not a dispatch gate"* MUST-NOTs the fallback in as many words |
| Is an advance a new root, or the caller's chain? | **§3.1b, landed:** *"it always advances under its stored `dispatch_capability`… regardless of who reached it. A trigger reaches it; it does not own it."* **New root** |
| Must the `dispatch_capability` cover the resource? | **§4.2, landed:** *"The capability MUST grant the continuation's `target`, `operation`, and `resource`."* Already required |

**This is the strongest available evidence that the design is sound:** when the seats disagreed and
the text was consulted, the text was right and specific. It is also the argument against treating
every routed divergence as a spec gap — **the first move is to read the landed text, not to draft.**

## 2. §3.6a — the token that was named once and defined nowhere

§3.6 Step 4 read:

```
execute.deliver_token = generate_internal_deliver_token(continuation.data.deliver_to)
```

**That name appears exactly ONCE in the corpus — at its own call site.** No section, guide, or
appendix said what it returned.

**So "each impl improvised" was not looseness; there was nothing to be faithful to.** The
improvisations agreed **same-implementation** and disagreed **across the wire** — go's B fell back to
the connection's session capability, which is why go→go passed while the cross-impl pairing did not,
and go's own source comment said as much.

**This is the canonical instance of the rule the corpus had already written down and not
mechanized:** *a `MUST` may not name a referent the corpus does not define.* `EXTENSION-REGISTRY`
§6a.9 stated it as prose beside one instance; **this was the second live instance, and it had already
cost two cycles of cross-implementation misrouting.** It is the evidence that forced **D10** — *a rule
that has recurred twice buys a mechanism or gets retired* — and it is now caught by
`undefined-wire-referent` in `entity-system-arch-tools`.

## 3. What v1.22 pins

- **§3.6a defines the `deliver_token`** — what it is derived from, what it grants, and its scope.
- **The advance re-roots at its own `dispatch_capability`**, and the connection-authority fallback is
  **removed** (it is the mechanism behind the same-impl/cross-impl split above).
- **§4.3's bundle MUST carry a `system/peer` identity for every `granter` *and* every `grantee`** in
  the transported chain — not best-effort, and a bundler that cannot resolve one **fails at bundle
  time** with `chain_unreachable` rather than dispatching an incomplete bundle.

  **This is `entity-core-rust`'s ask, and the ask named the consequence of ruling it this way.** rust
  read both bundlers at source — go's `core/capability/chainbundle.go` and their own
  `core/protocol/src/verify.rs` — and found them **structurally identical**: both `continue` past an
  unresolvable signer, and neither collects grantee identities except where a grantee also happens to
  be a granter. Their framing is the reason this is a spec fix rather than a bug report:

  > *"A MUST on the verifier paired with best-effort on the bundler is an interop bug by
  > construction — the same shape as the `pending_hash`-with-no-referent trap, one layer out.
  > **Neither seat is wrong; there is nothing to be wrong against.**"*

  **So ruling it "must" converted a spec gap into a known implementation gap in all three trees**, by
  rust's own prior statement of what each answer would imply — *"if it must, `collect_chain_bundle`'s
  best-effort omission is an error in all three impls … and we will fix ours."* **That consequence is
  a deliberate, accepted cost of the ruling, and it belongs in the record rather than only in §5's
  "rust and py owe the v1.22 changes."**

## 4. Build state

**Built and measured in go the same day** (`26675e9`): **99 P / 0 F in all three pairings.** Both
blocking asks resolved against the fixtures exactly as §3.6b directed.

**Their diagnosis is worth recording** because it explains why the defect was invisible: the
validator's dispatch capabilities were scoped **peer-relative**, so the fix canonicalized them against
the validator's namespace rather than the executing peer's — **they had only ever matched because the
advance ran under the delivery's broader capability.**

**rust and py owe the same four changes.** *(rust's §4.3 fail-closed bundler, built against this
version, then produced the reentrant `dispatch-outbound` 502 — found, established as impl-correlated,
and fixed at `c09d402`. That is downstream of this fold, not a defect in it.)*

## 5. Open questions for the cohort

1. **rust and py owe the v1.22 changes**, and §4.3's identity-bundle MUST is the specific one:
   **all three bundlers omitted grantee identities best-effort**, so this is an accepted,
   ruling-created gap in every tree rather than two seats lagging one. go closed theirs at `26675e9`;
   rust said in the ask that they would fix theirs if it ruled this way; py has not been asked.
   Until all three land, the three-way 99/0F is a go-measured number against fixtures, not a
   cross-impl convergence claim.
2. **Was the connection-authority fallback load-bearing anywhere else?** It was removed as a defect;
   nothing has swept for other reliance on it.

## 6. Fold plan

**Already folded** (v1.22). Ratification means §3.6a's definition and the re-rooting stand.

## 7. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8.

**Sources opened:** `entity-core-go`
`docs/status/ROUTING-2026-08-14-the-rexec-seam-was-ours-and-fixing-it-exposed-what-was-holding-it-up.md`
(§2's "The ask (arch)" — **three numbered asks, counted**, plus its closing routing list) ·
`entity-core-rust` `docs/status/ROUTING-2026-08-13-q-*.md` (**ask 5**, in their own words, not via
go's relay of it). Landed text read at `EXTENSION-CONTINUATION.md` §4.3.

**Build state re-pinned 2026-08-15:** go `2df96f8`, rust `462f2c2`, py `f33526f`, all clean.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **H1** | A | **The arc has four inputs, not three.** rust's ask 5 is the sole source of §4.3's v1.22 MUST and was credited to nobody; the header said *"three asks"* | **Header + §3 corrected** |
| **H2** | B | **The ruling's accepted cost was not recorded.** Ruling "must" makes `collect_chain_bundle`'s omission an error **in all three trees** — which rust stated in advance as the consequence of that answer | **§3 expanded; §5.1 sharpened** |

**Why this one is worth catching even though the ruling is right.** go's packet *relayed* rust's ask
(*"rust's ask 5 …"*), and the first pass reconstructed the arc from go's packet. **Reading the
relaying seat's document instead of the filing seat's is the same shortcut as `REVISION` v3.11** — it
produced a correct ruling here and a mis-credited record, which is the version of the failure that
does not announce itself.

### What the review confirmed as correct — and this is the arc that matters most

**§1's table is exact.** go asked three things; the corpus already answered two, in the words the
proposal quotes, at the sections it names. **Verified against go's own numbered asks and the landed
text, not against arch's summary of either.**

**This is the strongest evidence in the whole reconstruction that the design is sound**, and it
survives review intact: when the seats disagreed and the text was consulted, **the text was right and
specific.** The handoff nominated this as the cycle's one genuinely good sign; it is.

**Also confirmed:** §2's *"appears exactly ONCE in the corpus — at its own call site"* is the
`undefined-wire-referent` class's second live instance and is correctly identified as the evidence
that forced D10. §4's go build (`26675e9`, 99 P / 0 F in all three pairings) and its diagnosis —
peer-relative dispatch capabilities that *"had only ever matched because the advance ran under the
delivery's broader capability"* — are go's own words, correctly attributed.

**Worth crediting, because it is the behaviour this process wants:** rust scoped their ask honestly —
*"we have not run your repro … what is verified here is the mechanism, not the incident"* — and pinned
the decision point with two named tests rather than asserting the incident. **An ask that states what
it has not verified is worth more than one that does not.**

### Not covered by this pass

- **§5.2 is unchanged and still unswept** — nothing has looked for other reliance on the removed
  connection-authority fallback.
- **rust's and py's v1.22 work was not re-measured**; §5.1's caveat stands, and H2 sharpens what
  specifically is owed.
