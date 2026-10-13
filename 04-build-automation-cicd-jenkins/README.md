# Build Automation & CI/CD with Jenkins

Module 8 of the DevOps Bootcamp, and the point where the course moves from prerequisites to DevOps proper. Five hands-on projects, from installing Jenkins on a cloud server to a self-versioning CI pipeline that builds, tags, pushes and commits on every change.

## Overview

Building and publishing artifacts by hand does not scale: it interrupts whoever is doing it, it depends on their local setup, and it is impossible to audit. This module replaces that with a dedicated build server that does the work on every push.

The projects follow the order the lessons were delivered. Each one adds a capability to the same pipeline: first a server, then a CI workflow expressed three different ways, then reusable pipeline code, then automatic triggering, and finally automated versioning.

## Learning objectives

- Explain what build automation and continuous integration solve, and where Jenkins sits between the repository, the build tools and the registry
- Install and operate Jenkins as a Docker container on a cloud server
- Configure build tools, plugins and credentials for an instance shared by multiple jobs
- Write declarative pipelines as a `Jenkinsfile` versioned with the application code
- Use multibranch pipelines so every branch is tested while only the main line is built and shipped
- Extract common pipeline logic into a Jenkins Shared Library
- Trigger pipelines automatically from repository events
- Produce uniquely versioned artifacts without manual version management

## Technologies used

Jenkins, Groovy, Docker, Docker Hub, Git and GitHub, Java 17, Maven, DigitalOcean, Linux (Ubuntu)

## Shared context

All five projects share one environment, described once here rather than in each project README.

- **Jenkins server**: a 4 GB DigitalOcean droplet named `Jenkins-Server`, running Jenkins from the official `jenkins/jenkins` image with its data on the `jenkins-home` named volume. Only ports 22 and 8080 are open. Referred to throughout as `<jenkins-ip>`
- **Demo application**: the bootcamp's Java Maven application (Spring Boot, Java 17), in my own repository at `github.com/GiovanniDevs/Module8-jenkins`. The application source is not committed to this monorepo; these projects document the CI work around it
- **Registry**: images are pushed to my public Docker Hub repository `giovannidevs/demo-app`
- **Credentials**: two global Jenkins credentials, `dockerhub-credentials` for the registry, and `gitlab-credentials`, which despite its name holds my GitHub username and personal access token
- **Git platform**: the lectures use GitLab throughout; I did the whole module on GitHub, which mattered most in project 04, where the webhook setup differs substantially
- **Build tool**: Maven, configured in Manage Jenkins → Tools under the name `maven-3.9`, which is the exact string every Jenkinsfile's `tools` block uses

## Projects

| Folder                             | Project                         | Description                                                                                                                         |
| ---------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 01-install-jenkins-on-digitalocean | Install Jenkins on DigitalOcean | Ubuntu droplet, firewall, Docker, and Jenkins running as a container on a persistent volume                                         |
| 02-ci-pipeline-with-jenkinsfile    | CI pipeline with Jenkinsfile    | Build tools, Docker access and credentials, then the same CI workflow as a freestyle job, a pipeline job and a multibranch pipeline |
| 03-jenkins-shared-library          | Jenkins Shared Library          | Common build and Docker logic extracted into a separate Groovy library repository and imported by the Jenkinsfile                   |
| 04-webhook-trigger-ci-pipeline     | Webhook trigger                 | Push-triggered builds for both a regular pipeline and a multibranch pipeline, configured for GitHub                                 |
| 05-dynamic-version-increment       | Dynamic version increment       | Automatic patch bump, versioned image tags, version committed back to Git, and build-loop prevention                                |

## Key learnings

- Jenkins is a middleman, not a build tool: almost all of the work is connecting it to things (repositories, build tools, registries, credentials), and most failures happen at those joins
- The jump from freestyle jobs to a `Jenkinsfile` is the same jump as from clicking in a cloud console to infrastructure as code, and brings the same benefits: review, history and branching
- Naming is load-bearing. Tool names, credential IDs and shared library names are matched as plain strings, and a mismatch fails in ways that rarely point at the name
- Running Jenkins in a container is convenient until it touches the host: mounting the Docker socket works, but any daemon restart stops Jenkins and resets the permissions it depends on
- Automation closes loops you then have to open deliberately. A pipeline that commits to the repository it watches will trigger itself forever unless told not to
- Following lessons on a different Git platform was more useful than copying them: it forced me to understand what each step was for rather than which button to click
