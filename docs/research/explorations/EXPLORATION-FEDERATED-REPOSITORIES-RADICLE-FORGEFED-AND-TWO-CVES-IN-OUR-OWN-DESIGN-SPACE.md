# EXPLORATION — federated repositories: Radicle, ForgeFed, and two production CVEs in the shape we just proposed

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-directed, 2026-09-06:** *"The other one we may want to look at is federated repositories,
because there's a lot there as well. It's one area that I think we've kind of missed and it may
provide some more useful insight there."*

**The instinct was right, and the gap was specific.** Measured before starting: the corpus already
covers the **supply-chain** axis well — **TUF 27 mentions, OCI 143, Sigstore 7**, with TUF's role split
already absorbed into `EXTENSION-REGISTRY` §6a.7 (*offline content keys vs online freshness keys;
resolver enforces highest-seen `seq`*). But **Radicle: 0. ForgeFed: 0. Debian/apt: 0. Forge federation:
0.** The artifact-distribution axis was studied; the **collaborative-repository** axis never was.

---

## §0 The result

**Four findings, and the second is worth the whole study.**

1. **Radicle's `refs/rad/sigrefs` is the operator's binding assertion, shipping in production since
   2022** — a per-peer signed blob mapping ref names to object ids. Independent convergence on *a signed
   set of (mutable coordinate → immutable target) bindings*, which is `S-4d`'s general pattern.
2. **That design has had TWO disclosed vulnerabilities, both of which apply directly to our proposal,
   and one of them answers a cost objection we raised yesterday.** §2.
3. **Radicle's RFC 0662 names, as an unsolved problem, the exact thing `FEED` §4's mirror exists to
   solve.** §4. We are ahead here, and did not know it.
4. **Radicle explicitly declines to specify L5 authorization** — *"this RFC does not specify
   authorization logic and applications must instead implement this themselves"* — which is the same
   gap this session identified as `L-7`. **Two independent systems stop at the same layer.** §5.

---

## §1 What Radicle actually is, mapped onto our stack

**Per-peer namespaced git, gossiped, with no server and no host authority.** Every peer's view of a
repository holds every other peer's refs under that peer's own namespace — **structurally identical to
`ENTITY-CORE-PROTOCOL` §1.4's local view**, where a peer's tree holds `/{its_own_id}/…` authoritative
and `/{other_id}/…` cached.

| Radicle | Ours | Note |
|---|---|---|
| Repository ID (RID) | a peer namespace | both self-certifying, both derived from a key |
| `refs/namespaces/<peer>/…` | `/{peer_id}/…` | **the same local-view model** |
| **`refs/rad/sigrefs`** — per-peer signed blob of `(ref → oid)` | **no equivalent** | **this is the gap D-42 names** |
| `refs/rad/root` | *(implicit in absolute paths)* | see §2 |
| identity document + **delegates with a threshold** | `EXTENSION-QUORUM` + `EXTENSION-IDENTITY` | K-of-N, same shape |
| **COBs** — issues/patches as CRDTs over the git DAG | `EXTENSION-REVISION` version DAG + `app/feed` entries | §4 |
| git commit DAG | content-addressed entity graph | both merkle, both dedup |
| *"verification model inspired by **The Update Framework (TUF)**"* | `REGISTRY` §6a.7, TUF-tiered | **third independent arrival at TUF** |

**The convergence claim, stated carefully:** we and Radicle independently built per-peer namespacing
over a merkle store with self-certifying identity, and **both then reached for TUF for the freshness
layer.** That is not evidence either design is correct — it is evidence the *problem* has a stable
shape, which is the useful part.

---

## §2 The two CVEs, and why they matter to us specifically

**This is the payoff of studying a shipped system rather than a paper.** Radicle's signed-refs
mechanism — the one we are proposing to build — has been attacked twice in production.

### §2.1 The graft attack — a signature that was not scoped to its context

**Fixed in Radicle 1.1.0 (2024-12-05) by adding `refs/rad/root` to the signed refs blob.** Without it, a
signed `sigrefs` blob **was not bound to any particular repository**, so a validly-signed set of
bindings from one repository could be transplanted — *grafted* — into another. Including the identity's
**root commit OID** in the signed data scopes each signature to a single repository. Radicle later made
its absence a hard error (1.7.0), which broke enough users to need 1.7.1 within 24 hours.

> **What it means for us: a signed `{coordinate, target}` assertion must be scoped to the context it
> asserts within, or it is portable to contexts where it is false.**

**We appear to get this for free, and it is worth stating rather than assuming.** A tree path is
**absolute and begins with the peer-id** (`ENTITY-CORE-PROTOCOL` §1.4), so an assertion over
`/{alice}/blog/post` **cannot be grafted into Bob's namespace** — the coordinate names Alice, and the
verifier requires `signer == the path's first segment`. **The self-describing property that made the
binding assertion attractive (§4 of the ladder study) is the same property that makes it graft-proof.**
That is a real argument for the absolute-path design, arrived at from someone else's incident.

