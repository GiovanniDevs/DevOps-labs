# Nexus Setup and Artifact Publishing

> **Status:** Complete — documentation only.
> Hands-on follow-along project from Module 6 (_Artifact Repository Manager with Nexus_) of my DevOps bootcamp. I personally completed every step described here on my own cloud server. The applications containes secrets but none are sensitive as the droplet and credentials were destroyed.

## Overview

An artifact repository manager is the central place where a company stores the build outputs of its software — JAR files, npm packages, Docker images — and where its build pipelines fetch dependencies from. Sonatype Nexus is one of the most widely used tools for this, and it sits in the middle of a typical CI/CD process: build tools push artifacts into it, and deployment tools pull artifacts out of it.

In this project I installed and ran Sonatype Nexus Repository Manager 3 (OSS) on an Ubuntu cloud server, configured it to run securely as a dedicated non-root service user, exposed its web UI, and then published artifacts to it from three different build tools: Gradle, Maven, and npm.

The flow is straightforward: a build tool on my local machine authenticates against Nexus over HTTP and uploads its build output into a hosted repository on the droplet, where it becomes available to anyone else with access to that repository.

## Project objectives

- Provision an Ubuntu cloud server sized appropriately for Nexus, with firewall access controlled at the cloud level
- Install Nexus Repository Manager 3 and understand the separation between its application files and its persistent data
- Run Nexus as a dedicated non-root Linux user rather than as root
- Make the Nexus web UI reachable and complete initial administrator setup
- Create a Nexus user and role following the principle of least privilege, instead of sharing admin credentials with developers
- Publish a Java artifact to a hosted Maven repository from both Gradle and Maven, keeping credentials out of the build files
- Create a hosted npm repository and publish a Node.js package to it

## Technologies used

| Technology                                | Role in this project                                           |
| ----------------------------------------- | -------------------------------------------------------------- |
| DigitalOcean                              | Cloud provider hosting the Ubuntu droplet and its firewall     |
| Ubuntu                                    | Server operating system                                        |
| OpenJDK 17                                | Java runtime required by Nexus                                 |
| Sonatype Nexus Repository Manager 3 (OSS) | The artifact repository manager itself                         |
| Gradle                                    | Java build tool, publishing via the Maven Publish plugin       |
| Maven                                     | Java build tool, publishing via the Maven Deploy plugin        |
| npm                                       | Node.js package manager, publishing to a hosted npm repository |
| `wget`, `tar`                             | Downloading and extracting the Nexus distribution              |
| Linux user and permission management      | Running the service under a restricted account                 |

## Prerequisites

- A cloud server with **8 GB of RAM**. Nexus is memory-hungry, especially as it accumulates stored components, and this size keeps the instance stable and closer to a production baseline.
- SSH access to the server, which requires inbound TCP port 22 to be open in the cloud firewall before the droplet is reachable at all.
- The ability to add inbound firewall rules at the cloud provider level.
- Maven, Gradle, and npm installed locally, with projects available to build and publish.

## Implementation

### 1. Server preparation and Java

After creating the droplet, I opened inbound TCP port 22 in the DigitalOcean firewall so I could connect over SSH, then updated the package index and installed OpenJDK 17:

```bash
sudo apt update
sudo apt install openjdk-17-jdk
java -version
```

Java 17 is not an arbitrary choice — it is the version Nexus supports. Running Nexus against a different Java version commonly results in startup failures or unpredictable runtime behaviour, so confirming the active version with `java -version` before going further is worth the few seconds it takes.

### 2. Installing Nexus under `/opt`

I moved into `/opt`, downloaded the latest Nexus Unix archive from Sonatype's official download page, and extracted it:

```bash
cd /opt
sudo wget <nexus-download-url>
sudo tar -zxvf <nexus-archive-filename>
```

Extracting the archive creates **two** directories, and the distinction between them is the single most important concept in the installation:

- **`nexus-3.x.y-nn/`** — the application itself: the Nexus runtime, its binaries, and the executable used to start and stop the service.
- **`sonatype-work/`** — all persistent state: custom configuration, repository data, stored components and assets, logs, access logs, and installed plugins.

Nexus locates `sonatype-work` through its properties file, which means the two directories are genuinely independent. This has two practical consequences:

