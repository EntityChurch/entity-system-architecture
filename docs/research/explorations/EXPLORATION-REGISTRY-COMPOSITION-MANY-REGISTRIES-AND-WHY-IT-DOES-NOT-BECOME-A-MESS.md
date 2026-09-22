# EXPLORATION — registry composition: many registries, why it does not become a mess, and the four things actually missing

**Status:** Exploration (design record). Not a proposal, not normative.

**The question, in the operator's words:** *"I don't see this as being a system where it's just one
registry. There's going to be a lot of registries. I'll have my own registry, I publish my registry, I
publish the registry stuff that I use, the names I use. How do we stop that from turning into a total
mess?"* — with the standing one beside it: *"whether you need a registry of registries, and what all
that brings to the table, or you just let people follow and publish it themselves through whatever
networks they use."*

---

## §0 The result, and the first finding is about this document

**The mess is already prevented, by construction, and the answer is written down in four places that
have never been read together.** `GUIDE-RESOLUTION` §6 even quotes the operator's own framing back —
*"`alice@entity-church` reads 'Alice at the Entity Church Registry'"* — and pins the four name shapes
that make many registries a non-problem rather than a collision.

> **This document was opened as new research, and that was wrong.** The item was filed on the board an
> hour earlier as *"registry composition — **unstudied.** Nothing has been done on this."* **That is
> false**, and it is `AGENTS.md`'s own **L7** (*check the toolkit for the instrument before building
> one*) and **L16**'s corpus axis (*a claim that the corpus does not describe something is discharged
> by naming the region searched*) firing on the session that wrote both rules into a board the same
> day. **The region was never searched.** `spec coverage` was not run; `guides/` was not opened.
>
> **Recorded rather than quietly fixed, because the cause is the interesting part:** the question
> *sounded* like a network-topology question, and the answer lives in a **resolution guide**. That is
> exactly L16's *"layer membership predicts neither who owns a question nor who has answered it"*,
> arriving one day after it was written.

**So the substance of this document is: state the answer that exists, name the theory that says it is
the right one, and isolate what is genuinely left.**

| # | The mess-prevention rule | Where it already is |
|---|---|---|
| **1** | **A name carries its issuer.** `alice@entity-church` is *"Alice at the Entity Church Registry"* — so two registries both issuing `alice` are issuing **two different names**, not colliding on one | `GUIDE-RESOLUTION` §6.1 (four shapes) · REGISTRY §4.1a rows 4–5 |
| **2** | **A bare name never leaves your machine.** The catch-all admits **no name-transmitting backend**, as a MUST, with a ratified classifier for what counts as broad | REGISTRY §4.1 step 2, §4.1a row 6, §4.1b |
| **3** | **You follow keys, never names.** A follow's `subject` is a **peer-id**; the name you typed is kept only as `via`, *"for provenance display"* | `FEED` §2.4 |
| **4** | **Which registries you trust is your own local config, and nothing can install itself into it** | REGISTRY §4, §4.3 · RELAY §6.4 (*"resolver-config is peer-local and not synced… the aggregator is consumed, it does not install itself"*) |

**And the four things actually missing** — §5, in cost order: registry federation is **deferred behind a
blocker that does not exist** · the web-native backends are *"paper"* and they are the on-ramp · a
resolver-config cannot be **published** · and nobody has written down the theory, so the design reads as
a pile of choices rather than a forced answer.

---

## §1 The theory that makes this a forced answer rather than a preference

**Zooko's triangle:** a name cannot be simultaneously **global**, **secure** and **human-memorable**.
Every naming system in the field is a choice of two, and the ones that pretend otherwise have an
authority hidden in them.

**Stiegler's petname system is the resolution, and it is to stop asking one name to do all three.**
Read from the source:

| Name type | Properties | Stiegler |
|---|---|---|
| **key** | global + secure, **not** memorable | the cryptographic identifier |
| **nickname** | global + memorable, **not** unique | *"a nickname has a one-to-many mapping to keys"* |
| **petname** | secure + memorable, **not** global | *"a private bidirectional reference to keys"*, unique within one user's context |

> ***"The security of a petname system depends on the keys to prevent forgery, and on the petnames to
> prevent mimicry."***

**Our three are already exactly these, and the correspondence is not a coincidence — the local-name
backend's own source proposal is named `PROPOSAL-EXTENSION-REGISTRY-PETNAME`.** The decision was taken
once and the theory behind it was never written into the corpus, which is why *"there will be many
registries"* reads as a risk instead of as the design working.

