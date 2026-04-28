# Spec 02 — Projections, Conformal Intervals, Sampling, Late-Season Boosts

The foundation layer. Every downstream component (EV, opponent model, sims, diagnostic residuals) consumes the posterior σ from this layer.

---

## Hierarchical Bayesian projections

**Library:** PyMC 5 + nutpie via conda-forge. NumPyro / JAX off-limits on Windows.

**Structure:** Player nested in position nested in team with partial pooling.

```
α_team       ~ Normal(0, σ_team)
α_position   ~ Normal(α_position_global, σ_position)        # per position
α_player     ~ Normal(α_position[pos[i]] + α_team[team[i]], σ_player)
y_player     ~ Normal(α_player + X β, σ_y)
```

`X β` covariates: age, draft capital, prior-season usage (target share, carry share, snap share), team pace, opponent strength, schedule strength.

Output per player: full posterior — both `point_estimate` and `posterior_sigma`. Sigma is the entire point of using Bayes here. Best ball is won on spike weeks; mean projections without uncertainty miss the tails that matter.

**Rookie cold-start.** Rookies have no prior-season usage. Position-stratified hierarchical prior shifted by `f(draft_capital, college_features)`. Prior gets tighter as preseason news lands. May–July is the CLV window for rookie ADP.

**Recency weighting.** Sample weight `γ^(2026 − season)` with γ tuned by held-out backtest. Older seasons inform priors, recent ones dominate the likelihood.

**Run cadence:** initial fit at season start; partial refits weekly during draft season as ADP and news shift; full refit nightly during the season for in-progress weekly inference.

---

## Conformal prediction intervals

**Library:** MAPIE.

**Method:** CV+ with locally-normalized residuals. Wraps the Bayesian point estimates with marginal-coverage guarantees independent of model misspecification.

Why not stop at Bayesian σ: PyMC posteriors miss schedule effects, weather, and structural shifts the model doesn't see. Conformal layers on top with empirical coverage guarantees.

Output: `conformal_lower`, `conformal_upper` written alongside `point_estimate` and `posterior_sigma` in `projections`.

The sim layer (next section) draws from a mixture: 80% from the Bayesian posterior, 20% from the conformal interval treated as a uniform tail. Importance-sampled when the convex utility is in play.

---

## Quasi-Monte Carlo + Common Random Numbers + Importance Sampling

Replace `np.random` everywhere with `scipy.stats.qmc.Sobol(scramble=True)`.

**Why QMC:** Variance reduction. For ~1000 sims, Sobol gets the same accuracy as ~10,000 pseudo-random draws on smooth integrands.

**Common Random Numbers (CRN):** Same Sobol sequence reused across all candidate-pick comparisons. Eliminates noise from `EV(A) − EV(B)` comparisons because the same scenarios are evaluated for both. Without CRN, two near-identical players can flip ranks just from sim noise.

**Importance Sampling:** Convex tournament utility (Spec 05) is dominated by the 99th-percentile tails. Oversample tail scenarios via importance weights. The mean estimator is unbiased; variance falls 5–10× on tail-driven payouts.

---

## Lightweight Cox PH availability hazard

**Library:** lifelines.

**Scope (deliberately small):** Covariates limited to:
- age
- position
- prior-season games missed
- injury-history flag (multi-season indicator)

No snap-count rolling averages. No contact-injury subtypes. No `age × position` deep interactions. Injuries are rarely predictable; we want a coarse correction factor on weekly availability, not a complex survival model.

**Output:** Per player, a weekly hazard multiplier `h(t)` applied to that week's availability term. That's it. Goes into the projection as `expected_games_played × h(t)`.

**Refit cadence:** Season start, then weekly during the season as new injuries land.

---

## Late-season boost layer (additive, on top of Cox)

The Cox model is downside-focused. Some players' expected role *grows* in weeks 10–17 — that's an upside catalyst the survival model doesn't capture. This is its own layer.

**Boost magnitudes are fit on training seasons (BBM I–V) and validated on held-out BBM VI before deployment.** They never get re-tuned on BBM VI.

### Rookie late-season ramp

Rookies, especially WRs and RBs, materially outperform their full-season averages in W10–17 as playbooks open up.

