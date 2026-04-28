# Spec 09 — Deferred and Skip

Two sections. **Deferred** = good ideas, not now. **Skip** = won't help, won't ship.

---

## Deferred (post-launch backlog)

Each item has a "what triggers consideration" line so we know when to revisit.

### Density × outcome 2×2 partition
Bayesian GMM first (seconds, ~80% of signal). Conditional Neural Spline Flow only if GMM tail mass is mis-calibrated. The 2×2 of `log p̂(roster) ∈ {high, low}` × `outcome ∈ {advance, brick}` partitions rosters into typical-baseline / typical-bricked / rare-advanced / rare-bricked.
**Trigger:** Spec 04 residual + subgroup signals plateau, looking for orthogonal lens.

### β-TCVAE counterfactual nearest-neighbor
β-TCVAE on engineered features → 8–16-dim latents → faiss-cpu index. For any in-progress roster, find nearest historical finalists and bricks; decode interpolations to identify the 1–3 differentiating picks.
**Trigger:** Need to install PyTorch (deferred from day 1). Do this when latent-space search is the marginal upgrade over Leiden communities.

### DiCE feasible counterfactual swaps
Given an in-progress roster, find the minimum ADP-feasible swap that increases playoff probability. Pairs with β-TCVAE.
**Trigger:** β-TCVAE installed.

### Model-X knockoffs (MRC, not SDP)
FDR-controlled feature selection beyond BH-FDR. Catches false-positive features the marginal correlation tests miss.
**Trigger:** Spec 04 feature set grows beyond ~200 features and FDR contamination becomes plausible. May never trigger at personal scale.

### Time-varying graphical lasso
Cross-season drift in player co-occurrence networks. Detect when stack patterns are shifting structurally vs. season-to-season noise.
**Trigger:** ≥ 3 seasons of live drafting + diagnostic data showing potential drift.

### Set Transformer roster encoder
Replaces hand-engineered roster features with a learned permutation-invariant encoder.
**Trigger:** ≥ 10K of your own drafts accumulated; hand-engineered features show plateau in held-out validation.

### Stacking ensemble with NNLS
Combine baseline projections + Sleeper + your overrides via non-negative least squares meta-learner. Per-player optimal weights.
**Trigger:** Multiple projection sources active with non-trivial disagreement, baseline blend underperforming on held-out.

### Contextual bandits for override discovery
Auto-explore the override-bonus space to discover players the heuristics miss. Pulls arms = candidate overrides; rewards = realized CLV.
**Trigger:** ≥ 1 season of live CLV data with override hygiene rules in place.

### Mid-season Kalman / conjugate updating
Live posterior updating on Bayesian projections as games are played. Replaces nightly full re-fit.
**Trigger:** In-season tools become a focus (out of v1 scope).

### ECOD / COPOD geometric anomaly
Crossed with outcome labels for anomaly-aware advance prediction. Faster than One-Class SVM, scales to full data.
**Trigger:** Anomaly-detection becomes a productive lens, current methods plateau.

### SkopeRules / RuleFit
Human-readable rule extraction on the residual column. Output: "if `[stack_count ≥ 3 AND no_QB_round_1 AND TE_round ≤ 6]` then advance lift = 1.42."
**Trigger:** Subgroup discovery findings need cleaner rule presentation for dashboard.

### Bayesian network structure learning
Discover causal structure on roster-summary features. Heavy compute, slow convergence.
**Trigger:** Possibly never at personal scale. Reconsider if ≥ 5 seasons of live data accumulate.

### Causal transformer for next-pick prediction
Replaces hand-authored archetypes with a learned model when ≥ 10K drafts are observed.
**Trigger:** When your own draft data + opponents' observed picks reach ≥ 10K total drafts. Measure when archetype model's prediction error plateaus.

### Multi-source projections feed integration
FantasyPros consensus, RotoWire, Establish The Run if a free / low-cost feed becomes available.
**Trigger:** Free public feeds become reliable; or partnership opportunity.

### Auction draft / dynasty / keeper support
Different valuation models. Different roster construction.
**Trigger:** User decides to enter these formats. Currently out of scope.

