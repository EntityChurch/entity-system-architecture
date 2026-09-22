# EXPLORATION — the P2P coverage audit: what we have studied, and the five axes we have not

> ## ⚠ CORRECTED 2026-09-07 — FOUR OF THE FIVE `[ZERO]` GAPS ARE NOT ZERO. READ THIS FIRST.
>
> **The measurement below searched one directory of one repository and reported the result as
> *corpus-wide*.** The project's design reasoning through core revisions v0.01–v7.0 lives in a
> **separate pre-split archive of 1,068 documents** — not part of this published corpus — that this
> audit never opened and that no analyzer's scope reaches. Re-measured against it:
>
> | Gap | Claimed | Actually, in the archive |
> |---|---|---|
> | **1 — Certificate Transparency / equivocation** | `[ZERO]`, *"the single highest-value unread body of work"* | **`equivocation` 13 docs · `inclusion proof` 15 · `transparency log` 6 · `certificate transparency` 4** |
> | **2 — Prolly trees / Merkle Search Trees** | `[ZERO]`, *"apparently never examined ATProto's tree"* | **A dedicated primary-source audit** — `EXPLORATION-MST-AND-SORT-ORDERED-AUDIT`, whose §1 is *"AT Protocol's MST in production (primary-source verified)"* — structure, what the root commits to, why Bluesky chose it, and its production scars — plus §2.2 Prolly trees and §2.3 Merkle Search Tree (Auvolat–Taïani 2019). `prolly`/`merkle search tree`: **7 / 6 docs** |
> | **3 — Hypercore / Dat** | `[ZERO]` | **1 doc.** Substantially stands |
> | **4 — Gossip and dissemination** | `[~ZERO]`, *"we do not have dissemination"* | **115 docs.** `EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP` separates relay / routing / gossip as three primitives, draws the relay-routing boundary, and defers gossip **with a named home**; `PROPOSAL-EXTENSION-GOSSIP` exists as a stub. `epidemic broadcast` 10 · `anti-entropy` 12 |
> | **5 — Perkeep permanodes** | `[ZERO]` | **1 doc.** Substantially stands |
>
> **What survives, and it is most of the value.** The *literature* pointers are still worth reading —
> `gossipsub` (1) and `plumtree` (0) really are near-absent, RFC 6962/9162 were never read at source,
> and the archive's coverage is scattered rather than a landscape read. **What does not survive is the
> ranking and the framing.** Gap 1 is not an unread field; it is a *scattered* one that was never
> synthesized. Gap 2 is not open at all — it was audited primary-source and given a verdict. **A gap
> that has been studied and not consolidated is a different, much cheaper problem than one that has
> never been looked at, and the two call for opposite work.**
>
> **The cause, which is the durable half and the transferable one.** A repository split moved the
> *conclusions* into this corpus and left the *derivations* in the archive — and every search
> instrument was scoped, reasonably, to the surviving half. So a *"have we studied X?"* question
> returns a confident **no**, and the next study re-derives from scratch, usually worse than the
> original. **The archive is now title-indexed and the index is part of the search path.**
>
> **The general form, for anyone auditing coverage of anything:** *an absence measured inside a scope
> is evidence about the scope, never about the subject.* State the region searched, and check whether
> the project's own history has a region outside it. *(This is the second confirmed false negative
> from this one cause in two days; the first retracted a claim that a foundational theorem was new to
> us.)*

**Status:** Exploration (design record). Not a proposal, not normative. **This is a measurement plus a
ranked gap list, and it is meant to be re-run — with the scope correction above.**

**Scope:** the **peer-to-peer communication structure** — replication, sync, discovery and
verification — rather than what is inside the Merkle trees, which may be a revision system, a site, or
ordinary data. The ranking below is by relevance to that axis, not to content modelling.

---

## §0 The result

**The corpus is strong on naming, transport, and artifact distribution.** The claim that followed —
that it has a hole on the axis that matters most for what we just designed — **is withdrawn as
measured; see the correction banner.** What is true is narrower and still worth acting on:

> **The equivocation-detection family — Certificate Transparency and its descendants — is present in
> the archive across 13–15 documents and has never been read as a landscape or synthesized into this
> corpus.** It is the deployed answer to the exact problem `EXTENSION-TREE` §3.3a states we cannot
> solve: *"a publisher that has not republished and an origin withholding a newer root produce
> byte-identical results at the consumer."* **The work owed is consolidation plus a primary-source
> read of RFC 6962 / 9162 — not a first contact.**

**Five gaps, ranked as originally written. Read the banner for what each one actually is.**

---

## §1 Method — and the invocation, quoted

**L7's standing rule: a figure from a search is void unless the command that produced it is quoted.**
**Quoting it is what exposed this audit's own defect** — the scope is visible in the first two words.

