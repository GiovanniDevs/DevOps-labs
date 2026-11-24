# Nexus as a Private Docker Repository

Hands-on follow-along project from Module 7 of the DevOps Bootcamp. Builds on
`03-dockerize-nodejs-app`; takes the locally built image and publishes it to a private
Docker registry hosted on Nexus. Completed against a self-hosted Nexus instance running on
Oracle Cloud rather than the DigitalOcean droplet used in the lecture.

## Overview

A locally built image is only useful on the machine that built it. In a real pipeline the
build server pushes the image to a registry and the deployment server pulls it from there.
Docker Hub is one option, but most companies keep their application images private.

This project configures Nexus — already in use as an artifact repository from Module 6 — as
a private Docker registry, then pushes the `my-app` image to it. The interesting part is not
the push itself but everything that has to be lined up first: a hosted repository, a
dedicated role and user, an HTTP connector on its own port, firewall rules at two separate
layers, a Docker-specific authentication realm, and a Docker client willing to talk to an
unencrypted registry.

## Project objectives

- Create a Docker hosted repository in Nexus
- Create a role and user with permission to push to it
- Expose the repository on a dedicated port via an HTTP connector
- Open that port through both firewall layers on the cloud server
- Enable the authentication realm Docker clients require
- Configure the local Docker daemon to trust an insecure registry
- Tag and push the image, and confirm it arrives

## Technologies used

Docker, Nexus Repository Manager 3, Oracle Cloud Infrastructure, iptables, Docker Desktop
on WSL2

## Architecture

```
  Local machine (Windows + WSL2)            Oracle Cloud server
  ┌────────────────────────────┐            ┌──────────────────────────────┐
  │ Docker Desktop             │            │  Nexus                       │
  │   my-app:v1                │            │                              │
  │        │                   │   :8083    │   ┌──────────────────────┐   │
  │        │ docker push ──────┼───────────────▶│ docker-hosted repo   │   │
  │        │                   │            │   └──────────────────────┘   │
  │ Browser ───────────────────┼───────────────▶ Nexus web UI :8081        │
  └────────────────────────────┘            └──────────────────────────────┘
```

Two ports, two purposes. Port 8081 serves the Nexus web UI and its REST API. Port 8083 is a
dedicated HTTP connector that speaks the Docker Registry API and nothing else. A Docker
client cannot share the UI port, because the Docker protocol expects the registry to live at
the root of the host it connects to.

## Prerequisites

- A running Nexus instance reachable over the network (see Module 6, or
  `07-nexus-as-docker-container` for a containerised install)
- Administrator access to the Nexus web UI
- An image available locally to push — produced by `03-dockerize-nodejs-app`
- Permission to edit the cloud provider's network rules and the server's local firewall

## Implementation

### 1. Create the Docker hosted repository

In the Nexus UI, _Settings → Repositories → Create repository → docker (hosted)_. Named
`docker-hosted`. A _hosted_ repository stores artifacts uploaded to it, as opposed to a
_proxy_ repository that caches someone else's.

### 2. Create a role and user

_Settings → Security → Roles → Create role_, named `NxDocker`, with the privileges for the
new repository attached. The role is then assigned to a user. Nexus does not grant access to
a new repository automatically — without this step the push fails on authentication even
with correct credentials.

### 3. Add an HTTP connector on port 8083

Back in the repository settings, the hosted repository gets an HTTP connector on port
**8083**. This is what makes the repository addressable as a Docker registry at
`<nexus-ip>:8083`.

### 4. Open port 8083 on the server

This is where a cloud server differs from the tutorial's droplet. Oracle Cloud filters
traffic at two independent layers, and both have to allow the port:

**Layer 1 — VCN Security List.** In the OCI console, an ingress rule for TCP 8083 from
`0.0.0.0/0` on the subnet's security list.

**Layer 2 — the instance's own iptables.** Oracle's Ubuntu images ship with a pre-populated
iptables chain that ends in a catch-all REJECT. A new ACCEPT rule has to be inserted _above_
that REJECT, not appended after it:

```bash
sudo iptables -I INPUT 6 -p tcp --dport 8083 -j ACCEPT
sudo netfilter-persistent save
```

