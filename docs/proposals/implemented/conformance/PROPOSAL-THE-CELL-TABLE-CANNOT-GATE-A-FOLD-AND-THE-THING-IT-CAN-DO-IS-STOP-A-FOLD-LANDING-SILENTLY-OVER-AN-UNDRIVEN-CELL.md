# The cell table cannot gate a fold, and the thing it CAN do is stop a fold landing silently over an undriven cell

**Status:** **RATIFIED and FOLDED — `GUIDE-CONFORMANCE` §5.3a, §5.3a.1, §5.3a.2, §5.3a.3.** `CD-2` is resolved: **it binds now**, forward from `ENTITY-CORE-PROTOCOL` `0.8.2.32`.
**Author:** architecture team
**Answers:** `KB-18` / `F73`'s gating half, asked by `entity-core-keystone` four times
**Corrects:** this team's own `F73(a)` condition, ruled 2026-09-12

---

## 0. What is owed, and for how long

`entity-core-keystone` asked arch to **adopt the 146-cell scope table as the gating artifact** — the
rule that says when a fold may land. It has been asked in four rounds. Arch's stated reason for holding
it each time was honest and is still true: *a process ruling about when a fold may land is bigger than a
retraction round, and deciding it inside one would bury it.*

**The fourth asking arrived with a number, and the number is why this is being answered now.**

---

## 1. The measurement that makes this unignorable

`entity-core-keystone` closed a month-long arc across five defect families and measured the whole
roster in one run, report-age span **0.04 h**:

| family | before | after |
|---|---|---|
| **B** — `K1` forgery | 7 of 45 reproduced the forgery | 46 of 46 clean |
| **E** — grant `exclude` at dispatch | 4 peers never read it | 46 of 46 |
| **F** — Dimension 4 | 4 failing | 45 of 46 |
| **A** — §3.3 ladder | **0 of 45** | 44 of 46 |
| **G** — §6.3 `check_path_permission` | **ZERO working instances** | 44 of 46 |

**All of that moved 3 of 35,788 conformance severities** — one real (`turbowarp` CAP-6a WARN→PASS,
which §4.11 earned) and two timing oscillations they declined to bank.

⇒ **The executed check set cannot distinguish a cohort in which §6.3's designated sole enforcement has
ZERO working instances from one in which it has 44.** Nor four peers honouring an attenuated capability
as unattenuated from none. Nor the `K1` forgery reproducing on seven peers from none.

**That is not an argument about process. It is a measurement of what green means**, and it was produced
by the seat whose own instrument reports the green.

---

## 2. ⛔ The condition this team already ruled is unmeetable, and that is the first thing to say

`F73(a)`, ruled 2026-09-12:

> ✅ **ADOPTED**, with one condition. **A fold lands when its cells are filled and their vectors are
> driven green**, not name-mapped.

**Measured against the census that same ruling reproduced** — `640 → −320 → −32 → −16 → −126 → 146`:

| | cells |
|---|---|
| total | **146** |
| carrying a **named vector** | **37** |
| **unmeasured** | **109** |
| ruled-but-ungated | 32 |
| known DEFECT | 1 |

**109 of 146 cells have no vector at all**, and the ruling's own next clause — *a cell counts covered
only when a check has been DRIVEN against it* — makes the 37 an upper bound rather than a count.

⇒ **Read literally, `F73(a)`'s condition blocks every fold, indefinitely.** Nothing has landed under it
because nothing could. A condition that cannot be met is not a strict gate; **it is an unenforced one**,
and the difference between those two is invisible from the outside — which is exactly why four folds
have landed since it was ruled without anybody noticing they were landing under a condition that
forbade them.

**This proposal corrects our own ruling.** Not keystone's ask — theirs was *adopt the table*, and the
table is adopted. The condition arch attached to the adoption is the defective part.

---

## 3. Why a gate is the wrong instrument here, stated as a rule this toolkit already holds

**A gate that is always red teaches people to ignore it.** This corpus has made that call four times
and written it down each time — `spec coverage` exits 0 because most extensions legitimately need no
guide; `sdksync` warns on 47 unpinned blocks rather than reddening; `inventory` and `declare` hold their
backlog in a ratchet; `pins` is a reader because the backlog is large. **Every one of those is the same
judgement: hold the debt, gate the delta.**

A cell gate at 37-of-146 is a first run of 109 reds. It would be skipped within a week, and then the
cell table — which is a genuinely good instrument — would be a thing people route around.

### 3a. But the opposite is also ruled out, and by the §1 measurement

*Do nothing* is not available either. The measured state is that **five defect families, four of them
cohort-wide, moved the executed number by 3 in 35,788.** A fold landing over an undriven cell today
leaves no trace anywhere: the cell is uncovered before and uncovered after, the suite is green before and
green after, and **nothing in any artifact records that a normative change was made in a region no check
can see.** That is `unobserved-must` at the fold boundary, and it is the shape `L17` was ratified on —
a normative MUST landing with no conformance check and no declared site.

⇒ **The question is not gate-or-not. It is: what does a fold owe when it lands over a cell nothing
drives?**

---

## 4. The ruling — disclosure, not permission

> ### `GUIDE-CONFORMANCE` §5.3a — a fold discloses the cells it touches `[MUST]`
>
> **A proposal folding a normative change into the core protocol MUST carry a CELL DISCLOSURE: the
> cells of the scope table its deltas touch, and for each one whether a check has been DRIVEN against
> it — `driven` · `named-vector-not-driven` · `no vector`.**
>
> **An undriven cell does NOT block the fold.** A fold may land over any number of them, and most will.
> **An undisclosed cell does block it**: a fold whose disclosure is absent, or which names fewer cells
> than its deltas touch, is not ready to land.
>
> **The drive state is read from the anchor's table, never asserted from the spec side.** Arch does not
> run code and cannot observe coverage; the census is `entity-core-keystone`'s artifact and its
> `--summary` is the source. **A disclosure that cites no run is a guess with a table's formatting.**
>
> **A cell disclosing `no vector` is a REQUIREMENT ON THE CHECK SET, created by the fold and owned by
> the seats that build it.** It is recorded as an open item at the moment of the fold, against the
> check-set author, and it is not discharged by the fold landing.

