# Deploy the Application Stack with Docker Compose

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Brings together
`03-dockerize-nodejs-app` and `04-nexus-docker-hosted-repository`: the application image is
pulled from the private registry and started alongside its database as a single Compose
stack. The "deployment server" was simulated locally rather than on a remote machine — the
stack runs exactly as it would on a server, just on the same host that built it.

## Overview

Until now the Node.js application ran from source on the host while MongoDB and Mongo
Express ran in containers. This project completes the picture: all three run as containers,
defined in one Compose file, started with one command.

The important change is where the application image comes from. Rather than being built
locally, it is pulled from the private Nexus registry set up in `04`. That is the real
deployment pattern — the build happens once, somewhere else, and the deployment server only
pulls and runs.

## Project objectives

- Add the application itself as a service in the Docker Compose file
- Pull the application image from the private Docker registry rather than building it
- Change the application's database connection string from a host address to a Compose
  service name
- Start the whole stack with a single command and verify it end to end

## Technologies used

Docker, Docker Compose, Node.js, MongoDB, Mongo Express, Nexus Repository Manager 3

## Architecture

```
  ┌──────────────────────── Docker Compose network ───────────────────────┐
  │                                                                       │
  │   my-app          ──▶   mongodb          ◀──   mongo-express          │
  │   :3000                 :27017                 :8081                  │
  │                                                                       │
  └───────────────────────────────────────────────────────────────────────┘
        ▲                       ▲                         ▲
        │ 3000:3000             │ 27017:27017             │ 8081:8081
     host ports

  Image sources:
    my-app         <nexus-ip>:8083/my-app:v1   (private Nexus registry)
    mongo          Docker Hub
    mongo-express  Docker Hub
```

All three services sit on the network Compose creates automatically, so they address each
other by service name. The host port mappings exist only so a browser on the host can reach
the application and the database UI.

## Prerequisites

- Docker and Docker Compose installed
- An application image already pushed to the private registry (`04`)
- Credentials for that registry, and the registry listed under `insecure-registries` on the
  Docker client

## Implementation

### 1. The Compose file

The committed `docker-compose.yaml`:

```yaml
version: "3"
services:
  my-app:
    image: <nexus-ip>:8083/my-app:v1
    ports:
      - 3000:3000
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

The only structural difference from `02-docker-compose-mongo-mongoexpress` is the `my-app`
service. Everything else is carried over unchanged.

### 2. The fully qualified image name

`image: <nexus-ip>:8083/my-app:v1` is what makes Compose pull from Nexus. A bare `my-app:v1`
would send Docker to Docker Hub and fail. The registry host and port are part of the image
name, which is the same rule that governed `docker push` in `04`.

### 3. Authenticate before starting

```bash
docker login <nexus-ip>:8083
```

The registry is private, so the pull is rejected without a prior login. This is the one
manual prerequisite that is easy to forget, because nothing in the Compose file mentions
credentials.

### 4. Point the application at the Compose service name

The application's connection string had to change:

```
mongoUrlLocalhost        →  mongoUrlDockerCompose
mongodb://localhost:27017 → mongodb://mongodb:27017
```

When the application ran on the host, `localhost` correctly meant "the host, where MongoDB's
port is published". Inside a container, `localhost` means the application's own container,
which has no MongoDB. The service name `mongodb` resolves through Compose's internal DNS to
the database container.

### 5. Rebuild and re-push

Because the connection string lives in the application source, the change means a new image:

```bash
docker build -t my-app:v1 .
docker tag my-app:v1 <nexus-ip>:8083/my-app:v1
docker push <nexus-ip>:8083/my-app:v1
```

This is the loop the project is really teaching — a code change is not deployed until it has
been rebuilt, re-pushed, and re-pulled.

### 6. Start the stack

```bash
docker-compose -f docker-compose.yaml up
```

Compose creates the network, pulls any missing images, and starts all three containers.

## Verification

- `docker ps` lists three running containers
- The application responds at `http://localhost:3000`
- Mongo Express is reachable at `http://localhost:8081` and shows the database
- An edit made through the application appears in Mongo Express, confirming the application
  container is genuinely talking to the database container rather than to a stale local
  instance
- The startup logs show the `my-app` image being pulled from `<nexus-ip>:8083`, not built

## Useful commands

```bash
docker-compose -f docker-compose.yaml up        # attached, logs in the terminal
docker-compose -f docker-compose.yaml up -d     # detached
docker-compose -f docker-compose.yaml down      # stop and remove containers + network
docker-compose -f docker-compose.yaml pull      # refresh images without starting
docker-compose -f docker-compose.yaml logs -f my-app

docker login <nexus-ip>:8083
docker ps
```

## Security considerations

- The MongoDB and Mongo Express credentials in the Compose file are the bootcamp's
  throwaway tutorial defaults, kept as they were used. They are not real credentials and
  were never used outside these exercises.
- In a real deployment these belong in environment variables supplied at deploy time, or in
  a secrets manager, never in a file committed to the repository. Compose supports
  `${VARIABLE}` placeholders resolved from a `.env` file precisely for this.
- The registry address is replaced with `<nexus-ip>` in this README. The server address is
  not secret, but there is no reason to publish it either.
- Publishing MongoDB's port 27017 to the host is convenient for local inspection and
  unnecessary in a real deployment, where only the services that need external access
  should expose ports.

## Troubleshooting

**The `my-app` image fails to pull.** Either `docker login` has not been run against the
registry, or the registry is missing from `insecure-registries` on the Docker client. The
error text distinguishes them: an authentication failure names the credentials; an HTTPS
failure complains about the server certificate or an HTTP response.

**The application starts but cannot reach the database.** The connection string is still
pointing at `localhost`. Confirm the running image is the rebuilt one — a successful pull of
an _old_ image looks identical in the logs.

**Mongo Express shows connection errors.** `ME_CONFIG_MONGODB_SERVER` must match the Compose
service name exactly, and `ME_CONFIG_MONGODB_URL` must be set explicitly. The Docker Hub
documentation omits the latter; the project's GitHub page is the accurate reference.

## Key learnings

- A deployment server does not build images, it pulls them — and encoding the registry in
  the image name is how Compose knows where to pull from
- `localhost` means something different inside a container, and that single assumption is
  what has to change when an application moves from the host into the stack
- A source change is not live until the image is rebuilt and re-pushed; the registry is the
  handoff point between building and deploying
- One `docker-compose up` replacing three `docker run` commands is the practical argument
  for Compose, and it scales with the number of services
- This stack has no volume, so the database starts empty every time the containers are
  recreated — the gap that `06-docker-volumes-persistence` closes

## Project structure

```
05-deploy-with-docker-compose-on-server/
├── README.md
└── docker-compose.yaml
```

The Node.js application source is the instructor's demo application and is not committed to
this repository. The image referenced by the `my-app` service is built by
`03-dockerize-nodejs-app` and published by `04-nexus-docker-hosted-repository`.
