# Run Nexus as a Docker Container

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Revisits the Nexus
installation from Module 6 — originally a manual install of Java, Nexus, and its service
configuration — and replaces the whole thing with a single container.

## Overview

Installing Nexus directly on a server means installing a JDK, downloading and unpacking
Nexus, creating a dedicated user, setting permissions, and wiring up a systemd service.
Running it as a container collapses that into one `docker run`: the image already contains
Java and Nexus, and the only server-side prerequisite is Docker itself.

This project provisions an Ubuntu cloud server, installs Docker, and runs Nexus as a
container with a named volume for its data. The lecture used DigitalOcean; the steps are
provider-neutral and apply to any Ubuntu cloud server.

## Project objectives

- Provision a cloud server and configure its firewall for SSH and Nexus
- Install Docker as the only server-side dependency
- Run Nexus as a container with persistent storage
- Verify the container, its data volume, and the user it runs as

## Technologies used

Docker, Nexus Repository Manager 3 (`sonatype/nexus3`), Ubuntu

## Implementation

### 1. Provision the server and open the ports

An Ubuntu cloud server with firewall rules allowing inbound **22** for SSH and **8081** for
the Nexus web UI.

### 2. Install Docker

Docker is the only thing that has to be installed on the host. No JDK, no Nexus download, no
dedicated system user — all of that is inside the image. This is the entire point of the
exercise: the server's configuration shrinks to "has Docker".

### 3. Create a volume and run the container

```bash
docker volume create nexus-data

docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3
```

- `-d` runs it detached
- `-p 8081:8081` publishes the UI on the host
- `--name nexus` gives the container a stable name for later commands
- `-v nexus-data:/nexus-data` mounts the named volume at the path Nexus uses for its
  repository data, configuration, and logs — without it, every container recreation would
  mean a fresh, empty Nexus

## Verification

```bash
ss -tlnp | grep 8081      # or: netstat -lnpt
docker ps
```

The port is listening and the container is running. The UI then loads at
`http://<server-ip>:8081`.

Check what the container is actually running as:

```bash
docker exec -it nexus /bin/bash
```

The prompt comes up as `$` rather than `#`, and `whoami` returns `nexus` — the image ends
with a `USER nexus` instruction, so the process does not run as root inside the container.

Check the volume:

```bash
docker volume ls
docker volume inspect nexus-data
```

`inspect` gives the host mountpoint. The same files are visible at `/nexus-data` inside the
container, which matters in practice for three things: backing up Nexus by copying the
volume, reading the generated admin password on first start, and getting at the logs without
the UI.

## Useful commands

```bash
docker ps
docker logs nexus
docker logs -f nexus
docker exec -it nexus /bin/bash
docker stop nexus
docker start nexus
docker rm nexus                 # safe: the data lives in the volume
docker volume inspect nexus-data
```

Removing and recreating the container with the same `-v nexus-data:/nexus-data` flag brings
Nexus back with all repositories and users intact.

## Key learnings

- Containerising a service moves its dependencies from the server into the image, which is
  why the host only needed Docker
- Comparing this against the manual install from Module 6 makes the trade-off concrete:
  minutes instead of a long setup, and a reproducible result instead of a hand-configured
  machine
- Named volumes are what make a container disposable — the container can be deleted and
  recreated because the state is not in it
- Well-built images do not run as root, and `docker exec` is the quickest way to check
- The volume is also the backup unit; knowing where it lives on the host is operational
  knowledge, not trivia

## Project structure

```
07-nexus-as-docker-container/
└── README.md
```

Documentation only — the work is server provisioning and Docker commands, with no files to
commit. The Nexus instance produced here is the same kind of instance configured as a
private Docker registry in `04-nexus-docker-hosted-repository`.
