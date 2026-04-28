# Spec 03 — Stacks and Position-Signature Taxonomy

Stacks are not a single phenomenon. They are two parallel phenomena (QB-anchored vs non-QB) with sub-structures defined by *position composition*, not just cardinality. This spec lays out the feature engineering and the diagnostic outputs.

---

## Two-track architecture

QB-anchored and non-QB stacks treated as separate phenomena throughout. Findings about one don't pollute findings about the other.

### Track A — QB-anchored stacks

Definition: Primary QB on the roster + ≥1 same-team pass-catcher.

Per-cardinality features:
```
qb_stack_size_2        bool    # QB + 1 PC
qb_stack_size_3        bool    # QB + 2 PCs
qb_stack_size_4        bool    # QB + 3 PCs
qb_stack_size_5_plus   bool    # QB + 4+ PCs
```

Sub-classification flags (orthogonal to size):
```
has_bring_back         bool    # opposing pass-catcher in W15-17 game
has_passcatching_rb    bool    # RB with target share ≥ threshold same team
has_secondary_qb       bool    # backup QB also rostered (rare)
```

### Track B — Non-QB stacks

Definition: ≥2 same-team players, no primary QB on roster from that team.

Per-cardinality features:
```
nonqb_stack_size_2        bool
nonqb_stack_size_3        bool
nonqb_stack_size_4_plus   bool
```

Sub-classifications by composition: WR+WR, WR+TE, RB+WR (pass-script), RB+TE.

---

## Empirical stack definition layer

Stack indicators are NOT hard-coded by position. They are **empirically defined via the vine-copula tail-dependence matrix** (Spec 04 section).

For each pair of players (intra-team), we estimate λ_U (upper-tail dependence) and λ_L (lower-tail dependence) using `pyvinecopulib`. Pairs with statistically significant λ_U > τ are stackable, regardless of position. This catches:
- TE2-as-stack-with-QB cases when target share concentrates,
- RB-WR stacks in pass-script teams,
- WR3 lottery tickets that co-spike with WR1 in shootouts.

The position-based features above remain — they're how humans think about stacks and they're what the side panel surfaces. The vine-copula derived features supplement them and feed the diagnostic engine.

---

## Position-signature taxonomy

Position composition is a first-class variable, equal in importance to cardinality. Two stacks of size 3 with different positions are different bets.

### Size 2

```
QB-WR1                  QB + the team's #1 WR (top-2 in projected target share)
QB-WR2plus              QB + a non-WR1 WR
QB-TE                   QB + TE
QB-RB-passcatching      QB + RB with target share ≥ threshold
WR-WR-sameteam          two same-team WRs, no QB
WR-TE-sameteam          WR + TE, no QB
RB-WR-sameteam          RB + WR (pass-script)
RB-TE-sameteam          RB + TE (RZ correlation)
```
Plus game-stack variants (cross-team in same game).

### Size 3

```
QB-WR-WR                pure passing stack
QB-WR-TE                passing + RZ
QB-WR-RB                passing + check-down
QB-TE-RB                RZ + check-down
QB-WR-bringback         QB + WR + opposing PC in same W15-17 game
```
Plus non-QB equivalents.

### Size 4

```
QB-WR-WR-TE             passing tree with RZ
QB-WR-WR-RB             passing tree with check-down
QB-WR-WR-bringback      passing tree with opposing PC
QB-WR-TE-bringback
QB-WR-RB-bringback
```

### Size 5+

```
Full passing trees with/without bring-back, with/without RB.
Aggregated as "5_plus" beyond this for sample-size reasons.
```

---

## Per-signature diagnostic outputs

For every signature defined above, the diagnostic engine produces:

