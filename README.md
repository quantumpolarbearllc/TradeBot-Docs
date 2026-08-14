# TradeBot — Documentation

Design and state documentation for a daily-rebalance research system for US
small-cap equities.

**Start with [BRIEF.md](BRIEF.md)** — it is written to bring a reader fully
current in one pass, with no prior context.

| File | Contents |
|---|---|
| [BRIEF.md](BRIEF.md) | Current state, key decisions, what happens next |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Pipeline, components, invariants |

Field notes, the decision log, and the literature review are kept in the private
code repository rather than here. Their conclusions are summarised in
[BRIEF.md](BRIEF.md).

---

**This repository is documentation only.** The implementation lives in a
separate private repository. No credentials, strategy thresholds, or
proprietary logic appear here.

Claims in these documents are marked **MEASURED** (verified against a live API
on a stated date) or **ASSUMED** (not tested). That distinction is deliberate
and load-bearing.
