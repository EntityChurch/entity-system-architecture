# EXPLORATION — the public social stack: the complete map

**Status:** Exploration (design record). Not a proposal, not normative.
**Scope, deliberately: the PUBLIC case.** Confidential entries, session encryption, group key
distribution and membership'd conversations are real and are **out of scope here** — named in §8 with
what is already known about each. The public path is the one that has to be right first, because it
is the one that is copied.

**What this is.** The two companion explorations found the shape of the naming layer and the shape of
the republication network. This one assembles the **whole stack in one place**, layer by layer, rates
each layer by what exists, answers the seven questions any social system has to answer, works out the
**four content shapes** in enough detail to be proposable, and closes the two axes the corpus had
never covered: **moderation** and **the cost of participation**.

**This is the map. What is ready to become a proposal is §9.**

---

## §1 The stack

Read bottom-up. Each layer depends only on the ones below it, and **the top three are the ones that
do not exist.**

| # | Layer | What it does | State |
|---|---|---|---|
| **7** | **View** | timeline, thread rendering, ranking, search | per-application, never a contract |
| **6** | **Curation & exclusion** | what I republish, what I omit, whose omissions I inherit | 🔴 **unspecified — §5** |
| **5** | **Assembly** | the republished thread view; discovery via the reference graph | 🔴 **unspecified — §4.4** |
| **4** | **Content** | entry · reference · ordered index · context | 🔴 **EMPTY — §4** |
| **3** | **Reader loop** | the follow set, the cursor, refresh cadence, backoff | 🟡 designed, unbuilt |
| **2** | **Distribution** | signed root, `seq`, content-addressed walk, static hosting | 🟢 **deployed** |
| **1** | **Naming** | many inbound routes → one key; key → endpoints and claimed names | 🟢 designed; two gaps |
| **0** | **Identity & authority** | the key is the identity; grants are fine-grained and attenuable | 🟢 landed |

**The shape of the problem, stated once:** layers 0–2 are done and are the hard part. Layer 3 is
designed. **Layers 4, 5 and 6 do not exist, and they are the only ones a user ever sees.**

Nothing above layer 4 can be specified until layer 4 is, which is why the content vocabulary is the
keystone and everything else in this document is downstream of it.

---

## §2 The seven questions, and where we sit

Every social system answers these. Putting our answers beside the field's is the fastest way to see
what is genuinely different and what is merely ours.

| # | Question | The field's answers | Ours |
|---|---|---|---|
| 1 | **Who are you?** | server account · portable identifier over a ledger · a bare pubkey | **the key is the identity** — no authority, no registration |
| 2 | **How do I find you?** | one canonical name form per system | **a hub: many routes, one key** — routes are substitutable |
| 3 | **Where is your stuff?** | your server · your data host · whichever relays carry you | **wherever you put it** — content-addressed, so location is a hint |
| 4 | **What is a post?** | a typed record in a repo · an object wrapped in an activity · a signed event | ❌ **nothing** |
| 5 | **How do I know there is a new one?** | push to followers · firehose · poll relays | **pull a signed root, compare `seq`** — no per-follower state |
| 6 | **How does a conversation form?** | the server holds the thread | ❌ **nothing** — and §4.4 is our answer |
| 7 | **Who pays, and who moderates?** | instance admin · the company · relay operator | ❌ **unspecified** — §5, §6 |

**Three observations from the table.**

**(a) The infrastructure column is answered and the social column is not.** 1, 2, 3 and 5 are done.
4, 6 and 7 are empty. That is an unusual position: most systems ship the social layer first and spend
years fixing the infrastructure underneath it. We did the reverse, which is why the remaining work
looks small and is load-bearing.

**(b) Question 5 is where the model is structurally different and it is worth being precise.** Every
other system's publisher does work proportional to its follower count — deliveries to make, a
firehose to feed, relays to push to. **Ours does none: it writes to its own tree and stops.** A
million followers and one follower cost the publisher the same. The work moves to readers, who each
do a cheap check against a static origin. **That is RSS's economic shape, and RSS is the only system
in the survey that has lasted 25 years without a funding model.**

**(c) Question 7 has never been asked in this corpus at all.** The prior-art library says so about
itself — moderation *"is touched only in [the authority axis] … for any social tier this is a
first-class axis and it has no section here."* §5 and §6 below are that section.

