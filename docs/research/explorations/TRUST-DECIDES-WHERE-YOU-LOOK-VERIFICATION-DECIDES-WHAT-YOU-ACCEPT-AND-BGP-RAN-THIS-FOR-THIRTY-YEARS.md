# Trust decides where you look, verification decides what you accept — and BGP ran this for thirty years

**Status:** EXPLORATION (2026-09-13). **The refinement that closes the locator-table arc.** It resolves
the one attack that landed, on a distinction the corpus already holds, and it reads the largest
deployment of this exact model.

---

## §0 The sentence

> ⭐⭐⭐ **Trust decides WHERE YOU LOOK. Verification decides WHAT YOU ACCEPT. They are separate, and
> only the first one is transitive.**

**Everything below is that sentence's consequences.**

---

## §1 The correction that makes the attack small

**The preceding review modelled a reader as consuming tables it is handed.** ⛔ **That is the DHT's
model, not this one.** In Kademlia you accept records from whoever the protocol routes you to; **there
is no adoption step, which is exactly why anyone's poisoned record lands in your view.**

✅ **Here there is an adoption step, and it is the whole design.** **You build your table.** Merging
someone else's is a decision you make, about a party you chose, and the corpus already specifies the
machinery: `EXTENSION-REGISTRY` §4.1's **precedence dispatch list is the consumer's config**, and §5
puts the decision on the receiver — *"anyone can publish a binding claiming any name. **Receiver policy
decides.**"*

⇒ **`LK-78`'s amplification attack requires the victim's readers to have CHOSEN to merge from the
poisoner.** It is not a property of the mechanism; **it is a consequence of a specific bad adoption
decision, bounded to the readers who made it.**

---

## §2 The two trusts, which must never be conflated

**This corpus already rules one of them, and the ruling is the load-bearing half:**

| | **trust in the CONTENT** | **trust in the HINT** |
|---|---|---|
| question | *may I believe these bytes?* | *where should I look?* |
| answer | ⭐ **no trust needed, ever.** `EXTENSION-SUBSTITUTE`: fetch-by-hash after a local miss, **re-hash on receive**, ***no transitive trust*** — *"trust the content, trust the key, fetch from anywhere"* | **a routing preference, and it is where the merge decision lives** |
| transitive? | ⛔ **never** | ✅ **yes, deliberately — that is what makes a table graph work** |
| a wrong answer costs | ⛔ **nothing. It cannot happen** — the bytes do not verify and are discarded | **a wasted fetch (you) and load (the victim)** |

⭐⭐ **So a bad hint can never become bad data.** The worst a lying table achieves is **resource waste**.
That is why `LK-78` is an **abuse** problem and not a **security** problem, and why its mitigation is a
volume bound rather than anything cryptographic.

> ⭐ **And this is the structural difference from BGP, which is otherwise the same design.** *Accepting
> a BGP route IS accepting the traffic path — there is no second check.* **Here, accepting a hint costs
> a fetch and the bytes are verified regardless of who served them.** BGP has one trust decision doing
> two jobs; this has two mechanisms doing one each.

---

## §3 BGP is this model at global scale, and it has run for thirty years

**The similarity is not an analogy.** I publish my table. You decide whether to accept it. Peering is a
relationship, not a protocol right. ⇒ **this is the largest, longest-running deployment of
trust-scoped published-table merging that exists**, and it carries the internet.

### §3.1 What BGP says its own weakness is — and it is the one we do not have

> *"Current BGP design is based on **unconditional trust** between BGP peers. **BGP has no ability to
> verify the accuracy of routing information, so anyone can announce anything.**"*

⭐⭐⭐ **That is precisely the weakness content addressing removes.** *"I can reach this prefix"* is
**unverifiable** — checking it means routing traffic there, which is the damage. *"I hold these bytes"*
is **verifiable for 2 KiB.** **BGP's central, thirty-year, unsolved problem is the one problem this
design does not have.**

### §3.2 What BGP built instead, and all of it transfers

| BGP mechanism | what it is | here |
|---|---|---|
| ⭐ **`AS_PATH`** | the chain of who relayed this announcement to me | ⭐⭐ **the merge provenance** — *my table is a union of tables I pulled in, and I keep the history of each.* **BGP proves this is necessary: RPKI validates the ORIGIN, and *a route leak is a PATH problem, not an origin problem — the origin is legitimate, the propagation is wrong.*** Origin attribution alone is not enough |
| ⭐⭐ **max-prefix limit** | **a circuit breaker**: if a peer announces more than N, tear down the session | ⭐ **the direct answer to `LK-78`.** A trusted source that suddenly claims 10× its usual volume is the signal. **Twenty-five years deployed** |
| **prefix filtering** | check what a peer announces against what it *should* be able to announce | *does this source plausibly know about this namespace?* |
| **IRR-generated filters** | generate filters from published data so policy scales without per-peer hand-editing | how a large merger avoids manual curation |
| **monitoring** | *"continuous monitoring for unexpected path lengths, new intermediate parties, unknown origins, and sudden count jumps"* | ⭐ *"is what turns a major outage into a recoverable incident"* |

### §3.3 The honest limits BGP also supplies

- ⚠ **Max-prefix is blunt.** *A volume heuristic with no notion of which routes are legitimate — it
  catches fat-finger full-table leaks but **not a surgical leak of a handful of prefixes.*** **The
  cheap attack is stopped; the careful one is not.**