1. **Empirical advance rate** with bootstrap-BCa CIs and BH-FDR q-values.
2. **Capital-controlled residual advance rate** — same calculation, with pick-value chart (Spec 04) controlling for total draft capital spent on the stack. Catches "RB-heavy top 3" being apples-to-oranges between slot 1.01 and slot 1.12.
3. **λ_U** (composite upside co-spike) from the vine-copula matrix.
4. **λ_L** (composite downside co-bust).
5. **Tail asymmetry** = λ_U − λ_L.
6. **Conditional advance rates** sliced by:
   - Team implied total
   - Playoff schedule strength
   - Game-script projection (pass-heavy vs run-heavy)

---

## Position-pair lift table

Separate from full signatures, a flat table of building blocks:

| Pair | Sub-tier filter | Lift vs marginal | n |
|---|---|---|---|
| QB+WR | by target-share tier of WR (1, 2, 3+) | | |
| QB+TE | by RZ-share tier | | |
| QB+RB | by target-share tier of RB | | |
| WR+WR same-team | | | |
| WR+TE same-team | | | |
| RB+WR same-team | | | |
| RB+TE same-team | | | |

Surfaces which position pairs do the heavy lifting inside any composite stack.

---

## "How much is too much" diagnostic

Per stack track (A and B), three artifacts:

### 1. Marginal advance-rate curve by cardinality

Empirical advance / finalist / winner rates as a function of stack size. With:
- Bootstrap-BCa confidence bands.
- BH-FDR significance flags per size step.
- Two versions: raw curve and capital-controlled curve.

### 2. Inversion detection

Isotonic regression p-value tests for:
- Monotone increasing (advance rate keeps rising with size).
- Concave plateau (rises then flattens).
- Inverted U (rises then falls).

Output verdict per track per archetype:

> *"QB-anchored stacks peak at size 3 (q=0.002); size 4 shows no marginal benefit (q=0.41); size 5+ shows significant decline (q=0.03, n=4.2k)."*

### 3. Conditional inversion

Causal forest τ̂(x) heatmaps with stack-size as treatment (Spec 04). x = team implied total, draft slot, season, playoff schedule strength.

Surfaces:

> *"Size-4 net positive when implied total ≥ 27 (τ̂ = +1.4 advance pp), net negative when ≤ 23 (τ̂ = −2.1 advance pp)."*

### 4. Tail-asymmetry diagnostic

λ_U, λ_L, and asymmetry plotted against stack size. Stacks where adding a 4th/5th player drives λ_L (co-bust) up faster than λ_U (co-spike) is the *mechanism* behind inversion. λ_L vs stack-size curves shown alongside advance-rate curves so you see *why* large stacks hurt when they hurt.

---

## Cross-track stack interaction

Three combined views:

1. **Total stacked players curve** — joint marginal of `qb_stack_size + nonqb_stack_size`.
2. **Substitution vs. complement** — 2D heatmap of QB-stack-size × nonQB-stack-size advance rates. Are QB and non-QB stacks substitutes (more of one means less benefit from the other) or complements?
3. **Concentration vs. distribution** — single 5-stack vs. 3+2 split vs. 2+2+pair. Same total stacked players, different structures, different advance rates expected.

---

## Causal-forest treatments specific to stack composition

Spec 04 defines the causal-forest infrastructure. The treatments specifically addressing stack composition are:

```
QB-WR-WR vs QB-WR-TE at size 3, conditional on team implied total
QB-TE-RB vs QB-WR-RB at size 3
4-stack with TE vs 4-stack without TE
bring-back vs no bring-back conditional on stack composition
```

Each yields a τ̂(x) heatmap with drafter-skill controls and propensity trimming.

---

## What goes in the side panel

The side panel does NOT display this entire taxonomy live. It surfaces:

1. **Current stack signatures on the in-progress roster** — small chips: "QB-WR-WR (size 3)" and "WR-TE (size 2 non-QB)".
2. **Per-candidate stack contribution** — clicking a candidate shows: "Adds: QB-WR-WR-TE (size 4) at size-4 advance rate 18.3% (q=0.04 vs marginal 16.7%)".
3. **Inversion warning** — if completing a 5th-player stack moves you into the empirically-declining region, a yellow chip surfaces.

The full taxonomy lives in the diagnostic dashboard and in nightly reports.
