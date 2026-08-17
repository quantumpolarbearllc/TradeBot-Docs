# Architecture

**Status:** draft for review · **Last updated:** 2026-08-17

A daily-rebalance research and execution system for **US small-cap equities**,
built filings-first. This document is the wireframe: what the pieces are, why
the boundaries sit where they do, and which rules may never be broken.

Every design choice here traces to a measurement or a research finding. The
supporting evidence lives in `FEEDS.md` (field notes) and `RESEARCH.md`
(literature) in the private code repo; conclusions are restated here so this
document stands alone. Where something is assumed rather than measured, it
says so.

---

## 1. What this system does

Once per day, before the open, it decides what to hold.

It reads SEC filings (primarily) and news (secondarily) for a universe of
40–60 small-cap names, derives point-in-time features, and runs a
deterministic strategy over those features to produce a target portfolio.
Orders pass a turnover budget, a risk gate, and an approval gate before
reaching the broker.

**It is a drift-capture system, not a day-trading system.** The signals it is
built for — post-earnings drift, filing-language change, insider accumulation,
activist stakes — play out over weeks. See §7 for why that is a hard
constraint rather than a preference.

## 2. Scope

| Decision | Value | Why |
|---|---|---|
| Cadence | Daily rebalance, pre-open | Prior-day closes are available on the current data plan; intraday is not |
| Universe | 40–60 US small caps | Below ~40 names signal flow drops under one event/day |
| Cap band | ~$300M–$2B, with buffer | Below $300M coverage thins and costs escalate |
| Primary feed | SEC EDGAR | 100% universe coverage vs 69% for news |
| **Model in v1** | **None** | Every evidence-backed signal is computable without one — see §5.3 |
| Backtest window | **2016-01-04 → present** (~10.6 yrs) | Set by the data: daily bars start there, so earlier filings cannot be priced against |
| First milestone | **Backtest a Tier A strategy** | Execution is only built once a signal has earned it |

## 3. Milestones

Ordered so the expensive and dangerous parts are built last, and only if
earned.

| | Milestone | Gate to pass |
|---|---|---|
| **M1** | Ingest → raw store, running continuously | Data arriving daily, gaps visible |
| **M2** | Normalise → observations, documents, Tier A features | Features reproducible from raw |
| **M3** | **Backtest a Tier A strategy** ← *first real decision point* | Survives cost model and bias controls |
| M4 | Paper execution | Live results track paper expectations |
| M5 | `live_capped` | — |
| M6 | `live` | — |

**M1 runs unattended and needs neither a model nor a human.** M3 is where the
project either justifies continuing or does not.

## 4. Pipeline

```mermaid
flowchart TD
    subgraph ingest["Ingest — immutable, retrieved_at stamped"]
        A1[SEC EDGAR]
        A2[Market data<br/>bars raw + adjusted<br/>corporate actions]
        A3[News<br/>stored, not yet read]
    end

    A1 --> RAW[(Raw store<br/>never mutated)]
    A2 --> RAW
    A3 --> RAW

    RAW --> NORM[Normalise<br/>re-runnable]
    NORM --> OBS[(Observations<br/>numeric, revisable)]
    NORM --> DOC[(Documents<br/>text)]

    OBS --> FEAT[(PIT feature store<br/>as-of decision time)]
    DOC --> FEAT
    UNI[Universe<br/>as-of, buffered] --> FEAT

    DOC -.v2 only.-> EX["Extraction — LLM boundary<br/>states facts, never forecasts"]
    EX -.-> FEAT

    FEAT --> STRAT[Strategy<br/>deterministic, versioned]
    STRAT --> TURN[Turnover budget<br/>edge vs cost]
    TURN --> RISK{Risk gate<br/>compiled ceilings}
    RISK --> APPR{Approval gate<br/>promotion state}
    APPR --> EXEC[Execution<br/>idempotent]
    EXEC --> BROKER[(Broker)]
    BROKER --> RECON[Reconcile] --> OBS

    style EX fill:#3a3a4a,stroke:#7a7a9a,color:#ccc,stroke-dasharray: 5 5
    style RISK fill:#5a3a3a,stroke:#ba7a7a,color:#fff
    style APPR fill:#5a3a3a,stroke:#ba7a7a,color:#fff
    style RAW fill:#3a4a5a,stroke:#7a9aba,color:#fff
    style STRAT fill:#3a5a3a,stroke:#7aba7a,color:#fff
```

