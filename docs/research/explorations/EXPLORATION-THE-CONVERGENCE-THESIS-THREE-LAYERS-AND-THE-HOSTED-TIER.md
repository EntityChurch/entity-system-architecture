# EXPLORATION — the convergence thesis: three layers of agreement, and the hosted tier

**Status:** Exploration (design record). Not a proposal, not normative.

**The question underneath all of it:** *what is actually being agreed to, and what does the agreement
buy?* The social-stack map answers *what to build*. This one answers *why it composes* — and it
covers three things the map did not: **the hosted publishing tier** (participation for people who
will not self-host), **static and live as one spectrum rather than two systems**, and **whether the
compute work is the same kind of convergence as the data work** (it is, and the design already
exists).

**Rung discipline.** Every claim below is rated **S** specified · **I** implemented somewhere · **X**
cross-verified between independent implementations · **D** deployed. This exists because the easiest
mistake in a document like this is to describe a design and a deployment in the same sentence.

---

## §1 The thesis

> **A network is a group of people who agreed on a data model and on the code that reads it. Once
> that agreement holds and is verifiable, almost everything else — following, threading, moderation,
> hosting, discovery — is a consequence rather than a feature.**

This is worth stating flatly because it explains the shape of everything else. The social layer is
not a product built on a protocol; **it is what a shared vocabulary plus content addressing produces
when you stop needing a server to hold the shared state.** A thread is not an object anyone owns; it
is a shape formed by references. A feed is not a service; it is an index someone published. Moderation
is not a policy engine; it is what you chose to republish.

**And the corresponding risk is the same one, inverted:** where the agreement is *not* verifiable, it
silently stops holding. Two implementations of one landed convention currently emit different type
tags for the same object, and neither can see it, because a query over the wrong tag returns a
correct, complete, **empty** answer. **An unverified agreement is indistinguishable from agreement
right up until it matters.** That is why §2's third column is the load-bearing one.

---

## §2 Three convergences, one discipline

The agreement is not one thing. It has three layers, they are independent, and **each has its own
verification mechanism — all three of which reduce to comparing content hashes.**

| | Layer | The agreement | Verified by | Rung |
|---|---|---|---|---|
| **A** | **Implementation** | independent programs implement one specification | a conformance oracle over generated peers | **X** for the core; **nothing above core is generated** |
| **B** | **Data** | independent programs use one entity vocabulary | conformance vectors — example entities + expected hashes | **X/D** for core types; **S only** for the app tier, and one member has measurably diverged |
| **C** | **Compute** | independent runtimes evaluate one program identically | boundary-hash equivalence across engines | **S/I**, partial **X** |

### §2.1 Layer A — implementation

A generated cohort of peers, produced from the specification text rather than from each other, all
passing one conformance pin. **This is the strongest structural evidence the project has** — not that
the implementations are correct, but that the *specification is complete enough to be implemented
without reading a sibling*.

**The honest boundary:** the generated foundation is exactly one tier deep. The generator's input set
is the core protocol, so **its output scope is its input scope** — the extensions and the SDK are not
generated. That is a fact about the input manifest, not about the specification, and extending the
output means extending the snapshot first.

### §2.2 Layer B — data

**This is where the social work lives, and it is the weakest of the three.** Core entity types are
cross-verified and deployed. The application-tier vocabulary is specified and **ships zero conformance
artifacts across all three landed conventions** — each says so about itself.

**The cost of that gap is measured, not theoretical**, and it is the divergence in §1. This is why
every proposal that comes out of the social work has to carry its vector set as a gate rather than a
follow-up, and why the compatibility contract — *all old data must remain valid under the updated
vocabulary, and new data must be valid under the old* — belongs in the domain charter rather than in
one convention.

### §2.3 Layer C — compute, and yes it is the same kind of thing

**The intuition that compute convergence is the functional twin of data convergence is correct, and
the design for it already exists in this corpus.** From the generic-host exploration:

> **Transferability is now two ABIs, one per level.** The **compute IR** is the ABI for *evaluation*
> (each runtime brings its engine); the descriptor's **`(role, shape)` vocabulary** is the ABI for
> *I/O* (each host brings its drivers). **A program = `(descriptor + step IR + initial_state)`, all
> content-addressed — transfer those three by hash and any conformant host on any runtime runs it.**

That is, in the corpus's own words, the *"totally predictable, transferable, hashable stack"* — and it
is the same discipline as layer B one level up: **agree on a content-addressed vocabulary, verify by
hash.** Layer B says what an entity *is*; layer C says what a computation *is*. Both are artifacts you
can hand someone by hash and have them reproduce.

**The verification mechanism is genuinely parallel too.** Data convergence is checked by *"do two
implementations produce the same bytes for this entity?"* Compute convergence is checked by *"do two
engines produce the same boundary hash for this evaluation?"* — a differential corpus of
`(program, inputs, budget) → boundary-hash | error`. Same question, different noun.

