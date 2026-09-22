# The bare hash has four possible answers, we rejected three, and the fourth leaks at the leaves

**Status:** EXPLORATION (2026-09-13). **Short on purpose.** The preceding documents in this arc kept
answering a different question; this one states the actual one and its actual answer.

---

## §1 The question, with nothing added

**You hold a hash. Nothing else.** No peer, no namespace, no type, no provenance. **Where do you go?**

Two clarifications that the preceding documents got wrong and that matter:

- ⛔ **`system/content/public/{hex(H)}` is a CONVENTION, not a standard path.** Nothing in this system
  has a standard path. A peer may bind a hash anywhere, under any namespace, or nowhere. **Assuming
  you know the namespace is assuming the answer.**
- ⛔ **"Ask the peers you know" is not an answer to this question.** If you know the peers, the problem
  was never hard — you ask them, for every hash, and it works today. **The question is what to do when
  you do not.**

---

## §2 There are exactly four structures that can answer it

**This is not a design survey. It is closer to a counting argument.** A hash is a uniformly random
number. **It contains no information about location.** So a mapping from hash-space to location-space
must be *stored*, *computed from an agreed map*, or *searched*. That gives four structures and no
fifth:

| # | structure | how the hash reaches a location | cost |
|---|---|---|---|
| **1** | **a global index** | someone holds `hash → location` for everything and you ask them | **a party everyone must consult** |
| **2** | **a hash-partitioned distributed index** | ⭐ **the hash IS the routing key** — partition the index by hash prefix or XOR distance | a live overlay, and every node must be reachable from every other |
| **3** | **ask everyone** | no index; search the network | `O(peers)` per query |
| **4** | **provenance travels with the hash** | ⭐ **you never hold a bare hash** — whatever gave you the hash gave you a locator | **you cannot answer for a hash that arrived alone** |

**Structure 2 is the only one where a hash is genuinely an address.** That is what a DHT is, and it is
why every system that wanted this property built one.

---

## §3 This corpus has rejected 1, 2 and 3 — each for a recorded, independent reason

| | rejected because | where |
|---|---|---|
| **1 — global index** | it is the concentration failure the whole design avoids; the surveyed systems centralized because a lookup became a prerequisite for every traversal, and **Upspin died of exactly this** — federated storage and naming, one global key server, now being turned off | `F-10`, `F-16a`, `F-33`, `LK-19`, `LK-26` |
| **2 — DHT** | **twice, on independent grounds.** ① our peers are static origins that cannot answer a live lookup, measured against **>70% provider-record decay** ② ⭐ **XOR proximity says nothing about whether a connection can be established** — NAT'd peers make Kademlia *"degrade or fail"* via the local-minimum trap, and `EXTENSION-NETWORK` §6.7 exists because our underlay is restricted by construction | `F-13`, `LK-7`, `LK-36` |
| **3 — ask everyone** | **not rejected — bounded.** It is correct on a fleet and dead on the open internet, and we specified it twice (`QUERY {expand, ttl, seen_peers}`) | `LK-32` |

⇒ ***The corpus has chosen structure 4, and `APP-CONVENTION-REFERENCE` §1 IS that choice, stated
normatively:*** *"a reference is routable if and only if it names one — **bytes alone name no holder
you can go and ask**."*

> ⭐⭐ **So the honest answer to the question in §1 is: THIS SYSTEM DOES NOT ANSWER IT, BY DESIGN.**
> A hash that arrived with no provenance is not findable here, and **that is a deliberate trade, not an
> unfinished feature.** Every prior document in this arc that treated it as an open gap was wrong about
> its own corpus.

---

## §4 But structure 4 leaks, and this is the real finding

**Provenance travels at the top of an artifact and is lost at its leaves.**

```
system/content/blob := { total_size, chunk_size, chunking, chunks: [hash] }
```

**That is the whole manifest.** Four fields, **no locator**, and the spec says so deliberately: *"the
blob carries no semantic metadata — content type, filename, purpose, etc. belong on the referencing
entity."*

**So:**

- The **reference** that brought you the blob names a publisher ✅
- The blob's **500 chunk hashes** carry nothing ⛔

⇒ **While the publisher is reachable this is invisible — you ask them for all 500.** The moment they
are not, **you hold 500 orphans, and a third party who has chunk 237 and would gladly serve it is
unreachable by any mechanism in the system.**

⭐ **This is a much narrower gap than "we need a content lookup", and it is the one that is real.**

---

## §5 What the deployed answer to exactly this is

**BitTorrent.** A `.torrent` or magnet link carries **`tr=` tracker URLs** alongside the infohash.
**The provenance travels inside the artifact.** The DHT is a *fallback*, added later, and — measured in
this survey — **PEX cannot bootstrap and no layer won.** The thing that makes BitTorrent work from a
standing start is that **the file you were handed tells you where to go.**

**That is structure 4, implemented as a field.**

**The design space here is small, and it is a field question rather than a mechanism question:**

| | shape | cost |
|---|---|---|
| **(a)** | **accept it** — chunks are fetchable only from someone who holds the blob; mirroring happens at blob granularity | simple; **what OCI and Nix do** — a mirror holds the whole layer |
| **(b)** | ⭐ **the blob, or its referencing entity, carries a locator set covering the closure** — one locator for the artifact, implying its chunks | one field; **this is the `tr=` answer** |
| **(c)** | per-chunk locators | a fan-out of provenance; **nobody does this** |

⚠ **And the limit that survives all three:** a locator names peers **the publisher knew about**. A peer
who acquired the content later and would happily serve it **is still invisible** — and finding *them*
requires structure 1, 2 or 3. **There is no fourth way to find an unknown holder. That is permanent.**

---

## §6 What should be written down

1. ⭐ **The refusal, as a stated limit rather than an open question.** *We do not answer "bare hash, no
   provenance."* It has been re-opened at least four times in this arc alone because **it is recorded
   as a gap instead of as a decision.**
2. ⭐ **The leaf-provenance gap** (§4) — a travelling manifest loses the ability to locate its own
   parts. **Small, concrete, and fixable with a field.**
3. **The permanent limit** (§5) — an unknown holder of a known hash is not findable, and **no amount of
   design changes that without adopting a rejected structure.**

**None of these is a lookup mechanism, and that is the point.**

---

## §7 Cross-references

`APP-CONVENTION-REFERENCE` §1 (the choice, stated normatively), §2.3 (`via`) · `EXTENSION-CONTENT` §2.1
(the blob's four fields), §5.1 (metadata belongs on the referencing entity) · `EXTENSION-NETWORK` §6.7
(restricted reachability) · register rows `F-10`, `F-13`, `F-16a`, `F-33`, `LK-5`, `LK-7`, `LK-19`,
`LK-26`, `LK-32`, `LK-36`, `LK-58` · and the five survey documents plus the synthesis and walkthrough
that preceded this, **all of which answered a question other than §1's.**
