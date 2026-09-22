# PROPOSAL — the vocabulary lifecycle: transitive compatibility, an experimental segment, and the freeze trigger

**Status:** DRAFT 2026-09-09
**Target:** `specs/SPECIFICATION-FORMAT.md` §8.7 (amend) and a new §8.7a
**Tier:** project — it binds every specification at every tier, as §8.7 already does

---

## §0 What this is, in one paragraph

**§8.7 landed a compatibility contract for published type vocabularies and it is the right rule.**
This proposal does not reopen it. It closes **three gaps that only became visible once the rule
existed**: the contract is *pairwise* where this substrate needs it *transitive*, there is no way to
say *"this vocabulary is not a commitment yet"*, and nothing states when a vocabulary stops being
editable at all.

**All three are small edits to landed text.** The reason to do them now rather than later is that
**every one of them gets more expensive the moment a third party publishes against a vocabulary** —
which is a thing that can happen without anybody's permission, and which is precisely the event §3
proposes to make load-bearing.

---

## §1 The contract is pairwise; this substrate needs it transitive

### §1.1 What it says now

§8.7 binds *"all data valid under **a previous version** must remain valid under the current one."*
Singular. Read literally that is a chain of pairwise guarantees: v3 is compatible with v2, and v2
with v1, and nothing states that **v3 is compatible with v1**.

### §1.2 Why the weak form is the wrong one here

The distinction is well worn in schema-registry practice, where the two modes are named separately
precisely because teams keep choosing the weak one and getting hurt. **The failure mode is
long-retention replay**: pairwise compatibility is sufficient when consumers only ever see recent
data, and insufficient the moment old data comes back.

**This system replays old data by construction, in three separate ways:**

1. **A mirror re-serves entries indefinitely.** Republication is a first-class act, and a mirrored
   entry may have been authored under any prior version of the vocabulary.
2. **A reader walking an old signed root is reading old data by definition.** The static posture
   makes a dormant tree a full participant — that is the point of it — so *"nobody has that version
   any more"* is never true.
3. **Content addressing makes old bytes permanent.** An entry's hash is its name. Data is not
   migrated in place here; there is no in-place rewrite to migrate it *with*.

**So a chain of pairwise steps is not the guarantee this system needs**, and the gap is silent: each
individual step is conformant while the composition fails.

### §1.3 The delta

> **§8.7 amendment.** *All data valid under **any previous version** of the vocabulary MUST remain
> valid under the current one, and data produced under the current one MUST remain valid under **every
> previously published version**.* Compatibility is **transitive**, not pairwise, and a spec MUST NOT
> narrow it to the immediately preceding version.
>
> **Why transitive:** entries are content-addressed, republished by mirrors and served from dormant
> trees, so **there is no version horizon past which old data stops arriving.**

**Cost of adopting it:** none for any conformant spec today, because the concrete rules §8.7 already
lists — new fields optional, no type changes, no renames, no repurposing, breaking change takes a new
tag — **are individually transitive already.** This amendment makes the guarantee say what the rules
already deliver, and closes the reading where a future spec satisfies each step and breaks the chain.

### §1.4 What this delta does NOT change, checked against the landed text

**It changes two words and nothing else, and that is worth stating because the delta reads like more
than it is.** §8.7 is **already bidirectional** — it binds *"all data valid under a previous version
MUST remain valid under the current one, **and data produced under the current one MUST remain valid
under the previous**."* The amendment turns *a previous* into **any** previous, and *the previous*
into **every** previously published version. **Both directions were already obligations; only their
reach moves.**

**So this introduces no forward-compatibility obligation and widens nothing a producer owes beyond
the five concrete rules.** A reviewer reading the delta without the landed sentence beside it will
reasonably suspect a second, larger obligation is being carried in under the word *transitive*. It is
not — and the way to confirm that is to open §8.7, not to reason about the amendment's wording.

---

## §2 An experimental segment — and the failure it must be designed against

### §2.1 The need, and it is created by §8.7's own strictness

§8.7 is strict on purpose: publish a tag and you have committed to it. **That strictness leaves no
way to try something.** Today an implementation experimenting with a new shape has exactly two
options — publish it and be bound forever, or keep it unpublished and get no cross-implementation
feedback at all. Both are bad, and the second is worse for this project, because **the seats
discover by building** and the whole convergence method depends on shapes meeting each other early.

