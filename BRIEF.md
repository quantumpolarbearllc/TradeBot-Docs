# BRIEF — read this first

**Last updated:** 2026-09-04 · **Phase:** M1 and M2 complete; **M3 has run and the answer was no**; a bounded pre-work sequence is settling one implementation error before anything else is decided

This file exists so a Claude Desktop session (or any collaborator) can get
fully current in one read, with no prior context and nothing pasted in.

**If you are Claude Desktop:** you are the reader, not the writer. Claude Code
owns the repository. Anything you produce comes back as a file the owner drops
in — see §8.

---

## 0. If you read nothing else

**M3 ran. The answer was no, and the stop fired.** A filings-event signal set did
not beat random selection from the same eligible universe — measured over
2016-2026, four arms, two cost models, 200 seeds. **And no arm beat a passive
index fund:** the best of them returned +6.38/+8.44% annualised against **IWM's
+11.51%** over the same 2,669 trading days. Measured transaction cost is
**4.91-6.12%/yr** and is the dominant term.

**§3's ranker and M4's execution path were never built.** There is no broker
integration, no idempotency, no reconciliation, no approval gate. Every HTTP call
in the package is a `GET`.

**Two corrections were then recorded, and they change what to propose:**

1. **The gross comparison was not like-for-like.** The backtest books delistings
   at −100%; an index fund does not. Corrected, the universe's gross performance
   is *approximately the index*, so the burden falls entirely on the signal to
   cover its own trading cost. **The bar is ~5-6%/yr at a 60-day hold, falling
   to ~1.4%/yr at an annual one.**
2. **One signal family was implemented on the wrong side** — long the leg its
   source literature shorts. **It is not currently known whether the literature
   was tested or its mirror image.** Settling that is a correctness fix, already
   pre-registered with a stop, and it is the next thing to run.

**If you are asked what to do next: the answer is not a new signal family and not
a new threshold.** A pre-registered stop forbids exactly that,
and best-of-N is a selection dressed as a finding. The sequenced pre-work is:
settle the sign, measure the long-leg alpha it produces, compare it against the
cost frontier, and only then decide whether a question exists.

---

## 1. What is being built, in one paragraph

A daily-rebalance research and execution system for **US small-cap equities**
(40–60 names). It reads **SEC filings primarily** and news secondarily,
derives point-in-time features, and runs a deterministic strategy to produce a
target portfolio. Orders pass a turnover budget, a risk gate, and a human
approval gate before reaching the broker. It is a **drift-capture system** —
signals play out over weeks, not hours. **v1 contains no LLM at all.** Sequence
is backtest → paper → small live → scale.

## 2. Where things stand

| | Status |
|---|---|
| Design | **Complete** through the 80th decision. The ten open questions now carry dispositions — one is answered, one is partly answered, the rest are moot rather than resolved — see §5 |
| Feed measurement | **Complete** — five feeds measured against live APIs |
| Literature review | **Complete** |
| Droplet | Hardened, 100 GB volume, stack running, **daily ingest on a timer** |
| **M1 — ingest → raw store** | **Complete.** Both feeds backfilled, arriving daily, gaps visible |
| **M2 — normalise → features** | **Complete.** Normalisation and the Tier A features both — filings, bars, corporate actions, 13(f) share classes, security identity, the 13D/G CUSIP↔CIK edge, the quarterly full index, insider transactions, earnings CAR and filing-language distance |
| **M3 — backtest** | **Run, and answered no.** See §0. Four arms, two cost models, 200 seeds, 5,628 logged trials across 16 registrations |
| Execution (M4–M6) | **Not started, and not being started.** No broker path exists |
| Backtest harness | **Built and used.** Four arms, every run logged, both trial tables append-only by trigger |
| Old repo | Archived. Not a reference for anything |

The foundations are the parts every later component sits on: a clock that can
be replayed at a past instant, mode resolution that fails safe to paper,
settings resolved once into a frozen object, and a migration runner that
refuses to re-apply or reorder schema changes.

