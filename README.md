# ArenaDuel (working title)

A personal, educational project: a mobile real-time 1v1 card-battle game with towers, lanes and elixir, built to learn how a complete online game fits together — meta server, matchmaking, battle server, deterministic simulation and a mobile client.

> **Status:** early development. Currently working on **M0 – repo skeleton**. See [Milestones](#milestones).

---

## Highlights (planned)

- **Server-authoritative battles** driven by a **deterministic shared simulation** (`Game.Core`) that runs identically on the server and the client.
- **Data-driven content:** cards, units, towers and arenas live in JSON files, not in code. A content hash prevents client/server data mismatches.
- **Modern stack:** .NET, ASP.NET Core, PostgreSQL, Redis, Godot 4 (C#), 64-bit Android (arm64-v8a).
- **Resolution-independent UI:** portrait layout designed to behave on 16:9 through 21:9 screens and tablets.

For the full design, read **[ARCHITECTURE.md](ARCHITECTURE.md)** — it is the source of truth for structure, rules and the protocol.

---

## Repository layout

```
.
├─ ARCHITECTURE.md        Design and rules (read this first)
├─ docs/                  Assets/licenses, dependencies, protocol notes
├─ data/                  Game content (JSON) shared by client and server
├─ shared/
│  ├─ Game.Core/          Deterministic simulation (pure C#, no engine deps)
│  ├─ Game.Core.Tests/    Headless unit and determinism tests
│  └─ Game.Protocol/      Network message contracts (MessagePack)
├─ server/
│  ├─ Server.Meta/        Accounts, profile, decks, chests, matchmaking entry
│  ├─ Server.Battle/      Authoritative battle host
│  └─ Server.Tests/
├─ client/                Godot 4 (.NET edition) project
├─ tools/                 Data validator, bot runner, load tests
└─ docker-compose.yml     PostgreSQL + Redis (and later the servers)
```

---

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| [.NET SDK](https://dotnet.microsoft.com/download) | 8.0 | All projects target `net8.0` for now |
| [Godot](https://godotengine.org/download) | 4.x, **.NET edition** | The standard (non-.NET) build cannot run C# projects |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | recent | For PostgreSQL and Redis |
| Git | recent | |

For Android builds you will additionally need the Android SDK, JDK, and a keystore (kept **outside** the repo). See the Godot docs for exporting C# projects to Android.

---

## Getting started

Run these from the repository root.

### 1. Build and test

```bash
dotnet build
dotnet test
```

### 2. Start the infrastructure

```bash
cp .env.example .env        # then edit values if needed
docker compose up -d        # PostgreSQL + Redis
```

### 3. Run the servers

```bash
dotnet run --project server/Server.Meta
dotnet run --project server/Server.Battle
```

Both expose `GET /health` (returns `200 OK`). Check `Properties/launchSettings.json` in each project for the local ports.

### 4. Open the client

Open the `client/` folder in the **Godot .NET** editor, let it build the C# solution, and press Play.

---

## Milestones

| # | Milestone | Goal |
|---|---|---|
| M0 | Repo skeleton | Solution builds, CI green, Docker Compose up |
| M1 | Game.Core v0 | Fixed-point math, PRNG, arena grid, one unit, one tower, headless bot match |
| M2 | Game.Core v1 | Elixir, deck cycle, validation, targeting, projectiles, spell, win rules |
| M3 | Offline client | Godot battle scene vs bot, correct on all aspect ratios |
| M4 | Meta server | Guest auth, profile, cards, decks, migrations |
| M5 | Online battle | Matchmaking, battle server, protocol, desync detection and resync |
| M6 | Progression | Trophies, arenas, chests, card levels |
| M7 | Content + art | Cards, arenas, original art/audio, arm64 Android release |

A milestone is closed only when its acceptance criteria in `ARCHITECTURE.md` are met.

---

## Development rules (short version)

- `Game.Core` must stay deterministic: **no floats, no `System.Random`, no wall-clock time, no order-dependent collections.**
- Gameplay numbers belong in `data/`, never in code.
- The server decides outcomes; the client never sends results, currency or trophy changes.
- Database changes only through EF Core migrations.
- `dotnet build` and `dotnet test` must pass before a task is considered done.
- Record every dependency in `docs/DEPENDENCIES.md` and every asset in `docs/ASSETS.md`.

The full list (including rules for AI coding agents) is in section 12 of `ARCHITECTURE.md`.

---

## Working with AI coding agents

This project is designed to be built with AI assistance. Suggested workflow:

1. Point the agent at `ARCHITECTURE.md` first.
2. Give it **one milestone or one small task at a time**.
3. Require real `dotnet build` / `dotnet test` output before accepting a result.
4. Run and play-test everything yourself — game feel, art and on-device behavior need a human.
5. Commit after every working step.

---

## Content and legal notes

- All names, art, audio, code and protocols in this project are original or properly licensed. **No third-party proprietary assets, code, binary protocols or keys are used.**
- Other open-source private-server projects may have been studied as *reference reading only*; no code was copied from them.
- Third-party assets are listed with their licenses in [`docs/ASSETS.md`](docs/ASSETS.md).

---

## License

To be decided. Until a `LICENSE` file is added, all rights are reserved by the author.
