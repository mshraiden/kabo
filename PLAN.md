# KABO — Multiplayer Card Game: Implementation Plan

## Status

- **Repo created:** [github.com/mshraiden/kabo](https://github.com/mshraiden/kabo) (public)
- **Skeleton scaffolded** on `main` per the Project Structure below — workspaces, configs (`package.json`, `tsconfig.*`, `vite.config.ts`, `index.html`) are real/functional; all source files (`.ts`/`.css`) are intentionally empty placeholders for rules + UI to be filled in later
- A copy of this plan lives in the repo as `PLAN.md`
- Test push verified on `client/src/animation/cardAnimator.ts` (commit `328c2f0`)
- **Next:** fill in `shared/` types/constants/events, then start on Build Sequence step 2 (server bootstrap)

---

## Context

Building a custom web-based card game called KABO from scratch. The user has their own set of rules (not standard KABO). The game needs real-time 4-player multiplayer with shareable room codes so friends can join from anywhere — similar to how Gartic Phone or skribbl.io work. Card visuals should be fluid and art-quality (like Uno online). Rules are decoupled from transport so they can be added after the core infrastructure is built.

---

## Tech Stack

- **Runtime:** Node.js 20 + Express + Socket.io (single port, WebSocket + polling fallback)
- **Frontend:** Vite + vanilla TypeScript (no framework needed — fixed DOM structure)
- **Monorepo:** npm workspaces with three packages: `shared/`, `server/`, `client/`
- **Cards:** CSS art + inline SVG suit paths — no image assets, no Canvas, scales perfectly at 4K
- **Animations:** Web Animations API (FLIP technique for dealing, CSS transitions for flip/hover)
- **Hosting:** Railway (production) + ngrok (dev testing with friends before deploying)

---

## Project Structure

```
kabo/
├── package.json                      # workspaces: ["client","server","shared"]
├── tsconfig.base.json
│
├── shared/
│   ├── types.ts                      # Card, Player, GameState, PlayerAction, RoomState
│   ├── constants.ts                  # SUITS, VALUES, MAX_PLAYERS=4
│   └── events.ts                     # socket event name constants
│
├── server/
│   └── src/
│       ├── index.ts                  # Express + Socket.io bootstrap
│       ├── socketHandler.ts          # thin socket.on() adapter layer
│       ├── roomManager.ts            # room code generation, join/leave, lifecycle
│       ├── gameSession.ts            # holds one GameEngine per room, routes actions
│       └── engine/
│           ├── GameEngine.ts         # interface: initialize / processAction / isActionValid / checkEndCondition
│           ├── KaboRules.ts          # (stub until rules are known) implements GameEngine
│           └── index.ts              # exports ActiveEngine
│
└── client/
    └── src/
        ├── main.ts                   # entry, socket connect, reads ?r= param for auto-join
        ├── socket.ts                 # typed socket singleton
        ├── screens/
        │   ├── Lobby.ts              # create/join room, show code, copy-link button
        │   └── Game.ts               # game board, orchestrates components
        ├── components/
        │   ├── Card.ts               # single card render (CSS art + SVG)
        │   ├── Hand.ts               # a player's hand
        │   ├── Deck.ts               # deck + discard pile visuals
        │   └── PlayerSeat.ts         # one of 4 seats around the table
        ├── animation/
        │   └── cardAnimator.ts       # FLIP deal, flip-reveal, throw-to-pile
        └── styles/
            ├── cards.css             # card face art, suits, back pattern
            └── table.css             # green-felt table, 4-seat layout
```

---

## Room / Invite System

1. Player A clicks **Create Room** → server generates a 6-char code (e.g. `XYZ123`) from unambiguous charset (`ABCDEFGHJKMNPQRSTUVWXYZ23456789`)
2. Client builds shareable link: `https://kabo.railway.app/join?r=XYZ123`
3. **Copy Link** button puts it in clipboard
4. Player B opens link → `main.ts` reads `?r=` on load → auto-emits `joinRoom`
5. Owner sees live player list; clicks **Start Game** when 2–4 joined
6. Rooms auto-delete after 2 hours of inactivity

### Socket Event Contract

| Client → Server | Server → Client |
|---|---|
| `createRoom` | `roomCreated { code, playerId }` |
| `joinRoom { code }` | `roomJoined { room }` |
| `leaveRoom` | `roomUpdated { players }` |
| `startGame` | `gameStarted { state }` |
| `playerAction { type, ... }` | `stateUpdate { state, events }` |
| | `roomError { message }` |

---

## Card Graphics

- **Card face:** white rounded rectangle, CSS `box-shadow` for depth
- **Suits:** inline `<svg>` path — heart, spade, diamond, club each < 200 bytes, no HTTP requests
- **Back:** CSS `repeating-linear-gradient` pattern, no image needed
- **Flip:** `transform: rotateY(180deg)` on `.card-inner` with 0.4s ease transition
- **Deal animation:** cards start at deck XY, animate to hand position using FLIP technique
- **Hover:** `translateY(-8px)` + stronger `box-shadow`
- **Colors:** CSS custom properties (`--suit-red`, `--suit-black`) for easy theming

---

## Game Logic Isolation (Rules Drop-In Interface)

```typescript
// server/src/engine/GameEngine.ts
interface GameEngine {
  initialize(players: Player[]): GameState;
  processAction(state: GameState, action: PlayerAction): ActionResult;
  isActionValid(state: GameState, action: PlayerAction): boolean;
  checkEndCondition(state: GameState): EndResult | null;
}
```

- `gameSession.ts` is the only file that touches both Socket.io and GameEngine — pure adapter
- `KaboRules.ts` implements `GameEngine` — no socket imports, no DOM, pure logic
- Adding rules = fill in `KaboRules.ts`. Zero changes to transport or UI layers.
- `GameState` is a plain serializable object — safe to broadcast via JSON

### PlayerAction Union Type

```typescript
type PlayerAction =
  | { type: 'DRAW_FROM_DECK'; playerId: string }
  | { type: 'DRAW_FROM_DISCARD'; playerId: string }
  | { type: 'PLAY_CARD'; playerId: string; cardId: string; targetPosition?: number }
  | { type: 'KNOCK'; playerId: string }
  // add more as rules require
```

---

## Hosting — How Friends Connect

| Option | Cold Starts | Cost | Verdict |
|---|---|---|---|
| **Railway** | None (persistent) | ~$0–$5/mo | Recommended |
| Render (free) | 15 min spin-up | Free | Bad for multiplayer |
| Fly.io | None | Free (3 VMs) | Good, more setup |
| ngrok | N/A (tunnels local) | Free | Dev testing only |

### Railway Deployment Steps

1. Push repo to GitHub
2. Connect Railway to repo, set build command: `npm run build`
3. Set start command: `node server/dist/index.js`
4. Railway injects `PORT` automatically — no config needed
5. Share the `.railway.app` URL as your invite base

**For dev testing with friends:** `ngrok http 3000` → share the ngrok HTTPS URL → works instantly, no deploy needed

---

## Build Sequence

1. `shared/` — types, events, constants
2. Server bootstrap — Express + Socket.io, static file serving
3. Room create/join/leave with live player list (lobby fully working)
4. Invite URL flow — query param auto-join + copy-link button
5. Card component — CSS art + SVG suits, static demo
6. Card animations — FLIP deal, flip reveal, throw-to-pile
7. Game board layout — 4 seats, table, hand positions
8. `GameEngine` interface + stub `KaboRules` (pass-through)
9. Wire `gameSession.ts` adapter to socket events
10. Deploy to Railway; test ngrok in parallel for dev iteration
11. **Fill in `KaboRules.ts` once rules are confirmed**

---

## Verification

- **Local:** `npm run dev` → open `localhost:3000` in two tabs → create room in tab 1, join in tab 2
- **Friend test (pre-deploy):** `ngrok http 3000` → share URL → friend joins on mobile
- **Production:** Deploy to Railway → share `.railway.app` link → test 4-player session
- **Game logic:** Unit test `KaboRules.processAction()` in isolation with `vitest`
