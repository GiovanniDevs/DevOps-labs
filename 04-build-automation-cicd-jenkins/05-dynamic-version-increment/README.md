# Dynamically Increment the Application Version in the Pipeline

Hands-on follow-along project from Module 8 of the DevOps Bootcamp. Final project in the module. Builds on `02-ci-pipeline-with-jenkinsfile` and `04-webhook-trigger-ci-pipeline`; every build now produces a uniquely versioned artifact, commits the new version back to Git, and does so without retriggering itself.

## Overview

Up to this point the pipeline pushed a fixed tag, so each build overwrote the last image. That makes it impossible to say which build produced which artifact, or to roll back to a known one.

This project makes the version part of the pipeline. Maven increments the patch version in `pom.xml`, the pipeline reads the new value back out and combines it with the Jenkins build number to tag the image, and a final stage commits the updated `pom.xml` to the repository so the next build carries on from there. That last step creates a loop with the webhook from project 04, which is solved with a build strategy that ignores Jenkins' own commits.

## Project objectives

- Increment the patch version automatically during the build
- Tag the Docker image with `<version>-<build-number>`
- Commit the version change back to the repository from Jenkins
- Stop those commits from triggering another build

## Technologies used

Jenkins, Groovy, Maven, Docker, Docker Hub, Git and GitHub, Java 17

## Implementation

### 1. Increment the version with Maven

Maven keeps the application version in `pom.xml`. `build-helper:parse-version` splits the current version into parts and also calculates the next value for each; `versions:set` writes a new version; `versions:commit` removes the backup file that `versions:set` leaves behind.

```groovy
stage('increment version') {
    when {
        expression {
            BRANCH_NAME == 'main'
        }
    }
    steps {
        script {
            echo 'incrementing app version...'
            sh 'mvn build-helper:parse-version versions:set \
                -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                versions:commit'
            def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
            def version = matcher[0][1]
            env.IMAGE_NAME = "$version-$BUILD_NUMBER"
        }
    }
}
```

Only the patch part advances, which is the right default when every commit to the main branch builds. Swapping `nextIncrementalVersion` for `nextMinorVersion` bumps the minor version instead.

The new version is read straight back out of the file with a regex: `matcher[0][1]` is the text inside the first `<version>` tag. It is then combined with Jenkins' built-in `BUILD_NUMBER` and stored in `env.IMAGE_NAME`, which makes it available to every later stage.

The dollar signs need triple backslashes so that they survive the Groovy string and reach Maven as literal `${...}` placeholders rather than being expanded by the shell.

### 2. Build with a clean target directory

```groovy
sh 'mvn clean package'
```

`clean` is not optional here. Each build produces a JAR named after the new version, and the Dockerfile picks it up with a wildcard. Without `clean`, old JARs accumulate in `target/`, the wildcard matches several files, and the image ends up with more than one JAR and no way to know which to run.

### 3. Dockerfile with a wildcard

```docker
FROM amazoncorretto:17-alpine-jdk

EXPOSE 8080

COPY ./target/java-maven-app-*.jar /usr/app/
WORKDIR /usr/app

CMD java -jar java-maven-app-*.jar
```

`CMD` has to be in shell form. The exec form, `CMD ["java", "-jar", "*.jar"]`, takes the asterisk literally because there is no shell to expand it.

### 4. Tag and push the versioned image

```groovy
withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
    sh "docker build -t giovannidevs/demo-app:${IMAGE_NAME} ."
    sh 'echo $PASS | docker login -u $USER --password-stdin'
    sh "docker push giovannidevs/demo-app:${IMAGE_NAME}"
}
```

### 5. Commit the version back to Git

Without this, the increment only ever exists in the Jenkins workspace, so every build starts from the same version in Git and produces the same number again.

```groovy
stage('commit version update') {
    when {
        expression {
            BRANCH_NAME == 'main'
        }
    }
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'gitlab-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                sh 'git config --global user.email "jenkins@example.com"'
                sh 'git config --global user.name "jenkins"'

                sh 'git remote set-url origin https://${USER}:${PASS}@github.com/GiovanniDevs/Module8-jenkins.git'
                sh 'git add .'
                sh 'git commit -m "ci: version bump"'
                sh 'git push origin HEAD:main'
            }
        }
    }

```

Three things here are not obvious:

- Git refuses to commit without `user.email` and `user.name`, so they are set in the pipeline. They only need setting once, but leaving the commands in is harmless
- The remote URL is rewritten with the credentials embedded, because the Jenkins workspace has no SSH key or credential helper
- Jenkins checks out a **commit**, not a branch, so the workspace is in detached HEAD state. `git push origin HEAD:main` is required, because a plain `git push` has no branch to update

Despite the credential ID, these are my GitHub credentials; the ID was misnamed when it was created.

### 6. Break the build loop

With the webhook from project 04 active, Jenkins' own commit triggers the next build, which commits again, indefinitely. The fix is the **Ignore Committer Strategy** plugin: in the multibranch job, under Branch sources → **Build strategies**, add _Ignore committer strategy_, list `jenkins@example.com` as an ignored author, and tick the option that still allows builds from everyone else.

This prevents the pipeline from starting at all for those commits. Plugins that skip builds based on a commit-message keyword work too, but they leave a trail of skipped builds in the history.

The `target/` folder was also added to `.gitignore`, so the automated `git add .` only ever commits `pom.xml`.

![Steges](screenshots/5a.png)
_Jenkins pipeline stages showing the version increment and build process._

![DockerHub](screenshots/5b.png)
_Docker Hub showing the newly versioned image tag._

## Verification

- The console output shows the version change, for example processing a change from `1.1.0-SNAPSHOT` to `1.1.1`, followed by a clean Maven build
- Docker Hub shows a new tag in `<version>-<build-number>` form for every run, instead of an overwritten fixed tag
- The repository shows a commit authored by `jenkins <jenkins@example.com>` with the message `ci: version bump`, containing only the `pom.xml` change
- After adding the ignore rule, that commit no longer starts a build: the build number stops advancing on its own, while a commit of mine still triggers one

## Key learnings

- A version increment that is not committed back is not an increment, because the state has to live in Git, not in the workspace
- Automating a commit from CI turns the webhook into a feedback loop; filtering by committer email is the cleanest way out
- Detached HEAD is the normal state of a Jenkins checkout, and it changes how `git push` has to be written
- `mvn clean` and the Dockerfile wildcard are two halves of the same decision: a version in the filename only works if exactly one file matches
- Escaping passes through three layers here (Groovy string, shell, Maven) and each one eats a backslash
- Automated version commits change the team workflow too: developers have to pull Jenkins' commit before pushing their own

## Project structure

```
05-dynamic-version-increment/
├── README.md
├── Jenkinsfile    # full pipeline: increment, build, image, deploy placeholder, commit version
└── Dockerfile     # wildcard JAR copy, shell-form CMD
```

The deploy stage is a placeholder in this module; actual deployment comes in a later module.
