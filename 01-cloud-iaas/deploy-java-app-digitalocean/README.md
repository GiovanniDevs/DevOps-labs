# Create Server and Deploy Application on DigitalOcean

> Hands-on follow-along project from the TechWorld with Nana DevOps Bootcamp (Module 5 — Cloud & Infrastructure as a Service Basics). This folder is documentation-only: the application was the instructor's demo project and is not committed here.

## Overview

This project was my first hands-on work with Infrastructure as a Service. Instead of installing and running tools on my own laptop, I provisioned a Linux virtual server from a cloud provider and deployed a packaged Java application onto it — which is how applications actually run in practice.

The motivation comes from the rest of the bootcamp: tools like Nexus and Jenkins are meant to run on dedicated remote servers, not on a developer machine, so I needed a server I could provision, secure, connect to, and deploy onto. IaaS makes that practical: the provider owns and maintains the physical hardware, and I rent a virtual machine on demand and release it when I no longer need it.

DigitalOcean was the provider used in this module. AWS is the most widely used and most powerful option, but also the most complex to start with, so DigitalOcean's simpler account setup and interface made it a better place to learn the fundamentals. On DigitalOcean, servers are called **Droplets** and are Linux-based virtual machines.

## Project objectives

- Set up and configure a server on DigitalOcean
- Create and configure a new Linux user on the Droplet, as a security best practice
- Deploy and run a Java Gradle application on the Droplet

## Technologies used

- **DigitalOcean** — Droplets and cloud firewall
- **Ubuntu Linux** — the Droplet operating system
- **Java** — headless OpenJDK JRE on the server
- **Gradle** — building the application artifact locally
- **SSH / SCP** — authentication, remote access, and artifact transfer

## Workflow

The deployment follows a sequence that stays broadly the same across cloud providers:

```txt
build locally  →  transfer artifact  →  run on server  →  open the port
  gradle build        scp over SSH        java -jar        cloud firewall
```

## Prerequisites

- A DigitalOcean account. New signups receive free credits, which covered this project.
- An SSH key pair on my local machine.
- Java and Gradle installed locally, to build the application before transferring it.

## Implementation

### 1. SSH keys and Droplet creation

Before creating the Droplet I added my public SSH key to my DigitalOcean account. Keys registered this way are stored account-wide and can be reused across multiple Droplets, so this is done once rather than per server. This is conceptually identical to adding an SSH key to GitLab for repository access.

I then created a Droplet with an Ubuntu image, the smallest available basic plan — more than enough for a demo workload — and the London region, which is closest to me. The plan determines the CPU, memory, and storage allocated to the server.

I chose SSH key authentication rather than password access. Passwords can be brute-forced; key-based authentication cannot in any practical sense, and it also removes the need to type a password on every connection.

### 2. Firewall baseline

By default a new Droplet is publicly reachable on all ports, which is a security risk. The correct approach is the opposite: deny everything by default and open only what is genuinely needed.

I created a Droplet firewall with:

- **Inbound** — SSH on port 22, with the source restricted to my own public IP address rather than all IPv4/IPv6. Everything else is dropped.
- **Outbound** — all traffic allowed, so the server can reach package repositories to download tools such as Java.

Firewall configurations exist at the account level and can be assigned to several Droplets, so one well-defined rule set can be reused.

### 3. Connecting and installing Java

With port 22 open to my IP, I connected over SSH. No password is requested, because the key handles authentication:

```bash
ssh root@<server-ip>
```

The server starts as a completely clean Ubuntu instance with nothing pre-installed. Running `java` confirmed this — the command is not found, but Ubuntu helpfully prints the packages that would provide it. I refreshed the package index and installed a headless OpenJDK JRE, which is all that is needed to run a packaged application (no GUI libraries, no compiler):

```bash
apt update
apt install <openjdk-headless-jre-package>
```

The same `apt install` approach is how other tools, such as Docker or Node.js, would be added to the server.

### 4. Building, transferring, and running the application

The application is the instructor's Java/React example project, built with Gradle. I cloned it locally and built it from the project root:

```bash
gradle build
```

Gradle compiles the sources and produces a JAR file under `build/libs/`. I transferred that artifact to the server with SCP, which copies files over the existing SSH connection — no FTP server or extra service required:

```bash
scp build/libs/java-react-example.jar root@<server-ip>:/root
```

The three parts of that command are the local source path, the user and host to connect as, and the destination path on the remote server. After confirming the file had arrived with `ls`, I started it:

```bash
java -jar java-react-example.jar
```

The application is a Spring Boot app with an embedded Tomcat server, and the startup logs show the port it binds to — **7071** in this case. Knowing the actual port matters, because it is what the firewall rule has to match.

At this point the application was running but still unreachable from a browser, since nothing in the firewall allowed traffic to 7071. I added a second inbound rule:

- Type: Custom TCP
- Port: 7071
- Source: all sources

The rule applied immediately and the application became reachable. This is the key lesson of the step: a running application is not an accessible application until the cloud firewall allows traffic to its port.

I also ran the application in **detached mode**. In attached mode the process holds the terminal: output is visible in real time, but the session is occupied and closing it stops the application. Detached mode puts the process in the background, freeing the terminal and letting the application survive disconnecting from SSH — which is how server applications are run in practice. The exact command used in the demo is not recorded in my notes, so I have deliberately not reproduced it here.

