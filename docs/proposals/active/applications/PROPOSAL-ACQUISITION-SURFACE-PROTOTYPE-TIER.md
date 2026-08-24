# PROPOSAL — the acquisition surface as a PROTOTYPE tier: shell verbs, the room type, and how fast-moving app-tier work gets tracked

**Status:** DRAFT (2026-08-21) · **Living document — edit in place, this cycle.**
**Target:** `guides/GUIDE-SHELL-FRAMING.md` §4.2 (a SIGNALING row + a prototype tier) ·
`specs/applications/APP-CONVENTION-CHAT.md` (the room type, when CHAT folds)
**Provenance:** `entity-browser-rust`'s two-device run (`f9c8e7c`) and their design doc
`docs/plans/DESIGN-ACQUISITION-UX-2026-08-21-…` (`ee44da2`); arch's absorption
`docs/research/reviews/REVIEW-2026-08-21-THE-ACQUISITION-PATH-…`; workstream **T7**.
**Operator direction, 2026-08-21:** *"we're on a short timeline… document it, say this is prototype
for stuff like the shell verb and meet… start tracking it… we're tracking in a proposal all the
changes coming through because it may be coming through fast."*

---

## §0 What this proposal is for, and why it is shaped as a ledger

**One seat is building an acquisition surface faster than the normal proposal → ratify → fold cycle
turns.** `meet`, `net`, `connector`, a standing meet, and an ad-hoc room type all arrived inside
thirty-six hours, and more is coming this week. The choice is not *"ratify fast"* or *"make them
wait"* — both are wrong. It is to **give the surface a declared status that is honest about its
maturity**, so it can ship, be read by the other seats, and still be changed without anyone claiming
a broken contract.

**So this proposal does two things:**

1. **Establishes a PROTOTYPE tier** for app-tier surface that is real, shipping, and explicitly not
   yet a convergence contract (§1). This is the thing to land first; it is cheap and it unblocks
   everything else.
2. **Carries the running ledger** of what has arrived on that surface, with a disposition per item
   (§3). **New items get appended here rather than opening a new proposal each time.**

**What it is not.** It is not a ruling on browser-rust's UI, which is theirs. It is not a request for
anyone to wait. Every item in §3 is either already built or is theirs to build.

---

## §1 The PROTOTYPE tier — the actual proposal

`GUIDE-SHELL-FRAMING` §4.5 rule 2 already prescribes what happens with a genuinely new top-level verb:

> *"the proposing implementation **SHOULD** surface the name to the architecture team and the other
> reference impls for explicit convergence before adoption, and the chosen name becomes a precedent
> under rule 2."*

**That rule has exactly two states — surfaced-and-adopted, or not a verb — and the gap between them
is where all the real work lives.** `meet` is shipping in a product today. Calling it "not a verb" is
false; calling it a ratified precedent every implementation MUST adopt is premature, because it has
run on one substrate for one day.

### §1.1 The delta

Add to `GUIDE-SHELL-FRAMING` §4.2 a **SIGNALING row** (the table has none — the extension has no shell
verbs at all, which is why `meet` had nowhere to be registered), and a status column:

| Verb | Extension | Op | Status |
|---|---|---|---|
| `meet` | SIGNALING | Meet at a rendezvous key; learn peers' ids (§3 modes) | **PROTOTYPE** |

And the tier definition:

> **PROTOTYPE `[new]`.** A verb that ships in a real implementation and is **named, tracked and
> readable by the cohort, but is not yet a convergence obligation.** Another implementation MAY adopt
> the name and SHOULD NOT adopt a different one for the same concept. The name may still change; the
> semantics may still narrow. **A PROTOTYPE verb graduates to Tier E on a second independent
> implementation, or on a cycle's use without a change.** A verb MUST NOT sit in PROTOTYPE across two
> releases without a disposition — that is how a provisional label becomes a permanent one.

### §1.2 Why this is worth a guide edit rather than a note

**Because the failure it prevents is cheap now and permanent later.** Three seats independently
naming this concept `meet` / `rendezvous` / `pair` / `introduce` costs one table row to prevent today
and is unfixable once each has shipped a CLI its users type. The naming rule exists for exactly this
and had nowhere to put a verb that was not yet ready to bind everyone.

