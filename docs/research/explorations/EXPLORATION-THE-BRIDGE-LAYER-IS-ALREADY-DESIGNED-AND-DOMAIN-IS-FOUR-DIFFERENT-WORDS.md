# EXPLORATION — the bridge layer is already designed, it is unbuilt, and "domain" is four different words

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing is folded here.**
Third in the organization arc, after the degree-of-freedom study and the reader model.

**Why it exists.** The arc's last open directory question was *what does `specs/domains/` mean?* — asked
because the directory holds one document, is annotated in the architecture map with the words *"domain
specs"*, and reads as arbitrary beside three siblings that do not. **The question turned out to have a
real answer, most of it settled years ago, and none of it in this corpus.**

---

## §0 The result

| | |
|---|---|
| **"Domain spec" is defined exactly once in the published corpus — and circularly** | The architecture map's directory listing reads `domains/ ← domain specs`. The phrase occurs **once** in `specs/` and `guides/` combined, and that occurrence is the definition. **Nothing else says what a domain is** |
| ⭐⭐ **Why the definition is circular: the fourth directory is on a different axis from the other three** | `extensions/` · `sdk/` · `applications/` — **each annotated with its LAYER** — **each annotated with a LAYER.** `domains/` is annotated with itself, **because it has no layer to name.** The circularity is the symptom |
| ⭐⭐⭐ **The word carries four unrelated senses here, and the directory inherited the weakest** | a pre-split **repository area** · a **workstream** that a charter declares · the **scope** of a single bridge extension · this **directory**. §3 |
| ⭐⭐⭐ **The design being re-derived is settled, in the pre-split archive, and it answers the hard case** | **Three layers — protocol / transport / adapter** — with bridges as the adapter layer, one extension per bridge, install-time ingress/egress modes, and **no umbrella `EXTENSION-BRIDGE`.** §4 |
| ⚠ **Almost none of it survived — one piece did, and finding it corrected this study** | The three-layer model, the mode axis and the seven questions are absent; **the transport-vs-foreign-substrate discipline is PRESENT** in the extension-development guide §3.7. **And a bridge specification is cited by TWO landed documents and does not exist here** |
| **The tree prefixes have the same defect, measured** | Three reserved prefixes for the built world. **One has zero real occupants; one is 92% a single member; one has two named occupants and no specification.** Two of the three claim the filesystem. §6 |
| ⭐⭐⭐ **The decision that is actually live is not the directory's name** | It is **what the first bridge is called**, because there are none, and the first one sets the pattern for all of them. **That costs nothing today and is locked the moment one ships.** §8 |

---

## §1 The definition is circular, and it is the only one

The architecture map lists the four specification directories:

```
├── extensions/      ← L2.5 extension specs
├── sdk/             ← L3-L4 SDK specifications
├── applications/    ← L5 conventions
└── domains/         ← domain specs
```

**Measured across `specs/` and `guides/`: the phrase "domain spec" occurs once — that line.** The
authoring standard declares **six document kinds** and *domain* is not among them; the single document
in the directory declares no kind at all. **So a reader asking what belongs in `domains/` has exactly
one sentence available, and it restates the directory name.**

⇒ **This is not a reader failing to find the rationale. There is no rationale on this side of the
split.**

## §2 The fourth directory is on a different axis from the other three

Read the four annotations again. Three of them name a **layer in the stack**. The fourth names
**itself**.

That is the whole defect, and it explains why every attempt to fix it by renaming feels unsatisfying:
`bridges/` is not a layer either. **The set is not a taxonomy of one kind of thing — it is three layer
names plus one subject name**, and no rename repairs a category mismatch.

⚠ **It also dissolves a worry worth dropping.** These directories are **not** a document-class taxonomy:
the authoring standard's `Kind` axis is separate and orthogonal, and every one of these documents is the
same kind. **The directories group by *what the specification is about*, and that is a legitimate and
much weaker claim than it looks.** Nothing is broken by the mismatch; a reader is merely told less than
the shape implies.

## §3 ⭐⭐ "Domain" is four different words in this corpus

| Sense | Where it lives | What it means |
|---|---|---|
| **repository area** | the architecture document's own history, and its account of the pre-split layout | a top-level tree — *core-protocol domain*, *SDK domain* |
| **workstream** | the class map used by the corpus tooling: *"a domain charter declares what a domain is for and which workstream owns it"* | who owns a body of work. **Process, not protocol** |
| ⭐ **the scope of one bridge** | the pre-split archive: *"each specific bridge is its own single-extension domain"* | **the reach of a single extension.** A scope word — **not a document class** |
| **the directory** | `specs/domains/` | one specification |

**The third sense is the only one that touches bridges, and in it "domain" is not a class of document —
it is a statement that one bridge equals one extension.** ⇒ **the most plausible account of how the
directory came to exist is that scope word being read as a class name.**

