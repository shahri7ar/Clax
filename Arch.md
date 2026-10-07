# Clax
# ARCHITECTURE.md — ArenaDuel (working title)

> Personal / educational project: a mobile real-time card-battle game (2 players, 1v1, lanes, towers, elixir).
> This file is the single source of truth for structure and rules. AI coding agents MUST read it before making changes and MUST keep it updated when a decision changes.

---

## 1. Goals and non-goals

### Goals
1. Learn how a complete online game is structured: meta server, matchmaking, battle simulation, client.
2. Keep the *structure* of classic private-server card-battle games (account/meta server + separate battle server + data tables), but on a modern stack: 64-bit mobile, current Android target, modern networking, modern engine.
3. Server-authoritative battles with a **deterministic shared simulation** used by both server and client.
4. Fully data-driven content: cards, units, towers, arenas live in data files, not in code.

### Non-goals (for now)
- Clans, chat, friends, spectating, tournaments, events, replays, cosmetics shop, monetization.
- iOS build (Android + desktop editor only at first).
- Perfect anti-cheat. Goal is "server validates everything", not "unhackable".

### Content and legal rules (hard rules)
- **No third-party proprietary assets, code, binary protocols, message IDs, or encryption keys.** Everything is original or properly licensed (CC0 / MIT / own work).
- Original names for all cards, units, towers, arenas, and the game itself.
- Old repositories (e.g. ZrdRoyale, HashRoyale) are *reference reading only*: learn the flows, do not copy code. (They are GPL-3.0; copying would make this project GPL.)
- Track every third-party asset in `docs/ASSETS.md` with license and source link.

---

## 2. Repository layout

```
/
├─ ARCHITECTURE.md
├─ docs/
│  ├─ ASSETS.md            # asset licenses
│  ├─ DEPENDENCIES.md      # every NuGet / addon + why
│  └─ PROTOCOL.md          # generated/maintained from section 8
├─ data/                   # game content (JSON), shared by server + client
│  ├─ rules.json
│  ├─ cards.json
│  ├─ units.json
│  ├─ buildings.json       # towers + spawners
│  ├─ spells.json
│  └─ arenas.json
├─ shared/
│  ├─ Game.Core/           # deterministic simulation (pure C#, NO engine deps)
│  ├─ Game.Core.Tests/     # xUnit tests (headless)
│  └─ Game.Protocol/       # message contracts (MessagePack), shared by server + client
├─ server/
│  ├─ Server.Meta/         # ASP.NET Core: accounts, profile, decks, chests, matchmaking entry
│  ├─ Server.Battle/       # battle host: runs authoritative sim, relays validated commands
│  └─ Server.Tests/
├─ client/                 # Godot 4 (.NET edition) project
├─ tools/                  # data validators, bot runner, load tests
├─ docker-compose.yml      # postgres, redis, meta, battle
└─ .github/workflows/      # CI: build + test
```

---

## 3. Tech stack

| Area | Choice | Notes |
|---|---|---|
| Language | C# everywhere | Lets `Game.Core` be shared by client and server |
| Runtime | **net8.0 for all projects at first** | Match Godot's target framework; upgrade later in one step |
| Server | ASP.NET Core | REST for meta; WebSocket for battle |
| Serialization | MessagePack-CSharp | Compact, fast; contracts in `Game.Protocol` |
| Database | PostgreSQL + EF Core migrations | No manual schema edits, ever |
| Cache/queue | Redis | Matchmaking queue, session tickets |
| Client engine | Godot 4.x **.NET edition** | Android export arm64-v8a (64-bit) |
| Android | latest `targetSdk` required by Google Play at build time | Re-check yearly |
| Tests | xUnit | `dotnet test` must be green before any task is "done" |
| Infra | Docker Compose, GitHub Actions | |

Later (optional): replace WebSocket battle transport with UDP (LiteNetLib) as a learning step.

---

## 4. System overview