```bash
cd docs/research && for t in <68 terms>; do          # ← ONE directory of ONE repo.
  n=$(grep -rio -- "$t" . | wc -l); printf "%-32s %s\n" "$t" "$n"; done | sort -k2 -n
```

**The corrected invocation searches both regions** — this corpus and the pre-split archive, which
holds 1,068 of the project's roughly 1,300 design documents — and reports **documents matched**
rather than raw occurrences, since a term with thirty hits in one study is better covered than one
with sixty scattered as asides:

```bash
for t in <terms>; do printf '%-28s %s\n' "$t" "$(grep -rliE -- "$t" <corpus> <archive> | wc -l)"; done
```

**Three counts from the first pass were substring artifacts and are corrected here** — the same defect
`spec census` was rebuilt four times to avoid (**L8's twentieth form**: a count of a token is not a
census of the thing).

| Term | Naive | Word-bounded | Cause |
|---|---|---|---|
| `Dat` | 935 | **0** | matched *data*, *update*, *validate* |
| `ICE` | 904 | 45 | matched *service*, *choice*, *practice* |
| `DID` | 350 | 269 | matched *did* |
| `OCI` | 147 | 21 | matched *associated*, *reciprocity* |

**Counts are mention-frequency, not depth** — a term with 30 hits in one study is better covered than
one with 60 hits scattered as asides. Treat the table as a *screen for absence*, which is what it is
reliable for.

---

## §2 What is well covered — do not re-derive these

| System / topic | Hits | Where the depth is |
|---|---|---|
| **ATProto / Bluesky** | 229 / 12 | the reference arc; `REFERENCE-PRIOR-ART-…-BY-AXIS` |
| **Nostr** | 194 | the reference arc, NIP-01 opened at source |
| **Matrix** | 123 | `THE-CONVERGENT-DESIGNS-…` |
| **Willow** | 118 | `WILLOW-THE-NEAREST-NEIGHBOUR-READ-AGAINST-ITS-SPEC` — read against the spec |
| **SSB** | 98 | convergent-designs study |
| **Signal** | 191 | messaging/encryption |
| **DID** | 269 | identity arc |
| **TURN / WebRTC / STUN / ICE** | 145 / 104 / 29 / 45 | `CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE` |
| **DHT / rendezvous / mDNS** | 84 / 81 / 19 | discovery + signaling |
| **libp2p** | 72 | transport family |
| **Nix / TUF / OCI / Sigstore / Guix** | 64 / 37 / 21 / 9 / 3 | the supply-chain pair; **TUF's tier split is landed in `REGISTRY` §6a.7** |
| **IPFS / IPNS / BitTorrent** | 45 / 10 / 36 | naming + reference arcs |
| **petname / Zooko** | 35 / 13 | `GUIDE-RESOLUTION` §6.0 — landed |
| **Radicle / ForgeFed** | 36 / 12 | **added today** |
| **range-based set reconciliation** | 9 (+21 `set reconciliation`) | via the Willow study |
| **Automerge / Eg-walker** | 10 / 13 | `EXTENSION-REVISION` §5.4 |

---

## §3 The five gaps, ranked

### GAP 1 — Certificate Transparency and the equivocation-detection family ~~`[ZERO]`~~ **SCATTERED, NEVER SYNTHESIZED**

~~**Measured: `split-view` 0, `equivocation` 0, `consistency proof` 0, `transparency log` 0, `web of
trust` 0.**~~ **Those figures are this corpus only.** Adding the pre-split archive:
`equivocation` **13** · `inclusion proof` **15** · `transparency log` **6** ·
`certificate transparency` **4** · `web of trust` **4** · `consistency proof` **0**.

**So the gap is real but is a different gap.** The vocabulary is present; what is absent is a
landscape read and a synthesis — the material sits inside encryption proposals, registry-manifest
explorations and an authenticated-data-structures study, each using it for its own local purpose.
**`consistency proof` at zero across both regions is the sharpest true finding here**, and it is
precisely CT's freshness mechanism. Everything below stands as the argument for doing the work; only
*"has never been read here"* is withdrawn.

**Why it is first, and it is not close.** CT exists to answer one question: *how do you detect that a
server showed **different views to different clients**?* Its mechanisms are **inclusion proofs** (this
entry is in the log), **consistency proofs** (this log is an append-only extension of the one I saw
before), **gossip** between clients so views can be compared, and **witnesses/auditors** who
countersign what they observed.

**Map that onto our ladder and every rung we could not close is there:**

