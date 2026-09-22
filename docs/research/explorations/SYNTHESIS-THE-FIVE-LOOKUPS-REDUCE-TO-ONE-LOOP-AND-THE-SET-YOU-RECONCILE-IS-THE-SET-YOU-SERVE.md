# SYNTHESIS — the five lookups reduce to one loop, and the set you reconcile is the set you serve

**Status:** SYNTHESIS (2026-09-13). **This is a refinement pass over five survey documents, not a
proposal and not a ruling.** It argues one assembly, names what is already built, and is explicit
about the one move that is not yet anywhere.

**Refines:** `FIVE-KEYED-LOOKUPS` · `THE-CONTENT-LOOKUP-LANDSCAPE` · `THE-CONTENT-ADDRESSED-LANDSCAPE` ·
`THE-FOURTH-MECHANISM-IS-ASK-EVERYONE` · `WEB-CACHING-RAN-ALL-FOUR-MECHANISMS`. Register rows
`LK-1`…`LK-51`.

---

## §0 The result

**A design does converge, and its shape is this:**

> ⭐⭐⭐ **There is no content lookup. There is a set you would serve, a digest of it, and a
> reconciliation loop that keeps peers' copies of each other's digests current. *"Who has `H`"* is then
> a LOCAL query over digests you already hold — not a network operation at all.**

**Five things fall out of that one move, and each answers a question the survey left open:**

| the survey's open question | what falls out |
|---|---|
| **ICP's false hit** — a peer says HIT, then refuses | ⭐ **cannot occur.** The digest is computed over `serve_scope`, and **the same capability evaluator decides both the digest and the answer** — *"one ACL machinery, no drift"* is already the landed rule |
| **staleness** (`LK-8`, `LK-13`, `LK-48`) | ⭐ **already paid for.** A publisher advertising a signed pointer **already MUST republish within a 30-second convergence bound**; a digest rides that republish. **Reconciliation *is* the refresh** |
| **the orphaned-data / GC problem** (`LK-48`) | ⭐ **does not arise.** A digest is **derived from your own store**, never accumulated from others' claims. Nothing to collect |
| **the privacy leak** (`LK-14`, `LK-34`) | ⭐ **resolves on the existing default.** Within `serve_scope` **hash-knowledge is already the read authority** — so a digest grants no access; it only removes the need to guess. **Enumerating a `published-set` you deliberately published is not a leak.** `whole-store` would be, and it is already a warned debug opt-in |
| **the leaf-count / aggregation problem** (`LK-9`, `LK-15`, `LK-47b`) | ⭐ **granularity follows the relationship.** Reconciliation needs only a **total order**, which hashes have; routing needs topology correlation, which they never have. Per-blob inside a fleet, per-namespace with a stranger |

**And it is scale-invariant, which `LK-33` says is the property that decides survival:** at N=1 the
digest is your own index — `EXTENSION-QUERY` §2.2's reverse hash index, already a Level-1 **MUST**. At
N=2 you reconcile. At N=many you reconcile with whom you know, and for everyone else **the reference
already carries its publisher** (`APP-CONVENTION-REFERENCE` §1).

> ⚠ **The honest scope of the claim: almost none of this is new, and the one move that is new is
> small.** §4 lists every component and where it already lives. **The novelty is one sentence — *the
> set you publish a digest of is the set `serve_scope` already defines* — and everything else is
> composition.**

---

## §1 The reduction — four mechanisms, one axis

The survey found four ways to answer *who has `H`*. **They are not four designs; they are four points
on one axis — where the index lives.**

| | where the index lives | messages per query | what it requires | deployed as |
|---|---|---|---|---|
| **COMPUTE** | **nowhere — it is a function** | **0** | an **agreed map** | CARP · CRUSH · Tor's HSDir ring · Dynamo's preference list |
| **PUBLISH** | **distributed across holders**, each publishing its own slice | **0** at query time | **nothing live** | Cache Digests · git-annex's location log |
| **LOOK UP** | **centralized in infrastructure** | 1 | **something online** | Kademlia · trackers · IPNI · NDN's FIB |
| **ASK** | **nowhere — reconstructed per query** | **N** | **nothing at all** | ICP · Gnutella · our archived fan-out `QUERY` |

⭐⭐ **Set reconciliation collapses ASK and PUBLISH into one operation**, and that is the reduction the
whole survey was circling:

- *"Publish your set and I will read it"* is a **digest**.
- *"Tell me whether you have this"* is **ICP**.
- *"Let us establish where our sets differ"* is **reconciliation** — and its cost is **proportional to
  the difference, not to either set**.

**Reconciliation strictly dominates both wherever the sets overlap heavily** — which is precisely the
fleet case, the group case, and any two peers who have talked before. **Erlay's hybrid is the deployed
proof**: it did not replace the flood, it bounded it — announce to eight, reconcile with everyone else.

⇒ **The corpus does not need to choose one mechanism. It needs the loop, and the loop degrades
gracefully to each of the four as the scope of agreement shrinks.**

---

## §2 Scope selects the mechanism, and the map is publishable

| scope | membership | mechanism available | what the corpus already has |
|---|---|---|---|
| **one peer** | trivial | **local index** | `EXTENSION-QUERY` §2.2 reverse hash index — **MUST, Level 1** |
| **a fleet you own** | closed, known, complete | ⭐ **COMPUTE**, or reconcile everything | the archive's `system/sharding/config`; `EXTENSION-ROUTE`'s deferred computed hatch |
| **a group / organization** | bounded, administered | **reconcile within; publish across** | `EXTENSION-GROUP` · `EXTENSION-ROLE` · capability scoping |
| **the open network** | unbounded | **publish + the reference's own locator**; ask only within a bounded fan-out | `APP-CONVENTION-REFERENCE` §1 · `system/peer/transport-set` · `DATA-EXCHANGE`'s ordered SOURCE set |

**The move that makes this one design rather than four** — already recorded at `LK-12` and already
proposed three times in the archive — **is that the map is an ordinary signed entity.** A membership
set is a capability-governed artifact here; a placement rule over it is data. **So COMPUTE is available
exactly where a map can be agreed, and past that boundary the same digests still publish and still
merge.** You lose determinism, not the mechanism.

⭐ **CoralCDN is the deployed precedent and it is a three-level version of this table** — query your
level-2 cluster, then level-1, then global, stopping at the first hit.

---

## §3 The assembly

### §3.1 The set is `serve_scope`, and that is the whole move

`EXTENSION-NETWORK` §6.5.6 already defines what a peer answers for, and already makes it a
**capability token**:

> **the published `serve_scope.cap` IS the effective cap** passed to the evaluator: where the live
> surface asks *"does the connection's cap-set permit `get(path)`?"*, the serving surface asks *"does
> `serve_scope.cap` permit `get(path)`?"* — **one ACL machinery, no drift.**

⭐⭐⭐ **Therefore: publish a digest of `serve_scope`.**

**It is exactly the set you will serve, to exactly the audience that scope defines, because one
evaluator decides both the digest and the answer.** ICP's false hit — *an object present in cache but
not accessible for a sibling cache* — **cannot occur by construction**, and that is the failure that
took down inter-cache lookup thirty years ago.

**The per-peer form is the same machinery with the other input.** Where the surface is authenticated,
the evaluator takes the connection's cap-set instead of the published cap — so a per-peer digest is
*the existing evaluation, memoized*, not a new concept.

### §3.2 Staleness is already paid for

A publisher advertising a `signed_pointer` **already MUST** republish so the published root
**converges within 30 seconds**, with `seq` monotonic and `predecessor` chaining the prior root. **A
digest rides that republish.**

⇒ ⭐ **The survey's universal answer — soft state with mandatory refresh (`LK-48`) — is already
normative here for the root, and the digest inherits it at no additional cost.** Where the field pays
for re-announcement (Kademlia), periodic rebuild (Squid) or a revolving door (OpenDHT), **this corpus
already pays a republish for another reason.**

### §3.3 The privacy question resolves on the existing default

**The precise statement, which is sharper than anything earlier in this arc:**

- The static read routes **carry no request auth**. Within `serve_scope`, **hash-knowledge (content) or
  path-presence (tree) *is* the read authority**.
- ⇒ **A digest grants no access it did not already grant. It removes the need to GUESS.**
- ⇒ **Enumerating a `published-set` — the `SHOULD` default, and a set you deliberately published — is
  not a leak.** Enumerating `whole-store` would be, and `whole-store` is already a debug opt-in that
  ships with a startup warning.

