# REVIEW — the published locator table, attacked seven ways, and what survives

**Status:** REVIEW (2026-09-13). **Adversarial pass on this arc's own conclusion**, written because the
conclusion arrived easily and the five documents before it were each wrong in the *pessimistic*
direction. **The risk now runs the other way.**

**Reviews:** `A-TABLE-YOU-OWN-IS-NOT-KADEMLIA` and register rows `LK-69`…`LK-73`.

---

## §0 Verdict

**The design survives. Four of seven attacks fail outright, two force a named cost, and one is real,
unsolved here, and is the abuse axis this arc has deferred four times.**

| # | attack | outcome |
|---|---|---|
| 1 | *self-healing is hand-waving — nobody prunes a dead peer's table* | ⚠ **refined** — the mechanism is not the one claimed, and the real one works |
| 2 | *aggregator unions become a monotonically growing graveyard* | ⚠ **real, and answered by machinery already in the spec** |
| 3 | *it gives up Kademlia's guarantee* | ✅ **survives** — and the guarantee is not kept by the system that offers it |
| 4 | ⛔ **poisoned tables are a distributed amplification weapon** | ⛔ ⭐ **LANDS. Unsolved here. This is the review's finding** |
| 5 | *a table is a full inventory disclosure* | ⚠ **real, and granularity is the answer — arriving from a third direction** |
| 6 | *you still have to find the tables* | ✅ **survives** — identical to registry bootstrap, not a new problem |
| 7 | *it does not actually compose with the data layer* | ✅ **survives** — it composes exactly, and that is the strongest part |

---

## §1 Attack 1 — "self-healing" is not the mechanism that was claimed

**The claim:** a peer goes offline, everyone prunes it, the network converges.

⛔ **The asymmetry that breaks the naive version: you can prune entries you READ, but you cannot prune a
table you AUTHORED once you are gone.** A dead peer's table keeps being served — by whatever origin
holds it, by mirrors, by aggregators that unioned it — and it keeps pointing at things that may also be
dead. **The author is exactly the party who cannot fix it.**

✅ **But the real mechanism is second-order and it does work: readers prune SOURCES, not just entries.**
A table that keeps producing failures stops being consulted. Its entries do not have to be individually
retracted — **the whole source falls out of readers' precedence lists**, which is `EXTENSION-REGISTRY`
§4.1's dispatch list doing what it already does.

⇒ **Convergence is real but it is *reader-side selection*, not *author-side correction*.** That
distinction matters when writing the spec: **there is no retraction obligation, and there must not
be one**, because the party who owes it is unreachable by definition.

---

## §2 Attack 2 — the graveyard

**The attack:** an aggregator unions many tables; closure means aggregators read aggregators; dead
entries circulate indefinitely and the union grows monotonically. **That is OpenDHT's orphaned-data
problem arriving by a different road**, and it is the thing that killed a whole design's storage model.

⚠ **It lands against a naive version, and the corpus already carries the answer:**

- ⭐ **`ResolutionResult` already has `ttl` and `neg_ttl`** (§2.1) — positive and **negative** caching
  hints. **A locator claim is soft state by construction if it carries one.**
- ⭐ **The aggregator does not re-sign** (§8.2) — every entry stays attributable to its original issuer,
  so **age and origin are both inspectable** and a reader can apply policy to either.
- **`LK-48`'s survey finding applies unchanged:** every deployed shared index makes the holder
  re-assert. **The table model should too.**

⇒ **Requirement, not a nicety: a locator claim expires. An aggregator that unions without expiry is
building the thing OpenDHT refused to build.**

---

## §3 Attack 3 — you gave up the guarantee

**The attack, and it is the strongest theoretical one.** Kademlia offers: *any node finds any published
key in `O(log n)` hops with no prior relationship.* **The table model offers nothing of the kind** —
coverage is bounded by your reachable table graph. If no table you can reach carries `H`, you do not
find it, **even though somebody has it.**