The extraction stage is **dashed because it does not exist in v1**. Nothing
downstream of the feature store may call a model, and nothing upstream of it
may place an order.

## 5. Components

### 5.1 Ingest — raw and immutable

Fetches, stamps `retrieved_at`, stores the **raw payload unmodified**. Does
not parse, score, or interpret.

The biggest structural fix over the previous build, where documents were never
persisted — which made it impossible to characterise the corpus after the
fact, or to re-parse when parsing turned out wrong. **Separate fetch from
parse: if parsing is wrong you re-parse; you do not re-fetch.**

News is stored in v1 even though nothing reads it, so the corpus exists when
extraction arrives.

Each feed declares a **history horizon**; a query beyond it fails loudly. This
exists because one vendor returns `HTTP 200` with `[]` for out-of-range dates,
and a backtest reaching too far back would silently run with a feed missing.

Raw payloads land in Postgres rather than in files with a database index —
chosen for queryability, and affordable because gzipped primary documents are
~48× smaller than the full submission packages the API reports its sizes for.
Blob storage sits behind an interface, so moving documents out to files later
is a swap rather than a migration.

**The store is content-addressed**: one row per distinct response body, keyed
by hash, and one row per HTTP request referencing it. The filings metadata
endpoint supports no conditional GET, so the daily sweep re-downloads every
file in full and they are byte-identical unless the company filed. Stored per
response that is 37.3 GB/year; content-addressed it is 2.45 GB/year, and
nothing is lost — every fetch keeps its own `retrieved_at`, which is what the
point-in-time gate reads.

Failed fetches are recorded too, with their status. One vendor answers
throttling with a styled HTML error page under `503`, and status is the only
thing that distinguishes it from a document. Both tables are append-only,
enforced by the database: re-parsing is how a mistake gets fixed, and editing
the record is how a backtest becomes unfalsifiable.

**Backfill is phased by what each phase unlocks, and runs newest-first.**

| Phase | What | Unlocks |
|---|---|---|
| **1** | Submissions metadata, all candidate filers | PEAD, 13D/G, 8-K events — three of the four Tier A signals |
| 2 | Form 4 XML | Insider routine vs opportunistic |
| 3 | 10-K/10-Q documents | Lazy Prices — and ~61% of all storage |

Metadata is cheap and unlocks most of the signal; full filing text is the
expensive part and buys one signal. Phase 1 is queryable within the hour, so
the project becomes testable long before storage is a question.

Newest-first makes the backfill depth a **consequence of available disk rather
than a decision defended up front** — the most recent data is always held, and
reaching further back is resuming the job rather than redoing it.

**The window starts 2016-01-04, because daily bars do.** Filings older than
that cannot be priced against, so fetching them would spend storage and rate
limit on data that can never be tested.

### 5.2 Normalise — re-runnable

Raw payload → typed rows. Pure, deterministic, safe to re-run across the whole
raw store when a parser improves. **Derived tables are disposable**: they carry
no append-only protection and are rebuilt rather than patched, which is the
division invariant 14 actually draws — raw ingest is immutable, everything
downstream of it is not.

Four are built. **Filings** carry two timestamps, because the two EDGAR sources
arrive apart: a daily index names a filing the morning after it is filed, while
item codes and acceptance times arrive whenever a submissions fetch covers it. A
feature reading an 8-K item code gates on the later one, since gating on the
earlier would claim knowledge that did not exist yet. **Bars** keep both
adjustments, and treat the adjusted series as a snapshot rather than a fact — a
split rewrites it back to the beginning, so the newest fetch wins and every
earlier version stays in the raw store. **Corporate actions** map thirteen
differently-shaped types onto one: the security an action happened to, and what
it became.

