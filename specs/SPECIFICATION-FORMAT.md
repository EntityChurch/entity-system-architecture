# Entity System — Normative Specification Format

**Version**: 1.5
**Status**: Active
**Kind**: authoring-standard
**Authority**: binding

---

## 1. Purpose

This document defines the format for normative specifications in the entity system. Normative specs contain implementation requirements — what a conforming implementation MUST, SHOULD, and MAY do. They are the authoritative source for building interoperable implementations.

Normative specs are distinct from architectural documents. Architectural documents provide grounding, motivation, design rationale, and system-level understanding. Normative specs reference architectural documents but do not duplicate their content. Implementation requirements live in normative specs; everything else lives elsewhere.

---

## 2. Specification Language

Entity type definitions are the primary specification language. Protocol messages, data structures, and extension types are defined as `system/type` entities — the same type system the protocol itself uses.

### 2.1 Type Definition Notation

```
type_path := {
  fields: {
    field_name: {type_ref: "primitive/string"}
    optional_field: {type_ref: "primitive/uint", optional: true}
    reference_field: {type_ref: "system/hash"}     ; Entity reference
  }
  constraints: {
    field_name: {pattern: "^[a-z]+$"}
  }
}
```

Rules:

- Type paths use forward slashes: `system/protocol/execute`
- Field specs use the `system/type/field-spec` structure from the core protocol
- Comments use `;` (semicolon) — consistent with CBOR diagnostic notation
- Inheritance via `extends` field when applicable
- Entity references use `type_ref: "system/hash"` — the hash value identifies the referenced entity in the envelope's `included` map

### 2.2 Inline Comments

```
system/example := {
  fields: {
    id:     {type_ref: "primitive/string"}    ; Unique identifier
    status: {type_ref: "primitive/uint"}      ; HTTP-style status code
  }
}
; Additional context about the type as a whole.
; Multi-line comments use consecutive ; lines.
```

### 2.3 Primitive Types

Normative specs reference the eight core primitives:

| Type | Description |
|------|-------------|
| `primitive/string` | UTF-8 text |
| `primitive/bytes` | Binary data |
| `primitive/uint` | Unsigned integer |
| `primitive/int` | Signed integer |
| `primitive/float` | IEEE 754 floating point |
| `primitive/bool` | Boolean |
| `primitive/null` | Null |
| `primitive/any` | Unconstrained |

### 2.4 Collection Fields

```
array_field: {array_of: {type_ref: "primitive/string"}}
map_field:   {map_of: {type_ref: "primitive/any"}}        ; Keys always strings
```

### 2.5 Supplementary Structures

Not every structure needs to be a full `system/type`. Inline structures that appear only within one type definition use lowercase names without a registered type path:

```
grant-entry := {
  handlers:   {include: [string], exclude: [string]?}   ; system/capability/scope
  resources:  {include: [string], exclude: [string]?}   ; system/capability/scope
  operations: {include: [string], exclude: [string]?}   ; system/capability/scope
  peers:      {include: [string], exclude: [string]?}   ; system/capability/scope
  constraints: any?                                      ; handler-interpreted
}
```

These are documented alongside the type that contains them. They do not appear in `system/type/*`.

---

## 3. Algorithm Notation

Algorithms use pseudocode with the following conventions:

### 3.1 Function Signature

```
function_name(param1, param2):
  ; body
  return result
```

### 3.2 Control Flow

```
verify_something(input):
  if condition: DENY
  if other_condition:
    do_something()
    do_something_else()

  for item in collection:
    if !check(item): return false

  while condition:
    current = next(current)

  return true
```

### 3.3 Conventions

- `DENY`, `ALLOW`, `REJECT` as terminal outcomes (all caps)
- `lookup(hash, map)` for entity resolution from included maps
- `now()` for current timestamp
- Set operations: `∪` (union), `∈` (membership)
- Indentation for nesting (2 spaces)
- No language-specific syntax (no braces, no `def`, no `fn`)
- Comments with `;`

### 3.4 Error Conditions

```
if invalid_state:
  return error(status_code, "error_code", "Human-readable message")
```

Or for verification algorithms, the pattern `if bad: DENY` with final `ALLOW`.

---

## 4. Requirements Language

Per RFC 2119:

| Keyword | Meaning |
|---------|---------|
| **MUST** | Absolute requirement. Non-compliance breaks interoperability. |
| **MUST NOT** | Absolute prohibition. |
| **SHOULD** | Recommended. Valid reasons to deviate may exist, but implications must be understood. |
| **SHOULD NOT** | Discouraged. |
| **MAY** | Optional. Implementations choosing not to implement remain conformant. |

Keywords appear in **UPPERCASE BOLD** in prose, or uppercase in pseudocode comments.

### 4.1 A constraining requirement states the failure it prevents

**[MUST]** A normative requirement that **constrains a party, a namespace, or a path** states the
failure it prevents, in the same paragraph — or cites the section that does.

**An unexplained prohibition is an over-broad prohibition by default.** A later reader cannot scope
what was never explained, and a reader guessing conservatively guesses **wider**, not narrower. The
rule is written against a measured case: the core protocol's `system/*` path reservation carried no
rationale in any revision from the file's first commit, and its widest reading — the one every
implementer arrived at independently — made it impossible to install any standard extension on any
peer, because every standard extension owns a `system/*` namespace. The rule was not wrong. It was
unscoped in the one way that mattered, and nobody could tell, because it never said what it was for.

This is an authoring property, not a style preference: the *because* clause is the only thing that
lets a downstream reader decide whether a new case falls inside the rule or outside it.

**Mechanically checkable:** a `MUST NOT` naming a path prefix or a class of actor, with no *because*
clause in its paragraph and no citation to one, is the finding. Existing debt is held and paid down
file by file rather than gating.

---

## 5. Document Structure

Every normative spec follows this structure. Sections may be omitted if empty, but the ordering is fixed.

### 5.1 Required Sections

```
# Title — Normative Specification

**Version**: X.Y
**Status**: Draft | Active | Superseded
**Depends**: [list of normative specs this extends or requires]

---

## N. Type Definitions
    ; All types introduced by this spec

## N. Algorithms
    ; Pseudocode for required behaviors

## N. Constants
    ; Status codes, error codes, limits, enumerations

## N. Conformance
    ; **Requirement id prefix:** `XXX`
    ### N.1 Requirements
        ; a table — one row per requirement, each with a stable id (§8.5a)
    ### N.2 Types Installed        ; if the spec installs types
    ### N.3 Handler Registered     ; if the spec registers a handler
```

**The inventory is a table of identified requirements, not headings of bullets `[MUST]`.** The
grouped-by-level form — `### N.1 MUST Implement` / `N.2 SHOULD` / `N.3 MAY` /
`N.4 Implementation-Defined` — was this standard's shape through v1.2 and is **superseded**. It
cannot express what §8.5a requires: a level that belongs to the row rather than to the heading
above it, and a requirement that can be cited by something other than the section it sits in.

### 5.2 Optional Sections

Specs MAY include additional sections before Conformance:

- **Foundations** — core concepts and definitions (used by the core protocol spec)
- **Flow diagrams** — ASCII wire diagrams for multi-step interactions
- **Security considerations** — threat model, mitigations
- **Extension points** — how this spec can be extended by other specs

