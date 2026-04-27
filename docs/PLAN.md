# Fantasy Draft Engine — Build Plan

A draft assistant in the spirit of LegUp Sidekick, Draft Caddy, and ETR's Solver Draft Assistant. Browser-overlay first, best-ball focused, with a portfolio layer.

---

## 1. Competitor feature scrape

### LegUp Sidekick (Legendary Upside)
- Browser overlay on Underdog and DraftKings best ball.
- **Dynamic rankings** that re-sort after every pick based on roster construction, same-team stacking, bring-back correlation, QB bye overlap, and stacking optionality.
- Toggle between LegUp rankings, ADP, and user-uploaded custom rankings.
- **Player Availability %** — probability a player survives to your next pick (BBM-style standard contests only; not Eliminator, Weekly Winners, SuperFlex, Marathon, Sprint).
- **Market Visualizer** — where LegUp rankings diverge from market ADP.
- **Jaccard Similarity** tool — portfolio-level diversification across all your drafts.
- Big Board view; access to all LegUp written + podcast content.

### Draft Caddy (Endgame Syndicate)
- Browser extension (Chrome + Firefox) for Underdog, DraftKings, Drafters. ~$29.99/mo.
- Configurable best-ball settings; user controls every overlay element + colors.
- Custom ADP CSV upload (DraftKings supports name + position + team matching).
- **Suggest a Player** (primary + secondary suggestion).
- Week 15/16/17 playoff correlation highlighting on player rows and schedule cells.
- Live player-exposure overlays across drafts in progress.

### ETR Solver Draft Assistant
- Browser overlay for Underdog + DraftKings best ball, plus Underdog Battle Royale (weekly snake).
- **Smart Player Recommendations** — dynamically scores remaining pool against your roster + contest parameters; adapts to stacking vs. contrarian objectives.
- Auto-sync to ETR Underdog/DraftKings rankings or in-season DFS projections (with ETR sub).
- Custom rules: player groups, exposure caps, locks/excludes, stacking rules, contrarian rules.
- Personalized ownership tracking from your historical drafts.
- Lives inside the larger Solver suite (DFS optimizer, sims, bankroll tracker).

### Adjacent tools (worth borrowing from)
- **Spike Week Draft Hacker** — fully customizable overlay colors; playoff-week highlighting; DraftIQ exposure sync.
- **Best Ball Team Builder** — side-by-side companion (not overlay); build paths; portfolio up to 500 teams (Pro).
- **Best Ball Overlay** — CLV tracking, playoff stack visualization.
- **FantasyPros Draft Wizard / RotoWire / Draft Sharks** — redraft-league sync (Yahoo, ESPN, Sleeper, CBS, NFL).

---

## 2. Synthesized feature matrix

| Capability | LegUp | Draft Caddy | Solver | Ours (target) |
|---|---|---|---|---|
| Underdog overlay | ✅ | ✅ | ✅ | ✅ |
| DraftKings overlay | ✅ | ✅ | ✅ | ✅ |
| Drafters overlay | — | ✅ | — | ✅ |
| Snake / Battle Royale | — | — | ✅ | ✅ |
| Redraft sync (Yahoo/ESPN/Sleeper) | — | — | — | ✅ (stretch) |
| Dynamic re-ranking after each pick | ✅ | partial | ✅ | ✅ |
| Custom rankings upload (CSV) | ✅ | ✅ | ✅ | ✅ |
| Player Availability % to next pick | ✅ | — | — | ✅ |
| Stack / correlation highlighting | ✅ | ✅ (W15–17) | ✅ | ✅ |
| Bye-week conflict warnings | ✅ | — | partial | ✅ |
| Suggest-a-player (top N) | partial | ✅ | ✅ | ✅ |
| Exposure tracking across drafts | partial | ✅ | ✅ | ✅ |
| Portfolio Jaccard / diversification | ✅ | — | — | ✅ |
| Market vs. our-rankings visualizer | ✅ | — | — | ✅ |
| Custom rules (locks, fades, caps, stacks) | partial | partial | ✅ | ✅ |
| Build-path tracking | — | — | — | ✅ (from BBTB) |
| Customizable overlay colors / fields | partial | ✅ | partial | ✅ |
| Post-draft grading + portfolio dashboard | — | partial | ✅ | ✅ |