---

## §3 What is already settled, so it is not re-litigated

Pulling forward from the companion work, because the map is misleading without it.

- **Naming is a hub, not a chain.** Many inbound routes converge on a self-certifying key; the route
  is discardable once it terminates. **Adding naming routes forever costs the layers above nothing.**
- **The reverse direction is two questions** — key → endpoints, and key → *what names it claims* — and
  both want **one self-published signed record**, with names verified in the forward direction rather
  than by a reverse index.
- **Domain loss costs a label, not an identity.** Followers follow the key. Host loss costs nothing.
  **Key loss is the only fatal row**, and the natural fix — a successor signed by the old key — is
  exactly what a compromised key must not be able to assert.
- **Ten social products reduce to six shapes**: Entry · Reference · Collection · Reaction · Relation ·
  Context. **Context and audience are orthogonal** and collapsing them is the expensive mistake.
- **Public following of a stranger is the static route only**, because a stranger has no grant.
- **The primary operation is the expensive one.** Measured on a real tree: verifying a known region
  costs about the tree's depth; *discovering what is new under a prefix costs the whole tree*, because
  the structure is keyed by hash bits rather than path locality. **An index at a known key is what
  fixes that**, and it is the reason §4.3 exists.

---

## §4 The four shapes, worked out

Enough detail to be proposable. Four shapes cover layers 4 and 5; the other two of the six (Reaction,
Relation) are thin or already owned elsewhere.

### §4.1 The entry

```
app/feed/entry
  author       the key — MUST equal the namespace it is authored under
  created_at   author-clock ms — a display heuristic, never a correctness authority
  body         an embed node — text, image, link, media, interactive
  reply?       { root, parent }  — present ⇒ this is a reply
  context?     a reference — what this is PART OF (not who it is for)
  prev?        this author's previous entry — the per-author causal chain
  attachments? content hashes, deduped by the store
signed by author; content hash is its permanent identity
```

**`body` is an embed node and this convention would define no content types of its own.** A photo
post and a text post are one shape with a different embed inside — which is exactly why the embed
convention was factored as foundational, and reusing it here is the payoff.

**`author` MUST equal the namespace.** An entry under someone else's namespace claiming a different
author is invalid. **This single rule is what keeps republication (§4.4) from being forgery.**

### §4.2 The reference — the shape everything points with

```
reference = { peer, path, hash }
             ↑     ↑     └─ WHAT it is — the assertion, self-describing, any hash format
             │     └─ where they put it — a HINT
             └─ who published it
```

**The hash is the claim; the locator is a convenience.** A consumer holding the bytes fetches
nothing. One that does not may get them **from anybody** — the author, a mirror, a cache, a stranger —
because the hash validates them regardless of source.

**This is derived from a split in the field, not invented.** One deployed lineage references a reply
by **location alone**, so a reply is only as trustworthy as whatever currently answers that location
and a parent can be edited underneath its replies. The other adds a **content hash to the locator**
for exactly that reason. We get the second free, and we add the `peer` term because that is the unit
that resolves through §3's hub.

**Threading carries `root` as well as `parent`.** With `parent` alone, assembling a thread is a hop-
by-hop walk and one unreachable author truncates everything below them. **`root` lets any holder of
any entry name the whole thread in one step**, which is what makes a partial view assemblable and a
mirror findable. The mail lineage reached this by carrying the entire ancestry chain; root+parent is
its bounded form.

**A thread needs no genesis entity: the root entry *is* the thread**, and its hash is the thread id.

### §4.3 The index — a bounded head page with an immutable back-chain

The site convention already forbids the obvious answer — a flat exhaustive collection — and it is
right: that is the download-the-index anti-pattern and it grows without bound. **This satisfies that
ruling rather than reopening it.**

```
app/feed/index-page          head page at a known key: /{peer}/app/feed/index
  entries     [reference]    newest first, bounded count
  prev?       content hash   the next-older page — IMMUTABLE, content-addressed
  updated_at
```

- Only the **head** is mutable. A full page is **never rewritten**, so a reader that has seen it never
  refetches it and dedup makes holding it free.
- A reader **fetches the head and walks back only as far as its cursor**, then stops.