- **Upgrades** replace only the versioned application directory. `sonatype-work` is left untouched, so configuration and stored artifacts survive the upgrade.
- **Backups** only need to cover `sonatype-work`. If the server is lost, restoring that directory recovers the Nexus instance, because all of its state lives there.

### 3. Running Nexus as a dedicated user

Running a network-facing service as root means that compromising the service compromises the whole machine. Creating a dedicated account limits the blast radius, so before starting Nexus for the first time I created a `nexus` user and the corresponding group:

```bash
sudo adduser nexus
```

I then transferred ownership of both the application and the data directory to that user:

```bash
sudo chown -R nexus:nexus /opt/nexus-3.x.y-nn
sudo chown -R nexus:nexus /opt/sonatype-work
```

Ownership alone is not enough — the startup script needs to be told which account to run under. Inside the extracted Nexus directory I edited `bin/nexus.rc` and set:

```txt
run_as_user="nexus"
```

With that in place I switched to the `nexus` user and started the service from the `bin` directory of the extracted Nexus folder:

```bash
sudo su - nexus
./nexus start
```

Finally, I added a custom inbound TCP rule for **port 8081** in the DigitalOcean firewall. Nexus listening on the port inside the server is only half the requirement; until the cloud firewall also allows that port, the UI stays unreachable from outside. The UI then loaded at:

```txt
http://<nexus-ip>:8081
```

On first login Nexus provides a default `admin` account and a setup wizard that requires an immediate password change.

### 4. Nexus user and role setup

Nexus permissions are split into two meaningful groups: `repository_admin` privileges, intended for the people administering Nexus, and `repository_view` privileges, intended for developers and build tools that need to browse, upload and download artifacts.

Rather than handing out the admin credentials to every build tool, I created a dedicated non-admin user under **Security → Users** and a new role under **Security → Roles**, granting it `repository_view` privileges so it could add, browse, edit and delete components. I then assigned that role to the new user, and used those credentials for every publishing step below.

### 5. Publishing artifacts

Both Gradle and Maven publish Java artifacts using the Maven repository format, and both upload into the default **Maven Snapshots** hosted repository that Nexus creates out of the box. No new repository was needed for the Java artifacts.

#### Gradle

I added the **Maven Publish plugin** to the Gradle build, which provides the `publish` task, then configured a `publishing` block containing two parts:

- **`publications`** — describing the artifact to upload: the JAR produced under `build/libs`, its version, and its extension.
- **`repositories`** — the target Nexus URL, built from the server address, port 8081, and the path to the Maven Snapshots repository, together with the credentials to authenticate with.

```
publishing {
    publications {
        maven(MavenPublication) {
            artifact("build/libs/my-app-$version"+".jar"){
                extension 'jar'
            }
        }
    }

    repositories {
        maven {
            name 'nexus'
            url "http://<nexus-ip>:8081/repository/maven-snapshots/"
            allowInsecureProtocol = true
            credentials {
                username project.repoUser
                password project.repoPassword
            }
        }
    }
}
```

Credentials are deliberately **not** hardcoded in the build file, since that file is committed to version control. Instead I stored them in a `gradle.properties` file as `repo_user` and `repo_password`, and referenced them from the build file via `project.repoUser` and `project.repoPassword`.

Because Nexus is being accessed over plain HTTP rather than HTTPS, Gradle blocks the upload by default. I had to explicitly allow the insecure protocol on that repository definition for the publish to succeed.

The project name is set in `settings.gradle` and combines with the version to form the JAR filename. After refreshing the project so Gradle resolved its dependencies, I built and published:

```bash
gradle build
gradle publish
```

This uploaded `my-app-1.0-SNAPSHOT.jar` to Maven Snapshots under the `com.example` group.

#### Maven

Maven's configuration lives in `pom.xml`. I added the **Maven Deploy plugin**, then a `<distributionManagement>` block containing a `<snapshotRepository>` entry with the Nexus Maven Snapshots URL and an `<id>` of `nexus-snapshots`.

```
    <distributionManagement>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <url>http://<nexus-ip>8081/repository/maven-snapshots</url>
        </snapshotRepository>
    </distributionManagement>
```

