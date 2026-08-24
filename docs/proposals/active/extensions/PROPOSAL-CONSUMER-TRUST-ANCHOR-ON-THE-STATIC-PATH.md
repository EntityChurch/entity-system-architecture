# PROPOSAL — a path→hash answer from a host is an authority claim, and the route that serves one says otherwise

**Status:** DRAFT
**Tier:** extensions — `EXTENSION-NETWORK` §6.5.3.1 / §6.5.5; `EXTENSION-REGISTRY` §5.1;
`EXTENSION-TREE` §3.3a (cross-reference only)
**Origin:** derived from the corpus while verifying `entity-browser-rust`
`ROUTING-2026-08-20-a` (`7bc1ccf`). Their finding is about their own wiring; **this is about our
text**, and it is what their wiring is conformant to.
**Read at:** arch `f3056dc` · browser-rust `7bc1ccf` · workbench-go `7492fe8` · go `5020e62` ·
rust `302b7f4` · py `5205895` · keystone `77204eb`
**Release scope:** **v2. It does not reopen `EXTENSION-REGISTRY` v1** — see §9.

---

## §0 Summary

Three landed texts state one principle. A fourth specifies the route that violates it, in detail,
with a status table and cache directives, and carries no consumer-side rule at all.

| Where | What it says |
|---|---|
| `EXTENSION-TREE` §3.3a | the published root is *"the anchor of the walk-from-signed-root threat model: a consumer fetches it, verifies the signature, and walks the hash-chain from `root_hash` — **never trusting paths the host claims outside that chain**"* |
| `EXTENSION-SUBSTITUTE` §7.2 | *"The `path_index` is an authority claim about the publisher's tree; **anyone serving the URL can forge it**, so hash-verifying the manifest against its own hash proves nothing about authorship."* Signature is **MUST**; without one a consumer **MUST NOT** use it for path resolution |
| `guides/GUIDE-SERVING-MODE` §8 | three display states, and *"Never checked"* MUST read as **"Not verified"**, never as neutral chrome |
| **`EXTENSION-NETWORK` §6.5.3.1** | the `TREE_GET` leaf route returns *"the **bound content hash**"* for a path, from the host, with **no signature requirement** — under the heading sentence *"the consumer trusts the math, not the host"* |

**`TREE_GET` leaf *is* a path→hash claim served by a host.** It is the same object SUBSTITUTE §7.2
forbids trusting unsigned, and the same object TREE §3.3a says a consumer never trusts outside the
chain. NETWORK specifies it as a first-class route and never says so.

**The consequence is not hypothetical and it is not an implementation bug:** every consumer in the
cohort resolves paths this way, and each is conformant (§7).

## §1 The derivation, from the text and not from an implementation

1. **The trust anchor on this path is stated as the content hash.** §6.5.3 step 4: *"the trust
   anchor is the content hash."* §6.5.5: *"The trust anchor in both modes is the content-hash +
   (when present) the manifest signature."*
2. **A content hash authenticates bytes against a hash the consumer already trusts.** It answers
   *"are these the bytes for H?"* It cannot answer *"is H what the publisher bound at this path?"*
3. **On the `TREE_GET` path the consumer does not already have H — the host supplies it** (§6.5.3
   step 5: fetch the leaf, read `H` from `data`, then `CONTENT_GET H`). So the hash check verifies
   the host's answer against the host's own pointer. **It is self-consistency, not authorship**, in
   exactly the words §7.2 uses to rule the same shape out for the manifest.
4. **Therefore, absent a signed root, `path → content` on `http-poll` is host-asserted end to end.**
   The math that is trusted instead of the host is math over an input the host chose.

That derivation uses no implementation and no peer report. It is four lines of the corpus against
each other.

## §2 The host is a third party in the deployment this spec calls load-bearing

The obvious objection is that the host is the publisher's own bucket, so trusting it is trusting
the publisher. **§6.5.3 does not describe that deployment as the general case — it describes the
opposite one and labels it:**

> **Multi-peer-shared-domain example (the load-bearing case).** When a single domain owner hosts
> content from multiple peers…

