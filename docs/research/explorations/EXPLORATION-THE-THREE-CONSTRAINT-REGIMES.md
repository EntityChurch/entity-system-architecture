# Exploration — the three constraint regimes: what governs a decision, and whose complexity it is

**Why this exists.** The boundary between the core protocol, the core extensions, and the network
family is described as "blurry" every time it comes up, and it gets re-derived from scratch each
session. It is blurry for a *reason*, and the reason is nameable: we have been sorting extensions on
one axis (**how necessary is it?** — `SYSTEM-ARCHITECTURE.md` §13.1's five tiers) when the thing that
actually feels different about NETWORK and SIGNALING is a **second, orthogonal axis** — *what governs
the decision, and therefore whose complexity we are looking at.*

**What it is for.** Reviewing a design, arguing about complexity, and deciding what "simplify" is even
allowed to mean. It changes the question from *"is this too complicated?"* — unanswerable — to
*"which regime is this complexity from?"*, which has a method.

**Status:** design-record synthesis. **Informative.** It classifies existing decisions; it rules on
none of them. Orthogonal to §13.1's tier axis and to `GUIDE-NETWORKING-MODEL`'s six concerns, and it
replaces neither.

---

## §1 The three regimes

### Regime I — **Substrate.** Mathematics and computer science govern.

A competent designer, starting anywhere, with no knowledge of our history or the internet's, arrives
at the same answer — or is wrong. Content addressing, cryptographic hashing, signature verification at
the terminal destination, Merkle structure, capability-chain verification, canonical-encoding
determinism, `tree = path → hash`, content-store dedup.

- **Degrees of freedom:** essentially none in *structure*.
- **Whose complexity:** nobody's. It is the complexity of the mathematics, and it is irreducible.
- **What "simplify" means here:** *being wrong.* A simpler Merkle proof is not a Merkle proof.
- **The failure mode:** re-deriving it locally and getting it subtly different — which is exactly why
  `AGENTS.md`'s five load-bearing invariants are foregrounded and why cross-impl conformance, not
  prose review, is the gate.

**The refinement that matters: a Regime I *structure* almost always contains Regime II *parameters*.**
That a hash must be collision-resistant is Regime I; *which* hash is arbitrary. **Hash agility exists
because someone noticed exactly this** — the v7.66–v7.70 arc removed a fixed 33-byte assumption, and
the `hash-width-pin` lint rule and the `content-hash` `(format_code, digest)` shape are the mechanism
that keeps the arbitrary parameter from re-fusing to the necessary structure. *When a Regime I
decision feels like it has a knob, the knob is Regime II and should be exposed as one.*

### Regime II — **Design space.** Coherence governs; we choose.

Several internally consistent answers exist and the system stays coherent under any of them. We pick
one for economy and coherence, not because the alternatives are wrong. The entity `{type, data}`
shape. The extension decomposition itself. The kebab/snake naming split. Four relay modes. The
three-slot capability model. Path conventions. `delivery_mode: push | poll` as a first-class choice.

- **Degrees of freedom:** real, and the reason the standing pins exist —
  `[[feedback_design_space_not_authority]]`, `[[feedback_expose_knobs_dont_pick_values]]`.
- **Whose complexity:** **ours.** This is the *only* regime where complexity is fully within our power
  to remove, and therefore the only regime where "this is too complicated" is a complete argument.
- **What "simplify" means here:** genuinely available, and the reduced-complexity discipline binds.
- **The failure mode, and it has two directions.** *Over-building* — inventing a mechanism where a
  composition of existing primitives would do. And *over-pruning* — cutting a point out of a design
  space because it looks over-broad, which is **L11**, and which cost three published rulings on
  RELAY's mode set. **An enumeration in Regime II is a design space; the expensive direction of a
  wrong guess is deletion.**

### Regime III — **Accumulated technology.** The world governs; we adapt.

Complexity imposed by decades of deployed technology we did not choose, cannot change, and must
interoperate with. NAT — because IPv4 was exhausted in 2011 and the workaround outlived the problem.
STUN / TURN / ICE — because NAT. WebRTC — because browsers will not hand out a socket. `http-poll` —
because a browser cannot listen. The CDN's `GET`-only, cache-keyed, no-callback shape. SMTP's envelope
/ body split. TLS. The browser sandbox and its storage limits. `MessagePort` message framing.

- **Degrees of freedom:** none, and *not* for Regime I's reason. Not because the mathematics forbids
  another answer, but because the installed base does. **This distinction is the whole point of the
  regime:** the constraint is contingent, historical, and would be different in a different world —
  and it binds us exactly as hard as if it were necessary.
- **Whose complexity:** **not ours, and still our problem.** Refusing it means not working on the
  actual internet. This complexity is *real* and cannot be designed away, only *placed*.
- **What "simplify" means here:** **almost nothing.** You can relocate it, quarantine it, or
  adapter-wrap it. You cannot delete it. **Arguing that an ICE flow is "too complicated" is arguing
  with 2011.**
- **The failure mode:** letting it leak inward, and mistaking it for our own bad design.

---

## §2 The rule that follows, and it is the load-bearing one

> **Regime III complexity must never leak into Regime I or II. It is quarantined at the edge, behind
> an adapter, expressed as data rather than as structure.**

Most of what looks like architectural fussiness in the network family **is this quarantine working**:

| Quarantine | What it keeps out of the substrate |
|---|---|
| **Transport profiles are entities** (`NETWORK` §6.5) | tcp / ws / http / http-poll / quic are *rows*, not branches in the dispatcher. A sixth transport is a new entity, not a new code path. |
| **Relay is a dumb carrier; routing is a plug-in seam** | The forwarding plane never learns a routing algorithm. Kademlia, BGP-learned, static tables are all *behind* `resolve_next_hop`, none inside relay. |
| **The reachability-class taxonomy** (full-duplex listener · half-duplex · held-connection · pollable · static publisher) | The browser's inability to listen becomes **one row in a dispatch table**, not a special case threaded through the protocol. |
| **`SIGNALING` as its own extension** | ICE/STUN/TURN — the most purely Regime III machinery we touch — is one extension the rest of the system can ignore. |
| **Byte-preservation at relay hops** (`RELAY` §9) | The inner envelope is never decoded, so no intermediary's encoder can perturb a Regime I hash. |

**Read that table the other way and it is a design principle, not a list of accidents:** every one of
those is a place where a Regime III fact was turned into Regime II *data* so it could not become
Regime I *structure*.

## §3 Why the boundary feels blurry — the actual answer

**Because an extension has a regime of *structure* and a regime of *motivation*, and they routinely
differ.** Sorting on either alone produces an ordering that feels wrong, which is the blur.

`EXTENSION-RELAY` is the clean example:

- Its **structure** — a mode set, an envelope shape, a capability model, a resolver seam — is
  **Regime II**. Several coherent designs exist; we chose one; the study that produced it says so
  explicitly and pins the modes as design space.
- Its **motivation** — that intermediaries are needed *at all* — is **Regime III**. In a world of
  universally reachable peers there is no relay extension. Relay exists because NAT, firewalls,
  browser sandboxes, and intermittent connectivity exist.

**So "is RELAY core-like or network-like?" has no single answer, and that is not a failure of the
question — it is two questions.** Its internals are ours to simplify; its existence is not ours to
question.

**The general form, and the diagnostic:**

> **Motivation tells you whether the thing gets to exist. Structure tells you how much freedom you
> have inside it. Complexity complaints must name which one they are about.**

*"Do we need SIGNALING at all?"* is a Regime III question and the answer is yes, because ICE exists.
*"Does SIGNALING need this many entity types?"* is a Regime II question and is fully open.

## §4 The classification

**Structure** = what governs the shape of the thing. **Motivation** = what governs its existence.
Tier is §13.1's, for cross-reference.

| Surface | Tier | Structure | Motivation | Note |
|---|---|---|---|---|
| V7 wire core, ECF, hashing, cap chains | 0 | **I** | I | The locked core. Regime I is *why* it is never renumbered. |
| `EXTENSION-TREE`, `-CONTENT` | 1 | **I** | I | `tree = path → hash`; dedup. Merkle/CAS results. |
| `EXTENSION-TYPE`, `-QUERY`, `-REVISION`, `-HISTORY` | 1 | **II** | I–II | Three-way merge is CS; the operation surface is ours. |
| `EXTENSION-INBOX`, `-SUBSCRIPTION`, `-CONTINUATION` | 1 | **II** | II | Async delivery, pub/sub, chaining — classic CS shapes, our composition. |
| `EXTENSION-COMPUTE` | 1 | **II** | II | Deliberate design space; the IR floor is ours. |
| `EXTENSION-CLOCK` | 1 | II | **III** | Wall-clock is a fact about the world; logical clocks are I. |
| `EXTENSION-IDENTITY`, `-ATTESTATION`, `-QUORUM`, `-ROLE`, `-GROUP` | 2a | **II** | II | K-of-N is I; the three-mechanism split is our choice. |
| **`EXTENSION-RELAY`** | 2b | **II** | **III** | §3's worked example. Mode set is design space; existence is NAT-forced. |
| **`EXTENSION-ROUTE`** | 2b | **II** | **III** | Paradigm-neutral storage (ours); paradigm choice is scale- and topology-forced. |
| **`EXTENSION-NETWORK` §6.5 transport profiles** | 2b | II | **III** | **The main landing site.** Where the accumulated stack is absorbed as data. |
| **`EXTENSION-SIGNALING`** | 2b | II | **III** | The most purely Regime III surface we own — ICE/STUN/TURN/WebRTC. |
| `EXTENSION-DISCOVERY`, `-REGISTRY` | 2b | **II** | II–**III** | Name→id is CS; DNS/DID:web/petname *shapes* are inherited. |
| `EXTENSION-ENCRYPTION` | 2b | **I** | II | Crypto construction is I; when-to-encrypt is ours. |
| **`GOSSIP`** *(roadmap)* | 2b | **I–II** | II | Anti-entropy / epidemic spread are **CS results with proofs** — closer to I than the rest of the family. |
| `CONTENT` blob manifest / chunk transfer | 1–2 | I–II | II | Torrent-shaped because chunk+hash is the CS answer, not because BitTorrent exists. |
| `EXTENSION-SUBSTITUTE`, CDN corridor | 2b | II | **III** | The CDN's shape is imposed wholesale. |
| `APP-CONVENTION-*` (L5) | 4 | **II** | III | Format is ours; the substrate set (markdown, browsers) is inherited. |

**Three readings worth having:**

1. **Regime III motivation is concentrated in exactly one place — Tier 2b, the network family.** The
   operator's intuition that "we veer into" something different at NETWORK/SIGNALING is **correct and
   now has a name**: it is the tier where motivation flips to III while structure stays II.
2. **Nothing in the system has Regime III *structure*.** That is the quarantine holding (§2). If a
   surface ever appears whose *shape* is dictated by an external stack, that is the finding.
3. **`GOSSIP` is the surprise.** It sits in the network family by topic but its structure is a
   proof-carrying CS result (anti-entropy digest reconciliation), so it is the one roadmap item where
   "design it right" means "read the literature," not "pick a coherent option." Worth knowing before
   it is drafted.

## §5 What this changes in practice

**When complexity shows up, ask which regime before asking whether it is justified.**

| Regime | "This is too complicated" is… | The right move |
|---|---|---|
| **I** | almost always **wrong** | verify against the mathematics; add a conformance vector |
| **II** | **a complete argument** | simplify — but check it is not a design space first (**L11**) |
| **III** | **an argument with history** | do not simplify; check the *quarantine* — is it data at the edge, or has it leaked inward? |

**Two review questions this yields, which nothing currently asks:**

1. **"Has a Regime III fact become Regime II structure anywhere?"** — the leak check. A branch in a
   dispatcher named after a transport; an entity type that exists only because of a browser
   limitation; a MUST whose rationale is "because CDNs do it that way."
2. **"Is a Regime II decision being defended as if it were Regime I?"** — the false-necessity check. It
   sounds like *"it has to work this way"* about something that has three coherent alternatives, and
   it is how a design space stops being one.

**And the honest thing this buys us with reviewers and implementers:** when someone says the network
family is more complicated than the core, the answer is not defensive. **It is more complicated, the
extra complexity is Regime III, it is not ours, and here is the quarantine that keeps it from reaching
the parts that are.**

## §6 What this does NOT claim

- **Not a replacement for §13.1's tiers.** That axis answers *how necessary*; this answers *what
  governs*. A surface has a coordinate on both.
- **Not a normative classification.** §4 is a working sort, informative, and several rows are genuinely
  arguable — `CLOCK` and `CONTENT` most of all. Argue them; that is what it is for.
- **Not a claim that Regime III complexity is acceptable wherever it lands.** The opposite: it is the
  complexity most likely to metastasize, which is why §2 is stated as a rule and §5 gives it a check.
- **Not earned as a discipline.** One synthesis, no incidents yet attributed to it. It is a lens, not a
  rung on the promotion ladder, and it must not be cited as though it were ratified.
