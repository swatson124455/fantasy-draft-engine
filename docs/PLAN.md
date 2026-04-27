# Locke's Picks — v1 Spec

Personal best-ball draft assistant for one user (Locke). Live, popup-only, Windows + Chrome, $0 budget, Python local-first.

> Inspired by LegUp Sidekick, Draft Caddy, and ETR Solver — but rebuilt as a personal tool with a Pick EV engine, team stacking logic, and a portfolio layer tuned for a 3k-drafts/season grinder.

---

## 1. Constraints (locked)

| | |
|---|---|
| Audience | Single user. No auth, no billing, no public release. |
| Platforms | Underdog + DraftKings best ball only. NFL only. |
| Browser | Chrome (MV3) only. Desktop. |
| OS | Windows. |
| Concurrent drafts | Up to 10 live tabs. |
| Page mutation | **None.** Content script is a passive observer. UI lives in a Chrome **side panel**. |
| Hosting | Local-first. Backend runs on `localhost`, SQLite at `%APPDATA%\Lockes-Picks\`. |
| Budget | $0 runtime. OpenAI used only as a dev-time accelerator. |
| Mobile | Out of scope for v1. |
| Auction / dynasty / keeper | Out of scope. |

---

## 2. Feature set (locked)

### Live draft mode
- Read pick stream from UD and DK draft pages via `MutationObserver` — no DOM writes.
- Drafted players removed from suggestions and the available board within ~200 ms.
- Side panel auto-switches to the active draft tab; all 10 sessions kept current in the background.

### Pick EV engine
For each available player `p`:
```
score(p) = blended_value(p)
        + stack_bonus(p, roster)
        - bye_conflict(p, roster)
        + late_season_bonus(p)
        + override(p)              # capped, ADP-decayed
        + scarcity_term
        - reach_penalty(p, pick)   # round-aware
        + flex_bonus(p, pick, roster)  # late-round stack / gap-fill unlock
