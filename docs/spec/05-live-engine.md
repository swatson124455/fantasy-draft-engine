# Spec 05 — Live Draft Engine

The runtime path: every observed pick triggers a recompute. Output is a top-N candidates list with EV scores, reasoning, and chips, delivered to the side panel within 250 ms.

---

## The objective: convex tournament utility / CVaR

**The single biggest change vs. naive recommenders.**

Most tools optimize *average projected points*. BBM-style tournaments are top-heavy (winner takes ~25% of the prize pool, top 0.1% take ~80%). The right objective is prize-curve-weighted percentile rank, not mean. Anchor: Hunter-Vielma-Zaman 2016 (arXiv:1604.01455), Haugh-Singal 2021 (Management Science).

**Method:**

1. Sample N team-outcome scenarios via Sobol QMC (Spec 02).
2. For each scenario, score the team and compare to a simulated field of opponents (Spec 03 archetype-driven).
3. Apply the payout curve to the team's percentile rank.
4. CVaR aggregator: `E[utility | utility > VaR_α]` with α = 0.05. Captures upside concentration without rewarding pure variance.

**Payout curve is a config table, not hard-coded.** `config/payouts/bbm6.toml`, `bbm5.toml`, `dk_best_ball.toml`. Switch per contest. BBM6 ≠ BBM5 ≠ DK best ball.

**EV score per candidate player p:**

```
score(p) = CVaR_α(payout(percentile_rank(team ∪ {p}))) − CVaR_α(payout(percentile_rank(team)))
        + flex_bonus(p, pick, roster)
        + override(p)                          # capped, ADP-decayed
        − reach_penalty(p, pick)               # round-aware
```

The CVaR delta is the principled core. The other terms are tactical adjustments.

---

## Round-aware reach tolerance (preserved from v1)

ADP variance grows with pick number. A 5-pick reach in R2 is meaningful; a 5-pick reach in R13 is noise.

```
adp_stddev(pick) ≈ 1.5 + 0.15 · pick                        # default; recalibrated nightly
reach_delta      = current_pick − player_adp                # +ve = reaching
reach_penalty    = (reach_delta / adp_stddev(pick))² · w_reach
```

Same numeric reach has high penalty early and ~zero penalty late.

`adp_stddev(pick)` is recalibrated nightly from the event-sourced draft log (Spec 01). Curve stored as a Parquet artifact and consumed by the recommender + Spec 02 sim layer + Spec 02 override decay threshold.

### Late-round flex bonus

When `adp_stddev(pick) > flex_threshold` (≈ R7+), the engine *unlocks* an additive bonus for picks that:

- Complete a viable stack (per Spec 03 signature catalog).
- Fill a thin position the roster needs.
- Pair with high-co-occurrence partners from the portfolio Leiden communities (Spec 04).
- Match an `live_clv_confirmed` finding from the diagnostic engine (Spec 04).

Late-round "reaches" that lock in a stack or roster gap become *recommendations*, not flagged reaches.

---

## Hierarchical Bayesian opponent archetype model

**Each of 11 opponents** gets an independent Dirichlet posterior over K=6 archetypes:

1. Zero-RB
2. Hero-RB
3. Robust-RB
4. Late-Round-QB
5. Stack-Heavy
6. ADP-Mechanical

**Pre-training (offline):** Fit per-archetype β coefficients on scraped Underdog draft data. Each archetype is a multinomial logit over (player, pick) given (round, position-already-rostered, available-pool). β coefficients stored in `model_artifacts` (Spec 01).

**Runtime (in-draft):** Closed-form Dirichlet-multinomial updates after each observed pick. ~10 lines of NumPy. No MCMC.

```python
# pseudo
posterior_alpha[opp][archetype] += likelihood(observed_pick | archetype, β)
```

Used as:
- Determinization distribution for ISMCTS.
- Posterior weights for Restricted Nash Response selection.
- Cluster exposure prediction (Spec 06: "the field is heavy on Stack-Heavy this draft").

**Why this matters:** No competitor ships opponent modeling. Every public tool treats the field as ADP-mechanical. With 6 archetypes and Bayesian updating, we anticipate runs (Stack-Heavy at slot 4 → expect pass-catcher run before our R3 pick).

---

## Restricted Nash Response counter-strategies

Anchor: Johanson-Zinkevich-Bowling 2007.

**Pre-compute one safe-exploitative strategy per archetype offline.** The strategy is a state-action policy: given roster state, what's the "best response" assuming the field is the given archetype?

