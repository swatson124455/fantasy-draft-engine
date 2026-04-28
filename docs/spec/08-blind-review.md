# Spec 08 — Blind-Review Discipline

The non-negotiables that prevent the diagnostic engine from becoming a p-hacking machine.

Five rules. None are optional. None are "we'll add this later." If a finding hasn't passed each gate, it doesn't influence the live engine.

---

## 1. Held-out season is sacred

**Train on BBM I–V. Evaluate on BBM VI.** Patterns that survive the held-out evaluation are real; patterns that don't are overfit. Don't argue with the held-out result.

**Implementation:**
- Held-out partition is a separate Parquet directory `data/parquet/picks/season=BBM_VI_HELD_OUT/`.
- Touched only behind a CLI flag: `--use-sacred-set`.
- Every use is logged to `audit.log` with timestamp, git commit SHA, the analyst's intent ("annual cycle re-evaluation"), and the model_version_id.
- Touched **once per analysis cycle** (~monthly). If a finding fails the held-out evaluation, you cannot re-touch the held-out set after tweaking the method until the next cycle.

**When the season ends and BBM VI becomes "in distribution":**
- Move BBM VI to training partitions.
- Designate the new most-recent season (BBM VII) as the next held-out set.
- The discipline carries forward.

---

## 2. Pre-registered feature set in versioned config

**Feature set is committed to `config/features.toml` BEFORE residuals are inspected.**

Adding a feature after looking at residuals is p-hacking. Even if the feature is "obviously the right thing." Even if you swear you're not going to torture it. Don't.

**Implementation:**
- `config/features.toml` is the source of truth.
- Every change creates a new row in `features_registered` (Spec 01) with timestamp, git commit SHA, and a free-text justification.
- Diagnostic-engine runs are tagged with the `feature_set_version` they used.
- Findings produced by an outdated feature set are flagged in the dashboard.

**Updates:** Once per analysis cycle (~monthly). Justification field must explain *why* the feature is needed and *what would happen if you didn't add it* — forces explicit reasoning.

---

## 3. Standardized signal-feed format

**Every diagnostic output looks the same** regardless of method (FP-Growth, MI, subgroups, causal forest, vine copula).

The format from Spec 04:

```json
{
  "finding_id": "kind-modelversion-yyyymmdd-NNNN",
  "kind": "association_rule | subgroup | community | stack_signature | causal_treatment",
  "description": "human-readable",
  "metric_value": 1.34,
  "metric_kind": "lift | tau | mi | advance_rate_delta",
  "p_value": 0.0008,
  "q_value_bh": 0.014,
  "bootstrap_ci": [1.18, 1.49],
  "permutation_null_p": 0.011,
  "garden_of_forking_paths_p": 0.034,
  "n_support": 412,
  "feature_set_version": "v3.2",
  "model_version_id": "...",
  "trained_on": "BBM I-V",
  "evaluated_on": "BBM VI",
  "status": "experimental | validated | live_clv_confirmed",
  "first_seen": "2026-04-28",
  "last_validated": null
}
```

Why standardize: removes the temptation to cherry-pick presentation. A finding with `q_value_bh = 0.04` and `n_support = 25` looks identical to one with `q_value_bh = 0.04` and `n_support = 4000`. The reader does the comparison; the format doesn't sell the finding.

---

## 4. Garden-of-forking-paths permutation diagnostic

**For any "interesting" finding, automatic 100 permutation re-mines with different random splits / threshold / parameter values.**

Implementation:
- For each reported finding, the diagnostic engine permutes (a) the train/eval fold split, (b) the support floor by ±20%, (c) the lift threshold by ±20%, and re-runs the mining 100 times.
- Counts how many permutations produce *any* finding equally interesting at the same threshold.
- If 8% of permutations produce something equally interesting, the finding's credibility is exactly that bad — it's a 1-in-12 result, not a p=0.04 result.

The `garden_of_forking_paths_p` field in the standardized format captures this number.

A finding with `q_value_bh = 0.014` but `garden_of_forking_paths_p = 0.34` is publication-quality on the surface but fails the actual robustness test. Both metrics are surfaced; the dashboard sorts by `garden_of_forking_paths_p` ascending by default.

---

## 5. Live CLV is the only ungameable validation

**A pattern graduates from "interesting" to "I'm changing my drafting" only after producing measurable CLV in live drafts.**

Why: held-out BBM VI is one season. ~150K finalist rosters. Sounds like a lot, but a single season can be a regime aberration. Live CLV across 50, 100, 500 of *your* drafts in a new season is the only test that can't be gamed by the analyst.

**Status transitions:**

| Status | Conditions |
|---|---|
| `experimental` | Just emitted. Reported but not influencing live engine. |
| `validated` | Survived held-out BBM VI (q < 0.05 on held-out evaluation). Allowed to nudge `flex_bonus` at *small* weight (e.g., 25% of full magnitude). |
| `live_clv_confirmed` | After ≥ N drafts of live CLV with cumulative significance (e.g., t-stat > 2 on per-pick CLV attributable to the finding). Full-weight contribution to live engine. |

N is calibrated per kind of finding. Stack-signature findings: ~50 drafts. Position-pair findings: ~100 drafts. Player-specific findings: ~30 drafts (since they're per-player and your portfolio quickly accumulates exposure).

**The CLV log** (Spec 01: `clv_log` table):
- Per-pick CLV attribution (your_pick_adp − market_adp_at_pick).
- Tied to `finding_id` if your pick was nudged by a finding.
- Aggregated: per-finding cumulative CLV with confidence intervals.

A finding that's `validated` but accumulates negative CLV across N drafts is *demoted* back to `experimental` and removed from the live engine until it earns its way back.

---

## How this interacts with the live engine

The live engine's `flex_bonus` is the only place findings can nudge recommendations. The weight applied to a finding scales with status:

```
flex_bonus_contribution(finding) = finding.metric_value × status_weight(finding.status)

status_weight:
  experimental         → 0.0    (reported in dashboard, no influence)
  validated            → 0.25   (small nudge)
  live_clv_confirmed   → 1.0    (full weight)
  demoted              → 0.0    (was confirmed, now isn't)
```

This is the actual moat. Competitors copy features by inspection. They cannot copy the discipline that prevents the model from chasing noise.

---

## Audit trail

`audit.log` is append-only. Every:
- Held-out set access.
- Feature config change.
- Finding status transition.
- Manual override of a finding's status.

is logged with timestamp, git commit SHA, user (always the same one — but recorded anyway), and free-text reason. Reviewable monthly.

---

## What this discipline rejects

- "I have a hunch this is real even though q=0.21" — no. The hunch is data; treat it like data and pre-register it.
- "Let me just slightly tweak the support floor and see what happens" — no. That's the garden of forking paths.
- "BBM VI confirmed it" without showing the held-out access logs — no. The audit log is the proof.
- "It's only an `experimental` finding so it doesn't matter what we report" — no. Reported findings shape your drafting *cognition* even when they don't shape the recommender. Discipline applies at all status levels.