The fourth reads the **quarterly 13(f) securities list** — the only source found
that says what a security *is*: common stock, a warrant, a unit, a right. The
universe screen needs that and neither vendor supplies it. One quarter of 121 is
published as fixed-width text; the rest are a mainframe report rendered to PDF,
where plain text extraction destroys the structure because issuer name and
description are both space-containing *and* space-separated, so no pattern can
find the boundary between them. Words carry positions, so the columns are
recovered from each document's own header row rather than from fixed offsets —
those move between eras, and hardcoding them parsed one era correctly while
returning **zero rows** for another without complaint. Checked against the one
quarter published both ways: 25,333 of 25,333 records identical, nothing dropped
and nothing invented. A layout that is not recognised now raises rather than
returning what it managed to find.

The URLs are discovered rather than constructed, for the same class of reason:
two directories are in use across the window and neither is derivable from the
quarter, so the index page is the authority. Building the obvious filename
resolves for **one quarter of forty-two**.

Two further feeds exist to serve identity. EDGAR's **quarterly full index**
enumerates every filing by every filer, including filers that no longer exist —
which is what today's ticker map cannot do, and what makes the delisted
population reachable at all. From it, **13D/G submissions** are fetched whole,
because one request then carries both the SEC header naming the subject company
and the document carrying the CUSIP. That pairing is the only place in the
system where the CUSIP↔CIK edge is stated outright, with a date, by a source
that survives the company being delisted.

A fifth builds **security identity**, in two stages that are deliberately kept
apart. *Observations* are evidence and accumulate: a corporate action stating
that one symbol became another on a date, a ticker map fetched on a day, a 13D/G
naming a CUSIP and a subject company. *Intervals* are inference — no source
states when a link **ended**, so the end of one is guessed from the start of the
next, and every boundary records whether it was observed or inferred so a
backtest can refuse a name near an uncertain edge. Intervals are rebuilt
wholesale rather than patched, because the inference rule itself will improve
and two rules' output in one table cannot be told apart afterwards.

Building it produced the largest single finding of the build. Bars are
symbol-keyed and filings are CIK-keyed, and the two evidence sources are not
symmetric: corporate actions state symbol↔CUSIP across thousands of dated days,
while symbol↔CIK came only from a current-state file with no dates in it. So a
symbol resolved to a filer **if and only if it still traded today** —
**14,060 of 21,351 priced symbols could not reach a filing, 65.9%**, and
specifically the delisted ones. Survivorship bias arriving through the identity
join rather than through a timestamp.

The runner *reports* what it cannot resolve rather than dropping it, which is
the only reason the gap was a number instead of a silence.

**A third evidence source closed most of it.** A 13D/G filing states CUSIP and
subject CIK together, with a date, and survives delisting — the only source here
that reaches backwards. Enumerated from EDGAR's quarterly full index (which lists
filers that no longer exist) rather than from submissions (which are reachable
only for companies with a ticker today), and with the subject read from the SEC
header rather than from the index, which names both parties and marks neither.
32,729 filings yielded an edge from **98.6%**, and the unreachable priced
population fell from 14,060 to **11,317 (53.0%)** — but among priced symbols
whose class is identified, common stock went from **46.2% unreachable to 13.3%**.
That is the survivorship number, and the raw count is dominated by warrants,
units and funds the screen never wanted.

Two limits are recorded rather than papered over. The edge is `(CUSIP, CIK)` and
carries no symbol, so it fills only an interval that already holds a CUSIP: the
3,401 priced symbols with no identity evidence at all were reached by exactly
none of it, and the count did not move by a single row. And a CUSIP naming two
CIKs is resolved to neither and reported — 456 of 13,778, mostly genuine
successor entities — but a real CUSIP bound to the *wrong* CIK yields one CIK and
so trips no filter at all. That case is unaudited and labelled **ASSUMED** clean.

### 5.3 Features, and why v1 has no model

Features are tiered by how much they can be trusted in a historical backtest.