### 5. Creating a non-root Linux user

On a fresh Droplet all work happens as `root`. Running applications as root gives them full root privileges over the machine, so the recommended practice is a dedicated user per application with only the permissions that application needs. Even administrative work should go through a personal non-root user that escalates privileges only when required. The official documentation for Nexus and Jenkins makes the same recommendation.

I created a new user, which also sets a password and creates a matching group, and added it to the `sudo` group so it can escalate privileges per command rather than logging in as root:

```bash
adduser <username>
usermod -aG sudo <username>
su - <username>
```

SSH login as the new user initially failed, which makes sense: the key I registered in the DigitalOcean interface was only installed for `root`. Authorised keys are per-user, so I configured the new user's own key from the command line:

```bash
mkdir -p /home/<username>/.ssh
sudo vim /home/<username>/.ssh/authorized_keys
```

I pasted my public key into that file and saved it, after which SSH login as the new user worked directly. For the root user I ended up with two public keys in `authorized_keys`, one for each of the local machines I connect from.

## Verification

I confirmed the deployment four ways:

1. **Browser access** — opening `http://<server-ip>:7071` served the application, confirming the end-to-end path: artifact on the server, process running, port open.
2. **Process check** — confirming the Java process is alive and seeing its process ID:

   ```bash
   ps aux | grep java
   ```

3. **Listening port check** — confirming the application really is bound to 7071, and which process owns it. `netstat` is not installed by default on a fresh Ubuntu Droplet:

   ```bash
   apt install net-tools
   netstat -lpt
   ```

   `-l` lists listening sockets, `-p` shows the owning process ID and name, and `-t` limits output to TCP. Adding `-n` shows numeric addresses and ports instead of resolving them to names.

4. **Non-root SSH login** — connecting as the newly created user succeeded without a password, confirming its `authorized_keys` was configured correctly.

## Useful commands

| Purpose                             | Command                                                        |
| ----------------------------------- | -------------------------------------------------------------- |
| Connect to the Droplet              | `ssh root@<server-ip>`                                         |
| Refresh the package index           | `apt update`                                                   |
| Build the application artifact      | `gradle build`                                                 |
| Copy the artifact to the server     | `scp build/libs/java-react-example.jar root@<server-ip>:/root` |
| Run the application                 | `java -jar java-react-example.jar`                             |
| Find the running Java process       | `ps aux \| grep java`                                          |
| Install net-tools                   | `apt install net-tools`                                        |
| List listening TCP ports and owners | `netstat -lpt`                                                 |
| Create a user                       | `adduser <username>`                                           |
| Grant sudo rights                   | `usermod -aG sudo <username>`                                  |
| Switch user                         | `su - <username>`                                              |

## Security considerations

- **Deny by default, open selectively.** A new Droplet accepts traffic on every port. The firewall closes that down and opens only port 22 and the single application port.
- **Restrict the SSH source.** Port 22 is open only to my own public IP, not to the whole internet, which removes the server from the reach of broad SSH scanning.
- **Keys, not passwords.** SSH key authentication is used for all access to the server.
- **Authorised keys are per-user.** A key registered through the provider's interface only reaches the root user; every additional user needs its own `authorized_keys`.
- **Don't work as root.** Each application should run under its own dedicated user with the minimum permissions it needs, and administrators should use personal accounts that escalate via `sudo`.
- **The application port was opened to all sources for demonstration purposes.** In a real deployment that source range would be narrowed, or the application placed behind a reverse proxy or load balancer rather than exposed directly.
- **No credentials, keys, or server addresses appear in this README.** Server IPs, usernames, and package versions are shown as placeholders.

## Troubleshooting

**SSH connections suddenly fail.** The firewall allows inbound SSH only from one specific IP address, and home IP addresses are frequently reassigned when a router or machine restarts. If a connection that previously worked starts timing out, the first thing to check is whether my public IP has changed and the firewall source needs updating — not the server itself.

## Key learnings

- IaaS removes hardware ownership from the equation: the provider handles physical servers, failures, and networking, and I provision and release virtual machines on demand.
- A cloud server is unprotected until I protect it. Security is an explicit configuration step, not a default.
- The deployment pattern — build locally, transfer the artifact, run it on the server, open network access — is consistent across providers, which makes it transferable knowledge rather than DigitalOcean trivia.
- SCP makes artifact deployment trivially simple, because it reuses the SSH trust already established.
- A running process and a reachable service are two different things. Diagnosing "it's running but I can't open it" means checking the cloud firewall, not the application.
- Detached execution is not a convenience but a requirement for anything meant to outlive an SSH session.
- Least privilege is enforced through Linux users: one user per application, sudo for escalation, and root reserved for nothing routine.

## Project structure

This folder contains only this `README.md`. The application deployed in this project was the instructor's demo project, so no source files are committed here.

## Acknowledgements

This project was completed as part of the **DevOps Bootcamp by TechWorld with Nana**, Module 5 — Cloud & Infrastructure as a Service Basics. The tutorial, demo application, and overall structure are the instructor's; the implementation documented here is my own hands-on work following along.
