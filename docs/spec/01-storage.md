# Spec 01 — Storage

Dual-track. SQLite is the system of record (correctness, transactions, durability). DuckDB on Parquet is the analytics path (10–50× faster on heavy GROUP-BYs over millions of rows).

---

## Locations

```
%APPDATA%\Lockes-Picks\
├── db.sqlite               ← system of record (WAL mode)
├── parquet/                ← analytics
│   ├── picks/              ← partitioned by season
│   ├── rosters/
│   ├── projections/        ← partitioned by scoring_profile, season
│   ├── adp/                ← partitioned by site, date
│   └── residuals/
├── models/                 ← versioned model artifacts (binary)
└── logs/
```

---

## SQLite schema

WAL mode + tuned PRAGMAs (`synchronous=NORMAL`, `temp_store=MEMORY`, `mmap_size=2GB`).

### Core tables

```sql
players                 (player_id PK, name, pos, team, age, draft_round, draft_pick, rookie_year, college, ...)
player_alias            (alias_id PK, player_id FK, source [sleeper|ud|dk|pfr], external_id, name_variant)
teams                   (team_id PK, abbr, division, conference, pace_estimate, ...)
schedule                (season, week, home_team, away_team, kickoff, implied_total_home, implied_total_away)

drafts                  (draft_id PK, site [ud|dk], contest_type, scoring_profile, season, started_at, completed_at)
picks                   (draft_id FK, pick_number, slot, player_id FK, ts)         -- composite PK
rosters                 (draft_id FK, slot, position, player_id FK)                -- materialized post-draft

scoring_profiles        (profile_id PK, name, ppr, te_premium, td_value, ...)
projections             (player_id FK, scoring_profile_id FK, season, point_estimate,
                         posterior_sigma, conformal_lower, conformal_upper, model_version_id FK)

rankings_uploads        (upload_id PK, uploaded_at, source_path, row_count)
rankings_rows           (upload_id FK, player_id FK, my_rank, my_tier, override_bonus, note)
overrides               (player_id FK, scoring_profile_id FK, value, adp_at_set, set_date,
                         decay_rate, capped_at)                                    -- composite PK

playoff_teams           (playoff_team_id PK, source [ud], season, contest, slot, advance_round, points, prize)
playoff_team_picks      (playoff_team_id FK, pick_number, round, player_id FK)     -- composite PK

draft_log               (event_id PK, draft_id FK, ts, event_type, payload_json)   -- event-sourced live log

model_artifacts         (model_version_id PK, kind, created_at, training_data_hash, hyperparams_json,
                         metrics_json, blob_path)                                  -- versioned, never overwritten

selector_packs          (pack_id PK, site, version, created_at, selectors_json, active_bool)
```

### Tables for the diagnostic engine

```sql
residuals               (model_version_id FK, playoff_team_id FK, residual, predicted, actual, fold)
                        -- 5-fold OOF residuals, the unifying primitive
features_registered     (feature_set_version PK, registered_at, features_json, git_commit_sha)
                        -- pre-registered feature config; immutable per version
findings                (finding_id PK, kind [association_rule|subgroup|community|stack_signature],
                         payload_json, p_value, q_value_bh, bootstrap_lower, bootstrap_upper,
                         permutation_null_p, status [experimental|validated|live_clv_confirmed])
clv_log                 (draft_id FK, pick_number, player_id, your_pick_adp, market_adp_at_pick,
                         clv, finding_id FK_nullable)
                        -- per-pick CLV; tied to findings to gate validation
```

---

## DuckDB on Parquet — analytics layer

DuckDB reads SQLite directly, but we materialize hot analytic tables to Parquet for speed:

- `parquet/picks/season=*/draft.parquet` — every pick across all seasons.
- `parquet/rosters/season=*/roster.parquet` — every team's full roster post-draft.
- `parquet/roster_pairs/` — pre-aggregated player-pair co-occurrence (Phase 1.5). Hot path for FP-Growth, vine copulas, graphical lasso.
- `parquet/projections/profile=ud_half_ppr/season=*/proj.parquet` — projections per profile.
- `parquet/residuals/model_version=*/r.parquet` — OOF residuals per model version.

**Polars** for ETL: SQLite → Parquet, nflverse → Parquet, ADP scrape → Parquet. Pandas only when polars doesn't support a downstream library.

DuckDB queries get auto-routed via a thin `analytics.query()` helper that prefers Parquet and falls back to SQLite ATTACH if a table isn't materialized.

---

## Event-sourced live draft log

Every observed pick is an immutable row in `draft_log` with `event_type='pick_observed'` and `payload_json` capturing the raw DOM-extracted state. Why:

1. **Replayability** — re-run the recommender on the same draft for testing.
2. **Opponent-model training data** — every pick by every opponent is now data.
3. **Nightly recalibration** — `adp_stddev(pick)` curve, archetype β coefficients, and ISMCTS rollout priors all refit overnight from the accumulated log.
4. **Bug forensics** — reconstruct any past in-draft recommendation exactly.

Other event types: `pick_overturned` (rare; UD has fixed bad picks before), `draft_started`, `draft_completed`, `tab_focused`, `selector_pack_swapped`, `recommendation_emitted`.

---

## Versioned model artifacts

Every retrain writes a new row in `model_artifacts` with a unique `model_version_id`. Projections, residuals, and findings reference this version. Mid-season recalibration is a new row, never an UPDATE.

A `model_artifacts.kind` enum covers: `lightgbm_advance_clf`, `bayesian_projections_ud`, `bayesian_projections_dk`, `cox_availability`, `archetype_betas`, `vine_copula_matrix`, `leiden_communities`, `causal_forest_treatments`, `pick_value_chart`, `ismcts_policy_prior`, etc.

`blob_path` points to a file under `models/{kind}/{model_version_id}.bin` — pickle/joblib for sklearn, ONNX where available, native format otherwise.

---

## Pre-registered feature config

`config/features.toml` (committed). Updated *once per analysis cycle* (~monthly), with the change recorded in `features_registered` and tied to the git commit SHA.

The held-out BBM-VI Parquet partition is gated behind a `--use-sacred-set` flag in the CLI. The flag is logged to `audit.log` so we know exactly when sacred data was touched.

---

## Backup

Nightly `db.sqlite` backup to `%APPDATA%\Lockes-Picks\backups\db-YYYYMMDD.sqlite.gz`. Parquet is regenerable from SQLite + nflverse, so it's not backed up. Models in `models/` *are* backed up (deterministic re-derivation isn't free).
