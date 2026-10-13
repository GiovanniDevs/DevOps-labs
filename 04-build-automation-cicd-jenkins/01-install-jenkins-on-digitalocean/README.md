# Install Jenkins on DigitalOcean

Hands-on follow-along project from Module 8 of the DevOps Bootcamp. First project in the module. Every later project in Module 8 runs on the Jenkins server built here. Documentation only: this folder contains no application files.

## Overview

Build automation needs a dedicated server. Building artifacts on a developer machine means stashing current work, switching branches, configuring registry credentials and waiting for the test suite before anything can be published, all of which blocks the developer and is easy to get wrong.

This project sets up that dedicated server: a DigitalOcean droplet running Jenkins as a Docker container, firewalled, persisted on a named volume, and initialised with an admin user and the suggested plugin set.

Jenkins was installed as a container rather than directly on the OS. Installing it on the OS means creating a Jenkins user, installing Java first, downloading binaries and configuring the service by hand. The official image removes all of that, and it matches how Nexus was deployed in Module 6.

## Project objectives

- Create and configure an Ubuntu server on DigitalOcean for Jenkins
- Restrict access to it with a dedicated cloud firewall
- Install Docker on the server and run Jenkins as a container
- Persist Jenkins data so the container can be replaced without losing jobs and configuration
- Complete the Jenkins initial setup and create the first admin user

## Technologies used

Jenkins, Docker, DigitalOcean, Linux (Ubuntu)

## Implementation

### 1. Create the droplet

A new Ubuntu droplet was created with 2 GB of RAM and named `Jenkins-Server`, so its purpose is obvious in the DigitalOcean dashboard. Jenkins needs at least 1 GB.

### 2. Configure the firewall

A separate firewall named `Jenkins Firewall` was created and attached to the droplet, rather than reusing the existing Nexus firewall, because each server gets its own rules.

| Port | Reason                                                                                                                      |
| ---- | --------------------------------------------------------------------------------------------------------------------------- |
| 22   | SSH access. Best practice is to whitelist a specific source IP, which was not practical here because my home IP is dynamic. |
| 8080 | The Jenkins web UI, open to all inbound traffic.                                                                            |

### 3. Install Docker on the server

The droplet was empty, and Docker is the only prerequisite. No Java install is needed, because Java ships inside the Jenkins image.

```bash
apt update
apt install docker.io
```

### 4. Run Jenkins as a container

```bash
docker run -p 8080:8080 -p 50000:50000 -d \
  -v jenkins-home:/var/jenkins_home \
  jenkins/jenkins
```

- `8080` is the web UI port, published to the host so the droplet address serves Jenkins
- `50000` is the port Jenkins controllers use to talk to worker nodes. It is unused in this single-node setup but published now so a cluster can be added later without recreating the container
- `jenkins-home` is a **named volume** mounted at `/var/jenkins_home`. Jenkins stores its jobs, plugins, users and credentials there, so without it a removed container means starting from scratch. Docker manages the physical path on the host, which can be inspected with `docker volume inspect jenkins-home`

### 5. Initialise Jenkins

The UI was opened at `<jenkins-ip>:8080`, which asks for the initial admin password generated at first startup:

```bash
docker exec -it <container-id> bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

The same file is reachable from the host through the named volume path, which is useful if the container is not running.

The suggested plugin set was installed (Git, Pipeline, Credentials and others), followed by creating the first admin user. At that point Jenkins is initialised and running, with no jobs defined yet.

## Verification

- `docker ps` shows the Jenkins container running with port 8080 published
- The Jenkins UI loads at `<jenkins-ip>:8080` and accepts the new admin user
- The dashboard is empty, confirming a clean install rather than a restored state
- `docker volume inspect jenkins-home` shows the volume exists and where Docker stores it

## Useful commands

```bash
docker ps -a                      # find the Jenkins container, running or stopped
docker start <container-id>       # restart Jenkins after a Docker daemon restart
docker volume inspect jenkins-home
docker exec -u 0 -it <container-id> bash   # enter the container as root
```

## Security considerations

- Only ports 22 and 8080 are open on the droplet; everything else is blocked by the firewall
- The initial admin password file is read once and the account replaced by a named admin user
- Jenkins is served over plain HTTP on port 8080 in this bootcamp setup. A production install would sit behind HTTPS with a real domain and restricted SSH sources

## Key learnings

- A build server is infrastructure in its own right: it needs its own host, its own firewall and its own lifecycle, separate from the applications it builds
- Running Jenkins as a container removes the Java and system-user setup entirely, but makes the named volume essential: the container is disposable, the volume is not
- Publishing port 50000 at install time costs nothing and avoids recreating the container later if agents are added
- Jenkins configuration lives under `/var/jenkins_home`, which is the single thing worth backing up

## Project structure

Documentation only. This folder contains the README and nothing else. The server was configured through the DigitalOcean dashboard and the shell, so there is no source code to commit.