✅ **It survives, on measured grounds rather than preference:**

- **The guarantee is not kept.** >70% of provider records unreachable is the measurement; the guarantee
  is about records that are *live*, and most are not. **An unkept guarantee is worse than an honest
  absence, because systems get designed against it.**
- **It is a guarantee about a property nobody could verify anyway** — `L-8`'s law: nothing makes a union
  complete, and no `complete` field will ever exist.
- **The trade is now statable:** *Kademlia offers universal reach it does not deliver; a table graph
  offers bounded reach that works.* ⇒ **state it as a limit in the spec, exactly as `LK-65` said the
  bare-hash refusal should be stated — because an unstated limit gets re-opened.**

---

## §4 ⛔ Attack 4 — the poisoned table is an amplification weapon. **This one lands.**

**Anyone may publish a table.** So anyone may publish a table asserting that **one victim peer holds a
million hashes.** Readers consult it. **Readers fetch. The victim pays.**

⛔ **This is not the false-hit problem and it is worse than it.** A false hit costs *the asker* one
wasted fetch — the cost this arc has repeatedly called cheap. **A poisoned table costs THE VICTIM,
multiplied by every reader who believed it, and the attacker pays almost nothing.** ⭐ **The property
that makes the design cheap — *a wrong entry only costs one fetch* — is stated from the asker's side,
and the attacker's leverage is on the other side of it.**

**Four mitigations exist in the surveyed field, and none is free:**

| mitigation | source | cost |
|---|---|---|
| ⭐ **probe before recording** — verify possession before listing | **iroh: downloads a random 2 KiB BLAKE3 chunk before it will store an announce**, so *"the worst an attacker could do is force futile connection attempts"* | the aggregator pays a probe per entry |
| **trust levels on sources** | **git-annex**: `trusted` / `semitrusted` / `untrusted` / `dead` | a human judgement, per source |
| **reader-side precedence and pinning** | `EXTENSION-REGISTRY` §4.1a — already the consumer's config | does nothing against a source you *do* trust |
| **rate limiting at the victim** | universal | the victim still pays, just less |

⭐ **iroh's is the strongest and it is the one that fits: content addressing makes possession
cheaply provable, so a claim can be VERIFIED rather than trusted.** *This is exactly the property
`LK-13` identified and the synthesis then set aside as "no longer load-bearing" — it is load-bearing
again, for a different reason: not staleness, but abuse.*

⛔ **What is genuinely unresolved:** a probe proves possession **at probe time**, to **the prober**. It
does not prove the victim consented to serve *your* readers, and a large aggregator probing everything
is itself a load source. **This is the abuse axis, it has been deferred four times in this arc, and it
is now the blocking item rather than a later one.**

---

## §5 Attack 5 — a table is an enumeration

**`LK-55` argued a digest leaks nothing new because, within `serve_scope`, hash-knowledge is already
the read authority — a digest only removes the need to guess.** ⚠ **A published table is a stronger
object than that argument covers: it is an explicit, portable, third-party-republishable enumeration of
what a peer holds.** *Removing the need to guess is exactly what an inventory disclosure is.*

✅ **The answer is granularity, and it arrives here from a third independent direction:**

> **A table entry does not have to name a hash. It can name a NAMESPACE** — *"for anything under this
> peer's published content, ask me"* — **which is one entry instead of a million, discloses nothing
> beyond what a published root already discloses, and is what makes an aggregator's table tractable.**

⭐ **`LK-15` reached per-namespace granularity from NDN's aggregation limit; `LK-9` reached it from leaf
count; this reaches it from privacy and from abuse surface at once.** **Three independent derivations
of the same shape is the strongest structural signal in the arc.**

⇒ **Per-hash entries are for the sharp case; per-namespace entries are the default.**

---

## §6 Attack 6 — you still have to find the tables

