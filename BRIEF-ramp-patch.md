> **Applied 2026-11-21, with corrections. Do not re-apply.** Edits 1, 2, 5 and 6
> went into `BRIEF.md` in corrected form, and edit 3 became a short "open before
> the first paper order" list. **Edit 4's conflict does not exist.** Nothing in
> the live path tops up a held position, so the ramp is the existing rule,
> applied by raising the book size for new opens. The proposed invariant 23 is
> left for the owner. Of the outstanding items: 1, 2, 3 and 4 are fixed as dated
> corrections in `ARCHITECTURE.md`, and 5 stays open. The patch is kept as the
> record of what was proposed.

# Patch instructions for BRIEF.md — record the ramp decision

**For:** Claude Code · **Target:** `BRIEF.md` (public docs repo) · **Date:** 2026-11-21

**How to apply.** These are insertions and small replacements, described by
anchor rather than by line number. The drafting session did not hold BRIEF.md
verbatim, so **do not treat this as a replacement file** — apply each edit
against the real text and preserve surrounding content. Keep the existing
`Last updated: 2026-11-21` line as it is; this is a same-day amendment.

Decision numbers below are written as `D-NNN`. Assign the next real numbers from
DECISIONS.md rather than guessing.

---

## Edit 1 — §0 "If you read nothing else"

Add as a new bullet, after the bullet describing the $12,000 paper book and
before the bullet on execution:

> **The $12,000 book is a consequence of the sizing rule, not a requirement.**
> It is equal weight × 120 positions × a $100 minimum unit. The adopted
> direction is to ramp position *size* rather than position *count* — holding
> the full 120-name book from the first session at whatever size capital
> supports, so the specification runs intact and capital becomes a scaling
> factor. The ramp is specified in `RAMP.md` in the private repository. It is
> not yet executable: it conflicts with the live-compatible no-re-buy rule, and
> that conflict is unresolved. See §5.

---

## Edit 2 — §2 "Where things stand", paper-stage row

Replace the row reading approximately *"Paper stage: planned with a $12,000
book"* with:

> **Paper stage.** Specification is 120 positions. A $12,000 book is the
> smallest that runs it at a $100 minimum unit. A size-ramp is the adopted
> direction and would lower that floor without changing the specification, but
> three gates stand in front of it: broker fractionability is unmeasured, the
> live-rule conflict in §5 is unresolved, and the selection rule is specified
> contradictorily. Paper has not been scheduled against any book size.

---

## Edit 3 — §5 Milestones / open-questions table

Add three rows, or an equivalent block if the table shape differs:

| Item | Status | Note |
|---|---|---|
| Broker fractionability | **ASSUMED, unmeasured** | Universe ∩ fractionable list, minimum notional, exit symmetry. Gates the size-ramp entirely. `RAMP.md` §2 |
| Ramp cost measurement | Not started | Re-run M3 across {size-ramp, count-ramp} × two contribution rates; report the difference in basis points. `RAMP.md` §3 |
| Live-rule / ramp conflict | **Open, blocks paper** | See §5 addition below |

---

## Edit 4 — new subsection, place in §5 near the other blocking items

> **The size-ramp conflicts with the live-compatible rule.**
>
> The 2026-11-21 correction established that the live system cannot buy a name
> it already holds; removing those re-buys cost about 37% of the signal arm's
> opens and roughly halved the measured effect. A size-ramp consists almost
> entirely of buying names already held, since every contribution tops up 120
> existing positions toward a new target weight. Read literally, the size-ramp
> is not executable.
>
> Resolving it requires a distinction the documents do not currently draw,
> between opening a new signal-driven position in a held name and scaling an
> existing position toward its target weight as equity changes. Three
> dispositions are set out in `RAMP.md` §4. Whichever is chosen must be recorded
> as a decision and reflected in the order-intent model **before any paper
> session runs**. If the first disposition is chosen, M3 must be re-run, because
> the current result was produced under a rule that forbids both actions.
>
> A paper run under an ambiguous rule validates operations for something the
> live system may not be permitted to do — the same error class as the
> 2026-11-21 correction and the 318 pre-filing timestamps before it.

---

## Edit 5 — §6 "Decisions made, do not re-litigate"

Add two entries, and record both in DECISIONS.md:

> **D-NNN — Ramp position size, not position count.** Capital is deployed by
> growing all 120 positions toward an equity ÷ 120 target, not by adding names
> over time. A count-ramp was rejected: it spends months in a book the project
> cannot measure, roughly doubles variance at 30 positions against 120, and
> re-opens question 6 by driving the funnel from roughly 8:1 to near 67:1
> during the ramp (**derived, not measured**). Adopted as direction only —
> execution is gated on fractionability and on the live-rule conflict above.

> **D-NNN — Returns are not read from a thin book.** Proposed as invariant 23:
> returns are not read, reported or acted on from a book below a stated position
> count, declared before the first live session. This makes an existing
> principle enforceable at the stage where it is hardest to hold. Two further
> invariants are proposed in `RAMP.md` §6, covering order-intent separation and
> a minimum-notional halt.

---

## Edit 6 — §8 "How this repo works"

Amend the sentence listing what is kept in the private repository so that
`RAMP.md` appears alongside the field notes, decision log and literature review.
Suggested replacement:

> Field notes, the decision log, the literature review and `RAMP.md` (the
> capital ramp strategy) are kept in the private code repository rather than
> here. Their conclusions are summarised above.

---

## Not in this patch, but outstanding

These were raised in the same session and are **not** addressed by the edits
above. Each needs its own disposition:

1. **The selection rule is specified contradictorily** in ARCHITECTURE.md — §0
   says the signal set beats random selection from the same universe, §10's
   question 6 says the opposite, the 2026-10-04 note sides with §0, and a third
   statement credits liquidity-ranking. Paper cannot execute an ambiguous rule.
   This is the single highest-priority fix.
2. **M3's gate is never marked passed.** If the intent is to proceed for
   operational reasons, state that explicitly rather than leaving it silent.
3. **Invariant cross-reference error**, ARCHITECTURE.md §5.2 and §10 — both cite
   invariant 14 for what the list numbers 16. Flagged 2026-08-17, still unfixed,
   and upstream of CLAUDE.md.
4. **"Exactly one POST"** in ARCHITECTURE.md §3 is unexplained.
5. **The universe trails IWM by roughly a point before any signal** — diagnosable
   with data already held, and plausibly a larger lever than further signal work.