⚠ **And the earlier framing needs this correction on the record:** *"there is always a second hop"* is
true of the **live EXECUTE surface**, where a capability gates every read, and **false of the static
serving surface**, where the scope is the gate and hash-knowledge suffices inside it. **Both are
correct designs; the digest question only ever concerned the second.**

⚠ **What remains, unchanged from `LK-34`:** the **interest** leak. Reconciling with a peer tells them
what you are looking for, and no derivation in `LK-22` addresses it. **Reconciliation narrows it
usefully** — you reconcile with peers you already chose to talk to — but it does not close it.

### §3.4 What a holder claim becomes

**It stops being a claim.** There is no *"I hold `H`"* assertion to go stale, be probed, or be
believed. There is:

- a **digest**, derived from your store, scoped by the capability you already evaluate, refreshed by a
  republish you already owe; and
- a **reconciliation**, whose cost is the difference and whose result is local.

⇒ `LK-13`'s probe and `LK-38`'s hinted handoff are **still useful and are no longer load-bearing** —
the probe becomes a cheap verification of a reconciled difference, and hinted handoff remains the right
shape for *temporary custody with an owner pointer*, which a digest cannot express.

---

## §4 What this is made of — and what is genuinely missing

| piece | where it lives | state |
|---|---|---|
| the local index (`hash → my entities`) | `EXTENSION-QUERY` §2.2 | ⭐ **landed, MUST, Level 1** |
| what a peer answers for, as a capability | `EXTENSION-NETWORK` §6.5.6 `serve_scope` | ⭐ **landed** |
| one evaluator for scope and answer | same, *"one ACL machinery, no drift"* | ⭐ **landed** |
| bounded-convergence republish (30 s), `seq`, `predecessor` | `EXTENSION-NETWORK` §6.5.6 | ⭐ **landed** |
| range-summarizable backend for reconciliation | the canonical Merkle trie; `EXTENSION-TREE` §3.7.1 | ⭐ **landed, built for other reasons** |
| the reconciliation technique itself | `THE-CONVERGENT-DESIGNS` §7 (RBSR, scored against the sketch family) | **studied, not specified** |
| an ordered, scheme-open set of sources | `DATA-EXCHANGE` §6.2 | **DRAFT, partially folded** |
| `absent` vs `refused` as distinct outcomes | `DATA-EXCHANGE` §6.2.1 | ⭐ **DRAFT — and it is exactly ICP's missing discrimination** |
| closure — a gathered result is the same kind of object | `DATA-EXCHANGE` §5.0 | **DRAFT, the anti-centralization spine** |
| a reference that carries its publisher | `APP-CONVENTION-REFERENCE` §1 | ⭐ **landed** |
| `peer → transports` | `EXTENSION-NETWORK` §6.5.1c | ⭐ **landed** |
| the agreed map for COMPUTE | the archive's `system/sharding/config` | ⛔ **specified three times, never brought forward** |
| bounded fan-out query | the archive's `QUERY {expand, ttl, seen_peers}` | ⛔ **specified twice; `EXTENSION-QUERY` §1.2 defers it as Level 3** |
| ⛔ **a digest OF `serve_scope`, and its reconciliation** | **nowhere** | ⛔ **this is the missing piece, and it is the whole proposal** |

> ⭐ **Read that column: one row is empty.** The honest novelty of this synthesis is a single
> composition — *the published set already has a definition, a capability evaluator, and a refresh
> cadence; give it a digest and reconcile it.* **Everything else is assembly, and two of the missing
> rows are documents this project already wrote and left in the archive.**

---

## §5 Where it lands

**On `DATA-EXCHANGE`, not beside it.** That proposal already owns the loop — SUBJECT, SOURCE, WITNESS,
POSITION, INTENT — and its spine is *the question is answerable by comparison*, which is this
synthesis's spine too.

**Three specific joins:**

1. ⭐ **A digest is a WITNESS over a set rather than over a subject.** §6.3's witness is *"opaque"* and
   *"currency is comparison"* — a reconciled digest is the same operation with the set as its subject.
   **§11's `Q6` — *are witnesses one operation with a mode, or several?* — is the question this lands
   inside**, and this is evidence for *several*.
2. ⭐ **`absent` vs `refused` is the discrimination a digest must preserve.** A digest that includes
   things the asker would be `refused` re-creates ICP's false hit; one that excludes them is
   `serve_scope`-filtered by construction. **The taxonomy already demands the distinction; the digest
   is how it is honoured in advance rather than per-request.**
