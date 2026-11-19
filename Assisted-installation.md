# OpenShift Assisted Installation with RHACM: Host Inventory and Disconnected Environments

> **Hands-on technical guide | Red Hat Advanced Cluster Management (RHACM) | Host Inventory | Connected and disconnected installations**

## Introduction

This article documents a hands-on workflow for preparing Host Inventory and installing OpenShift clusters through the Assisted Installer workflow available with Red Hat Advanced Cluster Management (RHACM).

The examples and observations are based on a lab environment. They retain the original configuration values and practical troubleshooting experience.

> [!IMPORTANT]
> Host Inventory, AgentServiceConfig, ClusterImageSet, and mirror-registry behavior can vary by RHACM, multicluster engine (MCE), OpenShift, and operator release. Verify the APIs and supported configuration for your installed versions before applying these examples.

## Contents

- [Overview and installation approaches](#overview-and-installation-approaches)
- [Prerequisites](#prerequisites)
- [Prepare Host Inventory](#1-prepare-host-inventory)
- [Configure a disconnected environment](#2-configure-a-disconnected-environment)
- [Configure the mirror registry](#3-configure-the-mirror-registry)
- [Create a ClusterImageSet](#4-create-a-clusterimageset)
- [Create Host Inventory credentials](#5-create-host-inventory-credentials)
- [Configure registry trust on the hub](#6-configure-registry-trust-on-the-hub)
- [Troubleshooting](#7-troubleshooting)
- [Technical verification notes](#technical-verification-notes)
- [Conclusion](#conclusion)

## Overview and installation approaches

### Recommended Host Inventory approach

My preferred approach is to create the hosts first and make them available in Host Inventory. Then, select those existing hosts when creating a cluster.

One benefit of this approach is that deleting a cluster does not necessarily require deleting the host records or removing the physical nodes from inventory.

> **Verify this behavior:** Cluster deletion, host deprovisioning, and host release are separate lifecycle operations. Confirm the exact host reuse behavior for your RHACM/MCE version before relying on it.

### Hosted Cluster

The original lab notes distinguish between preparing Host Inventory for connected and disconnected environments:

- Create Host Inventory for a connected environment.
- Create Host Inventory for a disconnected environment.

The exact workflow and use of the term Hosted Cluster should be checked against the RHACM/MCE release installed in your environment.

### Installation scenarios

The Host Inventory workflow can support different cluster topologies and network designs, subject to release support and infrastructure requirements.

| Scenario | Description and planning considerations |
|---|---|
| SNO | Single Node OpenShift installation. |
| Multi-node | Multi-node cluster installation. |
| Cluster-managed networking (IPI) | The lab notes identify two additional IP addresses for the API and Ingress endpoints. Confirm the exact network requirements for your platform and release. |
| User-managed networking (UPI) | Prepare the external load balancer, including HAProxy where applicable, and its configuration before cluster installation. |

## Prerequisites

Before configuring Host Inventory, prepare:

- An RHACM hub cluster with the required multicluster engine and Assisted Installer components.
- Administrator access to the hub cluster and the OpenShift CLI (oc).
- A default StorageClass, as required by the Host Inventory configuration in this lab.
- Hosts with the network connectivity and installation prerequisites required by the selected topology.
- For disconnected installations, an HTTP server that can host the RHCOS live ISO and be reached by the required components.
- A reachable mirror registry containing the required release and operator images.
- The registry CA certificate and suitable pull-secret credentials.
- DNS and load-balancer configuration appropriate to the selected network design.

**Expected outcome:** You can access the RHACM console, configure Host Inventory, and reach the storage, ISO, and registry services needed by the selected installation workflow.

## 1. Prepare Host Inventory

The following procedure is based on the lab workflow used for both connected and disconnected environments.

### 1.1 Configure Host Inventory settings

In the RHACM web console:

1. Navigate to **Infrastructure → Host inventory**.
2. Select **Configure host inventory settings**.
3. Configure the storage values used in this lab:

| Setting | Lab value |
|---|---:|
| Database storage | 30 |
| System storage | 150 |
| Image storage | 50 |

> **Sizing note:** These are the values used in this lab, not universal sizing recommendations. Confirm the expected units, minimum values, and sizing guidance for your installed release and inventory scale.

In this environment, the configuration took approximately 5–10 minutes to complete. The actual time may vary depending on cluster performance and storage provisioning.

### 1.2 Validate the Host Inventory components

Check the pods:

~~~bash
oc get po
~~~

The original lab output included these running pods. The names and ages below are examples from that environment:

~~~text
agentinstalladmission-858f989cbf-mc769   1/1   Running   0   19h
agentinstalladmission-858f989cbf-r8ln9   1/1   Running   0   19h
assisted-image-service-0                 1/1   Running   0   16m
assisted-service-5b46d88c7f-6spw5        2/2   Running   0   16m
~~~

Check the persistent volume claims and persistent volumes:

~~~bash
oc get pvc
oc get pv
~~~

**Expected outcome:** The Host Inventory-related pods are running, and the required PVCs are bound. If the pod names or namespaces differ, inspect the resources deployed by your installed RHACM/MCE release.

## 2. Configure a disconnected environment

In a disconnected environment, follow the same Host Inventory configuration process. The Assisted Installer components may start, but the assisted-image-service pod may not become ready if it cannot retrieve the RHCOS ISO from an external source.

### 2.1 Check the image-service logs

Check the pod status and inspect the image-service logs:

~~~bash
oc get po
oc logs assisted-image-service-0
~~~

In the lab, the image service attempted to download the ISO from an external Red Hat mirror. Because the environment did not have external connectivity, the download could not complete.

### 2.2 Host the RHCOS ISO locally

Download the RHCOS live ISO that matches the target OpenShift release and make it available through an HTTP server reachable from the Assisted Installer components.

Then inspect the AgentServiceConfig resource:

~~~bash
oc get agentserviceconfigs
~~~

The original notes contain this edit command:

~~~bash
oc edit agentserviceconfig/agant
~~~

> **Resource-name verification required:** The source notes spell the resource name as agant. This may be a typo. Do not assume the resource name; use the exact name returned by the command that lists AgentServiceConfig resources.

The following is the osImages fragment from the original lab configuration:

~~~yaml
osImages:
  - cpuArchitecture: 'x86_64'
    openshiftVersion: '4.17.16' # OpenShift version
    rootFSUrl: "" # Not required in this lab example; verify for your release
    url: 'http://10.4.60.59:8080/iso/rhcos-4.17.2-x86_64-live.x86_64.iso'
    version: '417.94.202502051822-0'
~~~

Field descriptions:

- **cpuArchitecture:** CPU architecture for the operating system image.
- **openshiftVersion:** OpenShift release associated with this image entry.
- **rootFSUrl:** Root filesystem image URL. It is empty in this lab example; confirm whether your release and workflow require it.
- **url:** URL of the RHCOS live ISO hosted on the local HTTP server.
- **version:** RHCOS image version identifier. Confirm this value for the exact image and OpenShift release; do not guess it.

The original notes refer to checking release metadata with the following command:

~~~bash
oc adm release info quay.io/openshift-release-dev/ocp-release:<ocp_version>-x86_64
~~~

Replace the placeholder with the target OpenShift version. Verify the exact method for identifying the RHCOS machine OS image and its version against the documentation for that OpenShift release.

**Expected outcome:** The image service can access the locally hosted ISO. Confirm this by checking the pod status and logs before proceeding.

## 3. Configure the mirror registry

For a disconnected installation, the required release and operator images must be available from a registry reachable by the hub and the hosts being installed.

The lab workflow uses a Quay mirror registry. Before updating the mirror configuration:

1. Prepare the destination repositories on the mirror registry.
2. Mirror the required content from the source registry.
3. Confirm that the repositories and image digests exist at the destination.
4. Check the registry synchronization status.
5. Ensure that the hub cluster trusts the registry's CA certificate.

### 3.1 Create the registry ConfigMap

The following ConfigMap is based on the lab configuration. It contains the CA bundle and registry mirror configuration used by the Assisted Installer service.

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quay-registry-details
  namespace: multicluster-engine
  labels:
    app: assisted-service
data:
  ca-bundle.crt: |
    <CA certificate PEM content>
  registries.conf: |-
    unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]
    [[registry]]
       prefix = ""
       location = "registry.redhat.io/multicluster-engine"
       mirror-by-digest-only = true
       [[registry.mirror]]
       location = "central-quay-registry.apps.ocp-hub.intern.example.com/multicluster-engine"
    [[registry]]
       prefix = ""
       location = "quay.io/openshift-release-dev/ocp-v4.0-art-dev"
       mirror-by-digest-only = true
       [[registry.mirror]]
       location = "central-quay-registry.apps.ocp-hub.intern.example.com/openshift/release"
    [[registry]]
       prefix = ""
       location = "quay.io/openshift-release-dev/ocp-release"
       mirror-by-digest-only = true
       [[registry.mirror]]
       location = "central-quay-registry.apps.ocp-hub.intern.example.com/openshift/release-images"
~~~

Replace the example registry hostnames with the paths used in your environment. The CA bundle must contain the PEM-encoded CA certificate data required to validate the registry endpoint.

> **Certificate note:** The openssl s_client command is useful for inspecting the certificate chain. Its terminal output is diagnostic material, not a ready-to-use CA bundle. Extract and verify the correct CA certificate before placing PEM content in ca-bundle.crt.

Inspect the certificate chain with:

~~~bash
openssl s_client -connect <quay-endpoint>:<port> -showcerts
~~~

### 3.2 Apply the ConfigMap

Save the manifest as quay-registry-details.yaml and apply it:

~~~bash
oc apply -f quay-registry-details.yaml
oc get configmap quay-registry-details -n multicluster-engine
~~~

The first command creates or updates the ConfigMap. The second confirms that it exists; it does not validate the registry mirror configuration itself.

### 3.3 Update AgentServiceConfig

The lab configuration references the registry ConfigMap through mirrorRegistryRef.

List the AgentServiceConfig resources:

~~~bash
oc get agentserviceconfigs
~~~

Edit the resource using its actual name:

~~~bash
oc edit agentserviceconfig/<agentserviceconfig-name>
~~~

The following is the relevant configuration fragment from the lab notes:

~~~yaml
mirrorRegistryRef:
  name: quay-registry-details
osImages:
  - cpuArchitecture: x86_64
    openshiftVersion: 4.17.16
    rootFSUrl: ""
    url: http://service.example.com:8080/iso/rhcos-4.17.2-x86_64-live.x86_64.iso
    version: 417.94.202502051822-0
~~~

This is a configuration fragment, not a complete AgentServiceConfig manifest. Add it at the correct location in the existing resource and verify that the field names and structure match the CRD installed in your cluster.

### 3.4 Mirror the required images

The original lab notes describe two approaches:

1. **ImageSetConfiguration workflow:** Create an ImageSetConfiguration and follow the supported OpenShift image-mirroring procedure for the required release and operator content.
2. **Manual repository workflow:** Create the required organizations and repositories on the destination Quay registry, mirror the images, and verify that the destination paths and digests match.

The manual approach requires careful repository and digest verification. A repository existing in Quay does not necessarily mean that every required image digest has been mirrored successfully.

> **Practical recommendation:** Use the documented image-mirroring workflow for repeatable disconnected installations. If you mirror repositories manually, maintain a record of repository names, image digests, and synchronization status.

## 4. Create a ClusterImageSet

The lab workflow uses a ClusterImageSet to make the required OpenShift release available for cluster creation through the RHACM console.

In the RHACM console, navigate to **Infrastructure → Clusters → Create cluster** and select the Host Inventory workflow. The required release image must be available through a ClusterImageSet for this workflow.

Check the existing ClusterImageSets:

~~~bash
oc get clusterimagesets
~~~

The following is the example manifest from the lab:

~~~yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  labels:
    channel: fast
    visible: 'true'
  name: img-4-17-16-x86-64
spec:
  releaseImage: central-quay-registry.apps.ocp-hub.intern.tpit.test/openshift/release-images:4.17.16-x86_64
~~~

Save the manifest as clusterimageset.yaml and apply it:

~~~bash
oc apply -f clusterimageset.yaml
oc get clusterimagesets
~~~

**Important:** The releaseImage value must point to the correct release image in your mirror registry. Confirm the repository path and tag against the content that was actually mirrored.

## 5. Create Host Inventory credentials

In the RHACM console:

1. Navigate to **Credentials**.
2. Select **Add Credential**.
3. Choose the **Host Inventory** credential type.
4. Enter a credential name.
5. Select or create a namespace for the credential.
6. Enter the base DNS domain used by the cluster.

The original lab used these example values:

| Field | Example value |
|---|---|
| Credential type | Host Inventory |
| Credential name | Any descriptive name |
| Namespace | rhacm-credentials |
| Base DNS name | intern.j2ctechnologies.com |

### 5.1 Add the pull secret and SSH key

- **Pull secret:** For a connected cluster, use the appropriate Red Hat pull secret. For a disconnected cluster, use a pull secret that includes authentication for the.
- **SSH key:** Add the public SSH key that you want to use for node access.

> **Lab troubleshooting note:** If cluster creation reports image-pull or authentication errors, check whether the pull secret includes credentials for the required registries. The original notes also recommend reviewing the cluster pull secret and global pull-secret configuration. Do not replace global credentials without understanding the impact on other cluster workloads.

## 6. Configure registry trust on the hub

When the hub cluster pulls release images from a private registry, it must trust the registry's CA certificate. The lab workflow uses the OpenShift image configuration resource and a ConfigMap in the openshift-config namespace.

### 6.1 Inspect the registry certificate

Use the following command to inspect the TLS certificate chain presented by the registry:

~~~bash
openssl s_client -connect <quay-endpoint>:<port> -showcerts
~~~

Use the appropriate CA certificate for registry trust. Do not assume that the leaf/server certificate is the CA certificate.

### 6.2 Create the registry CA ConfigMap

The original lab command creates a ConfigMap with registry-specific certificate filenames:

~~~bash
oc create configmap registry-cas -n openshift-config \
  --from-file=myregistry.corp.com..5000=quay-ca.crt \
  --from-file=otherregistry.com=central-quay-ca.crt
~~~

The filenames in the ConfigMap data must match the registry host and port format expected by the OpenShift image configuration API. Replace the example filenames and certificate paths with the actual registry hostnames, ports, and CA files in your environment.

### 6.3 Reference the CA ConfigMap

Patch the cluster image configuration to reference the ConfigMap:

~~~bash
oc patch image.config.openshift.io/cluster \
  --patch '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}' \
  --type=merge
~~~

This patch sets spec.additionalTrustedCA.name to registry-cas. It does not add registry allow-list entries.

### 6.4 Review registry source settings

Inspect the cluster image configuration:

~~~bash
oc get image.config.openshift.io/cluster -o yaml
~~~

The following is the relevant configuration fragment recorded in the lab:

~~~yaml
spec:
  additionalTrustedCA:
    name: private-quay-registry
  registrySources:
    allowedRegistries:
      - central-quay-registry.apps.ocp-hub.intern.example.com
      - quay.io
      - registry.redhat.io
      - service.example.com:8443
status:
  internalRegistryHostname: image-registry.openshift-image-registry.svc:5000
~~~

> **Configuration consistency check:** The patch command references registry-cas, while the original configuration fragment references private-quay-registry. Use the actual ConfigMap name you created and ensure the live Image resource references that same name. Do not apply both names blindly.

The original notes also included this discovery command:

~~~bash
oc get crd | grep image
~~~

This lists matching CRD names. To inspect the cluster image configuration itself, use the direct command shown above.

## 7. Troubleshooting

The following issues and observations are retained from the original lab notes. Registry names, image digests, pod names, and error output are environment-specific examples.

### Issue 1 — Release image cannot be read because the registry certificate is not trusted

**Observed error**

The ClusterImageSet release image could not be read. The error included:

~~~text
The Spec could not be synced due to backend error: failed to get release image.
tls: failed to verify certificate: x509: certificate signed by unknown authority
~~~

**Likely cause**

The hub cluster did not trust the CA that signed the private mirror registry's TLS certificate.

**What to check**

1. Confirm that the registry endpoint and port are correct.
2. Confirm that the CA certificate is valid and PEM encoded.
3. Confirm that the ConfigMap in openshift-config contains the expected registry CA data.
4. Confirm that image.config.openshift.io/cluster references the correct ConfigMap name.
5. Check that the registry hostname used in the release image matches the hostname covered by the certificate.

Useful commands:

~~~bash
oc get configmap registry-cas -n openshift-config
oc get image.config.openshift.io/cluster -o yaml
~~~

### Issue 2 — The Assisted Installer agent service cannot pull an image

**Observed symptom**

The node's agent.service repeatedly restarts because it cannot pull an image from the registry. The original lab notes captured the issue with:

~~~bash
sudo journalctl -u agent.service
~~~

**Troubleshooting steps**

1. Check whether the mirror registry is healthy.
2. Confirm that the required repository and image digest exist in the mirror registry.
3. Test registry connectivity by pulling the image manually from a suitable system with the same network access.
4. Check the health of the registry's backing storage.

Example image-pull test. Replace the example image and digest with the exact image reported in the error:

~~~bash
podman pull quay.example.com/multicluster-engine/assisted-installer-agent-rhel8@sha256:<image-digest>
~~~

Check the registry-side and node-side logs as appropriate. In the original lab, the underlying issue was an unhealthy Quay service caused by S3 connectivity problems.

**Practical lesson:** A registry endpoint can be reachable while the registry is still unable to serve image content. Check both registry application health and its backing object-storage service.

### Issue 3 — Nodes do not become available during installation

**Possible cause**

The mirror registry may not contain all required images, or the registries.conf mirror paths may not match the repository layout on the destination registry.

The following example is the mirror mapping used in the lab:

~~~toml
[[registry]]
   prefix = ""
   location = "registry.redhat.io/multicluster-engine"
   mirror-by-digest-only = true
   [[registry.mirror]]
   location = "central-quay-registry.apps.ocp-hub.intern.example.com/multicluster-engine"
~~~

**Troubleshooting commands**

Run these commands on the affected host, where applicable:

~~~bash
ps -ef | grep agent
sudo journalctl -u agent.service
sudo journalctl TAG=agent | less
sudo su
podman ps -a | grep assisted-installer
podman logs <container-id>
sudo journalctl -u bootkube
~~~

Use the output to identify the failing service or container and determine whether the problem is related to image pulls, registry authentication, mirror mapping, or another installation dependency.

### Issue 4 — Cluster installation cannot fetch required release images

**Observed error**

The installation reported that it could not fetch required images from the OpenShift release payload. The original message included errors for images under:

~~~text
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:<image-digest>
~~~

The original message suggested a possible network issue. In a disconnected environment, an incomplete mirror or an incorrect mirror path can cause the same type of failure.

**What to check**

1. Confirm that the required release payload images have been mirrored.
2. Confirm that the source repository paths match the location entries.
3. Confirm that the destination paths match the registry.mirror entries.
4. Verify that the required image digests exist in the destination registry.
5. Check the mirror registry synchronization status.

The following is the mirror mapping from the lab notes:

~~~toml
[[registry]]
   prefix = ""
   location = "quay.io/openshift-release-dev/ocp-v4.0-art-dev"
   mirror-by-digest-only = true
   [[registry.mirror]]
   location = "central-quay-registry.apps.ocp-hub.intern.example.com/openshift/release"

[[registry]]
   prefix = ""
   location = "quay.io/openshift-release-dev/ocp-release"
   mirror-by-digest-only = true
   [[registry.mirror]]
   location = "central-quay-registry.apps.ocp-hub.intern.example.com/openshift/release-images"
~~~

These mappings must correspond to the repository structure and content actually present in your mirror registry. A syntactically valid mirror configuration cannot compensate for missing images.

## Technical verification notes

The following details are intentionally highlighted for verification rather than silently changed:

1. **RHACM/MCE release compatibility:** Confirm the Host Inventory workflow, AgentServiceConfig fields, and Assisted Installer behavior against the installed release.
2. **AgentServiceConfig name:** The source notes contain the spelling agant. Use the exact resource name returned by oc get agentserviceconfigs.
3. **RHCOS image metadata:** Confirm that the ISO URL, openshiftVersion, and version identify a compatible RHCOS image for the target OpenShift release. Do not copy the sample.
4. **Root filesystem URL:** The example leaves rootFSUrl empty. Confirm whether this is supported for your release and installation workflow.
5. **Registry ConfigMap schema:** Confirm that ca-bundle.crt and registries.conf are the expected keys for the installed Assisted Service version.
6. **Registry CA content:** The ConfigMap must contain valid PEM certificate data, not the literal output or command text from openssl s_client.
7. **Mirror mapping:** Verify every source repository, destination path, and required digest. The example registry paths are specific to the lab.
8. **ClusterImageSet release image:** Confirm that the release image path and tag point to a valid mirrored OpenShift release payload.
9. **Image trust ConfigMap name:** The source notes use both registry-cas and private-quay-registry. Ensure the live Image resource references the ConfigMap that actually exists.
10. **Pull-secret behavior:** Confirm the correct pull-secret requirements for your connected or disconnected topology and RHACM/MCE release.
11. **Storage values:** The database, system, and image storage values are lab settings. Validate units and sizing for your deployment.
12. **Example output and digests:** Pod names, timestamps, registry hostnames, and image digests in this document are examples from the lab and should not be treated as.

## Conclusion

This guide captures a practical Host Inventory and Assisted Installer workflow for connected and disconnected OpenShift environments. It covers Host Inventory preparation, local RHCOS ISO configuration, mirror-registry setup, ClusterImageSet creation, registry trust, and troubleshooting installation failures.

The most important lesson from the disconnected lab is to validate the complete image-delivery path before starting a cluster installation: the required images must be mirrored to the expected repositories, the mirror configuration must point to those paths, and the hub and.

Use the commands and manifests as a record of the lab workflow, then adapt and validate them against the exact RHACM, MCE, OpenShift, and Assisted Installer versions in your environment.

## SEO description and suggested tags

**SEO description:** Hands-on guide to OpenShift Assisted Installer and RHACM Host Inventory for connected and disconnected environments. Covers AgentServiceConfig, local RHCOS ISO hosting, Quay mirror-registry configuration, ClusterImageSet, registry CA trust, and troubleshooting.

**Suggested tags:** Red Hat OpenShift, Assisted Installer, RHACM, Multicluster Engine, Host Inventory, Disconnected OpenShift, Quay, Mirror Registry, AgentServiceConfig, ClusterImageSet

## References

- [Red Hat Advanced Cluster Management documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes)
- [OpenShift Container Platform documentation](https://docs.redhat.com/en/documentation/openshift_container_platform)
