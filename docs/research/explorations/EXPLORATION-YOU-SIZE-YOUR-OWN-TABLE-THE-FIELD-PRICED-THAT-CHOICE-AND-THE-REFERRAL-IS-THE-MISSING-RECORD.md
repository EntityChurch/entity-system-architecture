# You size your own table — the field priced that choice, it is cheap, and the referral is the record we do not have

**Status:** EXPLORATION (2026-09-13). **Continues the locator-table landscape review on an axis no
previous document in this arc touched: *a table whose size is a local resource decision, paid for in
hops.*** It also closes the four systems the preceding document named as unread.

---

## §0 The findings

1. ⭐⭐⭐ **The state/hops trade is priced, and the price is `1 / log s`.** Holding a table of size `s`
   costs `O(log n · log log n / log s)` hops. ***A hundredfold smaller table costs roughly 1.5–2× the
   hops*** (the ratio is `log s₁ / log s₂`, with the network size held fixed). A small device that
   holds 1% of what an aggregator holds is not crippled — it is paying a small constant factor.
2. ⭐⭐⭐ **The curve's shape forces the topology: a few large tables and many small ones.** Three
   independent arrivals — a self-tuning overlay biasing its traffic *toward better-resourced
   neighbours*, a deployed file-sharing network that split into ultrapeers and leaves, and this
   design's aggregator. **The aggregator is not a convenience; it is where the curve puts you.**
3. ⭐⭐ **There is a second compression axis nobody here has used: FIDELITY, not just granularity.** A
   published table may be *probabilistic* — a keyword-hash bit table with admitted false positives —
   and that was the mechanism that made a flooding network scale. It is safe here for exactly the
   reason `LK-80` gives: a false positive costs one wasted fetch, and the content is verified on
   receipt.
4. ⭐⭐⭐ **The record that makes "more hops the less I hold" a design rather than a wish is the
   REFERRAL — *for this key, ask that table* — and this corpus has already identified it as the
   named missing mechanism**, deferred for a reason that does not apply to a hash-keyed backend.
   ⇒ **A locator entry and a referral are the same record; only the kind of thing "where" names
   differs.**
5. ⛔⭐⭐ **`LK-78`'s amplification attack has a deployed instance with a CVE.** OpenPGP's SKS
   keyserver network, 2019. **Same three preconditions, same outcome, and the fixes the ecosystem
   reached for are the three on our agenda** — plus one we did not have: **publish a claim with its
   CLAIMANT, never attached to its SUBJECT.**
6. ⭐⭐ **Quorum across independently-chosen sources is a mitigation this arc never named**, deployed
   twice. ⚠ **For locators it must be an ORDERING rule, never an admission rule** — a rare hash is
   known to one source by definition.
7. ⚠ **Retractions do not propagate through merged tables — measured, in a live federation.** The
   deployed workaround is a *separate retraction feed*, which is `LK-74a`'s asymmetry showing up as
   operational debt rather than as theory.

---

## §1 What was searched, and where

**Named so the negative is reviewable.** Three regions: this corpus (`specs/`, `guides/`, the
proposal and research workspace), the pre-split archive (1,068 documents), and the paper corpus.

| Term | Hits across all three regions |
|---|---|
| `accordion` · `kelips` · `epichord` · `beehive` · `one-hop DHT` | ⛔ **zero** |
| `ultrapeer` · `query routing protocol` · `QRP` | **2 documents, both this arc, both saying only *"Gnutella flooded and died"*** |
| `routing table size` / `hop count` as a *tradeoff* | **zero on this axis** — every hit is continuation bounds or relay store bounds |

⇒ **The state/diameter literature has never been read here, and the one deployed system that solved
the heterogeneous-capacity version of this problem is in the corpus only as a failure.** That is the
gap this document closes.

---

## §2 The curve, and the number that matters

**The question is *"if I hold less, how much more do I pay?"* and it has a published answer.**

> **`O(log n · log log n / log s)` lookup hops, where `s` is the routing table size** — Accordion
> (Li, Stribling, Morris, Kaashoek, NSDI 2005), §3.1, derived from an extension of Kleinberg's
> small-world analysis.

**Read the denominator.** Hops fall with the **logarithm** of table size, which cuts both ways and the
favourable direction is ours:

