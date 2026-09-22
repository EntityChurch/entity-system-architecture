# PROPOSAL — the asset position is not special, and a scope that names only the site cannot verify the site

**Proposes:** `APP-CONVENTION-SEMANTIC-CONTENT-SITE` v0.5.2
**Status:** DRAFT (2026-09-13).
**Tier:** applications — `APP-CONVENTION-SEMANTIC-CONTENT-SITE` v0.5.1 → v0.5.2. Two clauses, neither
widening a rule; both naming an authority that already binds.
**Answers:** two application-tier implementation findings on the asset position (§1), and one on a site's
capability scope (§2).

> **In one sentence.** Two implementations spent a week disagreeing about a clause `APP-CONVENTION-REFERENCE`
> §3.4 already rules, and a third clause is unsatisfiable as written because the evidence a site's
> verification rests on lives **outside** the subgraph the clause scopes.

---

## §1 A-24 / B-10 — a leading `/` at the asset position is site-root-relative, and §3.4 has said so

### 1.1 The state of the disagreement

The **one user-visible interop divergence in this arc**: the desktop implementation resolves three asset
reference forms the browser implementation refuses, so **the same published page renders a figure in one
reader and a gap in the other.** Both did the right thing with it — neither narrowed, neither
widened, both waited.

The browser implementation defended the refusal three times as *"a security property — subgraph
confinement"*, then **measured it and withdrew the framing** (`ROUTING-2026-09-12-e`): their asset base
is the site root either way, `/assets/figures/x.png` and `assets/figures/x.png` denote the same bytes,
and every hostile input is still refused with one leading slash removed — pinned as a test rather than
argued. The desktop implementation made the argument first and their wording was better.

### 1.2 The ruling, and it needs no new rule

**`APP-CONVENTION-REFERENCE` §3.4 is the authority and it is unambiguous:**

> **The base is the referring entity's own location.** A link with no scheme is resolved against it: a
> leading `/` is **root-absolute within the current site**; anything else is **directory-relative to the
> current page**.
>
> Producers of application-generated links … **SHOULD** emit root-absolute form, so that a link
> resolves identically from whatever page it is rendered on.

⇒ **The desktop implementation is conformant. the browser implementation's leading-slash refusal is
non-conformant with §3.4 and is a pure loss** — it refuses the form the same paragraph **recommends
producers emit**. Drop that clause; keep the `assets/` prefix and the `..` / `//` / `://` / `data:`
guards, which is what was doing the confinement work all along.

**There is nothing special about the asset position.** An asset ref is a bare string in a document
body, which is exactly what §3.4 governs. Both were looking for a rule scoped to their
position; the rule is scoped to the *form*, one convention over.

### 1.3 ⚠ A correction against ourselves, because we were in this exact place four days ago

The 2026-09-09 fold (`THE-EMBED-REF-IS-A-REFERENCE-AND-F-1-OUTLIVED-ITS-UNION`) retired `EMBED` `F-1`
and **named §3.4 as the authority in the same sentence** — then characterized the refusing
implementation as *"conformant with the reference convention and non-conformant only with the dead
rule."* **That half is wrong.** §3.4 says a leading `/` **resolves**, site-root-relative; an
implementation that refuses it is non-conformant with §3.4, not conformant with it. We identified the
right authority and then misread it in the implementation's favour, and the divergence stayed open for four more
days because nobody was told they had to change.

⇒ ***An authority named is not an authority applied.*** The fold cited §3.4 to kill a dead rule and did
not run the surviving implementations against it.

### 1.4 The change

> **`SITE` §4, new paragraph.** **[MUST]** A link or asset reference appearing as a bare string in a page
> body — including in an embed directive's asset position — is resolved per `APP-CONVENTION-REFERENCE`
> §3.4, which is the authority: **a leading `/` is root-absolute within the current site**, anything else
> is directory-relative to the current page. **This position is not special and this convention states no
> rule of its own about it.**

*(A restatement that names its authority, so a later sweep is a grep rather than a re-derivation.)*

### 1.5 What is NOT ruled here