**Rungs, honestly:** the IR and the evaluator are specified and implemented at several seats with a
differential corpus; **the generic host is at first-result stage at one seat with a design response
from a second**, and the `(role, shape)` driver vocabulary is a small new normative surface that has
not landed. **And the compute track is deliberately sequenced behind the current release** — the
reachability stack is the thing you cannot get wrong later, whereas compute refinements are mostly
*within* a peer.

**So: real, designed, partially built, and correctly not now.** The right posture is to *not* pull
compute into the social proposal, and to note that when it arrives it lands on the same rails — which
is exactly what makes it cheap later.

### §2.4 Why three layers rather than one

**They are independent, and that is the useful part.** A peer can hold your data without running your
programs; a runtime can execute your program without adopting your social vocabulary; a generated peer
can be conformant without either. **Each convergence is separately achievable and separately
verifiable**, so none of them is a prerequisite for shipping the others — which is what makes the
system approachable in pieces rather than as a monolith to adopt whole.

---

## §3 The hosted tier — the cheapest multi-tenant model there is, and it is not a platform

The map assumed everyone self-hosts. Most people will not, and the on-ramp matters more than any
feature. **The hosted option is architecturally interesting rather than merely convenient**, and it is
worth its own section because it is the shape that makes adoption plausible.

### §3.1 What the service actually does

```
peer decides what to publish  →  signs it  →  hands it to the service
                                                     │
                                          verify the signature
                                                     │
                                          write bytes to static storage
                                                     │
                                                    done
```

**And that is the whole service.** It has no read path — a CDN serves the bytes. It has no per-tenant
application, no per-tenant database, no session state, no feed assembly, no delivery queue. **It
interacts with a tenant exactly once per publish.**

### §3.2 Why this is not a platform, structurally

**The service never holds authority, because it never holds a key.** Everything it writes was signed
by the tenant before it arrived. So the complete list of what a hosted publisher *can* do to you:

| It can | It cannot |
|---|---|
| refuse to publish | **forge an entry in your name** |
| delete or withhold your bytes | **alter a byte of what you published** |
| observe what you publish, if unencrypted | **impersonate you anywhere** |
| correlate your activity | **prevent you publishing elsewhere** |
| stop existing | **take your identity, your followers, or your history** |

**The trust surface is availability and privacy only. It is not integrity and it is not identity.**
That is a categorically smaller relationship than any hosting arrangement in the surveyed field, where
the host is also the signer, the namer, and the authority.

**Two honest notes on the left column.** Withholding is real, and it is *indistinguishable from a
quiet publisher* at the consumer — the known detection gap, whose only mitigation is a second origin
or a mirror. And surveillance is real for unencrypted content, which is a reason confidential entries
are a genuine feature rather than a niche one.

**Exit is free and instant.** The tree is content-addressed and signed; moving it changes no hash,
breaks no reference, and requires no follower to do anything. **There is no lock-in surface to
construct** — not because the host is benevolent, but because there is nothing to hold hostage except
availability of bytes that are, by construction, copyable.

### §3.3 Why it is cheap enough to be free

Per tenant, the marginal cost is: **bytes stored, plus one signature verification per publish.** No
row in a users table that gets read on every request, no per-tenant process, no read amplification —
the CDN absorbs reads, and content-addressed bytes are safe to cache forever.

**Compare the shape of the cost in the field.** A federated instance pays for a running application
per community, and its cost scales with *activity*. A relay pays bandwidth proportional to the
*whole network's* firehose. A pinning service pays to keep content alive against a DHT that forgets.
**A publishing proxy pays for storage of exactly what its tenants wrote**, and nothing scales with
readership because readership is served by a cache tier.

**Which is why the economics survive popularity rather than inverting under it** — the failure mode
that closes federated instances is that the volunteer's cost grows with their community's activity,
and here it does not.

### §3.4 What it means that this is an option rather than the model

**The same tenant can leave and self-host with no migration**, because the hosted service was never
holding anything but bytes. That is the property to protect in any design: **a hosted tier must be a
convenience over the same substrate, never a different substrate with a bridge.** The moment a hosted
tier gets a feature that self-hosting cannot express, it has become a platform and the exit stops
being free.

**The design rule that follows, and it is checkable in review:** *a hosted publishing service MUST NOT
be able to produce any entity a self-hosting peer could not produce identically.* If it can, the
convergence broke at layer B.

---

## §4 Static and live are one spectrum, not two systems

Worth stating because "static publishing" reads as a lesser mode and it is not.

**A static store is already a relay.** A CDN-hosted tree is a fully participating publisher — it is
found the same way, verified the same way, followed the same way. Nothing in the reader's loop knows
or cares whether an always-on daemon is behind the bytes.

**So the postures are a cost/liveness dial on one system:**

| Posture | Cost | Gains | Loses |
|---|---|---|---|
| **Live peer** | a running process | interactive dispatch, delivery, request-time authority | costs uptime |
| **Hosted publish** | storage + a publish step | no uptime requirement, no ops | no request-time authority |
| **Static / dormant** | storage only | approximately free, indefinite | nothing new until revived |

**The same identity, the same tree, the same followers across all three, moving between them without
telling anyone.** That is *posture mobility*, and it is the property that dissolves the liveness tax —
the seeder-death, pin-or-perish, instance-shutdown failure family that every deployed peer-to-peer
system pays in some form.