**Two survivorship holes were found and closed, both the same mistake.** In each
case a *current-state* source was standing in for history, and in each case the
symptom was invisible because the output stayed plausible.

The first was **identity**. Symbol-to-CIK came only from a file listing
companies that trade today, so a symbol could be joined to its filings if and
only if it had not died — 65.9% of priced symbols unreachable, and specifically
the delisted ones. Reading the CUSIP-to-CIK edge out of 13D/G filings, which
state it with a date and survive delisting, cut that to 53.0%, and cut
*identified common stock* from 46.2% unreachable to **13.3%**.

The second was the **filing population itself**, and it was worse because
nothing pointed at it. The filings table was built from an API reachable only
for companies with a ticker today — 7,997 of them. The quarterly index lists
every filer that ever existed, and parsing what was already stored took it to
**548,914 filers over 8,888,946 filings**. Neither hole failed a test; both were
found by asking what a table could not contain.

**Both feeds are collected and current.** Filings metadata for every filer in
the window — 548,914 of them once the quarterly index was parsed, against the
7,997 the per-company API could reach — and the price substrate.

**Two symbol counts appear below and they are not the same thing.** Bars were
*requested* for **47,568** symbols: the union of the broker's asset list, the
EDGAR ticker map, and every ticker-shaped symbol named anywhere in the corporate
actions history. That union is deliberately over-broad, because any filter on
today's exchange or tradability is a survivorship filter wearing a disguise —
the failed banks now sit on OTC and would have been excluded. Of those,
**21,513 actually returned bars**, and that is the priced universe every
percentage below is taken against. It read 21,351 until 2026-08-20, when a
normalisation fix recovered 162 symbols that had no bars at all — so figures
measured before then use the smaller denominator. More than half the union never
traded in the window at all, which is the expected outcome of asking too widely
on purpose. A
scheduled job brings everything forward nightly, catches up dates it missed, and
reports when a date was never requested.

**Normalisation turns those payloads into typed rows.** Filings carry two
timestamps, because a filing is usually known to exist before its 8-K item codes
are — using one timestamp for both would claim knowledge we did not have. Bars
keep raw and adjusted apart, since a split rewrites the adjusted series
retroactively. Corporate actions are mapped onto one shape: the security an
action happened to, and what it became.

**A fourth normaliser reads the 13(f) securities list**, the only source found
that says what a security *is* — common stock, warrant, unit, right — which the
universe screen needs and neither vendor supplies. It is published as a
mainframe report rendered to PDF for 120 of 121 quarters, and the columns are
recovered from each document's own header row rather than from fixed offsets.
Against the single quarter published both as PDF and as authoritative text:
**25,333 of 25,333 records identical**, nothing dropped, nothing invented.

**Identity is built, and building it exposed the largest single problem in the
system.** Bars are symbol-keyed and filings are CIK-keyed, and symbols get
reused — `BBBY` covers two different companies in the window. Reconstructing
that mapping from dated evidence produced a number worth stating plainly:
**14,060 of 21,351 priced symbols could not reach a filing at all, 65.9%**, and
not a random two thirds — the delisted ones.

The cause was structural. Corporate actions state symbol↔CUSIP across 2,120
dated days; symbol↔CIK came only from a current-state file with no dates in it,
so a symbol resolved if and only if it still traded today. That is survivorship
bias arriving through the identity join rather than through a timestamp, against
a band whose membership turns over 2.10×.

**The fix worked.** EDGAR's quarterly full index enumerates filers that no longer
exist, and a 13D/G filing states CUSIP and subject CIK together with a date,
surviving delisting. The backfill of 32,729 filings yielded an edge from **98.6%**
of them — above the 95% a 60-filing sample predicted, which is the rare case here
of a sample proving pessimistic.