```
 ┌────────────┐   REST/HTTPS    ┌─────────────┐      ┌────────────┐
 │  Client    │ ──────────────► │ Server.Meta │ ───► │ PostgreSQL │
 │ (Godot)    │                 │             │ ───► │ Redis      │
 │            │ ◄── ticket ──── │ matchmaking │      └────────────┘
 │            │                 └──────┬──────┘
 │            │   WebSocket            │ creates match + ticket
 │            │ ◄────────────────────► ▼
 │            │                 ┌──────────────┐
 └────────────┘                 │Server.Battle │  runs Game.Core (authoritative)
   runs Game.Core               └──────────────┘
   (same code, same data)
```

- **Server.Meta**: everything outside a battle (login, profile, collection, decks, chests, matchmaking entry, post-battle rewards).
- **Server.Battle**: hosts battles. Holds the authoritative `Game.Core` simulation, validates player commands, assigns execution ticks, broadcasts them, detects desyncs, decides the winner, reports the result to Meta.
- **Client**: UI + rendering + a local copy of `Game.Core` for smooth display. The client never decides outcomes.

---

## 5. Game.Core (deterministic simulation)

Pure C# library. No Godot, no ASP.NET, no I/O, no logging frameworks, no threads.

### 5.1 Determinism rules (MUST follow)
1. **No `float` / `double` anywhere in `Game.Core`.** Use `Fix` (fixed-point struct, Q16.16 stored in `int`, `long` for intermediate math). Provide `Fix`, `FixVec2`, and integer-based sqrt / distance.
2. **No `System.Random`.** Use own PRNG (e.g. xorshift / PCG) seeded from the match seed; state is part of the simulation state.
3. **No `DateTime`, `Stopwatch`, or any wall-clock** inside the simulation. Time = tick count.
4. **No iteration over `Dictionary` / `HashSet` where order matters.** Use `List<T>` / arrays with stable ordering; entity IDs are monotonically increasing integers.
5. **No LINQ in hot paths** where ordering or allocation matters; if used, always with explicit `OrderBy(id)`.
6. All updates happen in a fixed order per tick (section 5.3).
7. The simulation is a pure function: `(State, Commands[tick]) → State'`.

### 5.2 Core concepts
- **Tick rate**: 20 ticks/second (50 ms). Configurable in `rules.json`; changing it is a breaking data change.
- **Arena**: tile grid (default 18 × 32), two halves, a river with bridges, per-tile walkability, deploy zones per player.
- **Entities** (all have `Id`, `OwnerIndex`, `Pos`, `Hp`):
  - `Unit` — moves, targets, attacks (ground/air flag, speed, range, damage, attack cooldown, target preference).
  - `Building` — static; towers (shoot) and spawners.
  - `Spell` — instant or timed area effect.
  - `Projectile` — travels to a target/position, applies damage on arrival.
- **Players**: elixir (fixed-point, max & regen from `rules.json`), 8-card deck, 4-card hand + next card, deterministic cycle order derived from seed.
- **Match phases**: `Regulation` → (`Overtime` if tied) → `Ended`. Durations from `rules.json`.
- **Win condition**: more towers destroyed; tiebreak = lowest remaining tower HP ratio; then draw. (Rules are data/config, adjust in `rules.json` + a single `WinRule` class.)

### 5.3 Per-tick update order
1. Apply scheduled commands for this tick (in order of `(tick, playerIndex, seq)`).
2. Regenerate elixir.
3. Spawn queued entities (cards deployed earlier whose deploy delay ended).
4. Update targeting (units, towers).
5. Move units (pathing on grid, separation / collision rules).
6. Resolve attacks → create projectiles / apply damage.
7. Update projectiles and apply their damage.
8. Update spells and status effects.
9. Remove dead entities (stable order by `Id`).
10. Check phase transitions and win condition.
11. Increment tick.

### 5.4 Commands
All player input is a `Command`; the sim has no other input channel.
```
PlaceCard { PlayerIndex, HandSlot (0..3), X, Y (tile coords) }
Emote     { PlayerIndex, EmoteId }          // cosmetic, ignored by sim state hash
Surrender { PlayerIndex }
```
Validation (in `Game.Core`, used by server as authority and client as a hint):
elixir sufficient, tile in own deploy zone (or spell-legal zone), slot valid, match not ended.

