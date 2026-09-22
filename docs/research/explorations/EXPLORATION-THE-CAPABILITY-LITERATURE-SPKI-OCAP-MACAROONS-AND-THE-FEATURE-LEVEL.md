# EXPLORATION — the capability literature against our capability system, and Radicle's fix ladder

**Internal working document.** Not a proposal, not normative. **Nothing is folded by this document.**

**Why this axis:** we shipped a capability system — `attenuat` 27 files, `caveat` 36, `revocation` 52 —
and the founding literature measured **zero**: `SPKI` 0 · `SDSI` 0 · `object capability` 0 ·
`capability safety` 0 · `Horton` 0, with `macaroon` 2 / `UCAN` 1 / `ZCAP` 1 / `CapTP` 1 all passing
mentions in a single sentence. **That is the largest build-without-reading gap in the corpus** and it is
on the surface that decides who may do what.

**Sources opened:** RFC 2693 (SPKI certificate theory) · *Capability Myths Demolished* (Miller, Yee,
Shapiro — via abstract + secondary; **the PDF did not extract and the HTML host refused, so the ocap
section below is flagged**) · the Macaroons construction (Birgisson et al. + Fly.io's implementation
notes) · **`heartwood` cloned from Radicle's own seed node** at `919d5c7` (2026-07-28).

---

## §1 The three findings that change something

**F1 — `max_delegation_depth` is a field SPKI examined and deliberately rejected, and its argument
lands on a case we ship.**
**F2 — third-party caveats are absent (0 hits) and we can have them almost free, because the
macaroon construction's complexity is an artifact of symmetric keys we do not use.**
**F3 — Radicle rolled out both CVE fixes as a monotone FEATURE LEVEL, which is the answer to the
flag-day problem we have hit twice and have no mechanism for.**

Everything else on this axis came back **confirming**, and §5 says so plainly rather than dressing it
up as work.

---

## §2 F1 — delegation depth

`ENTITY-CORE-PROTOCOL` §2956 ships two delegation caveats:

| Field | Meaning |
|---|---|
| `no_delegation` | if true, cannot delegate further |
| `max_delegation_depth` | maximum chain depth from this capability |

**SPKI has exactly the first and deliberately not the second.** RFC 2693 §4.1.4 settles delegation as a
**boolean**, and §§4.1.3–4.1.4 give the reason for rejecting depth limits: *predicting the proper
delegation depth is impossible, and temporary keys — the laptop case — always require one more level.*

**The argument bites us specifically, because our own identity model is the case SPKI names.**
`EXTENSION-IDENTITY` §11.3 puts **per-device agent keys with their own peer-IDs** under a controller.
So a grantee's *architecture* consumes delegation levels the granter cannot see: a grant issued
`max_delegation_depth: 1` to Alice works if Alice is one key and fails if Alice is a controller with a
laptop agent and a phone agent. **The granter has no way to know which, and the failure is a 403 at the
device, far from the grant.**

**What this is not.** It is not an argument that the field is wrong to exist — it is optional, it costs
nothing unset, and a depth bound is genuinely useful in a closed deployment where the topology is known.
**It is an argument that the field is a footgun in the open case, and that we have shipped it with no
guidance saying so.** SPKI's position is that `no_delegation` expresses the only intent a granter can
actually hold across a trust boundary: *this stops here*, versus *this may travel*.

**The check that would settle it:** does any handler or convention set `max_delegation_depth` to a
literal today? If yes, against a grantee that could be a multi-device identity, the bug is live rather
than latent. **Not measured here — it is a grep across five trees and it is the next thing to run on
this axis.**

---

## §3 F2 — third-party caveats, and why they are nearly free for us