```
Where `scarcity_term = expected_value_drop_if_skipped` from the Monte Carlo availability sim.

Surfaced as: top N candidates with EV value, one-line reasoning, late-season chip, stack badge.

### Round-aware reach tolerance
ADP variance is not constant — pick stddev grows roughly linearly with pick number. A 5-pick reach in round 2 is meaningful; a 5-pick reach in round 13 is noise. The engine treats this as a first-class primitive:

```
adp_stddev(pick) ≈ 1.5 + 0.15 · pick      # default; recalibrated from your draft history
reach_delta      = current_pick − player_adp           # +ve = reaching
reach_penalty    = (reach_delta / adp_stddev(pick))² · w_reach
```
Same numeric reach has high penalty early and ~zero penalty late.

**Late-round flex bonus.** When `adp_stddev(pick) > flex_threshold` (≈ round 7+), the engine *unlocks* an additive bonus for picks that:
- complete a viable stack (QB + 2nd pass-catcher, bring-back),
- fill a thin position the roster needs (e.g., still no TE2 entering round 13),
- pair with high-co-occurrence players from the portfolio matrix.

Effect: late-round "reaches" that lock in a stack or roster gap are *recommended*, not flagged.

The same `adp_stddev(pick)` feeds the MC sim (so availability % widens correctly late) and the override decay threshold (so a round-9 override isn't decayed by round-9-noise).

### Monte Carlo availability simulator
- Inputs: ADP per site, `adp_stddev(pick)` curve (recalibrated from your pick logs), opponent draft tendencies model.
- 5–10k sims, target < 100 ms per refresh.
- Output: P(player available at next pick) shown next to each candidate.

### Stacking logic
- QB + same-team WR/TE bonuses (additive per pass-catcher already rostered).
- Bring-back bonus on opposing pass-catcher in W15–17 games.
- Bye-week conflict penalty when a second QB shares a bye with rostered QB.

### Late-season bonus heuristic + override
Heuristic adders:
- Rookie: configurable bump.
- Handcuff to fragile/aging RB1: configurable.
- Schedule strength W10–17 (DVOA): ±%.
- Returning from injury / late starter: +%.

Manual override column in `rankings.csv`:
```
player_id, override_bonus, note
```
- `adp_at_override_time` and `override_set_date` snapshotted on save.
- **Auto-decay** as ADP moves toward your view: `decayed = original * max(0, 1 − adp_delta / decay_threshold(pick))`. The threshold scales with `adp_stddev(pick)` so a round-2 override decays on small ADP moves while a round-9 override needs a bigger move to decay.
- **Hard cap** at ±15% combined bonus per player.
- **Stale-override review panel** — weekly nudge to keep / reduce / remove.

### Custom rankings
- CSV upload only for v1 (LegUp-sheet style template provided).
- Columns: `player_id, name, team, pos, my_rank, my_tier, override_bonus, note`.
- Re-upload anytime; tool diffs and applies.
- Bye-week and team auto-joined from player table — you don't have to maintain those.

### Projections engine (build-own, free sources)
- **Baseline model**: trained on `nfl_data_py` historical seasonal stats (last 6 seasons), simple gradient-boost regression per position.
- **Blend**: weighted average of baseline + Sleeper API projections + your overrides.
- **Late-season bonus** layered on top.
- Refresh nightly via APScheduler.
- Stretch: weekly projections (W1–17) so playoff scoring is real, not estimated. Deferred to v1.1.

### Portfolio (offline analysis)
- All your completed drafts auto-imported (CSV from UD/DK + extension capture as backup).
- **Exposure dashboard**: per-player and per-stack across the portfolio.
- **Player-pair co-occurrence matrix**: heatmap of how often any two players appear together in your teams. Sortable, filterable by contest type.
- **Build-path heatmap**: distribution of your roster constructions (Zero-RB, Hero-RB, Late-QB, etc.).
- CSV / JSON export.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Chrome Extension (MV3, TypeScript)                         │
│  ├── content scripts (UD, DK)                               │
│  │     - MutationObserver, READ-ONLY, debounced             │
│  │     - emit pick events                                   │
│  ├── background service worker                              │
│  │     - WebSocket to localhost:8000                        │
│  │     - tab tracking                                       │
│  └── side panel (React)                                     │
│        - suggestions, EV, stacking, late-season chips       │
│        - portfolio summary                                  │
└────────────────────────┬────────────────────────────────────┘
                         │ ws://localhost:8000
┌────────────────────────▼────────────────────────────────────┐
│  Local Engine (Python 3.12, FastAPI)                        │
│  - draft session manager (10 concurrent)                    │
│  - recommender                                              │
│  - MC simulator (NumPy)                                     │
│  - projections service (APScheduler nightly)                │
│  - portfolio analytics                                      │
│  Storage: SQLite at %APPDATA%\Lockes-Picks\db.sqlite        │
│  Run: PyInstaller .exe, launched on login via Startup       │
└─────────────────────────────────────────────────────────────┘
```

**Run model**: `lockes-picks.exe` starts on Windows login, listens on `localhost:8000`. Chrome extension auto-connects. No internet required during a draft beyond the draft site itself.

**Data flow on a pick**:
1. Underdog DOM mutates (new pick announced).
2. Content script's `MutationObserver` fires → extracts player + pick number.
3. Background worker sends `{tabId, draftId, pick}` over WebSocket.
4. Engine updates that draft's state, recomputes top N + EV + availability.
5. Side panel receives updated payload, re-renders.

Target end-to-end latency: < 250 ms.

---

## 4. Data sources

| Need | Source | Method |
|---|---|---|
| Player IDs / bio | Sleeper API | nightly sync |
| ADP (UD, DK) | scrape public ADP pages | nightly cron |
| Pick variance per slot | aggregated from your own past drafts + bootstrap from historical public draft data | rebuilt weekly |
| Seasonal projections (baseline) | self-trained on `nfl_data_py` | season start + nightly |
| Seasonal projections (blend partner) | Sleeper API projections | nightly |
| Schedule + opponent strength | `nfl_data_py` schedule | season start |
| Live pick stream | extension content script | wss |
| Your draft history | UD CSV + DK CSV upload + live capture | on draft completion |