| Stiegler | Ours | Where |
|---|---|---|
| **key** | the peer-id — self-certifying, and `did:key:`/Base58 resolve to themselves with no backend at all | V7 §1.5 · `GUIDE-RESOLUTION` §6.1 |
| **petname** | the **local-name backend**, plus `FEED` §2.4's `label` — *"local, never authoritative"* | REGISTRY §6 · `FEED` §2.4 |
| **nickname** | a **registry name** — issuer-scoped, one-to-many across registries **by design** | REGISTRY §1 position 2, §6a |

**The load-bearing consequence, which is REGISTRY's own §1 position 2 stated as theory:** *"the
substrate gates no name claims. Anyone can publish a binding entity claiming any name. Whether a
receiver TRUSTS that binding is the receiver's policy."* **That is the nickname property — global and
memorable and deliberately not unique — and a system that tried to make registry names globally unique
would need the authority we exist to avoid.**

---

## §2 What the field does, and NIP-05 is the cleanest confirmation

**Nostr's NIP-05 is the same three answers, deployed, and it says so in its own text.**

- The identifier is **`<local-part>@<domain>`**, resolved by `GET https://<domain>/.well-known/nostr.json?name=<local-part>` returning a `names` map. **Issuer-scoped by construction** — *only a domain owner can issue identifiers for their own domain.*
- *"The NIP-05 is not intended to verify a user, but only to identify them, for the purpose of
  facilitating the exchange of a contact or their search."* — **it is a nickname, and it says it is not
  a verification.**
- ***"Clients must always follow public keys, not NIP-05 addresses."*** — **rule 3, verbatim, from a
  deployed system.**

**ATProto reaches the same shape from the other end**: a handle is a domain name, proven by DNS TXT or a
well-known path, while the durable identity is a DID — and `did:plc`'s own README admits the directory
that makes *that* layer work is **"a central directory server."** *(Read in the previous session; not
re-fetched.)*

**DNS is the counter-example and it is the instructive one.** It is the one naming system with a truly
global namespace, and it needed a single root and an authority to get it. **Every decentralized system
since has declined the global namespace instead of decentralizing the root** — and the two that tried
the other way (consensus-anchored names) bought uniqueness with a chain.

> **So the field's answer to *"how do many registries not become a mess"* is: they never share a
> namespace to make a mess in.** A name is only meaningful *at* an issuer, keys are what you actually
> follow, and the memorable-and-yours layer is local. **We already made all three choices.**

---

## §3 How a conflict actually resolves here, end to end

**The mess scenario, walked:** two registries both issue `alice`, and I have both installed.

1. **They are different names.** `alice@registry-one` and `alice@registry-two` — REGISTRY §4.1a row 5
   dispatches each to the registry its authority names. **No collision exists to resolve.**
2. **If I type bare `alice`**, the catch-all runs, and it admits **only** local-name, self-certifying,
   out-of-band and peer-issued — *no name-transmitting backend*, as a MUST. So a bare name resolves
   against **my own handles and pins**, or fails. **It does not go shopping.**
3. **If two backends both answer a scoped name**, `resolver_chain[].priority` orders them and §4.1.1
   returns **the first validated hit** — deterministic, local, and mine.
4. **If I pinned it**, §4.1 step 1 returns the pin *before* dispatch runs. **The user's explicit
   decision beats every registry**, by construction.
5. **If an aggregator surfaces two upstream answers**, §8.3 requires it to **surface both, not silently
   pick**, with **fail-closed** as the default consumer policy and explicit pin as the override.
6. **And the link form carries its own check:** `name@peer_id` means *"resolve this name however my
   chain does, but the answer MUST equal this peer-id"* — name for reach, key for trust, decided
   structurally because peer-ids are Base58 multikey and registry handles are not
   (`GUIDE-RESOLUTION` §6.3).

**Step 6 is the one to notice.** It means a link someone shares can be **self-checking** without any
registry being trusted: the name gets you there, the key tells you it was the right place. **That is
rule 3 made operational in a shareable artifact**, and it is landed.

---

## §4 Do we need a registry of registries?

**For resolution: no, and the reason is rule 1.** If a name carries its issuer, then routing is a
property of the name, not of a directory — `alice@entity-church` needs no meta-registry any more than
`alice@example.org` needs one. **A meta-registry would only be required for bare names, which by rule 2
never leave the machine.**

**For discovery: yes, and it is not a registry.** *"What registries exist that I might want to use?"* is
a real question and it is the same shape as *"what clusters exist"* — **a published list somebody
curates, which others choose to read.** That is an ordinary entity, and the corpus already runs this
pattern three times: **published block lists**, **publishable rankings**, and the mirror. *"Curation and
moderation are the same act"* generalizes one step further here: **a directory of registries is an
opinion about naming authorities, and opinions are data.**

**And a third thing exists that is neither, which is the aggregator (§5.1):** one peer that subscribes
to N registries' binding subtrees and serves the union as *one* backend, so a consumer installs one
entry instead of twenty. **That is a convenience over the transport, not an authority** — REGISTRY §8.2
requires the aggregator **not to re-sign**, so receivers verify against the original issuer.