| you hold | relative hops |
|---|---|
| 1,000,000 entries | 1.0 |
| 10,000 entries | **1.5×** |
| 100 entries | **3.0×** |

***A ten-thousandfold reduction in state costs a factor of three.*** **This is the strongest possible
result for a device that wants a small table**: the penalty for holding almost nothing is small and
bounded, and the reward for holding everything is correspondingly poor. **Nobody should build a large
table for latency; they should build one to serve others.**

### §2.1 Accordion is the design, already built

**It has a single user-facing parameter and it is not a size — it is a budget.** *"Accordion has a
single parameter, a network bandwidth budget, that allows control over the consumption of the
resource that is most constrained for typical users."* The table size is **not calculated**; it is
*"the result of an equilibrium between two processes: state acquisition and state eviction"* — you
learn as fast as your budget allows and evict what looks dead, and **where those two rates cross is
your table size.**

⭐ **The design lesson is the parameterization, not the algorithm.** A user cannot answer *"how many
entries should I hold?"* — they can answer *"how much bandwidth may this cost me?"* and *"how much
disk?"*. **Expose the budget; let the size be an outcome.**

⭐⭐ **And it addresses heterogeneity explicitly, with a rule we should copy verbatim:**

> *"The imbalance between a node's specified budget and its actual incoming and outgoing traffic is of
> special concern in scenarios where nodes have heterogeneous budgets in the system. To help nodes
> with low budgets avoid excessive incoming traffic from nodes with high budgets, an Accordion node
> biases lookup and table exploration traffic toward neighbours with higher budgets."*

***Route toward the better-resourced party.*** That single sentence is the ultrapeer rule, derived
independently, inside a flat overlay — and it is what stops a small device being crushed by being
useful.

### §2.2 ⚠ What transfers and what does not — stated carefully, because this arc got it wrong once

**This arc's central error was reporting a measurement of one implementation as a property of the
architecture.** So, explicitly:

| | transfers | does not |
|---|---|---|
| **the shape** — state and hops trade logarithmically, so small tables are cheap | ✅ | |
| **the parameterization** — budget in, size out | ✅ | |
| **bias traffic toward better-resourced peers** | ✅ | |
| **the bound itself** | | ⛔ **it is a theorem about greedy descent in an ID space**, and a published-table graph has no metric, so **no hop is guaranteed to make progress** |

⛔ **Without a metric there is no walk.** This is `LK-76` restated from the other side: the table model
has bounded reach because there is nothing that says consulting one more table gets you closer.
**§6 is where that is repaired, and it is repaired by a record rather than by a metric.**

### §2.3 The bound below the bound, and the aggregator is standing on it

**"On the Fundamental Tradeoffs Between Routing Table Size and Network Diameter in Peer-to-Peer
Networks"** (Xu, Kumar, Yu — INFOCOM 2003; extended in JSAC, Nov 2003) asks whether the observed
trade is the best achievable. Its answer, in the form that matters here:

- existing schemes sit at either **`O(log n)` table with `O(log n)` diameter**, or **`d` table with
  `O(n^(1/d))` diameter**;
- **better trades are achievable** — and **all of the algorithms that achieve them concentrate load
  on particular nodes.** The authors formalize this as *congestion*, and the standard trade is
  asymptotically optimal once you require it away.

⭐⭐ ***Our aggregator is that concentration, and the theorem says it is not a shortcut but a
purchase.*** A topology of many tiny tables plus a few enormous ones **buys near-constant lookup with
tiny per-node state, and pays for it in load on the few** — which is the same currency `LK-78`'s
amplification attack is denominated in, and the same failure the design exists to avoid at the
extreme (a single global index). **The aggregator is therefore a dial, not a component: how much
concentration are we buying?** That question is now askable and was not before.

