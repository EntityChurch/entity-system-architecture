# PROPOSAL — the registry's fourth actor: check the name you were already given, and bound what you cannot check

**Status:** **IMPLEMENTED 2026-08-18** — all twelve deltas folded, each verified against the tree.
D1 §6a.4 (`binding.name == norm`) · D2 §3 step 2a · D3 §6a.3 + §6a.4 (finite `ttl`) · D4 new §6a.1a
(the fourth actor) · D5 §11.1 (three vectors) · D6 `guides/GUIDE-RESOLUTION` §7a · D7 §2.1 + §3b
(`ttl` units — §3 left untouched as the canonical home) · D8 §6a.4 fail-closed diagnostics ·
D9 new §6a.3a (enumeration: the walk is the authority, the listing is a menu) · D10 §6a.3
(non-empty `transports`) · D11 §6a.9.2 · D12 §6a.9. `EXTENSION-REGISTRY` 1.6 → 1.7.
**Tier:** extensions — `EXTENSION-REGISTRY` §3, §6a.3, §6a.4, §6a.6, §11.
**Answers:** `entity-browser-rust` `ROUTING-2026-08-17-e` Findings 1 and 2 and the §4 design question
(`dev` @ `ce19e9f`), with their core-go row corrected.
**Read at:** arch `1e635f2` · browser-rust `ce19e9f` · core-rust `1a6955b` · core-go `41ecf0b` ·
core-py `81f7d0d` · legacy `b42fd2e`

---

## §0 Summary of the ruling

| Ask | Ruling |
|---|---|
| **F1 — the name→binding association is host-chosen and never checked** | **Confirmed, and it is a spec defect, not two impl slips.** Neither §3's receiver-verification steps nor §6a.4's pseudocode compares `binding.name` to the queried name. **3 of 3 impls affected** — their `core-go` row is wrong, see §2. Fix is one comparison, MUST. §1 |
| **F2 — a static host can withhold a revocation** | **Confirmed, and unfixable by any commitment a hostile origin serves.** The stated bound is *"ttl + revocation"* — and **§6a.4 explicitly permits `ttl: null` on a peer-issued binding**, so on a null-`ttl` binding the bound is not weak, it is *absent*. Rule: peer-issued bindings MUST carry non-null `ttl`. §3 |
| **§4 — is the by-name association meant to be committed or asserted?** | **Committed — and it always has been.** The premise that per-binding signatures give authenticity but not association is **wrong in our favour**: the signature covers a body containing `name`, so `sig(R, {name, target_peer_id})` *is* the commitment. The swap works only because the verifier discards a field the registry already signed. **No new mechanism. No registry signed-root MUST.** §4 |
| *(not asked)* | **`core-rust`'s `is_revoked` enumerates a prefix where §6a.6 and the source proposal's P7 ruling say O(1) by-target index.** Separate, conformance-shaped, verified in their tree. §5 |
| *(corpus)* | `PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND` — the document both impls' soundness comments cite — **was not in this corpus.** It exists in legacy and is pulled in with this proposal. §6 |
| **F3 — `ttl`/`issued_at` units unstated** `[2026-08-18]` | **Stated at §3, which they did not reach from §6a — and checking it found worse.** `ttl` is declared three times in three phrasings; **§3b declares a duration as an instant** in a normative field declaration. Their `None`-collapse point is granted. §10 |
| **F4 — a signed root cannot be enumerated** `[2026-08-18]` | **Wrong, in their favour — it can.** Trie leaves are `[key, value_hash]`; walking from `root_hash` yields the committed key set. Publish at `prefix: "system/registry/binding/by-name/"` and the walk **is** the browse — and hiding a name then requires withholding a node, which makes the walk fail visibly. §10 |
| **F5 — the manifest at a tree-leaf URL** `[2026-08-18]` | **Confirmed, self-corrected, and now the second seat on one seam.** Strengthens `…-MANIFEST-FRESHNESS` D5. Their process lesson lands as **AP-19**. §10 |
| **F6 — empty `transports` is resolvable and useless** `[2026-08-18]` | **Confirmed.** §4.1.2's §6.5 fall-through does not exist for a static publisher (§6.5.4: discovery is out-of-band in v1). Peer-issued bindings MUST carry transports; the pin carve-out stands. §10 |
| **Should the ISSUER also refuse to mint a null-ttl binding?** `[core-go spec-issue 2026-08-18-a]` | **Yes — and their lean is right in direction but sends the error to the party that cannot fix it.** D3 makes non-null `ttl` a property of a *valid binding*, so the issuer must not mint one. But when a request omits `requested_ttl` **and** the policy defines no `default_ttl`, that is an **operator misconfiguration**, and a 400 at register time blames the requester for it. **Gate it at policy-write**, exactly as §6a.9.2 already does for `domain-control`. §12 |

---

## §1 F1 — the registry already signed the association; the verifier throws it away

**Confirmed at the spec, which is where it starts.** Two places define what a receiver checks, and
neither mentions the name:

- **§3, receiver verification** — three steps: locate the signature by target-matching, verify it
  cryptographically against the issuer's key, apply receiver policy to the `trust_anchor`.
