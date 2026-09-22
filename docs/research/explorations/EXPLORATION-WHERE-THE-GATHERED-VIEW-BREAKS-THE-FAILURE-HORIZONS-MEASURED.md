# EXPLORATION — where the gathered view breaks: the failure horizons, measured

**Status:** analysis, 2026-09-13. **The first scale numbers this design has ever had** — everything
below marked *measured* comes from an implementation's own publish path against its real projector;
everything marked *extrapolated* says so.
**Subject:** the feed / reference / embed / data-exchange stack, read as *a general pattern for
peer-to-peer data exchange* rather than as a social feature.
**Rests on:** `SYSTEM-DATA-EXCHANGE` (closure, the two preconditions, the growth rule) ·
`APP-CONVENTION-FEED` §4 and §6 · the eight-axis prior-art reference · the two-level-index exploration ·
the coverage audit's axis list.

> **In one sentence.** The design survives its own scaling arithmetic further than it looks, the
> publisher is not where it breaks, and **every scale measurement taken so far is of a publisher while
> every wall inside the next year is on a reader.**

---

## §1 What changed: this was all derived, and now some of it is measured

Until this week every cost claim in this arc was an argument. A publisher implementation has now run
its own emit path at 100 → 16,000 entries and the numbers are different from the arguments in two
places that matter.

**Measured** — one publisher, one tree, real projector, debug profile *(which is the profile the build
actually ships, so these are production numbers with unknown headroom, not lab numbers)*:

| entries | index pages | tree keys | files on disk | tree bytes | project | finish |
|---:|---:|---:|---:|---:|---:|---:|
| 100 | 4 | 205 | 444 | 90 KB | 16 ms | 5.6 ms |
| 1,000 | 32 | 2,033 | 4,239 | 883 KB | 159 ms | 95 ms |
| 4,000 | 125 | 8,126 | 17,275 | 3.6 MB | 683 ms | 426 ms |
| 8,000 | 250 | 16,251 | 33,626 | 7.1 MB | 1.40 s | 817 ms |
| 16,000 | 500 | 32,501 | 66,640 | 14.1 MB | 3.12 s | 1.76 s |

**Two derived constants worth carrying, because they are the model's price list:**
**≈ 4.2 files and ≈ 2.03 tree keys per entry**, and **≈ 900 bytes of tree per entry.** An entry costs
four files because it *is* four things — the entry, its detached signature, its index-page binding and
the trie nodes that changed. §5 argues that is the architecture, not overhead.

**And the design property holds exactly.** Posting one more entry at 4,000 moves **4 bindings out of
8,129** — the `O(tree depth)` invalidation surface the key-addressed index exists to produce, satisfied
to the number. ⭐ **The publisher then re-emits all 8,129 anyway.** Nothing in the convention asks a
publisher to exploit the property, and no implementation does.

> **One superlinear term was found and removed** — a closure-membership test nested inside the closure
> walk, computing a diagnostic counter that is printed and read by nothing. **20.4 s → 1.76 s at
> 16,000 entries.** Everything else in the emit path is linear. *A report-only computation is still on
> the critical path*, and for as long as that line existed it was the ceiling on **every** publish the
> implementation did — sites, apps, registry, feed — not just this one.

---

## §2 The failure horizons — ordered by when they bite

**The ordering is the finding.** These were discovered in roughly the reverse of this order, because
design-level problems are visible on paper and reader-loop problems are only visible in a profile.

| horizon | what breaks | mechanism | status |
|---|---|---|---|
| **now, 2 peers** | nothing | — | measured green, end to end |
| **week 1 of real use, tens of authors** | ⛔ **the reader's poll loop** | no cursor: every poll re-reads the whole window regardless of change | **§3 — the nearest wall by years** |
| **week 1, first live peer with >1k entries** | ⛔ **the live-read leg** | no index in the tree ⇒ enumerate the whole prefix, fetch everything, *then* truncate | **§3** |
| **month 1–6, operational** | one flat entries directory; a debug build on the publish path | neither is architectural; both are unpulled levers | known, cheap |
| **year 1–5, a prolific publisher** | publish is `O(whole archive)` per post | the invalidation property is free and unused | **§4** |
| **year 5–45, a person; year 1, an organization** | ~50,000 entries ⇒ ~208,000 files, ~44 MB, **~15 s per post** *(extrapolated linearly from the table above)* | the same | **§4** |
| **at any scale, gathered views** | ~~a mirror was one unbounded entity~~ | ~~fetch 6 MB to read 50 entries~~ | ✅ **closed** — paged head this week |
| **never — structural** | global content-anchored search | *"what contains X"* has no anchor; it is a whole-corpus query by the nature of the question | **§7 — stated, not solved** |