### 5.3 Header Fields

| Field | Required | Description |
|-------|----------|-------------|
| Version | Yes | Semantic version of this spec — **a claim about this document's content**, moved under §9.1 and exempted by nothing (§9.2) |
| Status | Yes | Draft, Active, or Superseded |
| Kind | Yes | One of §10.1's six values |
| Authority | Yes | `binding` or `informative` (§10.2) |
| Governed-by | If applicable | The **sub-tier standard** whose rules bind this document, by path |
| Depends | If applicable | Normative specs required as prerequisites |
| Encoding | If applicable | Wire encoding reference (e.g., ECF) |

**`Kind` and `Authority` are declared because they are not derivable.** A reader —
and a tool — otherwise infers them from a filename or a directory, which is how a
document that governs five specifications came to be classed informational, and
how the corpus's own authoring standard came to be classed a guide. Neither
document was wrong; nothing had asked them.

**The header carries the declared fields and nothing else. Version history goes at the
END of the document, never in the header.** A specification's job is to say what the
thing **is**; what it **was** is secondary and belongs after the content, not in front
of it. Narrative wedged into the header does three things, all bad: it separates the
declared fields from each other, it puts a changelog where a reader's introduction
should be, and it grows without bound — one extension had reached **twelve paragraphs
and over three thousand words above its first section heading**, which a reader met
before learning what the extension does.

- **Where it goes:** a trailing `## Document History` section, newest entry first.
- **Where it is headed:** out. History is not lost by leaving a specification — it is
  in the revision control system and in the proposal that argued for the change. **A
  destination has not been settled, so the trailing section is the interim, not the
  answer.**
- **Enforcement:** `header-narrative` (position, gated) beside `document-history-section`
  (existence, held as accepted debt while the destination is decided).

**`Governed-by` names the sub-tier standard, and only that.** The project tier
(§10.3) binds every document unconditionally, so declaring it in every header
would be noise that goes stale. **Omit `Governed-by` when the document has no
sub-tier** — that omission *is* the statement that the project tier governs it
directly, and §10.3 says so once.

```
**Version**: 1.0
**Status**: Active
**Kind**: normative-spec
**Authority**: binding
**Governed-by**: `guides/GUIDE-APPLICATION-DEVELOPMENT.md`
```

### 5.4 Section Numbering

- Top-level sections: `## N. Title`
- Subsections: `### N.M Title`
- Cross-references: `§N.M` (e.g., "see §3.2")
- External cross-references: `DOC.md §N.M` (e.g., "see ENTITY-CORE-PROTOCOL.md §5.2")

The full addressing model — canonical reference forms, what a spec may cite, and
how unresolved references are classified — is in §11.

---

## 6. Tables

Tables are used for:

- **Constants and codes** — status codes, error codes, type enumerations
- **Conformance levels** — what each level requires
- **Field summaries** — when a type has many fields and prose is needed per-field
- **Comparison** — when distinguishing similar concepts

Format:

```
| Column | Column | Column |
|--------|--------|--------|
| data   | data   | data   |
```

---

## 7. Examples

### 7.1 Encoding

Examples use CBOR diagnostic notation (RFC 8949 §8), not JSON:

```
{
  "type": "system/protocol/execute",
  "data": {
    "request_id": "req-001",
    "uri": "entity://peer123/some/path",
    "operation": "get",
    "params": null
  }
}
```

When showing binary or encoded output, use hex with annotation:

```
ecfv1-sha256:a1b2c3d4...    ; 64 hex chars
```

### 7.2 When to Use Examples

Examples are supplementary. The type definition IS the specification — examples illustrate but do not define. If an example contradicts a type definition, the type definition wins.

---

## 8. Spec Conventions

§8.1–§8.5a are conventions for **system extension specs** (inbox, compute,
relay, …). **§8.6–§8.9 bind every binding spec in the corpus**, whatever its
tier: they are the project-tier rules for a document that mints a type
vocabulary, declares a capability, or defines a container (§10.3).

### 8.1 Dependency Declaration

```
**Depends**: ENTITY-CORE-PROTOCOL.md
```

Extensions MUST declare which normative specs they depend on. Types from dependencies are referenced by path, not redefined.

### 8.2 New Types

Extension specs define new types that layer on core protocol types. Extensions MUST NOT redefine core types. Extensions MAY define types that reference core types via `type_ref`.

### 8.2a The availability descriptor — the standard shape for a field that may have no value

**A field that can be absent has four distinct causes, and collapsing them is the dishonesty the
substrate floor forbids** (*deliver-or-signal, never silently drop*). A value may be missing because the
platform **cannot** produce it, because the caller is **not permitted** to see it, because it **has not
been measured**, or it may genuinely **have** a value. *"Field missing"* and *"zero"* answer none of
those questions, and **every heterogeneous surface re-invents this distinction unless it is standardized
once.**

An **availability descriptor** wraps an optional typed value `T` as an ordinary ECF map — **no new wire
mechanism**:

```
availability<T> := {
    state:     "value" | "unsupported" | "denied" | "unknown",   ; REQUIRED
    value:     <T>,        ; present IFF state = "value"; omitted otherwise
    fidelity:  "exact" | "coarse"    ; OPTIONAL; meaningful only when state = "value"
}
```

| `state` | Meaning | The consumer's next move |
|---|---|---|
| `value` | present and known (see `fidelity`) | use `value` |
| `unsupported` | this platform or build **cannot** produce the field | **never retry** — a structural no |
| `denied` | producible, but the caller's grant does not cover it | **retry with a broader grant** |
| `unknown` | not measured or not yet probed — **distinct from zero** | **re-poll** later |

**The split is exhaustive and each state drives different consumer behaviour**, which is the whole
reason it is four values and not a boolean.

**Rules `[MUST]`:**

- An implementation emitting an availability-typed field **MUST** set `state`.
- **`value` MUST be present when `state = "value"` and MUST be omitted otherwise — omitted, not
  `null`.** A conditional encode, never a null placeholder.
- **`fidelity: "coarse"` MUST be set when the value has been deliberately coarsened** — bucketed,
  rounded, or clamped for privacy — so a consumer never mistakes a bucket for a measurement.
  **An omitted `fidelity` reads as `exact`**, never as absent or unknown.
- **A reader MUST treat an unrecognized `state` as `unknown`**, per the MUST-ignore-unknowns discipline,
  so future states degrade safely.

**Extensions SHOULD reuse this shape rather than inventing a per-field "missing" convention.** Where a
standalone descriptor result needs a type name it is `system/availability`.

> **This is a synthesis, not an invention, and each source covers only one axis.** The support axis is a
> platform feature query (*can this build do it*); the permission axis is a permission-state query
> (`granted` / `denied` / `prompt`); the presence axis is a hardware-state enum (`Enabled` / `Absent` /
> `UnavailableOffline`). **No surveyed system unifies support, permission and measurement into one
> field** — that union is the contribution. A *health* axis is deliberately excluded: health and alerting
> are monitoring concerns, not properties of a read-only field.

**Not an error type.** `denied` and `unsupported` are **normal field states**, not operation failures —
an operation-level failure still returns the ordinary coded error.