- ⚠ **The coverage problem.** *"Filters cannot fully protect against leaks or hijacks if they are not
  implemented globally."* ⭐ **This transfers WEAKER here, for two reasons:** a bad announcement
  propagates through transit to parties who never chose the originator, whereas **a bad locator entry
  only reaches readers who chose to merge from someone who merged it** — and **a leak black-holes
  traffic while a bad hint wastes a fetch.**
- ⚠ **Prevention is incomplete, and that is the deployed posture.** BGP's answer after thirty years is
  **layering plus monitoring**, not a solution. **Anyone expecting this design to be attack-proof should
  read that as the ceiling.**

---

## §4 On reputation — the corpus, the design instinct and BGP agree

**No formal reputation system.** BGP has none: it has **filters, limits, and who you peer with.** The
alignment is worth stating because it is three independent sources reaching one answer:

- **the design instinct** — *it is about who you let in and what information you let in*
- **`EXTENSION-REGISTRY` §5** — receiver policy decides, and **no capability gates publication**
- **BGP** — thirty years, global scale, **no reputation layer, and the mechanisms that work are local
  policy and anomaly bounds**

⇒ **What replaces reputation is: your own adoption decisions, provenance so you can attribute a bad
entry, and a volume bound so a compromised-but-trusted source cannot do unbounded damage before you
notice.**

---

## §5 What the self-regulating claim actually amounts to

**Stated precisely, because the loose version does not survive §1 of the preceding review:**

✅ **True:** a source that produces bad hints **loses its place in readers' precedence lists** — not by
a reputation score, but because **each reader independently stops consulting it.** That is
`EXTENSION-REGISTRY` §4.1 doing what it already does, and it needs no coordination.

✅ **True:** a compromised trusted source is **attributable**, because the aggregator does not re-sign
(§8.2) and the merge keeps provenance. **You can name who did it and drop them.**

⚠ **Not automatic:** *people find out* is **monitoring**, and BGP's measured verdict is that monitoring
is what makes the difference between an outage and an incident. **It does not happen for free.**

⛔ **Not true without a bound:** between *compromise* and *finding out*, a trusted source can do
unbounded damage. ⭐ **That gap is exactly what max-prefix closes, and it is why the circuit breaker is
not optional.**

---

## §6 What this changes

| | before | after |
|---|---|---|
| **`LK-78` amplification** | ⛔ lands, unsolved, blocking | ⚠ **bounded**: requires a chosen source, capped by a volume circuit breaker, attributable by provenance. **Still the review item, no longer a design hole** |
| provenance of a merge | an idea | ⭐ **required, and BGP proves why — origin attribution alone misses path problems** |
| reputation | open question | **not building one; three independent sources agree** |
| the trust model | vague | ⭐ **two mechanisms: trust routes, verification accepts, only the first is transitive** |

---

## §7 What is still owed to the next session

1. ⛔ ⭐ **The abuse review**, now with a concrete agenda rather than an open worry: **the volume circuit
   breaker (max-prefix), the merge provenance chain (`AS_PATH`), plausibility filtering, and the
   surgical-attack limit BGP states outright.**
2. **Whether probe-before-record and a volume bound are redundant or complementary.** *(iroh probes;
   BGP counts. They defend different things — possession versus volume — and the cheap answer may be
   the counter.)*
3. ⚠ **The tiered-fallback idea, unexamined:** *I will consult less-trusted sources only if the trusted
   ones cannot answer.* **That is CoralCDN's level-2 → level-1 → global walk with trust as the axis
   instead of locality, and git-annex's trust levels as the vocabulary.** Cheap, and nobody has costed
   it here.
4. **Prior art still unread on this specific axis:** Scuttlebutt's hop-count follow-graph replication
   *(read here, but not for this question)* · PGP's web of trust, **the canonical failure, and the
   reasons are usability and revocation rather than topology** · Mastodon's defederation lists ·
   **Nix's `trusted-public-keys`, which is this model shipping quietly in a package manager.**

---

## §8 Cross-references

`EXTENSION-SUBSTITUTE` §1, §2.1, §6 (fetch-by-hash, re-hash on receive, **no transitive trust**) ·
`EXTENSION-REGISTRY` §4.1/§4.1a (consumer precedence), §5 (receiver policy), §8.2 (no re-signing),
§8.3 (conflict surfacing) · `EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` and
`ANALYSIS-SUPPLY-CHAIN-LANDSCAPE-ALIGNMENT` (*trust the content, trust the key, fetch from anywhere*) ·
`EXPLORATION-THE-LOCAL-VIEW-PROBLEM…` (no transitive trust at consultation time) ·
`REVIEW-THE-PUBLISHED-LOCATOR-TABLE-ATTACKED-SEVEN-WAYS` (which this refines) · register rows
`LK-69`…`LK-79`.

**Sources read for §3.** [ThousandEyes, *Best practices to combat route leaks and hijacks*](https://www.thousandeyes.com/blog/best-practices-combat-route-leaks-hijacks) ·
[Noction, *BGP filtering best practices* (PDF)](https://www.noction.com/wp-content/uploads/2019/08/BGP-Filtering-Best-Practices.pdf) ·
[Site24x7, *BGP route leaks*](https://www.site24x7.com/learn/bgp-route-leaks.html) ·
[Virtua, *BGP route filtering: prefix lists, AS-path, GTSM*](https://www.virtua.cloud/learn/en/concepts/bgp-route-filtering-security) ·
[*Global BGP attacks that evade route monitoring* (arXiv:2408.09622)](https://arxiv.org/pdf/2408.09622).