**And the honest-maturity half is the same discipline the conformance vocabulary already runs.**
`GUIDE-CONFORMANCE` §7.0 makes an implementer say *which kind* of vector a claim rests on; §5.2b.1
makes them state a satisfaction mode. **A verb table with no status column asks a reader to treat one
day of single-substrate use and three-way convergence as the same claim** — the same overclaim, one
layer up.

### §1.3 Explicitly NOT ratified, and why the restraint is the point

`net` and `connector` are **not** proposed here. browser-rust was asked whether they are intended as
durable verbs or local debugging affordances and has not answered. **A local affordance promoted to a
spec'd verb is much harder to withdraw than to add**, so the default is out. They join §3's ledger the
moment that seat says they are durable.

---

## §2 The room type — `app/chat/room`, not a degenerate conversation `[RULING]`

browser-rust's design §5.4 routes one question and recommends proceeding without waiting. **Arch
agrees they should not wait, and the answer changes what they build by about one line.**

### §2.1 Their proposal, and the defect in it

They propose deriving a room id from a `Conversation` with `title = name`, `created_at: 0`,
**empty `creator`**, **empty `initial_participants`**, `policy: open`.

**That object is not a conformant `app/chat/conversation`, and its own CDDL says so.**
`PROPOSAL-APP-CONVENTION-CHAT` §2, the `[LOCKED shape]`:

```cbor
"creator":              <peer_id>,      ; required, and "" is not a peer_id
"initial_participants": [+ peer_id],    ; CDDL `+` is ONE OR MORE — [] is invalid
```

Both proposed values violate the shape they are being encoded into. **And the failure is worse than
an invalid entity**, because that entity is *decodable*: a conformant reader that fetches it gets a
well-formed conversation with a zero-member roster and an unauthenticated creator, folds a roster of
nothing, and renders an empty conversation with no error. **A degenerate instance of a locked type is
the shape that passes every decoder and means something different to every reader.**

### §2.2 The ruling — and it is simpler than what they proposed

> **A room is a distinct entity type, `app/chat/room`, whose content-hash is the `room_id`:**
>
> ```cbor
> room = {                          ; type = app/chat/room
>   "name":       tstr,             ; the human word — the same word as the meet tag
>   "created_at": uint,             ; 0 for a derived room
> }
> ; content_hash(room) == room_id
> ```
>
> **No `creator`, no roster, no `policy` — not empty ones, absent ones.** The fields a room does not
> have should not appear at all, which is the ordinary absent-never-null discipline and here it is
> load-bearing: there is no admin to mis-assign because there is no field to mis-assign, and no reader
> can mistake it for a conversation.

**This is strictly cheaper than their version** — fewer fields, no sentinel values, and the same
`~6 lines` their table budgets. `wellknown_room_id(name)` hashes the two-field map. Storage,
delivery, and the union read are all unchanged, exactly as their §5.2 says.

**Their D5 instinct is right and arch is adopting the reasoning, not just the conclusion.** A
roster-derived room id has a genuine cross-peer seam hazard that arch's own earlier ruling did not
name: *four devices whose member lists differ by one entry derive two different room ids, both valid,
nobody errors, and two people sit in a room the others are not in.* That is an equivalence-collapse —
invisible locally, springing apart at the seam — and **deriving from the name removes the thing there
was to disagree about.** It is a better design than the one arch ruled on yesterday.

### §2.3 What this does and does not change about yesterday's ruling

`ROUTING-2026-08-21-n` §4 ruled the **roster-derived genesis** sound for `policy: closed` only. **That
ruling stands and is untouched** — it governs a closed conversation with a known roster, which is the
1:1 they ship today and the fixed group CHAT §6 describes. **A named room is a different object, not a
policy variant of that one**, which is precisely what browser-rust said and why they were right to
route it rather than assume either way.

**The 1:1 keeps its current derivation** (their §5.3.4). Two derivations for two objects is correct
here and is not duplication: one is *"the conversation between exactly these people"*, the other is
*"the room called this word."*

### §2.4 Two limits to carry into the UI, both theirs and both already named

- **A room name is guessable, exactly like a meet tag.** Their §5.3.2 already quotes `rendezvous.rs`
  — *"a tag is public by design and a weak secret is a tag in disguise"* — and that sentence belongs
  where a room name is typed. Membership has **no** boundary: whoever guesses the name and can reach a
  member is in.
