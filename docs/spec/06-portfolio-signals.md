# Spec 06 — Portfolio Signals

You construct the portfolio. The tool surfaces signals. No auto-construction, no prescriptive advice.

The headline distinction: most public tools optimize per-team. A 3K-team grinder is doing portfolio optimization across teams. The North Star metric is not "average team score" — it's `P(at least one entry top-0.1%)`.

---

## North-Star metric

```
P(at least one entry top-0.1%)
```

Computed from the team-outcome MC sim (Spec 02). For each team in the portfolio, sample N points trajectories under correlated player outcomes; estimate per-team `P(top-0.1%)`; aggregate across portfolio with the inclusion-exclusion approximation (independent-trial assumption is a small lie but acceptable at scale).

Surfaced as the headline number on the portfolio dashboard.

---

## Leverage Score

Per player and per stack:

```
leverage_score(player) = (your_exposure_pct − field_exposure_pct) × ceiling(player)
```

Where:
- `your_exposure_pct`: fraction of your N teams that include the player.
- `field_exposure_pct`: estimated from public ADP and observed Underdog draft data.
- `ceiling`: posterior 95th-percentile points (from Spec 02 Bayesian + conformal).

Per-stack leverage uses the same formula on stack signatures (e.g., "BUF QB-WR-WR (size 3)") — your_exposure on the signature minus field_exposure of the signature, times the ceiling of the signature in MC sim.

Surfaces:
> *"You're leveraging BUF passing game at +6.3% (high ceiling); you're field-aligned on SF at +0.4% (no edge)."*

---

## Effective N (Herfindahl-Hirschman)

Your 3K entries don't behave like 3K independent fingerprints. If you draft variations of the same build path, two teams that share 14 of 18 picks are nearly the same bet.

```
HHI(portfolio) = Σ p_i²       # over distinct fingerprints
effective_N    = 1 / HHI
```

Where `p_i` is the share of teams in the portfolio that fall into fingerprint cluster `i`. Fingerprint clusters defined by:
- Build-path label (Zero-RB / Hero-RB / etc.)
- Stack signature primary (QB anchor team)
- Top-3 player overlap

Output: a single number. *"Your 3K entries behave like ~280 unique fingerprints."*

If `effective_N` is too low, the dashboard surfaces which build paths or QB anchors are over-concentrated.

---

## Cluster exposure

Per NFL team, per game, and per Leiden community (Spec 04):

```
cluster_exposure(cluster) = N_teams_with_≥k_members / N_total_teams
```

Surfaces:
> *"You're 75% exposed to PHI passing game across your portfolio — that's 4× field."*

Used to flag concentration risk. A high-EV concentration is fine — that's the point. But it should be a deliberate choice surfaced in the dashboard, not an accident.

---

## Information Ratio vs ADP-bot baseline

Single number for "am I beating chalk."

```
ADP_bot_portfolio = portfolio drafted by always taking the highest-ADP player available
IR = (your_portfolio_E_payout − ADP_bot_E_payout) / σ(your_portfolio_payout)
```

Both portfolios scored under the same Spec 05 CVaR utility on the same sim seeds. The σ in the denominator is the standard deviation of your portfolio's simulated payout — annualized vs draft season.

Surfaces as a single chart: rolling IR over your last 100 / 500 / 1000 drafts. If IR drifts negative, the live engine surfaces it as a risk flag.

---

## Portfolio CVaR (α = 0.05)

The proper tournament risk metric.

```
portfolio_CVaR_α = E[portfolio_payout | portfolio_payout < VaR_α]
```

Captures: in the worst 5% of seasons, how bad is it? A high-mean / high-CVaR portfolio is preferable to a high-mean / low-CVaR portfolio at equivalent expectation.

---

## ADP velocity, acceleration, volatility-adjusted edge

Per player, daily metrics:

```
adp_velocity(player)     = adp_today − adp_7d_ago
adp_acceleration(player) = adp_velocity_today − adp_velocity_7d_ago
adp_volatility(player)   = stddev(adp over last 30 days)

vol_adjusted_edge(player) = (your_rank − adp) / adp_volatility
```

A player whose ADP is drifting up (positive velocity) and accelerating (positive acceleration) is a market that's catching up — your override on this player should be inspected (Spec 02 hygiene rules).

A player whose `vol_adjusted_edge` is high means your edge survives the noise — high-conviction value. Low and shrinking means your edge is just ADP noise.

Surfaces in:
- Override review panel (Spec 02): "Hampton ADP velocity +6/wk; review override."
- Market Visualizer: per-round contrarian/chalk view weighted by `vol_adjusted_edge`.

---

## Market Visualizer (UI surface, preserved from v1)

Per-round distribution of your picks vs. market ADP, visualized as a scatter:

- X-axis: your pick number.
- Y-axis: player ADP at the time of pick.
- Diagonal = chalk; off-diagonal = contrarian or in-line reach.
- Color: contrarian (above diagonal), chalk (on diagonal), reach (below diagonal at high cost).
- Size: `vol_adjusted_edge` (bigger = more conviction).

Per-position deviation chart: are you systematically reaching on TEs, fading early QBs, etc.

Drill-down: click any data point to see the draft context, full roster, and what the recommender said at that pick.

Auto-recomputed nightly. Picks that *became* contrarian as ADP moved are flagged with an arrow indicator.

---

## Position-spend-by-round delta heatmap

Your average draft capital spent per position per round vs. the playoff-team average (from Spec 04 historical mining).

```
Y-axis: position (QB / RB / WR / TE)
X-axis: round
Cell value: your_avg_spend(pos, round) − playoff_avg_spend(pos, round)
```

Diverging colormap (red = under-indexed vs playoff, blue = over-indexed). Headline annotation: *"You're under-indexed on R5 WR vs playoff teams (-3.1 picks of capital). Consider re-evaluating R5 WR strategy."*

Capital weighting uses the empirical pick-value chart from Spec 04.

---

## Player-pair co-occurrence matrix

How often any two players appear together across your portfolio. Sortable, filterable by contest type and scoring profile.

Used for:
- Detecting unintentional concentration.
- Cross-referencing against Spec 04 Leiden communities — are your high-co-occurrence pairs from the same community as historical playoff teams?
- Spec 05 flex_bonus partner suggestions.

Heatmap UI; click a cell to see the list of teams.

---

## Build-path heatmap

Distribution of your roster constructions (Zero-RB / Hero-RB / Robust-RB / Late-Round-QB / Stack-Heavy / Hybrid / etc.) across the portfolio.

Categorical bar chart with:
- Your share of each build path.
- Field share (from public ADP-driven simulation).
- Playoff-team share (from Spec 04 historical mining).

Surfaces over- and under-indexing.

---

## CSV / JSON export

Every dashboard view exports to CSV (for spreadsheet review) and JSON (for programmatic use). Default filename: `lockes-picks-{view}-{YYYYMMDD}.csv`.

---

## Refresh cadence

Live during draft season:
- Leverage Score: live, updates after each pick across the portfolio.
- Effective N: live.
- Cluster exposure: live.
- Market Visualizer: nightly.
- Position-spend heatmap: nightly.
- Build-path heatmap: nightly.
- Player-pair matrix: nightly.

CVaR / North Star metric: nightly + on-demand button in dashboard.