### 5.5 State hash and snapshots
- `StateHasher` computes a 64-bit hash of all sim-relevant state; called every N ticks (default 20).
- `Snapshot` / `Restore` for full state serialization (used for reconnect and desync recovery).
- Golden tests: a recorded command list + seed must always produce the same final hash on every platform.

---

## 6. Data model (data-driven content)

All gameplay numbers live in `/data/*.json`. Code contains **rules and mechanics**, data contains **values**.

### 6.1 Conventions
- IDs are lowercase snake_case strings (`"stone_guard"`), mapped to integer indexes at load time for speed.
- Numeric values that feed the simulation are stored as **integers** (e.g. speed in "milli-tiles per second", HP as int); `Game.Core` converts to `Fix` at load.
- `/data/manifest` (generated) contains a **content hash** of all data files. Client sends it in `Hello`; Battle server rejects a mismatch. This prevents the classic client/server data-desync bug.
- `tools/DataValidator` runs in CI: schema check, unknown references, duplicate IDs, sanity ranges.

### 6.2 Examples

`cards.json`
```json
{
  "cards": [
    {
      "id": "stone_guard",
      "type": "unit",
      "elixirCost": 3,
      "rarity": "common",
      "spawns": { "unit": "stone_guard_unit", "count": 1 },
      "deployDelayTicks": 20
    }
  ]
}
```

`units.json`
```json
{
  "units": [
    {
      "id": "stone_guard_unit",
      "hp": 1200,
      "speedMilliTilesPerSec": 1000,
      "targets": "ground",
      "targetPreference": "any",
      "attack": {
        "damage": 110,
        "rangeMilliTiles": 1200,
        "cooldownTicks": 24,
        "projectile": null
      },
      "hitboxMilliTiles": 500,
      "flying": false,
      "levelScaling": { "hpPercentPerLevel": 10, "damagePercentPerLevel": 10 }
    }
  ]
}
```

`rules.json`
```json
{
  "ticksPerSecond": 20,
  "regulationSeconds": 180,
  "overtimeSeconds": 120,
  "elixir": { "max": 10, "start": 5, "regenSecondsPerUnit": 2800, "overtimeMultiplier": 2 },
  "hand": { "deckSize": 8, "handSize": 4 },
  "arena": { "defaultArenaId": "arena_01" }
}
```
(All values are tunable placeholders.)

`buildings.json` defines towers (HP, damage, range, cooldown) and positions come from `arenas.json`.

---

## 7. Meta server (Server.Meta)

### 7.1 Responsibilities
Guest auth, profile, card collection and levels, decks, chests, trophies/arenas, matchmaking entry, receiving battle results.

### 7.2 Auth
- `POST /auth/guest` with a client-generated device ID → creates/returns a player and a JWT.
- Account linking (Google/Apple) is out of scope for now.

### 7.3 REST endpoints (v1)
| Method | Path | Purpose |
|---|---|---|
| POST | `/auth/guest` | Create/login guest, returns JWT |
| GET | `/me` | Profile, trophies, currency |
| GET | `/me/cards` | Collection and levels |
| PUT | `/me/decks/{slot}` | Set a deck (validated: 8 owned cards) |
| GET | `/data/manifest` | Data content hash + version |
| POST | `/matchmaking/queue` | Enter queue (deck slot) → returns `ticketId` |
| GET | `/matchmaking/status/{ticketId}` | Poll: waiting / matched (battle URL + ticket) |
| POST | `/chests/{id}/open` | Open a chest, grant rewards |
| POST | `/internal/battle-result` | Battle server → Meta (service-to-service, API key) |