**The arithmetic behind the year columns.** A person posting three times a day writes ~1,000 entries a
year, so 5,000 entries is about five years and 50,000 is a lifetime. **An organizational or machine
publisher reaches 50,000 in a year**, and a sensor or log feed reaches it in a week. ⇒ *the publisher
horizon is a function of who is publishing, and the design is comfortable for people and tight for
machines.*

---

## §3 ⭐ The inversion: we measured the wrong side

**Every number in §1 is a publisher's.** The two walls inside the next month are both a reader's, and
neither has been measured at all — one is read off the code, one is a profile of a 34-entry feed.

**The reader's poll loop, with the measurement scaled up.** With no cursor, a reader re-reads its whole
window every poll: **~103 signed-root fetches per poll, per followed author**, at a window of 50.
*(Measured as 71 fetches for a 34-entry feed — two per entry plus one per page — and scaled.)*

| followed authors | fetches per poll cycle | at a 5-minute cadence |
|---:|---:|---:|
| 10 | ~1,030 | 3.4 / s |
| 50 | ~5,150 | **17 / s** |
| 500 | ~51,500 | **171 / s** |

**With a cursor the steady state is one head fetch per author plus `O(new)`** — 500 authors becomes
~500 fetches a cycle instead of ~51,500. ***That is a ~100× factor available from a mechanism the
convention already specifies and no implementation built***, and it is the single largest performance
item anywhere in this arc.

> **Why it was not built is a communication defect, not a design one, and it is ours.** A ruling
> retiring a *published cursor field* said *"do not build `{page, applied}`"* without saying it was
> about the wire shape. Read alone — which is how a ruling is read — that forbids the mechanism. The
> implementation built no cursor of any kind and the convention's `O(new)` rule went unimplemented.
> **A ruling that says less than it means, in the direction that removes a requirement, costs more than
> an ambiguous spec** — the spec at least gets re-read.

**The live leg is worse and is structural.** The index is a *publish artifact*: nothing writes it into
the tree, so a peer read over a live connection has entries and no index **by construction**, and every
live read takes the enumeration fallback — list every key under the prefix (unbounded), fetch every
entry *and its signature*, and only then truncate to the window. **Reading the newest 50 entries of a
50,000-entry peer is one 50,000-key listing plus ~100,000 dispatches.** The window parameter is not
consulted until after all the work is done.

⇒ **The repair is a design question rather than a patch**: either a bounded listing verb, or the index
gets written into the tree at ingest — after which the convention's model holds with no fallback and no
exception. **This is the sharpest open item in the arc** and it is larger than it looks, because the
same asymmetry says something general: **the static and live legs are not two transports for one
mechanism; only one of them has the index the mechanism depends on.**

---

## §4 The publisher's ceiling, and why it is the comfortable one

`O(whole archive)` per post is the honest description: 17,275 files rewritten to add one sentence at
4,000 entries, ~208,000 at 50,000. **It is also the wall that is furthest away, easiest to fix, and
least dangerous** — it degrades smoothly, it is visible to whoever is paying for it, and the property
that fixes it is already proven present.

**Three independent levers, none pulled, in increasing order of effort:**

1. **Build in release.** The publish path runs unoptimized. Free, unmeasured, and nobody knows the
   factor.
2. **Emit only what moved.** The invalidation surface is **4 bindings out of 8,129**, measured. An
   emitter that diffed against the previous projection would do `O(depth)` work for an `O(depth)`
   change. *The convention already promises this; only the writer declines to take it.*
3. **Shard the entry directory.** The content store shards; the entry pointers do not. 50,000 entries
   is 50,000 files in one directory — fine for the filesystem, a real cost for any sync, mirror or CDN
   push that has to enumerate it.

> **The thing to notice is that (2) is not an optimization, it is the design already being true.** The
> gap between a property being *held* and being *used* is where this whole audit lives: the mirror was
> unbounded under a convention with a bounding rule one section up; the reader had no cursor under a
> convention specifying `O(new)`; the publisher re-emits everything under a rule guaranteeing it need
> not. **Three times, the specification was right and the implementation implemented the easy reading
> of it** — and each time the easy reading passed every check, because the checks were written at the
> size where the two readings agree.

