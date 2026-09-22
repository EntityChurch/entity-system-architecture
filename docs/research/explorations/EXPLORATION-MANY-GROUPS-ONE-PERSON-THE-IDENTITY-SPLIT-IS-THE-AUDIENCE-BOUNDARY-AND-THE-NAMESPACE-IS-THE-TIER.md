# Many groups, one person — the identity split is the audience boundary, and the namespace is the tier

**Status:** EXPLORATION (2026-09-13). **Worked concretely against landed text.** The question: *I belong to several groups that share sites, plus I have private sites of my own. How do the permission grants work, what does the tree look like, and how far does this get before encryption is required?*

---

## §0 The answer in four sentences

1. ⭐⭐⭐ **You do not partition one identity into per-group namespaces. You run one IDENTITY per audience** — `EXTENSION-GROUP` §2 makes a group an identity outright, and *"a host machine may operate as multiple agents, one per identity it serves."*
2. ⭐⭐ **Namespaces are for TIERS WITHIN one audience** — `public/` · `internal/` · `relationships/{contact}/` are already specified with their propagation mechanisms (§4.2). **Two different jobs; both are used.**
3. ⛔⭐⭐ **The reason the split is by identity and not by namespace is the STATIC surface:** a peer publishes exactly one signed root, at one path, over one `prefix` — **and the root's `prefix` IS the audience boundary**, because the closure a consumer walks from it is the whole subtree.
4. ⇒ ***On the live surface, capability scoping gets you all the way — arbitrary groups, arbitrary overlap, per-contact granularity, no encryption needed. Encryption becomes necessary for exactly two things: more than one audience out of ONE static view, and an untrusted host.***

---

## §1 The tree, concretely

**Alice is in a knitting group and a cycling group, and has private sites of her own. Her laptop runs three agents.**

```
/{alice}/                                        ← her own identity
    system/content/public/{hex(H)}               ← world-readable blobs
    system/content/private/{hex(H)}              ← her devices only
    sites/…                                      ← her own sites (app tier)
    system/peer/published-root                   ← ONE root, prefix = system/content/public/

/{knitting_grp}/                                 ← the knitting group IS a peer identity
    system/group/{knitting_grp}/
        identity/quorum/…                        ← visible to nobody outside
        identity/internal/…                      ← members only
        identity/public/…                        ← all contacts
        identity/relationships/{bob}/…           ← only Bob
        members/{member_peer_id}/{hash}
    system/content/shared/{hex(H)}               ← the group's blobs
    sites/…                                      ← the group's shared sites
    system/peer/published-root                   ← ITS OWN root, its own prefix

/{cycling_grp}/                                  ← same again, independently
```

⭐ **Three identities, one machine.** `EXTENSION-GROUP` §2: *"Multiple groups can run on the same agent… a host machine may operate as multiple agents, one per identity it serves."* **`EXTENSION-IDENTITY`'s per-runtime-peer `peer-config` is the mechanism, and it is landed.**

⚠ **Note what is NOT in the tree: a `shared/knitting/` namespace under `/{alice}/`.** That shape is available and it is the wrong one for a *group* — it makes Alice the authority for the group's content, puts the group's audience inside Alice's root, and gives the group no identity of its own to rotate, publish or be resolved by. **Per-audience namespaces under one identity are for audiences that are all facets of the SAME principal** — Alice's public vs private, or a company's public vs internal.

### 1.1 The two splits, and they do different jobs

| | **the namespace split** | **the identity split** |
|---|---|---|
| answers | *which tier of THIS principal's data is this?* | *whose data is this, and who is it published for?* |
| examples | `public/` · `internal/` · `relationships/{contact}/` · `content/public/` vs `content/private/` | Alice · the knitting group · the cycling group |
| enforced by | the **`resources`** scope on a cap (include / exclude, patterns) | separate keys, separate trees, **separate published roots** |
| granularity | as fine as you like, per contact | one per audience that needs its own static view |
| specified at | `EXTENSION-GROUP` §4.2, `EXTENSION-IDENTITY` §4.2 | `EXTENSION-GROUP` §2 |

⇒ **Namespaces for tiers within an audience. Identities for audiences that need their own root.** *Neither substitutes for the other, and reaching for the wrong one is the mistake this document exists to prevent.*

---

## §2 The grants, concretely

### 2.1 ⭐ Inbound reads: `peers` is UNREACHABLE, so the answer is `resources`

**This is the counterpart to the outbound case and it is the one that governs "who may read my group's content."**

`ENTITY-CORE-PROTOCOL` §3.5: *"On **inbound dispatch**, the refusal above runs at canonicalization — before handler resolution and before `check_permission`. So `target_peer == local_peer_id` always holds by the time Dimension 4 runs on this path. **The check is not redundant here; it is unreachable here.**"*

⇒ ***Who may read is expressed in `resources`. `peers` says nothing about it.*** A grant restricting *whose* data you may reach is about **outbound sub-dispatch**; a grant restricting *who may reach yours* is about the resource paths you hand out. **Two different questions that look like one.**

