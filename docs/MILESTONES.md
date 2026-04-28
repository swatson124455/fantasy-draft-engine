# Locke's Picks — Milestones

Granular phased task list. Each task is sized for a focused weekend or less. **Total realistic: 35–55 focused weekends** (≈ 6–12 calendar months solo).

Dependencies called out explicitly with `→` notation. Anything marked **(blocker)** must finish before downstream phases can start.

---

## Phase 0 — Data acquisition + repo scaffolding (3–5 weekends)

- [ ] 0.1 Repo scaffolding: monorepo, `engine/`, `extension/`, `data/`, `scripts/`, `docs/`, lint, types, GitHub Actions CI, Windows build script.
- [ ] 0.2 **(blocker)** Data acquisition assessment. For each: identify source, scrape/download, verify completeness. Output: a one-page audit of what we actually have.
  - BBM I–VI pick-by-pick rosters
  - BBM I–VI advance/finalist/winner outcome labels
  - Underdog ADP history (UD draft variance)
  - DraftKings ADP history
  - Sleeper API player IDs + projections feed
  - nflverse seasonal + game-level stats
  - nflverse / PFR injury logs + snap-share trends
  - Public BBM data drops (Hayden Winks, finalist tweets/threads)
- [ ] 0.3 Player ID alias table seed (Sleeper IDs ↔ UD names ↔ DK names ↔ PFR IDs).
- [ ] 0.4 Manual loader UI scaffolded for late-arriving CSV uploads.
- [ ] 0.5 nflverse historical pull (last 6 seasons) cached locally.
- [ ] 0.6 Sleeper API nightly sync stub.
- [ ] 0.7 ADP scraper for UD + DK (politely rate-limited, daily).

**Exit criterion**: We know exactly how many labeled rosters we have (could be 200K, could be 2M). Method thresholds in later phases calibrated to this number.

---

## Phase 1 — Storage + ETL backbone (3–4 weekends)

- [ ] 1.1 SQLite schema (system of record): `players`, `drafts`, `picks`, `rosters`, `exposures`, `rankings_uploads`, `overrides`, `projections`, `model_artifacts` (versioned), `playoff_teams`, `playoff_team_picks`.
- [ ] 1.2 SQLite WAL mode + tuned PRAGMAs.
- [ ] 1.3 DuckDB + Parquet analytics layer in `data/parquet/`.
- [ ] 1.4 polars ETL pipeline: nflverse → Parquet, picks → Parquet.
- [ ] 1.5 Roster-pair pre-aggregation tables (hot mining queries).
- [ ] 1.6 Event-sourced draft log: every observed pick is an immutable row.
- [ ] 1.7 Versioned model-artifact storage: mid-season retrains are new rows, not migrations.
- [ ] 1.8 Pre-registered feature config file (`config/features.toml`); committed before residuals are inspected.
- [ ] 1.9 Held-out BBM VI flag — separate Parquet partition, gated behind a "use sacred set" CLI flag.

---

## Phase 2 — Foundation models (5–7 weekends)

- [ ] 2.1 Empirical pick value chart fit from BBM data → `config/pick_value_chart.parquet`. Used as feature weight downstream.
- [ ] 2.2 Scoring profiles primitive: `ud_half_ppr`, `dk_full_ppr`. Profile-keyed projection table.
- [ ] 2.3 LightGBM `P̂(advance)` baseline classifier with 5-fold OOF predictions + isotonic calibration.
- [ ] 2.4 OOF residuals `r_i = y_i − P̂_i` written to `residuals` table.
- [ ] 2.5 TreeSHAP attribution on top/bottom residual deciles.
- [ ] 2.6 Hierarchical Bayesian projections (PyMC + nutpie): player nested in position nested in team, partial pooling, posterior σ per player. Both UD and DK profiles.
- [ ] 2.7 MAPIE CV+ conformal intervals with locally-normalized residuals around Bayesian projections.
- [ ] 2.8 Sobol QMC sampler harness (`scipy.stats.qmc.Sobol(scramble=True)`) replacing `np.random` in sims.
- [ ] 2.9 Common Random Numbers across candidate-pick comparisons.
- [ ] 2.10 Importance sampling for tail scenarios.
- [ ] 2.11 Recency weighting `γ^(2026 − season)` with γ tuned on held-out backtest.
- [ ] 2.12 Rookie cold-start prior: position-stratified hierarchical prior shifted by `f(draft_capital, college_features)`.
- [ ] 2.13 Lightweight Cox PH availability hazard. Covariates: age, position, prior-season games missed, injury-history flag. Output: weekly hazard multiplier.
- [ ] 2.14 Late-season boost layer: rookie ramp (W10–17, position-specific), sophomore upside-variance boost, handcuff conditional value (tied to lead-back hazard from 2.13).
- [ ] 2.15 Boost magnitudes fit on training seasons; validated on held-out BBM VI (one shot).
- [ ] 2.16 Custom rankings CSV import (template provided): `player_id, name, team, pos, my_rank, my_tier, override_bonus, note`.
- [ ] 2.17 Override hygiene: `adp_at_override_time` snapshot, ADP-decay with round-scaled threshold, ±15% hard cap, stale-override review panel data.