**Cost is O(new), not O(all)** — the property the measurement said was missing, obtained with no
collection field and no per-prefix published digest.

**The index is an optimization and must not be the authority.** A reader that cannot fetch it falls
back to enumerating the prefix — slower, same answer. An entry absent from the index is still a valid
entry. That is the valid-floor discipline, and it also stops the index becoming a place to lie by
omission about your own posts.

### §4.4 The mirror — assembly as a by-product

```
app/feed/mirror
  subject      reference    the thread root this view is of
  entries      [reference]  what this mirror holds — republished, UNMODIFIED
  gathered_at
  gathered_by  the key that assembled it
```

**Four rules, each closing a specific hole:**

1. **Republish the original bytes.** Not a re-encoding, not a re-normalization. **A re-encoded entry
   no longer verifies against its author's signature**, which would silently turn a mirror from
   evidence into hearsay. Byte preservation is already the corpus's rule for cross-peer forwarding;
   here it is what the whole model rests on.
2. **A mirror cannot claim completeness**, and the shape gives it no way to. The verifiable property
   is precisely that **a mirror can omit but never substitute.**
3. **A mirror is not authorship.** The gatherer signs the *mirror record*; each entry keeps its own
   author's signature. Attribution follows `entry.author`, always.
4. **Unknown fields survive**, automatically, because bytes are republished rather than re-serialized.
   An old implementation mirroring a new entry cannot strip what it does not understand.

**Rule 3 is the answer to "is republishing someone else's post legitimate?" and the answer is
structural rather than a norm.** You do not hold their private key. You cannot author in their name.
You cannot alter a byte without the signature failing. **All you can do is carry what they already
chose to publish** — which is what publishing has always meant on the open web, and which the format
makes verifiable rather than merely customary.

---

## §5 Moderation, worked out — and it is exclusion, not removal

**This axis has no section anywhere in the corpus.** The prior-art library names its own absence. This
is the first pass at it.

### §5.1 The constraint, stated honestly first

**Nobody can remove anything from anyone else's tree.** Not the author of the thread, not a mirror
operator, not us. That is not a policy choice — it is what content-addressed self-hosted publishing
*is*, and any design that pretends otherwise is lying to its users.

So the question is not *"how do we take things down"* but **"what does a participant actually
control?"** Three things, and they are enough:

1. **What I fetch.**
2. **What I render.**
3. **What I republish.**

### §5.2 Blocking is exclusion, and the model already supports it natively

**A mirror can omit but never substitute.** Omission is therefore the *native* operation of the
assembly layer — the one thing a mirror is structurally permitted to do. **So blocking is not a
feature bolted onto this architecture; it is the default capability of it.**

Concretely, blocking a key means: I stop fetching from it, I stop rendering it, and **my mirrors omit
it**. My thread views are assembled without it.

**The consequence that makes this work socially:** anyone who reads the network partly through my
mirrors **inherits my exclusions by default** — not as censorship, but as the ordinary result of
reading a view I assembled. And because a mirror can never substitute, **my exclusions are visible**:
compare my mirror with anyone else's and the difference is exactly what I left out.

### §5.3 Curation and moderation are the same act, and following is already subscribing to it

This is the result worth carrying. In a republication network, **choosing what to republish IS a
moderation decision**, and choosing whose mirrors to read **IS choosing whose moderation to accept.**
There is no separate moderation system to design, because the assembly layer already is one.

**The nearest prior art is a labelling service model** — a third party publishes moderation opinions
as data and clients subscribe to whichever they trust. That is the right shape and ours is a stronger
version of it: **the opinion is expressed by inclusion rather than by a parallel label stream**, so it
is self-evidencing and needs no second vocabulary.

**Shared block lists fall out for free.** A published list of excluded keys is ordinary data; anyone
may subscribe to someone else's; nobody is forced to. That is the pattern that demonstrably works in
practice, and it needs nothing new.

**Compare the field, briefly:** instance-level defederation is an admin decision applied to all that
instance's users, opaque to them and unappealable; relay-level filtering is similar with more churn;
a labelling service is subscribable and transparent but is a separate service someone must run.
**Ours is per-participant, transparent, subscribable, and requires no operator at all.**

### §5.4 What exclusion does not solve — stated plainly

