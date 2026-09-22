# EXPLORATION — evolvable schemas, and the vocabulary problem

**Status:** Exploration (design record). Not a proposal, not normative.

**Operator-requested:** *"evolvable data format schemas… how L5 layered apps coordinate, how they
agree, where they're allowed to disagree. You want to run a different client renderer — that's zero
cost. If you want to change the data model, that's more serious, because if we don't have agreement
there we can't communicate at all — and then we need bridges and translation mechanisms, which we want
to minimize."*

**The operator's framing is exactly the right decomposition and it has a name in the literature.**
There are **two** problems here that get conflated, they have different answers, and the field has a
verdict on each:

| | The **encoding** problem | The **vocabulary** problem |
|---|---|---|
| Question | *my parser meets a field it does not know* | *your `post` and my `post` are different things* |
| Failure | crash, or silent data loss | mutual unintelligibility with no error |
| Solved by | protobuf/Avro/Thrift, thirty years, **solved** | **nobody. This is where every decentralized system has bled.** |
| Our state | **strong — stronger than most of the field** (§2) | **partial — the largest open risk in the arc** (§4) |

**The result up front:** the encoding tier is in good shape and this read found **one real defect**
(§3, a MUST/SHOULD split across four normative homes for a single rule). The vocabulary tier is where
the work is, three deployed systems have paid for the lesson, and **their mitigations come in exactly
three layers of which we currently have one and a half.**

**Sourcing discipline (D12).** Every claim with a URL was opened. Claims about our own corpus cite
`(file, section)` at `entity-core-protocol` **`221d8c3`** (2026-09-03), read this session.

---

## §1 The encoding tier — what thirty years settled

### §1.1 The two mechanisms, and why we are in the first family

**Tag-based (Protobuf, Thrift).** Each field gets a manually assigned number; tags and wire types are
in the encoding, so a parser can **skip** what it does not know.

**Schema-resolution (Avro).** No tags. The reader holds both the writer's schema and its own and
resolves between them, which needs the writer schema available — hence a registry.

**We are neither, and that is worth naming.** Our entities are **open maps with string keys**
(`ENTITY-CORE-PROTOCOL` §2.10) carried in canonical CBOR, and the type is *itself a content-addressed
entity* that can be fetched. **That is closer to Avro's model — a resolvable schema — with the
registry problem dissolved, because a content hash is a schema reference that needs no registry and
cannot drift.** *(our reading)* It is a genuinely unusual position and it is a good one.

### §1.2 The hard-won rules, and where each one lands for us

| Field's lesson | Source | Our position |
|---|---|---|
| **Tag reuse is silent data corruption.** Reusing a retired field number makes old data deserialize into the wrong field. Protobuf added `reserved` for this | Kleppmann; Google's *Updating a Message Type* | **Not applicable — we key by string name**, so the failure needs a *name* to be reused. Our analogue is **repurposing a field name**, and it is already forbidden by charter #7 as drafted: *"a field's meaning is never narrowed and a tag is never repurposed"* |
| **`required` is un-removable in practice.** proto3 removed the keyword entirely | Protobuf | We have `optional: true` in field-specs and no `required` keyword. **The equivalent trap is a field a consumer treats as mandatory although the schema does not** — unmeasured |
| **Defaults are the load-bearing mechanism.** Adding or removing a field with a default is fully compatible; without one, removal breaks | Avro + Protobuf | **We have no defaults mechanism in field-specs.** Absent/null is a *content-hash-distinguishing* fact for us (`ENTITY-NATIVE-TYPE-SYSTEM` §2.4), which is a stronger constraint than either. **Open — §5.3** |
| **Compatibility direction dictates deploy order.** BACKWARD ⇒ new consumers first; FORWARD ⇒ new producers first; FULL removes the ordering constraint | Confluent | **This is L21's third shape in the vocabulary of another field.** Our *divergence unit* rule (*name what refuses what between the first seat landing it and the last*) is the same discipline, reached independently, and the field's version is sharper: **the deploy order is derivable from the compatibility mode.** Worth adopting the framing |
| **BACKWARD checks only the previous version; BACKWARD_TRANSITIVE checks all.** Bites teams replaying long-retention data | Confluent | Directly ours: **a mirror replays old entries forever.** Our compatibility contract must be **transitive**, not pairwise, and charter #7 as drafted does not say which |
| **Enum/union narrowing breaks consumers** who still meet the removed member | Avro/Protobuf | Ours is a `type_filter` / tag set. Same rule |
| **Skipping ≠ preserving.** Many Thrift implementations historically *dropped* unknown fields, silently corrupting read-modify-write proxies. proto3 (≥3.5) retains them | Kleppmann; Protobuf history | **This is the one that matters most to us and it is §2 and §3** |