**The cap the knitting group issues to a member:**

```
handlers:   {include: ["system/content", "system/tree"]}
operations: {include: ["get"]}
resources:  {include: ["/{knitting_grp}/system/content/shared/*",
                        "/{knitting_grp}/sites/*",
                        "/{knitting_grp}/system/group/{knitting_grp}/identity/internal/*"]}
; peers:    absent — this is an inbound read; Dimension 4 is unreachable
```

**The cap for a contact who is not a member** — same handler, same operation, a narrower resource set:

```
resources:  {include: ["/{knitting_grp}/system/group/{knitting_grp}/identity/public/*"]}
```

**And Bob, who gets one thing nobody else does:**

```
resources:  {include: ["/{knitting_grp}/system/group/{knitting_grp}/identity/public/*",
                        "/{knitting_grp}/system/group/{knitting_grp}/identity/relationships/{bob}/*"]}
```

⭐ **Three audiences, one handler, one operation, three resource scopes.** *The whole differentiation is in one dimension, and `exclude` is available when carving out is easier than enumerating.*

### 2.2 Alice's own private data needs no group machinery

```
; Alice → her own devices
resources:  {include: ["/{alice}/system/content/private/*", "/{alice}/sites/*"]}
peers:      {include: [alice_laptop, alice_phone]}      ; outbound: only her devices
```

⭐ **This is the fleet case from the locator work, and it is the same two dimensions doing the same two jobs: `resources` pins the shape, `peers` pins the party.**

### 2.3 Overlapping membership costs nothing

**Alice in both groups holds two caps, issued by two different granters, rooted at two different identities.** They do not interact, do not need reconciling, and neither group learns of the other from anything in the capability system. ⇒ ***multi-group membership is not a feature; it is the absence of one.*** **What would have needed designing is a single cap spanning both groups, and nothing wants that.**

---

## §3 ⛔ Where the static surface stops, and why the design is identity-per-audience

**This is the constraint that explains §0's first sentence.**

### 3.1 One root, one path, one prefix

`EXTENSION-TREE` §3.3a: the published root is bound at **`{peer_id}/system/peer/published-root`** — *"The peer's current published root is bound at that path **and nowhere else**."* It carries a **`prefix`**: *"REQUIRED. The prefix these trie keys are relative to… `/` designates the universal tree."*

⇒ **One signed root per identity, over one prefix.**

### 3.2 ⭐⭐ The prefix IS the audience boundary — not a leak, a definition

`EXTENSION-NETWORK` §6.5.6 Amendment 10: a publisher advertising a `signed_pointer` **MUST** resolve, under the same `serve_scope`, *"the **transitive trie-node closure** reachable by hash-link from `published-root.root_hash` — the trie root node + all interior nodes + all leaf-bound content hashes + the `published-root` entity + its signature."*

⭐⭐ **That closure is walkable, and walking it is the whole point of the threat model** — *"a consumer fetches it, verifies the signature, and walks the hash-chain from `root_hash` — never trusting paths the host claims outside that chain."*

⇒ ***Serving a signed root's closure serves everything under its prefix.*** **So the root's `prefix` is not a hint about scope; it is the access boundary of the static view, exactly.** A root over `/` published to one audience is published to that audience *entirely*.

### 3.3 ⛔ And the static read surface has no per-reader authorization at all

`EXTENSION-NETWORK` §6.5.6: *"The read routes carry **no request auth** (the client may not speak the protocol) — hash-knowledge (content) / path-presence (tree) is the read authority… **the published `serve_scope.cap` IS the effective cap** passed to the evaluator."*

⇒ **A static view has exactly one audience, decided once at publish time.** There is no reader to distinguish; the cap is a property of the *view*, not of a request.

### 3.4 ⇒ The conclusion, and the constraint agrees with the design

**One root per identity + the prefix as the boundary + no per-reader auth on the static surface ⇒ one static audience per identity.** Two audiences that each want a verifiable, CDN-servable, walk-from-signed-root view **need two identities.**

⭐ ***Which is exactly what the group extension already specifies.*** **The constraint and the design are the same fact seen from two sides** — and that is the strongest evidence in this document that the shape is right rather than convenient.

> ⚠ **One tension worth stating, because nothing states it.** `serve_scope` is declared *"which slice of that local view **this listener** answers for"* — per-listener, so a peer could in principle run several listeners with different scopes. **But Amendment 10 requires the closure of the published root to be resolvable under the same `serve_scope`**, so a listener whose scope is *narrower than the root's prefix* cannot serve a `signed_pointer` consumer. ⇒ **per-listener scoping and signed-root serving compose only when the scope covers the root's prefix.** A one-clause clarification is owed; see §6.

---

## §4 What leaks without encryption, honestly

**Four channels are closed and three are open. The open ones are the answer to *how far does this get*.**

### ✅ Closed, and already specified