### §2.2 The known failure mode, stated first because it should shape the design

**Experimental namespaces have a bad history in exactly one way: the experiment succeeds and the
marker becomes permanent.** The name ships, real deployments depend on it, and the ecosystem is left
with a production vocabulary that is spelled as though it were provisional — at which point removing
the marker is itself a breaking change, and the field's own advice on the most famous instance was
eventually to stop using the convention altogether.

**So a marker without a graduation path is worse than no marker.** The design constraint is:
*a provisional name must be structurally difficult to leave provisional.*

### §2.3 The mechanism already exists in this corpus — reuse it, do not mint a second one

**A provisional-status tier with exactly the right shape has already been designed here** for a
different surface (shell verb names). Its rules are:

- named, tracked and readable by all implementations, **but not a convergence obligation**;
- another implementation **MAY** adopt the name and **SHOULD NOT** adopt a different one for the same
  concept;
- the name may still change and the semantics may still narrow;
- **it graduates on a second independent implementation, or on a cycle's use without change**;
- ***it MUST NOT sit provisional across two releases without a disposition — that is how a
  provisional label becomes a permanent one.***

**That last clause is the answer to §2.2**, written before this problem was posed and against a
different surface. It should not be re-derived, and the vocabulary rule should use the same words.

### §2.4 The delta

> **§8.7a Experimental vocabulary `[new]`.** A specification MAY publish a type tag under an
> **`x-` segment** — `app/feed/x-poll`, not `app/feed/poll` — to state that the shape is **not yet a
> commitment.**
>
> - **§8.7 does not bind an `x-` tag.** It may change shape, change meaning, or be withdrawn.
> - **A consumer MUST treat an unknown `x-` tag exactly as it treats any unknown tag** — the
>   fallback path, not an error. An `x-` tag is not a request for special handling.
> - **Another implementation MAY adopt the name and SHOULD NOT mint a different one for the same
>   concept.** The segment exists to make convergence cheap, not to reserve territory.
> - **Graduation is by rename to the unprefixed tag**, at which point §8.7 binds it and the `x-`
>   name is retired and never reused. **Graduation is triggered by a second independent
>   implementation, or by a release cycle of use without change.**
> - ***An `x-` tag MUST NOT survive two releases without a disposition*** — graduate it or withdraw
>   it. A spec carrying one across a third release is non-conformant to this section.
>
> **The `x-` name is deliberately ugly and deliberately not free to keep.** The failure this section
> is designed against is not the experiment; it is the experiment that succeeds and never sheds its
> marker.

### §2.5 One consequence worth stating

**Graduation is a rename, and §8.7 forbids renames.** That is not a contradiction — it is the point.
§8.7 binds *published vocabulary*, and §8.7a's first clause says an `x-` tag is not that. **The rename
is the moment the commitment begins**, which is why the trigger for it has to be an event nobody
controls (§3) rather than an author's opinion that the shape is ready.

---

## §3 The freeze trigger — an event, not a decision

### §3.1 The gap

Nothing states when a vocabulary stops being editable. §8.7 binds *published* vocabulary and does not
define publication; §8.7a's graduation needs a trigger. **Both need the same event**, and leaving it
to the author's judgement is the weakest available answer, because the author is the party with the
strongest incentive to keep editing.

### §3.2 The answer the field converged on, and why it fits here especially well

**The strongest trigger in deployment is behavioural: a vocabulary freezes when a third party
implements it — with or without permission.** It is a good rule anywhere. It is a *particularly* good
rule here for a reason specific to this ecosystem:

**Publication is unilateral and permissionless by design.** Nobody grants permission to implement;
there is no registration step, no application, no gate. **So an author cannot know they have been
implemented, and that is exactly why the trigger must not be theirs to declare.** The rule is a
statement about what the author owes once the event has happened, not a notification protocol.

**It also composes with §2.3's graduation rule rather than competing with it** — *a second
independent implementation* and *a third party implements it* are the same event seen from two
sides, and it is worth noticing that this corpus and the field arrived at it independently.