**Owed check, not done here:** does `system/peer/published-root` bind its own context? It carries
`prefix` and is signed with a `signer`, and is fetched from a peer-qualified path — so the graft is
closed by the `signer` check. **But that is a derivation, not a measurement, and the honest status is
unverified.**

### §2.2 The replay attack — and it answers our open cost question

**Disclosed 2026-03-30; the fix records `refs/rad/sigrefs-parent` in the refs blob, and if present its
target MUST match the parent commit.** Without it, **a previous signed `sigrefs` commit could be
replayed** — an attacker re-presents an older, validly-signed binding set, and every signature checks
out. Signature validity is not recency.

> **This is precisely the objection raised against the operator's binding assertion one session ago:**
> *"without a counter it is replayable, and a counter per coordinate is per-path state the tree does not
> keep."* **Radicle hit the vulnerability, and their fix is not a counter — it is a PARENT POINTER.**

**A hash-chain gives replay resistance without any counter at all.** Each assertion names its
predecessor; a verifier holding assertion *N* rejects anything whose parent is not the one it knows.
Cost: one hash field. No global sequence, no per-path integer, no coordination.

**And we already have the field, in the very entity that is the general pattern.**
`system/registry/binding` carries **`supersedes: <system/hash | null>`** *"per ATTESTATION supersedes
chain"*, and `system/peer/published-root` carries **`predecessor`** — *"MUST carry the prior
`published-root` content hash once one exists"* (`NETWORK` §6.5.6).

> **So the cost objection is withdrawn.** A binding assertion needs `supersedes`, not `seq`, and the
> corpus has had the mechanism in two places the whole time. **`EXTENSION-ATTESTATION`'s supersedes
> chain is the general form**, and Radicle converged on it after being attacked.

### §2.3 The transferable lesson, which is about the shape of both bugs

**Both CVEs are the same failure and neither is a crypto failure.** The signature verified correctly in
both cases. What was missing was **binding the assertion to its context** — *which repository* (graft)
and *which point in its own history* (replay).

> ***A signature proves who. It does not prove where or when. Those must be fields.***

**That is a checkable authoring rule** and it generalizes past this study: any signed assertion about a
mutable coordinate needs three things — the **coordinate** (absolute, so it cannot be grafted), the
**target**, and a **predecessor** (so it cannot be replayed). `system/registry/binding` has all three.
**It is the fully-worked instance of the pattern, and it got there before we noticed it was a pattern.**

---

## §3 ForgeFed — the contrast, and it is instructive by being unlike us

**ForgeFed is ActivityPub extended with forge vocabulary** — repositories, commits, patches, tickets —
for **server-to-server** federation between forges (Forgejo is the main adopter; Vervis is the
reference implementation). Its design goal is that *"users of any ForgeFed-compliant service can
interact with other compliant forges without being a registered user of that foreign service."*

**Its unsolved problem is the one our substrate does not have.** The standing critique is that **anyone
can run an ActivityPub server and claim to be a given authority**; WebFinger and domain verification
help and do not close it. **That is host-as-authority**, and it is the failure mode
`GUIDE-RESOLUTION` §2's *names are receiver-relative* and V7 §1.5's self-certifying peer-ids dissolve by
construction: **our authority is a key, so there is nothing to impersonate.**

**Where ForgeFed is ahead of us: vocabulary breadth.** They have worked through tickets, patches,
reviews, and pushes as first-class federated objects across heterogeneous implementations. **We have
`app/feed/entry` and a proposal.** If the social/forum tier grows, their vocabulary is the map of what
a forum-shaped application actually needs, and it is CC0.

---

## §4 COBs — Radicle names our mirror's purpose as their open problem

**Collaborative objects are Radicle's issues and patches**, stored as git objects, identified by *"the
hash of the initial commit of the object as the ID"* (content-addressed, `ObjectId`), grouped by
`TypeName` (e.g. `xyz.radicle.issue`), replicated at
`refs/namespaces/<namespace>/cob/<typename>/<object ID>` so a peer can **filter by type instead of
fetching everything.**

**The CRDT is the DAG itself** — *"when two peers' histories synchronize, the commit graphs are unioned
in a non-destructive, idempotent way… the graph is reduced in topological order from root to tips to
materialize the state."* **That is `EXTENSION-REVISION` §5.4's model exactly** — *CRDT as a
computational artifact during merge, not a storage format*, with the causal record being the version
DAG's parent pointers. **Two systems, same answer, arrived at separately.**

**And then RFC 0662 states this, as an open problem:**

> *"I might 'create an issue' in a repository and anyone who is tracking me would see that issue, but
> **people who are tracking the project but don't have me in their tracking graph will only see the
> issue if the maintainer replies to it.**"*

