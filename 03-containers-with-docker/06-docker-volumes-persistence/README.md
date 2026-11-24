# Docker Volumes and Data Persistence

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Builds on
`05-deploy-with-docker-compose-on-server` by attaching a named volume to MongoDB, so data
survives when containers are removed and recreated.

## Overview

Every container so far has lost its data on restart. A container's writable layer is deleted
with the container, so a database running in one starts empty every time. A volume stores
that data outside the container, on the host, and re-attaches it to whatever container
mounts it next.

This project adds a named volume to the MongoDB service and verifies that a database created
through the UI is still there after a full `down` / `up` cycle.

## Project objectives

- Define a named volume in Docker Compose and mount it into the MongoDB container
- Understand which container path a volume has to target for a given database
- Prove persistence by destroying and recreating the containers
- Locate the volume on the host

## Technologies used

Docker, Docker Compose, MongoDB, Mongo Express

## Implementation

The committed `docker-compose.yaml`:

```yaml
version: "3"
services:
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - mongo-data:/data/db
  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_BASICAUTH_USERNAME=user
      - ME_CONFIG_BASICAUTH_PASSWORD=pass
      - ME_CONFIG_MONGODB_SERVER=mongodb
      - ME_CONFIG_MONGODB_URL=mongodb://mongodb:27017
volumes:
  mongo-data:
    driver: local
```

Two additions to the file from `02`:

- **`volumes: - mongo-data:/data/db`** inside the service — mounts the named volume at the
path MongoDB writes its data files to. The container-side path is not arbitrary; it has to
match where the database actually persists data. MySQL uses `/var/lib/mysql`, Postgres
uses `/var/lib/postgresql/data`.
- **The top-level `volumes:` block** — declares `mongo-data` so Compose creates it, with
`driver: local` telling Docker to allocate storage on the local filesystem.

Docker prefixes the created volume with the Compose project name, so the actual volume is
named something like `<project>_mongo-data` rather than `mongo-data`.

The Node.js application still ran on the host for this project; only the database and its UI
were containerised.

## Verification

Start the stack, then through Mongo Express create a database, a collection, and an entry:

```bash
docker-compose -f docker-compose.yaml up -d
# create data via http://localhost:8081
docker-compose -f docker-compose.yaml down
docker-compose -f docker-compose.yaml up -d
```

After the restart the database, collection, and entry were all still present — the
containers were destroyed and recreated, but the volume was re-attached.

Inspect the volume itself:

```bash
docker volume ls
docker volume inspect <project>_mongo-data
```

`inspect` reports the mountpoint, under `/var/lib/docker/volumes/`, where each volume has a
`_data` subfolder holding the files. On Docker Desktop for Windows and macOS this path lives
inside the Docker VM rather than directly on the host filesystem.

## Useful commands

```bash
docker volume ls
docker volume inspect <volume-name>
docker volume create <name>
docker volume rm <name>
docker volume prune                              # remove all unused volumes

docker-compose -f docker-compose.yaml down       # keeps volumes
docker-compose -f docker-compose.yaml down -v    # DELETES volumes and all data
```

`down -v` is the one to be careful with — it is the difference between restarting a stack
and wiping the database.

## Key learnings

- Container storage is disposable by default; persistence is something you opt into, not
something you get
- The container-side mount path is database-specific, and mounting the wrong path produces a
stack that works perfectly until the first restart
- Named volumes are managed by Docker and survive `down`, which is what makes them the right
choice for databases rather than bind-mounting a host directory
- Compose namespaces volume names by project, so the name in the file is not the name on
disk
- Testing persistence means actually destroying the containers — a `restart` proves nothing

## Project structure

```
06-docker-volumes-persistence/
├── README.md
└── docker-compose.yaml
```

The MongoDB and Mongo Express credentials in the Compose file are the bootcamp's throwaway
tutorial defaults, kept as they were used. They are not real credentials and were never used
outside these exercises.