# BRIEF — read this first

**Last updated:** 2026-08-13 · **Phase:** design complete, no code written yet

This file exists so a Claude Desktop session (or any collaborator) can get
fully current in one read, with no prior context and nothing pasted in.

**If you are Claude Desktop:** you are the reader, not the writer. Claude Code
owns the repository. Anything you produce comes back as a file the owner drops
in — see §7.

---

## 1. What is being built, in one paragraph

A daily-rebalance research and execution system for **US small-cap equities**
(~$300M–$2B, 40–60 names). It reads **SEC filings primarily** and news
secondarily, uses an LLM to extract *stated facts* from those documents,
converts them to point-in-time features, and runs a deterministic strategy to
produce a target portfolio. Orders pass a risk gate and a human approval gate
before reaching Alpaca. It is a **drift-capture system** — signals play out over
weeks, not hours. Sequence is research → paper → small live → scale.

## 2. Where things stand

| | Status |
|---|---|
| Design | **Complete, awaiting review** |
| Feed measurement | **Complete** — four feeds measured empirically |
| Literature review | **Complete** |
| Code | **None written.** New private repo not yet created |
| Old `TradeBot` repo | **Abandoned.** Not a reference for anything |

**Nothing is running. No data is being collected. No money is at risk.**

## 3. The five things that decide everything else

**1. EDGAR covers 100% of the universe; news covers 69%.**
Measured across 103 random small/micro-cap companies over three months: 32 had
zero body-bearing news, and **all 32 still filed**. Small caps file *more* 8-Ks
than large caps. Exhibit 99.1 on an 8-K carries the full press release — 35,102
characters for a $46M company. This is why the system is filings-primary.

**2. The model may never forecast.**
An LLM's weights contain the future relative to any historical backtest. Llama 2
prompted about Sept–Nov 2019 earnings calls mentions Covid-19 in >25% of cases.
This bias is invisible to every point-in-time control. So the model answers
*"what does this document say?"* and never *"is this good news?"* — extraction is
verifiable against the source; a sentiment score is not.

**3. Transaction costs decide viability.**
Costs run 2–4%/year for the smallest stocks — enough to erase the size premium.
At ~100 bps round-trip, monthly turnover loses by default. The resolution is
holding period: **decide daily, trade rarely.** Target signals (PEAD ~60 days,
filing-language drift) reward patience.

**4. Daily cadence dissolves the previous build's worst bug.**
EDGAR timestamps are date-only. v1 added an assumed delay to that midnight floor
and produced timestamps hours *before* filings existed — 318 contaminated
signals. At daily cadence a filing dated D is not eligible until D+1. The
machinery is not fixed; it is unnecessary.

**5. Small caps delist, and backtests must not hide it.**
Excluding delisted names can quadruple apparent returns. The universe is built
as-of each decision date with delisted names retained. **The terminal-return
convention is still an open question.**

## 4. Decisions already made — do not re-litigate

Full reasoning is recorded in `DECISIONS.md` in the private code repo. The
decisions themselves are listed here so this file stands alone.

- Start fresh; the previous repo is abandoned (D-001)
- Small caps, ~$300M–$2B (D-002)
- Filings-primary (D-003)
- Daily rebalance, pre-open, prior-day closes (D-004)
- 40–60 names (D-005)
- Model extracts facts only (D-006)
- Turnover is a budgeted constraint (D-007)
- Feeds: EDGAR + Alpaca bars + Alpaca news in v1; Finnhub at paper stage; YouTube deferred; fundamentals and social out (D-008)
- Docs public, code private (D-009)
- Claude Code writes, Claude Desktop reads (D-010)

**Three of today's conclusions reversed earlier ones in the same session.** They
are recorded as reversals in `DECISIONS.md`, because the reasoning matters:
small caps are *not* news-starved relative to large caps (only relative to
mega-caps); Finnhub does *not* dominate Alpaca news (it has no usable history);
and screening the universe by information availability is unnecessary.

## 5. What is explicitly out of scope

Named so it cannot creep back. The previous attempt declared six source kinds
and implemented two.

Fundamentals (restated → invisible lookahead) · YouTube (value unmeasured) ·
Social (manipulation surface) · Finnhub in v1 (no backtest history) · intraday
anything · options, macro, transcripts.

## 6. What happens next

1. Owner reviews `ARCHITECTURE.md`
2. Create the new **private** code repo
3. Write `CLAUDE.md` with the 18 invariants
4. Build ingest → raw store first; **start collecting forward data immediately**, since that clock is the long pole and cannot be bought later
5. Then normalise → extract → feature store → backtest

**The open questions in [ARCHITECTURE.md](ARCHITECTURE.md) §9 should be closed
before code, not during it.**

## 7. How this repo works

**This repo is public and contains documentation only.** The code lives in a
separate private repo. Strategy specifics, thresholds, and credentials never
appear here.

| File | Contents |
|---|---|
| **`BRIEF.md`** | This file — current state, read first |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Pipeline, components, the 18 invariants |

Three further documents live in the **private code repo**, not here. Their
conclusions are summarised above, so you do not need them to be current — ask
for a paste if you want the underlying detail:

| Private doc | Contents |
|---|---|
| `FEEDS.md` | Field notes — measured API behaviour and coverage data |
| `DECISIONS.md` | Dated decision log with full reasoning and reversals |
| `RESEARCH.md` | Literature review, each finding tied to a design implication |

**Reading order for a cold start:** this file, then `ARCHITECTURE.md`. That is
the whole public set and it is enough to be current.

**On trusting these documents.** Everything marked MEASURED was verified against
a live API on a stated date. Everything marked ASSUMED was not. That
distinction is load-bearing: the previous build produced an analysis resting on
figures that turned out never to have existed. If a number here has no marker
and no source, treat it as unverified and say so.