### §1.3 The `MUST-ignore` debate, and the invariant it produces

The IETF/W3C argument is live and useful. **For:** requiring unknown elements be ignored is the
simplest *substitution mechanism* — the thing that lets a V1 consumer turn a V1+extensions instance
into a V1 instance — and it is why HTML and HTTP headers evolved at all.

**Against, with real incidents behind each:**

1. **Unbounded tolerance for garbage.** Poul-Henning Kamp on HTTP/2 §5.5: *"such unlimited tolerance
   for what might be plain garbage seems unwise — a surprisingly large percentage of random garbage
   runs straight through that clause."*
2. **State desynchronization.** If an ignorable frame can carry state, ignoring it desynchronizes
   sender and receiver. Martin Thomson's resolution is the invariant: *if MUST-ignore, then unknown
   elements **cannot modify session/connection state**.*
3. **Extension smuggling.** An intermediary must not re-advertise or copy through flags it does not
   understand — *"you might be signing on to something you don't understand."*
4. **Security domains.** WS-Security specified that **all** extensions must be understood, with no
   substitution mechanism, because ignored data can be catastrophic there.
5. **Bloat as a DoS vector.** An unknown field full of padding stays valid and gets relayed.

> **The distilled rule, and it is the checkable one:** *`MUST-ignore` is safe only when paired with an
> explicit invariant that ignorable elements cannot carry state-changing semantics — plus a resource
> bound, so that "ignore" does not mean "accept unbounded work."*

**Checked against our corpus, and we hold two of three.**

- **The state invariant: we have it.** `ENTITY-NATIVE-TYPE-SYSTEM` §2.6.x (security): *"An attacker
  could include fields not defined in the type to influence handler behavior. Handlers MUST validate
  only known fields from the type definition and SHOULD ignore unknown fields for security-sensitive
  decisions."* That is Thomson's invariant, stated normatively, reached independently.
- **Smuggling: we are structurally immune.** Our forwarding rule is byte preservation, not
  re-advertisement — an intermediary carries the original bytes and the signature, so there is nothing
  to selectively copy through.
- **The resource bound: not found this session.** Unknown fields are hashed into the content hash and
  therefore counted by whatever size limits apply to an entity, so the bound may exist *incidentally*
  via entity size limits. **Whether an explicit bound exists on unknown-field content was not
  established** and is filed as §5.1 rather than claimed either way. *(A negative here would be an
  L16-class claim and this session did not do the exhaustive search that would license one.)*

---

## §2 Where our encoding tier is genuinely ahead

Three things, and they are not small.

**(1) Unknown-field preservation is a *correctness* requirement here, not a courtesy.** Everywhere
else in the field, dropping an unknown field costs you data. Here it changes the **content hash**, and
`ENTITY-CORE-PROTOCOL` §2.10 says so: *"content hashing covers all of `{type, data}` including unknown
fields… A peer that strips or rejects unknown fields produces different hashes for the same entity,
breaking content addressing for all downstream peers."* And `ENTITY-NATIVE-TYPE-SYSTEM` §2.4's framing
is the sharpest sentence in either repo on this: *"**Entity fidelity — preserving what you receive,
including unknown fields — is more fundamental than type conformance. A peer that strips unknown
fields is broken; a peer that doesn't validate types is merely Level 0.**"*

**That inverts the usual priority order and it is right.** In Kafka-land, schema validity is the gate
and fidelity is best-effort. Here fidelity is the gate.

