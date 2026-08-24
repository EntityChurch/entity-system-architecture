# EXPLORATION — the universal namespace as *the* multi-peer data model, and what it settles

**Status:** analysis. **Ratifiable content is routed to proposals**; nothing normative lands here.
**Occasioned by:** `entity-browser-rust`'s `REVIEW-THE-UNIVERSAL-NAMESPACE-AND-THE-SHAPE-OF-A-SHARE`
(`67057be`) and `entity-workbench-go`'s `REVIEW-SHARE-AND-CONNECTIVITY-ALIGNMENT` (`4b34418`), which
independently found the same defect on four different surfaces.
**Read at:** core-protocol `f83c256` · browser-rust `67057be` · workbench-go `4b34418` · core-go `6fc0af3`
· core-rust `f23fb3b` · arch `e40754f`

---

## §0 Why this document exists

Two seats audited their own trees against `ENTITY-CORE-PROTOCOL` §1.4 and found the same thing: **the
multi-peer data model is fully specified, and almost nothing is built to it.** browser-rust found it in
their share/offer layer; workbench-go found it in their sync layer. Both had implemented it *correctly*
in exactly one place — the content-site layer — and not reused it.

**That is not two app bugs.** Four surfaces across two independent trees missing one landed model is a
signal about the corpus: the model is stated once, in the core protocol, in a section about *addressing*,
and every extension that needs it re-derives it or doesn't. This document states it once in the place
that governs the extension family, and then shows that **three separately-filed open questions are the
same question** — including one currently blocking the RELAY audit.

---

## §1 The model, as landed

`ENTITY-CORE-PROTOCOL` §1.4 (core-protocol `f83c256`), verbatim:

> **Cached remote data** — entities under other peers' namespaces (`/bob_id/...`, `/carol_id/...`). This
> is the local peer's knowledge of what those peers have.
>
> The cached namespace for a remote peer is **structurally identical to that peer's own authoritative
> namespace — the same paths, the same types, the same semantics.** The difference is authority: only
> the key holder's data is authoritative.

Three authority layers, same section:

| layer | what it gives | what it does not |
|---|---|---|
| **1 — local tree control** | *"A peer has full write access to its entire local tree… `/bob_id/data/file1`. **Nothing in the protocol prevents this.**"* | says nothing about truth |
| **2 — namespace authority (cryptographic)** | *"A third party can verify by checking Bob's signature."* | **authenticity** |
| **3 — trust and freshness** | — | *"Signatures prove authenticity… **They do not prove freshness.**"* |