| Tier | What | Backtestable |
|---|---|---|
| **A — structured** | 8-K item codes, Form 4 XML, filing text *diffs*, prices, corporate actions. No model involved. | **Full history, uncontaminated** |
| **B — verifiable extraction** | Model extraction of stated facts | Historical but flagged; spot-audited against source |
| **C — judgment** | Sentiment, direction, ratings, forecasts | **Forbidden** |

**Every signal the literature supports is Tier A.** Filing-language change is a
text diff. Insider classification is Form 4 XML plus transaction history.
Activist stakes are a form type. Earnings events are an 8-K item code. None of
them requires a model.

So v1 ships without one. That removes the contamination problem, the API
dependency, the extraction cost, and an entire subsystem, while keeping every
signal with published evidence behind it. Extraction is added later, on top of
a working model-free baseline, where its incremental value can actually be
measured.

When Tier B is added, headline results stay reportable **Tier A only**, and
the A-versus-(A+B) delta is reported as the measure of how much is being
trusted to the model. That delta is diagnostic, not decisive — a large one
could be genuine signal or contamination, and it cannot distinguish them.

### 5.4 Universe — as-of, buffered, delisting-aware

Membership is rebuilt for each decision date from point-in-time inputs.

**Screen:** cap band, minimum liquidity, real common stock (not warrants,
units, or rights). Screens are structural eligibility only — economics are
handled per-trade in §5.6.

**Buffer zones on every threshold.** A name enters above the upper bound and
exits only below a lower one. Without hysteresis, names oscillating around a
threshold generate turnover the strategy never asked for, which directly
fights §7.

**Price floors apply to the raw as-of price, never the adjusted series.** One
sampled name traded at $1.27 while its adjusted history shows $2,540 for the
same day; it would pass any price floor on adjusted data while actually
trading with penny-stock spreads.

**Delisted names are retained with a terminal return, never dropped.** The
price feed simply stops at the last traded price — a failed bank's series ends
at $106 when equity went to zero. Classification:

| Event | Source | Terminal return |
|---|---|---|
| Cash merger | corporate actions (carries symbols) | Deal price |
| Stock merger | corporate actions | Convert to acquirer |
| Worthless removal | corporate actions | **−100%** |
| Bankruptcy | **EDGAR 8-K item 1.03** | **−100%** |
| **Unclassified** | — | **−100%** |

The default is pessimistic on purpose: an unknown terminal event must never be
able to flatter a backtest. Later classification can only improve a result.

**Positions key on CIK or CUSIP, never symbol** — tickers get reused, and a
symbol-keyed series silently splices two different issuers.

**The historical population is reconstructed, not inherited from today's asset
list.** EDGAR's quarterly full-index files enumerate every filing by every
filer, including companies that no longer exist — that is how the eligible pool
is rebuilt as of a past date rather than from the survivors.

The measurement that forces this: sampling names with usable history, **50 were
in band today, but 105 had been in band at some point — 2.10×**. A universe
built from today's membership tests roughly **48% of the names that actually
existed**, and specifically the half that survived, grew, or stayed put. The
other half delisted, went bankrupt, were acquired, or shrank out.

Documents are stored only for names that held **sustained** band membership,
which is the screen applied to itself rather than a storage compromise: with
buffer zones and monthly reconstitution, a name that dips across the lower
bound for a fortnight never enters the universe, so its filings are never
needed. Metadata is kept for everyone, because the as-of universe cannot be
rebuilt without it.

### 5.5 Feature store — point in time

Answers "what did we know as of the decision instant?"

At daily cadence the rule is simply **a filing dated D is eligible for
decision D+1**. Filing dates are public and historical, so this is derivable
without `retrieved_at` and is conservative by up to a full day. That makes
historical backfill legitimately backtestable, and makes the assumed-delay
machinery that contaminated the previous build unnecessary rather than merely
fixed.

`retrieved_at` remains the gate for the **live** path, where no such
public-record rule exists.

### 5.6 Strategy, turnover budget, and cost

**Strategy** is a named, versioned, deterministic function: features as-of →
target weights. No model calls, no network, no clock reads.