**That is the visibility hole of per-peer namespacing, and it is our thread problem stated in their
vocabulary:** a reply lives in the replier's namespace, so a reader who does not follow the replier
never sees it. **`FEED` §4 is an answer to it** — *"a mirror is one reader's answer to 'here is what I
gathered', published so the next reader does not have to gather it again"*, with merging defined as
**set union over signed content-addressed entries**, so two readers who gathered disjoint halves produce
the whole with no coordination.

> **We are ahead on this specific question and did not know it.** Their design has no mirror object;
> discovery is bounded by each reader's tracking graph. **Ours makes gathering a publishable artifact,
> which converts a topology problem into a content problem.** *(Stated as a comparison of designs, not
> a claim about deployed behaviour — `FEED` is an unlanded proposal and Radicle ships.)*

---

## §5 Both systems stop at the same layer, and that is the strongest signal in this study

**RFC 0662, on authorization:** *"This RFC does not specify authorization logic and applications must
instead implement this themselves."* Their change commits carry `X-Rad-Signature`,
`X-Rad-Author` and `X-Rad-Authorizing-Identity` trailers — **the mechanisms** — and the *policy* for
what a reader should accept is left to each application.

**That is `L-7` from this session's ladder study, independently confirmed:** the substrate provides
integrity, authorship and identity, and **the L5 question — *"given what I gathered, what may I believe
and what must I render as unattributed?"* — is unowned in both systems.**

**Which means the verification ladder is not us catching up. It is the layer the field has not
written down**, and the piece we can state that Radicle cannot is the **composer contract** (`L-4`):
*a reader can only climb a rung the publisher paid for.* Radicle's `sigrefs` pays for rungs 1 and 2
automatically at every push; **we currently pay for neither, because nothing signs individual
entities.**

---

## §6 What this changes

| # | Change | Where |
|---|---|---|
| 1 | **Withdraw the `seq`-per-coordinate cost objection.** A binding assertion needs **`supersedes`**, not a counter — hash-chain, one field, and `system/registry/binding` already has it | ladder study §4; register `S-4b` |
| 2 | **Add the scoping rule**: a signed assertion must bind its **context** and its **predecessor**, or it is graftable and replayable. *A signature proves who, not where or when* | new register row |
| 3 | **`system/registry/binding` is the reference instance of the general pattern**, with all three fields, and should be cited as such rather than re-derived | register `S-4d` |
| 4 | **`FEED` §4's mirror answers a problem Radicle documents as open** — strengthens the case for landing it | `D-44` |
| 5 | **Verify `published-root` is graft-scoped** — derived, not measured | new open item |
| 6 | **ForgeFed's vocabulary is the map for a forum tier**, CC0, if that tier grows | future |

---

## §7 Sources

**Read at source this session:** [`radicle-cob` API documentation](https://docs.rs/radicle-cob) —
`ObjectId`, `TypeName`, `History::traverse`, `Entry::author`, *"graphs of CRDTs"* ·
[RFC 0662 — Collaborative Objects](https://github.com/radicle-dev/radicle-link/blob/master/docs/rfc/0662-collaborative-objects.adoc)
(raw text) — the CRDT rationale, the `X-Rad-*` trailers, the authorization punt, and the tracking-graph
open problem.

**In-corpus, opened:** `ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` §§(TUF rows) ·
`EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` · `EXTENSION-REGISTRY` §3 · `EXTENSION-NETWORK` §6.5.6 ·
`PROPOSAL-APP-CONVENTION-FEED` §4/§4.1/§4.2 · `ENTITY-CORE-PROTOCOL` §1.4/§1.5/§3.5.

**NOT read at source — labelled, per L18, and not to be cited normatively until opened.** The
`rad/sigrefs` / `rad/root` / `rad/sigrefs-parent` details, the 1.1.0 graft fix and the 2026-03-30 replay
disclosure come from **search results summarizing** the Radicle release notes and disclosure post;
**both `radicle.dev` pages returned HTTP 403 and `docs.radicle.xyz` refused the connection.** The
ForgeFed section is likewise from a summary of `forgefed.org` and the project repository, not the
`spec/` texts. **The §2 findings are strong enough to act on and thin enough to re-verify before any of
this is folded** — the first task for whoever takes this forward is to open the Radicle disclosure and
the heartwood protocol spec directly.

## §8 What is unread

- **The heartwood protocol specification itself** (`radicle-dev/heartwood`) — the authoritative text for
  sigrefs, canonical refs and the delegate threshold.
- **Debian/apt's `Release` → `Packages` → `.deb` signed-index chain**, and **OCI/Docker manifest
  lists** — the two most-deployed *mirror-serves-signed-index* systems in existence, and neither was
  examined here. **The apt model is probably the closest prior art to a signed page map** (`S-4a`) and
  should be the next read.
- **Whether `system/peer/published-root` binds its context** (§2.1) — derived, unmeasured.
