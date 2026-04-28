# Locke's Picks — Plan Overview

Personal best-ball draft assistant + diagnostic engine. Single user, Chrome MV3 extension + local Python backend, Windows, $0 runtime budget. Drafts ~3K teams/season targeting BBM-style top-heavy tournaments. The user constructs portfolios; the tool surfaces signal. No prescriptive automation.

> Two coherent systems on shared infrastructure. The Live Draft Engine ships the in-draft experience; the Diagnostic Engine mines historical rosters offline. Same storage, same pre-registered features, same blind-review discipline.

---

## Two systems

| | Live Draft Engine | Diagnostic Engine |
|---|---|---|
| What | Pick recommendations, EV, opponent model, lookahead, in-draft portfolio signals | Mining historical BBM rosters for correlations, outliers, and inversions |
| When | Every pick of every active draft (≤ 10 concurrent) | Nightly batch + on-demand reports |
| Output | Side-panel cards + chips per candidate | Standardized signal feeds + dashboard |
| Validation | Live CLV in real drafts | Held-out BBM VI + permutation null |

---

## Constraints (locked)

| | |
|---|---|
| Audience | Single user. No auth, no billing, no public release. |
| Platforms | Underdog + DraftKings best ball only. NFL only. |
| Browser | Chrome (MV3) only. Desktop. |
| OS | Windows. |
| Concurrent drafts | Up to 10 live tabs. |
| Page mutation | **None.** Content scripts are passive observers; UI lives in a Chrome **side panel**. |
| Hosting | Local-first. Backend on `localhost`, SQLite + DuckDB+Parquet at `%APPDATA%\Lockes-Picks\`. |
| Budget | $0 runtime. OpenAI for dev only. |
| Out of scope | Mobile, snake/Battle Royale, redraft, auction/dynasty, DFS, in-season lineup, multi-user, multi-sport. |

---

## Stack

- **Backend:** Python 3.12, FastAPI, SQLite (system of record), DuckDB on Parquet (analytics), polars (ETL), NumPy.
- **Bayesian:** PyMC 5 + nutpie via conda-forge (NumPyro/JAX off-limits on Windows).
- **ML/stats:** LightGBM, scikit-learn, MAPIE, lifelines, mlxtend, pysubgroup, econml, igraph + leidenalg, pyvinecopulib, statsmodels, faiss-cpu.
- **Extension:** Chrome MV3, TypeScript, React side panel — thin observer; all logic in backend.
- **Deferred:** PyTorch (only used by backlog items β-TCVAE, NSF — install when those land).

---

## Spec files (read each separately)

- [`spec/00-data-acquisition.md`](./spec/00-data-acquisition.md) — what data we actually have, audit before scope.
- [`spec/01-storage.md`](./spec/01-storage.md) — SQLite + DuckDB dual track, event-sourced draft log, versioned model artifacts.
- [`spec/02-projections.md`](./spec/02-projections.md) — hierarchical Bayesian, MAPIE conformal, Sobol QMC + CRN, lightweight Cox PH, late-season boosts, scoring profiles.
- [`spec/03-stacks-and-taxonomy.md`](./spec/03-stacks-and-taxonomy.md) — two-track stack handling, position-signature taxonomy, vine-copula tail dependence, "how much is too much."
- [`spec/04-diagnostic-engine.md`](./spec/04-diagnostic-engine.md) — calibrated OOF residuals as unifying primitive, FP-Growth, MI, graphical lasso (SKEPTIC), Leiden, pysubgroup, causal forest, empirical pick-value chart.
- [`spec/05-live-engine.md`](./spec/05-live-engine.md) — CVaR tournament utility, payout-curve config, Bayesian opponent archetypes, RNR, ISMCTS, round-aware reach tolerance.
- [`spec/06-portfolio-signals.md`](./spec/06-portfolio-signals.md) — Leverage, Effective N, P(top-0.1%), Cluster exposure, IR vs ADP-bot, Portfolio CVaR, ADP velocity, Market Visualizer.
- [`spec/07-extension.md`](./spec/07-extension.md) — MV3 architecture, passive content scripts, React side panel, 10-tab concurrency, latency budget.
- [`spec/08-blind-review.md`](./spec/08-blind-review.md) — held-out BBM VI, pre-registered features, standardized signal feed, garden-of-forking-paths, live CLV gate.
- [`spec/09-deferred-and-skip.md`](./spec/09-deferred-and-skip.md) — backlog (β-TCVAE, DiCE, knockoffs, etc.) and permanent-skip list with rationale.

Phased task breakdown lives in [`MILESTONES.md`](./MILESTONES.md).

---

## Advisories (items pushed back on from the v2 draft)

1. **Timeline.** v2 proposed a 12-week core. Solo + $0 + no-rush realistic timeline is **6–12 months calendar / 35–55 focused weekends**. Ordering preserved; estimates rebuilt in `MILESTONES.md`.
2. **Data availability.** "~2.5M BBM rosters" is aspirational — full pick-by-pick rosters with outcomes for I–VI are not all publicly available in one place. Phase 0 is a **data acquisition assessment**; method calibration depends on actual N. See `spec/00`.
3. **ISMCTS sequencing.** Needs the Bayesian opponent archetype model + a policy prior to fit < 100 ms. Dependency enforced: archetypes → policy prior → ISMCTS. Detailed in `spec/05`.
4. **PyTorch in stack overview.** Only used by backlog generative models. Deferred from day-1 install.
5. **Restricted Nash Response, knockoffs, Bayesian network structure learning, causal transformer.** RNR kept in core (heavyweight but tractable); the others moved to backlog and flagged "possibly never needed at personal scale" in `spec/09`.
6. **Cox PH** kept lightweight per v2. Input data (game-by-game injury logs + snap-share trends) needs a completeness check in Phase 0.
7. **DuckDB + SQLite dual storage** is sound but adds operational complexity. Worth it; called out so it's not invisible.

**Preserved from earlier plan (not in v2 but still in scope):**
- Round-aware reach tolerance — explicit `adp_stddev(pick)` curve as side-panel signal and sim sanity check (`spec/05`).
- Market Visualizer UI surface (`spec/06`).
- Scoring profiles UD ½-PPR vs DK full-PPR — every projection respects the active profile (`spec/02`).
- Custom rankings CSV with `override_bonus` column + ADP-decay hygiene + hard cap + stale-review (`spec/02`).
- 10-concurrent-tab handling (`spec/07`).
- Underdog playoff team learning DB — folded into the Diagnostic Engine (`spec/04`); the playoff classifier surface is in the live engine (`spec/05`).

---

## Phases (summary — see MILESTONES for sub-tasks)

| Phase | Theme | Dogfood-able? |
|---|---|---|
| 0 | Data acquisition + repo scaffolding | — |
| 1 | Storage + ETL backbone | — |
| 2 | Foundation models (projections, conformal, Cox, late-season boosts) | — |
| 3 | Stack analytics + correlation mining | — |
| 4 | Subgroup + causal analytics | — |
| 5 | Live engine objective + sampling (CVaR, QMC, reach tolerance, override hygiene) | partial |
| 6 | Opponent modeling + ISMCTS lookahead | — |
| 7 | Backend + extension wiring (FastAPI, content scripts, side panel) | **yes** |
| 8 | Diagnostic UI + portfolio signals (Market Visualizer, Leverage, etc.) | yes |
| 9 | Live integration + CLV validation gate | yes |
| 10 | Polish, dogfood, bug bash | yes |