✅ **Survives, and it is not a new problem.** A table is found the way a registry backend is found:
**configured in your resolver-config, introduced by a citation, or unioned by an aggregator you already
consult.** `F-40a` already settled the general form — *anyone can be a registry, and it needs nothing
built.*

**What is genuinely inherited: cold start.** A peer with no configured sources and no citations has
nothing — **which is the same cold-start the system already has for names, and is not made worse.**

---

## §7 Attack 7 — does it really compose with the data layer?

✅ **It composes exactly, and this is the strongest part of the design.**

| requirement | satisfied by |
|---|---|
| a locator table is an entity | ordinary signed entity in your own namespace |
| it is kept current | `DATA-EXCHANGE`'s copy-current loop — **no new sync** |
| an aggregator's output is consumable by the next aggregator | ⭐ `DATA-EXCHANGE` §5.0 closure: **`gathered → gathered`** |
| conflicting answers surface rather than being picked | `EXTENSION-REGISTRY` §8.3 |
| the reader decides policy | §4.1 precedence; §8.3 consumer policy |
| an extension interprets it | ordinary handler dispatch |

⇒ **No new transport, no new sync, no new discovery, no new trust machinery.** The locator table is
*another data object the exchange layer already knows how to move*, and the navigation logic is an
extension reading it. **That is the whole claim, and it holds.**

---

## §8 What is worth keeping from the systems this arc has been rude about

**They got the topology wrong. Several of the parts are good and are directly reusable:**

| keep | from | for |
|---|---|---|
| ⭐ **probe before recording** (2 KiB) | iroh | §4 — the only real defence against poisoning |
| ⭐ **the hash as key, with no semantic interpretation needed** | Kademlia | the actual good idea: you can route on it without knowing what it is |
| **trust levels over sources** | git-annex | reader policy with a dimension resolver-config lacks |
| **`only-if-cached`** | Squid | a one-header way to make a false positive cheap |
| **digest deltas** | Squid | publish the diff, not the table |
| **reconcile the difference** | Erlay / RBSR | how two aggregators sync tables at `O(difference)` |
| **level-2 → level-1 → global** | CoralCDN | the precedence walk, which is resolver-config |
| **hinted handoff** | Dynamo | *"I have it, it is not mine, it belongs to X"* — temporary custody, which a locator entry cannot otherwise express |
| **blinded lookup index** | Tor v3 | held in reserve for a private variant (`LK-22`) |

---

## §9 What is owed, in order

1. ⛔ ⭐ **The abuse review** (§4). **Now blocking.** The amplification attack is cheap for the attacker
   and expensive for a third party, and the arc's *"a wrong entry costs one fetch"* argument is stated
   from the wrong side of the transaction.
2. **The locator-claim type** — `hash | namespace → [where]`, signed, **expiring** (§2), attributable to
   its issuer, **per-namespace by default** (§5).
3. **The resolution-contract delta** — §4.1.1's single-binding rule against a key that wants the union.
4. **Registry federation's normative text** — §8.2, *undesigned, not blocked*.
5. **Two limits written as limits, not questions** — the bare-hash refusal (`LK-65`) and bounded reach
   (§3). **Both have been re-opened repeatedly because they are filed as gaps.**

---

## §10 What would falsify this review

- **§1 is falsifiable operationally:** if readers do not in practice drop failing sources, convergence
  is a story. **Nobody has run it.**
- **§4 is falsifiable by cost:** if probe-before-record is cheap enough at aggregator scale, the attack
  is mitigated and item 1 unblocks. **That is a measurement and it has not been taken.**
- **§5's granularity claim is falsifiable by use:** if real workloads need per-hash entries at volume,
  the privacy and abuse surfaces both return.
- ⭐ **The whole design is falsifiable by one deployment**: two aggregators, one poisoned table, and a
  victim. **If the victim's load is bounded without trusting anybody, this is right.**
