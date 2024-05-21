---
author: "Rahul Bajaj"
title: "Slimming Down Containers: The Art of Minimizing Image Bloat"
description: ""
date: 2023-10-28T12:00:59-05:00
categories: ["containers"]
tags: ["container-image", "supplychainsecurity"]
cover:
  image: "/images/container.jpg"
  alt: "Container Build Images"
---

### Need for slimming down containers 

OpenShift, an enterprise-ready Kubernetes platform, offers a multitude of benefits. One such advantage is the Source-to-Image (S2I) build strategy, that simplifies the process of converting source code into deployable container images. This strategy enables developers to build container images without the need to define a container file explicitly. OpenShift clones the application's source code into a builder image that utilizes builder scripts, ultimately generating a container image deployable within the cluster.

Employing the S2I strategy proves convenient, allowing software development teams to prioritize software development over the creation of container files. However, a notable drawback emerges as this approach tends to generate oversized images. These bulky images can rapidly consume storage capacity within private registries, resulting in prolonged push and pull times from the image registry. Beyond storage concerns, larger image sizes impact the build process, slowing down the overall build time. Moreover, heightened image size correlates with increased security risks, creating a larger attack surface for potential supply chain attacks due to the inclusion of numerous packages. Recognizing these challenges, the focus turns to exploring strategies aimed at slimming down containers.

### Four primary practices to overcome this issue
1. Using small or minimal base images.
2. Avoiding multiple RUN instructions.
3. Excluding files and directories from the build context.
4. Utilizing multi-stage container builds.

#### Using small or minimal base image

Using small or minimal base images for containers involves selecting lightweight starting points that contain only the essential packages required to run your application. Opting for such base images, like Alpine or distroless images, significantly reduces the container size. It is crucial to identify base images that tailor to specific application needs, ensuring they contain the necessary libraries and dependencies while excluding unnecessary components. Minimizing the attack surface by selecting base images with fewer pre-installed packages also enhances container security.

#### Avoiding multiple RUN instructions

Avoiding multiple RUN instructions in Dockerfiles involves consolidating commands into a single RUN instruction wherever possible. By chaining commands together, we can reduce the number of intermediate layers created during the image build process. For instance, rather than having separate RUN commands for installing dependencies, copying files, and performing cleanup, we can combine these steps within a single RUN instruction. Here's an example:

```Dockerfile
# Bad practice: Multiple RUN instructions
FROM baseimage:latest
RUN dnf update
RUN dnf install -y package1
RUN dnf install -y package2
RUN dnf clean

# Better practice: Consolidating commands in a single RUN instruction
FROM baseimage:latest
RUN dnf update \
    && dnf install -y package1 package2 \
    && dnf clean
```

#### Excluding files and directories from the build context

We can make use of the `.containerignore` or `.dockerignore` file in the build context directory to prevent unnecessary files and directories from being copied in to the image.

An example of `.dockerignore` file tailored for Ruby application:

```
# Ignore version control files
.git
.gitignore

# Ignore unnecessary editor/IDE files
.vscode/
.idea/

# Ignore log and temp files
log/
tmp/

# Ignore dependency manager specific files
Gemfile.lock

# Ignore development and test-specific files
spec/
test/
```

#### Utilizing multi-stage container builds

Multistage builds are implemented by defining multiple 'stages' within a single Dockerfile. This strategy enables the separation of the build environment from the final runtime environment, leading to the creation of smaller and more secure container images. With multistage builds, each 'stage' performs specific tasks, ensuring that the final container image only includes the necessary artifacts generated from the preceding build 'stage.' This optimization results in a streamlined image tailored precisely to run the application, eliminating unnecessary packages or artifacts.

An example of the multi-stage build can be seen below:
```dockerfile
# Stage 1: Build stage
FROM openjdk-17 AS builder
WORKDIR /app
COPY . .
USER root
RUN apt-get update && apt-get install -y some-dependency
RUN mvn -Dmaven.repo.local=/tmp/.m2/repository package

# Stage 2: Runtime stage
FROM openjdk17:alpine-jre
WORKDIR /app
COPY --from=builder --chown=someuser:somegroup /app/target/myapp.jar .
USER someuser
CMD ["java", "-jar", "myapp.jar"]
```

The example above illustrates a two-stage process within a container file. The first stage handles all build operations, while the second stage exclusively copies the artifact generated in the first stage. This artifact is used for deploying the image to the cluster. Two key points emerge from this example:

1. Using different base image in the build and runtime stages. During the build stage, more packages may be necessary. For instance, in a Java application, we employ OpenJDK, the Java Development Kit (JDK), crucial for development and building. Conversely, the runtime stage utilizes the Java Runtime Environment (JRE), a subset of the JDK containing essential libraries to run the Java application. Hence, choosing base images tailored to specific needs is essential.
2. In the build stage, we employ `USER root` to execute all build-related tasks. This approach often becomes necessary for tasks like installing dependencies and setting up the environment, requiring elevated privileges. However, once the necessary artifacts are acquired, transitioning to a non-root user `USER someuser` in subsequent stages enhances security. It minimizes potential vulnerabilities and restricts access privileges within the final runtime environment, improving overall security posture.

### Conclusion

Lean container images are crucial for streamlined deployments, reducing storage overhead and minimizing resource consumption.