- **Exclusion is not removal.** A determined reader can always reach the original. This is the open
  web's property and it is not recoverable within this architecture.
- **It does not address a direct channel.** Unwanted material arriving in an inbox is a delivery-layer
  problem with a different answer — that layer has authority gates and this one does not.
- **There is no global takedown, for anything, ever.** Illegal content has no mechanism here. **This
  is the honest hard limit of the model** and it should be stated in any public document rather than
  discovered. What exists instead is that every *host* remains subject to its own jurisdiction, and a
  publisher who loses hosting loses reach — which is a real consequence, just not a protocol one.
- **Nothing stops someone mirroring you specifically to keep your words alive against your wishes.**
  A revision can be published; the old bytes remain valid. This is a genuine and permanent property.

### §5.5 The accountability model, since it is the other side of the same coin

**Responsibility sits with the publisher**, because the identity is a key the publisher holds and the
reach is hosting the publisher pays for. There is no platform interposed, which is the point — and
**the honest counterpart is that there is also no platform to appeal to.** No account restoration, no
support queue, no reversal of someone else's exclusion decision. Both halves are the same design
choice and a document that claims the first without the second is selling something.

---

## §6 What it costs to participate — the axis that decides whether any of this survives

The prior-art library's economics axis has the finding this whole architecture should be read
against:

> **The recurring failure is not technical.** In every federated system surveyed, the most common way
> a user loses data or reach is that **the volunteer running their tier stopped paying for it.** RSS
> is the exception, and it is the exception because the tier is a plain web server the publisher
> already had for other reasons.
>
> **Design implication:** an always-on tier that is *also useful for something else* survives; one
> that exists only to serve the network is **a donation with a half-life.**

**Our always-on tier is a static website.** The same bucket that serves someone's site serves their
feed, because they are the same tree. **So this inherits RSS's survival property rather than the
instance-churn property** — and that is not a claim about our design being better, it is a claim about
which side of the surveyed line it falls on.

### §6.1 The cost floor, honestly

| What | Cost | Required? |
|---|---|---|
| A key | free | **yes** — it is the identity |
| Somewhere to serve bytes | free tier → cheap object storage | **yes** |
| A domain | roughly ten dollars a year | **no** — see below |
| A server | — | **no** |
| Per-follower cost | **zero** | — |

**A domain is optional and this matters more than it sounds.** The identity is the key. A domain is
one *route* into the hub and there are others: a hosting provider can supply the name, a registry can
issue one, a petname works locally, a key can be handed over directly. **Someone with no domain and a
free static host is a full participant with a durable identity** — and if they later buy a domain,
nothing about their identity or their followers changes, because the name was never the identity.

### §6.2 What happens if the free tier goes away

The fair form of the worry: *free static hosting is free now because it is cheap for providers; would
it stay free if everyone used it?*

**Three reasons the exposure is smaller here than for any system in the survey.**

1. **There is no per-user server cost to a host.** A static bucket costs bytes stored and bytes
   served. There is no application, no database, no per-account overhead — which is precisely why free
   tiers exist for it and why the economics do not invert with popularity the way an instance's do.
2. **The cost scales with what you publish, not with who reads you** — and reads are cacheable by any
   intermediary, because content-addressed bytes are safe to cache forever. Popularity is served by
   the cache tier, not by the publisher's wallet.
3. **Moving is nearly free.** The tree is content-addressed and signed, so relocating it changes no
   hash, invalidates no reference, and requires no follower to do anything. **Hosting is a commodity
   here in a way it is not anywhere else in the survey** — which is exactly the leverage that keeps it
   cheap.

**And the mirror tier is the quiet answer to durability.** Anything republished by others survives its
author's host going away, verifiably. **A well-discussed thread is more durable than a quiet one** —
which is an unusual property and worth noticing: the network's memory is strongest exactly where
people engaged most.

**The honest residue:** if every free tier vanished, participation would cost a few dollars a year for
storage. That is a real barrier for some people and it is smaller than every alternative in the
survey, none of which is free either — they are subsidised by somebody, which is the failure mode.

---

## §7 The thesis, stated with its limits