Computed by self-play simulation: each archetype as opponent, learn the response that maximizes CVaR utility while staying within ε of Nash (so we don't get exploited if the field shifts mid-draft).

**At runtime:** posterior-weight or UCB-select across the K=6 strategies based on the opponent posterior:

```
recommendation = argmax_p Σ_archetype  P(field=archetype) · RNR_strategy[archetype](state, p)
```

Or UCB if exploration vs exploitation matters (early in draft season when archetype priors are weak, UCB explores; late season exploits).

---

## Information Set MCTS lookahead

Anchor: Cowling-Powley-Whitehouse 2012. ~400–600 LOC of NumPy.

**Why ISMCTS not PIMC:** Perfect-Information Monte Carlo (PIMC) silently breaks under "strategy fusion" — assumes future picks happen with full information when in reality picks are made under uncertainty. ISMCTS handles imperfect-information correctly.

**Method:**
1. At decision point, sample determinization from opponent posterior (each opponent's archetype is sampled from the Dirichlet, future picks rolled out under that archetype's policy).
2. UCT search over the resulting tree.
3. 500–2000 sims per decision under 100 ms with a learned policy prior.

**Policy prior:** Mixture of (a) ADP-prior, (b) per-archetype RNR strategy weighted by current opponent posterior, (c) a small uniform smoothing term.

**What this enables:** "If Higgins gets pushed past my next pick, I fall to bring-back stack." Can't reason this way without lookahead.

**Performance:** 100 ms budget at 2000 sims requires policy prior to skip terminal rollouts. Cold-start (no policy prior) hits 200–300 ms with 500 sims. Acceptable but degraded; flagged in side panel as "fast-path" until prior is learned.

**Sequencing dependency:** Archetype model and RNR strategies must be in place before ISMCTS lights up. See MILESTONES Phase 6.

---

## Custom rules

Beyond the override CSV (Spec 02), live engine respects:

- **Locks**: must-draft if available at any pick.
- **Excludes**: never draft.
- **Caps**: don't exceed N% portfolio exposure (caught at recommendation time, not after).
- **Stack rules**: e.g., "always pair QB with at least one same-team WR" — adds a penalty term to non-stacking picks.
- **Contrarian rules**: e.g., "if field exposure on player > X%, fade by Y%."

Stored in `config/rules.toml`. Stale rules surfaced for review monthly.

---

## Scoring profile-aware execution

The active draft's `scoring_profile` (UD ½-PPR or DK full-PPR) drives:
- Which `projections` rows are pulled.
- Which payout curve is loaded (UD BBM6 vs DK best ball).
- Which tier breaks apply.
- How the cross-format adjustment from Spec 02 transforms playoff-DB findings before they nudge the flex_bonus.

Side panel always shows a profile badge: "UD ½-PPR" or "DK PPR".

---

## Drafted-player removal

Picked players removed from candidate pool within ~200 ms via the event-sourced draft log (Spec 01). Idempotent — a single pick event triggers a single recompute even if the MutationObserver fires twice.

Hash key: `sha256(draft_id || pick_number)`.

---

## Performance budget (runtime path)

| Stage | Budget | Notes |
|---|---|---|
| Pick observed → background worker | < 20 ms | Chrome MV3 |
| Background → backend WebSocket | < 10 ms | localhost |
| Update opponent posterior | < 5 ms | closed-form |
| Sobol QMC sample 5K paths | < 50 ms | cached posterior |
| ISMCTS lookahead | < 100 ms | with policy prior |
| Score top-N candidates | < 30 ms | vectorized |
| Pack response, emit to extension | < 10 ms | |
| Extension render side panel | < 25 ms | React reconcile |
| **Total** | **< 250 ms** | |

If the budget is breached, fast-path mode disables ISMCTS and falls back to score-only. Telemetry tracks fast-path usage; if frequent, profile and fix.

---

## Output payload (to side panel)

```json
{
  "draft_id": "ud-12345",
  "scoring_profile": "ud_half_ppr",
  "current_pick": 48,
  "pick_in_round": 4,
  "round": 5,
  "adp_stddev": 8.7,
  "candidates": [
    {
      "player_id": 1234,
      "name": "Player Name",
      "pos": "WR",
      "team": "BUF",
      "score": 42.3,
      "ev_delta_cvar": 31.0,
      "flex_bonus": 8.2,
      "override_bonus": 3.1,
      "reach_penalty": 0.0,
      "playoff_prob_delta": 0.041,
      "stack_signature_added": "QB-WR-WR (size 3)",
      "availability_pct_next_pick": 0.34,
      "ismcts_visit_share": 0.42,
      "reasoning": "Completes BUF passing stack. Late-season schedule strong (W14-17). Field underweight on BUF stacks (Leverage = +6.3%).",
      "chips": ["stack", "late-season", "leverage"]
    },
    ...
  ],
  "in_progress_roster": {
    "stack_signatures": ["QB-WR (size 2)"],
    "playoff_similarity": 0.18,
    "build_path_label": "Hero-RB",
    "closest_match": {
      "playoff_team_id": 2143,
      "season": 2024,
      "advance_round": "Finalist"
    }
  },
  "fast_path": false
}
```

The side panel renders this JSON. All logic lives in the backend. The extension is a thin observer.