---

## §5 What is actually being exchanged — and the price of the answer

**The unit of exchange is a typed, individually-addressed, independently-signed entity.** That is the
whole architectural claim and everything else falls out of it.

| | BitTorrent | IPFS | here |
|---|---|---|---|
| unit | a fixed-size **piece** of an opaque file | an opaque **block** (UnixFS is a convention above it) | a **typed entity** — `{type, data}`, ECF-canonical |
| authorship | none — the torrent is the identity | none — a CID says *what*, never *who* | a **detached per-entity signature** at an invariant pointer |
| mutability | an infohash is immutable | weak — the mutable-pointer layer is the acknowledged soft spot | a **signed root** with a sequence number and a predecessor |
| discovery | a DHT: `infohash → who has it` | a DHT: `CID → who provides it` | ⛔ **nothing** |
| serving | peers | peers, or a gateway | **an ordinary static origin** — the tree is files |

**Why the typed-and-signed unit is load-bearing rather than a preference: closure is inexpressible
without it.** The property that an aggregator's output is the same kind of object as its input — so an
aggregator can be aggregated, and there is no privileged tier to centralize into — requires that a
republished item *still carries who wrote it*. **An opaque block cannot do that**: you can verify a
block's bytes against its address and learn nothing about authorship, so a republished block is
integrity without provenance and the whole gathered layer becomes hearsay. ⇒ ***the detached signature
is what makes republication evidence rather than assertion***, and it is the reason a third party
carrying your bytes is structurally unable to alter or claim them.

**The price is in §1's constants and should be stated plainly: ~4.2 files and ~900 bytes of tree per
entry.** BitTorrent's index costs ~6 bytes per peer per infohash; a block store amortizes across large
objects. **We pay per *item*, and the items are small.** For a timeline of text that is a real cost and
the numbers above are what it buys. **For large media it is nothing**, because the payload goes to the
content store and the entity is a pointer — which is why this pattern generalizes better to *catalogues
of large things* than to *firehoses of small ones*.

---

## §6 The one shape finding: a thread mirror is the fused index our own analysis argues against

**This is new, it follows from composing two things already in the corpus with this week's widening, and
it is not folded.**

The corpus's own analysis of reverse indexes concluded that `coordinate → WHO` and `WHO → WHAT` are
**two indexes**, that every surveyed system fuses them into `coordinate → CONTENT`, and that
**the fusion is what forces the firehose** — an aggregator must ingest everything because it cannot know
what will be queried. Splitting them makes the first index small (a set of identities, ~32 bytes each),
**merge-by-set-union**, and *able to omit but never forge* — a fabricated participant costs one wasted
fetch, because you verify by fetching their signed entry.

**A mirror carries `entries` — a list of pins to items. That is `coordinate → WHAT`: the fused shape.**

⇒ **And the resolution is that the two subject kinds want different shapes**, which the convention does
not distinguish:

| subject | writers | what a reverse index would hold | verdict |
|---|---|---|---|
| **live** — an author's timeline | exactly **one** | `{the author}` — trivial and useless | ✅ **the entry list is correct.** There is no split to make |
| **pinned** — a thread many parties contribute to | **many** | the participant set | ⚠ **the entry list is the fused form**, ~10× larger than the participant set, and it discards the union property the split was for |

**So the widening that let a mirror be of a timeline made the entry list right for the new case and left
it questionable for the original one.** For a 50-reply thread across 20 participants: a participant set
is ~640 bytes; the entry list is ~6 KB.

⚠ **Filed as a derived finding and deliberately not ruled.** The size argument alone does not carry it —
the corpus already corrected an earlier overstatement of that gap, and **the structural argument is
selectivity, not size.** What would settle it is a second gatherer implementation and a thread with real
participation; ruling it now would pin a shape on arithmetic.

---

## §7 What would falsify *"this is the general pattern"*

Stated as refutable claims, because a pattern nobody can attack is a slogan.

1. **"An aggregator's output is consumable by another aggregator."** Falsified if a gathered view ever
   requires a type or a code path a direct read does not. **Currently unfalsified and barely tested** —
   one implementation, and the aggregate-an-aggregator case has never been run.
2. **"A view is never wrong, only short."** Holds only while republication cannot substitute. Falsified
   by any format that admits a completeness claim, which is why there is a `MUST NOT` against providing
   a field to make one in.