### 8.3 New Operations

Extensions that add handler operations define them as `system/handler/operation-spec` entries:

```
my-handler/operations := {
  new_operation: {
    input_type:  "my-extension/input"
    output_type: "my-extension/output"
  }
}
```

### 8.4 Optional Fields on Core Types

Extensions MAY define optional fields that appear on core types (e.g., `deliver_token` on EXECUTE). These are documented in the extension spec, not the core spec. The core spec's Open Types (§2.7) guarantees preservation.

When an extension adds optional fields to a core type, the extension spec MUST:
- Define the field as a type definition
- State which core type it extends
- Specify behavior when the field is absent (default)
- Specify behavior when the field is present

#### 8.4.1 The permission is tiered — not every extension qualifies `[normative]`

> **An extension MAY define optional fields on core types only when its concept is durable and broadly applicable** — part of the standard capability layer that peers generally participate in. **An extension whose concern is specific to a technology landscape, a deployment topology, or a particular feature set MUST NOT extend core types.** It defines its own types and composes through a seam.

**Why the permission is tiered.** Open Types make *adding* a field cheap and make *removing* one impossible. A core type is permanent, so an optional field on it is carried, preserved and hashed by **every peer forever** — including peers that never implement the extension, and peers built long after the motivating technology is gone. The cost is unbounded in time and paid by everyone; the benefit is bounded and paid to one extension. That asymmetry is the whole rule.

**The test an author applies — three questions. All three MUST pass, and all three MUST be answered in writing in the proposal that adds the field.** A field that passes two is not a close call; it is a rejection.

1. **Durability.** *Would this field still make sense if the technology that motivated it disappeared?*
   `deliver_token` on EXECUTE describes **delivery authorization**, which any message-passing system has — it passes. A `nat_candidate` field describes an artifact of **middlebox behavior in the IPv4 era** — it fails.
2. **Universality.** *Would a peer that never implements this extension still reasonably carry this field on that type?* If it is dead weight for most peers, the field belongs in the extension's own types.
3. **Seam check.** *Is composing through a seam genuinely worse here — and why, concretely?* The alternative always exists and costs nothing, so a core-type field MUST be shown to be **better**, not merely convenient. **State what the seam design would look like and what it costs.** "A seam would be awkward" is not an answer; "a seam cannot express this because X" is. An author who cannot articulate the seam design has not established that they need the field.

**The operational form of question 3 — the discriminator that decides nearly every case:**

> **Does this information have to be carried in the entity's bytes, or does it only have to be available where the code runs?** **Bytes** ⇒ a field may be justified. **Call site** ⇒ use a seam.

Every field that qualifies today does so for this reason: `deliver_token` travels **on the wire** to a remote peer that must parse it; `constraints` travels **with the type definition** to readers who run no validator; `clock` on `system/revision/entry` must be **inside the hash** or it does not survive what it describes; `composition` must be visible to anyone loading the handler entity. A seam is a local call and can do none of these.

Conversely, connectivity data — candidates, observed addresses, punch coordination — is **call-site** data, which is why `EXTENSION-SIGNALING.md` composes through `EXTENSION-NETWORK.md` §10.3's `establish_live` seam and needs no core-type field at all.

**The alternative, stated once so it is never re-derived.** A non-qualifying extension composes exactly as NETWORK / RELAY / ROUTE / SIGNALING already do: **its own types, plus a seam the substrate exposes** (`dispatch_fallback`, `establish_live`, `resolve_next_hop`). Seams are additive, removable, and carried only by peers that install the extension. **This section restricts one mechanism precisely because a better one is already in universal use.**

**Namespace claims are a different act and are not governed here.** Defining a type *inside* a core-owned namespace (e.g. `system/protocol/*`) without adding a field to a core type is a separate question with a different cost; see §8.4.2.

#### 8.4.2 Types inside another spec's namespace `[normative]`

> **A type is owned by the spec that defines it, and a namespace is owned by the spec that owns the types in it.** A spec MUST NOT define a type inside a namespace another spec owns. Where a spec needs to *describe* such a type — because its own behavior turns on the type's shape — it **references** the owning spec and MUST NOT restate the definition.

**Restating is the failure this rule prevents, and it is not hypothetical.** A reproduced type definition is a second source of truth that drifts silently: nothing links the two copies, no gate compares them, and the divergence surfaces only when two implementations built from different copies fail to interoperate. Every instance found in this corpus arrived the same way — a spec quoted a type it did not own, for good local reasons, and the copy aged.

**So, whenever one spec mentions a type another spec owns:**

- **Name the owner at the point of mention** — *"defined in `X.md` §N, canonical there"* — every time, not once per document.
- **Never restate the field list.** Reference it. If a reader needs the fields inline for the passage to make sense, that is a signal the passage belongs in the owning spec.
- **A behavioral note about someone else's type is fine**; a definition of it is not. "This handler rejects a `delivery` whose `status` is absent" is a behavioral claim about the local handler; re-declaring `system/protocol/inbox/delivery := {...}` is a competing definition.

**Ownership decides *who may define*; it does not decide *where the type goes*.** These are two questions and conflating them produces circular reasoning — *"it is in `system/protocol/*` because core put it there, and core may put things there, so it belongs there."* That argument justifies any placement whatsoever and is therefore not an argument. **Placement is a separate, falsifiable claim: the type must satisfy what the namespace means.**

> **A namespace means something, and the meaning is fixed by its existing members — not by who owns it.** Before placing a type, enumerate the namespace's current contents and state the property they share. **If the new type lacks that property, it does not belong there, however legitimate the author's authority to put it there.** A namespace whose members share no property is not a namespace; it is a prefix, and it should be split.

**This test binds core specs exactly as it binds extension specs.** A core spec placing a core type in a core namespace can still place it wrongly, and "core defined it" is not a defense — a misplaced type misleads every reader who reasons from the prefix, which is what a namespace is *for*. Core authority makes a placement **authoritative**; it does not make it **correct**, and only the second is what a reader relies on.

**Corollary — a core-model spec may extend a core namespace.** The ownership half of this rule constrains *cross-owner* definition, not layering as such: a core-model document defining a type inside a core namespace is core extending core, which is ordinary. The placement half above still applies to it.

#### 8.4.3 Derived documents are never authoritative `[normative]`

> **A generated, condensed, or summarizing document is downstream of its sources and MUST NOT be cited as the definition of anything.** Ownership belongs to the spec that defines a type; a derived document only *reproduces* it. **A derived document that disagrees with its source is stale, not evidence of a defect** — and reconciling it is the maintainer's regenerate-or-retire hygiene, never a conformance gate and never a cohort action.

**Every derived document MUST say so in its header**, and every citation of one MUST resolve to the source instead. *(`EXTENSION-COMPUTE.md` §2.4 carries this note for `compute/error`.)*