1. ⭐ **Listing enumeration.** `EXTENSION-NETWORK` §6.5.6 (Amendment 5) closes three oracles by name: `count` **MUST** be the in-scope filtered total — *"a discrepancy leaks hidden-path existence"* · an out-of-scope or non-existent prefix returns **404 identical to not-held** · `?offset=&limit=` **MUST** run **post-scope**, since *"offset numbering over raw children leaks an offset oracle."* **In-scope-but-empty returns 200, because an empty published directory is legitimately observable.**
2. ⭐ **Content-store dedup does NOT cross the boundary.** `EXTENSION-CONTENT` §6.4.1: `get` *"consults the tree binding and serves only when the hash is bound under the requested namespace"* — and §6.4.2: a null `tree:get` means *"the hash is not bound under the namespace (so `system/content:get` under that namespace returns 404 **even if the hash exists in the content store under a different namespace**)."* ⇒ **one stored copy, N path bindings, N independent access decisions.** *Knowing a hash from one audience does not read it in another.*
3. **Membership visibility.** `EXTENSION-GROUP` §9.6 gives three publication modes — public, internal-to-group, per-relationship — *"Deployments choose per-group what's public, internal, or per-relationship."*
4. **Cross-audience inference from the capability system.** Two caps from two granters share nothing (§2.3).

### ⚠ Open — and these are the boundary

5. ⛔⭐ **One static view = one audience** (§3). *This is the big one and it is structural, not a gap.*
6. ⚠ **Path segments are plaintext.** `shared/knitting` tells anyone who sees a proof involving it that the group exists and what it is called. ⭐ **Cheap mitigation: use an opaque segment when the group's existence is the secret** — the path is a key, not a label, and nothing requires it to be meaningful. **A petname layer already exists for the human-readable side.**
7. ⛔ **A serving intermediary reads everything it serves.** Unencrypted content on a host you do not control is disclosed to that host. ⭐ **And this is precisely where content addressing pays off under encryption:** an intermediary can hold, serve and be verified on **ciphertext it cannot read**, because the hash settles identity without the bytes being interpretable. *That is the property that makes the encrypted case work, and it is why encryption is the answer to an untrusted host rather than to multi-group access.*

---

## §5 So how far before encryption?

| you want | works unencrypted? |
|---|---|
| several groups, different content per group, overlapping membership | ✅ **yes, fully** — one identity per group, `resources`-scoped caps |
| per-contact granularity inside a group (*Bob sees this, Carol does not*) | ✅ **yes** — `relationships/{contact}/`, already specified |
| private data of your own, readable only by your devices | ✅ **yes** — a namespace plus a `peers`-scoped grant |
| your own public site beside all of it | ✅ **yes** — its own namespace, its own root prefix |
| **two audiences served from ONE static/CDN view** | ⛔ **no** — one root, one prefix, no per-reader auth (§3) |
| **serving via a host you do not trust** | ⛔ **no** — the host reads what it serves (§4.7) |
| hiding that a group exists at all, from someone who sees a proof | ⚠ **partly** — opaque path segments help; traffic to the host does not |

⇒ ***Encryption is required for exactly two things, and multi-group access is not one of them:*** **more than one audience out of a single static view, and an untrusted host.** **Everything the question actually asked for works with capability scoping, namespace tiers, and one identity per audience — all of it landed.**

---

## §6 Two findings against landed text

1. ⛔ **`EXTENSION-GROUP` §4.2 cites the wrong section.** Its members row says *"Per the group's deployment policy (see **§10.4** privacy of group membership)"* — **but §10.4 is *Single-Oracle DAOs*, and the membership-privacy content is §9.6.** ⚠ **This class of defect is invisible to every gate available**: a citation that resolves to a **wrong real section** passes a stale-section check, because the section exists. **One-token fix, and worth recording as an instance rather than just repairing.**
2. ⚠ **The per-listener `serve_scope` / signed-root-closure tension is stated nowhere** (§3.4's note). A listener scoped narrower than the published root's `prefix` cannot serve a `signed_pointer` consumer, because the closure will not resolve. **Neither section is wrong; the composition is unwritten.**

---

## §7 What is owed

- **Nothing new to design for the question as asked.** The mechanisms are landed; what is missing is that **no guide walks this** — a reader assembling it needs `EXTENSION-GROUP` §2 and §4.2, `EXTENSION-IDENTITY` §4.2, `ENTITY-CORE-PROTOCOL` §3.5 and §3.6, `EXTENSION-TREE` §3.3a, `EXTENSION-NETWORK` §6.5.6 and `EXTENSION-CONTENT` §6.4.1 **at once**, and the assembly is the hard part. ⭐ **A worked multi-audience deployment belongs in a guide.**
- **The two §6 findings**, both one-clause.
- ⚠ **The encrypted case is out of scope here and stays open** — named so its absence is deliberate. **What this document establishes is where the unencrypted design actually stops**, which is the precondition for designing past it.

---

## Document history

- **2026-09-13** — created. Works the multi-group / private-sites question against landed text and locates the exact boundary at which encryption becomes necessary.