> ⚠ **Honest limit on this citation.** The primary PDF was not reachable from here (HTTP 418 at the
> conference mirror, 404 at the author's page). The result above is stated at the level that **two
> independent secondary restatements agree on**; the exact theorem statement, and a later refinement
> holding that the conjecture as literally stated fails but holds for *uniform* algorithms, are
> **read but not verified at the source**. Do not quote a bound from this paragraph.

---

## §3 The deployed instance: ultrapeers and leaves

**Gnutella appears in this corpus exactly twice, in one sentence, as the network that flooded and
died. That is the first half of its history.** The second half is the repair, and it is this design.

**The Query Routing Protocol.** A leaf hashes the keywords of what it holds into a bit table and
**sends that table to its ultrapeer.** The ultrapeer keeps **one table per leaf**, and forwards a query
to a leaf only on a hit. *"This is done without even knowing the resource names."*

**Every structural element of our design is there:**

| our design | QRP |
|---|---|
| you publish a table of what you hold | the leaf uploads its query routing table |
| an aggregator unions many tables | the ultrapeer holds one per leaf and checks them all |
| a small device holds less and relies on a bigger one | **leaf mode vs ultrapeer mode, assigned by capacity** |
| a table is kept current by copying the current version | ⭐ **`RESET` + `PATCH` — the delta, not the table.** *"To prepare a patch, the sender first subtracts the last route table sent from the current route table"* |
| a wrong entry costs one wasted fetch | ⭐ *"does not eliminate false positives… one can reasonably assume that they are rare"* — **harmless false positives** |

⭐ **`RESET`/`PATCH` is the second independent arrival at *publish the difference, not the table***
(the first was digest deltas in the web-cache survey, `LK-79`). **Two deployed systems, same
conclusion, and it is the mechanism that makes a large table affordable to keep current.**

⭐ **"Last-hop QRP" is worth stealing outright.** The tables were exchanged only where they paid best —
on the final hop, where a negative match *proves* nothing further can match. **Apply the expensive
mechanism at the boundary where its answer is conclusive**, not uniformly.

---

## §4 The compression axis this arc has not used: fidelity

**`LK-77` answered *"a published table is too big and discloses too much"* with GRANULARITY — a table
entry names a namespace, not a hash. That is one axis. QRP demonstrates a second, orthogonal one:**

> ⭐⭐ **A table entry need not be EXACT. A lossy, probabilistic summary of what you hold — with
> admitted false positives and no false negatives — is a legitimate published table.**

**Why it is safe here, and the argument is already ruled:** a false positive resolves to a fetch that
returns nothing or returns bytes that fail their hash. **`LK-80` — verification decides what you
accept — makes an approximate locator table exactly as safe as an exact one, and strictly cheaper.**

**Two axes, composable, and they answer different complaints:**

| complaint | axis | mechanism |
|---|---|---|
| my table is too big | **granularity** | one entry per namespace (`LK-77`) |
| my table is still too big | **fidelity** | a probabilistic summary, false positives admitted |
| publishing my table is an inventory disclosure | **both** | a namespace entry discloses nothing beyond a published root; **a lossy summary cannot be enumerated back into an inventory** |

⭐ **The third row is new and is the strongest argument for the fidelity axis.** `LK-77` conceded that a
published table is *"an explicit, portable, third-party-republishable enumeration of what a peer
holds."* **A probabilistic table is not an enumeration.** It answers *"do you have this?"* for a hash
you already possess, and **cannot be read out as a list** — which is the privacy property `LK-55` and
`LK-77` were both reaching for.

**Where the corner cases live:** the keyserver survey below shows what happens when anyone may append
to a shared structure; a lossy table has no per-entry attribution, so **a probabilistic table is
publishable by its holder only, never merged from third parties.** That is a real constraint and it
is the natural one: *you may summarize what you hold; you may not summarize what someone else holds.*

---

## §5 Choosing how much to hold, as shipped

**Two systems already ship the config file this design needs.**

### §5.1 A social graph with a hop dial

Scuttlebutt replicates by **distance in the follow graph**, and the distance is a local setting:
`friends.hops` (default **3**) and `friends.dunbar` (default **150** feeds). **The user picks the
radius; the radius determines the size.**

⭐⭐ **And the refinement is the part worth taking: fidelity is configured PER HOP.** The replication
scheduler is keyed by hops level — *hops 0: full; hops 1: a handful of index feeds; hops 2 and beyond:
two index feeds.* **Depth 0 is exact, and fidelity degrades with distance.**

⇒ ***"how much I hold locally, how stale I am willing to be, and how many hops I pay"* is not three
settings; it is one radius and a per-radius fidelity template.** That is the concrete shape of the
local resource decision, deployed, with a default.

⭐ **It also separates the two questions cleanly, and the separation is `LK-80`'s:** the follow graph
and the hop count decide **who** you replicate — *where you look* — and the scheduler template decides
**how much of them** you take. **Two mechanisms, one job each.**

### §5.2 A package manager that already runs the trust/verification split

Nix holds the two halves in **two separate settings**:

- **`substituters`** — an **ordered** list of places to look, *tried in the order given*. **That is
  trust, and it is the precedence dispatch list.**
- **`trusted-public-keys`** — what you will **accept**, enforced by `require-sigs` (on by default).

⭐⭐ **And the reason the second list exists is the finding.** Nix store paths are **input-addressed**,
so *"there is no way to verify that the store path contents you download from a substituter were
actually produced by the same Nix expressions you used when calculating the store path hash."*
**The key list exists to patch a missing content-address.** Nix's own stated future direction —
content-addressed derivations — *"aims to solve the trust issue without depending on
`trusted-public-keys` for signature verification."*

⇒ ***A deployed package manager is migrating toward the property this design has by construction.***
`EXTENSION-SUBSTITUTE`'s model was adopted from Nix; **the half Nix is trying to shed is the half we
never needed.**

⚠ **Two costs of the ordered-trusted-list model, from the same source, and they are ours too:** key
**rotation and revocation are hard** — every consumer holds the list in local config, so a rotation
requires all of them to update and *"does nothing about artifacts already in the store"* — and users
are actively asking for **non-transitive source trust** (*trusting substituters, but not theirs*).
**Our precedence lists inherit both.**

---

## §6 ⭐⭐⭐ The missing record: the referral

**§2.2 left the model without a walk: consulting one more table is not guaranteed to make progress,
so "more hops the less I hold" has no mechanism behind it.** Here is the mechanism, and it is one
record.

> ⭐ **A locator entry says `key → where the bytes are`. A REFERRAL says `key-range → where to ask
> next`. They are the same record; only the kind of thing "where" names differs.**

**With referrals the walk is bounded by the NAMESPACE HIERARCHY rather than by a distance metric** —
which is how the one deployed global lookup that has never been replaced works, in three hops, with no
metric anywhere. **A small table holds referrals; a large table holds answers.** That is precisely
*"more hops the less I hold"*, and it is the missing half of §2.

### §6.1 This corpus already found the hole, from the other direction

**`EXPLORATION-NAMING-LANDSCAPE-AND-THE-DNS-MAPPING` §5.2 names it exactly:** the resolver chain is a
client-side ordered list, and *"there is no way for a registry to say **names under `*.acme` are
answered by that registry**."* ⭐ **With the consequence already measured:** *"today that is configured
per-consumer, so it scales with the number of consumers rather than the number of zones."*

⇒ ***Without a referral record, every consumer must be configured with every table it might ever
need.*** **That is the actual reason a small device cannot hold a small table today** — not storage,
*configuration*. The device's table is small but its **config** must be complete.

### §6.2 Why the deferral does not bind the hash-keyed backend

**§5.2 defers delegation on a good argument: a hierarchy is also a central point of political failure,
and every surveyed post-hierarchy system declined to rebuild one.** ✅ **That argument is about a
MANDATORY ROOTED hierarchy with a single authority at the top. A referral in a locator table is
neither:**

- **no root** — any table may carry a referral; there is no apex anyone must consult;
- **not authoritative** — two tables may refer you to different places, and §8.3 already says surface
  both rather than silently picking;
- **not binding** — a referral is a hint about *where to look*, so **`LK-80` covers it completely: a
  wrong referral costs one wasted round trip and can cost nothing else.**

⇒ **The objection is to hierarchy-as-authority. A referral is hierarchy-as-hint, and the design
already has the machinery that makes hints safe.**

### §6.3 ⭐ What bounds the walk — and it retires an inventory item with a reason

⚠ **Nothing forces a referral to narrow.** `A → B → A` is expressible, and an adversary will write it.
**A referral chain therefore needs a hop budget and a loop breaker**, which is exactly the shape of
the bounded fan-out query specified at two core revisions and still sitting outside the corpus:
a hop budget plus a set of already-visited parties, with progressive partial results.

⭐ **That artifact was already flagged as worth bringing forward on inventory grounds. It now has a
requirement behind it:** referral walking does not work without it. **A mechanism with a caller is a
different item from a mechanism that is merely written down.**

### §6.4 A fourth derivation of per-namespace granularity

**Three independent derivations already reached *per-namespace by default*: an aggregation limit, leaf
count, and the privacy-plus-abuse surface.** ⭐⭐ **This is a fourth, and it is the one that explains
why the walk terminates:** *a per-hash table cannot refer — a hash is a uniformly random number and
carries no range to delegate. A per-namespace table is a delegation record.* **Granularity is not only
a size optimization; it is the thing that makes referral expressible at all.**

---

## §7 The abuse review, continued — `LK-78` has a CVE

**The agenda asked whether anyone has run the poisoned-table attack. Someone has, it has a number, and
the ecosystem's response is the most useful thing in this document.**

**CVE-2019-13050 — certificate flooding, SKS keyserver network, June 2019.** Hundreds of thousands of
signatures were appended to two well-known contributors' OpenPGP certificates. **The victims were not
the attackers' counterparties; they were third parties who had done nothing.** Importing a poisoned
certificate pulled **tens of megabytes instead of tens of kilobytes** and rendered the importing
client unusable — a denial of service against **every reader who consulted the shared table**, and
against the subject's usability permanently.

⭐⭐ **The three preconditions are ours, item for item:**

| SKS | this design |
|---|---|
| a key may carry unlimited signatures | **a table may carry unlimited entries** |
| **anyone may append a claim about anyone else's key** | **anyone may publish a claim about anyone else's holdings** |
| **no way to distinguish a legitimate claim from garbage** | ⚠ **for a locator, a claim is checkable — but only by paying the fetch, which is the damage** |
| write-only: never deleted, even after revocation | ⚠ **`LK-74a`: you cannot prune a table you authored once others have unioned it** |

⛔ **And the sentence that should end any argument for deferring the circuit breaker:** *the problem of
certificate poisoning and subsequent flooding had been known for years, with proof-of-concept attacks
and dire warnings, before anyone ran it.* **An architectural flaw with a known exploit path is not a
someday problem, and this one was declared unfixable in a backwards-compatible way once deployed.**

### §7.1 The fix we did not have: publish with the CLAIMANT, never attached to the SUBJECT

**Three responses were reached for, and two are already on our agenda.** The third is new and is
structural:

1. **Strip third-party claims entirely.** The replacement keyserver distributes no third-party
   signatures at all. ⛔ **Too strong for us** — a third party publishing *"I also hold this hash"* is
   the mechanism that dissolved `LK-67`'s permanent limit. **We cannot take this one.**
2. ⭐ **Consent: the subject attests.** The replacement publishes a third-party certification only if
   *the key holder explicitly marked it as attested.* ⭐⭐ **This is the answer to `LK-78`'s stated
   unresolved half** — *a probe proves possession at probe time, not that the victim consented to
   serve your readers.* **A consent marker proves consent. It is the one mechanism that does.**
3. ⭐⭐⭐ **Distribute the claim with its CLAIMANT rather than its SUBJECT** — the proposal to
   distribute signatures *with the signer rather than the signee*. **This is a pure structural fix:
   the flood lands in the liar's own object, which they pay to publish and readers choose to fetch;
   the victim's object does not grow at all.**

⭐⭐⭐ ***And the design already has property 3 by construction: a locator claim lives in the issuer's
own table, in their own namespace, signed by them, and an aggregator does not re-sign.*** **A
poisoner's million claims inflate the poisoner's table.** ⛔ **The exposure is precisely at the point
where that stops being true: an aggregator that unions into a HASH-KEYED index has re-created SKS's
shape — a shared structure, keyed by subject, that anyone may append to.**

⇒ **The rule this yields, and it is sharp enough to be normative:**

> ⭐⭐ **An aggregator bounds contribution PER ISSUER, never per key.** A per-key bound is
> unimplementable against a flood (the attacker picks new keys) and punishes the popular hash; a
> per-issuer bound is the circuit breaker already on the agenda, and it is what keeps the cost of
> lying attached to the liar.

⭐ **Two mechanisms, from two fields, arriving at the same rule**: the volume circuit breaker (drop a
peer that suddenly announces 10× its usual volume) is a *per-peer* bound, and the keyserver lesson is
a *per-issuer* attribution. **Same rule, and the convergence is the evidence that it is the right one.**

---

## §8 Quorum — a mitigation this arc never named, deployed twice

**Two independent systems make an assertion actionable only when several independently-chosen sources
agree:**

- **Federation blocklists.** One curated list *requires a domain to be blocked by a minimum number of
  additional reference servers — currently 6 of 7* before including it. The tooling exists to *"pull
  in a list of blocklists from a set of trusted sources, merge them into a combined blocklist"*:
  **published tables, sources chosen by the reader, merged locally.** This is our design, running, in
  a federation of real servers.
- **Nix.** `nix store verify --sigs-needed n` requires a path to be signed by **at least *n* different
  keys.** Proposals in flight extend it to *threshold rules requiring at least N signatures within a
  group.*

⚠ ⭐ **But quorum transfers only as an ORDERING rule, and getting this backwards would break the
design.** A blocklist is a *deny* decision, where requiring corroboration is conservative. **A locator
is a *find* decision, and a rare hash is known to exactly one source by definition** — requiring 6 of 7
would make the mechanism useless precisely where it is needed. ⇒ ***Corroborated entries are tried
FIRST; uncorroborated entries are tried LAST; nothing is excluded.*** **Cost: zero. It is a sort
order.**

⭐ **The same field also supplies the softer form:** blocklist entries carry a **severity** —
`noop` / `silence` / `suspend` — *so a low-confidence entry can be limited rather than fully acted
on.* **A low-confidence locator entry is tried last, not dropped.** Same idea, and it is the general
answer to *"what do I do with a hint I half-believe?"* — **degrade its priority, never its
admissibility.**

---

## §9 ⚠ Retractions do not propagate — measured, in production

**`LK-74a` reasoned that a table's author cannot prune what others have already unioned, and concluded
that convergence is reader-side selection. Federation blocklists are the live instance, and they show
what that costs in practice:**

> *"Regularly importing a list won't account for retractions"* — so the ecosystem maintains **a
> separate blocklist of retractions** for subscribers to also import.

⛔ **The consequence, stated plainly by the maintainers: a wrongly-added entry can stay in effect
indefinitely unless the retraction mechanism is separately wired up.** **`LK-74a`'s reader-side
selection is correct and it is not sufficient** — a reader drops a source that *keeps* producing
failures, which never fires for a source that is right about everything except you.

⇒ **Two things follow, and both are cheap:**
- ⭐ **Expiry is load-bearing and already available** (`ttl` / `neg_ttl` on the resolution result) —
  **an expiring claim retracts itself**, which is why `LK-75` called it a requirement rather than a
  nicety. **This is a second, independent, deployed confirmation.**
- ⚠ **A retraction channel is a known-necessary shape, and a separate feed is the known-bad version of
  it.** Prefer short expiry to a retraction feed; **if a retraction record is ever specified, it must
  travel in the same object as the claims it retracts**, or it reproduces the failure exactly.

---

## §10 The web of trust, and what it was actually trying to do

**The preceding document named it as the canonical failure, with usability and revocation as the
reasons. Both are true, and there is a sharper statement available:**

⭐⭐⭐ ***The web of trust attempted to make VERIFICATION transitive.*** Its whole purpose is to let you
accept a key you cannot check, because someone you trust vouched for someone who vouched for it —
**trust flowing into the *what do I accept* half.** `LK-80` says that half is not transitive, and the
web of trust is the field's largest experiment in trying anyway.

**And the ending is instructive: when the shared structure that carried it was flooded, the practical
response was to stop distributing third-party vouching altogether** — a critic's objection being
precisely that this *"kills a very important portion of the protocol, the Web of Trust."* ⭐ **Both
sides of that argument are right, and the reason the trade was forced is that the vouching could not
be verified by the reader.** ⇒ ***Content addressing does not solve the web of trust; it makes it
unnecessary*** — there is nothing to vouch for when the reader can check the bytes.

---

## §11 What this changes

**Nothing here overturns the design. Four things sharpen and one is new work:**

1. ✅ **A self-chosen table size is sound and the penalty is small** — ~2× hops per hundredfold
   reduction. **State the budget, not the size.**
2. ✅ **The aggregator is where the curve puts you, and it is a purchase** — near-constant lookup
   bought with concentrated load. **Say so, rather than treating aggregation as free.**
3. ⭐ **Add the fidelity axis beside the granularity axis**, and note that a probabilistic table is
   **not enumerable**, which is a privacy property the granularity axis alone does not give.
4. ⭐⭐ **The referral record is the missing mechanism**, it is the same record as a locator entry, the
   hierarchy objection does not bind it, and it needs the bounded-walk artifact that is already flagged
   for promotion.
5. ⭐⭐ **The abuse review's remaining items now have shapes:** a **per-issuer** bound rather than
   per-key; **consent-to-be-listed** as the answer to the probe's unresolved half; **quorum as an
   ordering rule**; **expiry over a retraction feed**.

**Still not measured, and it remains the real open question:** pruning cost at aggregator scale.

---

## §12 Sources

**Read for this document.**

- **Accordion** — [*Bandwidth-efficient management of DHT routing tables*, Li, Stribling, Morris,
  Kaashoek, NSDI 2005](https://pdos.csail.mit.edu/~strib/docs/dhtcomparison/accordion-nsdi05.pdf)
  (full text read; §1, §3.1, §3.2, §4.1 quoted).
- **The state/diameter bound** — [*On the Fundamental Tradeoffs Between Routing Table Size and Network
  Diameter in Peer-to-Peer Networks*, Xu, Kumar, Yu, INFOCOM 2003 /
  JSAC Nov 2003](https://ieeexplore.ieee.org/document/1258122/) — ⚠ **abstract and secondary
  restatements only; primary PDF unreachable, see the caveat in §2.3.**
- **Kelips** — [*Building an Efficient and Stable P2P DHT through Increased Memory and Background
  Overhead*, Gupta, Birman, Linga, Demers, van Renesse,
  IPTPS 2003](https://link.springer.com/chapter/10.1007/978-3-540-45172-3_15) — the `O(√n)` state /
  `O(1)` lookup corner, **probabilistic, not absolute**.
- **Gnutella** — [Query Routing Protocol specification](https://rfc-gnutella.sourceforge.net/src/qrp.html)
  (full text read; `RESET`/`PATCH`, false-positive language quoted) ·
  [Ultrapeer proposal](https://rfc-gnutella.sourceforge.net/Proposals/Ultrapeer/Ultrapeers.htm) ·
  [leaf mode and ultrapeer mode](https://shareaza.sourceforge.net/mediawiki/index.php/Developers.Gnutella.Notes.LeafModeAndUltrapeerMode).
- **Scuttlebutt hop scoping** — [`ssb-config`](https://modules.scuttlebutt.nz/client/ssb-config) ·
  [`ssb-replication-scheduler`](https://github.com/ssbc/ssb-replication-scheduler) (per-hops partial
  replication templates).
- **Certificate flooding** — [SKS Keyserver Network Under
  Attack](https://gist.github.com/rjhansen/67ab921ffb4084c865b3618d6955275f) ·
  [CVE-2019-13050](https://access.redhat.com/articles/4264021) ·
  [Sequoia analysis](https://sequoia-pgp.org/blog/2019/07/08/certificate-flooding-sks-gnupg-issues-the-sequoia-project/) ·
  [keys.openpgp.org FAQ](https://keys.openpgp.org/about/faq/) (attested certifications; the
  distribute-with-the-signer proposal).
- **Shared blocklists** — [FediBlockHole](https://github.com/eigenmagic/fediblockhole) (merge from
  chosen sources; severity tiers) · [Gardenfence](https://github.com/gardenfence/blocklist) (6-of-7
  consensus threshold) · [Fediverse blocklist
  commentary](https://seirdy.one/posts/2023/05/02/fediverse-blocklists/) (retraction propagation).
- **Nix trust model** — [Adding binary cache
  servers](https://nixos-and-flakes.thiscute.world/nix-store/add-binary-cache-servers) ·
  [`nix store verify`](https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-store-verify)
  (`--sigs-needed`) · [signing keys and input-addressing](https://docs.nixbuild.net/signing-keys/) ·
  [configurable signature verification](https://github.com/NixOS/nix/issues/14451) ·
  [non-transitive substituter trust](https://github.com/NixOS/nix/issues/9644).

---

## Document history

- **2026-09-13** — created. Continues the locator-table landscape review on the self-sized-table axis
  and closes the four systems named as unread by the preceding document.
