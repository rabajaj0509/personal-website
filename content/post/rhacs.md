---
author: "Rahul Bajaj"
title: "Unlocking RHACS: Vulnerability Management and Workload Hardening Policies"
categories: ["CI/CD", "OpenShift"]
tags: ["CI/CD", "supplychainsecurity"]
date: 2024-03-14T08:59:28-04:00
cover:
  image: /images/rhacs_security.jpg
  alt: "secure-k8s-cluster"
---

Recently, I have been working on Red Hat Advanced Cluster Security (RHACS) to identify security vulnerabilities and implementing cluster hardening rules within OpenShift clusters. Inspired by my experiences, I decided to share my insights through this blog post. Join me as we delve into the world of RHACS and explore its functionality.

### Introduction to RHACS

Red Hat Advanced Cluster Security (RHACS) is the downstream project for the upstream Stackrox project. In other words, enhancements to the source code are initially made in Stackrox, and then they undergo testing and packaging to become part of RHACS. There are three primary functionalities for which I am using RHACS:

  - Vulnerability Management
  - Workload Hardening Policies
  - Integration with the Compliance Operator

In this blog, we will focus on the first two functionalities, as the Compliance Operator is a comprehensive topic that deserves its own dedicated discussion

### Vulnerability Management

RHACS provides a robust mechanism to scan all the images that are deployed in your cluster.  It checks all the images you use and breaks them down into layers. Then it looks inside each layer to find out which software packages are there. After that, it compares these packages with a special database that collects information about Common Vulnerabilities and Exposures (CVE). This database gathers data from various sources, including Open-Source Vulnerability (OSV), Debian Security Tracker, Red Hat OVAL, National Vulnerability Database (NVD), and more.

#### Vulnerability Data Process

RHACS runs a process that involves several steps. First, it downloads the latest vulnerability data from the above mentioend sources. Next, it converts this data into a format that an in-house scanner can understand. Currently, we have two different in-house scanners: StackRox Scanner (Scanner v2) and Scanner V4. Once the data is processed, it is pushed to `definitions.stackrox.io`. This entire process occurs every 3 hours.

Within the cluster, there are three separate processes that play crucial roles:

- **Central**: Central queries definitions.stackrox.io every 5 minutes to fetch the most up-to-date vulnerability data. If there are no updates, no new data is sent.

- **Scanner**: The scanner queries Central approximately every 5 minutes for the latest vulnerability data. Again, if there are no updates, no new data is sent.

- **Central Reprocessing Pipeline**: Central periodically runs a “reprocessing” pipeline. During this process - all policies are re-evaluated, images are re-scanned, the scanners download each layer of an image, extract necessary information, and store it. While the “analysis” part may not be redone (since images are immutable), the scanners still check for any new vulnerabilities that may affect the image. By default, this reprocessing occurs every 4 hours. FYI, You can adjust the frequency by configuring the `ROX_REPROCESSING_INTERVAL` environment variable in Central. Keep in mind that this operation is CPU-intensive, especially for Central.

#### Vulnerability Report

RHACS provides a comprehensive vulnerability report. This report includes details about vulnerabilities within your OpenShift environment. For each vulnerability, it specifies the affected namespace, deployment, image, package, version, and remediation steps. Additionally, the report highlights the CVSS score, CVE number, and whether an update is available. RHACS helps you stay informed about vulnerabilties in your cluster. You can also `watch` images that might not be a part of your cluster with vulnerability management.

### Workload Hardening Policies

Policies are nothing but security best practices that should be applied consistently across all workloads within a cluster. While there may be exceptions or exclusions to specific policies, it remains crucial that resources adhere to these established security practices. There are three lifecycle phases in which policies are divided.

- **Build Policies**: These policies report violations during the creation of container images. For instance, they target best practices for building containers, such as avoiding the use of images with `Important` severity CVEs or executing all commands in the container as the root user during image build.
- **Deployment Policies**: Violations are reported at the time of deployment to the OpenShift cluster. These policies consist of best practices, such as avoiding the storage of secrets in environment variables or ensuring that service account tokens are not mounted until the service account is actually being used.
- **Runtime Policies**: These policies monitor violations while workloads are active on the OpenShift cluster. Violations reported at runtime include actions like executing `systemctl` inside a container, running `nmap` within a workload, or executing `iptables`.

![Policy Management](/images/rhacs-policies.png "Policy Management in RHACS")

There are in total 86 policies that are provided by default RHACS at the time of writing this blog. RHACS provides the ability to create custom policies as well.

Within the cluster, violations related to Deployment and Runtime policies are visible on the `Violations dashboard` in RHACS. In contrast, Build policies undergo checks during Continuous Integration pipelines. To perform CI checks, the `roxctl` command-line tool offers three specific commands for build-time analysis.


```bash
# Scan the specified image, and return scan results
$ roxctl image scan 

#Check images for build time policy violations, and report them
$ roxctl image check

# Check deployments for deploy time policy violations
$ roxctl deployment check 
```

Example of a Gitlab pipelines to check image and deployment policies:
```yaml
check-image: 
  stage: scan
  image: registry-redhat.io/advanced-cluster-security/rhacs-roxctl-rhel8:3.69.0
  script:
    - 'curl -k -H "Authorization: Bearer ${ROX_API_TOKEN}" "${ROX_CENTRAL_HOST}":443/api/cli/download/roxctl-linux -o roxctl && chmod +x ./roxctl'
    - ./roxctl --insecure-skip-tls-verify -e "${ROX_CENTRAL_HOST}:443" image check --image=<image-location>


check-deployment: 
  stage: scan
  image: registry-redhat.io/advanced-cluster-security/rhacs-roxctl-rhel8:3.69.0
  script:
    - set -x
    - 'curl -k -H "Authorization: Bearer ${ROX_API_TOKEN}" "${ROX_CENTRAL_HOST}":443/api/cli/download/roxctl-linux -o roxctl && chmod +x ./roxctl'
    - ./roxctl --insecure-skip-tls-verify -e "${ROX_CENTRAL_HOST}:443" deployment check --file=<deployment-location>
```

### Conclusion

- Vulnerability Management:
    - CVE data is fetched from sources every 3 hours.
    - Once fetched, information is stored at `definitions.stackrox.io`
    - Central queries `definitions.stackrox.io` every 5 minutes to fetch vulnerabilities.
    - Scanner's job is to get the information from Central and compare it with the
    workload in your cluster.
- Hardening Policies:
    - Policies are divided into three lifecycle phases: build, deploy, and runtime.
    - Build policies are part of the shift left initiative. By catching issues early in the development process, build policies enhance overall security.