- **§6a.4, the normative pseudocode** — `require sig.target == binding_hash` · `require sig.signer ==
  content_hash(canonical(registry.system_peer))` · `require verify_crypto(sig, pinned_key_of(registry))`
  · `require ("peer_issued:" + registry) in config.accepted_trust_anchors` · `require not revoked(…)`
  · `require binding.issued_at + binding.ttl > now()`.

`norm` is computed at the top of that pseudocode and used **only** to build the by-name path. It is
never compared against `binding.name`. The `ResolutionResult` it returns carries no name either, so
no caller can re-check it downstream. **A resolver that implements §6a.4 exactly as written is
vulnerable, which is the definition of a spec defect.**

**The framing that makes the fix obvious.** The binding body carries `name` and the registry's
signature covers the body. So the registry has *already* committed to "R asserts *this name* → *this
peer*". The static host does not forge that commitment and cannot — it substitutes **which** signed
commitment answers the query, and the consumer never notices because it never reads the half of the
commitment that would tell it. **The association is committed; the verification is what is missing.**

That also fixes the language for the cohort: this is not *"per-binding signatures do not give
association."* It is *"the verifier discards the association the signature already covers."*

### The MUST

> **§6a.4, new step in the VERIFY block:** `require binding.name == norm` — the resolver MUST compare
> the queried name (NFC-normalized per §6.3) against the `name` field of the **signed** binding body,
> and MUST refuse and advance the chain on mismatch. This is fail-closed per §6a.4's existing rule and
> is not a downgrade to `out_of_band`.
>
> **§3, receiver verification, new step 2a:** for any binding kind carrying an `issuer_signature`, the
> receiver MUST check that `binding.name` equals the name under which the binding was located. A
> signature proves *who issued it*, never *what it was issued for*.

**Cost: zero fetches, zero new artifacts, zero publishing changes.** Every implementation already
decodes the body before surfacing a result — `core-go` decodes it for the TTL check two steps earlier.
The comparison is reachable from data already in hand at the point the ruling assigns it (L12).

### Why the existing vectors all pass while every impl is open to this

`REG-PEERISSUED-VERIFY-FAIL-1` covers a binding signed by a **non-pinned key** — the host forging its
own binding. That is correctly rejected everywhere, and `entity-browser-rust` asserts it in the same
test file as the swap. **The substitution is the other half of that vector's own failure message**
(*"any peer able to serve the registry's by-name index could bind any name"*) and no vector exists for
it. Their `REG-PEERISSUED-NAME-SUBSTITUTION` request is granted in §7 (D5) — and the shape is worth
naming: **a suite in which every implementation passes every vector while all three share one hole is
not evidence of convergence, it is evidence the suite does not reach the surface.**

---

## §2 The cohort table, corrected — it is 3 of 3

Their §2 table reads `core-go`: *"no peer-issued resolve backend located. `by-name` appears only under
`cmd/internal/validate/` — go is the harness here. If go has a resolver we did not find, this row is
wrong and we would like to be told."* **The row is wrong, and asking was right.**

`entity-core-go` @ `41ecf0b` has `ext/registry/peerissued/` — `peerissued.go`, `register.go`,
`manifest.go`, `httppoll.go` and five test files. `(*Backend).Resolve` implements the full §6a.4
algorithm: normalize → manifest-or-pointer lookup → signature verify against the pinned key → decode
body → revocation → TTL → surface. `by-name` also appears in `ext/registry/peerissued/peerissued_test.go`,
`cmd/registry-issue-binding/main.go`, `cmd/peerissued-fixtures/main.go` and `core/types/registry_ext.go`
— the search that produced the row did not reach them.

**And go has the same hole.** `Resolve` decodes `body` for the TTL check and surfaces
`body.TargetPeerID` without ever comparing `body.Name` to `normalized`. Guards on the path accounted
for, not just the surfacing line: there is no name check in `lookupBinding`, `verifyBinding`, or
`manifestLookup`.

| impl | pinned at | status |
|---|---|---|
| `entity-core-rust` | `1a6955b` | **affected** — code read (arch) + behaviour measured (browser-rust) |
| `entity-core-py` | `81f7d0d` | **affected** — code read, two seats |
| `entity-core-go` | `41ecf0b` | **affected** — code read (arch), `(*Backend).Resolve`, **row corrected** |

**One thing go has that the others do not**, and it matters for §4: go implements §6a.7's signed
binding-manifest (`manifestLookup`, with `coverage: complete` handled as `authoritativeAbsent`). On
*that* path the association **is** committed, because the manifest is a signed `{name → hash}` map.
It does not rescue the general case — §6a.7 requires fall-through to the per-name pointer on any
manifest failure, so a hostile host withholds the manifest and the pointer path answers — but it
means the committed-association machinery is built, not hypothetical.

---

## §3 F2 — withholding, and the bound that was never enforced

**Confirmed, and their ranking of it above F1 is right.** The swap is detectable by inspecting what
you received. Withholding is not: `is_revoked` / `checkRevoked` asks the host whether the host has
been revoked, and absence is treated as "not revoked". `entity-core-py`'s own comment states the
mechanism — *"Skip the lookup and the peer serves a revoked name while every other check passes"* —
and **not serving it is indistinguishable from skipping it.**