**The standing example is now a cautionary one: `ENTITY-CORE-MACHINE-SPEC.md` was retired rather than repaired.** It was a condensed summary of the core specs, sourced from nothing and kept in sync with nothing, and it advertised itself as the *"complete machine-readable specification for generating a conforming peer"* — a claim no build ever relied on. Its drift was undetectable because **nothing gates spec-against-spec**: the only way to find it was a reader opening both documents side by side, which is exactly what does not happen. When two of its sections were finally examined, **both were wrong** — one missing an entire error-code row that implementations had already split over. **A derived document is therefore not repaid by re-syncing it: the maintenance burden is unbounded and the gate does not exist, so the disposition is retire.** Do not author a new condensed, "implementation", or "machine" edition of a spec in this corpus.

**Why this is normative and not a style preference.** A derived copy is indistinguishable from a source at a glance — same format, same declarations, same authority-looking prose — so an auditor comparing a spec against it will find real differences and report them as real defects. **That has happened here**: one audit cycle produced thirteen "drifted types," three escalated as interoperation-breaking, every one of them an artifact of diffing specs against their own stale output. **Before measuring conformance to a document, establish that something is actually sourced from it.**

#### 8.4.4 One top-level namespace segment per owner `[normative]`

> **A specification owns exactly one top-level `system/<segment>`, named for itself, and everything it defines nests beneath that segment.** A specification MUST NOT hold two top-level segments. A specification MUST NOT use a hyphenated top-level name (`system/foo-request`) where a nested path (`system/foo/request`) expresses the same thing.

**Hyphens separate words; `/` separates scopes.** `system/durability-request` and `system/durability-result` are not two concepts — they are one concept's request and result, spelled with the wrong separator, and they consume two top-level segments as a result.

**Name the segment for the owner, never for the problem domain.** `system/nat/*` describes *what the messages are about*; `system/signaling/*` describes *who defines them*. Only the second answers the question a namespace exists to answer — **given a type, which specification defines it?** A problem-domain segment also strands its siblings: the moment a second substrate arrives, the domain name no longer covers it and a third segment gets invented.

**This is a flat scheme on purpose, and it is not a status ranking.** A namespace segment costs nothing to peers that do not implement the extension — it is a prefix on types only its implementers hold. **The tiering test in §8.4.1 does NOT extend here**, and importing it would be applying a rule past its rationale: §8.4.1 exists because a field on a core type is carried, preserved and hashed by *every peer forever*, and that cost asymmetry is absent for a namespace. Nesting one extension's segment inside another's to signal that it is "less core" would also violate §8.4.2 — a specification does not define types inside a namespace another specification owns.

> **Precondition — a rename is not a cleanup until this is answered `[MUST, applies to §8.4.2 and §8.4.4 alike]`.** **The type string is content-hash input.** `Hash::compute` hashes ECF `{data, type}` (`ENTITY-CORE-PROTOCOL.md` §1.2), so renaming a type **rebinds the `content_hash` of every entity of that type** — universally, for every type, not as a special case. Before renaming anything, answer both in writing:
>
> 1. **Is there durable data of this type at rest?** An entity already written carries the **old** type string in its bytes. After a rename, a handler dispatching on the new string **does not match it** — the data is not corrupted, it is *orphaned*.
> 2. **Is any entity's `content_hash` referenced by something else?** A rename changes the hash, so every stored reference to it dangles.
>
> **If either is yes, the rename is a data migration, not a cleanup, and the type is NOT retroactively renamed.** Record the exception where the type is defined. **New types in the same family SHOULD adopt the correct namespace** — the grandfathering is for what already exists.
>
> **Worked example (`EXTENSION-ENCRYPTION.md` §5.1):** `system/encrypted` and `system/encryption-pubkey` keep their flat names. Renaming would rebind the hash of every encrypted entity at rest **and** change every `recipient_key` — which *is* `content_hash(pubkey)` — breaking peer/group encryption for all published keys. Both implementations independently ship the flat names, so the spec renaming away from them would have *manufactured* drift out of a cohort-converged surface.
>
> **This precondition exists because it was skipped.** The 2026-08-02 namespace pass classified that rename as a cheap in-step cleanup and had to revert it. The ideal in this section yields to the hash-input invariant, every time.

> **What "at rest" asks, and what it does not `[clarification, 2026-08-10]`.** Condition 1 is a **question of fact about data that exists**, not a question about the type's design. "This type is durable by design" is not a yes — nearly every type persists, and reading the condition as a property would make this section self-nullifying: no persistent type could ever be renamed, which is every type worth naming correctly. In a **no-installed-base** ecosystem (`AGENTS.md`: no back-compat, no migration windows, no dual-kind acceptance) the honest answer to condition 1 is normally **no**.
>
> What durability-by-design does change is **how the rename lands, not whether it may**: a type the receiver stores and later re-reads MUST be renamed as a **coordinated cohort cut** — every implementation changes the string in one round, and any peer holding such entities at cut time drains them first. There is no dual-kind acceptance window to hide behind, so the cut is the migration. Say so at the type when this applies.
>
> **Worked example, the other way (`EXTENSION-INBOX.md` §2.1) `[RATIFIED 2026-08-10]`.** `system/protocol/inbox/delivery` → `system/inbox/delivery` **stands.** Condition 2 is **no** — no `delivery` `content_hash` is referenced anywhere — and that is the leg on which encryption failed, where `recipient_key` *is* `content_hash(pubkey)`. Condition 1 is durable-by-design (the inbox is a persistent mailbox, §7) but factually empty, so it sets the *coordinated cut* obligation rather than blocking. The two rulings are consistent: **encryption failed condition 2 and inbox does not.** The cohort-convergence argument does not save the old name here — all three implementations converged on `system/protocol/inbox/*` *because the mis-homing prefix misled them*, which is the defect being corrected, and the sibling `notification` already re-homed to `EXTENSION-SUBSCRIPTION` on the same §8.4.2 grounds. Leaving `delivery` behind would strand the corpus half-corrected on a rule that admits no exception.

**Legitimate exceptions, which MUST be documented where they occur:**