**Net-new differentiators we'll build:**
1. **Pick EV engine** — score each candidate as `value_now − E[value_at_next_pick]` using a Monte Carlo simulator over ADP variance (most competitors give a static score).
2. **Roster construction optimizer** — solve remaining roster slots as a constrained optimization, not just "next best player."
3. **Multi-source rankings blender** — weighted blend of ETR / LegUp / FantasyPros / user CSV with per-source confidence.
4. **Playoff schedule + weather-adjusted correlation** scoring (W15–17 game environment).
5. **Open data layer** — exposures and projections exportable as JSON/CSV by default.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Browser Extension (MV3, TypeScript)                        │
│  ├── content scripts: Underdog / DraftKings / Drafters      │
│  │     - DOM scrape pick stream                             │
│  │     - inject overlay UI (React shadow-DOM)               │
│  ├── background service worker                              │
│  │     - WebSocket to engine, auth, cache                   │
│  └── popup: settings, rankings upload, account              │
└────────────────────────┬────────────────────────────────────┘
                         │ wss
┌────────────────────────▼────────────────────────────────────┐
│  Draft Engine API (Python FastAPI or Node Fastify)          │
│  - /draft/state  (POST pick events, GET state)              │
│  - /recommend    (returns ranked candidates + reasoning)    │
│  - /availability (Monte Carlo P(player at next pick))       │
│  - /portfolio    (exposures, Jaccard, build-paths)          │
└──────────┬─────────────────────────────┬────────────────────┘
           │                             │
   ┌───────▼────────┐           ┌────────▼─────────┐
   │ Recommender    │           │ Simulator        │
   │ - rules engine │           │ - ADP MC sims    │
   │ - blended rank │           │ - opponent model │
   │ - stack logic  │           │ - availability % │
   └───────┬────────┘           └────────┬─────────┘
           └──────────────┬──────────────┘
                  ┌───────▼────────┐
                  │ Data layer     │
                  │ Postgres + S3  │
                  │ - players      │
                  │ - projections  │
                  │ - ADP history  │
                  │ - user drafts  │
                  │ - rules        │
                  └────────────────┘
```

**Tech choices**
- Extension: Manifest V3, TypeScript, React + shadow DOM (style isolation), Vite.
- Backend: Python 3.12 + FastAPI; Pydantic models; uvicorn; Redis for hot draft state and rate limiting.
- Storage: Postgres (players, drafts, exposures, rules); S3 for projection snapshots and exposure CSVs.
- Sim: NumPy / Numba for Monte Carlo; precompute lookup tables where possible.
- Auth: Magic-link email + JWT; Stripe for subscriptions.

---

## 4. Data sources

| Need | Source | How |
|---|---|---|
| Player IDs / bio | Sleeper public API + manual mapping | nightly sync |
| ADP (Underdog, DK, NFC, FFPC) | scrape public ADP pages + user-imported drafts | scheduled crawl |
| Projections | partner feed (FantasyPros, Sleeper) or build our own from nflverse / nfl_data_py | nightly |
| Playoff schedule | nflverse schedule | static per season |
| Weather (W15–17) | OpenWeather / NOAA | T-3d before week |
| User draft pick stream | extension content script (DOM events) | live wss |
| User exposure history | extension upload + manual CSV | per draft |

Mapping players across sites is the perennial hard problem — plan for a curated `player_alias` table seeded once, with an admin UI to resolve mismatches.

---

## 5. Recommender — how a pick suggestion is computed

For each remaining player `p`:
```
score(p) = w1 * blended_rank_value(p)
        + w2 * roster_fit(p, current_roster)
        + w3 * stack_bonus(p, current_roster)
        + w4 * playoff_correlation(p, current_roster)
        - w5 * bye_conflict(p, current_roster)
        - w6 * portfolio_overexposure(p, user_portfolio)
        + w7 * scarcity(p, position, picks_until_next)
