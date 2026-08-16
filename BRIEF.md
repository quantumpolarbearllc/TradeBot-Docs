# BRIEF — read this first

**Last updated:** 2026-08-15 · **Phase:** M1 complete, M2 in progress — data collecting daily, no signals yet

This file exists so a Claude Desktop session (or any collaborator) can get
fully current in one read, with no prior context and nothing pasted in.

**If you are Claude Desktop:** you are the reader, not the writer. Claude Code
owns the repository. Anything you produce comes back as a file the owner drops
in — see §8.

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
| Design | **Complete.** All six original open questions resolved |
| Feed measurement | **Complete** — five feeds measured against live APIs |
| Literature review | **Complete** |
| Droplet | Hardened, 100 GB volume, stack running, **daily ingest on a timer** |
| **M1 — ingest → raw store** | **Complete.** Both feeds backfilled, arriving daily, gaps visible |
| **M2 — normalise** | **In progress.** Filings, bars, corporate actions and the 13(f) securities list done; security identity is what is left |
| Features, strategy, execution | Not started |
| Old repo | Archived. Not a reference for anything |

The foundations are the parts every later component sits on: a clock that can
be replayed at a past instant, mode resolution that fails safe to paper,
settings resolved once into a frozen object, and a migration runner that
refuses to re-apply or reorder schema changes.

**Both feeds are collected and current.** Filings metadata for all 7,997
distinct US filers, and the price substrate — 47,568 symbols over the full
window in both adjustments, plus the complete corporate actions history. A
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

**What remains in M2 is identity.** Bars are symbol-keyed and filings are
CIK-keyed, and symbols get reused — `BBBY` covers two different companies in the
window, and the delisted one's ticker list is now empty, so the mapping has to
be reconstructed from dated evidence rather than looked up.

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

Forty-three decisions are recorded with full reasoning in `DECISIONS.md`
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