**The turnover budget is where economics live.** Each proposed change is
costed and rejected when expected edge does not clear expected cost.

This is deliberately *not* a universe screen. A spread is a temporary
property; excluding a name permanently for being briefly expensive is both
crude and wasteful. The threshold is a ratio, not a price: **round-trip cost
must not exceed ~25% of expected edge.** Against measured spreads, marginal
names will fail this often — that is the constraint working.

Cost estimation from daily bars is unreliable for illiquid names, so v1 uses
liquidity as the proxy and recalibrates against **real fills** once paper
trading produces them. Fills are the only honest spread measurement.

### 5.7 Risk gate

Compiled hard ceilings. `min()` only — tightening always permitted, widening
never. **Any order reducing an existing position toward zero is always
approved**, regardless of kill switch, loss limits, or order counts. Ceilings
apply to the **combined** portfolio, so two strategies cannot jointly breach a
limit neither breaches alone.

### 5.8 Approval gate — promotion and demotion

| State | Capital | Approval | Exit criteria |
|---|---|---|---|
| `research` | none | n/a | Backtest survives cost and bias controls |
| `paper` | none | none | Live results track paper expectations |
| `live_capped` | hard cap | **every order** | Sustained consistency |
| `live` | scaled cap | exceptions only | — |

**Promotion state is stored per `(strategy_id, version)` — but bounded by a
compiled ceiling it can only tighten.** If the code says `paper`, no database
row can produce `live`. Promotion past the ceiling requires a commit, which is
what makes "explicit and human" mean something.

**Demotion and halting are automatic. Safety actions never wait for a human.**

| Trigger | Action |
|---|---|
| Reconciliation mismatch | **Halt immediately** (no threshold) |
| Daily loss beyond limit | Halt for the day |
| Drawdown from peak beyond limit | Demote to `paper` |
| Live vs paper divergence | Flag for review |
| Repeated risk-gate rejections | Suspend — the strategy is misbehaving |
| Required feed stale | **Suspend, not demote** |

Suspension and demotion are different on purpose: staleness is an
infrastructure failure, and demoting a strategy for the pipeline's fault
discards real information about the strategy.

**Disagreement between strategies is not a demotion trigger.** Two strategies
wanting opposite sides of a name is a portfolio-construction question — net at
the portfolio level. Demoting for disagreement would punish diversification.

### 5.9 Multi-strategy: one live, many by schema

**v1 policy: at most one strategy above `paper`.** Capital allocation across
live strategies, and netting when they disagree, is real design work not worth
doing before a second strategy exists.

**But the data model is multi-strategy from day one.** `strategy_id` and
version tag every order, fill, feature set, and P&L row. Tagging from the start
is nearly free; retrofitting attribution across a year of untagged fills is
not.

### 5.10 Execution and reconciliation

Idempotency keys on submission — retrying a submit that actually succeeded is
how you get a double position. **Never cache broker state and serve it as
truth**; reconcile against the broker and feed fills back as observations.

## 6. Invariants

Not style preferences. If a task seems to require breaking one, **stop and
ask**.

**Money safety**

1. The LLM never originates an order.
2. A risk limit is never widened — no env var, config key, or override flag.
3. Exits are always approved.
4. Promotion state is bounded by a compiled ceiling that data can only tighten.
5. Paper unless the mode string is exactly `live`. No aliases.
6. Order submission is idempotent; broker state is reconciled, never cached as truth.

**Point-in-time integrity**

7. Gate on `retrieved_at`, never `published_at`. At daily cadence, filing date + 1 day is the eligibility rule.
8. Never default a missing timestamp to `now()` — drop the record.
9. The universe is built as-of the decision date, with buffer zones, and delisted names retained with a terminal return defaulting to −100%.
10. Positions key on CIK or CUSIP, never symbol.
11. Every feed declares a history horizon; queries beyond it fail loudly, never empty.
12. No naive datetimes anywhere.

**Model boundary**

13. The model extracts stated facts. It never forecasts, rates, or predicts.
14. Malformed model output is dropped and logged, never repaired.
15. Document text is data, never instruction.

**Measurement honesty**

