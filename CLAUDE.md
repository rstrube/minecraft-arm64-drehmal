# CLAUDE.md — minecraft-arm64-drehmal

This file provides context for Claude Code and any AI assistant working on this project. It summarizes the project's purpose, architecture decisions, hardware context, and rationale for key choices made during setup.

---

## Project Overview

This project provides a Dockerized Minecraft Java Edition server running the **Drehmal: APOTHEOSIS** adventure map, designed to run on ARM64 single-board computers (SBCs). The server is built around the [`itzg/docker-minecraft-server`](https://github.com/itzg/docker-minecraft-server) image and uses Fabric modloader with a curated set of server-side performance mods.

The project is hosted on GitHub so that configuration changes (primarily `docker-compose.yml`) can be version controlled and pulled directly to the server without manual file copying. The world data itself is not included in the repository due to its size — setup instructions for downloading and extracting it are in the README.

---

## Hardware Context

The primary target hardware is the **Raspberry Pi** (Pi 4 and Pi 5 family), running a Debian-based Linux distro (Raspberry Pi OS or Ubuntu) with Docker installed. Any ARM64 SBC meeting those requirements should work.

The `docker-compose.yml` includes four hardware tiers targeting common Raspberry Pi configurations. Only one tier should be active at a time.

The project author runs an **Odroid N2+** (4GB RAM, 6-core Cortex-A73/A53), which fits at Tier 3 — the same RAM allocation and settings as the Raspberry Pi 5 (4GB). The N2+'s additional cores provide a meaningful benefit via C2ME's parallel chunk loading.

---

## Minecraft & Map Details

- **Map:** Drehmal: APOTHEOSIS v2.2.2f
- **Minecraft version:** Java Edition 1.20.1
- **Modloader:** Fabric
- **Fabric Loader version:** 0.15.11
- **Map size:** 12,000 x 12,000 blocks
- **Max recommended players:** 8
- **Map download:** https://www.drehmal.net/downloads (World File — Google Drive)

Drehmal is a massive open-world adventure map with heavy emphasis on exploration, survival, and narrative. It is 100% multiplayer compatible. All Drehmal mods are client-side except for the performance mods listed below.

---

## Docker Image

```
itzg/minecraft-server:java21
```

The `java21` tag is used explicitly rather than `latest` because:

- Minecraft 1.20.1 officially targets Java 21
- The `latest` tag tracks the newest Java version (currently Java 25) which breaks mod compatibility
- Using a specific tag prevents unexpected breakage on image updates

---

## Directory Structure

The repo is cloned into `/opt` on the SBC, giving a root directory of `/opt/minecraft-arm64-drehmal`:

```
/opt/minecraft-arm64-drehmal/
├── docker-compose.yml
├── mods/
│   ├── fabric-api-0.92.2+1.20.1.jar
│   ├── lithium-fabric-mc1.20.1-0.11.2.jar
│   ├── c2me-fabric-mc1.20.1-0.2.0+alpha.11.5.jar
│   ├── modernfix-fabric-5.18.1+mc1.20.1.jar
│   ├── ferritecore-6.0.1-fabric.jar
│   ├── memoryleakfix-fabric-1.17+-1.1.5.jar
│   ├── starlight-1.1.2+fabric.dbc156f.jar
│   └── lazydfu-0.1.3.jar
├── data/               # World data, server configs, logs (not in git)
│   └── world/          # Extracted Drehmal world folder
├── CLAUDE.md
└── README.md
```

The `data/` directory is created manually after cloning and is not committed to git. The world file is downloaded separately and extracted into `data/world/`.

---

## Server-Side Mods

All mods are for **Minecraft 1.20.1 / Fabric** and are stored in the `mods/` directory of this repository. They are the server-side subset of the [Drehmal 2.2 mod list](https://www.drehmal.net/2-2-mod-list). All other mods on the Drehmal list are client-side rendering/visual mods that players install locally via the Drehmal installer.

| Mod | Version | Purpose |
|---|---|---|
| Fabric API | 0.92.2+1.20.1 | Required by all Fabric mods |
| Lithium | 0.11.2 | General server-side game logic optimization |
| C2ME | 0.2.0+alpha.11.5 | Concurrent chunk loading across multiple CPU cores |
| FerriteCore | 6.0.1 | Memory usage optimization |
| ModernFix | 5.18.1 | Performance, memory, and bug fixes |
| Memory Leak Fix | 1.1.5 | Fixes various memory leaks |
| Starlight | 1.1.2 | Rewrites lighting engine for better performance |
| Lazy DFU | 0.1.3 | Delays DataFixerUpper initialization, reduces startup time |

### Notes on specific mods

**C2ME** is particularly beneficial on multi-core SBCs because it parallelizes chunk loading across all available CPU cores. Drehmal's 12k x 12k world means chunk loading is a significant workload. C2ME is an alpha build for 1.20.1 — if startup issues occur, it is the first mod to try removing.

**Starlight** has been archived by its author (March 2024) as Mojang improved their own lighting engine in newer Minecraft versions. For 1.20.1 it remains the correct choice. The recommended successor, **ScalableLux**, only supports 1.21+.

**Lazy DFU** significantly reduces server startup time by deferring DataFixerUpper (DFU) initialization, which Minecraft uses for world data migration. On a large world like Drehmal this makes a noticeable difference.

---

## UID / GID

The `docker-compose.yml` has UID/GID commented out by default for portability. If set, the container runs the Minecraft process as that user, meaning files written to `data/` will be owned by that user rather than root. To find your values:

```bash
id -u   # UID
id -g   # GID
```

Without UID/GID set, files in the data directory will be owned by root and will require `sudo` to interact with directly. For a personal server this is a minor inconvenience rather than a real problem.

---

## Key Docker Commands

```bash
# Start the server
docker compose up -d

# Stop the server
docker compose down

# View live logs
docker logs -f minecraft-drehmal

# Stop viewing logs (does NOT affect the server)
# Press: Ctrl+C

# Attach to server console (interact with Minecraft server directly)
docker attach minecraft-drehmal

# Detach from console WITHOUT stopping the server
# Press: Ctrl+P then Ctrl+Q
# WARNING: Do NOT press Ctrl+C while attached — this will stop the server
```

---

## Hardware Tier Reference

The `docker-compose.yml` includes four hardware tiers. Only one should be active at a time. `MEMORY`, `JVM_OPTS`, `VIEW_DISTANCE`, and `SIMULATION_DISTANCE` are all grouped together per tier.

| Tier | RAM | MEMORY | VIEW_DISTANCE | SIMULATION_DISTANCE | Example Hardware |
|---|---|---|---|---|---|
| 1 | 2GB | 1G | 6 | 4 | Raspberry Pi 4 (2GB) |
| 2 | 4GB | 2G | 7 | 5 | Raspberry Pi 4 (4GB) |
| 3 | 4GB | 3G | 8 | 6 | Raspberry Pi 5 (4GB) ← default |
| 4 | 8GB | 6G | 10 | 8 | Raspberry Pi 5 (8GB) |

The Odroid N2+ (4GB) fits Tier 3.

`MEMORY` is set below total RAM to leave ~1GB headroom for the OS. `SIMULATION_DISTANCE` should always be ≤ `VIEW_DISTANCE`. For an adventure map like Drehmal, lower simulation distance has minimal gameplay impact since players are exploring rather than running farms.

---

## Client Setup

Players connecting to the server must have the Drehmal client installed locally. All visual/rendering mods are client-side and are not present on the server. Players should:

1. Download and run the Drehmal installer from https://www.drehmal.net/downloads
2. The installer sets up Fabric 1.20.1, all client mods, and the resource pack automatically
3. Connect to the server IP on port 25565

---

## Why GitHub

The repository exists to:

- Version control the `docker-compose.yml` so configuration changes are tracked
- Allow updates to be pulled directly to the server with `git pull` rather than manual file copying
- Provide a reference for setting up the server on other ARM64 SBCs
- Document the rationale for mod and configuration choices for future reference

The world file is excluded from the repository (too large for git) and must be downloaded separately from the Drehmal website.

---

## Known Issues & Notes

- The server produces a `Can't keep up!` warning during initial startup while loading the Drehmal world. This is expected due to the world's size and resolves once the spawn area is fully loaded.
- C2ME is an alpha build — if the server fails to start, try removing it from the `mods/` directory first to isolate the issue.
- Starlight is archived and will not receive further updates for 1.20.1. It is stable and safe to use.
- The Drehmal world file is downloaded via Google Drive. If the SBC has no browser, download it on a PC and `scp` it to the SBC before extracting.