S3 and S4 are *"multi-peer shared domain"* and *"multi-peer shared with deduplicated content."* In
both, one operator serves several publishers' trees, and a publisher's readers are handed
`path → hash` by a party that is not the publisher. Rebinding one peer's path costs the host a file
write, produces a valid-hashing response, and is undetectable at the consumer — no signature to
fail, no `seq` to regress, no error to surface.

Every CDN deployment is this case too: `RUNBOOK-CDN-BROWSER-DEPLOYMENT` is the shipping shape, and
a CDN edge is precisely a host that is not the publisher.

## §3 The downgrade is selected by the party being trusted

`signed_pointer` is an **optional field the publisher writes into its own profile** (§6.5.3), and
§6.5.5 conditions the signature check on it — *"when present."*

So the consumer's assurance level is chosen by whoever authored the profile the consumer fetched,
and **its absence is indistinguishable from a publisher who does not sign at all.** There is no
state in which a consumer can say *"this publisher signs, and this answer is unsigned"* — the
distinction the whole `published-root` mechanism exists to make is not observable at the point of
use.

This is the absent-vs-withheld collapse the corpus already names twice (R-19's silently-short walk;
§3.3a's *"a publisher that has not republished and an origin withholding a newer root produce
byte-identical results at the consumer"*), on a third surface.

## §4 The invariant that should have caught it has no static counterpart

`EXTENSION-REGISTRY` §5.1 is the one place the corpus rules on what a resolution entitles a
consumer to:

> **Resolution never confers connection authority.** A binding … can be safely consumed for its
> `transports` field because dialing those transports does NOT admit the peer — **IDENTIFY … is the
> gate.**

That is correct and complete for a **live** transport. **`http-poll` has no IDENTIFY** — the host
is a bucket that does not speak the protocol, there is no session, and `nonce_required: false`
(§6.5.3). So a resolution that returns an `http-poll` transport hands the consumer a
registry-signed `target_peer_id` — a public key, verified against the pinned root — **and no
obligation ever to use it.** The gate §5.1 names is not merely unenforced on that path; it does not
exist there, and §5.1 does not say so.

**The key is present and unused.** That is the whole finding in one sentence: on the static path
the consumer holds everything it needs to verify authorship and the spec never asks it to.

## §5 Proposed deltas

| # | File | § | Delta |
|---|---|---|---|
| D1 | `EXTENSION-NETWORK` | §6.5.3.1 | Scope the heading sentence. *"The consumer trusts the math, not the host"* holds for `CONTENT_GET` (the consumer supplies `H`) and is **false for the `TREE_GET` leaf** (the host supplies `H`). State which route is which, and cite `EXTENSION-SUBSTITUTE` §7.2's authority-claim reasoning rather than restating it |
| D2 | `EXTENSION-NETWORK` | §6.5.5 | Add the consumer obligation, symmetric to §7.2: a path→hash binding obtained from `TREE_GET` **MUST NOT** be presented as publisher-authored unless it was reached by the §3.3a walk from a verified `published-root`. A consumer MAY still use it (this is how a content-only mirror works and it stays conformant) — what it MUST NOT do is call it authenticated |
| D3 | `EXTENSION-NETWORK` | §6.5.5 | The downgrade rule: a consumer that holds the publisher's `peer_id` from an authenticated source (a registry binding, a pin, an operator-declared deployment) and reaches a profile advertising **no** `signed_pointer` MUST surface that as the `GUIDE-SERVING-MODE` §8 **"Never checked"** state, never as a successful verified fetch |
| D4 | `EXTENSION-REGISTRY` | §5.1 | Scope the IDENTIFY invariant to transports that have an IDENTIFY, and name the static counterpart: on `http-poll` the substitute gate is the §3.3a signed-root walk against the resolved `target_peer_id`, and nothing else in that profile authenticates the publisher |
| D5 | `EXTENSION-TREE` | §3.3a | Cross-reference only — §3.3a's *"never trusting paths the host claims outside that chain"* is the consumer rule D2 pins; point at it so the sentence stops being the only home of a rule no route repeats |

**What this deliberately does not do.** It does not make `signed_pointer` mandatory, and it does not
forbid the two-hop fetch. A content-only mirror is a legitimate shape the corpus already carves out
(Amendment 10), and a deployment whose origins the operator declared out-of-band — a browser loading
its own site's origin list — is trusting the host on purpose and correctly. **The rule is about what
a consumer may claim, not about what it may fetch.** Presenting a host-asserted path→hash mapping as
a verified publication is the defect; fetching one is not.

## §6 The counter-argument, stated and answered

*The origin URL itself came from a registry-signed binding, and it is HTTPS — so the host is the one
the publisher named, over an authenticated channel. Is this not just the web's trust model, working?*

Yes — and that is the point. **The web's model is exactly what this system claims to improve on**,
in this document, on this route: *"content-addressed fetch over HTTP — the consumer trusts the math,
not the host."* If the delivered property is TLS-to-a-named-host, the sentence is an overclaim, and
an overclaim in a security-model sentence is worse than a missing feature, because implementers
build on it (they did) and UIs report it (they nearly did).

The narrower answer: §7.2 already rejected this reasoning for the manifest, where the URL prefix
also comes from the source peer's own entry. Nothing distinguishes the two cases except which
section they landed in.

## §7 Corroboration — cohort implementations, cited last and labelled as such

Per **L18**, none of the following is an argument. They are evidence the reading is what
practitioners reach for under the spec as written, and that the deltas describe something real.

- **`entity-browser-rust` `7bc1ccf`.** Their Site Browser fetches foreign trees over the two-hop
  against deployment-seeded origins; `session_cache` / `signed_fetch` have no call site on that
  path; `resolve_name` has exactly one caller and it prints a preview. Re-run in their tree, not
  taken from their report. They filed it as their own sequencing problem and explicitly asked that
  it **not** be read as a defect report against the spec. **We disagree with them on that one point
  and this proposal is why** — their wiring is a correct implementation of §6.5.5 as it stands.
- **`entity-workbench-go` `7492fe8`.** `fetch/layout.go` carries `SignedPointer` *"verbatim for the
  caller's disposition"* with the comment *"`signed_pointer` present means the publisher claims to
  have shipped the §6.5.3 closure; **it is a claim, not a proof**"*, and `fetch.SignedRoot` is a
  separate call the caller may simply not make. An SDK reached the same posture independently.

Two seats, two tiers, no coordination, same shape. Under `AGENTS-STANDARD`'s own rule that is
**cohort-consistency, not independent convergence** — which is why it is here and not in §1.

## §8 L16 record — every seat searched, at a named commit

`signed_pointer` / signed-root verification on a **consumer** path:

| Seat | Commit | Result |
|---|---|---|
| `entity-core-go` | `5020e62` | publisher/serving side only (`ext/httplive/closure_scope.go`) + `validate-peer` oracle. No consumer obligation |
| `entity-core-rust` | `302b7f4` | `core/peer/src/transport_profile.rs`, `http_live/scope.rs` — serving side |
| `entity-core-py` | `5205895` | type definitions only |
| `entity-workbench-go` | `7492fe8` | consumer path present, verification caller-optional (§7) |
| `entity-browser-rust` | `7bc1ccf` | verified path exists (`name` verb); the browse path does not use it (§7) |
| `entity-core-keystone` | `77204eb` | `protocol-generator/shared/test-vectors/v0.8.0/type-registry-shapes.json` and per-language `CONFORMANCE-REPORT.json` — the generator's typestore and its reports, **no consumer path**. Searched whole tree, not by language extension (L8's first form was this exact file set) |