And the invariant stated as a prohibition: *"Every layer that takes a path — store, listing, scope
filter, dispatch — operates on the absolute `/{peer_id}/...` form unchanged; **none may assume the local
peer's id.**"* §4.3 restates it (*"cached remote data stored under that peer's id prefix"*) and §1.7
carries the companion (*"Remote data is structurally identical to local data… the only difference is
authority"*).

**This is a complete design.** It is not a hint or an addressing convenience; it is the answer to "where
does another peer's data live," and it has a worked round-trip in the spec.

---

## §2 The distinction that makes it usable

Both reviews converged on the same rule, and it is the transferable result. **Two things key on a remote
peer id and they are not alike:**

| category | example | lives in | why |
|---|---|---|---|
| **Facts *about* a peer, authored by us** | our petname for Bob · our authz decision · our cache-freshness ledger · our per-site prefs | **our** namespace, Bob as a *key* | We are the authority. Bob never said it |
| **Copies of a peer's own data** | Bob's site · Bob's share manifest · Bob's followed subtree | **`/{bob}/…`**, structurally identical to his tree | Bob is the authority. We hold a cache |

**The substrate itself models the first category**, which is the precedent that settles it:
`system/capability/policy/{peer_pattern}`, where §6.2 says *"`{peer_pattern}` denotes **which remote
peer's calls the entry applies to**, not where the entry lives."* A policy entry about Bob is Alice's
entity in Alice's namespace.

**Collapsing the two is the defect.** It has now been found in four places:

| tree | surface | which way it collapsed |
|---|---|---|
| browser-rust | share / offer | copies of Bob's data held **nowhere durable** — an ephemeral in-memory cache re-fetched over the wire each browse |
| browser-rust | content sites | ✅ correct — mirror at `/{them}/…`, provenance ledger separately at `/{me}/…` |
| workbench-go | sync (`ReconcileSinceLastSeen`, `InstallRevisionFollowChain`) | copies of Bob's data land at **`/{alice}/watched/…`** — `source_prefix == target_prefix` forced, every caller passes a relative prefix, and `canonicalize` resolves relative to the **local** peer |
| workbench-go | content sites | ✅ correct — **but a port of browser-rust's design, not convergence** (their own header says so) |

**The cost is visible and was paid by hand in both trees.** browser-rust's `apply_offers` had to *learn*
replace semantics, because a listing that only inserts cannot represent a withdrawal. A subscribed mirror
has that property for free. Every surface that collapses the model re-pays that cost, once per surface.

---

## §3 Aggregation is a type-query over the universal tree, and this is the load-bearing part

browser-rust's `list_all_sites` (`src/content_site/discovery.rs`) is one `system/query/expression`
carrying **exactly one field — a `type_filter`** — issued across the whole local view with **no peer
filter**; ownership is derived afterward from the path's peer segment. Their own comment: *"The universal
tree carries the partition… it never filters by peer."*

**So the cross-peer aggregation primitive is: mirror under the owner's namespace, then query by type.**
Two consequences neither review fully drew out, and they resolve a question I ruled on yesterday:

### §3.1 The type tag is the cross-peer index key — so it MUST converge

`PROPOSAL-SHARE-AS-GRANT` §2.1 item 1 ruled that share **type tags** live under `app/share/*`. I ruled it
on naming-precedent grounds (`app/embed/*`, `app/site-*`). **browser-rust's F3 supplies the real reason,
and it is much stronger than the one I gave:** the type tag is what a cross-peer type-query matches on.
`app/entity-browser/share` would make browser↔go aggregation impossible **even with a perfect mirror**,
because the query would not match. The ruling was right; my justification for it was the weaker half.

### §3.2 The *path* does NOT need to converge — and it must not be forced to

This is the correction that falls out, and it lands on **browser-rust's §8**.

Their §8 proposes: **(1)** mirror remote share manifests to `/{them}/app/share/{id}`, *"same paths, same
types as theirs"*; **(6)** keep their own publications at `/{me}/app/entity-browser/shares/`.

**Those two are inconsistent with each other, and item 1 is the one that breaks §1.4.** If browser-rust
publishes its own shares at `app/entity-browser/shares/{id}`, then a `entity-workbench-go` peer mirroring
them must write `/{browser}/app/entity-browser/shares/{id}` — because §1.4 requires *the same paths*. A
mirror that rewrites the path to a convention of the mirrorer's choosing is not a cached namespace; it is
a translation, and it re-introduces exactly the "we assert our namespace over their bytes" problem the
review is trying to eliminate.

**The resolution:** a mirror **preserves the publisher's path verbatim**. It does not impose the
mirrorer's convention, and it does not require a shared path convention to exist at all — because
aggregation keys on the **type tag** (§3.1), not the path. Item 6 is correct and item 1 should read:

> mirror the publisher's subtree at `/{them}/{whatever prefix the publisher declares}`, verbatim.

**This is the charter's own stance, which browser-rust already quotes:** *"paths are convention; the
entity graph is coherence."*

### §3.3 Which raises the one genuinely open question — and it is already specified

If paths stay app-local, **how does a consumer learn which prefix to mirror?**

`EXTENSION-TREE` §3.3a: `system/peer/published-root` carries a **REQUIRED `prefix`** — *"the prefix these
trie keys are relative to… MUST end with `/`"* — and the spec is explicit about why it is required rather
than defaulted: *"any default would have to be one of §3.3's three shapes, silently promoting one
implementation's convention."* And: *"**Two conformant publishers may legitimately publish different
extents.** `prefix` is what distinguishes them… A consumer MUST read the extent from `prefix` and MUST
NOT infer it."*

**`published-root.prefix` is the mechanism that makes app-local mirror paths work.** It was designed for
the walk-from-signed-root threat model; it turns out to be the same field that answers "what do I
mirror." That is not a coincidence — both questions are *"what subtree does this peer commit to."*

---

## §4 Three open questions are one question

### §4.1 browser-rust's F4 — §2.1.3 narrowed, and the narrowing holds

They asked: is the residual of *"verification through aggregation"* exactly **(freshness + extent)**,
with **authenticity discharged by §1.4 layer 2**?

**Yes, and the reasoning is sound.** When Bob's entity sits at `/{bob}/…` carrying Bob's signature, a
third party reading it *from us* is in the position §1.4 layer 2 already describes — verify against the
key holder. The hard version of §2.1.3 existed because we were laundering Bob's bytes into *our* path,
where the path asserts an authority the bytes do not carry. **Fix the storage shape and the authenticity
half dissolves.**

What remains is precisely what §1.4 layer 3 and §3.3a name:

- **Freshness** — *"signatures prove authenticity… they do not prove freshness."* `published-root` carries
  `seq` with a MUST-reject-on-rollback rule, which is the freshness instrument.
- **Extent** — *which* subtree Bob claims to publish. `prefix`, per §3.3.

**So §2.1.3's residual is (freshness + extent), and `published-root` is the answer to both halves.**

### §4.2 …which is the same as §3.3's mirror-prefix question

§3.3 asks "what prefix do I mirror" and answers `published-root.prefix`. §4.1 asks "what extent does Bob
claim" and answers `published-root.prefix`. **These are one question with one answer**, and the fact that
they were filed separately — one as a federation-verification problem, one as an app storage-shape
problem — is itself the finding.

### §4.3 RETRACTED — this section overreached and must not be built on

> **RETRACTED 2026-08-17, same day, by the operator.** The section below argued that Mode A "is not an
> intermediary carrying opaque envelopes — it is a peer with a large local view," and presented that as a
> reframing of RELAY's mode set. **That is not a sound basis for any disposition, and the framing is
> wrong in a way worth stating precisely:**
>
> **An intermediary carrying opaque envelopes is the core relay function, not a problem to dissolve.**
> Moving encrypted messages between peers that cannot reach each other, hosting encrypted relay messages
> on a static CDN, and holding a live circuit for an unreachable peer are all valid, needed relay
> functionality. Nothing below displaces any of it, and writing "it is not X — it is Y" about one mode
> read as devaluing X.
>
> **What is actually wrong is the method.** This is the **third** time I have produced an argument about
> RELAY's mode set without doing the landscape/prior-art analysis the disposition requires —
> `HANDOFF-2026-08-17-relay-audit` §3a names eleven legacy documents, starting with the study that
> *produced* the four-mode model, and I have still not read them. **A decomposition argument is not a
> landscape analysis.** Showing that one shape of aggregation is expressible another way says nothing
> about what relay functionality the system needs, how SMTP / Nostr / ActivityPub / Matrix / mixnets
> solve it in current practice, or where ours fits among them.
>
> **The retained observation, at its real weight:** the §1.4 cached-remote-namespace model gives us a
> place to put another peer's data *as that peer's data*, which several federated systems lack. That is
> **an input to the landscape analysis, not a conclusion from one.** It may turn out to change where a
> boundary sits; it may turn out to change nothing. It is not evidence for any disposition until the
> study it would be reconciled against has been read.
>
> **Superseded by** `EXPLORATION-RELAY-LANDSCAPE-AND-PRIOR-ART` (in progress). The RELAY ruling remains
> **PROVISIONAL**; §5's SUBSCRIPTION-path finding and §1–§3 of this document are unaffected and stand.

<details>
<summary>Original §4.3 text, retained for the record — do not cite</summary>

This is the reconciliation the relay audit needs, and it arrives from an unexpected direction.

**RELAY Mode A** is *"an aggregator subscribes to N publisher peers' subtrees and serves a unified stream
or queryable view."* Under §1.4 + §3, that decomposes into two things that are **both already fully
specified and neither of which is relaying:**

1. **Subscribe N publishers' subtrees and materialize them locally** → `/{pub_1}/…`, `/{pub_2}/…`.
   Ordinary cross-peer subscription into cached remote namespaces. `EXTENSION-SUBSCRIPTION` §8.1–§8.2.
2. **Serve a queryable view** → a type-query with no peer filter over the local view (§3).

**An aggregator is not an intermediary carrying opaque envelopes between two endpoints. It is a peer with
a large local view.** That is why RELAY §1's definition does not fit it, why §10.4's MUST-NOT-decode
clause contradicts §1's *"queryable view"*, and why Go's `core/types/relay.go` refused to stub it —
**there was no coherent slot because the mechanism belongs to a different part of the system entirely.**

**And it explains the landscape study, which is the thing I was missing when I ruled and downgraded.**
The study grounds Mode A in Nostr, ATProto (*"a firehose of signed records"*) and Mastodon relays — every
real system it derived the enumeration from calls the aggregator a relay. **Those systems have no
universal namespace.** Lacking a place to *put* another peer's data as that peer's data, they must model
aggregation as a transport role, because the aggregator's own store is the only namespace available. We
have `/{bob}/…`. **The industry conflates the two because their data model forces them to; ours does
not.**

> **This does not settle the disposition and must not be spent as if it does.** It is a new argument for
> disposition 1 (aggregate leaves RELAY) on grounds *stronger* than the coherence argument I ruled on and
> then downgraded — the mechanism is fully specified elsewhere, so RELAY loses nothing by not carrying it.
> But the audit's §3a corpus is still unread, including the receive-side-opacity design record that
> quotes the same MUST NOT *with `aggregate` already in the list*. **The ruling stays PROVISIONAL.** What
> changes is that the audit now has a model to reconcile the landscape study against, which is exactly
> what it was missing.

</details>

---

## §5 What this says about the corpus

**One landed model, stated once, in a core-protocol section about addressing — and four surfaces in two
trees missed it.** That is the shape `AGENTS.md` already names for load-bearing invariants: *"the model
is correct but scattered."* §1.4 is on the FOREGROUND list (invariant 3, *"universal address space +
local-view authority"*), and it still did not reach the extension family.

**The gap is not in the core protocol; it is that no extension spec restates it as an obligation.**
`EXTENSION-SUBSCRIPTION` §8.2 specifies materializing upstream data *"into the local tree"* without
saying **at which path** — and that silence is precisely where workbench-go's sync layer landed remote
data under the local peer. A clean-room implementer reading SUBSCRIPTION alone cannot get this right.

**Routed as a proposal, not ruled here:** SUBSCRIPTION (and any extension that materializes remote data)
should state the §1.4 obligation explicitly — *materialized remote data is written under the source
peer's namespace at the source's declared prefix.* This is a T1 generator-readiness item as much as a
correctness one: an extension that cannot be implemented from its own text by an author who may not read
siblings is exactly what blocks the generated-extension pipeline.

---

## §6 What this document does NOT claim

- **Not** that RELAY Mode A's disposition is settled. §4.3 supplies an argument, not a ruling; the §3a
  legacy corpus is still unread and the ruling remains PROVISIONAL.
- **Not** that browser-rust's or workbench-go's site layers are independent convergence. workbench-go's
  is a **port** of browser-rust's and says so in its own header; citing it as ADR-0012 evidence would
  spend evidence we do not have.
- **Not** that any of §3.2's correction is built, or that either seat has agreed to it.
- **Not** a claim about `entity-core-py`'s query surface — browser-rust's §9 #6 asks whether a type-query
  is available there, and this document does not answer it.