- **Combined with unsigned messages (D7) and `debug_open_grants`, an open room has no authorship
  boundary either.** `load_messages` overriding the body's `author` with the path defends against a
  lying *body*, not against a peer that can write into the tree at all. **Not a blocker for a
  prototype; a blocker for anything called secure.** Recorded in §3 as R-6.

---

## §3 The running ledger — append here, do not open a new proposal

**This is the tracking vehicle the operator asked for.** Items arriving on the acquisition surface get
a row. A row leaves this table by folding into a spec/guide, or by being explicitly dropped.

| # | Item | Seat | Arch disposition | State |
|---|---|---|---|---|
| **P-1** | `meet` as a shell verb | browser-rust | **PROTOTYPE tier + SIGNALING row** (§1) | Guide edit owed — **arch** |
| **P-2** | `net`, `connector` verbs | browser-rust | **Not ratified, deliberately** (§1.3) — awaiting that seat's answer on durable-vs-affordance | **Question outstanding to browser-rust** |
| **P-3** | Room id derivation | browser-rust | **RULED — `app/chat/room`, two fields, no empty sentinels** (§2) | **Answered. Build it; do not wait** |
| **P-4** | Standing meet (30 s event → standing intent) | browser-rust | **App-tier, no arch ruling needed.** The decaying cadence is a protocol-adjacent constraint and their scar is correct — a fixed fast re-deposit produced ~970 deposits/side. `SIGNALING` §11.5's gate counts deposits and fails O(N); **a standing meet must stay inside a fixed O(1) per-tag budget or it fails the gate that already exists** | Theirs to build. **Arch flags the existing gate as the bound** |
| **P-5** | `node_in_force` / the four acquisition sources | browser-rust | **Property named** (T7); precedence stays impl policy | Absorbed |
| **P-6** | The served-URL gate | browser-rust | **Arch endorses their sequencing call** — §4 should not ship without it. Three of six defects lived on that path and no harness covers it | Theirs. **Not arch-gated, but arch agrees it gates §4** |
| **P-7** | `signaling-use` capability has no declared operation (**L17**) | arch | Proposal owed to SIGNALING | **OPEN — arch** |
| **P-8** | `GUIDE-NETWORKING-MODEL` LAN row false for browsers | arch | Guide edit | **OPEN — arch** |
| **P-9** | Acquisition blindness class in `GUIDE-CONFORMANCE` | arch | Guide edit | **OPEN — arch** |
| **R-6** | Chat messages are unsigned, and `delivery.rs`'s header calls them *"signed message logs"* | browser-rust | **The header is the defect to fix now**; signing itself is out of scope for this arc [their D7] | **Recorded, not scheduled.** A false comment asserting a security property is the highest-value line in a file to correct |

---

## §4 Conformance vectors `[REQUIRED before ratification]`

Deliberately thin, because a PROTOTYPE tier that demands a vector set is not a prototype tier.

- **For P-1 (graduation to Tier E only):** a second implementation registering `meet` with the same
  surface syntax. No wire vector — a shell verb is not a cross-peer seam.
- **For P-3 (required before `app/chat/room` folds into `APP-CONVENTION-CHAT`):** one vector pinning
  `content_hash({name, created_at: 0})` for a fixed name, so two implementations derive the same
  `room_id`. **Class, per L19: a fixture-corpus vector** — it is a pure derivation over agreed input
  with no peer, no boundary and no harness, which is the §7.0 *fixture corpus* row and not §7c.
  **Ownership: arch authors, since the derivation is arch's ruling.**
- **Derive-to-meet applies.** `room_id` is a value two parties must independently reproduce from an
  agreed input, so per `SPECIFICATION-FORMAT` §8.4.5 it is **pinned to the ECFv1-SHA-256 floor**,
  never the deriving peer's home format. **This is the one genuinely cross-peer property in the whole
  room design** and it is exactly the class the naming rule exists for — two peers on different home
  formats would otherwise derive different rooms for the same word and silently never meet.

---

## §5 Open / deferred

- **`[ASK-BROWSER-1]`** — are `net` and `connector` durable verbs or local affordances? (P-2)
- **Gossip membership** — browser-rust §5.3.1's growth path. Not this arc, and the standing meet
  covers the common case in practice.
- **Workbench-go WebSocket interop** — arch's A-5. **browser-rust has said that seat is not in a
  position to run it and the door stays open at no cost to them.** Not a dependency of anything here.
- **Graduation review for P-1** — owed at the next release boundary, per §1.1's no-two-releases rule.