| | before | after |
|---|---|---|
| priced symbols unable to reach a filing | 14,060 (65.9%) | **11,317 (53.0%)** |
| identified common stock unreachable | 2,835 (46.2%) | **829 (13.3%)** |
| symbols carrying a CIK | 10,387 | **13,638** |

The common-stock row is the one that matters, because the raw count includes
warrants, units and funds the screen never wanted. And the 10,387 was *exactly*
the size of the current-state ticker file, so every symbol gained beyond it is
one that no longer trades: the feed genuinely reached backwards, which was the
whole argument for enumerating from the index rather than from submissions.

**Those numbers are MEASURED. What they count rests on something ASSUMED, and a
reader of this file alone would not otherwise learn it.** Every figure in the
table above is a real count. But each one assumes the edge named the *right*
company, and that has not been audited.

The extraction has two guards and they cover two of the three failure modes. A
malformed CUSIP is caught by its check digit — that is how `NONEITEM3`, a
fragment of prose the parser stitched together, was kept out. A CUSIP naming two
different companies is withheld rather than resolved to either, and reported:
456 of 13,778, mostly genuine successor entities like a company renaming itself.

**The third mode trips neither guard.** A *real* CUSIP bound to the *wrong*
company yields exactly one company per CUSIP, so the contradiction check never
fires, and the check digit is no defence because the identifier is genuine. It
would look identical to a correct edge from every angle available downstream —
nothing can distinguish a filled company id from an observed one.

It is labelled **ASSUMED** clean, and the reason it is not simply tested is
itself the problem: corroborating an edge against the company's name is only
possible where that company still appears in the current ticker file, which
excludes precisely the delisted names the edge was built to reach. An audit
would bound the error on the population that never needed fixing and say nothing
about the one that did.

If it is wrong, some fraction of 3,689 joins attach one company's filings to
another company's price series — silently, with the output staying plausible.
That is the same shape as the failure this whole rebuild is a response to, which
is why it is stated here rather than left in the decision log.

**It narrows the problem rather than closing it.** 3,401 priced symbols carry no
identity evidence of any kind, and this feed reached none of them and
structurally cannot — its edge carries no symbol, so it only fills in a link that
already half exists. Universe selection, not the identity join, is now the thing
that blocks the first backtest.

**Two questions that were expected to block the backtest were tested, and one of
them dissolved.** Market caps are shares times price, and share counts for dead
companies were believed to be unrecoverable — which would have rebuilt
survivorship bias inside the cap screen, one stage after it had just been
removed from the identity join. Measured over 300 symbol-interval windows
against SEC filings data: names still trading reach usable coverage 58% of the
time, names whose interval ended 45–62%, **with no trend against age**. The gap
is real and roughly uniform. That is precisely what makes it safe to exclude
those names — the cost is universe size rather than correctness — so the
question stopped being a blocker.

**The other one got a number, and it is a large one.** The screen leaves far
more eligible names than the strategy can hold, so something must rank and
select, and that rule is where overfitting lives. Measured: **185 qualifying
events a trading day against 0.67–1.0 open positions — roughly 220 candidates
per slot.** The rule admits 0.45% of what qualifies, every day, which makes it
a bigger determinant of returns than the signals feeding it. It holds under
every slice; the scarcest signal alone, activist stakes, still runs 37 to 1.

An earlier idea — that watching everything and holding whatever carries a live
signal would reduce this to an occasional tiebreak — **does not survive the
measurement**, and is recorded as retracted. That tiebreak fires daily and
rejects 99.5% of candidates; it *is* the ranking rule.

There is also nothing in v1 to rank *by*. Every signal is an event rather than a
score — an item code is present or absent, a form type is or is not activist, an
insider filing is a classification — so any ranking must be invented from
something other than the signal, and scoring with a model is forbidden. The
answer is a pre-registered, parameter-free control: take the day's qualifying
events in descending liquidity until the turnover budget is spent. It is not
neutral, tilting toward the liquid end of the band, and that is stated rather
than hidden. Every richer rule must beat it on logged trials.