> **The three are worth keeping separate, because conflating them is how a system gets a root:**
> **routing** is in the name · **discovery** is a published opinion · **aggregation** is transport.
> Nothing in that list is allowed to become authoritative, and the spec already says so for the one
> that is most tempting.

---

## §5 What is actually missing

### §5.1 Registry federation is deferred behind a blocker that does not exist `[the finding]`

**`EXTENSION-REGISTRY` §8.2 — aggregator-as-meta-registry — is marked v1-DEFERRED**, inheriting
`EXTENSION-RELAY` §11.1a, which reads:

> *"Mode A's 'aggregator subscribes to N publisher peers' subtrees' requires **cross-peer subscription
> initiation** that does not exist in current substrate — subscription engines (the landed
> EXTENSION-SUBSCRIPTION and the cohort impls) are **local-tree-only**."*

**`EXTENSION-SUBSCRIPTION` v3.18 carries a top-level `## 6. Cross-Peer Delivery`:**

- **§6.1 Third-Party Delivery** — *"Peer A subscribes on Peer B… Peer B delivers notifications directly
  to Peer C"*, with the capability requirements in §6.2.
- **§6.3 Convergent mirroring `(v3.17)`** — ***"A mirror is a peer that reproduces another peer's
  subtree by subscribing to it and applying each reported transition locally"*** — with two **MUST**
  properties (bounded amplification at 1.5 × N; convergence-to-latest under delivery loss) and the
  closing line: ***"Verified cross-impl (Go / Rust / Python, fresh peers per directional pair) before
  this section was written; the bound is a folded result, not a new requirement."***
- And **`EXTENSION-REVISION` §6.3** builds on it in a third spec: *"B creates a subscription on A at
  `system/revision/{H}/head`… notifications land in B's inbox."*

**"A peer that reproduces another peer's subtree by subscribing to it" is the sentence §11.1a says does
not exist.** It exists, it is normative, and it is cross-impl verified.

**This is `AGENTS.md`'s L9 — *a deferral is a build-state claim and expires like one* — and it is not a
new finding: it was recorded as L9's fourth and load-bearing instance on 2026-08-17.** The rule went
into the ratchet; **the spec text was never corrected**, so nineteen days later the blocker is still
being inherited by a second document. **That is L13's fourth axis** (an arch-owned item whose remaining
work is an execution is an unfinished task, not a board row) **and L23's second-home problem**: the
finding was about RELAY, and REGISTRY §8.2 restates the deferral independently, so a fix to one leaves
the other standing.

> **What follows, stated carefully, because the honest claim is narrower than "unblocked."** Mode A
> still needs its own design — the `:subscribe` wire shape, aggregation semantics, retention (§8.1's
> *"unbounded by default is a known gap in the deferred mode"*). **What is void is the reason given for
> not doing it.** A deferral that says *"the substrate cannot do this"* and a deferral that says *"we
> have not designed this yet"* are different statements with different costs, and only the second one
> is true. **Registry federation is unblocked, not done.**

### §5.2 The web-native backends are "paper", and they are the on-ramp

`GUIDE-RESOLUTION` §6.4's own status table: **`dns-txt`, `well-known-url`, `did-web`/`did-key`** are all
**"paper — *working out the kinks*"**, and REGISTRY §12 defers the `domain-control` challenge format
explicitly so there is *"one domain-proof mechanism, not two."*

**This is where the operator's informal on-ramp lands** — *"if I already use ATProto and have a
following there, I put in my profile: here's where you find me on the entity system. That's the
authority."* Three things about it:

- **It needs nothing from us to work at all.** A claim in a foreign profile naming your peer-id is
  readable by anyone already following you there. **The reader's trust comes from the account they
  already follow, not from a protocol.**
- **It becomes checkable when it is reciprocal.** Your entity tree names your foreign handle, and the
  foreign profile names your peer-id. **Two claims that point at each other, verifiable from either
  end** — which is exactly `rel="me"` / IndieAuth's mechanism and Keybase's proof model, and it is
  ~200 LOC-shaped rather than a design problem.
- **NIP-05 is the same artifact one step more formal**, and `well-known-url` is one backend covering
  NIP-05, DID:web and WebFinger — already identified as *"highest coverage-per-effort in the naming
  stack."*

**And the operator's ruling should be recorded where a reader will find it:** *"We don't need DID:PLC.
Do we want to integrate with it? If you want to do it that way, go for it. Is it a requirement?
Definitely not, never has been, never will be."* **The corpus is already consistent with that** —
`did-web` and `did-key` are *consumed* as name shapes, and REGISTRY §12 lists presenting **as** a DID as
a deferred outbound bridge — but nothing states the position, and a reader of §4.1a's `did:web:*` row
could reasonably infer a dependency that is not there.