**No seat conditions acceptance of a path→hash binding on signed-root verification.** That is a
negative proved by a named search of every seat in `INDEX.md` §0's tier table, not by a partial grep.

## §9 Release scope — this does not reopen v1

`COHORT-OPEN-ITEMS` §0a: an OUT-list finding reopens `EXTENSION-REGISTRY` v1 only if a **v1-IN
surface, as specified, discloses a name to a third party the user did not name, or admits a binding
the trust root did not sign.**

- **No name disclosure.** Nothing here changes who a name is sent to.
- **No unsigned binding.** The registry binding — `name → target_peer_id` — is signed by the pinned
  root and verified. What is unsigned is the **content fetched afterwards**, one layer down, and
  `target_peer_id` is delivered correctly in every case.

**v2, by the line as written.** Recording it plainly rather than arguing it up: the escape hatch was
narrowed on purpose, four days ago, at the filing seat's own request, and a finding does not become
v1 work because it is interesting.

**The sequencing constraint it does create is real and is the app tier's, not the core's.** The
static path is defensible while every origin a consumer can reach was declared out-of-band by the
operator. **It stops being defensible the moment a foreign origin can arrive from a resolved name, a
typed URL, or a foreign link** — whichever of those ships first makes D2/D3 a prerequisite of that
feature rather than an improvement to it.

