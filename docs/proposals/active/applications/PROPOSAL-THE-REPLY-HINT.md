# PROPOSAL — the reply hint: how an author learns a reply exists without opening an inbox

**Status:** DRAFT — 2026-09-06. First pass. Nothing landed.
**⚠ LIKELY SUPERSEDED, same day, by `EXPLORATION-THE-TWO-LEVEL-INDEX-…` §5.** That document answers
§5.3's own strategic question — *does route 4 make this redundant?* — and the answer looks like **yes**:
a gatherer lets an author learn of a stranger's reply **by pulling from a source they chose**, so there
is no inbox, no open delivery grant and **no spam surface at all**, which is strictly better than
bounding one. **Do not develop this further before reading that §5.** What plausibly survives is the
**latency** case and the **no-gatherer-available** case, which is a much smaller claim than this
document is written with.
**Tier:** applications — `APP-CONVENTION-FEED` (unlanded) + `EXTENSION-INBOX` §1.1 grant scope.
**Closes:** the prerequisite `EXPLORATION-THE-READERS-LOOP` §3.3 names and does not solve.

---

## §1 The problem, and it is the only unsolved step in the reader's loop

`EXPLORATION-THE-READERS-LOOP` §2.5 establishes that the reverse-edge problem has **four** answers and
no fifth: the parent's server forwards (ActivityPub), an aggregator indexes everything (ATProto), relays
hold a reverse index (Nostr), or you replicate your neighbourhood (SSB). Its §3 maps ours:

| Route | Coverage | State |
|---|---|---|
| **1 — replies from people you follow** | your follow graph | **free, needs nothing, is the v1 answer** |
| **2 — mirrors** | + whoever the mirror read | specified (`FEED` §4) |
| **3 — the author publishes their own reply index** | the author's comment section | **blocked — see below** |
| **4 — aggregator backlink index** | broad | sketched, `EXTENSION-QUERY` §2.2 is the primitive |

**Route 3 is blocked on one thing, and it is not the index — it is that the author must learn the reply
exists.** `EXTENSION-INBOX` §1.1 delivers against a `deliver_token` **the recipient minted**. Measured:
there is **no unsolicited-delivery mode anywhere in the extension** — no open grant, no public inbox.
That is deliberate and it is the property the corpus claims as its anti-spam posture: *spam needs a
route, and every route is opt-in.*

**So the two halves are both true and in tension.** No open route means no spam; it also means a
stranger's reply cannot reach you, and *"someone replied to your post"* is not expressible.

## §2 The mechanism the web already shipped for exactly this

**Webmention (W3C Recommendation) is the same problem with the same constraint**, and its answer turns
on one property we do not currently use.

**The notification carries no content.** §3.1.3: the sender *"MUST post x-www-form-urlencoded `source`
and `target` parameters."* **Two URLs. That is the entire message.**

**The receiver does not trust it — it goes and looks.** §3.2.2: the receiver *"MUST perform an HTTP GET
request on source … to confirm that it actually mentions the target."*

> **The whole design is: the authority lives at the source, and the message is only a pointer to go
> look.** A forged notification fails verification and is dropped. Nothing the sender says is believed.

**Everything else is the receiver's business.** §4.1: *"Receivers MAY moderate Webmentions before
publishing them."* §3.2.1: *"What a 'valid resource' means is up to the receiver."* And the abuse
budget is stated as bounds rather than as a gate — §3.2 queue asynchronously *"to prevent DoS
attacks"*, §4.2 *"place limits on the amount of data and time they spend fetching unverified source
URLs."*

## §3 What is proposed

**A hint is two hashes: `(replier_peer_id, parent_entry_hash)`.** It asserts nothing and carries no
content. On receipt the author fetches the replier's tree, locates the entry, and verifies:

1. the entry's detached signature verifies against the replier's key (rung 1);
2. the entry's `reply.parent` **is** the named parent;
3. the parent is in fact the author's own entry.

