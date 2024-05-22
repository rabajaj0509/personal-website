---
author: "Rahul Bajaj"
title: "Enhancing Security: Using Renovate in Gitlab Pipelines for Automated Dependency Updates"
categories: ["CI/CD"]
description: ""
tags: ["CI/CD", "supplychainsecurity"]
date: 2023-12-21T11:46:02-04:00
cover:
  image: /images/renovate-dependency.png
  alt: "mend-renovate"
---

Open Source Software (OSS) projects have been distributed in packages for decades. Using packages allows developers to focus on new feature implementation. Major software distributions, such as Fedora, Debian, etc, typically consist of thousands of packages. These packages depend on each other to perform tasks efficiently by avoiding code duplication. The inter-dependence amongst the packages creates a software supply chain.

Examples of software security principles such as using software binaries that are signed by the software vendor, keeping binaries regularly updated, and constantly monitoring software behaviour are general to any software system. However, following these principles may not suffice for preventing a software supply chain from being infected with malicious code. Software supply chains are far from being simple. Recent supply chain attacks such as SolarWinds and Mimecast have raised serious security concerns. Despite the best effort of developers in addressing security concerns, any software project is prone to supply chain attacks. This is observed even in the context of major software organizations.

Securing your software supply chain goes beyond basic principals, it demands a proactive and adaptive appoach, especially when sophisticated threats are seen in the recent attacks. Renovate is becoming a crucial tool in this effort. Renovate, a powerful dependency management tool, steps in to enhance your defense strategy. Unlike traditional update approaches, Renovate actively monitors and patches vulnerabilities, making it a key player in safeguarding your software supply chain. In this blog, we’ll explore how integrating Renovate into GitLab CI pipelines can elevate your security measures, ensuring your project stays resilient against evolving cyber threats.

### IMPLEMENTATION APPROACH

There are multiple ways to configure the GitLab Runner, which is responsible for running CI/CD pipelines on GitLab. In the context of this implementation, each CI/CD pipeline runs in a separate container within the OpenShift instance. To facilitate this, a Dockerfile is used to generate the container image that will be executed through OpenShift. In this implementation, a container image is generated for Renovate, and this container image is executed as a cron job resource within the OpenShift cluster.

#### Dockerfile

Few key points to take note from the Dockerfile provided below are:

  1. We need to install the go and java programming languages in the Dockerfile to be able to update the go.sum and pom.xml files. Renovate works by inspecting configuration files in the application projects to identify the dependencies and their versions.
  2. Since we install the go programming language from source, we verify its checksum to avoid any discrepancies and thereby refrain from installing malicious code
  3. We run the Renovate command as a non-root user.

```Dockerfile
FROM node:latest

USER root

RUN mkdir -p /etc/rhsm/ca && \
    rm -f /etc/rhsm-host

RUN npm install --global renovate yarn && \
    curl -sfL --retry 10 -o /tmp/golang.tar.gz https://go.dev/dl/go1.21.3.linux-amd64.tar.gz && \
    echo "1241381b2843fae5a9707eec1f8fb2ef94d827990582c7c7c32f5bdfbfd420c8 /tmp/golang.tar.gz" | sha256sum -c && \
    mkdir -p /usr/local/go && \
    tar --strip-components=1 -zxf /tmp/golang.tar.gz --directory /usr/local/go && \
    mkdir -p /go/{path,cache,xdg} && \
    chmod -R 777 /go && \
    chown -R default /opt/app-root/src

RUN dnf install -y --disablerepo=* --enablerepo=ubi-9-baseos-rpms --enablerepo=ubi-9-appstream-rpms java-17-openjdk-devel

USER 1001

ENV GOPATH /go/path
ENV GOCACHE /go/cache
ENV PATH $PATH:/usr/local/go/bin

CMD ["renovate"]
```

#### OpenShift Cron Job

This cron job ensures that Renovate periodically checks and updates dependencies in specified GitLab repositories.

```yaml
spec:
  suspend: false
  schedule: "5 * * * *"
  startingDeadlineSeconds: 300
  concurrencyPolicy: "Forbid"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      activeDeadlineSeconds: 3600
      template:
        metadata:
          labels:
            app: <app-name>
            component: "cron"
        spec:
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: renovate-pvc
          containers:
            - name: "renovate"
              image: <registry>/renovate
              env:
                - name: RENOVATE_PLATFORM
                  value: "gitlab"
                - name: RENOVATE_ENDPOINT
                  value: "https://<gitlab-repository>/api/v4/"
                - name: RENOVATE_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: gitlab-secret
                      key: gitlab_token
                - name: GITHUB_COM_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: github-secret
                      key: github_token
                - name: RENOVATE_REPOSITORIES
                  value: <gitlab-repository>,<gitlab-repository>,<gitlab-repository>
                - name: RENOVATE_EXPOSE_ALL_ENV
                  value: "true"
                - name: GIT_CONFIG_COUNT
                  value: "1"
                - name: GIT_CONFIG_KEY_0
                  value: "safe.directory"
                - name: GIT_CONFIG_VALUE_0
                  value: "*"
                - name: LOG_LEVEL
                  value: "INFO"
          restartPolicy: "Never"
```

A secret needs to be created for storing the Gitlab token. Additionally, a volume with `ReadWriteAccess` within a configmap needs to be configured.