### Multi-sport (NBA, MLB, PGA, CFB)
Underdog Battle Royale weekly snake drafts; NBA best ball.
**Trigger:** User decides to play other sports at scale.

### TE-premium scoring profile
Some leagues use 1.5 PPR for TEs.
**Trigger:** User enters such a league. Architecture supports it; just a new row in `scoring_profiles`.

### Mobile / Underdog mobile app support
Currently impossible without rooted device or jailbreak; Chrome MV3 doesn't ship to mobile.
**Trigger:** User changes drafting workflow to require mobile.

### In-tool spreadsheet rankings editor
CSV-only is fine for v1. Spreadsheet grid (sortable, drag-to-reorder, tier coloring) is a UX upgrade.
**Trigger:** CSV workflow becomes painful in practice.

### Weekly W1–17 projections
Currently seasonal only. Weekly enables proper W15–17 playoff scoring.
**Trigger:** Phase 2.6 hierarchical model is stable; weekly is a downstream extension.

### Weather-adjusted correlations
W15–17 game environment (cold, wind, snow) shifts pass-vs-run game scripts.
**Trigger:** Late-season correlation findings show weather-conditional patterns worth modeling.

### DK historical playoff DB
DK-native winning team data, parallel to UD playoff DB. Currently we use UD scoring-adjusted to DK.
**Trigger:** UD model validated on its own ground first; DK data acquisition feasible.

---

## Skip permanently

Each with a one-line rationale. No "we'll see."

- **NumPyro / JAX libraries on Windows.** PyMC + nutpie via conda-forge solves the same problem without the Windows toolchain pain.
- **RealNVP / Glow / MAF.** Conditional NSF strictly dominates for tabular/structured data of this dimensionality.
- **Score-based diffusion models.** Overkill for ~150-D vectors. Massive infra cost for no gain over GMM/flow.
- **Energy-based models.** Training instability, no diagnostic upside for our problem class.
- **IWAE.** No diagnostic upside over β-TCVAE.
- **DeepSet / Set-Transformer GANs.** No log-likelihood, can't be used for density × outcome lens.
- **One-Class SVM, LOF on full 3M data.** O(N²), doesn't scale. ECOD/COPOD if needed.
- **Apriori.** FP-Growth strictly dominates on speed; no reason to use Apriori.
- **MIC (maximal information coefficient).** Inferior to mutual information per Kinney-Atwal 2014. Use MI.
- **graph-tool.** No Windows wheels. igraph + leidenalg covers our needs.
- **Full Deep CFR / ReBeL.** Overkill for 12-player general-sum draft. ISMCTS + RNR sufficient.
- **AlphaZero with policy/value net.** Premature. Use until ISMCTS plateaus, then revisit. Even then, justify against compute cost.
- **Mobile app.** Out of scope per architecture; MV3 doesn't ship to mobile.
- **Multi-user / auth / billing.** Single-user tool. Adding auth adds attack surface and zero value.
- **DFS lineup optimization.** ETR Solver does this. We focus on draft.
- **In-season weekly lineup optimizer.** Best ball auto-sets lineups; nothing to optimize. (Battle Royale would benefit, but Battle Royale is out of scope.)
- **Dynasty.** Different valuation model entirely. Not in scope.
- **Heavy Cox PH with deep covariate interactions.** Per Spec 02 design decision: kept lightweight, late-season boost layer handles the upside catalysts that make the survival model worth using.
- **Chrome Web Store publishing.** Single-user side-loaded extension. Web Store adds review overhead, signing, paid developer account. None of which we need.
- **Public release / SaaS / paid tiers.** Out of scope by user constraint. Adding any of this adds product, marketing, support, and TOS surface for zero value to the single user.

---

## How items move from this spec into the build

A deferred item gets its own MILESTONES.md phase entry only after:

1. The trigger condition is met.
2. The user explicitly says "do this now."
3. A spec/NN file is written or updated to capture details.
4. Items it depends on are confirmed in place.

A skip item never moves. If circumstances change such that a skipped item makes sense, reopen the discussion as a separate decision; don't re-litigate via this spec.