**Any check failing, the hint is dropped and nothing is recorded.** Verification is the same code path
the reader's loop already runs on every entry it ingests — there is no new verification surface.

**This is not a new mechanism.** It is an **open delivery grant scoped to one narrow operation** whose
payload is two hashes. The corpus's *every route is opt-in* property is preserved exactly: an author who
wants a comment section publishes the grant; an author who does not, does not, and is unreachable as
today. **The design work is the grant's scope and its bounds, not the plumbing.**

### §3.1 Why our version is strictly better than the web's, and it is not a small difference

**Webmention's source is a URL.** It can change after verification, or vanish — which is why §4.1 has to
make re-verification a `MAY` and why the vouch extension exists at all.

**Ours is a signed, content-addressed entry in the sender's own namespace.** Therefore:

- **A spam hint requires the spammer to actually publish a signed reply under their own peer-id.** The
  abusive act is *permanent, attributable and non-repudiable* — the opposite of a throwaway URL.
- **Rate-limiting has a stable subject.** The web cannot rate-limit a sender identity because it does
  not have one; we have a peer-id on every hint.
- **Verification is offline-ish and cheap** — content-addressed, dedups on ingest, and a hint for an
  entry already held costs nothing.

**The residual cost, stated plainly:** an open hint grant lets any peer cause the author one bounded
fetch. That is the price of a comment section and it should be written down rather than discovered.
Webmention's §4.2 bounds are the right shape and transfer directly.

## §4 What this does for the forum, and for blocking

**Route 3 unblocks**, which is the *"comments under my post"* product. And the operator's blocking
question answers itself from rules already ruled:

> *"If I blocked the guy who replied, do others see it?"*

**An author is under no obligation to act on a hint — ignoring it IS the block**, and it costs one
dropped message. Downstream, `FEED` §4.2's ruled position governs: a mirror **omits but never
substitutes**, so Alice's published thread simply does not contain Bob's reply. **Carol reading Alice
does not see it. Carol reading Bob, or any mirror that carried him, does.**

**That asymmetry is correct and is the design working**, not a leak: content-addressed data cannot be
un-published, so *nobody can make it disappear globally and nobody is forced to carry it.* The failure
mode of the alternative — an instance deleting content for everyone — is the one this project exists to
avoid.

## §5 Open questions

1. **Is the hint a grant at all?** A grant preserves *every route is opt-in* and is the conservative
   choice, which is why it is proposed. The alternative — a tiny always-on surface — breaks that
   property and should have to argue for itself.
2. **INBOX or FEED?** The mechanism is INBOX's; the vocabulary is FEED's. Leaning FEED with an INBOX
   grant-scope note, since the reply semantics are application-tier.
3. **Does route 4 make this unnecessary?** **Possibly, and this is the real strategic question.** An
   aggregator serving backlink queries gives the author their comment section by *polling* rather than
   by being told, with no open grant and no spam surface at all. **If route 4 lands first, this proposal
   may be redundant** — and the honest ordering is that route 4 is the better answer where an aggregator
   exists, and the hint is the answer where one does not.
4. **Bounds** — rate per peer-id, max hints per parent, retention of unverified hints. Unspecified;
   Webmention §4.2's *limit data and time on unverified sources* is the model.
5. **Is `system/attestation` the carrier?** It is signed, content-addressed and already routable; a hint
   may need nothing minted at all.

## §6 What this does NOT claim

- **Not a fifth answer to the reverse-edge problem.** §2.5's design space stays closed; this is
  plumbing for route 3, which is answer (a) in that table without the parent's *server*.
- **Not a notification system.** It is one bounded signal for one relationship — a reply naming your
  entry. Likes, follows, mentions and quotes are each their own question and none is assumed here.
- **Not measured.** §8.4 of the reader's-loop document is still true: *no open delivery grant has been
  deployed and observed being abused*, so the spam argument on both sides is structural.