- **Roots that belong to no extension** — `primitive/*` (core scalar primitives).
- **A distinct authored artifact, as opposed to an operational surface.** `compute/*` (the expression-language node types an author writes: `literal`, `lookup`, `apply`, `if`, `lambda`) is deliberately separate from `system/compute/*` (the handler's operation surface a peer invokes). The test for this exception is narrow: **two genuinely different kinds of thing, authored by different parties for different purposes** — not merely two groups of types from one spec.

#### 8.4.5 Never pin a hash width `[normative]`

> **A specification MUST NOT state a fixed byte length or hex-character count for a `content_hash`.** A content hash is the self-describing pair `(content_hash_format, digest)` (`ENTITY-CORE-PROTOCOL.md` §1.2); **its length follows its leading format varint and is never assumed.** Where a width helps the reader, give it as a *worked instance of a format* — "66 hex chars under ECFv1-SHA-256 (`0x00`), 98 under ECFv1-SHA-384 (`0x01`)" — never as the requirement.

**The rule the width usually meant to state.** Almost every fixed-33 / fixed-66 in this corpus was reaching for one of two real requirements, and both survive the generalization intact:

- **"Include the format-code byte"** — the digest-only form is what must be rejected, because dropping the code destroys the algorithm discriminator. State *that*, and the 64-char rejection follows from it.
- **"This field is unambiguous to parse"** — a fixed width is one way to get that, and usually not the one in play. Concatenations stay unambiguous if the variable-length component is **last** (or if every variable component is length-prefixed); a validator stays strict by rejecting **any input whose length disagrees with the length its own format byte implies** — which is *stronger* than a constant, since it also rejects a 98-char string claiming `00`.

**Why this is normative and not a style preference.** A width lock is invisible while one algorithm ships: the constant and the derived length are the same number, every test passes, and the two readings of the rule are indistinguishable. They spring apart the instant a second format is exercised — and by then the divergence is **already in the field**, under a MUST, in code nobody reviewed as wrong. **This has now happened twice.** `APP-CONVENTION-EMBED` v0.1 shipped a `hex33` schema atom past three independent reviews (`CHARTER.md` rule 6, the applications-layer form of this rule). `EXTENSION-NETWORK` §6.5.3.1 then pinned the served hash hex at 66 in the same bullet that justified the format byte as *crypto-agility* — and the 2026-08-10 cohort measurement found go and rust returning `400` where python returns `200` on a SHA-384 `CONTENT_GET`, each implementing a different half of the same sentence, with **30 of 31 SHA-384 conformance failures behind that one rule.** Neither implementation was wrong to read it as it did.

**The check.** Grep a draft for `33`, `66`, `49`, `98`, `hex33`, "fixed-length" and "both components are". Every hit is either a worked instance labelled as one, or a defect.

**The one width that is legitimately pinned, and why it is not an exception.** A value that two independent parties must *reproduce* from an agreed input — `EXTENSION-SIGNALING.md` §3.1's rendezvous key is the standing example — is pinned to the **SHA-256 floor** and does not follow the deriving peer's home format. That is not a width lock: it pins the *format*, and the width follows from the format exactly as this rule requires. The distinction is **authored content vs. reproduced lookup token**, and it is the whole test — authored content follows its author's home format and is therefore variable; a lookup token is pinned so two peers running different home formats cannot derive different keys for the same input and silently never meet.

#### 8.4.6 Which format a derived hash uses `[normative]`

§8.4.5 bars pinning a hash's **width**. This answers the question underneath it, which the corpus had applied in exactly one place: **whose `content_hash_format` does a hash carry?**

> **Two dispositions, and every hash in the corpus is one of them.**
>
> 1. **Hold-and-fetch — format-free.** You already have the hash: it travelled to you on the wire, or you read it off an entity. **Use it verbatim, at whatever width its own format byte implies, and never re-derive it.** A specification MUST NOT state its format or width. *(`TREE_GET` leaf `data`; `RELAY` `envelope_inner`; `ENCRYPTION` §7.3's HKDF `recipient_pubkey_hash`; the §3.5 invariant pointer `system/signature/{target_hex}`; `system/content/{ns}/{hex(H)}`; `QUORUM` §7's `{quorum_id_hex}` and `{hash_hex}`; `IDENTITY` §5.1's `{published_handle_hex}`.)*
> 2. **Derive-to-meet — pinned to the ECFv1-SHA-256 floor (`0x00`).** You **compute** the hash from an agreed input in order to construct a path, key, or token that **another party must independently compute and match**. It **MUST NOT** follow the deriving peer's home format. *(`SIGNALING` §3.1 rendezvous key; `REVISION` §3.1 `prefix_hash`; `{peer_id_hex}` in `ROLE` §1.3a.1 / §3.1; `{contact_id_hex}` in `IDENTITY` §5.1; and **the `system/peer` identity entity itself** — see the second worked example below.)*
>
> **The test, and it is one question.** *Would a second party, holding only the agreed input, have to know your home format to reproduce this value?* If yes, it is derive-to-meet and it is pinned. **A home format is not discoverable from a peer-id or a path**, so any rule that requires knowing it is already broken — the two peers simply never meet, with nothing failing loudly.

**A specification introducing a hash MUST state which disposition it is.** This is the §3.5 discovery-locality pattern applied to formats rather than paths, and it converts "audit indefinitely for new instances" into a checkable property.

**Why one rule and not a ruling per site.** These collapse into each other on a **single-format network**, which is every network anyone has run: derive-to-meet and hold-and-fetch produce the same bytes when there is only one format, so both readings pass every test and the distinction is invisible. It springs apart at the **cross-peer seam** the moment two peers run different home formats — the equivalence-collapse shape, and the same reason §8.4.5's width lock survived three reviews.

> **Worked example — `{peer_id_hex}` asserted both, and the definition is what gives (`ROLE` §1.3a.1) `[RULED 2026-08-10]`.** `ROLE` §1.3a.1 called `{peer_id_hex}` "the `content_hash` of their `system/peer` entity" **and** used it as a path segment every peer must construct for every *other* peer. Those cannot both hold. Home-format derivation is a **genuine collision**: peers A and B compute different `{peer_id_hex}` for the same third peer C, so their role assignments, identity certificates and quorum paths never meet. **Ruled derive-to-meet — pinned.** The definition is the half that was wrong: `{peer_id_hex}` is the **floor-format identity hash**, which *coincides* with the stored entity's `content_hash` exactly when that peer is SHA-256-home. That coincidence is the local collapse, not the definition.
>
> **Named cost, so it is a decision and not a side effect:** pinning derive-to-meet values makes ECFv1-SHA-256 **effectively mandatory** for any peer participating in role, identity, quorum, revision or signaling paths — stronger than V7 §1.2's "SHOULD support." That cost was already accepted for the rendezvous key; it is stated here rather than re-discovered per extension. It buys the property that any peer can construct any other peer's paths knowing only its peer-id.

> **Worked example, second half — the `system/peer` entity is derive-to-meet too, and pinning only the path segment was the error `[RULED 2026-08-10]`.** The ruling above pinned `{peer_id_hex}` while deliberately leaving the stored `system/peer` entity on its author's home format, on the reasoning that a stored entity "is authored content and stays hold-and-fetch." **That half was wrong, and it is wrong by this section's own test.** `system/peer`'s data is `{public_key, key_type}` (`ENTITY-CORE-PROTOCOL.md` §3.5 — `peer_id` is **not** in the hashable basis) — every field of it is recoverable from the peer-id, which is public, because a peer-id decodes to exactly `(public_key, key_type, hash_type)`. **An entity with no author-chosen content cannot be hold-and-fetch:** nobody ever fetches a `system/peer` entity to learn its hash, because every consumer derives it. Splitting the two produces **two `content_hash`es for one identity** — the exact state `ENTITY-CORE-PROTOCOL.md` §1.8 exists to prevent — with the path segment at the floor and the identity reference (`signature.signer`, cap `grantee`/`granter`) at the connection's active format. They coincide only while the active format *is* the floor, which is the same local collapse one layer down.
>
> **Ruled: a `system/peer` entity is authored under the ECFv1-SHA-256 floor unconditionally**, on every connection and whatever the peer's home format. Consequences, all of them narrowing rather than widening: §1.8's "use the authored hash, MUST NOT recompute" stops being a *coincidence* preserved by §4.5a's one-format-per-connection rule and becomes an identity — the authored hash **is** the floor-derived hash, permanently, so the prohibition needs no amendment and gets stronger. §4.5a item 1 excepts the identity entity from the active format; its stated purpose (make §5.2 `grantee == author` / `signer == author` byte-exact) is **over-satisfied**, since the identity hash is now the same bytes across every connection in the network rather than merely within one. And §1.2's "a peer's persistent state is uniformly its home format" carries one named exception, not a general leak.
>
> **A segment's disposition and the disposition of the entity stored at it are independent — this is not a contradiction.** A SHA-384-home peer writing `system/peer/status/{peer_id_hex}` puts a **floor-format** hex segment in the path and **home-format** content at it. That is the specified shape, not a drift: the segment is a *name* two peers must both construct (derive-to-meet), and the entity is that peer's own persistent state (`ENTITY-CORE-PROTOCOL.md` §1.2, home format). Reading a mixed pair as an internal inconsistency is the expected first reaction — say which half is which at every path that carries a hash, and the question stops arising.
>
> **What this deliberately does not do:** it does not make the floor mandatory for peers. A non-floor **home** format stays fully runnable — §1.2 still governs all stored content, §4.5a still governs envelope framing, capabilities and signatures. Exactly one entity type, of three public fields, is pinned. The alternative reading — promote §8.4.6's "effectively mandatory" prose to a `[MUST]` on peers — was **rejected**: it would retire `content_hash_format` negotiation as a live wire surface, and it makes any non-floor conformance arm illegal by construction rather than merely divergent.

### 8.5 Conformance

Extension specs have their own conformance section. An implementation MAY be conformant to the core protocol without implementing any extensions. Extension conformance is independent.

**A conformance item MUST declare its class, and "vector" alone does not `[MUST]`.** `GUIDE-CONFORMANCE` §7.0 names four different artifacts the word is used for — a **`validate-peer` check** (behavioral, driven over the wire against a running peer, authored by the oracle author), a **fixture-corpus vector** (static byte-level `.diag` + canonical `.cbor`, authored upstream), a **host-seam check** (an assertion about a peer's in-process API, driven by the peer's own harness and asserted over the wire — `GUIDE-CONFORMANCE` §7d), and an **impl-internal unit / property / fuzz test** (not cross-impl, not conformance). A spec that pins a conformance item **names which**, because the four have different authors, different homes, and wildly different costs to satisfy — and an undeclared item routes work to a seat that does not author it.

**The classes are not interchangeable as evidence, which is the substantive reason and not a filing convention.** A pure-function check any seat can satisfy in-tree with no harness proves almost nothing about interop; a behavioral check against a live peer may require harness capability nobody has built. **Reporting both as "3-way green" reads as one level of assurance and delivers two.** Where a spec's own text distinguishes them — *"landed 3-way GREEN, injected-reader"*, *"byte-pinned"* — it is making exactly this distinction, and it should be made everywhere rather than where an author happened to remember.

**And per `GUIDE-CONFORMANCE` §5.2b.1, a pinned item states its satisfaction mode when the state it needs is not constructible** — wire-driven where the enabling surface exists, in-process with a declared exclusion and an executed-and-dated mutation otherwise. **An item pinned without that check is a rule that cannot be discharged, which the next implementer discovers instead of the author.** *(Both rules recorded here 2026-08-19 after an extension pinned two behavioral checks as bare "vectors," one of them against a configuration surface that had no write operation to drive it — so the check was unconstructible and the MUST it served had no surface to bind to.)*

**Availability-descriptor vectors (§8.2a).** A surface using the descriptor pins **both** conditional-field
rules, not only the four states — the conditional rules are where implementations diverge:

- one vector **per `state`** — `value` / `unsupported` / `denied` / `unknown`;
- a **`coarse`-fidelity** vector (`{state: "value", fidelity: "coarse"}`), pinning the coarsen signal;
- a **`value`-with-`fidelity`-omitted** vector, pinning the **default-`exact`** rule — a reader must read
  an omitted `fidelity` as `exact`, never as absent or unknown;
- the **omit-not-null** rule exercised directly: a non-`value` state serializes with `value` **absent**,
  never `value: null`. **This one needs its own vector because it is invisible to a reader that only
  checks `state`** — a peer emitting `value: null` passes every state assertion and still breaks a
  consumer that branches on field presence.

### 8.5a The conformance inventory — one row, one requirement, one stable id

**A requirement is addressable or it is prose.** A citation of the form `SPEC §9.1` names a
section, and a conformance section routinely holds a dozen independently failable obligations —
so a check that cites one has not said which requirement it drives, and a requirement that no
check drives cannot be distinguished from one that several do. **Both are unanswerable today at
the extension tier, and both become mechanical once a row has a name.**

**The shape `[MUST]`:**

```
## N. Conformance

**Requirement id prefix:** `HIST`

### N.1 Requirements

| id | Requirement | Level | § |
|---|---|---|---|
| `HIST-R1` | Store transition entities in the content store | MUST | §3.1 |
| `HIST-R2` | Store head pointers at `system/history/head/{path}` | MUST | §3.2 |
| `HIST-R7` | Support `max_depth` retention | SHOULD | §3.3 |
```

**One row is one requirement `[MUST]`.** A row needing the word "and" between two independently
failable obligations is two rows. This is the rule the whole inventory rests on: a row that
bundles three obligations under one id gives a check set one name for three different failures,
which is the same defect as no name at all, wearing an id.

**`Level` is a closed vocabulary `[MUST]` — six values, no others:**

| Level | Meaning |
|---|---|
| `MUST` | required for conformance |
| `MUST NOT` | prohibited; **a requirement, and as checkable as a MUST** |
| `SHOULD` | required absent a stated reason (§4) |
| `SHOULD NOT` | discouraged absent a stated reason |
| `MAY` | permitted; a peer omitting it is conformant |
| `IMPL-DEFINED` | **the specification declines to constrain this surface** — a deliberate statement, and a row a check set MUST NOT test |

**`MUST NOT` is a level and not a fourth group.** Filing a prohibition under a heading named
*Implementation-Defined* — which several specifications did under the superseded shape — puts a
binding rule in the position a reader has been taught means *unconstrained*, and two
implementers reading the same section can reasonably reach opposite conclusions about whether it
binds them. A level attached to the row cannot fail that way.

**The id is `<PREFIX>-R<n>` `[MUST]`:**

- **The prefix is declared by the specification**, on its own line above the table. It is not
  inferred from the filename: a reader must not have to guess, and a tool inferring it is
  guessing on the reader's behalf.
- **One prefix per specification, covering requirements and conformance items alike.** Where a
  specification already uses a prefix for its conformance-item identifiers, that is the prefix.
- **`n` is allocated sequentially, never reused, and never renumbered `[MUST]`.** Rows may be
  added, reordered or retired freely; the number does not move. **A retired row keeps its id and
  its text, marked retired** — a citation that resolves to *"this was a requirement and no longer
  is"* is strictly more useful than one that resolves to nothing, and the table's growth is
  bounded by the specification's own history.
- **`R` distinguishes a requirement from a conformance item.** `ROUTE-R3` is a requirement;
  `ROUTE-EXACT-1` is an item that may exercise it. The `SUBJECT-CONDITION-N` item form is
  unchanged.

**A conformance item names the requirement ids it drives `[SHOULD]`:**

```
- `ROUTE-EXACT-1` — exact `match` → forward to `via`. **Drives:** `ROUTE-R2`, `ROUTE-R3`.
```

`SHOULD` rather than `MUST` because the two sides have different authors and land at different
times — an item authored before its requirement rows exist is not a defect. **What the annotation
buys is the pair of counts nobody can produce today: requirements no item drives, and items
driving no declared requirement.** The first is a rule the ecosystem believes it enforces and does
not. The second is a check asserting a private opinion — and a check asserting the wrong thing
passes exactly as green as one asserting the right thing.

**Numbered conformance levels compose with this and are not superseded.** Where a specification
defines tiers, the tier is a **column**, not a second sectioning axis; the rows carry both.

**Converting an inventory is a shape change and nothing else `[MUST]`.** Where converting one
surfaces a requirement that is really two, or one no check can reach, **that is a finding to
file, not a thing to fix inside a formatting edit** — silently splitting or dropping an
obligation under cover of reformatting is how a normative change lands with no proposal behind it.

### 8.6 A capability has a valid floor `[MUST]`

**A spec that defines a capability MUST define what a participant without it
does, and that participant MUST be valid rather than degraded.** Capability
*adds*; it is never *assumed*.

A specification that reads correctly only on a peer that has everything has not
been written against this system. Participants differ in what they have
installed by design — that is what an optional capability layer *is* — so
"assume the feature is present" is not a simplification, it is an unstated
dependency that surfaces as an interoperability failure at somebody else's seam.

### 8.7 A published vocabulary is a compatibility contract `[MUST]`

**A spec's published type vocabulary is a commitment. All data valid under a
previous version MUST remain valid under the current one, and data produced under
the current one MUST remain valid under the previous.** Concretely:

- **new fields are optional**
- **a field's type never changes**
- **a field is never renamed**
- **a tag is never repurposed**
- **a breaking change takes a new type tag**

A spec that must break compatibility **mints a new name and leaves the old one
meaning what it meant.**

**Why this is a `MUST` and not advice.** The ecosystem holds the *wire* half
already — unknown fields are MUST-ignore and the locked core is never renumbered
— and held nothing equivalent for a **type vocabulary**, which specs at every
tier mint constantly. **A naming divergence has no discovery path except somebody
reading both implementations**, because a type-filtered query using the wrong tag
returns a correct, complete, **empty** answer. It never surfaces as a byte
mismatch, because the two sides never hold each other's data at all.

***Enforcement point:*** the type tags a spec pins are greppable, and a shape
change fails **cross-impl comparison** — so a rename or a type change fails a
byte comparison instead of passing silently as an empty query. **The comparison
is the implementations'**; what the spec owes is that the case is *named* in its
conformance inventory (§8.5a), so there is a comparison to run.

### 8.8 Grow by handler or renderer, not by entity type `[MUST]`

**A new entity type is minted only when a conformant consumer must *behave*
differently, and a proposal minting one MUST name that behaviour.** Where a
spec's extension point is open — a body type, a handler, a renderer — new
products go there.

This is the anti-explosion gate. Without it every subject-matter category argues
for its own type, and the vocabulary a third party has to implement grows without
bound while consumer behaviour does not. **The test is behavioural, not
taxonomic:** two things that a consumer treats identically are one type
regardless of how differently a human would describe them.

### 8.9 A disposition property lives on the entity, never on its container `[MUST]`

**A property governing what a consumer may *do* with an entity — may it be
republished, must it be warned over, may it be cached — MUST live on the entity
itself, never on a collection, conversation, index, or mirror that contains it.**

***Containers do not travel; entities do.***

**The live case that produced it:** a message from a closed conversation, lifted
into a feed, carries no *do-not-mirror* bit — because the policy lived on the
conversation and the conversation stayed behind. The failure mode is bytes that
**verify**, in a tool that will republish them, with the **disposition missing
rather than wrong**, which no integrity check can catch. It generalizes to every
spec that mints a container.

---

## 9. Document Lifecycle

| Status | Meaning |
|--------|---------|
| Draft | Under development. Subject to change. |
| Active | Stable. Implementations should target this version. |
| Superseded | Replaced by a newer version. Reference only. |

### 9.1 Version bumps

**A document's version is a claim about that document's content.** The operative test is the
consumer's question, and it is the same one for every arm below:

> **The version moves when a conformant implementation of the previous text could be non-conformant
> under the new text.**

- **Minor** (X.Y → X.Y+1): **Additive** changes — new optional fields, new MAY requirements.
- **Minor** (X.Y → X.Y+1): **Correction** — the text changes what an implementation must emit,
  accept, refuse or compute, without adding a feature and without breaking a consumer that
  implemented what the document *meant*. A rule stated more precisely, a restatement re-pointed at
  its authority, a defect found by an implementation and fixed in place: all corrections, all bump.
- **Major** (X.Y → X+1.0): **Breaking** changes — new required fields, changed algorithms, removed
  types.
- **No bump**: **editorial** changes — typography, reflow, a moved narrative section, or a citation
  corrected to an authority whose rule is unchanged. **Nothing an implementation does could differ.**

**Where a document's version carries a trailing component managed separately from its release
number** — `ENTITY-CORE-PROTOCOL`'s fourth — the arms above apply to that component, and the
components above it are not the author's to move.

### 9.2 What may exempt a version bump: nothing

**The cause of a change decides which commit trailer applies. It never decides whether the version
moves.** A change discovered by an implementation rather than proposed by an author is a fact about
how the work reached the document, and the process carve-outs that recognize it
(`Spec-Change: cohort-finding`, `Spec-Change: hygiene`) exempt a **commit** from a **process**
obligation — proposal-first — and nothing else.

**A process carve-out can exempt a commit from a process obligation. It cannot exempt a claim from
being true.** A version header that does not move while the obligations under it do is a false
statement about content, and the reader it misleads is the one furthest away: a downstream consumer
pinning the document sees nothing move and is wrong about a requirement.

⚠ **The version answers one question and is not asked to answer two.** It says *the obligations in
this document moved*; it does not say which, how far, or how many times. A consumer needing byte
identity of a particular snapshot wants a digest, which is a different artifact answering a
different question. **Which specific obligations moved belongs in `## Document History` and in the
proposal it cites** — adding a second version-like field to carry it would give one fact two homes,
and the boundary between two homes is what drifts (§10.3: a rule belongs in one place and nowhere
else).

---

## 10. Document Kinds and Authority

**Kind and authority are independent axes.** What a document is *for* does not
determine whether it *binds*. A guide may be binding; a document that governs a
whole family may be advisory. Both axes are **declared** in the header (§5.3),
never inferred from a filename, a directory, or the other axis.

### 10.1 Kind — what the document is for

| Kind | Answers | Example |
|------|---------|---------|
| `normative-spec` | What must an implementation do? | `EXTENSION-TREE.md` |
| `authoring-standard` | What must a *document* be? | this document, `STYLE-NAMING-CONVENTIONS.md` |
| `tier-standard` | What must a member of this family be? | `guides/GUIDE-APPLICATION-DEVELOPMENT.md`, `guides/GUIDE-EXTENSION-DEVELOPMENT.md` |
| `process` | How does work move between parties? | `guides/GUIDE-CONFORMANCE.md` |
| `subject-guide` | How do I use this surface? | `guides/GUIDE-QUORUM.md`, `guides/GUIDE-REVISION.md` |
| `architecture` | Why is it this way? | `SYSTEM-ARCHITECTURE.md` |

### 10.2 Authority — whether it binds

`binding` · `informative`. A `binding` document's requirements are conformance
obligations and its changes are versioned. An `informative` document explains,
motivates, or records; it MUST NOT be the only home of a requirement.

**Authority is stated, not derived.** A subject guide is usually informative and
some are binding; an architecture document is usually informative and is not
required to be. **If the two axes disagree with a reader's expectation, the
header wins** — that is the point of declaring it.

### 10.3 The tier model — what governs a document

Governing standards form **two levels, and only two.**

- **The project tier** — `SPECIFICATION-FORMAT.md` and
  `STYLE-NAMING-CONVENTIONS.md`. These are the corpus-wide authoring standards
  and they **bind every document in the corpus, always.** They are the root, so
  a document does not declare them; a rule that is true of every spec belongs
  here and nowhere else.
- **A sub-tier** — a `tier-standard` governing one family (extensions,
  applications, …). It adds what is specific to that family and **restates
  nothing that is true corpus-wide.**

**Where a rule goes follows directly: true of everything ⇒ the project tier;
specific to one family ⇒ that family's tier standard.** A general rule filed
under whichever tier happened to discover it is the defect this section exists to
prevent — it leaves every other tier ungoverned by a rule that was always true of
them.

**A document with no sub-tier is a member of the project tier.** The top-level
`SYSTEM-*` and `ARCHITECTURE-*` specs are governed by the authoring standards and
by nothing else. That is a positive statement, not a gap.

**A sub-tier standard MAY restate a project-tier rule for its readers, and when it
does it MUST name the authority it is restating** and add nothing to it. A
restatement that does not name its authority is invisible from that authority, so
the next sweep of the rule misses it.

### 10.4 Rationale in a normative document

A `normative-spec` MAY include brief contextual notes (1–2 sentences) explaining
why a rule exists. Extended rationale belongs in an `architecture` document or in
the proposal that landed the rule.

---

## 11. Addressing and Cross-References

A spec does not stand alone — it cites other specs, and others cite it. The set
of specs is therefore a **corpus**: a graph of documents laid over each
document's own section tree. This section defines how a spec addresses itself
and others, what it is permitted to cite, and how an unresolved citation is
classified. It is the corpus-level companion to §5.4 (a single document's
internal numbering).

### 11.1 The Addressing Model

Each document is the canonical **host** for the sections it defines; each
`§N.M` is a **path** under that host. A fully-qualified address is therefore:

```
DOC.md §N.M        ; host (the file) + path (the section)
```

This mirrors the entity address space: the file is the host the way a `peer_id`
is the host of an entity tree, and `§N.M` is the path beneath it. The `.md` is
not decoration — it names the host, which is a real file in the corpus. An
address with no host is implicitly hosted by the current document.

### 11.2 Reference Forms

| Reference | Form | Example |
|-----------|------|---------|
| Same-document section | `§N.M` | "see §3.2" |
| Other spec, a section | `DOC.md §N.M` | "see ENTITY-CORE-PROTOCOL.md §5.2" |
| Other spec, whole document | `DOC.md` | "defined in EXTENSION-TREE.md" |

Rules:

- The canonical cross-document citation form is **`DOC.md`** (with `§N.M` when
  pointing at a section). The `.md` suffix is REQUIRED on a citation — it is the
  host identifier. A bare `DOC` with no `.md` is NOT a citation form.
- A target MUST be cited in one form across the corpus. Mixed `DOC` / `DOC.md`
  citations of the same target are **citation-form drift** and MUST be
  normalized to `DOC.md`.
- Section numbers use the **target's** numbering, including any amendment-letter
  form the target uses (`§N.Ma`).
- A prose name is not a citation. "the core protocol spec" is prose; the
  citation is `ENTITY-CORE-PROTOCOL.md`. Specs SHOULD give the citation, not
  only the prose name, wherever a reader would need to find the source.
- Naming a file as a filesystem artifact (e.g. in a code block, a path, or a
  `Depends` field) is not a citation and is exempt from these forms.

### 11.3 What a Spec May Cite

A normative spec references exactly three kinds of target:

| Target | Resolves to | Status |
|--------|-------------|--------|
| Another normative spec | a `.md` file in the corpus | MUST resolve |
| An architectural / guide document | a `.md` document outside the normative set | Informational; allowed |
| Anything else | — | MUST NOT |

The third row is the rule that matters for release: process and team artifacts —
review notes, session handoffs, arch-team update memos, status docs — MUST NOT
be cited from normative text. A ratified proposal MAY be named as historical
provenance, but only in a non-normative note (see §10), never as the source of a
requirement. The requirement lives in the spec; the proposal is how it got
there.

### 11.4 Unresolved References

A citation whose target is not a file in the corpus ("dangling") is one of three
things, and the disposition differs by class:

| Class | Meaning | Disposition |
|-------|---------|-------------|
| **Forward** | A spec planned but not yet written | Permitted ONLY when explicitly marked `(planned)` and confined to non-normative notes or an Extension Points section. An unmarked forward citation is an error. |
| **Stale** | A spec that was removed, renamed, or scope-cut | MUST be removed, or redirected to the superseding spec. |
| **Leak** | A name that is not a spec at all (a process/internal artifact) | MUST be removed — see §11.3. |

A qualified citation `DOC.md §N.M` whose document resolves but whose section
does not (a **stale section ref**) MUST be corrected to a section that exists in
the target.

### 11.5 Tooling

These rules are mechanically checkable. `spec-topology` builds the
canonical-name registry (file stem = host), resolves every citation against it,
and flags citation-form drift, dangling references, and stale section refs. The
addressing standard is the contract; the tool reports deviations from it. A
green corpus is one where every citation resolves, in the canonical form, to a
permitted target.

---

## Document History

**v1.4:** §10 rewritten on two axes — §10.1 six document kinds, §10.2 authority (`binding` / `informative`) **declared and never derived from the kind**, §10.3 the two-level tier model (a **project tier** binding every document, and **sub-tiers** adding what is specific to one family), §10.4 rationale. §5.3 gains `Kind`, `Authority` and `Governed-by`. §8 retitled *Spec Conventions* and gains **§8.6** a capability has a valid floor · **§8.7** a published vocabulary is a compatibility contract · **§8.8** grow by handler or renderer, not by entity type · **§8.9** a disposition property lives on the entity, never its container. **No existing §8 number moved.** The four are promotions of rules that were true corpus-wide and were filed under the one tier that discovered them.

**v1.3:** §5.1's conformance block and new §8.5a — the inventory is a table of individually identified requirements, `Level` is a closed six-value vocabulary, and each row carries a stable `<PREFIX>-R<n>` id that is allocated once and never reused. A conformance item names the requirement ids it drives.

**v1.2:** §4.1, new — a requirement that constrains a party, a namespace or a path states the failure it prevents. §8.5's conformance-class list follows `GUIDE-CONFORMANCE` §7.0 from three classes to four (the host-seam check, §7d).