⚠ **Stated as the most plausible account and not as a finding: no record was located that decides the
directory's name, and its absence is exactly what makes the question unanswerable from inside this
corpus.** The account is offered so the next reader starts from the right shelf, and it is falsifiable —
a naming record would settle it.

## §4 ⭐⭐⭐ The design is settled, and it answers the case that looks hardest

The pre-split archive carries a hostile-pass study of the transport family and bridge shape, and it lands
a three-layer separation:

| Layer | What it holds | Pluggable? |
|---|---|---|
| **Protocol** | envelope · signature · capability chain · identity · entities | **No — frozen across every transport** |
| **Transport** | how bytes get from A to B: live, async-queued, polling-static, mesh-pull, one-way-authenticated, routed-circuit, sneakernet | Yes, via the per-peer transport profile namespace |
| **Adapter** | **bridges** — translating an external system's idioms into protocol calls | Yes, one extension per bridge |

**And three rulings sit on top of it, all landed there:**

1. **There is no umbrella bridge extension.** *"Bridge is a development-time category, not a runtime
   surface."* Each bridge is its own extension. **Do not unify at the all-bridges level** — the
   fracturing signals are too sharp: different trust models, different idempotency, different capability
   flows.
2. **Each bridge is one extension with install-time modes** — egress-only, ingress-only, or both. The
   client direction and the server direction are the same extension configured differently.
3. **A cross-bridge discipline applies to each new bridge** — a checklist, not a shared runtime.

### 4.1 ⭐ Why one protocol can be both a transport and a bridge, and that is correct

**The case that makes the whole area look confused is the one the model handles cleanly.** A ubiquitous
web protocol appears at **two** layers, and they are different things:

- **As a transport binding** — envelopes moving between peers over a polling-static reachability shape.
  That is the transport layer, and it is a peer-to-peer concern.
- **As a bridge** — acting as an ordinary client against arbitrary third-party servers, or serving
  ordinary clients that have no idea an entity system is behind the surface. That is the adapter layer,
  and it is an interop-with-the-outside concern.

**These compose rather than conflict: a bridge can run over a transport, including the transport that
speaks the same protocol.** ⇒ *the sense that this protocol "transcended being a bridge" is a real
observation about its ubiquity, and it is not a defect in the model — the model already places it
twice, deliberately.*

⚠ **The lesson is about first examples, not about this protocol.** A first example that is simultaneously
infrastructure is a bad teacher: it makes the general shape look like a special case. **A bridge to a
version-control system or a package store would have taught the category better**, because neither is
load-bearing for the protocol itself.

## §5 ⚠ What survived, what did not — and a correction to this section's own first answer

> ⛔ **This section first read *"none of it is in this corpus"*, and that was wrong. It was measured by
> grepping the archive's LITERAL PHRASES — *adapter layer*, *development-time category*, the sibling
> bridge names — all of which return zero. The extension-development guide §3.7 carries the
> distinction in full under different vocabulary.** *Measuring a spelling and reporting it as a subject
> is this arc's most-repeated defect, and it was committed here while cataloguing it.*

**What DID survive:** the extension-development guide's §3.7, *Disambiguate HTTP-as-transport from
HTTP-as-foreign-substrate*. It separates **Mechanism A** — bytes on the wire are already entity-encoded,
so the protocol is transport for opaque verified bytes — from **Mechanism B** — the response is foreign
content that a bridge wraps as an entity for ingestion. It names the entity type, states a
disambiguation discipline (*ask whether the bytes are already entity-encoded before authoring*), and
records that two independent analytical chains once mis-conflated the two. **That is the §4.1 result,
landed here, and it predates this study.**

**What did NOT survive:** the three-layer model · the ingress/egress mode axis · the seven cross-bridge
questions · **and every bridge specification.**


Measured over `specs/`, `guides/`, the proposal workspace and the design record:

| Term | Occurrences here | But the SUBJECT? |
|---|---|---|
| *adapter layer* | 0 | — the layering itself is absent |
| *development-time category* | 0 | — the no-umbrella ruling is absent |
| the sibling bridge names | 0 | — no bridge but the web one is even named |
| **a bridge specification of any kind** | **0** | **absent, and cited twice** |
| the transport/foreign-substrate distinction | 0 *(by phrase)* | ⭐ **PRESENT — guide §3.7** |

⛔ **And TWO landed documents cite a bridge specification for the web protocol — the guide names it as
the normative home for Mechanism B, and an extension spec names it as a related document. Neither
resolves.** It exists in the pre-split archive as a proposal with a validation plan beside it. **A
normative pointer to a non-existent file is worse than an acknowledged gap: it reads as an instruction
the reader has failed to follow.**

