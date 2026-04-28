# Spec 04 — Diagnostic Engine

Mines historical BBM rosters (Underdog, Spec 00 audited) plus your own draft history for correlations and outliers. Pure signal; no prescriptions. The diagnostic engine never tells you what to draft — it surfaces patterns the live engine and you can use.

---

## The unifying primitive: calibrated OOF residuals

**Library:** LightGBM, scikit-learn (isotonic), captum (TreeSHAP via shap).

**Method:**
1. Train LightGBM `P̂(advance | features)` with 5-fold OOF predictions.
2. Apply isotonic calibration on OOF probabilities (uniform calibration target).
3. Compute residual `r_i = y_i − P̂_i` for every roster.

**Why it's the primitive:** Every other technique in this engine consumes or produces signals around residuals. Pure advance rates are confounded by ADP, slot, era. The residual strips out what an average roster from that draft state would have done; what's left is the pattern.

**Pre-registration is non-negotiable.** Feature set is committed to `config/features.toml` *before* residuals are inspected. Adding features after looking at residuals is p-hacking. The git commit SHA of the feature config is stored in `features_registered` (Spec 01).

**TreeSHAP attribution.** On top decile and bottom decile of residuals, decompose into per-feature contributions. This is the input to subgroup discovery (below). Top-decile SHAP says "what made these rosters over-perform"; bottom-decile says "what dragged these down."

---

## Pick capital weighting

**Method:** Empirical pick value chart fit from BBM data. Best-ball analog of a Jimmy Johnson chart, fit to outcomes not assumed.

For each pick number `p`, compute the average `r_i` contribution attributable to that pick (via TreeSHAP per-pick decomposition). Smooth the curve. The resulting `pick_value(p)` is a config artifact (`config/pick_value_chart.parquet`).

**Used for:**
- Feature weight in roster-summary features.
- Denominator in value-extracted-per-pick diagnostics.
- Capital-controlled variants of every Spec 03 advance-rate curve.

Without this, "RB-heavy top 5" lumps slot 1.01+2.12+3.01+4.12+5.01 with slot 1.12+2.01+3.12+4.01+5.12 — different bets. Pick-value weighting normalizes.

Surfaced as a config table you can inspect. Re-fit annually after each season's data lands.

---

## Correlation mining

### FP-Growth + lift / conviction / leverage

**Library:** mlxtend.

Fast frequent-itemset mining over rosters as itemsets-of-players. Output: every player itemset with support ≥ floor and lift ≥ τ.

For each rule `A → advance`:
- Fisher-exact p-value.
- BH-FDR adjusted q-value.
- Bootstrap-BCa 95% CI on lift.
- Within-slot conditional rate alongside marginal lift (catches ADP-confounded findings).

Support floors per cardinality (lower for higher cardinality):
- Pairs: support ≥ 100 rosters.
- Triples: support ≥ 50.
- 4-itemsets: support ≥ 25.
- 5+: support ≥ 15, with mandatory permutation null.

### Mutual information sweeps

Player → advance, conditional on slot. Catches non-linear associations that linear lift misses. Computed via histogram-binned `sklearn.feature_selection.mutual_info_classif` on dichotomized roster-membership features.

### Graphical lasso with nonparanormal SKEPTIC

**Library:** `sklearn.covariance.GraphicalLassoCV`.

Method: Apply Kendall's tau transform to roster-membership matrix to handle non-Gaussian co-occurrence; then graphical lasso for partial correlations.

Edges that survive partial correlation are *genuine* co-draft preferences — not artifacts of "both players are popular at slot 5." This isolates the human stacking intent from the ADP confound.

### Leiden community detection

**Library:** igraph + leidenalg.

PMI-weighted player co-occurrence graph → Leiden communities. Consensus clustering across 50 runs (different random seeds), label communities by stable membership.

Output table: `(community_id, members[], advance_rate_in_community_5plus_members, q_value)`.

Sample finding format:

> *"Rosters with ≥5 players from Cluster 7 (PHI passing game + LAR receiving) advance at 21.8% vs 16.7% baseline (q=0.018, n=2,847)."*

---

## Subgroup discovery

**Library:** pysubgroup.

