---
author: "Rahul Bajaj"
title: "Unraveling the Complexities: Navigating OpenShift’s Compliance Operator"
description: ""
date: 2024-06-22T10:29:34-04:00
categories: ["containers", "security"]
tags: ["compliance", "supplychainsecurity"]
cover:
  image: /images/compliance.jpg
  alt: "Openshift Compliance"
---

### Introduction to the Compliance Operator
The OpenShift Compliance Operator is all about keeping your OpenShift cluster secure and in line with governance policies. It does this by scanning both the OpenShift Platform 4 (ocp4) and Red Hat Core OS 4 (RHCOS4). Now, let's break down the types of compliance checks it can handle:

 - Platform (Cluster) Related Checks: These checks use the OpenShift API to make sure everything is shipshape.
 - Node Related Checks: These scans take a closer look at the filesystem of each node.

So, why should you use the Compliance Operator? Well, its a great companion to the Red Hat Advanced Cluster Security (RHACS) Operator, ensuring you're following the best security practices. If you are not familiar with RHACS, I recommend checking out my previous blog on the topic. Now, you might be wondering: if RHACS already covers compliance, why bother with the Compliance Operator? Let me explain the key differences:

 - Granular Control: Unlike some other tools, the Compliance Operator doesn't just assess running workloads on the cluster (like deployments).
 - Customization Options: It lets you tailor compliance checks to suit your organization's specific needs. We will dive deeper into this when we talk about remediation.
 - Automated Remediation: The Compliance Operator can automatically fix certain issues it finds through its automated remediation mechanism.

So, as you can see, the Compliance Operator provides a range of features that complement RHACS and give you more control and flexibility.

### Understanding the differnet parts of the Compliance Operator

If you're looking to wrap your head around the Compliance Operator, the official documentation is your go-to resource. It's got a handy diagram that breaks down each component and makes things easier to grasp.

![Architecture - Compliance Operator](/images/compliance-arch.png "Compliance Operator Architecture")

Now, let's take a quick peek into the main workflow involved in compliance management:

#### Step 1: Figure Out What You Need to Comply With
The Compliance Operator offers two **profile bundles**: one for the Platform (ocp4) and another for Node (rhcos4) checks. These bundles contain XML files like ssg-ocp4-ds.xml and ssg-rhcos4-ds.xml, which act as the source of truth. They define the compliance profiles and rules for scanning our environment. All the checks and their values are neatly packed in these XML files, which you can get from the openshift-compliance-content-rhel container in the Red Hat registry.

```bash
rbajaj@rhcos4$ oc get profilebundle
NAME     CONTENTIMAGE                                                                                                                               CONTENTFILE         STATUS
ocp4     registry.redhat.io/compliance/openshift-compliance-content-rhel8@sha256:7c1285f294f4630766bf158924ea7eff9b6549b2a9881e6dea6bad07887525a0   ssg-ocp4-ds.xml     VALID
rhcos4   registry.redhat.io/compliance/openshift-compliance-content-rhel8@sha256:7c1285f294f4630766bf158924ea7eff9b6549b2a9881e6dea6bad07887525a0   ssg-rhcos4-ds.xml   VALID
```

Next, the **compliance profiles** are security guidelines set by different frameworks like CIS (Center for Internet Security) and NIST (National Institute of Standards and Technology). Basically, compliance profiles usually come in pairs. For example, let's take ocp4-cis and ocp4-cis-node. When it comes to CIS standards, the ocp4-cis profile scans Kubernetes API resources, while the ocp4-cis-node profile focuses on scanning the nodes themselves.

```bash
rbajaj@rhcos4$ oc get profiles.compliance.openshift.io
NAME                       AGE   VERSION
ocp4-cis                   53d   1.4.0
ocp4-cis-1-4               53d   1.4.0
ocp4-cis-node              53d   1.4.0
ocp4-cis-node-1-4          53d   1.4.0
ocp4-e8                    53d
ocp4-high                  53d   Revision 4
ocp4-high-node             53d   Revision 4
ocp4-high-node-rev-4       53d   Revision 4
ocp4-high-rev-4            53d   Revision 4
ocp4-moderate              53d   Revision 4
ocp4-moderate-node         53d   Revision 4
ocp4-moderate-node-rev-4   53d   Revision 4
ocp4-moderate-rev-4        53d   Revision 4
ocp4-nerc-cip              53d
ocp4-nerc-cip-node         53d
ocp4-pci-dss               53d   3.2.1
ocp4-pci-dss-3-2           53d   3.2.1
ocp4-pci-dss-node          53d   3.2.1
ocp4-pci-dss-node-3-2      53d   3.2.1
ocp4-stig                  53d   V1R1
ocp4-stig-node             53d   V1R1
ocp4-stig-node-v1r1        53d   V1R1
ocp4-stig-v1r1             53d   V1R1
rhcos4-e8                  53d
rhcos4-high                53d   Revision 4
rhcos4-high-rev-4          53d   Revision 4
rhcos4-moderate            53d   Revision 4
rhcos4-moderate-rev-4      53d   Revision 4
rhcos4-nerc-cip            53d
rhcos4-stig                53d   V1R1
rhcos4-stig-v1r1           53d   V1R1
```

*Pro tip*: 

#### Step 2: Configure How Your Scans Should Work
This is where you get to specify how you want your compliance scans to be set up. You'll need to define scan settings that align with your compliance requirements.

#### Step 3: Connect Compliance Requirements with Scan Configurations
This step is all about linking the profiles that you need to comply with to the way you want to scan. By doing this, you ensure that the scans are performed according to the rules you've laid out.

#### Step 4: Keep an Eye on Compliance Scans
To stay on top of things, you'll want to use Compliance Suites to monitor your compliance scans. These suites help you effectively manage and organize the scanning process.

#### Step 5: Review the Results
Once the scans are done, you can check out the Compliance Check Results. These give you insights into the compliance status of your environment.