**What you stop needing a company for:** an account · a server · a name they control · a feed
algorithm · their delivery infrastructure · their moderation decisions applied to you · their
advertising interposed in your exchanges. All of it becomes *applications that publish state*, over a
substrate where identity is a key you hold and reach is hosting you rent from anyone.

**Four honest limits, so the claim is defensible rather than promotional:**

1. **Network effects are real and we have none.** The deployed systems have tens of millions of users
   between them. A better architecture does not move people; **what it can do is lower the cost of
   leaving**, which is a slower and more durable mechanism than winning.
2. **These are not novel parts.** Content addressing, signed mutable pointers and keys-as-identity all
   exist in the field. **The contribution is the synthesis plus posture mobility** — the same state
   being a live peer, a passive hosted store, or dormant, without changing what it is or who may read
   it.
3. **Simplicity is a feature we do not yet have.** The deployed systems work today and are easy to
   join. Ours is a stack of six layers, three of which do not exist.
4. **The hard social problems remain hard.** §5.4 is not a rough edge; it is the permanent shape of
   a system with no central authority. We should say so where anyone can read it.

---

## §8 Deliberately out of scope here, with what is already known

**Named so the map does not read as complete when it is not.**

- **Confidential entries.** Known: the format needs no change — an entry whose body is encrypted is
  the same entry, and nothing this layer reads lives inside the body, so a peer can index, thread and
  mirror encrypted entries without reading them. **A confidential entry can therefore be hosted
  anywhere hostile and republished by anyone, and remain unreadable.** Key distribution is the
  authority layer's problem, not the format's.
- **Membership'd conversations.** Already drafted elsewhere, with the same union-of-participants
  model, an explicit roster op-log, and delivery instead of publication. **The public and private
  models are the same shape with three differences** — an explicit context genesis, delivery rather
  than publication, and a bare hash rather than a full reference because participants are enumerable.
  Those two drafts should share the reference shape, and saying so now is cheap where later it is a
  migration.
- **Session encryption, group key rotation, forward secrecy.** Untouched. Real, and downstream of
  the confidential-entry decision rather than blocking it.
- **Key-change continuity.** The one genuinely hard unsolved problem — a successor signed by the old
  key is the natural shape and is exactly what a stolen key must not be able to assert.

---

## §9 The gap list, ranked — and what is ready to propose

| # | Item | Layer | Ready? |
|---|---|---|---|
| 1 | **The content vocabulary** — entry, reference, threading, context | 4 | ✅ **§4.1–4.2 are proposable now** |
| 2 | **The paged index** | 4 | ✅ **§4.3 is proposable now** |
| 3 | **The mirror convention** | 5 | ✅ **§4.4 is proposable**, and it is the newest material — its vector set is the gate |
| 4 | **A compatibility contract for the app vocabulary** | 4 | ✅ **ready, and probably belongs in the domain charter** rather than one convention |
| 5 | **The self-published peer record** (endpoints + claimed names) | 1 | ✅ widen the existing draft |
| 6 | **Bidirectional name verification** | 1 | ✅ small; the rule is already written by others |
| 7 | **`well-known-url` resolution** | 1 | ✅ build item, already ranked highest value-per-effort |
| 8 | **The follow loop** | 3 | 🟡 drafted; unblocked once (1) lands |
| 9 | **Exclusion + published block lists** | 6 | 🟡 **§5 is the first pass**; needs its own document |
| 10 | **Retention** | 5 | 🔴 unowned, and §4.4 makes it load-bearing rather than tidy-up |
| 11 | **Key-change continuity** | 0 | 🔴 unsolved |

**Items 1–4 are one coherent proposal** and it is the keystone: nothing at layers 5, 6 or 7 can be
specified without them, and they are the formats a third party copies, so they are worth more care
and less speed than anything else on the list.

**Two rules that should travel with that proposal**, because both were learned the expensive way:
**ship conformance vectors** — the one convention that shipped without them diverged between two
implementations within six days, and the divergence was undetectable from either side because neither
ever held the other's entity; and **state the compatibility contract** — *all old data must remain
valid under the updated vocabulary, and new data must be valid under the old.*

**Still owed before the mirror convention is normative:** the newsgroup flood-fill lineage and the
gossip-first social protocol, both already on this corpus's own unread list, and both squarely about
a network that assembles itself from partial views. That list has now predicted its own relevance
three times.
