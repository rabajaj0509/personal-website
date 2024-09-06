---
author: "Rahul Bajaj"
title: "Did You Forget these Crucial Day 2 Operations after OpenShift Deployment?"
description: "Discover the must-do Day 2 operations for your OpenShift deployment that ensure your clusters run smoothly and efficiently. Don’t miss these essential tips to keep your system optimized and secure!"
date: 2024-08-30T18:43:33-04:00
categories: ["openshift-administration"]
tags: ["automation", "configuration", "openshift-admin", "day2operations"]
cover:
  image: /images/openshift.jpg
  alt: "Openshift Container Platform"
---

Day 2 operations come into play after the initial installation of an OpenShift cluster. These operations involve a series of essential settings and configurations that prepare your cluster for developers' deployments. By performing these tasks, you ensure that the cluster remains healthy, secure, and efficient. Key activities include monitoring and logging, scaling resources, managing backups and recovery, enhancing security, optimizing resource allocation, and maintaining network configurations. These steps are crucial for maintaining the overall stability and performance of your OpenShift environment, making it ready for seamless development and deployment activities.

## Five Essential Day 2 Operations

In this blog, we are diving into five essential Day 2 operations that every OpenShift admin should implement after completing their cluster installation. These tips will help you optimize and maintain your OpenShift environment like a pro!

### Disabling the self provisioning in OpenShift 4

OpenShift gives admins the power to control who can access resources and how they do it. This is managed through the RBAC (Role-Based Access Control) mechanism. One handy feature is self-provisioners, which lets you decide which groups or users can create new projects. However, as a Day 2 operation, it is wise to restrict developers from provisioning new projects without oversight. Disabling self-provisioners offers several benefits: it helps you maintain control over resource creation, monitor usage, and prevent resource limits from being exceeded, which could negatively impact other namespaces.

When you set up a new OpenShift cluster, the default configuration for self-provisioners is a clusterrolebinding that grants the `system:authenticated:oauth` group the ability to create new projects. In other words, any user who can log in with their OAuth credentials can start their own projects within the cluster.

```bash
$ oc describe clusterrolebinding.rbac self-provisioners

Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
```

To prevent self-provisioning in your OpenShift cluster, you’ll need to make two key changes to the clusterrolebinding:

 * Remove the Group: Delete the `system:authenticated:oauth` group from the subjects field of the clusterrolebinding and set it to *null*. You can do this by:
    ```bash
    $ oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
    ```
 * Update the Annotation: Update the annotation `rbac.authorization.kubernetes.io/autoupdate: "false"` to prevent the clusterrolebinding from resetting to its default settings during automatic updates.
    ```bash
    $ oc edit clusterrolebinding.rbac self-provisioners
    ```

The final yaml for the clusterrolebinding should look like this:

```yaml
$ oc get clusterrolebinding self-provisioners -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "false"
  name: self-provisioners
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: self-provisioner
```
```bash
$  oc describe clusterrolebinding/self-provisioners
Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: false
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind  Name  Namespace
  ----  ----  ---------
```