Beam search depth 3 on the residual column. Run twice:
- Target = `r_i > 0` for positive outliers.
- Target = `r_i < 0` for negative outliers.

**P-values via permutation null** (analytical nulls are biased under selection). For every reported subgroup: shuffle the outcome label 1000× and re-mine; report the empirical p-value. Then BH-FDR.

Output format:

> *"Rosters matching `[stack_count ≥ 2 AND TE_round ≤ 7 AND no_QB_before_round_10]` advance at 2.3× baseline given structural features, n=8400, q=0.003."*

---

## Causal forest

**Library:** econml.CausalForestDML.

**Treatments (dichotomized):**
- has-double-stack
- has-bring-back
- RB-heavy-top-5
- late-onset-elite-TE
- stack-size contrasts (3 vs 2, 4 vs 3, 5+ vs 4)
- position-composition contrasts (Spec 03)

**Controls:** drafter-skill where available (your own pre-existing track-record CLV is one input).

**Procedure:**
1. Trim extreme propensities (drop |e(x) − 0.5| > 0.45).
2. Fit CausalForestDML.
3. Athey-Wager calibration test — if rejected, surface as caveat in the finding.
4. τ̂(x) heatmaps over context features (team implied total, draft slot, season, playoff schedule strength).

**Output format:**

> *"Bring-back conditional on QB-WR-WR-TE at size 4 produces τ̂ = +1.8 advance pp (95% CI [0.4, 3.2], n_treated=1,420, n_control=1,180, calibration q=0.62)."*

---

## Density × outcome (backlog)

Bayesian GMM first (seconds, ~80% of signal). Conditional Neural Spline Flow only if GMM tail mass is mis-calibrated.

The 2×2 of `log p̂(roster) ∈ {high, low}` × `outcome ∈ {advance, brick}` partitions rosters into:
- typical-baseline
- typical-bricked (negative outlier)
- rare-advanced (positive outlier)
- rare-bricked (lottery loss)

Materially distinct from the residual lens; both kept. Backlog because it replicates ~80% of the signal already captured by residuals + subgroup discovery, with diminishing returns until those are exhausted.

---

## Counterfactual nearest-neighbors (backlog)

β-TCVAE on engineered features → 8–16-dim latents → faiss-cpu index. For any in-progress roster: encode, find nearest historical finalists and nearest historical bricks, decode interpolations to identify the 1–3 picks that differ.

DiCE for ADP-feasible counterfactual swaps.

In backlog because it requires a working β-TCVAE (PyTorch, deferred) and the marginal interpretability gain over Leiden communities + subgroup discovery is modest until you have the latents.

---

## What gets reported

Standardized signal-feed format (Spec 08). Every diagnostic output looks the same:

```
{
  "finding_id": "fp-growth-2026-04-28-00012",
  "kind": "association_rule",
  "rule": "QB(BUF) ∧ WR(BUF) ∧ TE(BUF) → advance",
  "lift": 1.34,
  "support": 412,
  "fisher_p": 0.0008,
  "q_value_bh": 0.014,
  "bootstrap_ci": [1.18, 1.49],
  "within_slot_conditional": 1.21,
  "permutation_null_p": 0.011,
  "garden_of_forking_paths_p": 0.034,
  "status": "experimental",
  "model_version_id": "fp-growth-v3"
}
```

`status` transitions: `experimental` → `validated` (passes held-out BBM VI) → `live_clv_confirmed` (passes Spec 08 CLV gate). Only `live_clv_confirmed` findings are allowed to nudge the live recommender's `flex_bonus`.

---

## How findings flow to the live engine

1. Diagnostic engine emits finding to `findings` table with `status='experimental'`.
2. Held-out BBM VI evaluation (annual, sacred) bumps to `validated` if it survives.
3. Live drafts run with the validated finding contributing as a *small* nudge to `flex_bonus`.
4. CLV log (Spec 08) tracks per-pick edge attributable to the finding.
5. After N drafts of positive CLV with statistical significance, the finding bumps to `live_clv_confirmed` and gets full weight.

This is the data flywheel. Every season of CLV refines projections, opponent archetypes, stack tail dependencies, position-signature lifts, and pick-value chart simultaneously.