**Measured: `third-party caveat` 0 · `discharge` 5, all five ordinary English** (*"the risk is
discharged by when the check runs"*), none of them the credential sense.

**What a third-party caveat is.** A first-party caveat is a condition the verifier evaluates itself
(*expires at T*, *path under /x*) — that is our `constraints` map, handler-interpreted. **A third-party
caveat is a condition the verifier does not evaluate at all**: it says *this is valid only if you also
present a discharge from party P attesting condition C*. The verifier checks the discharge, never
contacts P, and **P and the verifier never communicate.**

**What it buys that we cannot express today:**

- **Audience / membership.** *"Valid only if you hold a token from Alice saying you are in her
  followers."* This is `APP-CONVENTION-SHARE`'s audience problem, and today the granter must enumerate
  members at grant time.
- **Delegated revocation.** *"Valid only if the revocation service issued a freshness statement in the
  last hour."* SPKI §5.4's framing — *"how long are you willing to let the world believe something
  false?"* — becomes a per-grant parameter instead of a system-wide one.
- **Attribute grants.** *"Valid only if some attestor says the holder is over 18 / is a maintainer /
  passed review."* Grant to a **property**, not to a key.

**The construction is complicated in macaroons for a reason that does not apply to us.** Macaroons are
built on a shared symmetric root key and HMAC chaining, so a third-party caveat has to smuggle a
per-caveat root key to two parties at once — hence the VID/CID pair, one copy of the key encrypted
under the current HMAC tag for the verifier and one encrypted under a pre-shared key for the third
party. **All of that machinery exists to move a symmetric secret.**

**We have public keys, content addressing, and a signed third-party claim object already.**
`system/attestation` is a discharge, field for field:

| Macaroon discharge | `system/attestation` |
|---|---|
| who discharged it | `attesting` |
| what it is about | `attested` |
| the condition met | `properties` |
| validity window | `not_before` / `expires_at` |
| supersession | `supersedes` |
| binding to the request | signed, content-addressed, carried in `envelope.included` |

**So the shape is:** a `constraints` key naming a required attestation — issuer, subject, and the
property to check — discharged by the grantee supplying the matching `system/attestation` entity in the
envelope. **No new type, no new crypto, no key exchange, and it is offline-verifiable** because the
attestation carries its own signature. The one real design question is **binding**: a macaroon binds the
discharge to the specific token cryptographically, and we would bind by requiring
`attestation.attested == the grantee` — which is weaker (the same attestation discharges any grant
naming that condition) and is probably the *right* weakness, since an attestation of membership
genuinely should be reusable.

**This is the most valuable unbuilt thing found on this axis.** It is not urgent and it is not owed —
it is a primitive the system is one `constraints` convention away from having.

---

## §4 F3 — Radicle's feature level, read at source, and the flag-day answer

**Sourcing debt discharged.** `radicle.dev` 403s the fetcher; the previous document's analysis rested on
search summaries. **`heartwood` clones cleanly from Radicle's own seed node** —
`z3gqcJUoA1n9HaHKufZs5FCSGazv5`, read at `919d5c7`. The source is better than the docs.

**Confirmed, and sharper than the summary version.** `Refs` is
`BTreeMap<RefString, Oid>` whose `canonical()` emits `"<oid> <name>\n"` per entry — **a signed map from
mutable coordinate to immutable target**, which is register **S-15**'s claim, now verified in the tree.

**The anti-replay fix is more elegant than reported.** `SIGREFS_PARENT` is **not a new field** — it is a
**reserved key inserted into the signed map itself** (`add_parent` does
`self.0.insert(SIGREFS_PARENT, commit)`). The predecessor rides inside the payload that was already
being signed. **No schema change, no new signature, no second object.**

**And here is the part we do not have.** `heartwood` carries a `FeatureLevel` enum —
**`None` → `Root` → `Parent`**, with `LATEST = Parent` — documented in its own schema as:

> *"The highest feature level known, which protects against **graft attacks and replay attacks**.
> Introduced in Radicle 1.7.0, in commit `d3bc868e84c334f113806df1737f52cc57c5453d`."*

**The two CVEs are the two rungs.** `Root` is the graft fix (bind the assertion to its repository),
`Parent` is the replay fix (bind it to its predecessor). A verifier **computes the level of the refs it
received** and can require a minimum.

**Why this matters to us more than the CVEs did.** We have hit the flag-day problem twice — a
previously-unenforced field going load-bearing and partitioning the cohort during adoption — and our
recorded answer is a *process* one: *name the divergence unit in the fold text so the seats can sequence
themselves.* **Radicle's answer is a mechanism.** The artifact declares which protections it carries;
adoption is monotone; a verifier tightens by raising a floor rather than by flipping a check; and an
old peer's output is **recognisably old rather than invalid**.

**We have no analogue.** Our extension version numbers describe the *document*, not the *artifact*, so a
verifier holding a `published-root` or a `system/registry/binding` cannot ask *which protections did the
producer apply?* — it can only check fields and infer. **That is a real gap and it is the one thing on
this axis I would actually design.** It also composes with the witness work: a witness cosigning a root
is exactly the party who should be able to say *"and it was at level N."*

---

## §5 What came back confirming — stated once, briefly

**These are not findings and are recorded so nobody re-derives them.**

- **SDSI is our namespace model.** RFC 2693 §2.8: *"the public key of the issuer is the identifier of the
  name space in which that name is defined."* That is our peer-id-as-first-path-segment, arrived at
  independently. It also means **SPKI's local-naming literature applies directly to `EXTENSION-REGISTRY`**,
  which is the third arrival at TUF already recorded in S-15.
- **Authorization over authentication** (RFC 2693 §3) — our capability carries the authority and the
  request is checked against it. Correct by construction.
- **Not bearer tokens.** §5.2 step 3 requires `capability.data.grantee == execute.data.author`, so a
  stolen capability is unusable. **We are stronger than macaroons here**, which are bearer credentials by
  design.
- **The confused deputy is identified and mitigated, in nine extensions.** §6.8's *caller-specified
  paths* rule — *the handler MUST verify the caller's capability covers the write path … MUST NOT
  substitute its own grant* — is the textbook fix, and §6.8's *propagated caller capability is not a
  dispatch gate* is its dual. `check_path_permission` is applied in TREE, QUERY, SUBSCRIPTION, CONTENT,
  COMPUTE, REVISION, TRANSACTION, HISTORY and CONTINUATION.
- **It is oracle-driven.** I expected to find this MUST undriven and it is not: `spec census` puts
  neither §6.3 nor §6.8 in `unobserved-must` (the core-protocol entries there are §5.9 and §5.10).
- **Validity windows exist** (`not_before` / `expires_at`) and match SPKI §5's timed-CRL reasoning.

**The honest overall verdict on this axis: the capability system is in good shape**, it is
ACL/certificate-shaped in the SPKI lineage rather than object-capability-shaped, that choice is forced
(unforgeable references do not survive a network boundary or third-party audit), and the tax it incurs —
the confused deputy — is the one hazard the corpus names most explicitly.

---

## §6 The one claim in this document I do not stand behind yet

**The object-capability comparison is under-sourced.** `srl.cs.jhu.edu` does not resolve, the Agoric PDF
returned unextractable binary, and `zesty.ca` refused the connection. **So *Capability Myths Demolished*
was read through its abstract and secondary summaries, not opened**, and the seven security properties
are not in hand.

**What that costs:** §5's sentence *"ACL-shaped, and the tax is the confused deputy"* is the right shape
but I cannot cite the property that makes it precise. **The specific question left open is whether the
`designation`/`authority` split is load-bearing for us**, i.e. whether there is any place where a handler
receives a caller-supplied *path* where it could instead receive a caller-supplied *capability*. If there
is, that is an ocap-style structural fix available inside our model, and it would be better than a MUST
each handler must remember.

**Do not cite §5's ocap framing in a proposal until the paper is opened.** Working mirrors to try:
ResearchGate, Semantic Scholar, or `usenix`/`combex` mirrors.

---

## §7 Owed

1. **Grep five trees for a literal `max_delegation_depth`** against multi-device grantees (§2). Cheap,
   decides whether F1 is live or latent.
2. **Open *Capability Myths Demolished*** and settle §6.
3. **The feature-level design** (§4) — the only thing here I would write a proposal for, and it composes
   with the witness work from the CT session.
4. **Third-party caveats** (§3) — a `constraints` convention over `system/attestation`, not urgent.
5. **Still unread on adjacent axes:** Sigsum / Go checksum DB · Hypercore · in-toto and SLSA (`in-toto`
   2, `SLSA` 3, `Sigstore` 5 — thin, and the supply-chain axis is otherwise well covered via TUF).