You can also find the openshift documentation about this topic [here](https://docs.openshift.com/container-platform/4.16/applications/projects/configuring-project-creation.html#disabling-project-self-provisioning_configuring-project-creation).

### Optimizing node configurations with maxPods and podsPerCore

Efficiently managing resources is crucial for maintaining a stable and performant OpenShift cluster. Two key parameters in the KubeletConfig that play a significant role in this are `maxPods` and `podsPerCore`.

* *maxPods*:  This parameter sets a hard limit on the number of pods that can run on a node. For example, setting maxPods to 250 means no more than 250 pods can be scheduled on that node, regardless of its capacity.
* *podsPerCore*: This parameter limits the number of pods based on the number of CPU cores available on the node. For instance, if podsPerCore is set to 15 and the node has 12 CPU cores, the maximum number of pods would be 180 (12 cores * 15 pods per core).

To determine the appropriate values for these parameters, consider the following:

* *Node Specifications*: Check the number of CPU cores and total memory available on your nodes. For example, if a node has 12 CPUs and podsPerCore is set to 15, the node can handle up to 180 pods.
* *Workload Characteristics*: Evaluate the resource demands of your typical workloads. If your pods are resource-intensive, you might need to lower the podsPerCore value to prevent resource contention.
* *Performance Monitoring*: Continuously monitor the performance and stability of your nodes. High pod density can lead to issues such as CPU throttling, memory pressure, and network congestion.

Here is an example of how you might configure these parameters in a KubeletConfig:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: set-max-pods
spec:
  autoSizingReserved: true
  kubeletConfig:
    maxPods: 250
    podsPerCore: 15
    streamingConnectionIdleTimeout: 240m
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: enabled
```

In this example, the node can run up to 250 pods, but if it has 12 CPU cores, the limit would be 180 pods (12 cores * 15 pods per core). Side note: to apply the above configuration, all mcps should have the label `custom-kubelet: enabled`.

Another important factor to consider when determining the values for maxPods and podsPerCore is the memory requirements. To calculate the memory needed for your node, you should take into account the average memory usage per pod. For example, if each pod uses 200 MiB on average and the node can support 180 pods, the total memory requirement would be:

Total Memory = 200MiB * 180 = 36,000 MiB

Adding a buffer for system overhead and unexpected spikes is also recommended. You can read more information about this topic [here](https://docs.openshift.com/container-platform/4.16/nodes/nodes/nodes-nodes-managing-max-pods.html).

### Sync time for all OpenShift nodes with Chronyd

Time synchronization is crucial for maintaining the accuracy and reliability of various operations in your cluster, such as cron jobs and logging. Chrony, a Network Time Protocol (NTP) client and server, excels in synchronizing the system clock faster and more accurately.

Imagine you have three Machine Config Pools (MCPs): one for master nodes, one for worker nodes, and one for infrastructure nodes. Each of these MCPs needs the Chrony Machine Config (MC) applied to them. To achieve this, we will create three Machine Configs to configure Chrony on all nodes within the MCPs.

Before applying the Machine Configs, it’s crucial to pause the MCPs. Here is how you can do it:

```bash
$ oc edit mcp master
spec:
  ...
  paused: true
  ...
```

Repeat this for each MCP to ensure they are paused.

Next, we will create a Butane config file to configure Chrony. Here is an example:

```yaml
variant: openshift
version: 4.16.0
metadata:
  name: 99-worker-custom
  labels:
    machineconfiguration.openshift.io/role: worker
openshift:
  kernel_arguments:
    - loglevel=7
storage:
  files:
    - path: /etc/chrony.conf
      mode: 0420
      filesystem: root
      overwrite: true
      contents:
        inline: |
          server 0.rhel.server.ntp.org iburst maxpoll 10
          keyfile /etc/chrony.keys
          local stratum 10
          driftfile /var/lib/chrony/drift
          mazupdateskew 100.0
          makestep 1.0 3
          rtcsync
          logdir /var/log/chrony
```

Save this file as `99-worker-custom.bu`. The prefix `99` ensures that this configuration is applied after all other configurations, as OpenShift processes MachineConfigs in lexicographical order.

Download the Butane binary and make it executable:

```bash
$ curl https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane --output butane
$ chmod +x butane
```

Now, we need to apply the Machine Configs for the master, worker, and infra roles. First, check the roles labeled on nodes using:

```bash
$ oc get nodes
```

Then, run the following script to create and apply the Machine Configs:

```bash
for i in master worker infra
do
  echo $i
  cp -a 99-worker-custom.bu 99-mc-chrony-$i.bu
  sed -i "s/worker/$i/g" 99-mc-chrony-$i.bu
  butane 99-mc-chrony-$i.bu -o 99-mc-chrony-$i.yaml
  oc create -f 99-mc-chrony-$i.yaml
done
```

This script customizes the Butane config for each role and applies it to the respective nodes.

Read more about this day2 operation [here](https://docs.openshift.com/container-platform/4.16/machine_configuration/machine-configs-configure.html#installation-special-config-chrony_machine-configs-configure).

### Update the .apps Wild Card Certificate

By default, OpenShift uses a self-signed wildcard certificate for applications under the `.apps` sub-domain. To enhance security, you can replace this with a certificate from a public CA. Follow these steps to ensure a smooth transition:

*Steps:*

  * Create a ConfigMap for the Root CA Certificate:

```bash
$ oc create configmap custom-ca --from-file=ca-bundle.crt=</path/to/example-ca.crt> -n openshift-config
```

Replace `</path/to/example-ca.crt>` with the path to your root CA certificate file. This step ensures that the root CA certificate is recognized by the cluster.

  * Update the Cluster-Wide Proxy Configuration:

This command updates the cluster-wide proxy configuration to trust the new CA certificate.

```bash
$ oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'
```

  * Generate a Certificate Signing Request (CSR):

```bash
$ openssl req -new -newkey rsa:2048 -nodes -keyout mykey.key -out mycsr.csr
```

This command generates a new private key and CSR. You will need to submit the CSR your organizations CA to obtain the signed certificate.

  * Submit the CSR to your organizations CA to obtain the signed certificate.


  * **Pro Tip** Verify that the Key and Certificate Have the Same Modulus:

Ensure that the MD5 checksums match. If they do, the key and certificate are a pair.
```bash
$ openssl rsa -noout -modulus -in mykey.key | openssl md5

$ openssl x509 -noout -modulus -in mycert.crt | openssl md5
```

  * Create a Secret in OpenShift:

This command creates a secret that stores your certificate and key.

```bash
$ oc create secret tls mysecret --cert=mycert.crt --key=mykey.key -n openshift-ingress
```

  * Update the Ingress Controller:
```bash
$ oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p '{"spec":{"defaultCertificate":{"name":"mysecret"}}}'
```

This updates the ingress controller to use your new certificate.

By following these steps, you can replace the default ingress certificate in OpenShift with a certificate from your organizations CA, ensuring enhanced security for your applications. Remember to verify that your key and certificate match by checking their modulus, and update the cluster-wide proxy configuration to trust the new CA certificate.

You can read more about this operation [here](https://docs.openshift.com/container-platform/4.16/security/certificates/replacing-default-ingress-certificate.html)

### Etcd Encryption

etcd serves as the primary datastore for Kubernetes, storing all cluster state data, including configuration details, metadata, and the current state of the cluster. This makes etcd the single source of truth for Kubernetes clusters. Its strong consistency guarantees ensure that data is reliably stored and accessible, even in the event of machine failures.

By default, etcd data is not encrypted, which can pose a security risk if sensitive data is exposed. Enabling etcd encryption adds an additional layer of security by encrypting the data stored in etcd. This is particularly important for protecting sensitive resources such as secrets, config maps, and tokens.

How to Enable etcd Encryption?

* Modify the APIServer Object:
```bash
$ oc edit apiserver
```
Set the encryption field type to aescbc:
```yaml
spec:
  encryption:
    type: aescbc
```
This uses AES-CBC with PKCS#7 padding and a 32-byte key for encryption.

* Verify Encryption: Check the status of the OpenShift API server to ensure resources are encrypted:
```bash
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'
```
The output should show EncryptionCompleted if successful.