### §5.3 A resolver-config cannot be published

**The operator's *"I publish the registry stuff that I use, the names I use"* has no artifact.**
REGISTRY §4.3 gives `set-resolver-config` / `get-resolver-config` — **local operations on a local
entity** — and RELAY §6.4 is emphatic that a config is *"peer-local and not synced"*, which is correct
and is the property that stops an aggregator installing itself.

**Publishing one voluntarily is a different act and is not specified.** It is small: a resolver-config
is already an entity; publishing it is a convention plus a name, and **adopting someone else's is a
trust decision the reader makes** — the same shape as a block list, a ranking, or a mirror. **The rule
that keeps it safe is already written**: nothing may install itself into another peer's config, so a
published config is an *offer*, never an update.

**This is the highest-value small item in the document**, because it is what turns *"there will be many
registries"* from a coordination problem into the thing the corpus is already good at: **an opinion,
published, that others may adopt.**

### §5.4 The theory is nowhere, so the design reads as arbitrary

Four documents hold the answer and none of them says *why* it is the answer. **A reader who has not read
Stiegler sees a bag of name shapes; a reader who has sees Zooko's triangle solved the standard way.**
One paragraph in `GUIDE-RESOLUTION` §6 would make the whole section self-justifying, and it costs
nothing.

---

## §6 What this changes on the board

| Item | Change |
|---|---|
| **D-16 registry composition** | **Not unstudied.** Reduced from *research a design* to **four bounded items** — §5.1 correct the stale deferral · §5.2 the web-native backend family (already roadmap P5) · §5.3 the publishable resolver-config · §5.4 one paragraph of theory |
| **NEW — correct RELAY §11.1a and REGISTRY §8.2** | **arch-owned, execution.** A void blocker in two specs, recorded in the ratchet nineteen days ago and never folded. **Registry federation is unblocked, not done** |
| **D-25 the informal on-ramp** | Sharpened: the one-way claim needs nothing; **the reciprocal two-way claim is the checkable version** and is the `rel="me"` family; NIP-05 is the formal one and `well-known-url` covers three ecosystems |
| **Record the DID position** | Operator: not a requirement, never will be — consumed as a name shape, and presenting *as* a DID stays a deferred outbound bridge |
| **A directory of registries** | **Not a registry.** A published opinion, same as a block list — and it should be built that way or it becomes a root |

---

## §7 What this does not settle

1. **Mode A's own design is untouched here.** §5.1 voids the stated blocker and nothing more — the wire
   shape, aggregation semantics and retention are real work.
2. **No deployment has been observed.** Every claim above is from spec text; there is one registry
   deployment in the ecosystem and the multi-registry case has never been run.
3. **Squatting is out of scope by design** (REGISTRY §12, *"per-backend concern; substrate has no
   opinion"*) — **which is correct for the substrate and is not an answer for a user** who types
   `alice@somewhere` and gets an impostor. Stiegler's second half is the mitigation (*petnames prevent
   mimicry*) and **no client has been observed implementing it.**
4. **§5.3's publishable config has no security review.** *"Adopt this person's naming setup"* is a
   trust decision with real consequences, and nothing here works out what a consumer should be shown
   before accepting one.
5. **The NIP-05 and petname reads are single-source.** Stiegler is the canonical petname reference and
   was read directly; the Zooko framing around it is standard but was not separately sourced.

---

## §8 Sources

**Primary, fetched for this document:**
[Nostr — NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md) (`local-part@domain`,
`/.well-known/nostr.json`, *"not intended to verify a user"*, ***"clients must always follow public
keys"***) ·
[Marc Stiegler — *An Introduction to Petname Systems*](http://www.skyhunter.com/marcs/petnames/IntroPetNames.html)
(keys / nicknames / petnames; Zooko's triangle; *"forgery… mimicry"*).

**In-corpus, opened for this document:** `EXTENSION-REGISTRY` §1, §2.2, §4.1a, §4.1b, §4.3, §8.1–§8.3,
§12, §14 · `GUIDE-RESOLUTION` §6, §6.1–§6.4 (**the authority for the four name shapes**) ·
`EXTENSION-SUBSCRIPTION` v3.18 §6.1, §6.2, **§6.3** (the refutation) · `EXTENSION-RELAY` §11.1a, §6.4
(**the stale deferral, in two specs**) · `EXTENSION-REVISION` §6.3 · `PROPOSAL-APP-CONVENTION-FEED`
§2.4 · `EXPLORATION-THE-PUBLIC-SOCIAL-STACK-…` §5.3 · `ROADMAP-SOCIAL-TIER-CLOSEOUT` §3 (P5, P6).