| Our open problem | CT's mechanism |
|---|---|
| **Rung 3** — *a withholding origin and a quiet publisher are byte-identical* (`TREE` §3.3a) | **consistency proof + gossip** — clients compare roots, and a split view becomes detectable rather than invisible |
| **Rung 4** — completeness is unattainable | **the log is the completeness claim**; an omission is a failed inclusion proof against a root others also saw |
| **D-45** — witness signatures, *"I don't know what it exactly tells you"* | **the witness/auditor role, fully worked** — signing what you observed, at a time, is CT's core primitive |
| **`published-root`'s `seq` monotonicity** | CT's **append-only** property, with a proof rather than an honour system |

> **This is the single highest-value unread body of work for this design.** We independently arrived at
> a signed monotone root and then wrote down, honestly, that it cannot prove freshness. **CT is twelve
> years of deployment on precisely that limitation.**

**Read at source:** RFC 6962 and RFC 9162 (CT v2), plus the gossip drafts.

### GAP 2 — Prolly trees and Merkle Search Trees ~~`[ZERO]`~~ **RETRACTED — THIS WAS DONE, PRIMARY-SOURCE**

> ~~**Measured: `prolly` 0, `Merkle search tree` 0.**~~ **Both non-zero in the pre-split archive
> (7 and 6 documents), and one of them is a dedicated audit of exactly this question.**
> `EXPLORATION-MST-AND-SORT-ORDERED-AUDIT` carries **§1 *"AT Protocol's MST in production
> (primary-source verified)"*** — structure per `atproto.com/specs/repository`, what the MST root
> cryptographically commits to, what ATProto production operations actually use, why Bluesky picked
> it, and its production scars at scale — then **§2.2 Prolly trees (Noms / Dolt / IPLD ADL)**,
> **§2.3 Merkle Search Tree (Auvolat–Taïani 2019)**, and **§4 a verdict** that the substrate choice
> stands, with errata against `EXTENSION-TREE` §3.7. Its companion
> `EXPLORATION-AUTHENTICATED-DATA-STRUCTURES` compares HAMT / Patricia / MPT / JMT against production
> deployments and reverses its own recommendation in an addendum.
>
> **The sentence below — *"we have studied ATProto heavily and apparently never examined its tree"* —
> is false, and it is the most expensive kind of false: it re-opened a settled question and ranked it
> second of five.** What may still be owed is carrying that audit's verdict and errata forward into
> this corpus, which is a fold, not a study.

**We chose CHAMP/HAMT — hash-keyed, order-destroying.** `EXTENSION-TREE` §3.7 records the trade
deliberately and even names the recovery path (a *"parallel path-prefix Merkle sidecar"*). **What was
never done is the comparison against the alternative family**: probabilistic B-trees (Dolt/Noms) and
**Merkle Search Trees, which is what ATProto actually uses for repositories** — order-preserving,
history-independent, and built so **two peers can diff structurally and sync only what differs.**

**Why it is on the communication axis and not the content axis:** the reason to want an MST is not
storage, it is **sync cost**. Our answer to *"what changed between your tree and mine"* is currently
range-based reconciliation or a full walk; theirs falls out of the structure. **We have studied ATProto
heavily (229 hits) and apparently never examined its tree.**

### GAP 3 — Hypercore / Dat `[ZERO here, 1 archive document]` — **substantially stands**

**The other major P2P replication substrate, and structurally the closest to our published root.** A
*core* is a **per-author append-only signed log with a merkle tree over it**, supporting **sparse
replication** — a reader fetches only the blocks it wants and verifies each against the signed root,
without the whole log.

**Two questions we owe it:** (a) **sparse verification** — their reader verifies an arbitrary block
against a signed root cheaply, which is rung 2 done as the *normal* path rather than an optional extra;
(b) **single-writer per core** — the constraint that buys them simplicity, and which we deliberately do
not take. Worth knowing what it buys before rejecting it.

### GAP 4 — Gossip and dissemination ~~`[~ZERO]`~~ **THE PRIMITIVE IS NAMED AND SCOPED; THE LITERATURE IS THIN**

> **`gossip` appears in 115 archive documents**, `epidemic broadcast` 10, `anti-entropy` 12.
> `EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP` — *"how information travels: relay, routing,
> and gossip (the three primitives)"* — **already draws the boundary this arc has been re-deriving**:
> relay is addressed unicast and forwards one hop; routing is the next-hop *decision* and is never
> relay's algorithm; gossip is **unaddressed** spread with its own loop control and dedup and no
> single destination. It reviews the V2/V3 message-forwarding history, and **defers gossip with a
> named home** — `PROPOSAL-EXTENSION-GOSSIP` exists there as a stub.
>
> **What survives is narrow and still true: the modern literature is near-absent** — `gossipsub` 1,
> `plumtree` 0 across both regions. So the owed work is a **literature read against a boundary we
> already drew**, not a first analysis of dissemination.

