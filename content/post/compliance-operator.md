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

If you're looking to wrap your head around the Compliance Operator, the [official documentation](https://github.com/openshift/compliance-operator/blob/master/doc/crds.md) is your go-to resource. It's got a handy diagram that breaks down each component and makes things easier to grasp.

![Architecture - Compliance Operator](/images/compliance-arch.png "Compliance Operator Architecture")

Now, let's take a quick peek into the main workflow involved in compliance management:

#### Step 1: Figure Out What You Need to Comply With
The Compliance Operator offers two *profile bundles*: one for the Platform (ocp4) and another for Node (rhcos4) checks. These bundles contain XML files like ssg-ocp4-ds.xml and ssg-rhcos4-ds.xml, which act as the source of truth. They define the compliance profiles and rules for scanning our environment. All the checks and their values are neatly packed in these XML files, which you can get from the openshift-compliance-content-rhel container in the Red Hat registry.

```bash
rbajaj@rhcos4$ oc get profilebundle
NAME     CONTENTIMAGE                                                                                                                               CONTENTFILE         STATUS
ocp4     registry.redhat.io/compliance/openshift-compliance-content-rhel8@sha256:7c1285f294f4630766bf158924ea7eff9b6549b2a9881e6dea6bad07887525a0   ssg-ocp4-ds.xml     VALID
rhcos4   registry.redhat.io/compliance/openshift-compliance-content-rhel8@sha256:7c1285f294f4630766bf158924ea7eff9b6549b2a9881e6dea6bad07887525a0   ssg-rhcos4-ds.xml   VALID
```

Next, the *compliance profiles* are security guidelines set by different frameworks like CIS (Center for Internet Security) and NIST (National Institute of Standards and Technology). Basically, compliance profiles usually come in pairs. For example, let's take ocp4-cis and ocp4-cis-node. When it comes to CIS standards, the ocp4-cis profile scans Kubernetes API resources, while the ocp4-cis-node profile focuses on scanning the nodes themselves.

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

*Pro tip - 1*: If you try to `$oc get profiles`, you might see the following output:

```bash
rbajaj@rhcos4$ oc get profiles
No resources found in openshift-compliance namespace.
```

CRDs can sometimes have long names that are complex to type, or they may have similar prefixes, which can lead to selecting the wrong CRD. For example, in this case, we want to use `profiles.compliance.openshift.io`, but OpenShift might pick up `profiles.tuned.openshift.io`. To avoid such situations, always run `$oc get crd | grep compliance`.

```bash
rbajaj@rhcos4$ oc get crd | grep profile
profiles.compliance.openshift.io
profiles.tuned.openshift.io

rbajaj@rhcos4$ oc get crd | grep compliance
compliancecheckresults.compliance.openshift.io
complianceremediations.compliance.openshift.io
compliancescans.compliance.openshift.io
compliancesuites.compliance.openshift.io
profilebundles.compliance.openshift.io
profiles.compliance.openshift.io
rules.compliance.openshift.io
scansettingbindings.compliance.openshift.io
scansettings.compliance.openshift.io
tailoredprofiles.compliance.openshift.io
variables.compliance.openshift.io

# Once you identify which crd you want to use:
rbajaj@rhcos4$ oc get profiles.compliance.openshift.io
```

*Pro tip - 2*: Before deciding which profile to use for scanning the compliance of your infrastructure, review the rules that a profile checks and the rationale behind each rule.

```bash
rbajaj@rhcos4$ oc get profiles.compliance.openshift.io ocp4-cis -o yaml
apiVersion: compliance.openshift.io/v1alpha1
description: This profile defines a baseline that aligns to the Center for Internet
  Security® Red Hat OpenShift Container Platform 4 Benchmark™, V1.5. This profile
  includes Center for Internet Security® Red Hat OpenShift Container Platform 4 CIS
  Benchmarks™ content. Note that this part of the profile is meant to run on the Platform
  that Red Hat OpenShift Container Platform 4 runs on top of. This profile is applicable
  to OpenShift versions 4.12 and greater.
... skipped metadata ...
rules:
- ocp4-accounts-restrict-service-account-tokens
- ocp4-accounts-unique-service-account
- ocp4-api-server-admission-control-plugin-alwaysadmit
- ocp4-api-server-admission-control-plugin-alwayspullimages
- ocp4-api-server-admission-control-plugin-namespacelifecycle
- ocp4-api-server-admission-control-plugin-noderestriction
- ocp4-api-server-admission-control-plugin-scc
- ocp4-api-server-admission-control-plugin-service-account
- ocp4-api-server-anonymous-auth
- ocp4-api-server-api-priority-gate-enabled
- ocp4-api-server-audit-log-maxbackup
- ocp4-api-server-audit-log-maxsize
- ocp4-api-server-audit-log-path
- ocp4-api-server-auth-mode-no-aa
- ocp4-api-server-auth-mode-rbac
- ocp4-api-server-basic-auth
- ocp4-api-server-bind-address
- ocp4-api-server-client-ca
- ocp4-api-server-encryption-provider-cipher
- ocp4-api-server-etcd-ca
- ocp4-api-server-etcd-cert
... and so on
```