### §3.3 The delta

> **Added to §8.7.** A vocabulary is **frozen** — §8.7's contract binds it in full — from the moment
> **any party other than its author publishes an implementation of it**, whether or not that party
> asked, and whether or not the author knows.
>
> **A spec MUST NOT rely on being told.** Because publication here requires no permission, an author
> cannot observe this event reliably; the obligation is to write as though it has already happened
> once the vocabulary is reachable by anyone.

---

## §4 Version-in-the-name is a known wrong answer — record it

**Recorded so nobody proposes it as an improvement**, because it is the obvious idea and it has been
tried and rejected in the closest comparable system.

> **Added to §8.7 as a note.** A version suffix in a type tag — `app/feed/entry-v2` — is **not** the
> mechanism for a breaking change. **The version controls handling; it does not control identity.** A
> breaking change takes a **new name for the new thing**, chosen for what it is rather than for when
> it arrived, and the old name keeps meaning what it meant. A numbered suffix encodes an ordering
> into an identifier that consumers must then parse to understand, and it invites the reading that
> the highest number is the correct one — which is false for a reader holding old data, and this
> system's readers hold old data by construction (§1.2).

---

## §5 What this proposal deliberately does NOT do

- **It does not reopen §8.7.** The contract is right; §1 sharpens one word of its scope.
- **It does not touch §8.8 or §8.9.** They are sound and unaffected.
- **It does not address defaults.** The asymmetry is real — *a default a consumer materializes changes
  no bytes; a default a producer materializes changes the hash* — and it is a genuinely separate
  question about the type system rather than about vocabulary lifecycle. **Filed, not folded in.**
- **It does not answer how much structure a general carrier envelope should hold.** That is the open
  design question this area still has, and it is downstream of these three.
- **It authors no check sets.** §2.4 and §3.3 are conformance-inventory rows for the specs that adopt
  them; the artefacts that exercise them belong to the implementations.

---

## §6 Open questions

1. **Is `x-` the right spelling?** It is the field's most recognizable marker, which is an argument
   for it (a reader knows instantly what it means) and against it (the most recognizable marker is
   recognizable *because* of the failure in §2.2). A distinct segment with no baggage is the
   alternative. **The mechanism does not depend on the answer.**
2. **Should graduation require the rename, or permit an alias?** A rename is clean and forces the
   disposition; an alias period is gentler on early adopters and reintroduces two names for one
   thing, which §8.8's spirit dislikes. **Recommended: rename, no alias** — the experimental window
   is the grace period, and adding a second one defeats §2.2.
3. **Does §3.3's freeze apply to an `x-` tag?** Proposed **no**, explicitly: an `x-` tag that a third
   party implements has satisfied §2.4's *second independent implementation* and should **graduate**
   rather than freeze in provisional form. Worth stating in the text so the two rules cannot be read
   as conflicting.
4. **Sourcing.** §1.2's mode names, §2.2's history and §3.2's trigger are drawn from a prior study in
   this corpus that opened its sources; **they were not re-opened for this draft.** The design does
   not rest on any of them — each argument is made from this system's own properties — but the
   attributions should be re-verified before ratification.

---

## §7 Deltas, consolidated

| # | Target | Change | Size |
|---|---|---|---|
| **D1** | `SPECIFICATION-FORMAT` §8.7 | compatibility is **transitive**, stated, with the replay reasoning | one paragraph |
| **D2** | `SPECIFICATION-FORMAT` §8.7a **(new)** | the experimental segment, its non-binding status, graduation, and the two-release disposition limit | ~15 lines |
| **D3** | `SPECIFICATION-FORMAT` §8.7 | the freeze trigger, plus *a spec MUST NOT rely on being told* | one paragraph |
| **D4** | `SPECIFICATION-FORMAT` §8.7 note | version-in-the-name recorded as a rejected answer | one paragraph |

**Enforcement points.** D1 and D4 are review-time rules over spec text. **D2 is mechanically
checkable and should be gated**: an `x-` segment is greppable, and *a tag carrying one across two
releases* is a comparison between the tag list and the release history — the same shape as the
existing expiry ratchet. D3 is not mechanically checkable and does not pretend to be.
