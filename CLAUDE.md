# Dragons workspace — project map

AR ice/fire collectible game + live score dashboard. Read this before touching
anything here.

## Layout

- `QUANTUM1/` — **source of truth** for the AR experience. 8th Wall Studio
  project (`@8thwall/ecs`). Build with `npm run build` inside this folder →
  outputs to `QUANTUM1/dist/`.
  - `src/TEST.js` — the `scatter-prefab` ECS component. Procedurally scatters
    ice/fire collectible entities, owns their spawn/show/grow/shrink/hide
    lifecycle (`animateObject`). Only Studio-placed objects are lighting/camera;
    everything else is code-driven from here.
  - `src/collectible.js` — the `collectible` ECS component (per-object touch
    handler), running ice/fire counters, HTML counter UI, sends scores to the
    relay via `score-sync.js`.
  - `src/score-sync.js` — owns: the ice/fire element-pick overlay (persisted to
    `localStorage['quantum1-player-element']`), the WebSocket connection to
    score-relay, the 30-second round timer, and the "Create My Universe" button
    that fires once the timer ends. Exports `whenElementSelected()`,
    `isRoundActive()`, `sendScore()`.
- `html-export/` — **stale**, do not use or edit.
- `score-relay/` — Cloudflare Worker + Durable Object (`ScoreRoom` in
  `src/worker.ts`), authoritative multiplayer relay. Deployed at
  `wss://score-relay.pkch101.workers.dev`. Players connect as
  `role=player&name=Ice|Fire`, dashboard connects as `role=viewer`. Deploy with
  `wrangler deploy` from this folder (Cloudflare account required).
- `dashboard/index.html` — **source of truth** for the score dashboard page.
  Single static HTML file, no build step. Connects to the relay over
  WebSocket, renders live bar/line charts + raw table keyed by player name
  (`'Ice'` / `'Fire'`), and shows a fullscreen ASCII "universe" reveal
  animation on the `{type: 'universe', winner}` broadcast.
- `ar-dre-site/` — the **published** GitHub Pages repo
  (`github.com/wittywit/ar-dre`, custom domain `wittywit.com`). Git repo lives
  only in this folder. Contains:
  - `ar/` — a synced copy of `QUANTUM1/dist/` (served at `/ar`)
  - `dashboard/index.html` — a synced copy of `dashboard/index.html` (served
    at `/dashboard`)
  - root `index.html` — landing page linking to both
  - `CNAME` — `wittywit.com`

**Never edit files inside `ar-dre-site/ar/` or `ar-dre-site/dashboard/`
directly** — they are build/copy outputs. Edit `QUANTUM1/src/` or
`dashboard/index.html` and re-sync:

```sh
cd QUANTUM1 && npm run build
rsync -a --delete QUANTUM1/dist/ ar-dre-site/ar/
cp dashboard/index.html ar-dre-site/dashboard/index.html
```

## Game flow (current)

1. Phone loads AR scene → element-pick overlay asks Ice or Fire
   (`score-sync.js`). Choice persists in `localStorage` and is sent as the
   player's `name` (`'Ice'` or `'Fire'`) to the relay.
2. A player only ever sees their own chosen element's collectibles spawn in AR
   (`TEST.js` gates `animateObject` calls on `matchesChosenElement`).
3. A 30-second round timer starts on connect. Collecting is blocked once the
   round ends (`isRoundActive()` guards in `collectible.js` and `TEST.js`).
   Scores stream to the relay + dashboard live throughout via `sendScore()`.
4. When the timer hits zero, the phone shows a "Create My Universe" button.
5. Tapping it sends `{type: 'universe'}` to the relay. The Durable Object
   sums `ice`/`fire` totals across all connected players server-side (not
   client-side, to stay authoritative) and broadcasts
   `{type: 'universe', winner: 'ice' | 'fire' | 'tie'}` to every socket
   (players + dashboard).
6. The dashboard listens for that message and goes fullscreen with a
   procedurally animated ASCII-art "universe emerging" effect — blue if ice
   won, red if fire won, grey on a tie. Closeable via an on-screen button.

## Conventions

- Never `git push` from `ar-dre-site/` without asking the user first, even if
  a prior push in the session was approved — see global git-push rule.
- Player identity going forward is fixed to `'Ice'` / `'Fire'`, not freeform
  names — the dashboard's bar/line/table rendering already keys off `name` so
  no restructuring was needed there.