You can check the contents of each rule with the following command:

```bash
# Replace ocp4-accounts-restrict-service-account-tokens with the rule you want to check the contents of.
rbajaj@rhcos4$ oc get rule ocp4-accounts-restrict-service-account-tokens -o yaml
```

This command will illustrate the rationale behind why this check must be performed, along with information about how you can verify whether this rule is applied in the cluster. It provides you with the commands to check if compliance has been met in your infrastructure. If not, it will also recommend the changes in settings that might be required to comply with the specified rule.

#### Step 2: Configure How Your Scans Should Work
This is where you get to specify how you want your compliance scans to be set up. You will need to define **scan settings** that align with your compliance requirements. The scan settings define technical parameters for the scan. Let us talk about a few important sections of the scan settings:

```bash
rbajaj@rhcos4$ oc get ss <scan-setting-name> -o yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSetting
... skipped metadata ...
rawResultStorage:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  pvAccessModes:
  - ReadWriteMany
  rotation: 3
  size: 1Gi
  storageClassName: <dynamically-provisioned-fs>
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists
roles:
- master
- worker
- infra
scanTolerations:
- operator: Exists
schedule: 0 1 * * *
strictNodeScan: true
```

The scan settings are responsible for specifying which nodes to perform the scan on. They determine when the scan will run using the `schedule` field as a cron job. The scan settings also define where the results of the scan must be stored, specifying the node with the `rawResultStorage` field.

#### Step 3: Connect Compliance Requirements with Scan Configurations
This step is all about linking the profiles that you need to comply with to the way (scan settings) you want to scan. By doing this, you ensure that the scans are performed according to the rules you have laid out. You can achieve this by define the *scan setting bindings*.

```bash
rbajaj@rhcos4$ oc get ssb <scan-setting-binding-name> -o yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
... skipped metadata ...
profiles:
- apiGroup: compliance.openshift.io/v1alpha1
  kind: Profile
  name: ocp4-cis-node
settingsRef:
  apiGroup: compliance.openshift.io/v1alpha1
  kind: ScanSetting
  name: <scan-setting-name>
```

#### Step 4: Keep an Eye on Compliance Scans
To stay on top of things, you will want to use *compliance suites* to monitor your compliance scans. These suites help you effectively manage and organize the scanning process.

```bash
rbajaj@rhcos4$ oc get compliancesuites
NAME                         PHASE   RESULT
<scan-setting-binding>-cis   DONE    ERROR
```

The name of the resource compliance suites is the name of your SSB suffixed with `-cis`. The result of the compliance suite indicates the outcome of the scan. In our case, the cluster is not compliant with the rules defined by the profiles and that there were errors during the execution of the scan. Other possible result values include:

 - SUCCESS: This indicates that the cluster is compliant with the rules defined by the profiles and the scans performed on the platform and the nodes.

To know more in detail why the compliancesuites result in error, we can check the *compliancescans* resource. This resource shows in detail about which roles exactly throw the error.

```bash
rbajaj@rhcos4$ oc get compliancescans
NAME                   PHASE   RESULT
ocp4-cis-node-infra    DONE    ERROR
ocp4-cis-node-master   DONE    NON-COMPLIANT
ocp4-cis-node-worker   DONE    INCONSISTENT 
```

Wow! These are some serious compliance failures in my cluster. Don’t worry, we can fix all of these, but we will address them in the next blog. For now, let’s just understand what these results really mean:

- ERROR: This means that the scan was not able to complete due to some reason, such as the pod responsible for performing the scan not being scheduled.
- NON-COMPLIANT: This simply means that there are a few checks which are failing, and we can resolve them using the *Compliance Check Results* (Step 5).
- INCONSISTENT: This means that two nodes with the same role yield different check results. In this case, we need to either adjust the list of roles or check nodes with the same role that have different configurations. This should not happen since we execute machine config pools on these nodes, and nodes with the same roles are expected to have similar configurations.

#### Step 5: Review the Results
Once the scans are done, you can check out the Compliance Check Results (CCR). These give you insights into the compliance status of your environment.

```bash
rbajaj@rhcos4$ oc get ccr
NAME                                                                          STATUS           SEVERITY
ocp4-cis-node-infra-file-groupowner-cni-conf                                  PASS             medium
ocp4-cis-node-infra-file-groupowner-kubelet-conf                              PASS             medium
ocp4-cis-node-infra-file-groupowner-multus-conf                               PASS             medium
ocp4-cis-node-infra-file-groupowner-ovn-cni-server-sock                       PASS             medium
ocp4-cis-node-infra-file-groupowner-ovn-db-files                              PASS             medium
ocp4-cis-node-infra-file-groupowner-ovs-conf-db                               PASS             medium
ocp4-cis-node-infra-file-groupowner-ovs-conf-db-lock                          PASS             medium
ocp4-cis-node-infra-file-groupowner-ovs-pid                                   PASS             medium
ocp4-cis-node-infra-file-groupowner-ovs-sys-id-conf                           PASS             medium
ocp4-cis-node-infra-file-groupowner-ovs-vswitchd-pid                          PASS             medium
ocp4-cis-node-infra-file-groupowner-ovsdb-server-pid                          PASS             medium
ocp4-cis-node-infra-file-groupowner-worker-ca                                 PASS             medium
ocp4-cis-node-infra-file-groupowner-worker-kubeconfig                         PASS             medium
ocp4-cis-node-infra-file-groupowner-worker-service                            PASS             medium
ocp4-cis-node-infra-file-owner-cni-conf                                       PASS             medium
ocp4-cis-node-infra-file-owner-kubelet                                        PASS             medium
ocp4-cis-node-infra-file-owner-kubelet-conf                                   PASS             medium
ocp4-cis-node-infra-file-owner-multus-conf                                    PASS             medium
ocp4-cis-node-infra-file-owner-ovn-cni-server-sock                            PASS             medium
ocp4-cis-node-infra-file-owner-ovn-db-files                                   PASS             medium
ocp4-cis-node-infra-file-owner-ovs-conf-db                                    PASS             medium
ocp4-cis-node-infra-file-owner-ovs-conf-db-lock                               PASS             medium
ocp4-cis-node-infra-file-owner-ovs-pid                                        PASS             medium
ocp4-cis-node-infra-file-owner-ovs-sys-id-conf                                PASS             medium
ocp4-cis-node-infra-file-owner-ovs-vswitchd-pid                               PASS             medium
ocp4-cis-node-infra-file-owner-ovsdb-server-pid                               PASS             medium
ocp4-cis-node-infra-file-owner-worker-ca                                      PASS             medium
ocp4-cis-node-infra-file-owner-worker-kubeconfig                              PASS             medium
ocp4-cis-node-infra-file-owner-worker-service                                 PASS             medium
ocp4-cis-node-infra-file-permissions-cni-conf                                 FAIL             medium
ocp4-cis-node-infra-file-permissions-kubelet-conf                             PASS             medium
ocp4-cis-node-infra-file-permissions-multus-conf                              PASS             medium
... many more
```

The CCR is a long list of all the checks and their individual scan results. The results can either be PASS, FAIL, or MANUAL. MANUAL means the check is not automated; manual steps are provided within the rule, and one has to manually check your cluster’s compliance with the rule. We will talk about remediating failed checks in future blogs.

### Quick Overview of the Scanning Process

In the beginning of this blog, we made it clear that there are two types of compliance checks that the operator can perform: platform and node. Let’s conclude this blog with a brief understanding of how compliance checks are performed in both cases.

For node-related checks, when the scan takes place, multiple pods are run. Each pod is scheduled on one cluster node. The pods use the host path volume to mount the root file system of the node into the pod, and then the pod runs OpenSCAP to scan this root filesystem.

For platform-related checks, a separate API check pod is run. This pod scans the OpenShift API resources. It runs an API resource collector, saving these resources on the local filesystem in the pod. Then, the pod runs OpenSCAP, which scans the API resources.