That ID is the link between the project and its credentials. The credentials themselves go in `~/.m2/settings.xml` — Maven's global settings file, outside the project and therefore outside version control — inside a `<servers>` block whose `<id>` matches the one declared in `pom.xml`, along with the Nexus username and password.

```bash
mvn package
mvn deploy
```

`mvn package` builds the JAR into the `target/` folder, and `mvn deploy` triggers the deploy plugin to upload it. This published `java-maven-app-1.1.0-SNAPSHOT.jar`, also under the `com.example` group — so both the Gradle and the Maven artifacts appear together in Nexus, grouped by their shared group ID.

## Verification

I verified the setup at each stage rather than only at the end:

**Java version is correct before installing Nexus:**

```bash
java -version
```

**The Nexus process is running after starting the service:**

```bash
ps aux | grep nexus
```

**Nexus is listening on the expected port:**

```bash
sudo netstat -lptn | grep 8081
```

**The UI is reachable from outside the server** — loading `http://<nexus-ip>:8081` in a browser and completing the admin setup wizard confirmed both that the service was healthy and that the cloud firewall rule was correct.

**Artifacts actually arrived** — after each publish I checked **Browse → Components → Maven Snapshots** in the Nexus UI. Both `my-app-1.0-SNAPSHOT.jar` and `java-maven-app-1.1.0-SNAPSHOT.jar` were present under the `com.example` group, each shown with its associated assets. The npm package was confirmed the same way in its hosted repository.

A useful detail here: the build tools reporting success is not in itself proof. Checking the repository contents in the UI is what actually confirms the artifact was stored and is retrievable.

## Useful commands

```bash
# Start and stop Nexus (run as the nexus user, from the bin directory
# of the extracted Nexus folder)
./nexus start
./nexus stop

# Check the service is running
ps aux | grep nexus

# Check the listening port
sudo netstat -lptn | grep 8081

# Publish from Gradle
gradle build
gradle publish

# Publish from Maven
mvn package
mvn deploy
```

## Security considerations

- **Nexus does not run as root.** A dedicated `nexus` user owns both the application and data directories, and `nexus.rc` enforces that the service starts under that account. If the service is compromised, the attacker inherits a restricted account rather than the whole machine.
- **Build tools authenticate as a least-privilege user.** A dedicated Nexus user with only `repository_view` privileges is used for publishing, rather than sharing the admin account.
- **Credentials are kept out of version control.** Gradle reads them from `gradle.properties`, and Maven reads them from the global `~/.m2/settings.xml`. Neither the build file nor `pom.xml` contains a password.
- **Network exposure is minimal.** Only ports 22 and 8081 are opened in the cloud firewall.
- **HTTP is a known limitation.** This setup accesses Nexus over plain HTTP, which is why Gradle had to be explicitly told to allow an insecure repository target. Credentials and artifacts travel unencrypted, which is acceptable for a learning environment but would need TLS in any real deployment.

## Key learnings

- **The `sonatype-work` directory is the whole instance.** Separating application binaries from persistent state is what makes both upgrades and backups simple: replace the versioned application folder to upgrade, back up `sonatype-work` to protect the data.
- **Service accounts are a cheap, high-value security control.** Creating a dedicated user takes one command, and it meaningfully limits what a compromise of the service can reach.
- **Least privilege matters more as soon as more than one person is involved.** Splitting `repository_admin` from `repository_view` means developers and CI tools get exactly the access they need, and admin credentials never circulate.
- **Credentials belong outside the build file.** Both Gradle and Maven provide a dedicated place for them, and both link project configuration to credentials through a matching identifier rather than inline values.
- **Snapshots and releases are a deliberate separation**, not just a naming convention — the version suffix routes the artifact into a different repository with a different purpose.
- **An open port on the server is not an open port on the network.** Nexus listening on 8081 was only half the job; the cloud firewall is a separate layer that has to be configured independently. This is the first thing to check when a service is running but unreachable.
- **Nexus abstracts every format behind the same two concepts.** A component is the package you upload; assets are the individual files inside it. That consistent model is what lets one tool and one endpoint serve Maven, npm, Docker and everything else.

## Project structure

This project is documentation-only — the folder contains only this `README.md`. The applications used as publishing sources were the instructor's demo projects, so committing them here would add nothing.
