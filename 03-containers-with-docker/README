# Containers with Docker

This module covers containers and Docker end to end: what containers are and how they
differ from virtual machines, the main Docker and debugging commands, running an
application's dependencies in containers for local development, replacing long
`docker run` commands with Docker Compose, packaging an application into a custom image
with a Dockerfile, hosting a private Docker registry on Nexus, deploying a multi-container
stack with Compose, persisting database data with volumes, and running Nexus itself as a
container.

All seven projects are hands-on follow-along projects from the bootcamp. I completed every
step myself, on my own machine and my own cloud infrastructure.

## Learning objectives

- Explain what a container is, what problems it solves, and how Docker differs from a virtual machine
- Use the main Docker commands to pull images, run containers, publish ports and debug running containers
- Run an application's dependencies in containers and connect to them from a locally running app
- Define a multi-container setup declaratively with Docker Compose instead of individual `docker run` commands
- Package an application into a custom Docker image with a Dockerfile
- Configure a private Docker registry on Nexus and push images to it
- Deploy a multi-container application from a private registry with Docker Compose
- Persist container data with named Docker volumes
- Deploy a third-party application (Nexus) as a Docker container

## Technologies used

Docker, Docker Compose, Node.js, MongoDB, Mongo Express, Nexus Repository Manager,
Oracle Cloud, Linux (Ubuntu), WSL2, Docker Desktop

## Shared context

The projects in this module share the same material, so it is described once here rather
than repeated in every project README.

**The demo application.** Projects 01 to 06 all use the instructor's Node.js demo
application: a small user-profile page that reads and writes a single document in a MongoDB
collection. I did not write this application — it is provided by the bootcamp as the
vehicle for the Docker work. Its source is not committed to this repository, because
committing someone else's demo app adds nothing to the documentation.

**Local environment.** All local work was done on Windows using WSL2 with Docker Desktop.

**Database containers.** MongoDB and Mongo Express are used throughout, both pulled from
Docker Hub. MongoDB runs with root credentials supplied as environment variables at
startup, and Mongo Express needs those same credentials plus its own basic-auth login to
reach it. `ME_CONFIG_MONGODB_URL` has to be set explicitly — the Docker Hub documentation
for the image is out of date and omits it, which causes connection errors.

The credentials that appear in this module's files and commands are the bootcamp's
throwaway tutorial defaults. They are not real secrets and were never used anywhere else.

**Nexus.** Projects 04 and 05 use the Nexus instance I built in Modules 5 and 6. I rebuilt
that setup on Oracle Cloud rather than the bootcamp's DigitalOcean, which adds a second
firewall layer to every port I open — see project 04.

**Compose command syntax.** The lectures use the standalone `docker-compose` binary, so
that is the form recorded in these READMEs. With the modern Compose plugin the equivalent
commands are `docker compose ...` with a space.

## Projects

| Folder                                                                                | Project                                            | Description                                                                                                      |
| ------------------------------------------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| [`01-docker-local-development`](01-docker-local-development/)                         | Use Docker for local development                   | Run MongoDB and Mongo Express as containers on a user-defined network and connect a host-run Node.js app to them |
| [`02-docker-compose-mongo-mongoexpress`](02-docker-compose-mongo-mongoexpress/)       | Docker Compose — run multiple containers           | Replace the two `docker run` commands with a single Docker Compose file                                          |
| [`03-dockerize-nodejs-app`](03-dockerize-nodejs-app/)                                 | Dockerize the Node.js application                  | Write a Dockerfile and build the application into a custom image                                                 |
| [`04-nexus-docker-hosted-repository`](04-nexus-docker-hosted-repository/)             | Create a Docker repository on Nexus and push to it | Configure a Docker hosted repository, role and registry port on Nexus, then push the image                       |
| [`05-deploy-with-docker-compose-on-server`](05-deploy-with-docker-compose-on-server/) | Deploy the application with Docker Compose         | Deploy the three-container stack, pulling the app image from the private registry                                |
| [`06-docker-volumes-persistence`](06-docker-volumes-persistence/)                     | Persist data with Docker volumes                   | Attach a named volume to MongoDB so data survives container recreation                                           |
| [`07-nexus-as-docker-container`](07-nexus-as-docker-container/)                       | Deploy Nexus as a Docker container                 | Run Nexus itself as a container on a cloud server with a named volume                                            |

## Key learnings

- Containers package an application with its dependencies, which is why moving a database
  into a container removes the need to install and configure it on the host at all
- Containers on the same user-defined network reach each other by name, and Docker Compose
  creates that network automatically — the single biggest reason Compose is simpler than a
  set of `docker run` commands
- `localhost` inside a container refers to that container, not the host, so containerising
  an application forces its connection strings to change from `localhost` to service names
- A container's filesystem is disposable: anything written inside it is gone when the
  container is recreated, which makes volumes mandatory for anything stateful
- Docker decides where to push based on the image name, so an image destined for a private
  registry has to be tagged with that registry's host and port
- Running a third-party service as a container moved work I had done manually in Module 6 —
  installing Java, creating a service user, setting permissions — into the image itself
