# EXPLORATION — the P2P coverage audit: what we have studied, and the five axes we have not

**Status:** Exploration (design record). Not a proposal, not normative. **This is a measurement plus a
ranked gap list, and it is meant to be re-run.**

**Operator-directed, 2026-09-06:** *"Let's pull together all our research, see if there's any gaps we
missed like this… do a little bit of scoping just to make sure there's not something else we're missing
in this whole big bag of peer-to-peer stuff."*

**Scope, per the operator:** *"We're primarily looking at the **peer-to-peer communication structure**
more so than what's inside the merkle trees — could be revision system, could be a site, could be just
regular data."* **So this audit ranks by relevance to replication, sync, discovery and verification —
not to content modelling.**

---

## §0 The result

**The corpus is strong on naming, transport, and artifact distribution, and has a hole on the axis that
matters most for what we just designed.**

> **Zero occurrences, corpus-wide, of `split-view`, `equivocation`, or `consistency proof`. The phrase
> `inclusion proof` appears only in the two documents written today.** So **the entire
> equivocation-detection literature — Certificate Transparency and its descendants — has never been
> read here**, and it is the deployed answer to the exact problem `EXTENSION-TREE` §3.3a states we
> cannot solve: *"a publisher that has not republished and an origin withholding a newer root produce
> byte-identical results at the consumer."*

**Five gaps, ranked. The first is worth more than the other four together.**

---

## §1 Method — and the invocation, quoted

**L7's standing rule: a figure from a search is void unless the command that produced it is quoted.**

```bash
cd docs/research && for t in <68 terms>; do
  n=$(grep -rio -- "$t" . | wc -l); printf "%-32s %s\n" "$t" "$n"; done | sort -k2 -n
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

### GAP 1 — Certificate Transparency and the equivocation-detection family `[ZERO]`

**Measured: `split-view` 0, `equivocation` 0, `consistency proof` 0, `transparency log` 0, `web of
trust` 0.**

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

### GAP 2 — Prolly trees and Merkle Search Trees `[ZERO]`

**Measured: `prolly` 0, `Merkle search tree` 0.**

**We chose CHAMP/HAMT — hash-keyed, order-destroying.** `EXTENSION-TREE` §3.7 records the trade
deliberately and even names the recovery path (a *"parallel path-prefix Merkle sidecar"*). **What was
never done is the comparison against the alternative family**: probabilistic B-trees (Dolt/Noms) and
**Merkle Search Trees, which is what ATProto actually uses for repositories** — order-preserving,
history-independent, and built so **two peers can diff structurally and sync only what differs.**

**Why it is on the communication axis and not the content axis:** the reason to want an MST is not
storage, it is **sync cost**. Our answer to *"what changed between your tree and mine"* is currently
range-based reconciliation or a full walk; theirs falls out of the structure. **We have studied ATProto
heavily (229 hits) and apparently never examined its tree.**

### GAP 3 — Hypercore / Dat `[ZERO, confirmed word-bounded]`

**The other major P2P replication substrate, and structurally the closest to our published root.** A
*core* is a **per-author append-only signed log with a merkle tree over it**, supporting **sparse
replication** — a reader fetches only the blocks it wants and verifies each against the signed root,
without the whole log.

**Two questions we owe it:** (a) **sparse verification** — their reader verifies an arbitrary block
against a signed root cheaply, which is rung 2 done as the *normal* path rather than an optional extra;
(b) **single-writer per core** — the constraint that buys them simplicity, and which we deliberately do
not take. Worth knowing what it buys before rejecting it.

### GAP 4 — Gossip and dissemination `[~ZERO]`

**Measured: `gossipsub` 0, `plumtree` 0, `epidemic broadcast` 2.**

**We have discovery (DHT 84, rendezvous 81, mDNS 19) and we do not have dissemination.** *"How does an
update reach the peers that care, without flooding"* is a distinct, well-studied problem —
epidemic broadcast trees, gossipsub's mesh + gossip hybrid, and their failure modes. **`EXTENSION-RELAY`
and `EXTENSION-SUBSCRIPTION` are the sections this bears on**, and `FEED`'s mirror model is a
dissemination design that has never been compared against the literature.

### GAP 5 — Perkeep permanodes, and the stable-identity-for-mutable-content pattern `[ZERO]`

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

- **Mention count is not depth.** A zero is reliable; a 30 is not evidence of a good study.
- **It does not cover the content axis on purpose** — the operator scoped it out. CRDT/local-first
  (Braid, Yjs) shows as zero and that is *fine*, because `EXTENSION-REVISION` §5.4 already sits on
  Eg-walker and the operator's own read is that the tree contents *"could be the revision system, could
  be a site, could be just regular data."*
- **No claim that the covered systems were read at source.** Several rows in prior studies are
  explicitly labelled as unopened corroboration (L18), including ATProto's `rev`, SSB's log and IPNS's
  validity window — all three still unopened, and all three are load-bearing for axis 8.

---

## §6 Sources

**In-corpus, measured this session:** the full `docs/research/` tree (65 explorations + reviews), by the
invocation in §1. **No external sources read for this document** — it is an audit of ours, and the five
gaps name what to read rather than reporting on it.