16. Raw ingest is immutable; parsing is re-runnable.
17. A backtest without a spread-based cost model is invalid, not approximate.
18. Raw and adjusted prices are both required and serve different purposes: adjusted for returns, raw for sizing and cost.
19. **Hypotheses are pre-registered.** A test specified before seeing the data may run unattended; selecting a winner after seeing results may not. Every trial is logged automatically — you cannot deflate a Sharpe by a trial count you did not record.
20. Every reported number traces to a stored row.
21. **Measure; do not estimate — and a measurement covers only what it covered.** Any number, limit, size, rate, or behaviour that will inform a decision gets checked against the real thing. Where that is impossible the figure is labelled **ASSUMED**, with what would settle it and what it costs to be wrong. Generalising past what was tested — one endpoint to a feed, one host to a vendor, one tier to a plan — is worse than an open estimate, because it carries the authority of having been checked. Findings state their scope. Six confident assertions during design were each wrong until measured, by factors up to 48×, and every one was about to decide something. **And one instance is never a class:** before reporting anything about a *set* — endpoints, quarterly files, eras, symbols — enumerate it and sample at least three, the oldest, the newest and one between, never the first that worked, and state the sample size in the finding. This half is written as a procedure because the rule above it was followed in letter three times and still produced the wrong answer, most recently a quarterly file declared machine-readable on the strength of one file when 1 of 121 has a text version.

**Operations**

22. Claude Code writes this repo; Claude Desktop reads it. Never two writers.

## 7. Why turnover is a hard constraint

The literature says two things at once: **the alpha is largest in small caps,
and the costs are largest in small caps.** Transaction costs consume 2–4% per
year for the smallest stocks — enough to erase the size premium outright.
Measured round-trip cost in the target band is on the order of ~95 bps.

The resolution is holding period. Post-earnings drift runs ~60 trading days;
filing-language change shows no announcement effect at all and accrues later.
These signals reward patience.

**Daily decision cadence, weeks-to-months holding period.** Trading daily
would pay the spread roughly 20× more often against a signal that does not
move that fast.

## 8. What runs unattended

**The machine accumulates evidence continuously. Humans interpret it in
sessions.**

Safe to run unattended: ingest, normalisation, feature computation,
pre-registered tests, paper trading, walk-forward evaluation of a promoted
strategy, health and anomaly reporting, and all demotion, halt, and suspension
actions.

**Never unattended:** parameter search, strategy generation, or anything that
selects a winner across variants. Unattended search does not accelerate
research — it destroys the ability to believe any result, because every trial
raises the significance threshold.

Permanently human-gated, even when everything else is automated: promotion
past the compiled ceiling, widening any risk limit, and deploying new or
changed strategy logic.

## 9. Out of scope for v1

Named so they cannot creep back. The previous attempt declared six source
kinds and implemented two.

- **Model extraction** — deferred until a model-free baseline works
- **Fundamentals** — vendor data is restated, producing invisible lookahead
- **YouTube** — value unmeasured, entity resolution costly
- **Social** — highest manipulation surface
- **Second news vendor** — no backtest history; add at the paper stage
- **Intraday anything** — cadence is daily; recent SIP data unavailable on the current plan
- **Multi-strategy capital allocation** — schema supports it, policy defers it
- **Options, macro, transcripts**

## 10. Open questions

Five earlier questions are resolved and folded into the sections above, and
buffer widths and demotion thresholds have starting values. **Ten remain, and
the list grew rather than shrank as M2 was built** — most of these were found by
running the code against real data, which is the point of building it.

They fall into three groups. **1–3 need real fills** and cannot be settled
before paper trading. **4–6, 8 and 10 gate M3**, and are one question in
different clothes: which securities exist, which are eligible, and which can be
joined to a filing. **9 gates M4** and nothing else. **7 gated M3 until it was
measured**; its premise did not survive, and it is now a documented exclusion
rule with a narrower remainder.

1. **Market impact is unmeasured.** Only spread was estimated, and only from
   daily bars. At small size impact should be minor, but "should be" is not a
   measurement.
