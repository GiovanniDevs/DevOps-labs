# Nexus Setup and Artifact Publishing

Hands-on follow-along project from Module 6 of the TechWorld with Nana DevOps Bootcamp (Artifact Repository Manager with Nexus). I completed every step on my own DigitalOcean server, which has since been destroyed. This folder holds the Gradle and Maven build files I used to publish to Nexus. The demo applications themselves are not committed.

## Overview

An artifact repository manager is where a company stores its build outputs (JAR files, npm packages, Docker images) and where build pipelines fetch dependencies from. Build tools push artifacts in, and deployment tools pull them out. Sonatype Nexus is one of the most popular options, alongside JFrog Artifactory.

In this project I installed Nexus Repository Manager 3 (OSS) on an Ubuntu server, ran it as a dedicated non-root user, opened its web UI, and published Java artifacts to it from Gradle and Maven using a least-privilege Nexus user.

## Project objectives

- Provision an Ubuntu server sized for Nexus, with access controlled by the cloud firewall
- Install Nexus Repository Manager 3 and understand how its application files are kept separate from its data
- Run Nexus as a dedicated non-root Linux user
- Make the web UI reachable and complete the initial admin setup
- Create a Nexus user and role with only the permissions needed to publish, instead of sharing admin credentials
- Publish a Java artifact to a hosted Maven repository from both Gradle and Maven, keeping credentials out of the build files

## Technologies used

| Technology                       | Role in this project                                               |
| -------------------------------- | ------------------------------------------------------------------ |
| DigitalOcean                     | Hosts the Ubuntu droplet and its firewall                          |
| Ubuntu                           | Server operating system                                            |
| OpenJDK 17                       | Java runtime required by Nexus                                     |
| Nexus Repository Manager 3 (OSS) | The artifact repository manager                                    |
| Gradle                           | Builds and publishes a Java artifact with the Maven Publish plugin |
| Maven                            | Builds and publishes a Java artifact with the Maven Deploy plugin  |
| `wget`, `tar`                    | Download and extract the Nexus archive                             |

## Prerequisites

- A server with **8 GB of RAM**. Nexus needs a lot of memory, especially as stored components grow, and 8 GB is closer to a production baseline.
- SSH access, which needs inbound TCP port 22 open in the cloud firewall before the droplet is reachable at all.
- Permission to add inbound rules to the cloud firewall.
- Gradle and Maven installed locally, with a project for each to build and publish.

## Implementation

### 1. Prepare the server and install Java

After creating the droplet I opened inbound TCP port 22 in the DigitalOcean firewall, connected over SSH, and installed OpenJDK 17:

```bash
sudo apt update
sudo apt install openjdk-17-jdk
java -version
```

Nexus supports a specific Java version, and the wrong one can cause startup failures or odd runtime errors. `java -version` confirms Java 17 is active before going any further.

### 2. Install Nexus under `/opt`

In `/opt` I downloaded the latest Nexus 3 Unix archive from Sonatype's download page with `wget` and extracted it with `tar -zxvf`. Extracting creates two directories:

- **The application directory**, named after the Nexus version: the runtime, the binaries, and the `bin/nexus` script that starts and stops the service.
- **`sonatype-work/`**: all persistent state, including configuration, repository data, stored components, logs, access logs and installed plugins.

Nexus finds `sonatype-work` through its properties file, so the two directories are independent:

- **Upgrades** replace only the application directory. `sonatype-work` stays untouched, so configuration and stored artifacts survive the upgrade.
- **Backups** only need `sonatype-work`. Restoring it recovers the whole instance after a server failure, because all of its state lives there.

### 3. Run Nexus as a dedicated user

If a service running as root is compromised, the attacker has the whole machine. Before starting Nexus for the first time I created a `nexus` user and group:

```bash
sudo adduser nexus
```

I then gave that user recursive ownership of both directories. The same command ran once for `sonatype-work` and once for the versioned application directory:

```bash
sudo chown -R nexus:nexus /opt/sonatype-work
```

Ownership alone does not make the service use that account. In the application directory I edited `bin/nexus.rc` and set:

```txt
run_as_user="nexus"
```

Then I switched to the `nexus` user and started the service from the `bin` directory:

```bash
sudo su - nexus
./nexus start
```

Last, I added an inbound TCP rule for **port 8081** in the DigitalOcean firewall. Nexus was already listening on 8081 inside the server, but the UI stayed unreachable until the cloud firewall allowed that port. It then loaded at `http://<nexus-ip>:8081`. On first login, the setup wizard asks for a new password for the default `admin` account.

### 4. Create a Nexus user and role

Nexus splits permissions into `repository_admin` privileges, for the people who administer Nexus, and `repository_view` privileges, for developers and build tools that browse, upload and download artifacts.

Instead of giving the admin credentials to every build tool, I created a non-admin user under **Security → Users** and a role under **Security → Roles** with `repository_view` privileges, which cover add, browse, edit and delete. I assigned the role to the user and used those credentials for all the publishing below.

