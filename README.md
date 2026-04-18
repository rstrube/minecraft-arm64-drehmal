## Introduction

This is a docker compose file and a collection of specific mods to run Minecraft + Drehmal APOTHEOSIS as a persistent Minecraft server on a headless SBC (Single Bord Computer, e.g. Raspberry Pi).
The docker compose file is:

* Based on [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server)
* Uses Minecraft 1.20.1
* Uses the awesome [Dremhel APOTHEOSIS 2.2.2f](https://www.drehmal.net/) for world.

The repo contains the compose file, as well as specific mods that are all designed to work together.

## Assumptions

* If your setting up the SBC as a headless server, you should be able to SSH into it
* You can issue sudo commands on the SBC
* You already have docker installed and your user is part of the `docker` group

## Setup on Arm64 SBC

1. Clone this repo to desired location on SBC (e.g. `/opt`), change ownership, create data directory

*Note: the rest of these instructions assumes you'll installing everything into `/opt/minecraft-arm64-drehmal`*

```
cd /opt
sudo git clone https://github.com/rstrube/minecraft-arm64-drehmal
sudo chown -R $USER:$USER /opt/minecraft-arm64-drehmal
mkdir -p /opt/minecraft-arm64-drehmal/data/world
```

2. Download latest [Dremhel APOTHEOSIS](https://www.drehmal.net/) world.zip file.

*Note: because it's a Google Drive Link if your SBC doesn't have a web browser, download it on a PC and scp it to the SBC*

```
scp Drehmal\ APOTHEOSIS\ 2.2.2f.zip user@sbc:/opt/minecraft-arm64-drehmal/.
```

3. Extract the world.zip file and rename it

```
cd /opt/minecraft-arm64-drehmal
unzip Drehmal\ APOTHEOSIS\ 2.2.2f.zip -d data/world

# The extracted folder should be renamed to 'world'
mv data/Drehmal* data/world
```

## Modify Docker Compose File
The `docker-compose.yml` file has mulitple SBC tiers defined within it (tier 3 is uncommented by default).  Based on the specific hardware of your SBC, uncomment your hardware tier for the best performance on your hardware.

In addition, it's good practice to set the `UID` and `GID` environment variables in the compose file.  to the user that will run the docker container.  If you do not set these, logs and changes to the `/data` subdirectory will be owned by `root` which might make it a little more difficult to maintain (cleanup, modify, etc.).

## Run Container
After modifying the docker compose file, you can start up the container using:

```
docker compose up -d
```

You can attach to the container using:

```
docker attach minecraft-drehmal
```

To disconnect use `CTRL+Q,P`

You can view logs:

```
docker logs -f minecraft-drehmal
```

To stop viewing logs use `CTRL+C`

## Server Side Mod Information
Drehmal recommends both server and client side modes.  Below is a list of server mods that are package with this repository and match what Drehmal v2.2.2f recommends.

### Fabric API 0.92.6
wget "https://cdn.modrinth.com/data/P7dR8mSH/versions/UapVHwiP/fabric-api-0.92.6%2B1.20.1.jar"

### Lithium 0.11.4
wget "https://cdn.modrinth.com/data/gvQqBUqZ/versions/iEcXOkz4/lithium-fabric-mc1.20.1-0.11.4.jar"

### C2ME alpha.11.16
wget "https://cdn.modrinth.com/data/VSNURh3q/versions/s4WOiNtz/c2me-fabric-mc1.20.1-0.2.0%2Balpha.11.16.jar"

### ModernFix 5.24.3
wget "https://cdn.modrinth.com/data/nmDcB62a/versions/Qt5OXLYh/modernfix-fabric-5.24.3%2Bmc1.20.1.jar"

### FerriteCore 6.0.1
wget "https://cdn.modrinth.com/data/uXXizFIs/versions/unerR5MN/ferritecore-6.0.1-fabric.jar"

### Memory Leak Fix 1.1.5
wget "https://cdn.modrinth.com/data/NRjRiSSD/versions/5xvCCRjJ/memoryleakfix-fabric-1.17%2B-1.1.5.jar"

### Starlight 1.1.2
wget "https://cdn.modrinth.com/data/H8CaAYZC/versions/XGIsoVGT/starlight-1.1.2%2Bfabric.dbc156f.jar"
