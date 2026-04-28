# Spec 07 — Chrome Extension

Thin observer. Reads picks, posts them, displays JSON. All logic lives in the backend (Spec 05). MV3's 30-second service worker would otherwise bite us.

---

## Manifest V3 architecture

```
extension/
├── manifest.json                # MV3
├── public/
│   └── icons/
└── src/
    ├── content/
    │   ├── ud_observer.ts       # Underdog passive DOM observer
    │   └── dk_observer.ts       # DraftKings passive DOM observer
    ├── background/
    │   ├── service_worker.ts    # WebSocket bridge, tab tracking
    │   └── selector_pack.ts     # Hot-loaded site selectors
    └── sidepanel/
        ├── App.tsx              # React side panel root
        ├── components/
        └── styles/
```

`manifest.json` essentials:
```json
{
  "manifest_version": 3,
  "name": "Locke's Picks",
  "permissions": ["sidePanel", "storage", "tabs"],
  "host_permissions": [
    "https://underdogfantasy.com/*",
    "https://*.draftkings.com/*"
  ],
  "background": {"service_worker": "background/service_worker.js"},
  "side_panel": {"default_path": "sidepanel/index.html"},
  "content_scripts": [
    {"matches": ["https://underdogfantasy.com/draft/*"], "js": ["content/ud_observer.js"], "run_at": "document_idle"},
    {"matches": ["https://*.draftkings.com/draft/*"],  "js": ["content/dk_observer.js"], "run_at": "document_idle"}
  ]
}
```

---

## Content scripts (passive observers)

**Hard rule: zero DOM mutation.** No overlays, no row coloring, no badge injection. The page renders exactly as Underdog/DraftKings rendered it.

**Method:** `MutationObserver` on the draft-board container. On mutation:
1. Extract pick events (player + pick number + slot).
2. Diff against last-known state.
3. Emit `chrome.runtime.sendMessage({type: 'pick_observed', payload})`.

**Debouncing:** MutationObserver fires multiple times per UI update. Debounce to 50 ms; the backend de-dupes via `sha256(draft_id || pick_number)` (Spec 05).

**Error handling:** If selectors fail to extract a pick (e.g., site updated their DOM), the content script:
1. Logs the failure to background.
2. Falls back to the next selector pack version (Spec 01).
3. If all packs fail, surfaces a "selector pack outdated" toast in the side panel and links to the manual update flow.

### Selector pack hot-loading

`selector_pack.ts` fetches the latest selectors from `localhost:8000/selectors/{site}/latest` on extension startup. Stored in `chrome.storage.local`. Re-fetched daily.

If Underdog redesigns mid-season, the user updates the pack via a `localhost:8000/admin/selectors` POST without re-publishing the extension or re-uploading to the Chrome Web Store. (Side-loaded extension; Chrome Web Store publishing is out of scope since this is single-user.)

---

## Background service worker

**WebSocket bridge:** Persistent `wss://localhost:8000/ws` connection. Reconnect with exponential backoff on disconnect. On reconnect, send `replay_state` request keyed by `draft_id`s of all active tabs.

**Tab tracking:** Maintains a map of `tab_id → draft_id → site`. On tab focus change, emits `tab_focused` event so the side panel auto-switches to the active draft.

**Concurrent drafts:** Up to 10 active draft tabs. Each gets its own observer instance and independent backend session. Background worker multiplexes pick events over the single WebSocket.

**MV3 30-second termination:** The service worker can be killed at any time. State must be re-derivable on wake from `chrome.storage.local` + a backend replay. Specifically:
- Tab map stored in `chrome.storage.local`.
- Observer instances re-created from content scripts on wake (content scripts persist independently of service worker).
- WebSocket re-opened on wake; backend replays state.

---

## Side panel (React)

**Layout:**
```
┌──────────────────────────────────────┐
│ [UD ½-PPR]  Pick 48 (R5.4) of 216    │  ← header: profile badge + pick context
├──────────────────────────────────────┤
│ TOP CANDIDATES                       │
│  1. Player Name (WR-BUF) — 42.3      │  ← top-N from Spec 05
│     +stack +late-season +leverage    │
│     34% to be available next pick    │
│  2. ...                              │
│  3. ...                              │
├──────────────────────────────────────┤
│ MY ROSTER                            │  ← in-progress roster
│  QB: ...                             │
│  Stacks: QB-WR (size 2)              │
│  Build path: Hero-RB                 │
│  Playoff similarity: 18%             │
├──────────────────────────────────────┤
│ PORTFOLIO STATUS                     │  ← compact, full dashboard in main app
│  Effective N: 280                    │
│  Top 0.1% prob: 4.2%                 │
│  IR vs ADP-bot: +1.8                 │
└──────────────────────────────────────┘
```

**State management:** Zustand (lightweight) with subscriptions to backend WebSocket events. No Redux; not needed.

**Performance:** Re-render only the candidates list on a pick event. React.memo on roster and portfolio sections — they update on different cadences.

**Tab-switch behavior:** When user switches Chrome tabs to a different draft, side panel content updates within ~50 ms. Background draft sessions continue updating in the backend regardless of which is on screen.

---

## Latency budget (extension contribution)

From Spec 05 total budget of 250 ms:

| Stage | Budget | What |
|---|---|---|
| MutationObserver fire → debounce → extract | 30 ms | content script |
| sendMessage to background | 5 ms | Chrome IPC |
| Background → WebSocket emit | 10 ms | localhost wss |
| Backend round-trip | 150 ms | Spec 05 budget |
| WebSocket recv → side panel state update | 10 ms | |
| React re-render | 25 ms | |
| Paint | 20 ms | browser |

If a pick takes > 250 ms, the side panel displays a "thinking..." spinner; the user knows the recommendation is loading and not stuck.

---

## Settings popup

Minimal popup (not the side panel) for:
- Show backend status (connected / disconnected / fast-path).
- "Reload selector pack" button.
- "Reset draft session" button (for testing).
- Link to open the main dashboard in a new tab (Phase 8 portfolio dashboard runs at `http://localhost:8000/dashboard`).

No drafting logic in the popup.

---

## Auth

None. Single user, localhost only. Backend listens only on `127.0.0.1:8000`. Extension has `host_permissions` only for the draft sites and `localhost`.

---

## Distribution

- Single user, side-loaded via Chrome's "Load unpacked" developer mode.
- Build via Vite: `pnpm build` → `extension/dist/`.
- Versioned alongside the backend `.exe`: same git tag, same release.

No Chrome Web Store publishing. No paid signing. Zero distribution overhead.

---

## What the extension explicitly does NOT do

- Render any UI on the draft page itself.
- Auto-pick or auto-confirm anything.
- Communicate with any service other than `localhost:8000` and the draft site itself.
- Modify any HTTP request or response.
- Read any DOM node outside the matched draft URL paths.
- Capture or transmit anything when you're not on a draft page.
