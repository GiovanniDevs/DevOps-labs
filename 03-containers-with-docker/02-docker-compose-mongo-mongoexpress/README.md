# Docker Compose — Run Multiple Containers

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Builds on
`01-docker-local-development`; replaces the two `docker run` commands with a single Docker
Compose file. Documentation only — this folder contains nothing beyond this README.

## Overview

Starting MongoDB and Mongo Express by hand means re-typing two long `docker run` commands
with all their flags and environment variables every time. Docker Compose captures the same
configuration in one YAML file, so the whole set of containers starts and stops with a
single command.

## Project objectives

- Express the two `docker run` commands from project 01 as a Docker Compose file
- Start and stop both containers with one command
- Understand how Compose handles networking and container startup order
- Observe that container data does not survive recreation

## Technologies used

Docker, Docker Compose, MongoDB, Mongo Express

## Implementation

The Compose file maps one-to-one onto the previous `docker run` commands: each key under
`services` becomes a container, with its `image`, `ports` and `environment` nested inside.

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
```

Points worth noting:

- **There is no network configuration.** Compose creates a network for the project
  automatically and registers each service name on it, so the manual
  `docker network create mongo-network` step from project 01 is no longer needed.
  `ME_CONFIG_MONGODB_SERVER=mongodb` now resolves through the Compose service name.
- **`restart: always` on Mongo Express** handles startup order. Both containers start at
  once, so Mongo Express often comes up before MongoDB is accepting connections and exits
  with a connection error. Restarting it until MongoDB is ready is the simplest fix.
  `depends_on` with a health check is the more precise option.
- **Indentation is significant.** All services sit at the same level, with their own
  configuration nested beneath them.
- **Container names are prefixed.** Compose prefixes each service name with the project
  name, which defaults to the name of the directory holding the Compose file.

Start and stop the stack:

```bash
docker-compose -f <compose-file>.yaml up     # attached, logs in the terminal
docker-compose -f <compose-file>.yaml up -d  # detached
docker-compose -f <compose-file>.yaml down   # stop and remove containers and the network
```

## Verification

- `docker ps` showed both containers running under their Compose-prefixed names
- `docker network ls` showed the network Compose had created for the project, confirming it
  had not needed a manually created one
- Opening Mongo Express on port 8081 showed that the database and collection created in
  project 01 were **gone** — the containers had been recreated and their data went with them
- After recreating the `user-account` database and `users` collection, I started the Node.js
  application, edited the profile, and saw the change persist, confirming
  application-to-database connectivity through the Compose network
- Running `down` and then `docker network ls` showed the network had been removed; running
  `up` again recreated it

## Key learnings

- A Compose file is not a new concept, it is the same `docker run` configuration in a
  structured, editable, version-controllable form
- Compose creating the network automatically is the most practical difference from
  `docker run` — service names become hostnames with no extra work
- `docker-compose down` removes containers _and_ the network, so anything written inside a
  container is lost. This is the limitation that makes volumes necessary, addressed in
  `06-docker-volumes-persistence`
- `restart: always` is a blunt but effective answer to startup ordering between dependent
  containers

## Project structure

This folder contains this documentation and the Compose file itself. At this stage the file
defined just the two services shown above; a later version, with a volume added, lives in
[`06-docker-volumes-persistence`](../06-docker-volumes-persistence/).