~~**We have discovery (DHT 84, rendezvous 81, mDNS 19) and we do not have dissemination.**~~ We have
discovery, and we have dissemination *named and bounded* without the failure-mode literature.
*"How does an update reach the peers that care, without flooding"* is a distinct, well-studied
problem —
epidemic broadcast trees, gossipsub's mesh + gossip hybrid, and their failure modes. **`EXTENSION-RELAY`
and `EXTENSION-SUBSCRIPTION` are the sections this bears on**, and `FEED`'s mirror model is a
dissemination design that has never been compared against the literature.

### GAP 5 — Perkeep permanodes, and the stable-identity-for-mutable-content pattern `[ZERO here, 1 archive document]` — **substantially stands**

**Perkeep's *permanode* is a random, signed, immutable node whose identity never changes, with mutation
expressed as signed claims that reference it.** That is **a third answer to the mutable-pointer problem**
alongside our `published-root` and the operator's binding assertion — and it inverts ours: the identity
is content-free and *only* the claims are signed. **Small body of work, directly on the `S-4a` question,
and cheap to read.**

### Lower priority, recorded so the next audit does not re-discover them as novel

**GNUnet 0 · Freenet 0 · Tahoe-LAFS 0 · Delta Chat 0 · MLS 2 · Braid/Yjs 0.** MLS is the notable one —
**group key agreement** is a real axis we have barely touched (`EXTENSION-GROUP` exists), but it is
encryption rather than the replication structure the operator scoped this to.

---

## §4 The better instrument — audit by AXIS, not by system

**A system list finds gaps by luck; an axis list finds them by construction.** This is the transferable
half of the audit, and it is what should be re-run rather than the grep.

| # | Axis | Status | Best-covered source |
|---|---|---|---|
| 1 | **Naming / addressing** | ✓ strong, landed | petnames, Zooko, DID |
| 2 | **Discovery** — who has it | ✓ | DHT, rendezvous, mDNS |
| 3 | **Connectivity** — reaching them | ✓ strong | WebRTC/ICE/TURN study |
| 4 | **Replication** — moving bytes | ✓ | IPFS, BitTorrent |
| 5 | **Sync / reconciliation** — converging efficiently | ~ partial | RBSR via Willow; **MST/prolly missing (GAP 2)** |
| 6 | **Dissemination** — propagating updates | **✗ GAP 4** | — |
| 7 | **Authorship / verification** | ✓ **strong as of today** | the verification ladder |
| 8 | **Freshness / anti-rollback** | ~ partial | TUF tier split, `seq` |
| 9 | **Equivocation / split-view detection** | **✗ GAP 1 — zero** | — |
| 10 | **Completeness / omission detection** | **✗ rung 4, named unattainable** | — |
| 11 | **Access control** | ✓ | capabilities; Willow's model |
| 12 | **Group key agreement** | ~ thin | MLS unread |
| 13 | **Moderation / abuse** | **✗ unexamined** | — |
| 14 | **Incentives** | ✗ (likely out of scope) | BitTorrent tit-for-tat |

**Axes 9 and 10 are the same gap seen twice, and GAP 1 addresses both.** Axis 13 is genuinely
unexamined and is a social-tier question rather than a structural one — flagged, not ranked.

---

## §5 What this audit does NOT establish

- ~~**Mention count is not depth.** A zero is reliable; a 30 is not evidence of a good study.~~
  **Half of that is wrong and it is this audit's central defect.** A high count is indeed not evidence
  of depth — but **a zero is only ever reliable about the region searched**, and four of five zeros
  here were artifacts of a scope that excluded 1,068 of the project's design documents. *A zero is a
  fact about where you looked. Publish the region with it or do not publish the zero.*
- **It does not cover the content axis on purpose** — that was scoped out. CRDT/local-first
  (Braid, Yjs) shows as zero and that is *fine*, because `EXTENSION-REVISION` §5.4 already sits on
  Eg-walker, and the tree's contents may be a revision system, a site, or ordinary data.
- **No claim that the covered systems were read at source.** Several rows in prior studies are
  explicitly labelled as unopened corroboration (L18), including ATProto's `rev`, SSB's log and IPNS's
  validity window — all three still unopened, and all three are load-bearing for axis 8.

---

## §6 Sources

**Measured 2026-09-06:** the `docs/research/` tree of this repository alone — 65 explorations and
reviews — by the invocation in §1. **That is roughly 5% of the project's design record, and treating
it as the whole is the defect corrected at the top of this document.**

**Re-measured 2026-09-07:** the same terms against this corpus **plus the pre-split architecture
archive** — 1,068 documents spanning core revisions v0.01 through v7.0, now title-indexed and part of
the standard search path.

**No external sources read for this document** — it is an audit of ours, and the five gaps name what
to read rather than reporting on it.