3. **"No privileged tier."** Falsified the day finding a gatherer requires a lookup service. ⛔ **This
   is the live risk**, and §8 is why.
4. **"It works the same on a direct connection."** ⛔ **Currently false**, and §3's live-leg asymmetry is
   the reason: only one of the two legs has an index. **The strongest single falsifier on the list, and
   it is ours.**
5. **"Global content-anchored search."** ⛔ **Never claimed and should keep not being claimed.**
   *"What contributes to coordinate C"* has an anchor you hold; *"what contains the word X"* has none.
   The design can serve it at the honest price — somebody holds a lot of content — and that is a mirror
   at scale, not an index. **Say plainly that global search is not offered** rather than implying the
   split covers it.

---

## §8 The blind spots, by axis

**A coverage audit of the field ranked four axes weakest: dissemination, equivocation, completeness and
moderation** — noting that completeness and equivocation are *the same hole seen twice*. Read against
what this arc built:

- ⛔ **Discovery — `coordinate → who` does not exist here, and it is the one thing both comparable
  systems have.** A mirror's address is computable **once you know whose mirror you want**; nothing
  names the gatherers. The corpus argues this is narrower than it reads — a gatherer's output is an
  ordinary entry in an ordinary stream, so forward traversal from edges you already hold finds one —
  and that argument is **derived and has never been run.** What genuinely remains is cold start, which
  is the system's existing first-peer problem and not the gatherer's. ⚠ **The standing caution is the
  valuable part: solving this with a registry lookup would reintroduce exactly the privileged tier the
  design avoids, so a registry-shaped answer should be suspected on that ground before it is designed.**
- ⛔ **Equivocation / split view.** A publisher can serve different roots to different readers and
  nothing detects it. Reading the deployed answer elsewhere establishes this is **a twelve-year-old open
  problem** whose best-known treatments are recent and outside the standards series — *so the honest
  position is state of the art, not shortfall*, and the gap should be named rather than engineered
  around.
- ⚠ **Completeness.** Unverifiable by construction and always will be: a gatherer's source list is its
  own choice, union improves coverage monotonically and never certifies it. **Correctly handled** — the
  format cannot claim completeness — but it means *"did I see everything?"* has no answer at any scale.
- ⛔ **Retention.** Republication makes storage grow monotonically and **no rule anywhere says what a
  reader keeps or for how long.** A thread mirror grows forever. Carried as an open item and it is the
  one that turns from tidy-up into load-bearing the moment gathering is real.
- ⛔ **Moderation.** Unaddressed, and the prior-art record is blunt about the cost: a surveyed system
  with a mandatory append-only chain published immutability-as-a-harassment-vector as a known
  consequence. **The gathered layer is where this arrives** — a mirror republishes what its subject
  wrote, and *"a gatherer may omit"* is the entire mechanism available.
- ⚠ **Sustainability.** The recurring failure in every federated system surveyed is not technical: the
  volunteer running the always-on tier stops paying. **The exception is the one whose tier is a plain
  web server the publisher already had.** ⇒ ***this design's static-origin property is its strongest
  sustainability argument and should be treated as a feature, not an implementation detail.***

---

## §9 What to measure next, in order

**Every item here is a measurement, not a build, and the first three are reader-side because §3 is.**

1. **A reader over many authors.** Nothing has walked more than a handful. The 500-follow arithmetic
   and the source-leg saving that justifies the whole mirror are **derived, and the derivation has
   never met a profiler.**
2. **The live leg at size.** Its cost is read off the code and has never been run against a peer with
   more than a few entries.
3. **A browser-side render at the window size**, on a real device over a real origin. Every number in
   §1 is a native publisher or a native reader.
4. **Aggregating an aggregator.** The fixed point is the central claim of the composition tier and no
   check has ever exercised two hops.
5. **A gathered view past one page**, now that the shape supports it — the anti-vacuity arm being that
   the view spans more than one page before anything is asserted about reading it.
6. **Two independent gatherers of one subject**, which is what makes the coordinate derivation a
   measurement instead of one implementation's guess agreeing with itself.

> **The methodological point this arc earned.** Three separate defects here were *a specification being
> right and unexploited*, and all three were invisible to every check because **the checks were written
> at the size where the correct and the lazy reading agree.** A fixture with one page, one author or one
> profile does not distinguish them. ⇒ ***the anti-vacuity arm is not a nicety on a scaling check; it is
> the check.***