**No money is at risk**, and that is now a structural claim rather than an
assurance. There is no strategy and no broker integration; beyond that, nothing
in the package reads the trading mode, the live broker host does not appear in
it anywhere, and every HTTP call it makes is a `GET`. An order needs a POST, so
no code path can place one in any mode.

## 3. The eight things that decide everything else

**1. EDGAR covers 100% of the universe; news covers 69%.**
Measured across 103 random small/micro-cap companies over three months: 32 had
zero body-bearing news, and **all 32 still filed**. Small caps file *more* 8-Ks
than large caps. Exhibit 99.1 on an 8-K carries the full press release — 35,102
characters for a $46M company. This is why the system is filings-primary.

**2. v1 needs no model, because every evidence-backed signal is structured.**
Filing-language change is a text *diff*. Insider classification is Form 4 XML
plus history. Activist stakes are a form type. Earnings events are an 8-K item
code. Dropping the model removes an entire subsystem, the API dependency, and
the contamination problem below, while keeping every signal with published
evidence behind it.

**3. If a model is ever added, it may extract but never forecast.**
An LLM's weights contain the future relative to any historical backtest, and
that bias is invisible to every point-in-time control. Extraction of stated
facts is verifiable against the source document; a sentiment score is not.

**4. Transaction costs decide viability.**
Costs run 2–4%/year for the smallest stocks. Measured round-trip cost in the
target band is ~95 bps, converging on the literature. The resolution is
holding period: **decide daily, trade rarely.** Target signals (post-earnings
drift ~60 days, filing-language drift) reward patience.

**5. The price feed has no delisting return — it just stops.**
A failed bank's series ends at $106 after a −60% day; equity actually went to
zero. One name's series ends three months before its bankruptcy filing.
Terminal events are therefore classified explicitly, and **anything
unclassified defaults to −100%** so an unknown outcome can never flatter a
backtest.

**6. Split adjustment is a retroactive revision, so both price series are
required.** One sampled name traded at $1.27 while today's adjusted series
shows $2,540 for that same day. Adjusted prices for returns; raw prices for
sizing and cost. A price floor applied to adjusted data would pass a penny
stock.

**7. Pre-registration, not automation, is the research discipline.**
Unattended *search* destroys the ability to believe any result, because every
trial raises the significance threshold. A hypothesis committed before seeing
the data may run unattended; selecting a winner after seeing results may not.

**8. Twice as many names passed through the cap band as sit in it today.**
Measured: 50 names in band now, 105 in band at some point — 2.10×. A universe
built from today's membership would test roughly **48% of the names that
actually existed**, and specifically the survivors. So the historical filer
population is reconstructed from EDGAR full-index files rather than inherited
from today's asset list. The backtest window is **2016-01-04 → present**, set
by where daily bars begin — earlier filings cannot be priced against.

## 4. How work continues between sessions

**The machine accumulates evidence continuously. Humans interpret it in
sessions.**

Runs unattended: ingest, normalisation, feature computation, pre-registered
tests, paper trading, health reporting, and every halt/suspend/demote action.

Never unattended: parameter search, strategy generation, or anything that
selects a winner across variants.

Permanently human-gated: promotion past the compiled ceiling, widening any
risk limit, and deploying new or changed strategy logic.

The system makes progress while unattended. It never *decides* while
unattended.

**Growth is budgeted, not discovered.** Every quantity that can grow — table
sizes, rebuild durations, disk use — carries a declared limit with the
measurement that set it. Crossing one is reported as an expired assumption
rather than a failure, so the response is to re-measure while it is still cheap.

