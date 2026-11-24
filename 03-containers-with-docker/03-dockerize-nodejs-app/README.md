# Dockerize the Node.js Application

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Builds on
`02-docker-compose-mongo-mongoexpress`; packages the demo application itself into a Docker
image so it can be deployed rather than run from source.

## Overview

Up to this point the Node.js application ran on the host while only its dependencies were
containerised. To deploy it anywhere, the application has to be packaged into its own
image. This project writes the Dockerfile that does that — the same step a CI server such
as Jenkins performs automatically after a commit.

## Project objectives

- Write a Dockerfile that packages the Node.js application and its dependencies
- Build a tagged image from it
- Verify the image contents and environment from inside a running container

## Technologies used

Docker, Node.js (`node:20-alpine` base image)

## Implementation

The committed `Dockerfile`:

```docker
FROM node:20-alpine
ENV MONGO_DB_USERNAME=admin \
    MONGO_DB_PWD=password
RUN mkdir -p /home/app
COPY ./app /home/app
WORKDIR /home/app
RUN npm install
CMD ["node", "server.js"]
```

Instruction by instruction:

- **`FROM node:20-alpine`** — every image is built on another image. Basing this on the
  official Node image means Node and npm are already present, rather than starting from a
  bare Linux image and installing them. Alpine keeps the result small.
- **`ENV`** — sets the database credentials inside the image. Defining them in the Compose
  file instead is generally better, because they can then be changed without rebuilding the
  image. Both are valid and the module demonstrates each.
- **`RUN mkdir -p /home/app`** — `RUN` executes during the build, inside the image, not on
  the host.
- **`COPY ./app /home/app`** — `COPY` is the one instruction that reads from the host. Only
  the `app` directory is copied, which holds just the files needed to run the application.
  Copying the whole project directory would pull in the Dockerfile itself and other files
  that have no business in the image.
- **`WORKDIR /home/app`** — sets the default directory for everything after it. Without it,
  `CMD ["node", "server.js"]` runs from the image's default directory and cannot find
  `server.js`.
- **`RUN npm install`** — dependencies are installed during the build rather than copied
  from the host. Copying a host `node_modules` risks shipping stale packages built for a
  different platform; installing from `package.json` means every build produces current
  dependencies.
- **`CMD ["node", "server.js"]`** — the entrypoint, run when the container starts. `RUN`
  happens at build time and can appear many times; `CMD` happens at runtime and only the
  last one applies.

Build the image:

```bash
docker build -t my-app:v1 .
```

- `-t` names and tags the image
- `.` is the build context — the directory Docker reads the Dockerfile and `COPY` sources
  from

The file must be named exactly `Dockerfile`, with a capital D and no extension.

## Verification

```bash
docker run my-app:v1   # attached, so startup logs are visible in the terminal
docker ps
docker exec -it <container-id> /bin/sh
```

Inside the container:

```bash
ls /home/app   # application files present in the expected directory
env            # MONGO_DB_USERNAME and MONGO_DB_PWD set as declared
exit
```

`/bin/sh` is used rather than `/bin/bash` because Alpine-based images do not ship bash.

## Useful commands

```bash
docker images                 # list images
docker ps -a                  # include stopped containers
docker stop <container-id>
docker rm <container-id>
docker rmi my-app:v1
docker build -t my-app:v1 .   # rebuild
```

A changed Dockerfile always needs a rebuild — images are not updated in place. The old
image cannot be removed while a container is still using it, so containers go first.

## Key learnings

- A Dockerfile is a build script for an image, and writing one by hand makes it obvious
  what a CI pipeline is actually doing when it "builds and pushes an image"
- The order of instructions matters: `WORKDIR` before `CMD`, and `COPY` before
  `RUN npm install`, or the build succeeds and the container fails at startup
- Installing dependencies inside the image rather than copying them from the host is what
  makes the image reproducible
- Being deliberate about what gets copied keeps the image clean; a build context is not a
  reason to ship the whole working directory

## Project structure

```
03-dockerize-nodejs-app/
├── README.md
└── Dockerfile
```

The application source the Dockerfile copies from `./app` is the instructor's demo
application and is not committed to this repository.

The `MONGO_DB_USERNAME` and `MONGO_DB_PWD` values in the Dockerfile are the bootcamp's
throwaway tutorial defaults, kept as they were used. They are not real credentials and were
never used outside these exercises. In a real project these would be supplied at runtime
rather than baked into the image.
