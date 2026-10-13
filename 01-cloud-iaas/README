# Cloud and Infrastructure as a Service Basics

This module moved my work off my pc and onto a rented cloud server. It introduced Infrastructure as a Service (IaaS) with DigitalOcean as the provider: creating an Ubuntu server (a Droplet), securing it with SSH keys and a cloud firewall, deploying a Java application to it, and setting up a non-root Linux user.

The module has one project, a hands-on follow-along from the bootcamp. I completed every step myself on my own DigitalOcean account.

## Learning objectives

- Explain IaaS and how it differs from running physical servers on-premise
- Create and size a Linux virtual server with a cloud provider
- Secure a new server with SSH key authentication and a deny-by-default firewall
- Deploy a packaged Java application to a remote server and reach it from a browser
- Check that an application is running and which port it listens on
- Set up a non-root user with sudo rights and its own SSH access

## Technologies used

DigitalOcean (Droplets, cloud firewall), Linux (Ubuntu), Java (OpenJDK), Gradle, SSH, SCP

## Shared context

**Why a cloud server.** Later modules install Nexus and Jenkins. Teams run these tools on dedicated servers, not on a developer's laptop, so this module builds that kind of server first.

**Why DigitalOcean.** The bootcamp uses DigitalOcean for its simple account setup and interface. AWS is the most widely used provider but more complex to start with. New accounts get free credits, which covered everything in this module.

**The demo application.** The project deploys the instructor's Java/React example application, built with Gradle. I did not write it, and its source is not committed to this repository.

## Projects

| Folder                                                                     | Project                                                   | Description                                                                          |
| -------------------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| [`deploy-java-app-digitalocean`](./deploy-java-app-digitalocean/README.md) | Create a server and deploy an application on DigitalOcean | Firewall an Ubuntu Droplet, deploy a Java JAR over SCP, and add a non-root sudo user |

## Key learnings

- With IaaS the provider owns and repairs the hardware. I rent virtual machines when I need them and release them when I don't.
- A new Droplet accepts traffic on every port until I add a firewall, so securing the server is part of creating it.
- Build locally, copy the artifact, run it on the server, open the port. The sequence is the same on any provider.
- Running an application as root gives it full root privileges. Each application should get its own user with only the permissions it needs, and admin work should go through a personal user with sudo.