**(2) The property/mechanism split in `ENTITY-CBOR-ENCODING` §5.4 is better engineering than the
field's.** It states the observable property — *a forwarded entity re-presents the exact bytes that
hash to its validated content hash, and all received content (known fields, unknown fields, CBOR tags,
null-vs-absent) survives the round-trip* — then permits **two** mechanisms: store-and-forward original
bytes (unconditionally robust), or lossless-parse-plus-canonical-re-encode **iff** receipt validation
is a strict re-encode-and-compare **and** the parsed representation is lossless at every nesting level.
And it makes the precondition **testable**: re-encode implementations MUST round-trip every
`encode_equal` vector byte-identically.

**That is exactly the Thrift hazard, correctly diagnosed and gated**, and the field does not have an
equivalent.

**(3) `null` vs absent is a first-class distinction.** *"Absent and null produce different CBOR bytes
and therefore different content hashes"* (`ENTITY-NATIVE-TYPE-SYSTEM` §2.4). Avro's null-union
convention exists to paper over exactly this and produces a well-known evolution gotcha (the default
must conform to the *first* branch of the union). **Ours cannot paper over it and therefore cannot get
it wrong silently.**

---

## §3 The defect this read found — one rule, four homes, two strengths

**Measured at `entity-core-protocol` `221d8c3` (2026-09-03), by reading all four sites.**

| # | Home | Text | Strength |
|---|---|---|---|
| 1 | `ENTITY-CORE-PROTOCOL` **§2.10** | *"Unknown fields **MUST** be preserved on storage and forwarding."* + the correctness rationale | **MUST** |
| 2 | `ENTITY-NATIVE-TYPE-SYSTEM` **§2.4** | *"Unknown fields **MUST** be preserved."* | **MUST** |
| 3 | `ENTITY-CORE-PROTOCOL` **§1.8** step 5 | *"**Preserve unknown fields**: **SHOULD** preserve fields not understood."* | **SHOULD** |
| 4 | `ENTITY-CBOR-ENCODING` **§5.4** step 5 | *"**SHOULD** preserve unknown fields"* | **SHOULD** |

**Site 4 is the one that matters, because §5.4 declares itself the canonical home:** *"This section is
the canonical home of the entity-fidelity contract… an implementation or validator citing any older
location should cite `ENTITY-CBOR-ENCODING.md` §5.4."*

**So the document that claims authority over the rule carries the weaker form of it, and the
correctness argument for the stronger form lives in a different document.**