⇒ **The split moved conclusions and left derivations, and here it did something narrower and worse: it
kept the NAME and left the reasoning behind.** A name whose rationale is not on the same side of a
boundary as the name is a name nobody downstream can evaluate — which is precisely the state the arc
found this directory in.

## §6 The reserved tree prefixes carry the same defect, and here it is measured

Three top-level prefixes are reserved for the built world:

| Prefix | Declared as | Measured occupancy |
|---|---|---|
| `host/` | *machine resources — filesystem, processes, network, hardware* | ⛔ **zero real occupants.** Six textual matches, all incidental |
| `local/` | *device-local data and handlers* | **153 references — 140 of them one member.** The rest are single mentions |
| `bridge/` | *external system connections* | **8 references, two names, zero specifications** |

⭐ **Two of the three claim the filesystem.** `host/` names it explicitly; `local/` is where it actually
lives. **Nothing states which wins** — and the live answer emerged by practice, not by rule. ⇒ **a
reserved prefix lost its declared subject to a neighbour and no instrument noticed**, because reserving
a prefix and specifying one are different acts and only the second leaves a document.

### 6.1 Why the surviving member is named the way it is, and it is a real distinction

**The distinction the naming encodes is worth keeping even if the words change.** A filesystem bridge
and a process bridge are **not** like a bridge to a package store or a mail system:

- **They are assumed present.** The system runs *on* them. They are not an optional integration a
  deployment may or may not have.
- **They are substrate as well as adapter** — which is why they attract a different prefix from things
  reached *across* a boundary.

> ⇒ ***The built world is not one category.*** There is *what we are standing on* and there is *what we
> are reaching out to*, and a taxonomy with one bucket for both will keep feeling wrong. **That is the
> content of the current split, arrived at by instinct, and it is defensible** — the objection to it is
> that the instinct was never written down.

## §7 What a rename would cost, measured

**The surviving member's namespace appears in 312 files across five independent implementations** — the
heaviest single seat carries 152, and every one of the five carries it. It is not a specification-side
identifier; **it is a path that shipped.**

⇒ **A rename is not a documentation edit. It is a coordinated change across five code bases for a
comprehensibility gain that the arc's own test does not return.** *(And it would fail both landed rename
tests independently.)*

## §8 ⭐⭐⭐ The live decision is the FIRST BRIDGE'S NAME, not the directory's

**This is the useful reframe and it inverts the priority.**

| | Cost now | Cost later | Reversible |
|---|---|---|---|
| Renaming the existing directory or namespace | **high** — 312 files, five implementations | higher | no |
| ⭐ **Naming the first new bridge** | **zero — there are none** | **locked the moment one ships** | until then, yes |

**There are no bridges.** The category is designed, named in a reserved prefix, cited by a landed spec,
and **entirely unbuilt.** The next one written sets the convention for every one after it — the
archive's convention, a directory-matching convention, or a third — **and whichever is used first will
be the pattern, whether or not anyone decides it is.**

⇒ **The rising-threshold argument applies here at its sharpest, because the threshold is currently
zero.** This is the one place in the whole arc where the organizational choice is genuinely free, and it
is the one place nobody was looking, because the attention went to the expensive end.

**What a decision here needs to settle, and it is small:**

1. **The name class** for a bridge specification, and whether it is the same word as the directory.
2. **Whether a bridge is an extension.** The archive says yes — one extension per bridge, no umbrella.
   If that stands, the directory question may dissolve on its own.
3. **Where the assumed-present ones sit** relative to the reached-across ones (§6.1), given that the
   distinction is real and the current split already encodes it.

## §9 What this does NOT claim

- **It does not claim the current naming is wrong.** It claims the reasoning is not on this side of the
  split, and that a name whose rationale is unreachable cannot be evaluated by the people it is for.
- **It does not propose a rename.** §7 prices one and the arc's tests do not return it.
- **It does not settle how the archive's three-layer model should be folded**, only that it exists, that
  it answers the questions being re-derived, and that it is cited by a landed spec through a document
  that is not here.
- **It does not claim the directory was created by misreading a scope word.** §3 marks that as the most
  plausible account and as falsifiable.
- **It does not claim anyone else has bridge work under way.** A named search across the independent
  implementation trees found nine files mentioning a bridge namespace, all incidental. **The category is
  unbuilt everywhere, not merely unspecified here.**

---

## §10 Sources opened

- The architecture document's directory map and its account of the pre-split layout.
- The authoring standard's six document kinds, and the single document in the directory.
- The pre-split archive's transport-family and bridge-shape study, and the framing review it cites.
- The reserved-prefix table, and a corpus-wide census of the three prefixes.
- The landed extension spec whose related-documents line cites an absent bridge specification.
- Five implementation trees, for the namespace occupancy count.
