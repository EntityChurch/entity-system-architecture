# PROPOSAL — the reflection listener has no satisfiable admission rule, and the one it needs is already written one spec over

**Status:** DRAFT (2026-09-09)
**Target:** `specs/extensions/EXTENSION-SIGNALING.md` — §8.2 split into §8.2 (mailbox) + a new
§8.2a (reflection); §9.3 gains a response-composition bound; §13 item 2's disposition is rewritten.
**Depends on nothing new.** The mechanism proposed here is `EXTENSION-NETWORK.md` §6.7.2's, restated
for the surface it was never applied to.

**One sentence.** `EXTENSION-SIGNALING` §8.2 is a `[MUST]` whose scope reaches both unwrapped
listeners, and **on the reflection listener each of its three obligations is either meaningless or
impossible** — so a public reflector is non-conformant by construction, and the anti-amplification
rule that surface actually needs exists in `EXTENSION-NETWORK` §6.7.2 for a *different*, *less
exposed* surface.

---

## 0. Why this is opened now

§13 item 2 has read *"needs a deployment call before a public unwrapped surface is exposed — the
mechanism and error code are pinned; the numbers are not"* since the surface was specified. A
deployment preparing to expose one read that disposition, took it at face value, and asked for
thresholds.

**The disposition is half true.** For the **mailbox** it is exactly right: §8.2 pins the dimensions
and §9.2 pins the code, and only the numbers remain. **For the reflection listener nothing is
pinned**, and the sentence that says otherwise is the reason nobody has looked.

This proposal is the shape. **The numbers stay out of the spec** and §4 says why.

## 1. The defect

§9 is *The Unwrapped Protocol* and §9.1 places **two** listeners under it: the mailbox (§9.2, TCP,
length-prefixed CBOR) and reflection (§9.3, UDP, RFC 5389 STUN unmodified). §8.1's admission table
addresses *"Unwrapped"* as one row. §8.2 then says:

> **MUST.** A deployment offering the unwrapped surface rate-limits **per source and per key**, and
> refuses excess with `rate_limited` rather than dropping silently.

Applied to §9.3, that has three clauses and all three fail.

| Clause | On the mailbox | On the reflection listener |
|---|---|---|
| **per source** | well defined | **well defined, and the only one that survives** — though see §1.1: the source is *asserted*, not observed |
| **per key** | the 33-byte rendezvous key, present on `offer` and `collect` | **the dimension does not exist.** A STUN Binding Request carries no rendezvous key and no client identity of any kind. There is nothing to key a second bucket on |
| **refuses with `rate_limited`, not a silent drop** | §9.2's error table carries the code | **impossible, and backwards.** `rate_limited` is a mailbox response code; RFC 5389's error space does not contain it, and §9.3 states *"RFC 5389 is the document and this spec adds nothing to it."* Emitting a non-standard error would break the browser ICE agent §9.3 exists to serve |

**Both readings of the scope are bad, which is the tell.** Read narrowly — §8.2 governs only the
mailbox — and **the reflection listener has no admission rule at all**, on the one surface in this
specification that is unauthenticated, connectionless and reachable by anyone. Read broadly and it
has a rule no conformant implementation can satisfy. §11.1 makes the broad reading normative: a
server-role peer *"implements … §8.2 rate limiting if it exposes the unwrapped surface."*

### 1.1 The third clause is not merely impossible — it prescribes the attack

The reason a public UDP reflector is a security object at all is **source spoofing**: an attacker
sends Binding Requests carrying a victim's source address, and the reflector answers the victim.
Every byte of every response is aimed traffic the attacker did not have to send.

Against a spoofed source there is exactly one correct behaviour for a request over the limit, and it
is **to send nothing**. §8.2 forbids it in the same sentence that mandates the limit. A conformant
implementation, reading the rule as written, would answer an attacker's flood in order to avoid
dropping silently.

**A `[MUST]` whose literal satisfaction is the failure mode it exists to prevent is a defect, not a
wording preference.**

