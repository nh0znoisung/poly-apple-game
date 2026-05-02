# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Poly Apple** is an educational, real-time multiplayer game where players compete by drawing polynomial equations to "eat apples" on an HTML5 Canvas. The project uses a decoupled microservice architecture with entirely independent frontend and backend codebases that can be deployed separately.

**Current Architecture**: MVP-level with file-based JSON persistence and in-memory state management.

## Development Commands

### Local Development (Run Both Simultaneously)

**Terminal 1: Backend Server**
```bash
cd backend
npm install
npm run dev
```
- Runs on `http://localhost:3000`
- Uses Nodemon for auto-reload on file changes
- Watches `server.js`

**Terminal 2: Frontend UI**
```bash
cd frontend
npm install
npm run dev
```
- Runs on `http://localhost:5173` (Vite dev server)
- Hot Module Replacement (HMR) enabled
- No build step needed during development

### Build for Production

**Frontend:**
```bash
cd frontend
npm run build
```
Outputs optimized bundle to `dist/` folder. Deploy to CDN (Vercel, Netlify, etc.).

**Backend:**
```bash
cd backend
npm start
```
Runs production mode without Nodemon. Deploy to Node.js hosting (Render.com, Railway.app, etc.).

## Architecture & Key Systems

### Frontend Architecture (`/frontend`)

**Tech Stack**: Vanilla JavaScript (ES6+), HTML5 Canvas, CSS3, Vite, Socket.io-client

**Files**:
- `index.html` - Main entry point. Contains UI layout with three screens: Lobby, Game, Summary.
- `public/script.js` - Core game logic (~19KB). Handles:
  - DOM manipulation (screen toggling)
  - Canvas rendering (coordinate system, graph drawing)
  - Collision detection (equation intersection with apples)
  - Real-time event handling (Socket.io listeners: `playerJoined`, `gameStart`, `gameStateUpdated`)
- `style.css` - Custom CSS with glassmorphism aesthetic

**Key Design Pattern**:
The frontend parses user-input polynomial equations (e.g., `x^2 + 2x`) and renders them as continuous curves on Canvas. It continuously checks if the curve intersects with apple coordinates. When a collision is detected, it emits `updateGameState` to the server (server is the authoritative referee to prevent cheating).

**BACKEND_URL Configuration**:
Currently hardcoded in `public/script.js` to `http://localhost:3000`. Update this before deploying to production to point to your live backend URL.

### Backend Architecture (`/backend`)

**Tech Stack**: Node.js, Express.js, Socket.io, File-based JSON persistence

**Core File**:
- `server.js` - Monolithic server (~19KB). Contains all logic:
  - HTTP/Express routing (serves static files)
  - Socket.io event handlers
  - In-memory state management (Map objects)
  - File I/O persistence
  - Room lifecycle management
  - Spectator mode support

**Data Persistence**:
- **Runtime**: In-memory `Map()` objects for ultra-fast O(1) lookups: `playersDb`, `roomsDb`, `sessionsDb`, `spectators`
- **Durability**: Every 2 seconds, state is dumped to JSON files via `fs.writeFileSync()`:
  - `data/players.json` - Player identities (UUID, name, avatar index)
  - `data/rooms.json` - Active room states (Waiting, Playing, Ended)
  - `data/sessions.json` - Match history, eaten apples, equations used
- **Load**: On startup, `loadState()` reconstructs Map objects from JSON files

**Key Systems**:

1. **Room Management**:
   - 6-character alphanumeric room codes (generated randomly)
   - Room states: `waiting` (1 player) → `playing` (2 players) → `ended`
   - Max 2 players per room
   - Avatar conflict resolution: prevents duplicate avatars in same room

2. **Game Session**:
   - Triggered when both players emit `gameReady`
   - Apple positions pre-generated server-side based on `appleDensity` and `gridSize`
   - Server stores all apple eats to `eatenApples` Map to prevent client-side hacking
   - Spectators receive full sync payload: `eatenApples`, `elapsedTime`, `history`

3. **Garbage Collection (Cron Job)**:
   - Runs every 60 seconds
   - Checks if rooms have active Socket.io connections
   - Inactive rooms expire after 5 minutes
   - Permanently deletes expired rooms to prevent memory leaks

4. **Spectator Mode**:
   - Allows clients to join rooms as watchers (not counted in player count)
   - Server sends complete game state snapshot to spectators on join
   - Spectators receive real-time updates

**Socket.io Events**:
- `createRoom` - Player creates a new room
- `joinRoom` - Player joins existing room
- `gameReady` - Both players ready; triggers game start
- `getAvailableRooms` - Query available public rooms
- `updateGameState` - Client notifies server of apple eaten
- `gameEnd` - Server broadcasts final results

## Data Model

### Player
```javascript
{
  id: string,           // UUID or socket.id
  name: string,         // Player username
  avatarIndex: number,  // 0-7 (prevents avatar conflicts)
  lastActive: timestamp
}
```

### Room
```javascript
{
  id: string,           // 6-char room code
  creatorId: string,    // Player who created room
  config: {
    appleDensity: number,   // 0-1 (% of grid points that have apples)
    timePerTurn: number,    // seconds per match
    mode: string,           // e.g., 'balanced', 'hard'
    gridSize: number        // e.g., 10x10 coordinate system
  },
  status: string,       // 'waiting' | 'playing' | 'ended'
  players: [string],    // Array of player IDs (max 2)
  expiresAt: timestamp  // Null if active, else cleanup deadline
}
```

### Session
```javascript
{
  id: string,
  roomId: string,
  startTime: timestamp,
  status: string,       // 'ongoing' | 'ended'
  applePositions: [{x, y}],  // Pre-generated server-side
  eatenApples: Map<string, player_id>,  // Prevents cheating
  history: [{equation, player_id, timestamp}],
  playerStats: {player_id: {score, time}}
}
```

## Scaling & Future Roadmap

See `docs/scaling_architecture.md` for a detailed roadmap on scaling to **1000 CCU** including:
- Database migration (JSON → MongoDB)
- Redis for room/matchmaking state
- Firebase Auth integration
- Elo rating system
- Horizontal scaling with Socket.io Redis Adapter
- Frontend refactor (React/Vue for non-game screens)

## Important Notes

### Performance Considerations
- **Frontend**: Vanilla JS + Canvas chosen over frameworks to maximize rendering performance
- **Backend**: Node.js + Socket.io handle ~1000 CCU comfortably on modest hardware (2 Core, 4GB RAM)
- **Database**: Current file-based approach is an MVP limitation. With 1000+ CCU, will require MongoDB + Redis.

### Security
- **Authoritative Server**: All critical events (apple consumption, scoring, game end) validated server-side to prevent client-side hacking
- **No Auth**: Current MVP lacks authentication. Future phases will integrate Firebase Auth or similar
- **CORS**: Enabled to allow decoupled frontend/backend on different domains/ports

### Deployment
- **Frontend**: Static CDN (Vercel, Netlify) - no backend needed
- **Backend**: Node.js platform (Render, Railway, Heroku) - must have Node.js runtime
- Remember to update `BACKEND_URL` in frontend code before deploying

## Memory Aid (from Previous Conversations)

- Anti-trivial game rules: Per-degree quota (3 uses for degree ≤ 1) with escalating cooldowns (2s/4s/8s)

## Workflow Rules
- After completing each task, automatically update the 
  "Completed Tasks" and "Current Status" sections in this file
- Document clearly: what was done, which files were modified, 
  any remaining issues or known bugs
- Remind user to run /clear before starting a new task