**And it composes with §3:** hosted publishing is not a fourth posture, it is *who operates* the
middle row. A tenant can move from hosted to self-hosted and the network sees a peer that changed
nothing.

**The one thing static genuinely cannot do** — and this should never be blurred — is **request-time
authority**. A static origin cannot evaluate a grant at fetch time, so audience control on the static
path is **cryptographic rather than capability-gated**. That is why confidential entries matter and
why the two mechanisms are not interchangeable.

---

## §5 What else this generalizes to — and where the honest boundary is

The intuition that this reaches beyond social — repositories, build systems, the general infrastructure
of the web — is worth checking rather than asserting, because it is the kind of claim that is either
substantive or marketing.

**Checked: the substrate for a code forge is largely present, and it is present as specification.**
The revision extension specifies a **version DAG** with storage, traversal, **common-ancestor
finding**, **divergence detection**, merge configuration and a path inventory. That is the object
model and the merge base — the load-bearing half of what a distributed version-control system is.

**And the social work just supplied most of the collaboration half.** Read across:

| A forge has | Here it is |
|---|---|
| content-addressed objects | the store — **native** |
| commits and a merge base | the version DAG — **S** |
| branches and refs | naming over the DAG — **not specified** |
| issues and discussion | **the thread model of the social work** |
| code review on a change | a reference to a revision + a thread — **both shapes now exist** |
| forks | **a mirror, which is the same act** |
| releases | content-addressed artifacts — **native** |
| a package registry | a registry binding + content addressing — **S/I** |
| CI | compute — **S/I, deferred** |

**So the honest claim is narrow and still strong: the primitives exist across four already-specified
areas, and what is missing is the convention layer that names them — the same gap as the social one,
one tier over.** Not *"we could build a forge"* as a hope, but *"the missing piece is a vocabulary,
and we now know exactly what writing one costs, because we just scoped the equivalent."*

**Three things that do not come free, stated so the generalization is not read as unbounded:**

1. **Search and indexing at scale.** Nothing here indexes the network. Discovery-by-reference-graph
   (walking from participant to participant) is real and is *not* search. Every deployed system solves
   search centrally, including the decentralized ones.
2. **The legal and abuse surface.** No global takedown exists for anything, ever. That is the
   permanent shape of the model and it is a genuine limit, not a rough edge.
3. **Network effects.** A better architecture does not move people. **What it can do is lower the cost
   of leaving**, which is slower and more durable than winning, and which is the only honest mechanism
   on offer.

---

## §6 What this changes about the plan

Nothing in the social gap list is reordered by this document. Three additions:

| # | Item | Why it surfaced here |
|---|---|---|
| **1** | **The *publish-proxying* posture, written down** — a third party that publishes on a tenant's behalf, including §3.4's rule that it may produce no entity a self-hoster could not | It is the adoption on-ramp for people who will run nothing at all, and **it is the one part of the hosted tier that is specified nowhere.** *(Corrected 2026-09-04: an earlier form of this row said the hosted tier as a whole was unspecified. It is not. **The edge side is written down and exercised** — a provider-neutral requirement set covering TLS on a custom apex, a version-prefix rewrite that makes a publish an atomic flip with a rollback target, directory indexes, and permissive CORS **on error responses as well as successes**, so that a resolver can distinguish "the entity is not there" from "the browser blocked me" — with a probe that measures all five and a post-publish byte comparison of every key at the edge. What is genuinely missing is the **proxying** posture and, on the producing side, emitting the publishable tree from a store that is not a filesystem.)* |
| **2** | **The compatibility contract belongs in the domain charter**, not in one convention | §2.2 — it is layer B's verification discipline, and it is a property of the domain rather than of any member |
| **3** | **A forge-tier scoping note** — the primitives are enumerated in §5 and the missing piece is named | So the next person to ask *"could we do repos?"* gets the enumerated answer rather than re-deriving it, and does not mistake specification for deployment |

**And one posture recommendation on compute:** leave it out of the social proposal entirely. It is the
same kind of convergence, it lands on the same rails, and its track is deliberately sequenced behind
the reachability work for a reason that still holds. **Naming the relationship — §2.3 — is the useful
act; pulling it forward is not.**

---

## §7 The gaps this document adds

- **Withholding detection** remains the unmitigated hole in §3.2, and the hosted tier makes it more
  likely to matter rather than less, because a hosted tenant's availability is someone else's
  decision. Mirrors are the mitigation and they are §4's material in the social map.
- **The `(role, shape)` driver vocabulary** is an unlanded normative surface with no owner and no
  deadline, and it is the thing standing between the compute design and cross-runtime programs.
- **Refs and branches over the version DAG** — the one genuinely absent forge primitive, named in §5
  and owned by nobody.
- **Search** has no design, no owner and no prior-art section. Unlike moderation, which turned out to
  be native to the mirror model, **there is no reason to expect search to fall out of anything** — it
  is a real, separate problem and the map should stop implying otherwise by omission.