### 7.4 Database (PostgreSQL, EF Core migrations)
```
players(id, device_id, name, trophies, gold, gems, created_at)
player_cards(player_id, card_id, level, copies)
decks(player_id, slot, card_ids[8])
chests(id, player_id, type, state, unlock_at, rewards_json)
matches(id, player_a, player_b, seed, started_at, ended_at, winner_index, trophy_delta_a, trophy_delta_b, final_state_hash)
```
All economy changes happen in a transaction on the server. The client never sends "I won" or amounts.

### 7.5 Matchmaking (initial)
Redis list/sorted set keyed by trophy range; pair the first compatible two; if waiting > N seconds, match a **bot** (a bot is just a Command generator running on the Battle server). Meta creates `matchId`, `seed`, and per-player one-time `battleTicket`s.

---

## 8. Battle server and protocol

### 8.1 Model: server-validated deterministic lockstep
- The Battle server runs the authoritative sim at 20 Hz.
- Clients send `PlaceCard`. The server validates it against its own state, assigns an **execution tick** = `currentServerTick + inputDelayTicks` (default 3), and broadcasts it to both players.
- Clients apply commands **only at the announced execution tick** on their local `Game.Core` copy. Both sims stay identical.
- Every `hashIntervalTicks` the server sends its state hash; clients compare. On mismatch the server sends a `Resync` snapshot.
- The server alone decides the winner, from its own sim.

### 8.2 Transport
WebSocket (binary frames), TLS in production. MessagePack payloads. First byte/union tag = message type. Heartbeat ping/pong every 5 s; drop after 15 s silence; allow reconnect with the same ticket within 30 s (server sends a snapshot).

### 8.3 Messages (contracts in `Game.Protocol`)

Client → Server
| Message | Fields |
|---|---|
| `Hello` | ticket, protocolVersion, dataHash |
| `PlaceCardCmd` | clientSeq, handSlot, x, y |
| `EmoteCmd` | emoteId |
| `HashReport` | tick, hash |
| `Ping` | clientTime |
| `Leave` | — |

Server → Client
| Message | Fields |
|---|---|
| `HelloAck` | playerIndex, serverTick |
| `BattleStart` | startTick, seed, arenaId, decks (both), tickRate |
| `CommandApplied` | executeTick, playerIndex, command |
| `CommandRejected` | clientSeq, reason |
| `HashCheck` | tick, hash |
| `Resync` | tick, snapshot |
| `BattleEnd` | winnerIndex, finalHash, rewardsPreview |
| `Pong` | clientTime, serverTime |

### 8.4 Battle lifecycle
1. Meta creates match → both clients get battle URL + ticket.
2. Clients connect, send `Hello`; server checks ticket, protocol version, data hash.
3. When both are connected: server sends `BattleStart` with a start tick slightly in the future.
4. Loop: commands → validation → `CommandApplied` → sim tick → periodic `HashCheck`.
5. On end: server sends `BattleEnd`, posts result to Meta (`/internal/battle-result`), closes.
6. Disconnect > grace period = forfeit.

---

## 9. Client (Godot 4, .NET)

### 9.1 Principles
- **Presentation only.** The client renders state from `Game.Core`; it contains no game rules.
- Rendering interpolates between the previous and current tick using `alpha = timeSinceTick / tickDuration`.
- Visual nodes are created from sim entities via a `View` layer (`EntityId → Node` map); sim never references Godot types.
- Network code and sim stepping live in plain C# classes; Godot scripts are thin glue.

### 9.2 Scenes (initial)
```
Boot            → loads data, checks manifest, guest login
MainMenu        → profile, deck, Battle button, chests
DeckBuilder
Matchmaking     → queue + cancel
Battle          → arena view + HUD (elixir bar, hand, next card, timer)
Result          → outcome, rewards
```
Autoloads: `Session` (JWT, profile), `GameData` (loaded JSON + hash), `NetClient`.