**And nothing is decided on an estimate.** Any number that will inform a
decision is checked against the real thing before it is used — the API, the
database, the actual response — and where that is impossible it is labelled
ASSUMED with what would settle it. This is invariant 21, and it exists because
six confident figures during design were each wrong when finally measured, by
factors up to 48×. It applies to anyone working on this, including a Claude
Desktop session reading this file: a plausible number is not a finding.

## 5. Milestones

| | Milestone | Gate |
|---|---|---|
| M1 | Ingest → raw store, running continuously | Data arriving daily, gaps visible |
| M2 | Normalise → features | Features reproducible from raw |
| **M3** | **Backtest a strategy** ← *first real decision point* | Survives cost and bias controls |
| M4 | Paper execution | Live tracks paper |
| M5–M6 | Capped live, then live | — |

**M1 and M2 are complete; M3 has run and answered no (§0); M4–M6 were never
started.**

**Five of the ten open questions gated M3**, and they are one question wearing
different clothes: *which securities exist, which are eligible, and which can be
joined to a filing.* **They now carry dispositions, and the distinction matters:**
question 6 is **answered**; questions 4, 5, 8 and 10 are **moot rather than
resolved** — the thing they blocked is not being built, the defects are
unchanged, and none of them fails loudly. Anything re-pointing this instrument at
a new question inherits all four silently.

| # | Question | Why it blocks |
|---|---|---|
| 4 | Identity intervals begin at first *observation*, not first trade | A name trading from 2016 whose first dated evidence is 2020 has no identity for four tradeable years. Unmeasured |
| 5 | 3,401 priced symbols carry no identity evidence at all | Nothing to join *from*. The 13D/G edge reached none of them and structurally cannot — it fills a link that already half exists |
| **6** | **Universe selection beyond the screen — ANSWERED** | **Was the largest, at ~220 candidates per open slot. The answer is that selecting on the signal set is worse than not selecting at all, and ranking the universe by liquidity beats both (§0)** |
| 8 | What counts as common stock is undecided | Over twelve thousand distinct 13(f) class labels, because they embed rates and expiry dates. This decides which securities are *eligible at all*, and closed-end funds currently carry the same label as operating companies |
| 10 | The 13D/G subject company is unaudited | Described above. A wrong-but-well-formed edge trips no guard |

Questions 1–3 need real fills and cannot be settled before paper trading —
**except that question 3's cost half is now partly answered without them**: the
arms rank identically under both cost models, so the result is not
fill-dependent. Question 9 gates M4 and nothing else, and carries a standing
condition that outlives M3: the resolved trading mode must be persisted with
every order **before** the first `POST`, never after.

**The first signal family is being built now.** Activist stakes: every reporting
person's percent of class, read off the 13D/G cover page. Two things about it
are worth knowing because they generalise.

The documents were already held — but only *one per company*, which was exactly
right for the job they were fetched for (establishing which company a filing is
about) and useless for this one, which needs every stake as a dated event. 3.9%
of the relevant events were held. The re-fetch **completed** — 280,479 documents,
back to the start of the window.

And the extractor was sampled across five eras before a line of it was written.
That found the modern format has **two** schemas rather than one, where a parser
knowing only the first returns nothing for the commoner filing type *and still
looks healthy*, because the rarer type keeps parsing. Yield went 83% to 100% on
that plus one HTML-entity fix. Both discoveries came from sampling oldest,
newest and three between rather than checking one document — which is a written
rule here precisely because it has paid off repeatedly.

**The first backtest can now be run, and it has a stop attached.** Three
variants of the same strategy, identical but for which names they pick: one
selecting at random from the day's signals, one applying the candidate ranking
rule, and one selecting at random from the whole eligible universe while
ignoring signals entirely.

That third arm is the one that matters. Comparing ranking rules to each other
cannot distinguish "the ranking is useless" from "the signals carry nothing" —
and if selecting at random from signals is no better than selecting at random
from anything, there is nothing here to rank and no rule can rescue it.

**The stop is binding, and was agreed before any of it ran.** If that is the
result, the milestone ends rather than becoming "try a different signal set".
Recorded in advance because that is the only moment the answer is honest.

