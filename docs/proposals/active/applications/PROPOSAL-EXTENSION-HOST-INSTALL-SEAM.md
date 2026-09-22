# PROPOSAL — the extension host install seam: naming the third installer, and making a generated peer a thing you can build an extension into

**Status:** DRAFT (2026-09-01)
**Target:** `specs/sdk/SDK-OPERATIONS.md` §11.6 + §11.6.7 (the host API and its namespace rule) ·
`entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` §6.2 + §9.1 (the scope of the reserved-namespace
MUST — a **fourth-component** bump, `0.8.2.4`) · `guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1
(the reserved-prefix table names its authority) · `guides/GUIDE-CONFORMANCE.md` §7.0 + a new §7d
(the check class the wire oracle structurally cannot run).
**Provenance:** operator direction, 2026-09-01 — the goal is that these become libraries you can
construct and build from, and the deliverable is the design that gets there, the amount of work it
takes, and what it needs to look like.
Measured against `entity-core-keystone` `5a53b75` and `entity-core-protocol` `df87098`.
Absorbs and corrects `docs/research/reviews/REVIEW-2026-09-01-THE-KEYSTONE-AUDIT-…` §4.
**Workstream:** T1 (`entity-system-generator`) — this is deliverable #0, ahead of any extension.

---

## §0 The one-sentence version

**A generated peer is already a library, already has a dispatch index, and already has both halves of
the dispatch fork; what it does not have is a public way to bind a language-native handler body — and
the reason no peer has one is that every normative home describes the installer as *the
application*, so nothing in the corpus authorizes the party that actually installs an extension.**

Name that party, give the primitive a shape the profile can decline explicitly, and gate it with the
one check class the wire oracle cannot run.

> **Second pass added a D11 for consumer registration; third pass withdrew it.** A standard extension
> has three surfaces — handler, emit consumer, entity types — and §11.6 covers the handler. The
> second pass read `SYSTEM-COMPOSITION` §1.2's *"the peer builder/wiring code is responsible"* as a
> missing primitive. **It is not.** `emit` is the primitive; the registration mechanism is the
> implementer's, and `entity-core-go` built it — `AddNamedSyncHook` at `core/store/notifying.go:105`,
> `7262f17`. §1.2 is correctly scoped and the proposal was about to specify an API three
> implementations already ship. **Withdrawn in §2.5a.**
>
> **What the read did find is in §2.5b, and it is better than the thing it replaced:** the reference
> peer's own wiring runs `compute/reactive` *after* structural summaries and auto-version, where
> §2.2 puts it before both — and **nothing in 68 conformance categories tests consumer ordering.**
> That is D12, it is routed to core-go rather than ruled, and it is the strongest argument in this
> document for generating the wiring instead of hand-writing it.

---

## §1 What was measured, and what it corrects

Everything in this section is a source read in a named tree, per the read-the-live-worktree rule. It
is here because **the previous session's framing — *"the 46 peers are conformance artifacts, not
libraries"* — was true as a sentence and wrong as a conclusion**, and the correction is the design.

### 1.1 The peers are constructible in-process. Measured, not inferred.

| Peer | Constructor | Packaging manifest |
|---|---|---|
| `go` | `func NewPeer(seed []byte, opts ...Option) (*Peer, error)` — **exported** (`src/peer/bootstrap.go:101`) | Go module |
| `rust` | `pub struct Peer` (`src/peer/core.rs:117`) | `Cargo.toml` |
| `typescript` | `export class Peer implements PeerServices` (`src/peer.ts:32`) | `package.json` |
| `ruby` · `dart` · `elixir` · `swift` · `prolog` | — | `.gemspec` · `pubspec.yaml` · `mix.exs` · `Package.swift` · `pack.pl` |

**The library packaging is done.** It was done because S1's profile step makes each pod
language-idiomatic, and idiomatic in every one of these languages means *a package*. Nobody set out
to make the peers consumable and they are consumable anyway.

### 1.2 The dispatch fork already exists — native body first, entity-native second

`go/src/peer/peer.go:413–420`, the tail of `runChain`:

```go
stripped := p.stripLocal(pattern)
if inst, ok := p.handlers[stripped]; ok {          // (a) language-native body
    return inst.handleOp(operation, &dispatchCtx{...})
}
return p.entityNativeDispatch(pattern)             // (b) expression_path body
```

The same fork is in `rust` (`src/peer/core.rs:1117`, `expression_path` resolution) and `typescript`.
**This is the §11.6 architecture, correct, already built.** `p.handlers` is
`map[string]handler` — the dispatch index — and it is populated at bootstrap
(`bootstrap.go:121,153`) and never after.

### 1.3 The declaration half of §11.6.1 is already implemented *and already gated*

This is the correction that matters most. The review said the seam was *"unbuilt, unmeasured."* The
declaration half is both built and measured:

- **Built:** `go/src/peer/handlers.go:499` `register` writes, in order, the handler entity at the
  pattern, associated `system/type/*` entries, the self-issued grant at
  `system/capability/grants/{pattern}` plus its signature, and the interface entity at
  `system/handler/{pattern}`. That is §11.6.1 steps 1–3 (plus the `types` pre-step), over the wire,
  via `system/handler:register`.
- **Gated:** the oracle carries an **eleven-check `core_register_*` family** — `core_register_op_status`,
  `core_register_op_result`, `core_register_manifest_at_path`, `core_register_handler_at_path`,
  `core_register_grant_at_path`, `core_register_grant_signature_at_invariant_path`,
  `core_register_body_binding`, `core_register_reserved_refused`,
  `core_register_reserved_publishes_nothing`, `core_register_unregister_status`,
  `core_register_unregister_signature_removed` — present in the `CONFORMANCE-REPORT.json` of **all 46
  peers**. The negative half is mutation-tested in both directions and its own message says so.

**So §11.6.1 has four mutations and the cohort builds three.** The missing one is step 4 — *bind the
language-native callable in the dispatch index* — and it is missing for a structural reason, not an
oversight: **it is the only one of the four that cannot be driven over the wire**, so it is the only
one no wire oracle could ever have asked for.

### 1.4 …and `core_register_body_binding` proves the point rather than covering it

That check's message reads: *"entity-native echo body bound at `app/validate/core-register/echo/expr`."*
It binds an **entity-native** body — a `compute/literal` at an `expression_path` — because that is the
body kind a wire oracle can install. `go/src/peer/peer.go:333` handles exactly `compute/literal` and
returns `501 unsupported_expression` for anything richer.

**A `compute/literal` cannot be a TREE handler, a CONTENT handler, or a QUERY handler.** Those bodies
walk a store, hash bytes, and do I/O. Every standard extension needs the language-native path, and the
language-native path is precisely the one nothing gates.

> **This is L8's eighteenth form pointed at a gate rather than a measurement.** Two body mechanisms
> reach one observable — a `200` from a registered pattern — and the existing check attributes it to
> the mechanism it can drive. Green here says nothing about the seam. The gate in §4 below is
> designed to be attributable for exactly this reason.

### 1.5 The actual structural blocker is the namespace rule, not the collision rule

`ENTITY-CORE-PROTOCOL` §6.2 (`df87098`, line 3035):

> *"System handlers live under `system/*` paths. Implementations MUST NOT allow **user-installed**
> handlers to register at `system/*` paths."*

`go` implements it at `handlers.go:504` as `403 forbidden_pattern` for any pattern under `system/`,
and the oracle gates the refusal *and* the no-leak negative half.

**Every standard extension lives under `system/*`.** `system/content`, `system/query`,
`system/history`, `system/revision`, `system/subscription`, `system/inbox`, `system/clock`,
`system/compute`, `system/relay`, `system/route`, `system/transaction`. So:

> **The wire install path structurally cannot install any standard extension, and it is right not to.**
> `system/handler:register` is the *application* install path. Installing an extension is a different
> act by a different party, and the corpus has never named that party.

The previous session filed TREE's 409 collision as the blocker (review §4 Seam 3). **That is real and
it is narrower than this.** The 409 hits two extensions; the namespace rule hits all twenty-six.

### 1.5a The rule has never carried a rationale — measured, and it is why it over-reached

`[operator, 2026-09-01: a design that will not let us install system extension handlers is not
worth having, and the rule forbidding it was asserted with no rationale behind it.]`

**Checked rather than assumed, because the charge is specific.** `git log -S` on the sentence across
all 47 commits to `ENTITY-CORE-PROTOCOL.md`: it is present in **`507f109`, the v0.8.0 release — the
first commit the file has.** It is original core text and no recent session wrote it. **And it has
never been justified anywhere**: a corpus-wide search for a rationale — the reservation's purpose,
the attack it prevents, why `system/*` specifically — returns **nothing**, in `specs/`, in `guides/`,
and in every proposal ever written. The only documents in the corpus that discuss it are the ones
opened today. `entity-core-go`'s implementation comment restates the rule and cites §6.2; it does not
explain it either.

**So the operator is right about the mechanism and half right about the cause.** The rule is not
recent overreach by an arch session — but **an unexplained MUST is an over-broad MUST by default**,
because every downstream reader has to guess its extent, and a reader guessing conservatively
guesses *wider*. That is exactly what happened: go implemented the widest reading (correctly, for
the path it was on), and this session's own first pass read it as covering the in-process path too.
**A rule with no stated purpose cannot be scoped by anyone who did not write it.**

**Where it is wrong to delete it, and this is the part not to get carried away on.** On the dispatch
path the reservation does real work: without it, any caller holding a sufficient grant could
`register` at `system/tree` and **shadow the peer's own core tree handler** — silently substituting
its store operations. That is a live privilege-escalation, it is the reason the rule exists, and
`core_register_reserved_refused` plus `core_register_reserved_publishes_nothing` are the two checks
that hold it. **D1 keeps that property intact and removes only the reach it never earned.**

**So D1 gains a third clause: write the rationale down.** A MUST whose purpose is unstated is the
defect that produced this entire item, and leaving the purpose unstated while narrowing the rule
would reproduce it one release later.

### 1.6 Three homes, three different words for the installer, no definition anywhere

| Home | Word used |
|---|---|
| `ENTITY-CORE-PROTOCOL` §6.2 | *"**user-installed** handlers"* |
| `ENTITY-CORE-PROTOCOL` §9.1 conformance list | *"**user** handlers"* |
| `SDK-OPERATIONS` §11.6.7 | *"**application-owned** dynamic handlers"* · *"**Application code** SHOULD NOT register handlers under `system/runtime/` or directly under another extension's namespace"* |

All three mean the same distinction and none of them defines it. **There are three parties, not two:**

1. **The peer's own bootstrap** — installs `system/tree`, `system/handler`, `system/protocol/connect`,
   `system/type`. Not subject to §6.2 (it *is* the system).
2. **Application code** — installs `app/…`, `local/…`, domain paths. Subject to §6.2. This is who all
   three sentences are about.
3. **The extension installer** — peer-owner code composing a peer out of a core peer plus a set of
   standard extensions, installing at `system/{ext}/…`. **Named nowhere.**

Party 3 is not a new idea being invented here. `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 already
describes its output — six rows of the reserved-prefix table are of the form *"(When EXTENSION-X
registered.) …"*, which presupposes a party that registers an extension and owns the resulting
namespace. **The guide describes the consequence of a party the normative text does not admit exists.**

---

## §2 The delta

### 2.1 `ENTITY-CORE-PROTOCOL` §6.2 — scope the MUST to the dispatch path *(core; fourth component)*

The current sentence is not wrong; it is unscoped in the one way that matters. Replace:

> System handlers live under `system/*` paths. Implementations MUST NOT allow user-installed handlers
> to register at `system/*` paths.

with:

> System handlers live under `system/*` paths. A handler installed **through dispatch** — that is, via
> the `system/handler` handler's `register` operation (§6.13(a)) — MUST NOT be installed at a
> `system/*` path; such a request MUST be refused `403 forbidden_pattern`, and the refusal MUST
> publish none of the artifacts a successful registration would publish.
>
> **Why:** a caller that could install at `system/*` could shadow a bootstrapped system handler —
> registering at `system/tree` to substitute the peer's own store operations for every subsequent
> dispatch. The reservation exists to make a peer's system surface unreachable from the wire, and it
> is a property of the **dispatch path**, not of the `system/*` prefix.
>
> A peer's own composition — bootstrap handlers (§6.9) and standard extensions installed in-process
> by peer-owner code — installs at `system/*` by construction and is not constrained by this rule.
> The protocol specifies no in-process installation mechanism; see `SDK-OPERATIONS.md` §11.6 for the
> SDK-tier contract that does.

**Rationale for the shape.** The behaviour every conformant peer ships today is unchanged — the
refusal is still a MUST, still `403 forbidden_pattern`, still no-leak. What changes is that the rule
now states **the axis it was always about** (how the handler arrived) instead of a proxy for it (who
is imagined to own it), **and that it states its purpose at all** (§1.5a — it never has, in any
commit since v0.8.0). Nothing in any tree has to move.

### 2.1a `SPECIFICATION-FORMAT` — a constraining MUST states the failure it prevents

§1.5a is not a one-off. **An unexplained MUST is an over-broad MUST by default**, because a later
reader cannot scope what was never explained and a careful reader guesses *wider*. That is a
corpus-wide authoring property and it belongs in the authoring standard, not in a single fix:

> **[MUST]** A normative requirement that **constrains a party, a namespace, or a path** states the
> failure it prevents, in the same paragraph. A prohibition whose purpose is unstated cannot be
> correctly scoped by any reader who did not write it.

Landed as an authoring standard so the linter can grow a rule against it, exactly as `SPECIFICATION-FORMAT`
§8.5 already carries **L19**'s class-and-satisfaction-mode requirement. **Mechanically checkable and
cheap:** a `MUST NOT` naming a path prefix or a class of actor, with no *because* clause and no
citation to one, is the finding. The existing debt does not gate — it is held and paid down file by
file, the `.spec-baseline.json` pattern.

**This is a `0.8.2.4` fourth-component bump, not a release.** Per L14 as amended, the first three
components are the operator's.

### 2.2 `ENTITY-CORE-PROTOCOL` §9.1 — the conformance row follows

> `- System path reservation — a handler installed via dispatch (`system/handler:register`) MUST NOT
>   be installed at `system/*`; refused `403 forbidden_pattern`, publishing nothing (§6.2)`

Per **L23**, a rule has every normative home it is stated in. This row restates §6.2 in different
words (*"user handlers"*) and would otherwise be left asserting the retired reading — the exact
half-sweep L23's second shape was ratified on.

### 2.3 `SDK-OPERATIONS` §11.6 — the host API gets its third caller and its declared decline

Two additions to §11.6's opening.

**(a) Name party 3.** After the *"SDKs that support runtime handler registration … MUST expose a
`register_handler` primitive"* paragraph:

> `register_handler` has two callers with different namespace rules, and the SDK MUST NOT conflate
> them:
>
> - **Application registration.** The caller installs its own handlers at `app/…`, `local/…`, or any
>   open content domain. §11.6.7's namespace guidance applies.
> - **Extension installation.** Peer-owner code composes a peer by installing a standard extension's
>   handlers at that extension's own `system/{ext}/…` namespace, as enumerated by that extension's
>   spec. **This is the only mechanism by which a standard extension is installed on a peer**, and it
>   is not subject to §6.2's dispatch-path reservation, because it is not a dispatch.
>
> An SDK MUST NOT apply the §6.2 `403 forbidden_pattern` refusal inside `register_handler`. That
> refusal belongs to the `system/handler:register` **operation**. An SDK that applies it to the
> in-process primitive makes standard-extension installation impossible.

**Why this sentence has to be explicit rather than inferable:** a peer author implementing
`register_handler` reads §6.2, sees a MUST about registering at `system/*`, and copies the guard in.
That is the *conservative* reading and it is the one that breaks the ecosystem. The reference
implementation already made the analogous choice on the wire path — correctly there — so the
copy-across is the likely path, not a hypothetical one.

**(b) The decline is declared, never inferred.** §11.6's MUST is already conditional (*"SDKs that
support runtime handler registration"*). Make the condition legible:

> A peer that does not support runtime handler registration **MUST declare that**, rather than
> omitting the primitive silently. A conformance run against such a peer records the extension-host
> class as **declined**, which is distinct from absent and from failing.

**This is L17, applied before the fact rather than after.** A capability with no declared site is a
rule two conformant peers cannot both implement — and this one has 46 peers, several of which
(`asm-x86_64`, `wasm-wat`, `sql`, `turbowarp`, `pd`, `datalog`, `riscv64`) have no obvious notion of
a first-class callable. **Their answer is "declined," and "declined" must be a value someone can
write down.** The declaration site is `profile.toml` on the generator side (§5.2) and the
`system/validate/*` surface on the conformance side (§4.3).

### 2.4 `SDK-OPERATIONS` §11.6.7 — the namespace paragraph is scoped to its caller

Current text — *"Handlers registered via `register_handler` can use any pattern the caller chooses …
No namespace restriction applies to application-owned dynamic handlers"* — is correct about
applications and silent about extensions, and its next paragraph (*"Application code SHOULD NOT
register handlers … directly under another extension's namespace"*) reads as a prohibition on
precisely what an extension installer does. Add:

> The paragraphs above scope to **application registration**. An extension installer installs at the
> namespace the extension's own spec enumerates; that is the extension's namespace by construction,
> not another party's. `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` §4.1 is the authority for the
> reserved-prefix set.

### 2.5 `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 — the table names its authority

Per **L23's fourth shape** — a restatement that does not say it is one is invisible from the
document that owns the set. §4.1's reserved-prefix table is the corpus's most-read enumeration of
`system/{ext}/` ownership and it points at nothing. Add above the table:

> Each extension's own spec is the authority for the namespace it owns and the layout inside it; the
> rows here are the reserved prefixes an implementer meets first, not the whole set. `system/*`
> installation is governed by `ENTITY-CORE-PROTOCOL.md` §6.2 (dispatch path) and
> `SDK-OPERATIONS.md` §11.6 (in-process path).

### 2.5a ~~`SYSTEM-COMPOSITION` §1.2 — the consumer half of the seam~~ **WITHDRAWN**

> **D11 is withdrawn. There is no gap here — the primitive exists, it is `emit`, and the registration
> mechanism is the implementer's to build, which is exactly what §1.2 says.**
> `[operator, 2026-09-01: "the primitive is the thing the implementer builds… emit is the primitive.
> read the reference implementations."]`
>
> **Measured at `entity-core-go` `7262f17`, which is what should have happened before drafting:**
> `core/store/notifying.go` is the emit pathway — `NotifyingLocationIndex` wraps the location index
> and fires consumers on every `Set`/`Remove`. Registration is
> **`AddNamedSyncHook(name, fn)`** and `AddNamedSyncHookWithPattern(name, pattern, fn)`
> (`notifying.go:105,115`), with the content-event equivalent in `notifying_content.go:39`. The
> builder surfaces them as options — `WithNamedSyncHook`, `WithNamedSyncHookPattern`,
> `WithNamedContentHook`, `WithBindingHook` (`core/peer/builder.go:291–350`) — and `core/peer/peer.go:216–231`
> installs them. Ordering is slice order, iterated at `notifying.go:275`, with **no sort anywhere**.
> `SetMaxCascadeDepth`, `SetEmitSuppressed` and cascade-halt-on-non-200 are all there too.
>
> **So §1.2 is not under-specified; it is correctly scoped.** It specifies the *model* — the
> processing function, the attribution metadata, the ordering rule — and leaves the registration API
> to the implementation, the same way V7 §6.6 fixes no handler-body vocabulary. Writing
> `register_consumer` into a spec would be **arch inventing an API for a thing three implementations
> have already built**, which is the boundary `AGENTS-STANDARD` draws in the other direction and L0
> rule 1 forbids reasoning past.
>
> **What was actually wrong in the drafting:** the §2.5a text reasoned from *"no primitive is named
> in the spec"* to *"no primitive exists,"* which is **L8's eleventh form** — absence from a document
> read as absence from the world — and the trees were one directory away. The three consequences it
> listed (§2.7 consumer-only extensions, §2.6 both-surface extensions, position-as-composition-property)
> are all **true and all already handled** by the built mechanism: a consumer-only extension is a
> `WithNamedSyncHook` with no `WithHandler`, and QUERY is both, exactly as §2.6 describes.
>
> **D12 survives and is strengthened** — see §2.5b. The ordering has no conformance check, and that
> gap is real.

### 2.5b The ordering divergence D12 exists to catch — found in the reference peer, unnoticed

**`cmd/entity-peer/main.go:423–429` at `7262f17` registers consumers in an order that does not match
`SYSTEM-COMPOSITION` §2.2.** Registration order is execution order (`notifying.go:275`, no sort), so
the wired order is:

| Wired | Hook | §2.2 position |
|---|---|---|
| 1 | `query/index-maintainer` | 1 ✓ |
| 2 | `clock/advancement` | 2–3 ✓ |
| 3 | `history/recorder` | 4 ✓ |
| 4 | `tree/root-tracker` | **6** — structural summaries |
| 5 | `revision/auto-version` | **7** |
| 6 | `compute/reactive` | **5** |
| 7 | `subscription/notification` | 8 ✓ |

**Compute runs after structural summaries and auto-version; §2.2 puts it before both**, and states
the reason in terms: *"Running compute before structural summaries and subscription ensures derived
state has settled before summaries and notifications reflect it"*, and *"Auto-version reads the
tracked root maintained by structural summaries at position 6 — the summary MUST have settled before
auto-version reads it."*

**Stated carefully, because the cascade complicates it:** compute's writes cascade recursively and
fire the full consumer list at their own depth, so summaries and version entries *do* eventually
reflect derived state. What the wired order changes is the **outermost** pass — a summary and a
version entry are produced from a root that does not yet include the writes compute is about to make.
Whether that intermediate state is externally observable is the question, and §2.2 asserts it is
(*"the summary reflects settled derived state, not intermediate cascade state"*).

**This is core-go's tree and the finding is routed, not ruled** — they may have a reason, and
`ext/` has no ordering constraint the spec knows about. What is arch's is the absence of a check:
**nothing in 68 conformance categories tests consumer ordering**, and this divergence has sat in the
reference peer unnoticed. §2.2 states its own failure in wire-observable terms, so it is checkable.

**And it is the strongest argument in this document for the generator.** The composition is a
hand-written options list in one `main.go`; the spine it must match is a nine-row table in an
859-line spec. **A generated wiring program cannot get this wrong, and a hand-written one already
has.**

<details>
<summary>The withdrawn §2.5a surface analysis, kept because the three-surface framing is still right</summary>

**A standard extension is not only a handler**, and the surface this proposal addresses is one of
three:

| # | Surface | Install contract today |
|---|---|---|
| 1 | **Handler** — an EXECUTE target at `system/{ext}` | `SDK-OPERATIONS` §11.6. Specified in full; §2.1–2.5 close it |
| 2 | **Emit consumer** — a processing function on the tree-change and/or content-store pathway, at a normative ordered position, with a declared class | `SYSTEM-COMPOSITION` §1.2 + §2.2. **The model and the ordering are normative. No primitive is named anywhere** |
| 3 | **Entity types** | `HandlerSpec.types` (§11.6.1). Closed |

§1.2 specifies the model and its metadata — *"each consumer provides a processing function that
receives tree change events… registration metadata: handler pattern, handler grant hash, and a
default operation name"* — and then assigns the work:

> *"Ordering is by registration order… **The peer builder/wiring code is responsible for registering
> consumers in the correct order.**"*

**That is the same unnamed party §1.5a–§1.6 are about, arriving on a second surface.** And here there
is not even an over-broad primitive to scope: there is no primitive at all. No way to register a
consumer, declare its **position**, declare its **event type** (consumers register for tree events,
content events, or both — §2.2), or declare its **classification** (transparent / bounded reactive /
unbounded reactive, §2.1).

**Why this is not a smaller version of the same item.** Three reasons it binds harder:

1. **§2.7 consumer-only extensions have no handler at all.** *"Persistence is the canonical example…
   there is no `system/persistence` handler."* `register_handler` cannot install one **by
   construction**, however §6.2 is scoped. Surface 1 does not reach this class.
2. **Extensions that have both surfaces are useless with one.** §2.6 names QUERY: the
   `system/query` handler serves explicit EXECUTEs while a *separate* sync hook maintains secondary
   indexes on every tree write. Install the handler alone and the indexes are never maintained —
   which is most of what QUERY is for, and it fails *silently*, returning stale results rather than
   an error.
3. **Position is a composition property, not an extension property** (§2.4: *"the ordering applies to
   whichever consumers are present"*), and it carries **normative constraints**: `EXTENSION-REVISION`
   — *"MUST NOT register auto-version at positions ≤ 6 or at the same position as subscription"* —
   and §2.2's own rule that reversing 7 and 8 produces *"subscribers seeing a change without a
   version entry."* **A wrong order is an observable inconsistency, not a crash**, so nothing catches
   it without a check that looks for it.

**The delta.** §1.2 gains the primitive it currently describes in prose, at the same tier as §11.6
(SDK, not core — the protocol fixes no consumer vocabulary, the same reason §11.6 is not core):

> ```
> register_consumer(spec: ConsumerSpec, body: ConsumerBody) → Handle
>
>   ConsumerSpec := {
>     events:         [tree | content]   ; which pathway(s). At least one.
>     position:       uint               ; the §2.2 slot. Resolved by the installer, not chosen
>                                        ; by the extension.
>     classification: transparent | bounded-reactive | unbounded-reactive   ; §2.1
>     attribution:    { handler_pattern, handler_grant, default_operation } ; §1.2's metadata,
>                                        ; used to set per-write context (§1.4). For a
>                                        ; consumer-only extension the pattern is synthetic
>                                        ; and no entity exists at it (§2.7).
>     self_guard:     [path-prefix]?     ; the paths whose writes this consumer MUST NOT
>                                        ; re-process. Required for `bounded-reactive`.
>   }
>
>   Errors:
>     409  A consumer is already registered at this position.
>     400  Invalid spec (no events; bounded-reactive with no self_guard; position violates a
>          constraint declared by the extension owning an adjacent slot).
> ```

Three properties this shape is chosen for, each earned by something already in the corpus:

- **`position` is assigned by the installer, never requested by the extension.** §2.4 makes it a fact
  about the composition; an extension that picked its own number would be wrong in every system with
  a different member set.
- **`self_guard` is required for `bounded-reactive` rather than advisory.** §2.1's definition of the
  class *is* "direct self-recursion is structurally prevented by a self-guard" — a bounded-reactive
  consumer without one is not a bounded-reactive consumer, it is an unbounded one that has not
  noticed. **L17**: the classification is a declared value, so it needs a site and a check that reads
  it.
- **A `Handle`, matching §11.6.2** — same close semantics, same idempotence, so an extension
  uninstalls as one unit across both surfaces.

**And the ordering needs a conformance check, because it is observable and nothing tests it.** §2.2
states the failure in terms — a subscriber seeing a change with no version entry — which is exactly a
wire-observable assertion. Filed as part of §7d's class (§4.3), one row: install REVISION and
SUBSCRIPTION, write under a tracked prefix, and assert the version entry is present in the
notification the subscriber receives.

**All of the above is withdrawn with D11** (see §2.5a). It is retained only because the
three-surface framing — handler · emit consumer · types — is the right way to describe an extension,
and because the fields it sketched turn out to be a fair description of what `entity-core-go` already
built: `NamedSyncHook{Name, Pattern, Fn}`, the content-event variant, cascade-halt on non-200, and
`SetMaxCascadeDepth`. **The mistake was proposing to specify them.**

</details>

### 2.6 `GUIDE-CONFORMANCE` — a new §7d for the check class in §4

§7.0 is the *"three different things are called a vector — say which one"* table. The extension-host
check is a **fourth** thing and it must be added there rather than filed under one of the three, per
**L19** as ratified on exactly that mistake: *naming a class from §7.0's table is not sufficient —
open the section that owns the surface.* There is no owning section yet, so this proposal makes one.
Shape in §4.3.

---

## §3 What this makes possible, stated as the goal rather than the gap

With §2 landed, the composition an extension generator emits is this, and it is ordinary code:

```go
// entity-system-generator output — the CONTENT extension, installed on a generated core peer.
peer, _ := entitycore.NewPeer(seed)                        // keystone artifact, unchanged
h, err := peer.RegisterHandler(entitycore.HandlerSpec{     // §11.6 primitive, new
    Pattern:       "system/content",
    Name:          "content",
    Operations:    contentext.Operations(),                // generated from EXTENSION-CONTENT
    InternalScope: contentext.Scope(),
    Types:         contentext.Types(),
}, contentext.Body(peer.Store()))                          // generated: the language-native body
defer h.Close()
peer.Listen(addr)
```

**Nothing in that snippet is exotic and nothing in it is new machinery.** `NewPeer` exists.
`Listen` exists. The tree writes exist and are gated. The dispatch fork exists. The one new symbol is
`RegisterHandler`, and it is a wrapper over four writes the peer already performs plus one map
insert into an index the peer already has.

---

## §4 The gate — and why it cannot be a `validate-peer` category on its own

### 4.1 The structural problem

`validate-peer` drives a peer over TCP. **It cannot call an in-process API.** So there is no way to
write a wire check that says *"this peer's `register_handler` works."* This is the same wall that
kept the seam unmeasured for the whole life of the cohort, and it is why *"add a category"* is not
the answer.

### 4.2 The mechanism already exists — §7a's test-handler hook

Every generated peer already boots into a conformance mode that installs handlers a normal run does
not have: `go/src/peer/peer.go:26`, `conformance bool // --validate: §7a system/validate/* handlers`.
**The extension-host check is that hook with one difference: the handlers must be installed *through
the public primitive*, not compiled into bootstrap.**

### 4.3 `GUIDE-CONFORMANCE` §7d — the host-seam check class

> **Class:** host-seam check. **Driven** in-process by the peer's own harness; **asserted** over the
> wire by the oracle. **Authored** by arch (the reference handler and its expected observable);
> **built** per-peer by the generator; **run** by the existing `validate-peer` transport.

The peer's `--validate-host` harness performs, in order, using **only public API**:

| # | Harness action | Oracle asserts over the wire |
|---|---|---|
| 1 | `register_handler` a reference handler at `app/validate/host-seam/echo` with a **language-native** body | the three §11.6.1 tree artifacts exist at their paths |
| 2 | — | dispatch to the pattern returns the **native body's** result (§4.4) |
| 3 | `register_handler` the **same pattern** again; publish the outcome at `app/validate/host-seam/collision` | that entity records `409` |
| 4 | close the handle | dispatch → `404`; all three tree artifacts gone |
| 5 | `register_handler` at `system/validate/host-seam/ext` (the party-3 path) | the three artifacts exist — **the §2.1 delta, measured** |

### 4.4 The reference body must be inattributable to no other mechanism

**The whole check turns on step 2 and this is the part to get right.** If the reference body returns a
constant, its response is byte-identical to what `core_register_body_binding`'s `compute/literal`
already produces, and a peer that implements nothing new passes. So:

> The reference body MUST return a value that **no `compute/literal` can produce** — specifically, a
> value derived from *both* a field of the request params *and* state captured at registration time
> (e.g. `f(params.n, captured_salt)`). A `compute/literal` returns a fixed entity; a
> compute-expression body cannot see registration-time host state at all.

**This is L8's eighteenth form written into a gate as a design rule instead of learned from it
afterwards** — two mechanisms on one path produce one observable, so the check is void until the
observable distinguishes them.

### 4.5 Both controls are mandatory

Per L8's seventeenth form and its second shape — **a gate is validated in both directions:**

- **Negative:** a harness mutated to skip §11.6.1 step 4 (tree writes, no index bind) **MUST go RED**.
  Without this the check passes on a peer that only writes the tree, which is the state all 46 peers
  are in today.
- **Positive:** an unmutated harness **MUST go GREEN**, and a peer declaring `host_seam = "declined"`
  **MUST report `declined`, not WARN.** A population where most peers score inconclusive is the tell
  that the reference answer is wrong, and this project starts with 46 peers of which some genuinely
  cannot comply.

Both runs are recorded in the check's own message, as the `core_register_reserved_publishes_nothing`
family already does.

---

## §5 The amount of work

### 5.1 It is one contract, one gate, one sweep — not forty-six ports

**The peers are generated.** A phase-contract change propagates by regeneration, and keystone runs
cohort-wide sweeps as routine work — the `v0.8.2.3` sweep now in flight touches all 46 peers for two
amendments of 50 and 19 changed spec lines. This is the same shape and the same cost.

| # | Item | Owner | Size |
|---|---|---|---|
| 1 | §2.1–2.2 core delta + `0.8.2.4` | **arch** | one paragraph, one list row |
| 2 | §2.3–2.5 SDK + guide delta | **arch** | three paragraphs |
| 3 | §2.6 / §4.3 `GUIDE-CONFORMANCE` §7d + reference handler + expected observable | **arch** | one section, one fixture |
| 4 | `PHASE-S3-PEER.md` Foundation row: replace *"handler interface contract … actual handlers are community-installed"* with the §11.6 contract **by reference**, plus the exported-types requirement | **keystone** | one row of one shared file |
| 5 | `profile.toml` `[host]` block: `registration = "native" \| "declined"`, body shape, handle idiom — the §11.6.3 table is already the per-language answer for four languages | **keystone** | one block × 46, mostly mechanical |
| 6 | `--validate-host` harness + the S4 wiring | **keystone** | one runner, generated |
| 7 | Regenerate | **keystone** | one sweep |
| 8 | `validate-peer` §7d transport + the two controls | **`entity-core-go`** (oracle owner) | one category |

### 5.2 Per-peer, concretely — measured on `go`

`go` is the worst case among the three trees read, because its contract types are the most internal:

- `dispatchCtx` (`peer.go:71`) and `outcome` (`peer.go:56`) are unexported, and `dispatchCtx` carries
  `conn` — an internal transport handle. **They cannot simply be renamed**; the seam needs a public
  `HandlerContext`/`HandlerResult` pair that projects the safe subset and a private adapter. That is
  the real per-peer work and it is ~100 lines.
- `handler` (`peer.go:80`) is an unexported single-method interface — trivially mirrored by a public
  func type.
- `RegisterHandler` reuses `handlers.go:499`'s existing write sequence; the new code is the 409
  pre-check, the index bind, the compensation unwind, and an idempotent `Handle`. ~150 lines.
- **No change to `runChain`.** The fork at `peer.go:414` already prefers the native index.

`typescript` is the best case: `registerHandler(handler: Handler)` is already public
(`src/peer.ts:173`) and needs the spec's shape — `HandlerSpec` in, `Handle` out, 409, compensation.
`rust`'s `register_handler` is the wire operation and is *not* the seam, so rust is closer to `go`.

### 5.3 What this does **not** require

- **No core protocol wire change.** No new message type, no renumbering, nothing near the locked wire
  core. §2.1 rewords an existing MUST and changes no peer's behaviour.
- **No change to the 46 peers' existing conformance posture.** Every `core_register_*` check keeps
  passing unchanged.
- **No extraction of `validate-peer` from `entity-core-go`.** Standing recommendation from the review
  §4 Seam 4 — it is the oracle *because* it is not the thing under test, and 66,651 lines should not
  move on a hypothesis.

---

## §6 Sequencing, and which extension is genuinely first

Measured across all 26 extension specs for the two properties that decide installability:

| Property | Extensions |
|---|---|
| **Service-owning** (§11.6.9 lifecycle required) | `NETWORK` (19 hits) · `SIGNALING` (12) · `REGISTRY` (1) |
| **Extends a bootstrapped core handler** (409 by §11.6.1's collision rule) | `TREE` · `TYPE` |
| **Owns its own `system/{ext}` pattern — installs cleanly once §2 lands** | the remaining 21, incl. `CONTENT` · `QUERY` · `HISTORY` · `REVISION` · `SUBSCRIPTION` · `CLOCK` |

**So the first extension is `CONTENT`, not `TREE`** — and that is a scheduling correction, not a
retreat from TREE. TREE's own §9 says it *"adds the operations above to the same handler"* as core's
`system/tree`, and `entity-core-go` built them together in `core/tree/` rather than composing them.
**TREE is a core-profile delta wearing an extension's filename**, and it also owes two core bootstrap
type IDs. It is worth doing and it is a different proposal.

Recommended order: **§2 seam → `CONTENT` → `QUERY` → `HISTORY`** — then TREE once its own ruling
lands, then the service-owning three once `NETWORK`/`RELAY`/`REGISTRY` stop converging under T3.
Generating against text that is still moving is how the `EXTENSION-COMPUTE` v3.24 incident happened
(**L21**).

---

## §7 Delta table

| # | File | Section | Change |
|---|---|---|---|
| D1 | `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` | §6.2 | scope the reservation to the dispatch path; **state the shadowing failure it prevents** (never stated since v0.8.0, §1.5a); name the composition carve-out; point at SDK §11.6 |
| D10 | `specs/SPECIFICATION-FORMAT.md` | authoring standards | a MUST constraining a party, namespace or path states the failure it prevents (§2.1a) — linter-checkable, existing debt held not gated |
| ~~D11~~ | ~~`specs/SYSTEM-COMPOSITION.md` §1.2~~ | — | **WITHDRAWN (§2.5a).** `register_consumer` is not a spec gap. The primitive is `emit`, and the registration mechanism is the implementer's: `AddNamedSyncHook` / `AddNamedSyncHookWithPattern` (`entity-core-go` `7262f17`, `core/store/notifying.go:105,115`), surfaced as `WithNamedSyncHook`/`WithNamedContentHook`/`WithBindingHook` (`core/peer/builder.go:291–350`). §1.2 specifies the model and correctly leaves the API alone. Reasoned from *"not named in the spec"* to *"does not exist"* — **L8's eleventh form**, with the tree one directory away |
| **D12** | `guides/GUIDE-CONFORMANCE.md` | §7d | one consumer-ordering row: install REVISION + SUBSCRIPTION, write under a tracked prefix, assert the version entry is present in the notification. §2.2 states the failure in wire-observable terms, **nothing in 68 categories tests it, and the reference peer's wiring already diverges** (§2.5b) |
| D2 | `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` | §9.1 | conformance row follows D1 |
| D3 | `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` | line 3 | `0.8.2.3` → `0.8.2.4` (fourth component only) |
| D4 | `specs/sdk/SDK-OPERATIONS.md` | §11.6 | two callers named; SDK MUST NOT apply §6.2 inside the primitive; decline is declared |
| D5 | `specs/sdk/SDK-OPERATIONS.md` | §11.6.7 | scope the namespace paragraphs to application registration |
| D6 | `specs/sdk/SDK-OPERATIONS.md` | header | version bump (ordinary authoring work, L14) |
| D7 | `guides/GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` | §4.1 | the table names its authority (L23 fourth shape) |
| D8 | `guides/GUIDE-CONFORMANCE.md` | §7.0 | fourth row: host-seam check |
| D9 | `guides/GUIDE-CONFORMANCE.md` | new §7d | the class, the harness contract, the reference body's inattributability rule, both controls |

**Homes enumerated by subject** per L23, across `specs/` and `guides/` in both repos, whole
documents: `ENTITY-CORE-PROTOCOL` §6.1 (one-handler-per-path; dispatch index consistent with tree),
§6.2, §9.1 · `SDK-OPERATIONS` §2.5, §11.6, §11.6.7 · `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 ·
`EXTENSION-COMPUTE` §1041 (builtin override prohibition, self-described as a subset of §6.2 — **no
edit needed, it is scoped to bootstrap-registered builtins and stays true under D1**).
`ENTITY-CORE-MACHINE-SPEC` is not a home: retired 2026-08-31.

---

## §8 Cohort impact

| Seat | Delta |
|---|---|
| `entity-core-keystone` | items 4–7 of §5.1 — the phase contract, the profile block, the harness, the sweep. **Also: the name is `entity-system-generator`, singular; four files in `protocol-generator/shared/` carry the plural.** |
| `entity-core-go` | item 8 — the §7d transport in `validate-peer`, with both controls. No change to the peer. |
| `entity-core-{rust,py}` | none from this proposal. D1 changes no behaviour. |
| `entity-browser-rust` · `entity-workbench-go` | none normative. D4's *"SDK MUST NOT apply §6.2 inside the primitive"* is worth checking against their own `register_handler` — if either applies the guard, they cannot host a standard extension either. |
| `entity-system-generator` | the whole of §3 and §6 is its charter. |

**Routing:** consolidated end-of-day packet only. `ROUTING-2026-09-01-a` (keystone) and `-b` (core-go)
already went out today; a second same-day packet to either seat is **L13's fifth axis**.

---

## §9 Open items

1. **`Handle` across a `declined` profile.** A peer that declines runtime registration still needs a
   *build-time* composition path if it is to host an extension at all (§11.3's static registration).
   Is a build-time-only extension host a supported posture, or does declining mean "core peer, no
   extensions"? **Leaning: supported** — `asm-x86_64` composing a fixed extension set at assembly
   time is a legitimate peer — but it needs its own class in §7d and it is not on the critical path.
   *Carried per L9: this item does not fold with the proposal.*
2. **§11.6.9 service-owning under the generator.** Deferred with `NETWORK`/`SIGNALING`/`REGISTRY`;
   nothing in §2 depends on it.
3. **V2 (L1 capability-checked registration).** §11.6's stated target. Out of scope; D4 is written so
   it does not foreclose it.
4. **TREE and TYPE as core-profile deltas.** Owed a separate proposal, including the two bootstrap
   type IDs `EXTENSION-TREE` §9 names.