> ### §9.1 The trigger is a **navigation path**, not a feature — corrected by `entity-browser-rust`, and it is tighter than what it replaces `[2026-08-20]`
>
> **Adopted verbatim as the operative wording:** the prerequisite fires on **any path by which bytes
> reach the renderer from an origin the deployment did not supply.**
>
> **The feature framing was already stale when it was written, and their tree is why.** `name open`
> **already ships** (`views/shell/model.rs`) — it resolves the name, walks both signed roots, and
> reports `"{key} verified — {N} bytes"` with a preview. So "resolved-name-open" is not a future
> feature whose arrival we are waiting to gate; it exists, on the verified path, on a surface nobody
> browses with. What it does not do is **navigate** — it prints ~160 bytes into the shell scrollback.
>
> **So the two halves already sit in one binary, one wire apart**, and the verified half is the one
> nobody uses:
>
> | | reachable state | surface |
> |---|---|---|
> | `verified` | the spike — pinned key, two signed roots, `seq` floor | shell `name` verb |
> | `not verified` (**the only** state) | the product — `http_poll` two-hop, no trust anchor | Site Browser |
>
> **The risk this creates is specific, and it is the reason the framing has to change.** The cheap,
> obvious, plausible-looking next commit is *"make `name open` open the site"* — one call from the
> shell into the navigate path. Done naively it **inherits the Site Browser's trust model**, so a
> resolved, key-pinned, signed-root-verified name renders in chrome that says `not verified` — or
> worse, is wired to say `verified` on the strength of the **resolution** while the page bytes came
> down the unverified path. **That is D2's violation arriving as a UI convenience, and it would look
> like an improvement in review.** A feature-keyed trigger does not catch it, because no new feature
> ships; a navigation-path-keyed trigger does.
>
> **They have blocked that commit deliberately in their own `AGENTS.md` rather than leave it to be
> discovered later**, which is the right disposition and is recorded here so no future arch packet
> reads the gap as an omission.
>
> ### §9.2 D3 scopes to **foreign-origin** content — an owned site is outside the taxonomy
>
> **Granted, and D3 is amended rather than defended.** As written — every row carries a verification
> state — D3 would force a regression browser-rust already avoided: stamping `not verified` on the
> user's **own** tree, which no origin served them and which has nothing to verify. **That is crying
> wolf on the one case with no adversary**, and a taxonomy that fires on everything is one users learn
> to read past. **D3 binds content obtained from a foreign origin.**
>
> **Two further decisions from their build are adopted into D3 as requirements, because both are the
> kind of thing that erodes silently:**
>
> 1. **The `verified` arm MUST carry a date.** Bare *"Verified"* reads as *"this is current"* — exactly
>    the claim a published root cannot support, since **a publisher who has not republished and an
>    origin withholding a newer root are byte-identical at the consumer.** The date is the only
>    staleness signal a consumer has, and it is the first thing a deadline removes.
> 2. **The three-state shape is built even where two arms are unreachable** (`None → "not verified"`,
>    `Some(0) → "verify failed"`, `Some(t) → "verified as of {date}"`). `GUIDE-SERVING-MODE` §8's own
>    rule is that a missing state is an arch gap to file rather than a string to invent, and vocabulary
>    invented under deadline is how the third row loses its date.
>
> **Their evidence for D3's app-tier half shipping predates this proposal** — `d8e9e82`, *"say 'not
> verified', because neutral chrome reads as fine"*, three days before the delta was written. **They
> explicitly do not claim D3 is discharged** (it binds all consumers; theirs is one implementation of
> its app-tier half), and that scoping is correct and is adopted with it.