2. **Spread estimation is unreliable for illiquid names** — the daily-bar
   estimator inverts, reading thinly-traded names as *cheaper* than liquid
   ones. Real fills from paper trading are the fix.
3. **The thresholds are conventional, not calibrated.** Buffer widths, the
   ~25% cost-to-edge ratio, and the demotion limits are all defaults awaiting
   real data.
4. **Identity intervals begin at first observation, not first trade.** A symbol
   trading from the window's start whose first dated evidence appears years
   later carries no identity for the years between. Independent of the
   CUSIP↔CIK gap and not closed by fixing it. Unmeasured.
5. **Some priced symbols carry no identity evidence of any kind** — absent from
   every corporate action and from the ticker map, so unlike the rest there is
   nothing to join *from*. Currently unclassifiable as well as unjoinable. The
   13D/G edge reached **exactly none** of them and structurally cannot: it
   carries no symbol, so it fills only a link that already half exists. They are
   now 30% of the residual gap and need a source that dates symbol↔anything.
6. **Universe selection beyond the screen.** The screen leaves far more
   eligible names than the target universe size, so something must rank and
   select — and that rule is where overfitting will live. **The long-standing
   blocker for M3, though 4, 5 and 7 now stand in front of it: a ranking rule
   over a universe missing its delisted names would rank the wrong set.**
7. **Historical market caps are not always reconstructible — but the gap is not
   survivorship-correlated, and no longer gates the first backtest.** The
   original framing held that the failures were "mostly delisted names," which
   would have rebuilt survivorship bias inside the cap screen. Measured over 300
   symbol-interval windows against SEC XBRL: surviving names reach 90%
   point-in-time coverage 58% of the time, against 45–62% for names whose
   interval ended, with no trend against age. So the rule is **no reconstructible
   cap, no entry** — a roughly unbiased haircut on universe size rather than a
   survivor filter. Three narrower pieces remain open: 2016–2019 is untested
   because those names still lack a CIK (question 5); about 28% of windows have
   partial coverage that is probably question 4's interval granularity rather
   than missing data; and end-of-life decay was looked for and not found, in a
   sample of three.
8. **What counts as common stock is undecided.** The 13(f) class label is kept
   verbatim, and there are over twelve thousand distinct values because labels
   embed rates and expiry dates. Exact matching against a whitelist is out; the
   screen needs a prefix or pattern rule, and that rule decides which securities
   are eligible at all.
9. **The resolved trading mode is recorded nowhere, and must be before anything
   executes.** Settings are resolved once into a frozen object so that "what was
   the trading mode when that order went out?" has exactly one answer for the
   life of a process — but nothing writes that answer down, so afterwards there
   is none. Harmless today and not for long: nothing in the package reads the
   mode, the live broker host is absent from it entirely, and every HTTP call it
   makes is a `GET`, so no order can be placed in any mode. All three stop being
   true at M4, which is the first code that branches on mode and the first that
   POSTs. The mode must be persisted with every order before that, not after.
10. **The 13D/G subject CIK is unaudited, and its one wrong answer is
   invisible.** A malformed CUSIP is caught by its check digit, and a CUSIP
   naming two CIKs is withheld and reported — but a *real* CUSIP bound to the
   *wrong* CIK yields one CIK per CUSIP and trips neither guard. Corroborating
   against issuer names is only possible where the filer still appears in the
   current ticker file, which excludes the delisted names the edge was built
   for. Labelled **ASSUMED** clean. If it is wrong, some fraction of the joins
   attach one company's filings to another company's prices, and nothing
   downstream can distinguish a filled CIK from an observed one.

A further question — whether numeric feeds could skip raw-payload storage — was
**closed by measuring it.** Gzipped bar payloads cost 25.9 B/bar against 146.5 B
for the typed row derived from them, so keeping raw costs about half a gigabyte
across the whole history: the saving the exception was trading for does not
exist. It would also have been wrong on its own terms, since the adjusted price
series is retroactively revised and re-fetching cannot recover what the vendor
said on a past date. Invariant 14 stands with no exception.