### 9.3 Screen / aspect-ratio rules (to avoid the legacy-client stretching problem)
- Portrait game. Design resolution 1080 × 1920.
- Project setting: stretch mode `canvas_items`, aspect `expand`.
- All UI uses `Control` anchors / containers; no absolute pixel positions for layout.
- Arena camera is orthographic-style: **fit arena width**; on taller screens extend background art above/below; on wider/shorter screens add side margins. Never stretch non-uniformly.
- Respect device safe area via `DisplayServer.GetDisplaySafeArea()` for HUD placement (notches, rounded corners).
- Test matrix: 16:9, 18:9, 19.5:9, 20:9, 21:9, 4:3 (tablet).

### 9.4 Android export
- ABI: `arm64-v8a` (64-bit). `armeabi-v7a` optional.
- Gradle build, current `targetSdk`, release keystore stored outside the repo.
- Verify the Godot C# Android export support level in the official docs for the Godot version used.

---

## 10. Testing strategy

1. **Game.Core unit tests**: Fix math, PRNG, targeting, pathing, damage, elixir, win rules.
2. **Determinism tests**: same seed + commands ⇒ identical hash across runs; hash stored as golden values.
3. **Replay tests**: recorded command logs in `/shared/Game.Core.Tests/Replays/*.json` replayed in CI.
4. **Bot-vs-bot soak**: `tools/BotRunner` runs N thousand headless matches; asserts no exceptions, matches end, hashes match between two sim instances.
5. **Server tests**: command validation, ticket handling, reconnect, desync recovery.
6. **Data validation** in CI (`tools/DataValidator`).
7. Manual: aspect-ratio matrix and a real arm64 device before each milestone is closed.

---

## 11. Milestones (each has acceptance criteria)

| # | Milestone | Done when |
|---|---|---|
| M0 | Repo skeleton | Solution builds, CI runs `dotnet test`, Docker Compose starts Postgres + Redis |
| M1 | Game.Core v0 | `Fix` math + PRNG + arena grid + 1 unit type + 1 tower type; headless console sim runs a full match between two scripted bots; golden hash test passes |
| M2 | Game.Core v1 | Elixir, hand/deck cycle, `PlaceCard` validation, targeting, projectiles, spells (1), win rules; BotRunner soak passes |
| M3 | Offline client | Godot battle scene with placeholder shapes; play vs bot locally on `Game.Core`; correct on the aspect-ratio test matrix |
| M4 | Meta server | Guest auth, profile, cards, decks, migrations; client Boot → MainMenu → DeckBuilder works |
| M5 | Online battle | Matchmaking + Battle server + protocol; two clients play; hash checks pass; resync works |
| M6 | Progression | Trophies, arenas, chests, card levels, rewards (server-side only) |
| M7 | Content + art pass | 8+ cards, 2+ arenas, original/licensed art and audio, arm64 Android release build |

Do not start milestone N+1 before N's acceptance criteria are met.

---

## 12. Rules for AI coding agents

1. Read this file first. If a task conflicts with it, stop and propose a change to this file rather than silently deviating.
2. Work in **small tasks** (one feature or fix per change). Describe the plan in 3–6 lines before coding.
3. Always run `dotnet build` and `dotnet test` before saying a task is done; report the actual output.
4. In `Game.Core`: no floats, no `System.Random`, no wall-clock, no dictionary-order dependence. If unsure, add a determinism test.
5. Never put gameplay numbers in code; add them to `/data` and validate them.
6. Never add a dependency without recording it in `docs/DEPENDENCIES.md`.
7. Never commit secrets, keystores, or `.env` files.
8. Never use or recreate third-party proprietary assets, names, or protocols (section 1).
9. Server decides outcomes. The client must never send results, currency amounts, or trophy changes.
10. Database changes only through EF Core migrations.
11. Keep Godot scripts thin; logic belongs in plain C# classes that can be unit-tested.
12. Commit after every working step with a clear message.

---

## 13. Open decisions (fill in as they are made)

- [ ] Final game name and art direction.
- [ ] Number of arenas / card pool size for the first playable.
- [ ] Bot difficulty model (scripted vs simple heuristic).
- [ ] When to move battle transport from WebSocket to UDP.
- [ ] Hosting target for the servers (local only vs VPS).