**Every run is logged whether it succeeds or not**, in an append-only table the
database itself refuses to let anyone edit. A finished run cannot be rewritten
and a hypothesis cannot be amended after seeing results. A strategy that beats
the market on the fortieth variant tried and on the first are different claims,
and only the count distinguishes them — so the count is not something anyone has
to remember to keep.

**Backtest before paper trading** is deliberate: paper requires building the
entire execution path, which is a large and dangerous surface to build before
knowing whether the signal works.

**One prerequisite is already known for M4.** The resolved trading mode is not
recorded anywhere — settings are resolved once into a frozen object, so a
process has one answer for its lifetime and leaves none behind it. That is
harmless while nothing can trade, and M4 is the point at which it stops being
harmless, so the mode gets persisted with every order before that milestone
rather than after.

M1 backfills history rather than only collecting forward, phased so that
metadata — which unlocks three of the four signals — lands first, and running
newest-first so backfill depth is a consequence of available disk rather than a
decision defended up front.

## 6. Decisions made — do not re-litigate

Forty-nine decisions are recorded with full reasoning in `DECISIONS.md`
(private repo). Headlines: start fresh · small caps · filings-primary · daily
rebalance · 40–60 names · **no model in v1** · backtest-first · turnover as a
budgeted constraint · buffer zones on universe thresholds · terminal events
default to −100% · both price series retained · pre-registration · one live
strategy by policy, many by schema · demotion automatic, promotion human ·
raw store in Postgres with blobs behind an interface · backfill immediately,
phased and newest-first · historical universe reconstructed from EDGAR
full-index · window fixed at 2016-01-04 by bar history · raw payloads stored for
every feed with no exception · price the whole symbol union and filter later ·
the daily job reads an index rather than sweeping · derived tables are
disposable and rebuilt, raw never is.

**Several conclusions reversed earlier ones during design.** They are recorded
as reversals because the reasoning matters — including that small caps are not
news-starved relative to large caps (only relative to mega-caps), that the
second news vendor does not dominate the first (it has no usable history), and
that forward data collection does *not* block backtesting at daily cadence.

**Two more reversals came from building rather than designing.** The 13(f)
PDFs were accepted on measurements of 54.5% recall and 98.3% agreement;
rebuilding the parser positionally reached 100% on both, so the earlier figures
described a technique rather than the document. And a check-digit pass rate
reported as "95–100%" turned out to be measuring a mixed population — real
securities pass at 100.0000%, while the SEC's constructed option identifiers
fail by design and are half the rows.

## 7. Out of scope for v1

Model extraction · fundamentals (restated → invisible lookahead) · YouTube
(value unmeasured) · social (manipulation surface) · second news vendor (no
backtest history) · intraday anything · multi-strategy capital allocation
(schema supports it, policy defers it) · options, macro, transcripts.

## 8. How this repo works

**This repo is public and contains documentation only.** The code lives in a
separate private repo. Strategy specifics, thresholds, and credentials never
appear here.

| File | Contents |
|---|---|
| **`BRIEF.md`** | This file — current state, read first |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Pipeline, components, the 22 invariants |

Three further documents live in the **private code repo**: `FEEDS.md` (measured
API behaviour), `DECISIONS.md` (dated log with full reasoning), and
`RESEARCH.md` (literature with design implications). Their conclusions are
summarised above, so you do not need them to be current — ask for a paste if
you want the underlying detail.

**Reading order for a cold start:** this file, then `ARCHITECTURE.md`. That is
the whole public set and it is enough to be current.

**On trusting these documents.** Everything marked MEASURED was verified
against a live API on a stated date. Everything marked ASSUMED was not. That
distinction is load-bearing: the previous build produced an analysis resting on
figures that turned out never to have existed. If a number here has no marker
and no source, treat it as unverified and say so.
