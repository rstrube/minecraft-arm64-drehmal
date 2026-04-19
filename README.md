# minecraft-arm64-drehmal

A Dockerized Minecraft Java Edition server running the **Drehmal: APOTHEOSIS** adventure map, designed to run on ARM64 single-board computers (SBCs). Built around [`itzg/docker-minecraft-server`](https://github.com/itzg/docker-minecraft-server) with Fabric modloader and a curated set of server-side performance mods.

---

## Overview

- **Map:** [Drehmal: APOTHEOSIS v2.2.2f](https://www.drehmal.net/) — a massive 12,000 x 12,000 block open-world adventure map with heavy emphasis on exploration, survival, and narrative. 100% multiplayer compatible for up to 8 players.
- **Minecraft version:** Java Edition 1.20.1
- **Modloader:** Fabric (loader 0.15.11)
- **Docker image:** `itzg/minecraft-server:java21`

---

## Assumptions

- You have an ARM64 SBC running a Debian-based Linux distro (e.g. Ubuntu)
- You can SSH into the SBC and issue `sudo` commands
- Docker is installed and your user is part of the `docker` group

---

## Setup

### 1. Clone the repo

```bash
cd /opt
sudo git clone https://github.com/rstrube/minecraft-arm64-drehmal
sudo chown -R $USER:$USER /opt/minecraft-arm64-drehmal
mkdir -p /opt/minecraft-arm64-drehmal/data/world
```

*The rest of these instructions assume the installation path is `/opt/minecraft-arm64-drehmal`.*

### 2. Download the Drehmal world file

Download the latest [Drehmal: APOTHEOSIS](https://www.drehmal.net/downloads) world `.zip` file from the downloads page (World File — Google Drive link).

> **Note:** Because it's a Google Drive link, if your SBC has no browser you'll need to download it on a PC and `scp` it across:

```bash
scp "Drehmal APOTHEOSIS 2.2.2f.zip" user@sbc:/opt/minecraft-arm64-drehmal/.
```

### 3. Extract the world file

```bash
cd /opt/minecraft-arm64-drehmal
unzip "Drehmal APOTHEOSIS 2.2.2f.zip" -d data/world
```

> **Note:** If `unzip` is not installed: `sudo apt install unzip`

### 4. Configure the Docker Compose file

The `docker-compose.yml` has multiple hardware tiers defined within it — Tier 3 is uncommented by default (Odroid N2+ / 4GB RAM). Uncomment the tier that matches your hardware for best performance. See the [Hardware Tiers](#hardware-tiers) section below for details.

Optionally, set the `UID` and `GID` environment variables to the user that will run the Docker container. If not set, logs and changes to the `data/` directory will be owned by `root`. To find your values:

```bash
id -u   # UID
id -g   # GID
```

### 5. Start the server

```bash
cd /opt/minecraft-arm64-drehmal
docker compose up -d
```

On first startup the server will take a minute or two to load the Drehmal world. You may see a `Can't keep up!` warning during this initial load — this is expected and will resolve once the spawn area is fully prepared.

---

## Managing the Server

### View live logs

```bash
docker logs -f minecraft-drehmal
```

Press `Ctrl+C` to stop viewing logs. This does **not** affect the running server.

### Attach to the server console

```bash
docker attach minecraft-drehmal
```

This drops you into the live Minecraft server console where you can type commands directly (e.g. `list`, `op username`, `save-all`).

To detach **without stopping the server**, press `Ctrl+P` then `Ctrl+Q`.

> **Warning:** Do **not** press `Ctrl+C` while attached — this will stop the server.

### Stop the server

```bash
docker compose down
```

### Pull config updates from GitHub

```bash
cd /opt/minecraft-arm64-drehmal
git pull
docker compose down
docker compose up -d
```

---

## Client Setup

Players connecting to the server must have the Drehmal client installed locally. All visual and rendering mods are client-side and are not present on the server. Each player should:

1. Download and run the Drehmal installer from https://www.drehmal.net/downloads
2. The installer automatically sets up Fabric 1.20.1, all client mods, and the resource pack
3. Connect to the server IP on port `25565`

---

## Hardware Tiers

The `docker-compose.yml` includes four hardware tiers. Only one should be active at a time. `MEMORY`, `JVM_OPTS`, `VIEW_DISTANCE`, and `SIMULATION_DISTANCE` are all grouped together per tier.

| Tier | RAM | MEMORY | VIEW_DISTANCE | SIMULATION_DISTANCE | Example Hardware |
|---|---|---|---|---|---|
| 1 | 2GB | 1G | 6 | 4 | Pi 4 2GB |
| 2 | 3-4GB | 2G | 7 | 5 | Pi 4 4GB |
| 3 | 4GB | 3G | 8 | 6 | Odroid N2+ ← default |
| 4 | 6-8GB | 5G | 10 | 8 | Pi 5 8GB |

`MEMORY` is set below total RAM to leave ~1GB headroom for the OS. `SIMULATION_DISTANCE` should always be ≤ `VIEW_DISTANCE`. For an adventure map like Drehmal, lower simulation distance has minimal gameplay impact since players are exploring rather than running farms.

---

## Server-Side Mods

The `mods/` directory contains the server-side subset of the [Drehmal 2.2 mod list](https://www.drehmal.net/2-2-mod-list). All other mods on the Drehmal list are client-side rendering/visual mods installed locally by players via the Drehmal installer.

| Mod | Version | Purpose |
|---|---|---|
| Fabric API | 0.92.6+1.20.1 | Required by all Fabric mods |
| Lithium | 0.11.4 | General server-side game logic optimization |
| C2ME | 0.2.0+alpha.11.16 | Concurrent chunk loading across multiple CPU cores |
| FerriteCore | 6.0.1 | Memory usage optimization |
| ModernFix | 5.24.3 | Performance, memory, and bug fixes |
| Memory Leak Fix | 1.1.5 | Fixes various memory leaks |
| Starlight | 1.1.2 | Rewrites lighting engine for better performance |
| Lazy DFU | 0.1.3 | Delays DataFixerUpper initialization, reduces startup time |

### Notes on specific mods

**C2ME** is particularly beneficial on multi-core SBCs because it parallelizes chunk loading across all available CPU cores. Drehmal's 12k x 12k world means chunk loading is a significant workload. C2ME is an alpha build for 1.20.1 — if the server fails to start, it is the first mod to try removing to isolate the issue.

**Starlight** has been archived by its author (March 2024) as Mojang improved their own lighting engine in newer Minecraft versions. For 1.20.1 it remains the correct and stable choice. The recommended successor, **ScalableLux**, only supports 1.21+.

**Lazy DFU** significantly reduces server startup time by deferring DataFixerUpper (DFU) initialization, which Minecraft uses for world data migration. On a large world like Drehmal this makes a noticeable difference.

### Re-downloading mods

If you need to re-download the mods, use the following commands from the `mods/` directory:

```bash
cd /opt/minecraft-arm64-drehmal/mods

# Fabric API 0.92.6
wget "https://cdn.modrinth.com/data/P7dR8mSH/versions/UapVHwiP/fabric-api-0.92.6%2B1.20.1.jar"

# Lithium 0.11.4
wget "https://cdn.modrinth.com/data/gvQqBUqZ/versions/iEcXOkz4/lithium-fabric-mc1.20.1-0.11.4.jar"

# C2ME alpha.11.16
wget "https://cdn.modrinth.com/data/VSNURh3q/versions/s4WOiNtz/c2me-fabric-mc1.20.1-0.2.0%2Balpha.11.16.jar"

# ModernFix 5.24.3
wget "https://cdn.modrinth.com/data/nmDcB62a/versions/Qt5OXLYh/modernfix-fabric-5.24.3%2Bmc1.20.1.jar"

# FerriteCore 6.0.1
wget "https://cdn.modrinth.com/data/uXXizFIs/versions/unerR5MN/ferritecore-6.0.1-fabric.jar"

# Memory Leak Fix 1.1.5
wget "https://cdn.modrinth.com/data/NRjRiSSD/versions/5xvCCRjJ/memoryleakfix-fabric-1.17%2B-1.1.5.jar"

# Starlight 1.1.2
wget "https://cdn.modrinth.com/data/H8CaAYZC/versions/XGIsoVGT/starlight-1.1.2%2Bfabric.dbc156f.jar"

# Lazy DFU 0.1.3
wget "https://cdn.modrinth.com/data/hvFnDODi/versions/0.1.3/lazydfu-0.1.3.jar"
```

---

## Docker Image

The `java21` tag is used explicitly rather than `latest` because:

- Minecraft 1.20.1 officially targets Java 21
- The `latest` tag tracks the newest Java version (currently Java 25) which breaks mod compatibility
- Using a specific tag prevents unexpected breakage on image updates

---

## Directory Structure

```
/opt/minecraft-arm64-drehmal/
├── docker-compose.yml
├── mods/
│   ├── fabric-api-0.92.6+1.20.1.jar
│   ├── lithium-fabric-mc1.20.1-0.11.4.jar
│   ├── c2me-fabric-mc1.20.1-0.2.0+alpha.11.16.jar
│   ├── modernfix-fabric-5.24.3+mc1.20.1.jar
│   ├── ferritecore-6.0.1-fabric.jar
│   ├── memoryleakfix-fabric-1.17+-1.1.5.jar
│   ├── starlight-1.1.2+fabric.dbc156f.jar
│   └── lazydfu-0.1.3.jar
├── data/                   # Not committed to git
│   └── world/              # Extracted Drehmal world folder
├── CLAUDE.md
└── README.md
```

---

## Known Issues & Notes

- The server produces a `Can't keep up!` warning during initial startup while loading the Drehmal world. This is expected due to the world's size and resolves once the spawn area is fully loaded.
- C2ME is an alpha build — if the server fails to start, try removing it from `mods/` first to isolate the issue.
- Starlight is archived and will not receive further updates for 1.20.1. It is stable and safe to use.
- The Drehmal world file is hosted on Google Drive. If the SBC has no browser, download it on a PC and `scp` it to the SBC before extracting.