```
rookie_late_season_boost(player, week) =
    f(draft_capital, position, weekly_target_share_trend) × ramp(week)
```

Ramp is a smooth function (sigmoid centered ~W11), not a step. Position priors:
- WR rookies: strongest pattern.
- RB rookies: second-strongest.
- TE rookies: muted.
- QB rookies: highly variable, treated cautiously.

### Sophomore boost

Year-2 leap is real at WR and TE. ~15–25% **upside variance** increase relative to rookie-year sample. This is a *spike-week probability* boost, not a mean shift — exactly the kind of asymmetry best ball values.

Larger when team-level circumstances changed favorably:
- New starting QB.
- New OC.
- Teammate injury that creates target/touch opportunity.

These conditions are tracked in a manually curated `sophomore_context.csv` per season (Spec 00).

### Handcuff RBs as conditional spike assets

Value almost entirely contingent on the lead back's hazard.

```
handcuff_value = lead_back_injury_hazard × handcuff_role_inheritance × weeks_remaining
```

- `lead_back_injury_hazard`: directly from the Cox model above. Tied, not duplicated.
- `handcuff_role_inheritance`: 1.0 for clear RB1 inheritor, 0.4–0.7 for RBBC handcuff. Curated table per season.
- `weeks_remaining`: late-season weighting kicks in — handcuffs are vastly more valuable when the inheritance window aligns with W15–17.

The handcuff's full *distribution* gets a long right tail tied to the lead back's hazard, not just a mean shift. The sampler in Spec 05 draws from a mixture: low-mean baseline most weeks, fat right tail conditional on the lead back going down.

---

## Scoring profiles (UD ½-PPR vs DK full-PPR)

The Bayesian model is profile-aware. The same model fits each profile by swapping the scoring parameter in the likelihood:

```
y_player(profile) ~ Normal(α_player + X β, σ_y)        # where y is computed under `profile` scoring
```

Outputs are stored as separate rows in `projections` keyed by `(player_id, scoring_profile_id, season)`.

Tier breaks recompute per profile. So tier-3 RB on UD ≠ tier-3 RB on DK. Reach penalty and EV scoring (Spec 05) use the profile of the active draft.

Profile-agnostic transfer (transfers directly):
- Build-path labels.
- Stack patterns.
- Bye-week conflict logic.

Cross-format adjustment (UD → DK) for playoff-DB lessons:
1. Re-project player values through `dk_full_ppr`.
2. Re-bucket players into tiers under DK scoring before computing playoff-similarity features.
3. Apply per-position scaling on "round X position spend" (calibrated from UD vs DK projection-distribution comparisons).
4. Build-paths and stacks transfer with no adjustment.

---

## Custom rankings + override hygiene

CSV upload (no in-tool grid for v1). Template:

```
player_id, name, team, pos, my_rank, my_tier, override_bonus, note
```

`override_bonus` is a percentage (-15 to +15). Applied in the recommender as an additive term to the EV score (Spec 05).

**Hygiene rules:**
- On upload, snapshot `adp_at_override_time` and `override_set_date` per player into `overrides` table.
- **Auto-decay** as ADP moves toward your view:
  ```
  decayed = original × max(0, 1 − adp_delta / decay_threshold(pick))
  decay_threshold(pick) = base × adp_stddev(pick) / adp_stddev(48)
  ```
  So a round-2 override decays on small ADP moves; a round-9 override needs a bigger move.
- **Hard cap** at ±15% combined bonus per player.
- **Stale-override review panel** — weekly: lists overrides where ADP has moved >X spots toward your view, or player is hurt/cut/inactive, or override is >60 days old and ADP unchanged. You hit Keep / Reduce / Remove.

---

## Performance budget

| Component | Budget |
|---|---|
| Hierarchical Bayesian fit (full) | ~2–5 hours nightly. Acceptable. |
| Hierarchical Bayesian inference (per player, cached) | < 1 ms |
| MAPIE conformal | one-time per fit, ~minutes |
| Cox PH fit | < 1 minute |
| Sobol QMC sample of 5K paths | < 50 ms |
| Override decay recompute | < 100 ms for full table |

Slow-path components run nightly. The runtime path during a draft only reads cached projections + posterior samples.
