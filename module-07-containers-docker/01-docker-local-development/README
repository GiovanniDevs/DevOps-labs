# Use Docker for Local Development

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. First project in the
module's Node.js chain. Documentation only — this folder contains no application files.

## Overview

The demo Node.js application needs a MongoDB database, and a database UI is useful while
developing against it. Rather than installing MongoDB on my machine, I ran both MongoDB and
Mongo Express as Docker containers on a user-defined Docker network, and ran the Node.js
application itself on the host against the containerised database.

This is the local-development pattern the rest of the module builds on.

## Project objectives

- Create a user-defined Docker network so two containers can reach each other by name
- Run MongoDB as a container with root credentials supplied at startup
- Run Mongo Express as a container configured to connect to that MongoDB container
- Connect the host-run Node.js application to the containerised database

## Technologies used

Docker, Node.js, MongoDB, Mongo Express, WSL2 with Docker Desktop

## Implementation

### 1. Create a Docker network

```bash
docker network create mongo-network
```

Containers on the same user-defined network can reach each other by container name alone —
no host or port is needed between them. This is what lets Mongo Express find MongoDB.

### 2. Start MongoDB

```bash
docker run -d \
-p 27017:27017 \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=password \
--net mongo-network \
--name mongodb \
mongo
```

- `-p 27017:27017` publishes MongoDB's port on the host, so the Node.js application running
  outside Docker can still reach it
- The two `MONGO_INITDB_ROOT_*` variables create the root user at first startup. The same
  values are needed by Mongo Express and by the application
- `--name mongodb` is set explicitly rather than left to Docker, because Mongo Express
  references the container by this name

### 3. Start Mongo Express

```bash
docker run -d \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
-e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
-e ME_CONFIG_BASICAUTH_USERNAME=user \
-e ME_CONFIG_BASICAUTH_PASSWORD=pass \
-e ME_CONFIG_MONGODB_SERVER=mongodb \
-e ME_CONFIG_MONGODB_URL=mongodb://mongodb:27017 \
--net mongo-network \
--name mongo-express \
mongo-express
```

- The admin credentials must match the ones MongoDB was started with
- `ME_CONFIG_MONGODB_SERVER=mongodb` is the MongoDB _container name_, which resolves only
  because both containers sit on `mongo-network`
- `ME_CONFIG_BASICAUTH_*` sets the login for the Mongo Express UI itself
- `ME_CONFIG_MONGODB_URL` has to be set explicitly. The Docker Hub documentation for this
  image is out of date and omits it, and without it the container fails to connect

### 4. Create the database and collection

I created the `user-account` database and its `users` collection through the Mongo Express
UI on port 8081. The `MONGO_INITDB_DATABASE` environment variable would have been an
alternative way to create the database at container startup.

### 5. Connect the application

The Node.js application runs on the host and connects with a `MongoClient` to
`localhost:27017` using the root credentials set in step 2. It fetches the profile and
upserts changes to the `users` collection.

## Verification

- Editing the profile in the application UI and refreshing the page returned the updated
  values, confirming the write reached MongoDB and was read back
- `docker logs mongodb` showed incoming connections from the Node.js application, and
  end-of-connection events after each request

## Useful commands

```bash
docker ps                       # confirm both containers are running
docker logs --tail 100 mongodb  # last 100 log lines
docker logs -f mongodb          # follow logs live
docker network ls               # confirm mongo-network exists
```

## Key learnings

- Running dependencies as containers removes host installation entirely — nothing about
  MongoDB or Mongo Express was installed on my machine
- Container-to-container traffic uses the container name, but an application outside Docker
  still needs a published port and `localhost`. The two addressing models coexist, and
  knowing which applies where is the whole point of this project
- Setting `--name` explicitly is not cosmetic: another container depends on that name
- Reading an image's own documentation matters more than following a tutorial blindly —
  `ME_CONFIG_MONGODB_URL` is missing from the Docker Hub page and is required

## Project structure

This project is documentation only. The folder contains this README and no application
files; the demo application is the instructor's and is not committed.