`-I INPUT 6` inserts at position 6; `-A` would append to the end, where the rule is never
reached. `netfilter-persistent save` makes the change survive a reboot.

### 5. Enable the Docker Bearer Token Realm

_Settings → Security → Realms_. The **Docker Bearer Token Realm** has to be moved into the
active list. Docker clients authenticate with bearer tokens rather than the basic auth Nexus
uses by default, so without this realm `docker login` fails regardless of credentials.

### 6. Allow an insecure registry on the Docker client

The connector serves plain HTTP. Docker refuses to talk to a non-HTTPS registry unless it is
explicitly whitelisted. In Docker Desktop, _Settings → Docker Engine_:

```json
{
  "insecure-registries": ["<nexus-ip>:8083"]
}
```

Applying this restarts the Docker daemon.

### 7. Log in, tag and push

```bash
docker login <nexus-ip>:8083
docker tag my-app:v1 <nexus-ip>:8083/my-app:v1
docker push <nexus-ip>:8083/my-app:v1
```

Docker decides where to push from the image name itself — there is no `--registry` flag. An
image named `my-app:v1` goes to Docker Hub; the same image retagged with a registry prefix
goes to that registry. Tagging does not copy the image, it adds a second name pointing at
the same layers.

## Verification

- The Nexus UI, under _Browse → docker-hosted_, lists `my-app` with tag `v1`. Expanding it
  shows the individual image layers stored as separate assets.
- The REST API returns the same thing:

```bash
curl -u <user>:<password> \
  "http://<nexus-ip>:8081/service/rest/v1/components?repository=docker-hosted"
```

- The connector port responds to the Docker Registry API:

```bash
curl -m 5 http://<nexus-ip>:8083/v2/
```

A `UNAUTHORIZED` response is the expected success case — it proves the registry endpoint is
live and demanding credentials. A timeout means the traffic never arrived and the problem is
the firewall, not Nexus.

## Useful commands

```bash
docker images                                   # confirm the retagged image exists
docker tag my-app:v1 <nexus-ip>:8083/my-app:v1
docker push <nexus-ip>:8083/my-app:v1
docker pull <nexus-ip>:8083/my-app:v1
docker logout <nexus-ip>:8083

sudo iptables -L INPUT -n --line-numbers -v     # rule order and packet counters
sudo netfilter-persistent save
```

## Security considerations

- The connector serves plain HTTP, which means credentials and image layers cross the
  network unencrypted. Acceptable for a practice environment, not for anything real — a
  production registry sits behind TLS, usually via a reverse proxy.
- `insecure-registries` lowers the Docker client's guard for that specific host. It should
  list only the registry it is needed for, and be removed when it is not.
- The ingress rule opens 8083 to `0.0.0.0/0`. Restricting it to known source addresses is
  the obvious hardening step.
- A dedicated role and user per consumer is better than reusing the admin account; it keeps
  the push credentials scoped to one repository.

## Troubleshooting

**`docker login` fails with an authentication error.** Either the Docker Bearer Token Realm
is not active, or the user's role is missing privileges on the repository.

**`docker push` is rejected as an insecure registry.** The `insecure-registries` entry is
missing, misspelled, or the Docker daemon was not restarted after the change. The entry must
match the registry name used in the image tag exactly, port included.

## Key learnings

- Docker's registry target is encoded in the image name, not passed as a flag — which makes
  `docker tag` the mechanism for choosing where an image is published
- Nexus treats a Docker registry as a repository like any other, but it needs three separate
  pieces configured before a client can reach it: the repository, the connector, and the
  bearer token realm
- A cloud server can have more than one firewall, and a rule at the provider level means
  nothing if the host-level firewall rejects the packet first
- iptables rule _order_ is as important as rule content, and packet counters are the fastest
  way to tell a mis-ordered rule from a missing one
- Serving a registry over plain HTTP forces an explicit client-side exception, which is the
  protocol making a point about what it expects

## Project structure

```
04-nexus-docker-hosted-repository/
└── README.md
```

Documentation only. The work in this project is Nexus and firewall configuration performed
through the Nexus UI, the Oracle Cloud console, and the server shell; there are no files to
commit. The image pushed here is built by `03-dockerize-nodejs-app` and consumed by
`05-deploy-with-docker-compose-on-server`.