**Why disclosure is the achievable form, in one sentence: it converts an invisible omission into a
countable one, which is the only move available to a party that cannot run the checks.**

### 4a. What this changes about the four folds already landed

Nothing retroactively — **and that is a decision, not an oversight.** `0.8.2.23`, `.24`, `.25` and `.26`
landed without disclosures. Reconstructing them now would produce four tables nobody measured, which is
the `F73(a)` defect (*"the coverage column is a source read and is the weakest part of the census"*)
committed deliberately. **The rule binds forward.** Whether the backlog is reconstructed is a separate
question and belongs to whoever runs the next census.

---

## 5. The enforcement point, because a rule without one is theater

**`§3`'s own standard applied to this rule: name the file, the grep, or the gate.**

| half | enforcement |
|---|---|
| **the disclosure EXISTS and has the required shape** | ⭐ **mechanical, and arch-side** — a `spec` analyzer over `docs/proposals/`: a proposal whose `Status:` line names a core revision carries a `## Cell disclosure` section with one row per cell and a closed-vocabulary state column. This is the same shape as `inventory` and `declare`, and it ratchets |
| **the disclosure is TRUE** | ⛔ **not mechanical from here, and saying so is the point.** Arch cannot verify a drive state; the anchor can, and their `scope-cell-table.py --summary` is the artifact that does. **The disclosure cites the run it was read from — repo, commit, date — and is verifiable by the seat that owns the table, not by us** |

⚠ **The second row is the honest limit of this ruling and it is stated rather than papered over.** A
seat could disclose a cell as `driven` that is not, and nothing in arch's tree would catch it. That is
the same exposure every build-state claim in this corpus carries, and the same remedy applies: the claim
is pinned to `(repo, commit, date)` and expires (`spec expiry`).

---

## 6. What this ruling deliberately does NOT do

- **It does not make the cell table normative in a specification.** The table is a conformance
  instrument, it lives in the anchor's tree, and a specification that cited it would be pinning a build
  artifact inside a durable document — the defect this repo has recorded four times.
- **It does not slow a fold down.** A disclosure is a section in a proposal that already exists. The
  cost is reading the census once per fold, which is the act the ruling exists to make happen.
- **It does not resolve `F73`'s underlying complaint**, which is that the executed check set is
  insensitive to cohort-wide defects. **Disclosure makes that insensitivity countable; it does not
  reduce it.** What reduces it is `KS-9d`'s tranche A — 6 families, 28 cells, no new fixture — which is
  open and is `entity-core-go`'s.
- **It does not decide WHEN it binds.** See §7.

---

## 7. When it binds — RESOLVED: now

**It binds now**, forward from `ENTITY-CORE-PROTOCOL` `0.8.2.32`. Every core revision folded after
that one carries a disclosure; the revisions at or below it are outside the rule and are not
reconstructed (§4a).

The alternative was to start it after the release, leaving the release path untouched. It was not
taken, and the reason is the one that decided it: **the release is the moment with the most folds
and therefore the most cells being crossed silently**, the cost is one census read per fold, and a
rule adopted after the release it was most relevant to is a rule announced rather than demonstrated.

**What binding now actually costs, stated so it can be checked against later:** a section in each
remaining fold's proposal, written at the fold, from a census read taken then. It adds no review
round, no approval step and no dependency on another seat's schedule — an undriven cell does not
block anything, which is the whole design.

---

## 8. What is owed back to `entity-core-keystone`

| # | item |
|---|---|
| 1 | **The ask is ANSWERED, at the fourth asking, and the delay is ours.** Their reason for not re-arguing it was courteous and they were owed better |
| 2 | **Their ask was right and arch's condition on it was wrong.** They asked us to adopt the table; we adopted it and attached a condition that no fold could meet, and four folds then landed under it without anyone noticing. The correction is against ourselves |
| 3 | **The `--summary` output is now load-bearing for every core fold.** That makes `scope-cell-table.py` a cross-seat instrument rather than an internal one — ⚠ **its output shape is now something other seats depend on, and a change to it is a change to this rule's input** |
| 4 | ⭐ **`KA-5`'s set-difference lesson applies to this rule directly and is folded into it:** a disclosure built per-delta measures each delta's own cells and asks nothing about whether the deltas covered the fold. **The disclosure is taken over the FOLD, not accumulated over its deltas** — same control, one layer up |

---

## 9. Open items

| # | item | owner |
|---|---|---|
| **CD-1** | ✅ **BUILT — `spec disclose`.** Checks that a core fold's proposal carries the disclosure, that it cites a run, and that every state is inside the closed vocabulary. It scopes a **landed** fold by the revision it landed as and a **pending** one by the fact that it lands past the binding line — the second half is the one a revision-only rule gets silently wrong, and two live core proposals are exactly that shape | arch — done |
| **CD-2** | ✅ **RESOLVED — it binds now**, forward from `0.8.2.32` (§7) | closed |
| **CD-3** | Whether the four landed revisions get reconstructed disclosures | whoever runs the next census; arch's position is **no** (§4a) |
| **CD-4** | `KS-9d` tranche A — the thing that actually reduces the insensitivity | `entity-core-go`, open |