**Their honest note that a signed root does not close this is correct and we are not going to pretend
otherwise.** A commitment structure converts *silent substitution* into *detectable absence* — a fetch
error instead of a wrong answer, which is a real gain — but it cannot manufacture liveness. Withholding
an interior trie node collapses to the same `Ok(None)`, and `PublishedRootClient` has no enumeration,
so *"list this registry's revocations"* is not expressible against a signed root today.

**So the bound is `ttl`, and here is the finding neither seat filed: the bound is not enforced.**

`PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND` §5's *Stale/rollback* row gives the mitigation as *"`ttl` +
revocation check + (live) subscription invalidation."* `EXTENSION-REGISTRY` §6a.3 says peer-issued
bindings carry *"`ttl` set (issued bindings expire, unlike sticky local-names)"* — **in a
parenthetical, describing, not requiring.** And §6a.4's pseudocode says:

```
require binding.issued_at + binding.ttl > now()  (or ttl null)
```

**`(or ttl null)` is the whole finding.** A peer-issued binding with `ttl: null` is permitted by the
normative algorithm, never expires, and — with the revocation withheld — is **permanently
unrevokable**. The stated mitigation degrades to "bounded by ttl", and nothing obliges a ttl to exist.
This is precisely the shape `AGENTS.md` records under **L12**: *when a ruling degrades to "bounded by
X," verify that something enforces X before publishing the bound.* Second instance, second surface,
same session family.

### The MUST

> **§6a.3 / §6a.4:** a binding with `kind: "peer-issued"` **MUST** carry a non-null `ttl`. A resolver
> **MUST** refuse a peer-issued binding whose `ttl` is null and advance the chain. §6a.4's
> `(or ttl null)` carve-out is scoped to kinds whose trust source is local (`local-name`,
> `out-of-band`, `self-certifying`), where stickiness is the point and no third party serves the bytes.

**Why this and not a mechanism.** It is the web-PKI answer — short-lived certificates beat online
revocation precisely because a hostile transport can suppress OCSP but cannot suppress an expiry the
verifier computes locally. `issued_at + ttl` is checked against the consumer's own clock from data
inside the signed body: **the one check on this path that a hostile host cannot influence at all.**
It converts *permanent* revocation bypass into a bypass bounded by the registry's chosen re-issue
interval, and it makes that interval the operator's explicit security knob rather than an accident.

**The honest residue, stated so no one reads this as closure:** withholding still buys the attacker
the remaining TTL, and against a hostile origin publishing is **append-durable, not retract-durable**
— the same position `ROUTING-2026-08-17-d` Q4 took for content, now confirmed to be load-bearing for
revocation rather than philosophical. They were right that this is where it stops being abstract.

---

## §4 The design question, ruled — committed, and it always was

**Ruling: the by-name association is *committed*, the commitment is the per-binding signature over a
body carrying `name`, and a registry is NOT required to publish a signed root over its own tree.**

Their §4 rests on *"per-binding signatures give authenticity. They do not give association
(Finding 1) or completeness (Finding 2)."* **The first half is not right, and the correction is the
cheap fix rather than the expensive one.** `sig(R, {name: "protocol.example", target_peer_id: B, …})`
is exactly an assertion about an association. It gives association to any verifier that reads it.
F1 is not a missing commitment; it is a discarded one.

**Completeness is a different question and a signed root does not answer it either**, as they
established themselves. What *does* speak to completeness is already specified: §6a.7's signed
binding-manifest with **`coverage: complete`** — authenticated denial-of-existence on the
DNSSEC-NSEC / TUF-complete-targets model, format pinned since the source proposal §2.2a, implemented
today in `entity-core-go`. That is the sanctioned answer for "the registry asserts this is all of it,"
and it costs no new mechanism either. It still does not defeat withholding, because a hostile host
declines to serve the manifest and §6a.7 mandates fall-through — which is the correct design (a
consumer must not be denial-of-serviced into failing closed on every name) and the reason §3's TTL
bound is the real backstop rather than a workaround.

**On the registry publishing a signed root: SHOULD, not MUST, and it is not release-gating.**

- **For:** it is the same pipeline they are already building for domains; it makes silent substitution
  of *any* registry tree node surface as a fetch error rather than a wrong answer; the +4.7% trie
  overhead they measured is a rounding error; and §8.1's *"the registry's bindings are entities; the
  substrate's content-fetch machinery applies directly"* means a registry gets it by being an ordinary
  publisher, not by being special.
- **Against a MUST:** it drags every static registry into `EXTENSION-NETWORK` §6.5.6's `signed_pointer`
  closure obligation and the 30 s convergence MUST, for a property the one-line name comparison already
  delivers. **A MUST here would be arch mandating a second mechanism to cover a check we forgot to
  specify** — and per `AGENTS-STANDARD`, an implementation-visible obligation is not the place to
  discharge our own defect.