- **The `content-hash` arm.** Whether a bare multibase/hex binds at the asset position is a **different
  address space** — resolving an asset by content hash with no site scope at all — and neither implementation has
  a preference it can justify. **Still open, and the harder half.**
- **The `entity+ref://` arm.** An absolute `entity+ref://` URI carries a **peer-absolute tree path**
  (§3.1), which is a different base from §3.4's site root. That is consistent, not contradictory —
  **the two forms have different bases by design** — but whether the asset position accepts the atom at
  all is unruled. *This is the conflation in both framings: they called §3.4's bare-string rule and
  §3.1's URI path "the `path` form", and they are two forms with two bases.*
- **`site:`** at the asset position. Unanalyzed.

---

## §2 A-34 — the site's capability scope is right, and it cannot make the site verifiable

### 2.1 What was measured

The desktop implementation measured it in both directions, in a publish test:

| grant | result |
|---|---|
| `system/tree:get` + `system/content:get` over **the site subgraph only** | every page reads; **nothing verifies** — `403 capability_denied` on `system/peer/published-root` |
| the same plus `system/peer/published-root` and `system/signature/*` | reads **and** verifies — 63 keys, 5 CHAMP nodes, signature checked against the publisher's peer-id |

### 2.2 The clause is right about content and silent about evidence

**`SITE` §2:** *"The site's capability scope is the site subgraph's own root, not a slot carved out of
someone else's namespace."* **That is correct and is not being widened.** The scope of the *content*
really is the subgraph.

**But the evidence lives outside it, by design and by two other specs.**
`system/peer/published-root` (`EXTENSION-TREE` §3.3a) and `system/signature/{hex(H)}` (V7 §5.2) are
**fixed locations in the publisher's namespace**, deliberately not inside any subgraph they commit to —
that is what makes one signed root cover a whole tree. ⇒ **the capability scope of *the site* and the
capability scope of *reading the site verifiably* are different sets, and only the first is named.**

### 2.3 Why this earns a sentence rather than being left to implementers

**The failure is silent and it points at the wrong machine.** A reader whose grant covers the site alone
does not see *permission denied on the root*; it sees **a publisher that has never published a root** —
the other party's fault, on the reader's screen, with the grant looking complete and minimal.

⭐ **And the narrow grant is the one a careful implementer writes**, because it is what the clause says
and because least-privilege is the instinct. *The trap is baited with good practice.*

### 2.4 The change

> **`SITE` §7 *Cap scope*, new paragraph.** **The scope of the site's content and the scope of verifying
> it are different sets.** A site subgraph is the capability scope of its **bytes**. The evidence that
> makes those bytes **verifiable** — the publisher's `system/peer/published-root` (`EXTENSION-TREE`
> §3.3a) and the `system/signature/` location of its root (V7 §5.2) — sits **outside the subgraph, in
> the publisher's namespace, by design**: a signed root that lived inside the subtree it commits to
> could not commit to itself.
>
> **[MUST]** A grant intended to make a site **verifiable** covers those two locations in addition to
> the site subgraph. **A grant covering the subgraph alone yields a readable, unverifiable site**, and
> the reader observes it as *"this publisher has published no root"* — **a false statement about the
> publisher**, produced by the reader's own grant. An implementation **MUST NOT** report an
> authorization failure at these locations as an absent published root.

### 2.5 Scope of the fix

- **`SITE` §2's scope sentence is unchanged.** This adds an adjacent obligation; it does not widen the
  content scope, which the desktop implementation explicitly did not ask for and which would be wrong.
- **The `[MUST NOT]` half is the transferable one**, and it is this corpus's standing *could-not-look
  wearing a verdict's clothes* shape, arriving at the capability layer: **a 403 answered as a 404 is an
  authorization result reported as a fact about another party.**
- **Measured on one implementation.** Whether a publisher could place its root somewhere that makes the
  narrow grant sufficient is unchecked — and under `EXTENSION-TREE` §3.3a it cannot, because the path is
  fixed.
- **No public (`default`) grant exists on either seat yet.** That is the next build and it is the one
  where getting this wrong names everybody rather than one peer.