### 5. Publish artifacts

Gradle and Maven both publish Java artifacts in the Maven repository format, so both upload to the **Maven Snapshots** hosted repository that Nexus creates by default. No new repository was needed. Snapshot versions go to Maven Snapshots, and a version without the `-SNAPSHOT` suffix would go to Maven Releases.

#### Gradle

I added the **Maven Publish plugin**, which provides the `publish` task, and a `publishing` block in [`build.gradle`](./gradle-app/build.gradle) with two parts:

- **`publications`** describes the artifact: the JAR under `build/libs`, its version and its extension.
- **`repositories`** sets the Nexus URL (server address, port 8081 and the Maven Snapshots path) and the credentials.

```groovy
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

The build file is committed, so the credentials are not in it. They live in a `gradle.properties` file, and the build file reads them as project properties. Gradle also blocks uploads over plain HTTP by default, which is why this repository needs `allowInsecureProtocol = true`.

The project name in `settings.gradle` combines with the version to form the JAR filename. After refreshing the project so Gradle resolved its dependencies, I built and published:

```bash
gradle build
gradle publish
```

This uploaded `my-app-1.0-SNAPSHOT.jar` to Maven Snapshots under the `com.example` group.

#### Maven

In [`pom.xml`](./maven-app/pom.xml) I added the **Maven Deploy plugin** and a `<distributionManagement>` block pointing at Maven Snapshots, with the ID `nexus-snapshots`:

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://<nexus-ip>:8081/repository/maven-snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

The ID links the project to its credentials. These live in `~/.m2/settings.xml`, Maven's global settings file, which sits outside the project and outside version control. It has a `<servers>` entry with the same ID, plus the Nexus username and password.

```bash
mvn package
mvn deploy
```

`mvn package` builds the JAR into `target/`, and `mvn deploy` runs the deploy plugin to upload it. This published `java-maven-app-1.1.0-SNAPSHOT.jar`, also under `com.example`, so the Gradle and Maven artifacts appear together in Nexus.

## Verification

I checked each stage as I went rather than only at the end:

```bash
java -version                    # Java 17 is active before installing Nexus
ps aux | grep nexus              # the Nexus process is running
sudo netstat -lptn | grep 8081   # Nexus is listening on port 8081
```

- Loading `http://<nexus-ip>:8081` in a browser and completing the setup wizard confirmed that the service was healthy and the firewall rule was correct.
- After each publish I opened **Browse → Components → Maven Snapshots**. Both `my-app-1.0-SNAPSHOT.jar` and `java-maven-app-1.1.0-SNAPSHOT.jar` were there under `com.example`, each with its assets. I relied on this view, not on the build tool's success message, to confirm the artifact was stored.

## Useful commands

```bash
# Start and stop Nexus (as the nexus user, from the bin directory)
./nexus start
./nexus stop

# Publish from Gradle
gradle build
gradle publish

# Publish from Maven
mvn package
mvn deploy
```

## Security considerations

- **Nexus does not run as root.** The `nexus` user owns both directories, and `nexus.rc` makes the service start under that account. A compromised service gets a restricted account, not the whole machine.
- **Build tools use a least-privilege user.** Publishing uses a Nexus user with `repository_view` privileges only, and the admin account is not shared.
- **Credentials stay out of version control.** Gradle reads them from `gradle.properties` and Maven from `~/.m2/settings.xml`. Neither `build.gradle` nor `pom.xml` contains a password, and neither credentials file is committed.
- **Only ports 22 and 8081 are open** in the cloud firewall.
- **HTTP is a known limitation.** Nexus is reached over plain HTTP, so credentials and artifacts travel unencrypted. That is acceptable on a short-lived learning server, but a real deployment would need TLS.

## Key learnings

- `sonatype-work` is the whole instance. Keeping data separate from the application makes upgrades (replace the application directory) and backups (copy `sonatype-work`) simple.
- A dedicated service user takes one command to create and limits what a compromised service can reach.
- Splitting `repository_admin` from `repository_view` gives developers and CI tools the access they need without passing admin credentials around.
- Gradle and Maven both keep credentials in a separate file and link them to the project by a property name or a matching ID.
- The `-SNAPSHOT` suffix decides whether an artifact goes to Maven Snapshots or Maven Releases.
- Nexus listening on 8081 was only half the job. The cloud firewall is a separate layer, and it is the first thing to check when a service is running but unreachable.
- Nexus describes every format with two terms: a component is the package you upload, and assets are the files inside it. That is how one tool can serve Maven, npm, Docker and other formats.

## Project structure

```txt
nexus-setup-and-artifact-publishing/
├── README.md
├── gradle-app/
│   └── build.gradle   # Maven Publish plugin and publishing block
└── maven-app/
    └── pom.xml        # Maven Deploy plugin and distributionManagement block
```

The demo applications were the instructor's, so only their build files are committed, to show the Nexus publishing configuration.