**So: build `entitychurchregistry.org` with a signed root because it is free and strictly better, not
because it is required, and do not wait on this ruling to do it.** B16's emitter shape does not change:
the artifacts are the same by-name pointers, binding bodies and signatures either way — a signed root
is an *addition* over the same tree, not a different tree. **B16 is unblocked.**

**L11 discharge, because this is a ruling inside a design space.** The study that produced this
space was read before ruling: `PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND` §2.2 (the by-name index),
§2.2a (the signed manifest, including the size-bound / delegated-sharding ladder), §2.3 (the P7
by-target revocation ruling) and §5 (the security analysis), in the legacy tree at `b42fd2e`,
pulled into this corpus with this proposal (§6). The landscape documents it sits under —
`PLAN-REGISTRY-AND-DISCOVERY-LANDSCAPE`, `EXPLORATION-REGISTRY-CRUX-SYNTHESIS-cgid-10-217` — are
located and named; **they are not read, and this ruling is scoped so it does not need them**: it adds
one check to an existing algorithm and declines to add a mechanism. A ruling that *removed* a mode or
pruned the backend set would need them first, and that is not this.

### Where §5's threat model actually went wrong, which is the durable lesson

The source proposal's §5 has seven rows. Every one models the adversary as a **requester** (squatting,
replay, wrong-peer binding), as the **registry itself** (key compromise), or as the **consumer's own
fallback logic** (downgrade to pins). **There is no row for the party that serves the bytes.**

That party did not exist as a distinct actor when §5 was written — the registry was assumed to serve
its own tree, so "the host" and "the registry" were one. The static-CDN deployment splits them, and
**a threat model gains an actor without any sentence in it becoming false.** Every row in §5 is still
correct. The table is still incomplete, and no review of the rows would have found it, because the
defect is a missing row.

The *"a receiver only accepts what their pinned registry signed"* sentence (§5, Wrong-peer binding) and
the *"at worst … stale/older"* bound that `entity-core-py` inherited from it are both true under a
three-actor model and both wrong under four. **That is why this routed correctly rather than being
patched in two codebases** — `entity-browser-rust`'s call.

---

## §5 A separate finding — `core-rust` scans where the spec says index

`extensions/registry/src/peer_issued.rs` `is_revoked` (read at `1a6955b`):

```rust
location_index.list(&revocation_prefix(registry)).into_iter().any(|entry| { … })
```

