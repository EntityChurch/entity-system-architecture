# PROPOSAL — `path_required` belongs in the core status table, because the condition that raises it is core's

**Status:** **IMPLEMENTED (2026-09-08)** — folded the same session; kept as the reference proposal.
**Target:** `ENTITY-CORE-PROTOCOL.md` §3.3's 400 row — enumerate `path_required` beside the five
more-specific 400 codes already named. Consequential edits: `GUIDE-EXTENSION-DEVELOPMENT` §4.1
(stops claiming to be the authority for a core status code) and `EXTENSION-CONTENT` §6.2 / §6.3
(two citations that name a **line number**).
**Raised by:** a seat generating extension implementations from the specs, which hit the
contradiction while building a code set and could not resolve it from the corpus.

---

## 1. The contradiction, in four documents that are each internally consistent

**`EXTENSION-CONTENT` MUSTs the code, twice**, at `system/content:get` and `system/content:ingest`:

> Calling `system/content:get` without a `resource` **MUST** return **`400 path_required`**.

**`EXTENSION-CONTENT`'s own Appendix A declares itself the closed set for that handler**, and does
not contain the code:

> a more-specific code is conformant only where one is **defined for the operation in a spec code
> set**, and this table is that set for this handler. An undefined spelling is non-conformant and
> falls back to the status's default.

Six rows. `ambiguous_input`, `missing_input`, `hash_mismatch`, `blob_not_found`,
`blob_pending_sync`, `capability_denied`. **No `path_required`.**

**`ENTITY-CORE-PROTOCOL` §3.3's 400 row enumerates five more-specific spellings** — `invalid_path`,
`invalid_params`, `unexpected_params`, `chain_depth_exceeded`, `signature_path_conflict` — and
`path_required` is not one of them.

**`GUIDE-EXTENSION-DEVELOPMENT` §4.1 says it is**:

> **This section is the authority for that status.** `path_required` is a *more-specific 400* under
> `ENTITY-CORE-PROTOCOL.md`'s 400 row, **which sanctions them by name** alongside `invalid_params`
> and `unexpected_params`.

**That sentence is false about this code.** The 400 row sanctions **five spellings**, by name — not
the *category* of more-specific 400s. A code not on that list and not in an extension's own set is,
by the rule Appendix A states, non-conformant.

**Nobody was careless.** Each document is right about the part of the corpus it can see. The
contradiction lives in the seam, which is exactly where no single document's review looks.

---

## 2. What it costs today, and it is not theoretical

Measured, live, at the time of filing:

- **All five implementations emit `path_required`** — three generated ports and both ground-up
  reference implementations.
- **Two conformance checks assert the literal string** (`get_path_required`,
  `ingest_path_required`).

**So a peer that obeys `EXTENSION-CONTENT` Appendix A fails the conformance suite, and a peer that
passes the suite emits a spelling its own extension calls non-conformant.** Both readings are in the
corpus, in normative text, and the suite arbitrates in favour of the one the specification does not
license.

**This is not a divergence to leave to implementations.** It is a wire-observable token a caller
branches on. The remedy differs — *supply a resource* is a different instruction from *fix your
request* — which is precisely the argument §4.1 already makes for the code existing at all.

---

## 3. Two candidate fixes, and they are not equivalent

### 3.1 Add the row to `EXTENSION-CONTENT` Appendix A

`(any) | path_required | 400 | a directly-callable op was called without its resource`