---

## Phase 3 — Stack analytics + correlation mining (5–7 weekends)

- [ ] 3.1 Two-track stack feature engineering: QB-anchored vs non-QB. `qb_stack_size_2..5+`, `nonqb_stack_size_2..4+`.
- [ ] 3.2 Position-signature taxonomy across cardinalities 2–5+ (full enumeration in `spec/03`).
- [ ] 3.3 Sub-classifications: with/without bring-back, with/without pass-catching RB, etc.
- [ ] 3.4 Vine copula tail-dependence matrix via `pyvinecopulib`. Per-pair λ_U and λ_L.
- [ ] 3.5 Empirical "stackable" definition layer — any pair with significant λ_U is stackable regardless of position.
- [ ] 3.6 Position-pair lift table: QB+WR (by target-share tier), QB+TE (by RZ-share tier), QB+RB, WR+WR same-team, WR+TE same-team, RB+WR, RB+TE.
- [ ] 3.7 FP-Growth + lift/conviction/leverage via `mlxtend`. Fisher-exact p-values, BH-FDR, bootstrap-BCa CIs.
- [ ] 3.8 Within-slot conditional rates shown alongside marginal lift.
- [ ] 3.9 Mutual information sweeps: player → advance, conditional on slot.
- [ ] 3.10 Graphical lasso with nonparanormal SKEPTIC trick (`GraphicalLassoCV` + Kendall's tau).
- [ ] 3.11 Leiden community detection on PMI-weighted player graph; 50-run consensus clustering.
- [ ] 3.12 "How much is too much" module per track: marginal advance-rate curves with bootstrap-BCa, BH-FDR.
- [ ] 3.13 Inversion detection: isotonic regression p-value test (monotone vs concave-plateau vs inverted-U).
- [ ] 3.14 Tail-asymmetry diagnostic: λ_U − λ_L per stack size; surfaced alongside advance-rate curves.
- [ ] 3.15 Cross-track stack interaction: total-stacked-players curve, substitution-vs-complement 2D heatmap, concentration-vs-distribution.
- [ ] 3.16 Capital-controlled variants of all curves using the pick-value chart from 2.1.

---

## Phase 4 — Subgroup + causal analytics (3–4 weekends)

- [ ] 4.1 `pysubgroup` beam-search depth 3 on residual column. Run twice (positive and negative residuals).
- [ ] 4.2 Permutation-null p-values (analytical nulls biased under selection); BH-FDR.
- [ ] 4.3 Split-conformal anomaly p-values within-year.
- [ ] 4.4 `econml.CausalForestDML` on dichotomized treatments: has-double-stack, has-bring-back, RB-heavy-top-5, late-onset-elite-TE.
- [ ] 4.5 Causal forest treatments expanded to position-composition contrasts: QB-WR-WR vs QB-WR-TE, QB-TE-RB vs QB-WR-RB, 4-stack with TE vs without, bring-back conditional on composition.
- [ ] 4.6 Drafter-skill controls included where data allows.
- [ ] 4.7 Trim extreme propensities; Athey-Wager calibration test.
- [ ] 4.8 Conditional-inversion τ̂(x) heatmaps over team implied total, draft slot, season, playoff schedule strength.

---

## Phase 5 — Live engine objective + sampling (3–4 weekends)

- [ ] 5.1 Payout curve config layer: `config/payouts/bbm6.toml`, `bbm5.toml`, `dk_best_ball.toml`.
- [ ] 5.2 Convex tournament utility / CVaR-based EV scoring (Hunter-Vielma-Zaman 2016 anchor).
- [ ] 5.3 Round-aware reach tolerance: explicit `adp_stddev(pick)` curve. Surfaced as side-panel chip ("5-pick reach in R2 = penalty; 5-pick reach in R13 = noise").
- [ ] 5.4 `flex_bonus` for late-round picks that complete a stack / fill a thin position / pair with high-co-occurrence partners.
- [ ] 5.5 Recommender top-N output with one-line reasoning string, late-season chip, stack badge, scoring-profile badge.
- [ ] 5.6 Idempotent pick handler keyed on `{draftId, pickNumber}` hash.

---

## Phase 6 — Opponent modeling + ISMCTS lookahead (5–7 weekends)

- [ ] 6.1 **(blocker for 6.5+)** Hierarchical Bayesian opponent archetype model: K=6 archetypes (Zero-RB, Hero-RB, Robust-RB, Late-Round-QB, Stack-Heavy, ADP-Mechanical). Pre-train per-archetype β coefficients offline.
- [ ] 6.2 Closed-form Dirichlet-multinomial runtime updates (~10 lines NumPy).
- [ ] 6.3 Restricted Nash Response counter-strategies — one per archetype, pre-computed offline (Johanson-Zinkevich-Bowling 2007).
- [ ] 6.4 Posterior-weighted or UCB-selected RNR at runtime.
- [ ] 6.5 Policy prior derived from archetype model for ISMCTS rollout guidance.
- [ ] 6.6 Information Set MCTS lookahead implementation (~400–600 LOC NumPy, Cowling-Powley-Whitehouse 2012).
- [ ] 6.7 ISMCTS perf budget tuning: 500–2000 sims per pick under 100 ms with policy prior.
- [ ] 6.8 Determinization distribution sourced from opponent posterior.

---

## Phase 7 — Backend + extension wiring (4–5 weekends) — **dogfood-able after this phase**

- [ ] 7.1 FastAPI routes: `/draft/state`, `/recommend`, `/availability`, `/portfolio`, `/diagnostics`.
- [ ] 7.2 Pydantic request/response schemas.
- [ ] 7.3 WebSocket draft session manager handling 10 concurrent sessions.
- [ ] 7.4 Reconnection + draft-state replay on socket loss.
- [ ] 7.5 PyInstaller `.exe` build.
- [ ] 7.6 Windows Startup folder shortcut for autostart on login.
- [ ] 7.7 Chrome MV3 manifest + service worker.
- [ ] 7.8 Underdog content script: passive `MutationObserver`, debounced, READ-ONLY.
- [ ] 7.9 DraftKings content script: parity with UD.
- [ ] 7.10 Versioned selector pack stored in SQLite, hot-loadable without re-publishing the extension.
- [ ] 7.11 Background service-worker WebSocket bridge to localhost:8000.
- [ ] 7.12 Tab-tracking: side panel auto-shows the active draft tab.
- [ ] 7.13 Side panel React UI: suggestions list, EV column, late-season chips, stack badges, scoring-profile badge, `adp_stddev` reach label.
- [ ] 7.14 Drafted-player removal across all tabs within ~200 ms.

---

## Phase 8 — Diagnostic UI + portfolio signals (3–5 weekends)

- [ ] 8.1 Standardized signal-feed format (every diagnostic output looks the same).
- [ ] 8.2 Garden-of-forking-paths permutation diagnostic on every reported finding (100 permutations, alternative-thresholds).
- [ ] 8.3 Portfolio dashboard: exposures, player-pair co-occurrence matrix, build-path heatmap.
- [ ] 8.4 Market Visualizer: per-round contrarian/chalk view, position deviation chart, drill-down.
- [ ] 8.5 Leverage Score per player and per stack: `(your_exposure − field_exposure) × ceiling`.
- [ ] 8.6 Effective N (Herfindahl-Hirschman) on player exposures, team-clusters, build archetypes.
- [ ] 8.7 P(at least one entry top-0.1%) from MC sim — the actual North Star.
- [ ] 8.8 Cluster exposure per NFL team / game / archetype.
- [ ] 8.9 Information Ratio vs ADP-bot baseline.
- [ ] 8.10 Portfolio CVaR at α=0.05.
- [ ] 8.11 ADP velocity, acceleration, volatility-adjusted edge.
- [ ] 8.12 Position-spend-by-round delta heatmap (your average vs playoff teams' average).

---

## Phase 9 — Live integration + CLV validation gate (2–3 weekends)

- [ ] 9.1 Playoff-similarity score per in-progress roster (cosine to advancing-team centroid).
- [ ] 9.2 Playoff-prob chip per candidate in side panel.
- [ ] 9.3 Closest-historical-match link in side panel.
- [ ] 9.4 ISMCTS recommendation + reasoning surfaced.
- [ ] 9.5 Live CLV tracker: per-pick CLV, per-draft CLV, rolling CLV.
- [ ] 9.6 CLV validation gate: any new pattern from the diagnostic engine is flagged "experimental" until it produces measurable CLV in N live drafts.

---

## Phase 10 — Polish, dogfood, bug bash (3–4 weekends)

- [ ] 10.1 Real-draft dogfood (the user, on real BBM and Battle Royale entries).
- [ ] 10.2 End-to-end latency budget: ≤ 250 ms from pick observed to side panel re-rendered. Profile and optimize.
- [ ] 10.3 10-tab concurrent perf testing.
- [ ] 10.4 WebSocket reliability: reconnect, replay, dedup verified under load.
- [ ] 10.5 Selector-pack hot-load verified end-to-end.
- [ ] 10.6 Stale-override weekly review email/notification.
- [ ] 10.7 Bug bash whatever surprises in production.

---

## Phase totals

| Phase | Weekends |
|---|---|
| 0 | 3–5 |
| 1 | 3–4 |
| 2 | 5–7 |
| 3 | 5–7 |
| 4 | 3–4 |
| 5 | 3–4 |
| 6 | 5–7 |
| 7 | 4–5 |
| 8 | 3–5 |
| 9 | 2–3 |
| 10 | 3–4 |
| **Total** | **39–55** |

Dogfood-able from end of Phase 7. Phases 8–10 happen in parallel with real drafting once the extension works.