**It also disagrees with itself.** §5.4's own governing paragraph states the property unconditionally
(*"all received content — known fields, unknown fields, CBOR tags, null-vs-absent — survives the
round-trip"*) and makes losslessness a **precondition (b)** of the re-encode mechanism. So the prose
binds and the numbered list does not, four lines apart.

### §3.1 Why this is not merely cosmetic, and how far the exposure actually goes

**Under mechanism A (store original bytes, forward original) the SHOULD is close to vacuous** — the
bytes carry unknown fields whether or not the parser noticed them.

**Under mechanism B (lossless parse + canonical re-encode) it is load-bearing**, and clause (b) is the
only thing binding it. An implementer reading the numbered list — which is *"where implementers read"*,
in §5.4's own words about a different rule — could take mechanism B and treat losslessness as a
SHOULD. The result is a peer that accepts an entity, re-encodes it without a field it did not model,
and **publishes a different content hash for what it believes is the same entity**, which §2.10
correctly calls *breaking content addressing for all downstream peers*.

**We already have the app-tier vector for it:** `PROPOSAL-APP-CONVENTION-FEED` **FEED-6** — *"an entry
carrying an unknown field, mirrored, round-trips byte-identical."* **The rule is gated at the
application tier and stated as a SHOULD at the core.** That is the wrong way round.

### §3.2 The disposition — the fix, the owner, the size

**This is L23's shape** (a rule has every normative home it is stated in, and the enumeration is by the
rule's *subject*) and **L23's fourth shape** specifically (a restatement that does not name its
authority, so the divergence is invisible from either copy).

**The fix, precisely:** raise sites 3 and 4 to **MUST**, and add the fourth shape's pointer — each of
sites 1, 2 and 3 names `ENTITY-CBOR-ENCODING` §5.4 as the authority, so the next sweep is a grep
rather than a search.

- **Owner: arch.** Both files are in `entity-core-protocol`, which is ours.
- **Size: four edits and a `0.8.2.x` fourth-component bump.**
- **Gate: proposal-first (L1).** This is a normative change to the core spec — it is a decision, not an
  execution, so it does not discharge under L13's fourth axis. **The proposal is small and is arch's to
  write.**
- **Routing when it lands: L21's fifth shape applies** — the fold's audience is every seat that
  implements entity fidelity, which is all three ground-up trees plus keystone's generated cohort,
  and the relay is scoped by the diff, not by what each seat shipped.
- **Before writing it, L23's ratified enforcement point binds:** enumerate the homes **by subject**
  (*"what obligation exists about content a peer did not model"*), whole-document, across every
  `specs/` file in **both** repos plus every conformance/MUST list. **The four above are what one
  session's grep found; they are not certified to be all of them.**

---

## §4 The vocabulary tier — where everyone bleeds, and the three-layer answer

**This is our §8.1: *"whether independent people adopt one vocabulary is a social question and it is
the open one."*** Three deployed systems have now answered it, and the answers agree.

### §4.1 Scuttlebutt — the failure, unmitigated

*"Community splintering arose from **uncoordinated evolution of message schemas**, potentially
isolating subgroups."* SSB had a strong cryptographic layer and no vocabulary governance, and it
fragmented. **That is the null case and it is the one to avoid.**

### §4.2 ATProto Lexicon — the strongest governance model in deployment

**The core rule, and its stated reason, is our situation exactly:**

> Lexicons may change *"within some bounds to ensure both forwards and backwards compatibility. The
> basic principle is that all old data must still be valid under the updated Lexicon, and new data
> must be valid under the old Lexicon."* Non-optional fields cannot be removed; retain removed fields
> marked deprecated.
>
> **There is no in-place mechanism for a breaking change.** *"If larger breaking changes are
> necessary, a new Lexicon name must be used"* — because *"given the distributed storage model of
> atproto, developers **do not have a reliable mechanism to update all data records in the
> network**."*

**That last clause is the whole argument and it is ours verbatim.** We cannot rewrite other people's
trees either. **Independent derivation of charter #7's *"a breaking change takes a new type tag."***

**Four further findings we should take:**

1. **When a Lexicon freezes:** *"At a minimum, public adoption and implementation by a third party —
   even without explicit permission — indicates the Lexicon has been released and should not break
   compatibility."* **A cleaner freeze rule than anything in our charter draft**, and it puts the
   trigger on *someone else's* action, which is the honest place for it.
2. **Experimental namespaces are declared in the name**, e.g. a `.temp.` segment, so a schema can
   develop in the live network without committing. **We have no such convention** and our seats are
   building in the live network right now.
3. **Version-in-the-name was proposed and rejected by their community**: *"having the version in the
   NSID is not a great idea… the version should instead control handling, rendering, and response."*
   **Worth knowing before we reinvent it**, since `app/feed/entry-v2` is the obvious wrong move.
4. **The sidecar pattern is the alternative to versioning**, and it is the interesting one: a second
   record at the same key adds fields *without modifying the original lexicon*, and can be included in
   API responses. Their own open questions on it — *"how to handle versioning, how to prevent spam
   sidecar records, and what the governance model is for shared community namespaces"* — are worth
   inheriting rather than rediscovering.

**And an honest counterweight, because their model is not vindicated:** a published critique notes
consumers are effectively always on the *latest* Lexicon, so publishing a new version means consumers
adopt it and can break from factors outside their control — **and Bluesky itself has shipped breaking
changes** between `#savedFeedsPref` variants. *The rule is right and compliance is hard even for its
authors.*

### §4.3 Nostr — the three-layer mitigation, and the tell

Nostr encodes anti-fragmentation norms in its acceptance criteria: a NIP should be *implemented in at
least two clients and one relay*, *optional and backwards-compatible*, and — the key line —
**"There should be no more than one way of doing the same thing."**

It has **three layers**:

1. **A normative rule** — one way to do each thing.
2. **A registry** — `registry-of-kinds`, machine-readable, to prevent collisions. Though the README
   now concedes *"this table is not exhaustive"* and points at *"alternative registries following the
   same YAML schema"* — **the schema is the coordination point, not the registry.**
3. **Runtime fallbacks** — **NIP-31**: a custom kind carries an `alt` tag with a human-readable
   summary *"so a user who knows nothing about the kind can understand it"*; **NIP-89**: handler
   discovery, routing an unknown kind to an app that understands it.

**And the structural reason Nostr needs all three: kinds carry no self-describing semantics.** An `r`
tag means one thing in kind 1 and something else in kind 10002, so tag vocabulary is scoped per-kind
and two groups minting different kinds for one use case produce **mutually unintelligible**
vocabularies.

> **The line worth carrying, and it is the honest read of their own docs: *the existence of the third
> layer is the tell that the first two do not fully hold.*** Every decentralized system that has run
> long enough has built a graceful-degradation path for vocabulary it does not know. Nobody has
> prevented the divergence.

### §4.4 Matrix — versioning the *rules*, not the data

Matrix's answer is **room versions**: a room *upgrades*, versions *"aren't ordered or hierarchical"*
(a room can technically go from 2 to 1), and a version bundles a whole rule-set — v12 exists to switch
to State Resolution v2.1. **This is the model for versioning *behaviour* rather than *shape*, and it
is the one that fits our `[OPEN-FEED-6]` question** (does a chat message unify with an entry): the
thing that needs to agree is *how a consumer must behave*, which is our taxonomy floor's test already.

---

## §5 Where we stand, and what is owed

### §5.1 The scorecard against the three-layer model

| Layer | Nostr | ATProto | **Us** |
|---|---|---|---|
| **1. Normative rule** — one way to do each thing; no in-place breaking change | ✓ (NIP criteria) | ✓ (Lexicon evolution rules) | **drafted, not landed** — charter #7 (compatibility) and #8 (the growth rule) are D3/D8 in `PROPOSAL-APP-CONVENTION-FEED` §8/§9 and are **not in `CHARTER.md`** |
| **2. Registry / collision prevention** | ✓ (registry-of-kinds, incomplete) | ✓ (NSID + DNS-rooted authority) | **✓ and better** — our types are **content-addressed resolvable entities**, not opaque integers. A type is fetchable and self-describing, which is precisely what Nostr's kinds are not |
| **3. Runtime fallback** | ✓ (NIP-31 `alt`, NIP-89 handlers) | ~ (deprecation, sidecars) | **✓ and better** — `APP-CONVENTION-EMBED`'s **mandatory non-empty `fallback`** plus the §6 ladder plus *"unknown `format` → drop clean, never show source text"*. Anti-graveyard §8 already forbids the empty-fallback failure |
| **Experimental namespace convention** | reserved kind ranges | `.temp.` in the NSID | **none** |
| **Freeze trigger** | de facto | *third-party implementation, even without permission* | **none stated** |
| **Transitivity of the compatibility contract** | n/a | implied | **unstated in the #7 draft** |

**So: layer 2 is ours and is the field's best; layer 3 is ours and is the field's best; layer 1 is
drafted and unlanded.** That is a better position than the scorecard's first reading suggests — **but
layer 1 is the one that governs the other two**, and charter #7/#8 sitting in a proposal is exactly
`BEARINGS-2026-09-04` §4.1's *"both belong to the domain, not to one member."*

### §5.2 The four additions this read earns for charter #7/#8

Stated so that whoever writes the charter edit does not have to re-derive them. **Not proposed here**
(L1); these are inputs to the proposal that already exists as D3/D8.

1. **A freeze trigger.** ATProto's is the best available: *a convention's vocabulary freezes when a
   third party implements it, with or without permission.* Ours currently freezes on nothing.
2. **Transitivity.** Confluent's `BACKWARD` vs `BACKWARD_TRANSITIVE` distinction, and the reason it
   bites: **long-retention replay.** A mirror replays old entries forever, so our contract must be
   transitive against *all* prior versions, not the previous one. **Say which.**
3. **An experimental-namespace convention.** Both prior arts have one and our seats are building in
   the live network today. A `app/feed/x-…` or equivalent segment costs nothing now.
4. **Version-in-the-name is a known wrong answer.** Record it, with ATProto's community reasoning, so
   nobody proposes `app/feed/entry-v2` as an improvement. **The version controls handling, not
   identity.**

### §5.3 Open questions

1. **Is there a resource bound on unknown-field content?** §1.3's third invariant. Possibly satisfied
   incidentally by entity size limits; **not established**, and establishing it is an exhaustive search
   across both repos, not a grep.
2. **Do we need defaults in field-specs?** The field says defaults are what make add/remove
   compatible. We have `optional: true` and a hash-distinguishing null/absent rule, which is stricter
   and may make defaults *impossible* rather than merely absent — **a default that a consumer
   materializes changes no bytes, but a default that a producer materializes changes the hash.**
   That asymmetry is worth a paragraph somewhere and has none.
3. **`[OPEN-FEED-6]` in this light.** Matrix's room-version model and our taxonomy floor ask the same
   question — *must a conformant consumer behave differently?* If the two seats answer that question
   for chat-vs-entry, the tag question answers itself. **Worth relaying to both seats as the framing**,
   since it converts a naming argument into a behavioural test.

---

## §6 Sources opened

**Encoding tier:**
[Kleppmann — *Schema evolution in Avro, Protocol Buffers and Thrift*](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html) ·
[Confluent — Schema Evolution & Compatibility Types](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html) ·
[INNOQ — Schema evolution with Apache Avro](https://www.innoq.com/en/blog/2023/11/schema-evolution-avro/) ·
[Conduktor — schema evolution best practices](https://www.conduktor.io/glossary/schema-evolution-best-practices)

**The MUST-ignore debate:**
[Poul-Henning Kamp on HTTP/2 §5.5, ietf-http-wg](https://lists.w3.org/Archives/Public/ietf-http-wg/2017JanMar/0316.html) ·
[Martin Thomson on MUST-ignore and session state](https://lists.w3.org/Archives/Public/ietf-http-wg/2013AprJun/0395.html) ·
[XML.com — *Extensibility, XML Vocabularies, and XML Schema*](https://www.xml.com/pub/a/2004/10/27/extend.html) ·
[TLS 1.2 servers ignoring supported_versions (s2n-tls #4240)](https://github.com/aws/s2n-tls/issues/4240)

**Vocabulary tier:**
[ATProto — Lexicon spec](https://atproto.com/specs/lexicon) ·
[ATProto — Lexicon Style Guide](https://atproto.com/guides/lexicon-style-guide) ·
[Lexicon versioning discussion (lexicon-community #30)](https://github.com/orgs/lexicon-community/discussions/30) ·
[Draft Lexicon Style Guide — "Lexinomicon" (atproto #4245)](https://github.com/bluesky-social/atproto/discussions/4245) ·
[Lexicon versioning schemes — critique](https://verdverm.com/topics/atproto/lexicon/lexicon-versioning-schemes) ·
[Nostr NIPs README — acceptance criteria, kind registry](https://github.com/nostr-protocol/nips/blob/master/README.md) ·
[NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) ·
[NIP-31 — dealing with unknown event kinds](https://nips.nostr.com/31) ·
[Matrix — Room Versions](https://spec.matrix.org/latest/rooms/) ·
[SSB — schema fragmentation, ICN 2019](https://conferences.sigcomm.org/acm-icn/2019/proceedings/icn19-19.pdf)

**Our corpus, read at `entity-core-protocol` `221d8c3`:**
`ENTITY-CORE-PROTOCOL` §1.8, §2.10 · `ENTITY-CBOR-ENCODING` §5.4, Appendix E ·
`ENTITY-NATIVE-TYPE-SYSTEM` §2.4, §2.6-security · `APP-CONVENTION-EMBED` §4–§6, §8 ·
`PROPOSAL-APP-CONVENTION-FEED` §8, §9, FEED-6

## §7 What is unread

- **Cap'n Proto and FlatBuffers** — the zero-copy family, whose evolution rules differ from Protobuf's
  in ways that may matter for a canonical-encoding substrate. Not opened.
- **ASN.1's extension marker (`...`)** — the oldest MUST-ignore mechanism in the field, forty years of
  telecom deployment behind it, and the one most likely to have already solved §5.3's default question.
  **Highest-value single omission in this document.**
- **JSON-LD / RDF vocabulary governance** — the only prior art for *decentralized* vocabulary minting
  with no registry at all, which is nominally our model. Named in our corpus twice and never read.
- **Protobuf's own `reserved` history and the proto2→proto3 migration retrospectives** — read here
  only through secondary summaries.