```
Then rank, and surface top N with one-line reasoning ("WR1 stack with rostered QB; survives W17 bye conflict").

`scarcity` uses `availability_pct(p, next_pick)` from the simulator — the same number shown on the overlay.

**Availability simulator**: Monte Carlo over ADP distribution + opponent draft tendencies (per-pick mean and stddev from historical pick logs); 5–10k sims under 100ms cached.

---

## 6. UX — what shows up on the draft screen

1. **Top-of-pick overlay strip**: "Recommended: J. Chase (94) — stack QB+WR; 38% to be available next pick".
2. **Rank delta column** next to ADP — green/red diff vs. our blended rank.
3. **Stack badges** on player rows for teammates of rostered players.
4. **Playoff badges** — small W15/16/17 schedule strength chips.
5. **Bye conflict warning** — red dot if drafting them creates a same-bye QB problem.
6. **Custom rule chips** — "LOCK", "FADE", "CAP HIT" on the row when a rule fires.
7. **Side panel** — current roster construction, exposure %, build-path match.

Everything customizable in popup: which columns, colors, thresholds, which rankings to blend.

---

## 7. Portfolio layer

- All completed drafts auto-sync via the extension.
- **Exposure dashboard**: per-player and per-stack exposure across all teams in a contest.
- **Jaccard similarity matrix** — heatmap of how unique each team is vs. the rest of your portfolio.
- **Build-path tracker** — which configurations (e.g., Zero-RB, Robust-RB, Onesie-Late) you're heavy/light on.
- **Market visualizer** — distribution of your picks vs. market ADP per round, highlighting where you're contrarian.
- CSV / JSON export and shareable read-only links.

---

## 8. Milestones

| # | Milestone | Scope | ~Effort |
|---|---|---|---|
| 0 | Repo scaffolding | mono-repo (extension/, api/, web/, infra/), CI, lint, types | 3 d |
| 1 | Data foundation | player table, alias mapping, ADP ingest, projections ingest | 1.5 wk |
| 2 | Engine v1 | static blended rank, scarcity calc, FastAPI endpoints, tests | 1.5 wk |
| 3 | Underdog content script | DOM scrape, pick events, basic overlay rendering | 1.5 wk |
| 4 | Recommender v1 | top-N suggestion, stack bonus, bye conflict, reasoning strings | 1 wk |
| 5 | Availability % simulator | Monte Carlo, caching, perf budget < 100 ms | 1 wk |
| 6 | DraftKings + Drafters scripts | parity with Underdog | 1.5 wk |
| 7 | Portfolio dashboard (web) | Next.js, exposures, build-paths, Jaccard | 2 wk |
| 8 | Custom rules + CSV rankings | popup UI, rule DSL, rules eval in recommender | 1 wk |
| 9 | Auth + billing | magic link, Stripe, plan gating | 1 wk |
| 10 | Battle Royale / snake support | per-format ruleset, snake-aware scarcity | 1 wk |
| 11 | Beta polish, telemetry, docs | feature flags, error reporting, support docs | 1 wk |

Total to public beta: **~12–14 weeks** for one engineer; ~7–8 weeks for two.

---

## 9. Risks & open questions

- **TOS / scraping**: Underdog, DK, Drafters may not love a DOM-injecting overlay. Read TOS, ship as user-installed extension (user is the agent), no MITM. Be ready to pivot to a side-by-side companion (à la Best Ball Team Builder) if any platform sends a C&D.
- **DOM brittleness**: each platform redesign breaks the scraper. Mitigation — versioned selector packs hot-loaded from API so we can patch without a Chrome Web Store re-review.
- **Player ID mapping** — recurring source of subtle bugs. Invest in admin tooling early.
- **Projection licensing**: building our own from nflverse is free but heavy. Partner feed shortcuts months of work but eats margin.
- **Latency**: overlay must respond < 200 ms after each pick or it feels broken. Pre-compute aggressively, push state via wss.
- **Pricing**: Sidekick ~$50/season tiers, Draft Caddy $29.99/mo, Solver tiered. Likely land at $19–29/mo seasonal with portfolio in higher tier.

---

## 10. What this plan deliberately defers

- DFS optimization (Solver does this; we stay focused on draft).
- In-season lineup tools (separate product).
- Mobile app (extension first; native later if there's demand).
- Auction drafts (post-MVP).