Cheap and local. **And it repeats itself in every extension appendix written after it**, because
the condition is not this extension's. It is *"a directly-callable op was called without its
resource"* — which is a rule stated in the **core protocol §3.2**, applying to every op in the
system. Twenty-six extensions would each restate a core condition in their own words, and the
corpus already knows how that ends: one rule with many normative homes drifts at the homes nobody
edits (`SPECIFICATION-FORMAT` §8.4.3's argument, one layer up).

### 3.2 Enumerate it in `ENTITY-CORE-PROTOCOL` §3.3's 400 row — **proposed**

One edit. It makes an **already-published claim true** rather than adding a new rule: §4.1 asserts
today that the core row sanctions this code, and after this change it does. Every extension
appendix inherits it for free, and no extension has to restate a core condition.

**The test that decides between them: whose condition is it?** A code belongs in the set owned by
the document that defines the condition raising it. `hash_mismatch` is content's, because only
content's ingest can produce it. `path_required` is raised by the **dispatcher**, before any
handler runs, on a rule the core states — so it is core's, and locating it in an extension is
locating it where it cannot be complete.

---

## 4. The edits

### 4.1 `ENTITY-CORE-PROTOCOL.md` §3.3, the 400 row

Add `path_required` to the enumerated more-specific 400 codes, with the condition stated in one
clause:

> `path_required` — a **directly-callable** op was invoked with no `resource` (§3.2). Raised at
> dispatch, before the handler runs; available to every handler for that reason, and never a
> synonym for `invalid_request` — the remedy differs, and the code is what selects it.

**Fourth-component version bump**, which is what that component is for: it signals that the core text
moved, without inventing a release.

### 4.2 `GUIDE-EXTENSION-DEVELOPMENT` §4.1 — stop being the authority

Two corrections in one paragraph, and the second is the one that matters:

- *"which sanctions them by name"* → the row **names** `path_required`, once §4.1 lands. The
  sentence stops being a claim about a category and becomes a claim about a list, which is what the
  row actually is.
- *"This section is the authority for that status"* → **it is not, and a guide never is.** A guide
  teaches; it does not own a wire-observable status code. The authority is the core status table.
  **This is the shape worth naming: a rule stated in a non-normative document acquires normative
  weight from being cited, and then the citations are what keep it alive** — two shipped extension
  `MUST`s point at this paragraph today.

### 4.3 `EXTENSION-CONTENT` §6.2 and §6.3 — two citations name a line number

Both `MUST` sites cite the authority as **`GUIDE-EXTENSION-DEVELOPMENT.md` §171**. There is no
§171. **171 is the line number** the paragraph happened to sit on when the citation was written;
the section is **§4.1**. A line number is not an address — it is invalidated by any edit above it,
silently, and no gate in this corpus resolves one.

Both citations become `GUIDE-EXTENSION-DEVELOPMENT.md` §4.1 — and, once §4.1 lands, the core §3.3
row beside it, since that is where the code is defined.

**No row is added to `EXTENSION-CONTENT` Appendix A**, deliberately. Appendix A is *this handler's*
code set; `path_required` is every handler's, and duplicating it there would re-create in one
extension exactly the many-homes problem §3.1 rejects.

### 4.4 The scope was wider than §4.1–§4.3 — enumerated by the rule's SUBJECT, not its tokens

**Writing §4.3 surfaced that this proposal had enumerated its homes by searching for the token
`path_required`.** The rule's subject is *"an extension appendix must not define a code the core
defines"*, and searching by subject rather than token found **four more rows in two appendices**:

| Appendix | Restated core code | Was |
|---|---|---|
| `EXTENSION-HISTORY` | `path_required` (400) | **carrying a false authority claim** — *"the authority for this status is the extension-development guide's error-code section"*, which the guide did not have |
| `EXTENSION-HISTORY` | `unexpected_params` (400) · `unsupported_operation` (501) · `storage_error` (500) | each labelled *"§3.3's enumerated specific"* — correctly describing itself as a copy |
| `EXTENSION-CONTENT` | `capability_denied` (403) | §3.3's 403 default |

**Each of those rows was a second normative home for a core rule**, in a table whose own preamble
says it is *the* code set for the handler — which is the failure this proposal is about, sitting
inside the documents raising it.

**They are removed as rows and kept as information**, which is the distinction that matters: each
appendix now carries one sentence naming the core codes that handler emits and saying they are
defined in §3.3 and not there. An implementer still learns which codes arise; nothing is defined
twice.

**Both appendices also gain the rule that made the original contradiction possible:**

> **This table is closed over the codes this handler DEFINES, not over every code it may emit.**

Without that sentence, *"this table is that set for this handler"* plus *"an undefined spelling is
non-conformant"* reads as excluding the core's own codes — which is how a handler came to `MUST` a
spelling its own appendix forbade.

---

## 5. What this does not do

- **It does not change any implementation's behaviour.** Five emit `path_required` today. This
  makes the specification agree with them, which is the direction that costs nothing to adopt and
  is the reason to do it now rather than after a sixth implementation guesses differently.
- **It does not open the 400 row to arbitrary extension codes.** The row remains an enumeration.
  This adds one entry, for a condition the core itself raises.
- **It does not touch the locked wire core.** A status code's spelling is not a wire-format
  renumbering; §3.3's table is exactly where such a spelling is declared.

---

## 6. Open item

**Does the same argument reach any other code an extension `MUST`s but no code set defines?** The
filing seat found this one by building a code set from the specs, which is a search nobody else has
run. **The question is worth a sweep and is not part of this proposal** — this proposal fixes the
instance, and the sweep is `spec census`'s `unobserved-must` axis pointed at the code sets rather
than at the checks.
