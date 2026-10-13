# Module 6: Artifact Repository Manager with Nexus

This module covered artifact repositories and how to run one. I installed Sonatype Nexus Repository Manager on a DigitalOcean server, ran it as a non-root service user, set up a least-privilege Nexus user, and published Java artifacts to it from Gradle and Maven.

The module has one project, a hands-on follow-along from the bootcamp. I completed every step myself on my own server.

## Learning objectives

- Explain what an artifact repository manager does and where it sits in a CI/CD process
- Install Nexus on a Linux server and keep its application files separate from its data
- Run a third-party service as a dedicated non-root user
- Control access to Nexus with users and roles based on least privilege
- Publish artifacts to a hosted Maven repository from Gradle and Maven without putting credentials in the build files

## Technologies used

Nexus Repository Manager 3 (OSS), DigitalOcean, Linux (Ubuntu), OpenJDK 17, Gradle, Maven

## Shared context

**The server.** Nexus ran on an 8 GB DigitalOcean droplet, set up with the same habits as [Module 5](../01-cloud-iaas/README.md): a cloud firewall that opens only the ports in use, and a dedicated Linux user for the service.

**The demo applications.** The instructor provided the two Java applications used as publishing sources, one built with Gradle and one with Maven. Only their build files are committed, because that is where the Nexus publishing configuration lives.

**Later rebuild.** In Module 7 I rebuilt this Nexus setup on Oracle Cloud and used it as a private Docker registry.

## Projects

| Folder                                                                                   | Project                             | Description                                                                                        |
| ---------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| [`nexus-setup-and-artifact-publishing`](./nexus-setup-and-artifact-publishing/README.md) | Nexus setup and artifact publishing | Install Nexus as a non-root service on a cloud server and publish JARs to it from Gradle and Maven |

## Key learnings

- An artifact repository manager gives a team one place to store build outputs and fetch dependencies, so artifacts move from build to deployment through Nexus instead of being copied straight onto servers.
- Hosted repositories hold a company's own artifacts. Nexus creates Maven Releases and Maven Snapshots by default, one for tested versions and one for versions still in development.
- Running a service involves more than installing it: deciding what to back up, which user it runs as, and which ports the firewall opens.
