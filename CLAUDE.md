# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dockerized Minecraft Java Edition server running the **Drehmal: APOTHEOSIS** adventure map (v2.2.2f) on ARM64 SBCs. Built on [`itzg/docker-minecraft-server`](https://github.com/itzg/docker-minecraft-server) with Fabric modloader. The only file that changes in normal use is `docker-compose.yml`.

---

## Key Constraints

- **Mod versions are fixed** — they must match the [Drehmal 2.2 mod list](https://www.drehmal.net/2-2-mod-list) exactly. Do not upgrade mod versions without verifying Drehmal compatibility.
- **Minecraft version is 1.20.1** — locked by Drehmal.
- **Docker image tag is `java21`**, not `latest` — `latest` now tracks Java 25+, which breaks mod compatibility.
- **Server settings that must not change:** `ENABLE_COMMAND_BLOCK: true` (Drehmal uses command blocks for story/mechanics), `SPAWN_PROTECTION: 0` (players must interact with the Drehmal spawn area), `MAX_PLAYERS: 8` (Drehmal's recommended maximum).

---

## Hardware Tiers

`docker-compose.yml` has four commented hardware tiers — **only one should be active at a time**. When switching tiers, comment out the current tier's `MEMORY`, `JVM_OPTS`, `VIEW_DISTANCE`, and `SIMULATION_DISTANCE`, then uncomment the new tier's block. Tier 3 is the default (active/uncommented).

| Tier | RAM | MEMORY | VIEW_DISTANCE | SIMULATION_DISTANCE | Example Hardware |
|---|---|---|---|---|---|
| 1 | 2GB | 1G | 6 | 4 | Raspberry Pi 4 (2GB) |
| 2 | 4GB | 2G | 7 | 5 | Raspberry Pi 4 (4GB) |
| 3 | 4GB | 3G | 8 | 6 | Raspberry Pi 5 (4GB) ← default |
| 4 | 8GB | 6G | 10 | 8 | Raspberry Pi 5 (8GB) |

`MEMORY` is kept below total RAM to leave ~1GB headroom for the OS. `SIMULATION_DISTANCE` must always be ≤ `VIEW_DISTANCE`.

The project author runs an **Odroid N2+** (4GB, 6-core Cortex-A73/A53), which fits Tier 3.

---

## Directory Structure

```
/opt/minecraft-arm64-drehmal/   ← deploy path on the SBC
├── docker-compose.yml          ← only version-controlled config file
├── mods/                       ← server-side mod JARs (version-controlled)
├── data/                       ← world data, server configs, logs (gitignored)
│   └── world/                  ← extracted Drehmal world folder
└── original-world-zip/         ← original Drehmal zip kept for re-extraction (gitignored)
```

The `data/` and `original-world-zip/` directories are gitignored and must be created and populated manually on each new deployment.

---

## Docker Commands

```bash
# Start the server
docker compose up -d

# Stop the server
docker compose down

# Pull config updates and restart
git pull && docker compose down && docker compose up -d

# View live logs (Ctrl+C to stop — does NOT stop the server)
docker logs -f minecraft-drehmal

# Attach to server console (type Minecraft commands directly)
docker attach minecraft-drehmal
# Detach WITHOUT stopping: Ctrl+P then Ctrl+Q
# WARNING: Ctrl+C while attached WILL stop the server
```

---

## Server-Side Mods

Mods are the server-side subset of the Drehmal 2.2 mod list. All other Drehmal mods are client-side and installed by players via the Drehmal installer.

| Mod | Version | Purpose |
|---|---|---|
| Fabric API | 0.92.6+1.20.1 | Required by all Fabric mods |
| Lithium | 0.11.4 | General server-side game logic optimization |
| FerriteCore | 6.0.1 | Memory usage optimization |
| ModernFix | 5.24.3 | Performance, memory, and bug fixes |
| Memory Leak Fix | 1.1.5 | Fixes various memory leaks |
| Starlight | 1.1.2 | Rewrites lighting engine for better performance |
| Lazy DFU | 0.1.3 | Delays DataFixerUpper initialization, reduces startup time |

**Starlight** is archived (March 2024) — it will never be updated for 1.20.1. It is stable and correct for this version. The successor, ScalableLux, only supports 1.21+.

### Re-downloading mods

```bash
cd /opt/minecraft-arm64-drehmal/mods

wget "https://cdn.modrinth.com/data/P7dR8mSH/versions/UapVHwiP/fabric-api-0.92.6%2B1.20.1.jar"
wget "https://cdn.modrinth.com/data/gvQqBUqZ/versions/iEcXOkz4/lithium-fabric-mc1.20.1-0.11.4.jar"
wget "https://cdn.modrinth.com/data/nmDcB62a/versions/Qt5OXLYh/modernfix-fabric-5.24.3%2Bmc1.20.1.jar"
wget "https://cdn.modrinth.com/data/uXXizFIs/versions/unerR5MN/ferritecore-6.0.1-fabric.jar"
wget "https://cdn.modrinth.com/data/NRjRiSSD/versions/5xvCCRjJ/memoryleakfix-fabric-1.17%2B-1.1.5.jar"
wget "https://cdn.modrinth.com/data/H8CaAYZC/versions/XGIsoVGT/starlight-1.1.2%2Bfabric.dbc156f.jar"
wget "https://cdn.modrinth.com/data/hvFnDODi/versions/0.1.3/lazydfu-0.1.3.jar"
```

---

## UID / GID

`UID` and `GID` are commented out in `docker-compose.yml` by default. Without them, files written to `data/` are owned by root and require `sudo` to interact with. Set them to run the container as a specific user (`id -u` / `id -g`).

---

## Known Issues

- `Can't keep up!` warnings appear during initial startup while the large Drehmal world loads — expected, resolves once the spawn area is fully prepared.
- The Drehmal world download is a Google Drive link. On a headless SBC, download on a PC and `scp` to the SBC.