3. **Closure applies unchanged.** A reconciled digest is content in your own namespace, signed, and
   consumable by the next peer with no new type — `gathered → gathered`.

---

## §6 What this does not solve

**Named plainly, because a synthesis that answers everything is not to be trusted.**

1. ⛔ **Cold start for a stranger's content where no reference carried a locator.** If you hold a bare
   hash, have met nobody who serves it, and nobody you reconcile with has it, **there is no answer
   here.** *The corpus's position is that this case is rarer than it sounds — a reference names its
   publisher, and citations introduce peers — but that is an argument, not a mechanism.*
2. ⛔ **Global content-anchored search.** Already ruled out on record (`F-24`): *"what contains X"* has
   no anchor and is a whole-corpus query by the nature of the question. **We should keep saying we do
   not offer it.**
3. ⚠ **The interest leak** (§3.3). Narrowed by reconciling only with peers you chose; not closed.
4. ⚠ **Erasure coding** (`LK-28`, `LK-39`). A digest over whole objects cannot express *3 of 10
   shares*. Freenet's resolution — code into blocks, then content-address each share — works, but the
   digest is then over shares and the K-of-N assembly is the reader's.
5. ⛔ **Abuse and resource exhaustion.** ***The named next axis***, and the survey says it is where the
   real difficulty lives — CoralCDN's retrospective puts most of its *what went wrong* material there,
   and OpenDHT designed its storage model around it. **`DATA-EXCHANGE` §11's `Q2` is already an
   instance:** *a publisher who cannot serve live has no way to stop every reader trying, and
   experiences the failure as load they cannot attribute.* **A reconciliation loop is a new thing
   strangers can make you do work for, and this synthesis has not costed it.**

---

## §7 What would falsify this

- **The false-hit claim is testable by construction**: build a digest from `serve_scope`, ask for
  something outside it, and check the answer is `absent`-or-`refused` **consistently with what the
  digest said**. If the digest and the evaluator can disagree, §3.1 is wrong.
- **The reconciliation claim is testable on the existing trie**: reconcile two peers' served sets by
  subtree fingerprint. **If range summarization over the served set is not cheap on the structure we
  already maintain, the whole assembly loses its economics.**
- **The privacy claim is falsifiable by scope**: if a `published-set` digest reveals anything a
  hash-guesser could not already obtain, §3.3 is wrong.
- ⭐ **The strongest falsifier is the abuse axis**: if an unauthenticated stranger can make a peer do
  unbounded reconciliation work, this design has traded a lookup problem for a denial-of-service
  problem — **and §6.5 is why that review comes next rather than later.**

---

## §8 Where this leaves the arc

**Not a proposal yet, and the reason is §6.5.** The assembly is coherent, its pieces are nearly all
landed, and the one missing row is small. **But the survey's own evidence says the mechanism is rarely
what kills these systems** — three independent instances now — **and the thing that does kill them is
the axis we have not reviewed.**

**So the order is: abuse and resource exhaustion first, then a proposal.** If the loop survives that
review it is worth specifying; if it does not, better to learn it now than in a fold.

**And two archive documents should be brought forward regardless of the outcome**, because they are
already written and are load-bearing for two of the four mechanisms: the **sharding config as the
agreed map**, and the **bounded fan-out query**.

---

## §9 Cross-references

`EXTENSION-NETWORK` §6.5.6 (`serve_scope`, the capability token, the convergence bound, Amendment 10's
signed-root closure) · §6.5.1c (`transport-set`) · `EXTENSION-QUERY` §2.2 (reverse hash index), §1.2
(the deferred cross-peer handler) · `EXTENSION-TREE` §3.3a, §3.7.1 · `APP-CONVENTION-REFERENCE` §1,
§2.3 · `EXTENSION-ROUTE` §3, §4 · `PROPOSAL-KEEPING-A-COPY-CURRENT…` (`DATA-EXCHANGE`) §5.0, §6.2,
§6.2.1, §6.3, §11 · `PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` ·
`EXPLORATION-THE-CONVERGENT-DESIGNS-WILLOW-SSB-USENET-MATRIX-AND-THE-FIVE-VERDICTS` §7 · the five
survey explorations named at the head · register rows `LK-1`…`LK-51`, `F-13`, `F-24`, `F-32`, `V-4`.