---

## 5. Repo layout

```
lockes-picks/
├── engine/                    # Python backend
│   ├── api/                   # FastAPI routes
│   ├── recommender/           # EV engine, stacking, late-season
│   ├── simulator/             # Monte Carlo
│   ├── projections/           # baseline model + blender
│   ├── portfolio/             # exposures, co-occurrence
│   ├── ingest/                # ADP + Sleeper sync
│   ├── db/                    # SQLite models, migrations
│   └── tests/
├── extension/                 # Chrome MV3
│   ├── src/content/           # UD + DK observers
│   ├── src/background/
│   ├── src/sidepanel/         # React side panel
│   └── manifest.json
├── data/                      # CSVs, model artifacts (gitignored)
├── scripts/                   # build .exe, install on Windows
└── docs/
```

---

## 6. Milestones

| # | Milestone | Output | Effort |
|---|---|---|---|
| 0 | Repo + tooling | monorepo, lint, CI, Windows build script | 1 wknd |
| 1 | Data foundation | player table, alias mapping, Sleeper sync, ADP ingest | 2 |
| 2 | Projections v1 | baseline model + Sleeper blend + late-season bonus | 2–3 |
| 3 | Local backend skeleton | FastAPI, SQLite schema, draft session model, ws | 1 |
| 4 | UD content script | passive DOM observer, pick event pipeline, real-draft verified | 2 |
| 5 | DK content script | parity with UD | 1–2 |
| 6 | Side panel UI | React side panel: suggestions, EV column, late-season chips | 2 |
| 7 | EV engine + sim | Monte Carlo, EV scoring, perf < 100 ms | 2 |
| 8 | Stacking + bye logic | bonuses, bring-back, bye conflicts | 1 |
| 9 | Custom rankings + override hygiene | CSV import, ADP-decay, stale review | 1 |
| 10 | Portfolio dashboard | exposures, player-pair matrix, build-paths | 2 |
| 11 | 10-tab harden | concurrent state, tab switching, perf tests | 1 |
| 12 | Polish + dogfood + bug bash | real drafts, fix what surprises | 2 |

**Total ~19–22 focused weekends.** Dogfood-able from milestone 4.

---

## 7. Risks I'm watching

- **DOM brittleness on UD/DK**: SPAs with churning class names. Mitigation — versioned selector pack stored in SQLite, hot-loadable without re-publishing the extension. If UD redesigns mid-season, fix is one config update.
- **Build-own projections quality vs. competitors who license**: v1 will be coarser than ETR. Mitigation — your manual rankings + override column carry the edge until the model matures.
- **WebSocket reliability across 10 tabs**: handle reconnection, draft-state replay on reconnect.
- **Pick-event de-duplication**: MutationObserver can fire on the same DOM update multiple times. Idempotent pick handler with hash of `{draftId, pickNumber}`.
- **Self-sourced ADP volatility**: small N early-season → noisy availability sim. Mitigation — bootstrap from public ADP for first few weeks until your own draft history catches up.

---

## 8. Explicitly out of scope (v1)

- Mobile / app drafts
- Snake / Battle Royale
- Redraft (Yahoo/ESPN/Sleeper) sync
- Auction / dynasty / keeper
- DFS lineup tools
- In-season lineup optimization
- Multi-user / auth / billing
- Multi-sport
- AI/LLM "explain this pick" in the runtime path
- Weekly W1–17 projections (deferred to v1.1)
- In-tool rankings spreadsheet editor (CSV-only for v1)
- Jaccard similarity (player-pair co-occurrence is the simpler equivalent we're shipping)
- Market visualizer (deferred)
- Weather-adjusted correlations (deferred)