## 2. The rule already exists, one spec over — on the surface that needed it less

`EXTENSION-NETWORK` §6.7.2, the dial-back half of reachability:

> **MUST.** The dial-back targets the **requesting peer's own observed source address** — the
> address the asked peer *itself observed* on the request connection — and **never an address
> supplied in the request body.** The asked peer **MUST** rate-limit dial-backs per requester and
> keep the dial-back payload small and fixed-size, so there is no amplification factor.

and §6.7.1's grant note, on `observe-address`:

> the operation only echoes the source address the peer itself observed, so it leaks nothing the
> requester does not already imply by connecting, and the response is a single small address (no
> amplification factor). Still **rate-limited**.

**That is precisely the reflection listener's rule, written for a surface with three defences §9.3
does not have.** `observe-address` runs over an established connection (so the source is observed,
not asserted), behind `system/capability/network-reflect`, inside the entity protocol. §9.3 runs
unauthenticated, connectionless, over a transport where the source address is an attacker's free
parameter.

**The hazard was correctly identified and the rule landed on the wrong listener.** The two surfaces
share a purpose and no vocabulary — one says *dial-back* and *observed address*, the other says
*reflection* and *STUN* — which is why a sweep by term never joined them.

*(§8.3, the existing anti-amplification `[MUST]`, does not cover this and does not claim to. It
bounds **a peer's** unsolicited probes toward a candidate during the punch — its own rationale is
that *"an open `lobby` bucket is a traffic amplifier aimed by whoever fills it."* Different actor,
different direction, different surface.)*

## 3. The proposed delta

Three edits. None changes a wire format; none touches the mailbox's behaviour.

### 3.1 Split §8.2 by listener

**§8.2 — Mailbox admission (MUST).** Unchanged text, scoped explicitly to §9.2: *"A deployment
offering the §9.2 mailbox rate-limits per source and per key, and refuses excess with `rate_limited`
rather than dropping silently."*

**§8.2a — Reflection admission (MUST), new.** Proposed:

> **MUST.** A deployment serving §9.3 reflection rate-limits Binding Requests **per source
> address**, and **silently discards** requests above the limit. This is the deliberate exception to
> §8.2's refuse-rather-than-drop rule: the source address of a UDP Binding Request is asserted by
> the sender and not observed, so a response to an over-limit source is traffic aimed at whoever the
> sender named. There is no conformant way to signal refusal — RFC 5389 defines no such error and
> §9.3 adds nothing to it — and there is no second dimension to limit on, because a Binding Request
> carries no key and no identity.
>
> **MUST.** The Binding Success Response carries `XOR-MAPPED-ADDRESS` and **no attribute the request
> did not require.** A public reflector MUST NOT emit `SOFTWARE`, and MUST NOT emit the legacy
> `MAPPED-ADDRESS` that §9.3 permits, unless a deployment has a stated reason: each optional
> attribute raises the amplification factor against the minimal 20-byte request, and §9.3's own
> normative floor is that clients read `XOR-MAPPED-ADDRESS`.

**Why a fixed factor rather than a threshold.** The response bound is a property of the
implementation and is the same everywhere, so it belongs in the spec. The *rate* is a property of a
box and a link and belongs to a deployment — §4.

### 3.2 §9.3 gains the composition bound by reference

One sentence pointing at §8.2a, because §9.3 is where an implementer composing the response is
reading. §9.3's *"this spec adds nothing to it"* stays true of the **protocol**; the bound is on
what a deployment chooses to include, which RFC 5389 leaves open.

### 3.3 §13 item 2's disposition is rewritten

Current text asserts the mechanism is pinned. Proposed: item 2 narrows to *the mailbox's numbers,
which are a deployment call*, and the reflection half is recorded as closed by §8.2a rather than
left in an open item that reads as *"only numbers remain."*

### 3.4 The conformance rows