`EXTENSION-REGISTRY` §6a.6 is unambiguous: *"an O(1) index lookup, **not** a scan:
`system/registry/revocation/by-target/{hex(binding_hash)}`"*, and the source proposal §2.3 lands it as
the **P7 ruling, NORMATIVE**, with the reasoning stated (*"O(1) lookup vs O(N) scan, same
anti-internet-scale reasoning as the manifest"*). `entity-core-go`'s `checkRevoked` reads the
by-target path and is conformant.

**This is not F2 and does not change F2's disposition** — both shapes ask the host and both are
defeated by withholding. It is a conformance divergence with a scaling consequence, and it is the kind
that stays invisible because a scan over a small revocation set returns the same answers. Routed to
`entity-core-rust` as a cohort finding; no spec change.

---

## §6 Corpus — the proposal both impls cite was not in this corpus

`EXTENSION-REGISTRY` §6a opens: *"Full design rationale + the cohort spec-doubt rulings (P1–P7) live
in `proposals/implemented/PROPOSAL-PEER-ISSUED-REGISTRY-BACKEND.md`."* **That file was not there.**
`entity-core-py`'s docstring cites *"the proposal §5"* for a security bound; `entity-core-rust`'s
`RegistryTreeReader` docs carry the same argument. **Two implementations inherited a soundness claim
from a document nobody in this repo could open.**

It is **not** a phantom of the `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` kind (`ROUTING-2026-08-17-l`).
It exists, complete and ratified, in the legacy tree at
`docs/architecture/v7.0-core-revision/proposals/implemented/`, and it is one of the 32 pull-ins on
`STATUS-2026-08-17-the-dangling-citation-worklist-is-38-documents-not-585.md`'s worklist. **That
distinction is worth stating loudly**, because the fixes are opposite: a phantom's citation is deleted,
a pull-in's citation is made to resolve. Checking which one it was cost one `find` and would have
produced a false finding against three impls if skipped.

**Pulled in with this proposal**, unmodified, per the STATUS doc's rule (*"copy it, do not rewrite
it"*). It moves this item off the 32 and closes the citation at `EXTENSION-REGISTRY` §6a.

---

## §7 Spec deltas (executable form)

| # | File | Section | Edit |
|---|---|---|---|
| **D1** | `EXTENSION-REGISTRY.md` | §6a.4 | Add `require binding.name == norm` to the VERIFY block, positioned before the revocation check, with the one-sentence reason: a signature proves who issued a binding, never what it was issued for. Fail-closed per the existing rule. |
| **D2** | `EXTENSION-REGISTRY.md` | §3 | New receiver-verification step 2a: for any kind carrying an `issuer_signature`, the receiver MUST check `binding.name` against the name it looked the binding up under. Placed here because it is not peer-issued-specific — every signed kind (`dns-txt`, `well-known-url`, `did-web`, `consensus-anchored`) inherits the same substitution when its backend ships. |
| **D3** | `EXTENSION-REGISTRY.md` | §6a.3 + §6a.4 | `kind: "peer-issued"` MUST carry non-null `ttl`; resolver MUST refuse null and advance. Scope §6a.4's `(or ttl null)` to the local-trust kinds. State the reason: `issued_at + ttl` is the only check on this path a hostile byte-server cannot influence. |
| **D4** | `EXTENSION-REGISTRY.md` | §6a.1 or a new §6a.0a | **Name the fourth actor.** One short paragraph: the backend's reads are transport-agnostic, so the party serving the bytes is not necessarily the registry, and a static origin is a distinct actor with distinct powers — it may substitute which signed artifact answers a read, and it may withhold one. What that actor cannot do: forge a signature, alter a body, or move the consumer's clock. This is the frame D1 and D3 follow from, and its absence is why §5's threat model reads complete. |
| **D5** | `EXTENSION-REGISTRY.md` | §11.1 | Two conformance vectors: **`REG-PEERISSUED-NAME-SUBSTITUTION-1`** — a validly-signed, current, unrevoked binding for name *X* served at `by-name/{Y}`; resolver MUST refuse and advance (their requested vector, and the one the existing four are blind to). **`REG-PEERISSUED-NULL-TTL-1`** — peer-issued binding with `ttl: null` → refuse. |
| **D6** | `guides/GUIDE-RESOLUTION.md` | §7 Trust model | The four-actor frame in user-facing terms, and the honest revocation word: against a hostile origin a revocation is bounded by the binding's TTL, not by the revocation's publication. This is what an operator needs to pick a TTL. |
| **D7** | `EXTENSION-REGISTRY.md` | §2.1 · §3b · §6a.3 | **The `ttl` unit, made consistent** (§10 F3). §2.1's `<ms-since-epoch duration>` → `<ms duration>`. §3b's `uint ; ms since Unix epoch, per §2.1` → `uint ; ms duration` — it declares a duration as an instant, in a normative field declaration, and mis-cites §2.1 as its authority. §6a.3 gains a one-clause pointer to §3's declaration so a reader working from §6a meets the unit. **§3 is already correct and is not edited** — it is the canonical home. |
| **D8** | `EXTENSION-REGISTRY.md` | §6a.4, fail-closed paragraph | A resolver **SHOULD** surface, to its own operator, which check failed (signature / expiry / revocation / unsupported kind / name mismatch), while the value returned to the chain stays the undifferentiated dead-end the rule requires. Explicitly scoped: this is local diagnostics, **MUST NOT** change what crosses a peer boundary (V7 §5.5a governs that, and governs a different direction). |
| **D9** | `EXTENSION-REGISTRY.md` | §6a.3 or a new §6a.3a | **Enumeration** (§10 F4). State that the authenticated form of *"what names does this registry carry"* is a walk of the published trie from `published-root.root_hash`, and that a registry intending to be browsable **SHOULD** publish at `prefix: "system/registry/binding/by-name/"` so the trie's key set is the name set. A served listing artifact (`{path}{tree_listing_suffix}`) is a **transport-trusted convenience and MUST NOT be presented as authoritative** — a hostile origin can omit an entry from a listing, and cannot omit one from a walk without the walk failing. |
| **D10** | `EXTENSION-REGISTRY.md` | §6a.3 | **A `kind: "peer-issued"` binding MUST carry a non-empty `transports`** (§10 F6). §4.1.2's pin carve-out is unchanged and the contrast is stated: a pin is the user's own assertion, so reaching the peer is the user's problem; an issued binding for a statically published peer has no §6.5 fall-through, because §6.5.4 makes profile discovery out-of-band in v1. |
| **D11** | `EXTENSION-REGISTRY.md` | §6a.9.2, beside the `domain-control` bullet | **`set-issuer-policy` MUST reject a policy that can mint an invalid binding.** A policy running live registration (any mode that can reach *approve*) with `default_ttl: null` MUST be refused **`400`**, *"rather than storing a policy it cannot enforce"* — the same sentence, the same reason, and the same subsection as the existing `domain-control` refusal. This is where the input actually lives: `default_ttl` is the **operator's** field, set through the **operator's** operation, and it is the only place the missing value can be supplied. |
| **D12** | `EXTENSION-REGISTRY.md` | §6a.9, admission pseudocode | **Fail closed when the stored policy is already bad.** D11 binds `set-issuer-policy`; it does not answer the policy seeded out-of-band, written directly to the tree, or predating the rule — §6a.9.2's store-first rule makes all three reachable. A registry whose resolved `ttl` would be null (request omitted `requested_ttl`, policy has no `default_ttl`) MUST refuse with **`403 policy_rejected`** — the existing pinned layer-2 code — and **MUST NOT mint the binding, MUST NOT substitute an implementation-chosen default.** An impl-chosen default is the failure §6a.9.2's `404`-synthesis bullet already rejects one level up: it turns an operator's unset field into a silently-invented policy. The operator-facing diagnosis belongs in the message string, which §6a.9 makes impl-local and un-asserted. |

**Not in scope:** any registry signed-root MUST (§4) · any change to §6a.7's manifest · any new
freshness mechanism · `EXTENSION-REGISTRY` §8.2 federation · any change to §6a.4's fail-closed
*return value* (D8 adds diagnostics beside it, never in it).

---

## §8 Cohort impact

| Seat | Owed |
|---|---|
| **`entity-core-rust`** | D1 (one comparison in `resolve_one`) · D3 (null-ttl refusal) · §5's by-target revocation lookup, replacing the prefix scan · correct the `RegistryTreeReader` doc-comment's soundness argument |
| **`entity-core-go`** | ~~D1 · D3~~ **both landed `d948b4f`, verified.** Revocation lookup already conformant. **Its `manifestLookup` path is unaffected by F1** — a signed manifest commits the association — but the pointer path is not, so the check belongs after both. **Now owed: D11 + D12** (§12) — the issuer half they routed, ruled against their lean on where the error goes. |
| **`entity-core-py`** | D1 (`_peer_issued_resolve`) · D3 · correct `registry_peerissued.py`'s *"at worst … stale/older"* docstring. **The docstring is the priority** — a comment stating a wrong bound is how the next implementer reproduces the hole, and this one propagated from a proposal nobody could read. |
| **`entity-browser-rust`** | Nothing blocking. **B16 is unblocked** (§4) — the emitter's artifacts do not change. Their `registry_static_resolve.rs` swap + withhold tests are the reference vectors for D5; we would rather adopt theirs than author new ones. |
| **oracle / keystone** | D5's two vectors. No re-pin — this adds cases, changes no encoding. |

**No wire change, no new entity type, no new error code, no renumber.** D1 and D3 are checks over data
already fetched; D5 is test surface.

---

## §9 The three open items — all three closed 2026-08-18

**§9.1 — D2's breadth: CONFIRMED at §3.** `entity-browser-rust` implemented the check at the consumer
and reports it cost one string comparison against a field already decoded, with the cost invariant
across backends **because it is a property of the body, not of the backend**. That is the better
argument and it replaces the original one: any kind carrying an `issuer_signature` over a body
containing `name` has the same discarded commitment, so scoping to §6a would make the next backend's
author re-derive the separability *that is the thing we filed*. D2 stands at §3.

**§9.2 — D3 vs precedes: RULED, and the mechanism already exists.** The filing seat declined to answer
(*"we ship no precedes and have never run the offline path"*) — the right call, and it leaves the
ruling to us. **A precede does not get a distinguished TTL, and the offline-first-run case is not what
precedes are for.**

§6a.4 says a precede is *"just a warm cache"*, and an expired precede is therefore **a cache miss, not
a failure** — the consumer fetches. That covers every case except *"first run, offline, and the
binding has expired,"* and a longer TTL does not solve that one either; it only moves the cliff. The
mechanism for **"trust this name without reaching the registry"** is already specified and is a
different thing: **`pinned_bindings` (§4, §4.1.2)** — sticky, `ttl: null`, override everything, and
explicitly *"exempt from GC until unpinned"* (§9). A distribution that must resolve offline
indefinitely ships **pins**, not long-lived precedes.

That keeps the two paths honestly separated: a precede is the registry's assertion, cached, and
expires like one; a pin is the distribution's own assertion and is meant to be sticky. **Giving a
precede a second expiry rule would be a second trust path wearing a cache's name.**

**§9.3 — `ResolutionResult` carrying `name`: NOT ADDED.** The filing seat does not use `resolve_one`'s
result at all (they walk the registry's signed root and decode `BindingData` directly, so they hold
`name`), and they make the argument against better than the proposal did: **adding a field to make a
redundant check possible invites the redundant check to be treated as the primary one.** Once D1
lands, defence-in-depth is not a reason. The display argument is real; if a future SDK proposal adds
it, it is **documented display-only** and MUST NOT be presented as an association check.

---

## §10 Four findings from building it (`ROUTING-2026-08-18-b`) — dispositions

### F3 — the unit: their diagnosis is off by one section, and the real defect is worse

**The unit IS specified**, at §3, which is the binding entity's canonical type declaration:
`issued_at: <ms-since-epoch>` and `ttl: <ms duration | null>`. §6a.3 describes the peer-issued
specialization and does not restate it, which is why a reader working from §6a never meets it. That
part is navigation, and D7 fixes it with a cross-reference.

**Checking their finding turned up the real one.** `ttl` is declared three times in this spec, in
three different phrasings, and **two are incoherent**:

| where | text | verdict |
|---|---|---|
| §3 (binding) | `ttl: <ms duration \| null>` | **correct** |
| §2.1 (`ResolutionResult`) | `ttl: <ms-since-epoch duration \| null>` | **incoherent** — a duration is not since-epoch |
| §3b (service-advertisement) | `ttl: uint ; ms since Unix epoch, per §2.1` | **wrong** — declares a duration as an instant, in a normative field declaration, and cites §2.1 as its authority |

A reader who lands on §3b is told a `ttl` is an absolute timestamp. That is a live unit defect in the
spec, not a doc gap, and it is exactly the error `entity-browser-rust` paid a red gate to find in
their own emitter. **Their bug and this text are the same mistake at two ends of the wire.**

**On the `None` collapse — their second-order point is the sharper one and it is granted.** §6a.4's
fail-closed rule returns `None` for a failed signature, a revocation, an expiry and an unsupported
kind alike, so the symptom of a unit error is *"the registry is broken."* There is **no confidentiality
argument** for that collapse here: V7 §5.5a's `Denied`-vs-`NotFound` discipline governs what a peer
tells a *remote* requester, and this is a local resolver reporting to its own operator. D8 adds a
SHOULD for a locally-surfaced reason, and is explicit that it MUST NOT change what crosses a peer
boundary.

### F4 — the premise is wrong, in their favour: a signed root **is** enumerable

**Ruling: an authenticated registry browse exists today, needs no new mechanism, and the shape is a
deployment choice they have already made everywhere else.**

`EXTENSION-TREE` §3.1's conformance fixture #2 shows a trie leaf as `82 60 58 21 <H>` — `[key,
value_hash]`, **the key is in the node**. §3.5's diff pseudocode names the primitive:
`walk_entry_collect_bindings(entry)` → *"list of (key, value_hash)"*. So walking every node reachable
from `published-root.root_hash` yields **the complete key set the signature commits to.**

What §3.4.2 actually says (*"to reason about bindings under prefix X, consumers use LocationIndex …
rather than walking trie subtree structure"*) is about **efficient prefix scan** under hash-keyed
routing, which scatters related keys. It is not a statement that the key set is unrecoverable — and
for a registry the distinction dissolves, because the registry can **publish at
`prefix: "system/registry/binding/by-name/"`**. Then the trie's key set *is* the name set, and the
walk is the browse. That is the same per-purpose tracked-prefix ruling
`PROPOSAL-STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` §3 gave for `sites/`, applied one tier up.

**And it delivers exactly the property they said had no form.** With a served `.list`, hiding a name
is undetectable. Under a trie walk, hiding a name requires **withholding a node**, and the walk then
fails visibly instead of returning a short list. That converts *silently hidden* into *visibly
incomplete* — which is the completeness guarantee, in the strongest form a static origin permits.

**Their `by-name.list` is not wrong; it is a menu.** Keep it — it is one fetch instead of O(N) and it
is the right first-paint artifact. It is **not the authority**, and D9 says so normatively so that a
registry-browser UI does not present a transport-trusted list as a registry's contents.

**This narrows F2's tail too.** *"List this registry's revocations is not expressible against a signed
root"* has the same answer: it is expressible, by the same walk, if the revocation prefix is tracked.
That does not make revocation *fresh* — an older signed root is still servable and the TTL is still
the cold-start bound (§3) — but it removes "cannot enumerate" from the list of reasons.

### F5 — confirmed, adopted, and it lands as a spec pin

Their self-correction is correct on the text: §6.5.3.1 makes `MANIFEST_GET` *"singular/terminal: no
suffix, no trailing slash"*, while `{path}{tree_leaf_suffix}` serves a bare 2-key `system/hash`
pointer. A 3-key wire entity at `…/published-root.bin` is the two routes crossed.

**Their read that this is a cohort-wide seam is right, and it is now two of three app-tier seats:**
`entity-workbench-go` writes an `http-poll` transport-profile entity at `{outDir}/manifest`
(`ROUTING-2026-08-17-l` §2). Different error, same question — *what exactly is served at
`manifest_url_prefix`* — and each seat can pick differently while every local test passes.
`PROPOSAL-STATIC-PUBLISH-HANDSHAKE-AND-MANIFEST-FRESHNESS` **D5** already pins the slot; this is a
second, independent instance arriving before that proposal folded, which is the evidence that D5 is
the right shape rather than a one-seat correction.

**Their process lesson is adopted into the catalog** (AP-19): *when a checker disagrees with an emitter
you wrote in the same session, the emitter is the more likely defect.* They fixed the gate first, for
three commits — and reported it against themselves, which is the only reason it is available to
anyone else.

### F6 — confirmed: a peer-issued binding MUST carry transports

§4.1.2's carve-out — *"Empty `transports` on a pin is acceptable; pins assert binding authority.
Transport resolution per NETWORK §6.5 finds reachable endpoints"* — is sound **for a pin, and for a
live target.** For a statically published peer there is nothing for §6.5 to find: §6.5.4 states
discovery of a publisher's transport profiles is **out-of-band for v1**. So the fall-through the
sentence promises does not exist on the path a registry is most useful for, and a consumer resolves a
peer-id and stops.

**Ruling: a `kind: "peer-issued"` binding MUST carry a non-empty `transports`** (D10). The §4.1.2 pin
carve-out is unchanged — a pin is the *user's own* assertion and reaching the peer is then the user's
problem, which is the difference. This is not "discover the origin out of band is wrong"; it is that a
registry whose bindings omit transports **adds nothing operationally** (their words, and the
`entity-deployment.json` they already ship is the out-of-band form doing the work instead).

---

## §11 The offer — accepted

`entity-browser-rust` offers their publisher as a worked Amendment-10-conformant reference: trie
closure projected at publish (root + interior + leaf-bound content + `published-root` + its
signature), `--verify` **failing the tree** on an incomplete closure, and a §6.5.3 profile matched
field-for-field without adaptation.

**Accepted, and it is the highest-value artifact on this track**, because the failure mode it gates is
the one prose review cannot reach: *every pointer resolves and a pinned consumer still gets nothing.*
That is the exact shape `EXTENSION-NETWORK` §6.5.6's Amendment-10 closure MUST exists to prevent, and
until now the cohort had the rule and no worked example of satisfying it. Routed to the cohort as a
reference shape, **not** as a normative requirement — the contract stays "MUST resolve / MUST upload"
(`…-MANIFEST-FRESHNESS` D3), mechanism impl choice.

---

## §12 The issuer side of D3 — ruled (`entity-core-go` spec-issue 2026-08-18-a)

**The question, restated.** D3 as routed is a *resolver* rule. The proposal's binding-validity claim
is stronger — *"`kind: "peer-issued"` MUST carry non-null `ttl`"* — which is a property at **mint**
time. `entity-core-go`'s §6a.9 issuer resolves `ttl := body.RequestedTTL`, falls back to
`policy.DefaultTTL`, and mints with `ttl` still nil when both are absent. **The issuer produces what
its own resolver now refuses.** They declined to invent the disposition and routed it. That was the
right call, and the reason is §6a.9's own history: this section has twice manufactured a conformance
failure by pinning a `MUST` whose disposition the corpus had not defined (`pending_hash`, the
statuses table's missing carrier column).

**Concur on the premise, and it needs no new rule.** An issuer minting a binding no conformant
resolver will honor is not a second defect — it is D3 observed from the other end. A binding's
validity does not depend on which side of the wire you are standing on.

**Where we depart from their lean, and it is the load-bearing half.** Their preference is *refuse the
register-request with a 400 when the resolved ttl is null.* Directionally right; **wrongly addressed.**
The two ways to reach a null resolved `ttl` are not the same event:

| how | whose input is missing | who can supply it |
|---|---|---|
| request omits `requested_ttl`, policy **has** `default_ttl` | nothing is missing | *(no error — the default applies, which is what `default_ttl` is for)* |
| request omits `requested_ttl`, policy has **no** `default_ttl` | the **policy's** field | the **operator**, via `set-issuer-policy` |

**In the only failing row, the requester has done nothing wrong and can do nothing about it.** Their
request was well-formed; the registry is misconfigured. A `400` there is a client-error status for a
server-side omission, and it teaches requesters to start sending `requested_ttl` defensively — which
hands TTL selection to the party the threat model treats as untrusted (§5, and D3's whole reason:
`issued_at + ttl` is the one check a hostile byte-server cannot influence).

**So gate it where the input lives.** D11 puts the refusal on `set-issuer-policy`, which is
**already the shape this subsection uses**: §6a.9.2 refuses to store a `domain-control` policy
*"rather than storing a policy it cannot enforce."* A live-registration policy with no `default_ttl`
is the same class — a stored configuration that cannot produce a conformant result — and it is caught
at the moment the operator can still fix it, with the operator's own capability
(`registry-manage-issuer-policy`) in hand.

**D12 closes the door D11 cannot reach**, and it exists because §6a.9.2 already learned this lesson
once: the `400` at `set-issuer-policy` is not the only thing standing between a bad stored policy and
a bad outcome, because store-first resolution admits policies seeded by CLI flag, written directly to
the tree, or predating the rule. That is verbatim why `REG-ISSUER-DOMAINCTRL-STORED-1` exists beside
`REG-REGISTER-DOMAINCTRL-1`. We take the same pair.

**Rejected: synthesize a default at the issuer.** It is the third option their spec-issue raises, and
it fails on §6a.9.2's own reasoning one bullet up — `get-issuer-policy` **MUST NOT synthesize a
default `open`** because that silently converts an operator's curated registry into a
first-come-first-serve one. An implementation-chosen TTL is the same move on a security-relevant
field: two registries would answer the same request with different binding lifetimes from the same
stored policy, which is a §5.10 cross-peer determinism split, and the operator would never see it.
**A protocol-wide TTL floor is rejected for the same reason plus one more** — there is no defensible
number, and picking one makes every registry that forgot to configure a TTL look configured.

**Their fixture note is answered:** the ruling is *issuer refuses*, so
`TestRegister_AllowlistMode_DenyThenAllow` setting `RequestedTTL` stays valid as written — it tests
the allowlist gate, and the added field is no longer incidental but correct. No `DefaultTTL` needs to
be added to its policy; a second test should assert D11's refusal at `set-issuer-policy` instead.

**Vector:** **`REG-ISSUER-NULLTTL-POLICY-1`** — `set-issuer-policy` with a live mode and
`default_ttl: null` MUST be refused `400`; then write that policy entity directly and attempt live
registration with a request omitting `requested_ttl` — MUST refuse `403 policy_rejected` and MUST
publish nothing. Same two-stage shape as `REG-ISSUER-DOMAINCTRL-STORED-1`, for the same reason.