§11.1 already obliges §8.2 for a server-role peer exposing the unwrapped surface; it gains §8.2a on
the same condition, scoped to a peer that serves §9.3. §11.4's *"Rate-limit thresholds"* entry
stays and is narrowed to the rates — **the response composition leaves implementation-defined**,
which is the substantive move: it is currently a free choice and it is the half that sets the
amplification factor.

**These rows have no check driving them today and this proposal does not pretend otherwise.** The
reflection listener's conformance is measured by nothing at present; naming the obligation is the
prerequisite for a check, not a substitute for one.

## 4. Thresholds are a deployment's, and the corpus already says so

§11.4 lists rate-limit thresholds as **Implementation-Defined**; §13 item 2 calls for *a deployment
call*. Both are right and this proposal does not move them.

**A number in the specification would be wrong in a way that is hard to withdraw.** A useful
threshold is a function of the link, the instance size, the admission posture and the observed
handshake cost — quantities the specification cannot see. A pinned figure would be either so high it
protects nothing or so low it fails a legitimate deployment, and once published it would be cited as
a conformance floor rather than as the guess it is.

**What a deployment is owed instead is the shape, and that is §3:** which dimension to limit on,
what to do at the limit, and a bound on the response that makes the factor a known constant rather
than an implementation accident. With those pinned, choosing a rate is an operational exercise
against a live box.

**Non-normative, for a deployment sizing one — offered as a starting point to measure from, not as a
recommendation to adopt:** a token bucket keyed on the source address, aggregated at a prefix rather
than a single address (an attacker with a /64 of IPv6 source addresses defeats per-address
accounting for free); a burst sized to a handful of requests, since an ICE agent gathers a small
fixed number of candidates and then stops; and a refill rate set from measurement rather than
prediction. A reflector's honest traffic is close to constant per session and nothing legitimate
polls it.

## 5. `[ASK-ARCH]` — the decisions owed

1. **Is the split right, or should §8.2 stay one rule with a reflection carve-out?** Recommended:
   split. One rule with an exception for the case where two of its three clauses do not apply is
   the shape that produced the defect.
2. **`MUST` or `SHOULD` on the silent discard.** Recommended `MUST` — the alternative is a
   conformant amplifier.
3. **Does the response-composition bound go further** — a byte ceiling on the Binding Success
   Response, rather than an attribute allowlist? Recommended: allowlist. A byte ceiling has to be
   restated for IPv6 and would need re-deriving whenever an attribute is added.
4. **Does §8.2a belong to this specification at all**, or does the reflection listener's hygiene
   belong beside `EXTENSION-NETWORK` §6.7.2 with the two stated once? Recommended: state it here
   and cross-reference §6.7.2, because the reader composing a Binding Success Response is reading
   §9.3. **Whichever way this goes, the two must cite each other** — their separation is what let
   the gap open.

## 6. What this proposal does not do

- **It does not set a threshold**, and adopting it does not unblock anyone from choosing one. §4.
- **It does not touch the mailbox.** §8.2's per-source-and-per-key rule is correct for §9.2 and its
  error code is reachable there.
- **It does not revisit §8.5.** The unwrapped surface's cleartext rendezvous key is a stated,
  accepted v1 cost and is a different question.
- **It does not claim the reflection listener is exercised by any check.** §3.4.

## References

- `EXTENSION-SIGNALING.md` §8.1, §8.2, §8.3, §9.1, §9.2, §9.3, §11.1, §11.4, §13 item 2
- `EXTENSION-NETWORK.md` §6.7.1, §6.7.2 — the rule this restates, and its rationale
- `PROPOSAL-CONNECTION-NODE.md` §5 open item 2 — the origin of §13 item 2's wording
- `PROPOSAL-SIGNALING-ICE-PROVISIONING-LIFETIME.md` — the adjacent open work on §4.5.1's
  provisioning; disjoint from this, and neither blocks the other
- RFC 5389 §15.2 (`XOR-MAPPED-ADDRESS`), §15.10 (`SOFTWARE`)
