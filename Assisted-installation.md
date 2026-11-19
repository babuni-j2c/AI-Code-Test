
**Note: (Best approach always is create host and make host available on host inventory and use existing host to create cluster**

cause in future if you want to delete cluster, the cluster will get dalete but not the node and the node will availabe/free for configure new cluster.

**Note: Hosted Cluster: (Only possible once hosts are available on host inventory)**
```
- Create HostInventory for Connected ENV.
- Create HostInventory for Disconnected ENV.
```

**Host inventory Assisted Cluster installation**
```
1) SNO
2) MultiNodes
3) Cluster Managed Network (IPI | Required 2 Additional Ips 1-api 2-Ingress)
4) User Managed Network (UPI | Required Haproxy Load Balancer and the configuration details before cluster installation)
```



**Pratical:**


 Steps: (Connected Cluster | Disconnected Cluster)
```
-> Create Host Inventory
-> Configure "Configure Host Inventory Setting"
```
```
0) prerequisite: StorageClass with Default setting

1) ACM-WebUI
2) Infrastructure -> Host inventory -> 'Configure host inventory settings'  -> Configuration 
                                         - Database Storage: 30
                                         - System Storage: 150
                                         - Image Storage: 50
3) It will take 5 to 10 minutes for the configuration.
```

**Validation:**
```
 $ oc get po 
agentinstalladmission-858f989cbf-mc769                 1/1     Running   0               19h
agentinstalladmission-858f989cbf-r8ln9                 1/1     Running   0               19h
assisted-image-service-0                               1/1     Running   0               16m
assisted-service-5b46d88c7f-6spw5                      2/2     Running   0               16m
```
```
 $ oc get pvc
 $ oc get pv
```


**For Disconnected ENV:**
```
1) Follow the same process and create "Configure host inventory settings"
2) Everything will get start but the 'assisted-image-service-0' pod will not get running state.
$ oc logs assisted-image-service-0
(It will try to download the ISO from internet "mirror registry.redhat")
3) Please update the configuration to pull ISO from local server. (add additional perameters.)
```
```
$ oc get agentserviceconfigs 
$ oc edit agentserviceconfig/agant
```
```
  osImages:
  - cpuArchitecture: 'x86_64'
    openshiftVersion: '4.17.16' ##OpenShift Version
    rootFSUrl: ""               ##Not require, else copy the rootfs https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/
    url: 'http://10.4.60.59:8080/iso/rhcos-4.17.2-x86_64-live.x86_64.iso' ##Download and make available on local server. Use the above URL 
    version: '417.94.202502051822-0' ## oc adm release info quay.io/openshift-release-dev/ocp-release:<ocp_version>-x86_64 -> machine-OS
```
```

Add the CM for the registry details:
- RootCA
- Registry Path
```

**Create a CM**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: quay-registry-details
  namespace: multicluster-engine
  labels:
    app: assisted-service
data:
  ca-bundle.crt: |
    <openssl s_client -connect <quay-endpoint>:<port> -showcerts>
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
```
```
Note: 

Here i am using new Quay-mirror-registry to install all cluster using RHACM-HuB cluster. For that 1st need to mirror all the images 
       from quay-standalone to quay-ocp-registry server (Create manuall all the repo and mirror all the repo and check the sync status)
```

**-> Update agentserviceconfig**
```
$ oc edit agentserviceconfig/agant

  mirrorRegistryRef:
    name: quay-registry-details
  osImages:
  - cpuArchitecture: x86_64
    openshiftVersion: 4.17.16
    rootFSUrl: ""
    url: http://service.example.com:8080/iso/rhcos-4.17.2-x86_64-live.x86_64.iso
    version: 417.94.202502051822-0
```



**For Disconnected ENV:**
```
- Use the existing quay-mirror-registry or
- Create new quay-mirror-registry from operator on openshift cluster
- Mirror all the images from standalone quay-mirror-registry to quay-mirror-registry operator on openshift
- 2 ways to do the mirror process
   1) Default process (Create imagesetconfig, follow the RH docs for mirror image process)
   2) Create manually new organization on Quay operator registry and mirror the images
```     
```
1) Create ClusterImageSet (Without ClusterImageSet, The "Host Inventory" option will get visible)
- Infrastructure -> Clusters -> Create Cluster -> Choose "Host Inventory" (This option is not visable without clusterImageSet)
 ```
```
$ oc get clusterimagesets


apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  labels:
    channel: fast
    visible: 'true'
  name: img-4-17-16-x86-64
spec:
  releaseImage: central-quay-registry.apps.ocp-hub.intern.tpit.test/openshift/release-images:4.17.16-x86_64 ##Your registry -> openshft/release-images docker/podman pull path
```



***Create the Credentials***
```
1) ACM-WebUI -> Credentials -> Add Credential  -> Credential Types -> Hosted Inventory
                           - Credential types: Host Inventory (Auto filled)
                           - Credential Name: (Any name)
                           - Name Space: rhacm-cerdentials(any-NS | create It)
                           - Base DNS Name: intern.j2ctechnologies.com
```
```
  Next:
                    - PullSecret: Add the pull-secret (Note: For connected cluster, use default pull-secret | for disconnected cluster, use your image registry pull-secret)
                         Note: (During the cluster creation if you faced any issues releted to pull-secret, pls update new-registry and global pull secret)
                    - sshKey: add the sshKey
```


**Update the Image.config.openshift.io for Pull images from quay-registry and add the certificate.**

-> get the Certs.
```
$ openssl s_client -connect <quay-endpoint>:<port> -showcerts
```

-> Create the CM
```
$ oc create configmap registry-cas -n openshift-config \
--from-file=myregistry.corp.com..5000=quay-ca.crt \
--from-file=otherregistry.com=central-quay-ca.crt
```
```
$ oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}' --type=merge
```
```
$ oc get crd| grep image
$ oc edit image.config.openshift.io

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
```



**Error-00: (If  Image.config.openshift.io is not updated)**
```
The Spec could not be synced due to backend error: failed to get release image 'central-quay-registry.apps.ocp-hub.intern.tpit.test/openshift/release-images:4.17.16-x86_64'.
Please ensure the releaseImage field in ClusterImageSet 'img-4-17-16-x86-64' is valid, (error: command 'oc adm release info -o template --template '{{.metadata.version}}' 
--insecure=false central-quay-registry.apps.ocp-hub.intern.tpit.test/openshift/release-images:4.17.16-x86_64 --registry-config=/tmp/registry-config2756971530' exited with 
non-zero exit code 1: error: unable to read image central-quay-registry.apps.ocp-hub.intern.tpit.test/openshift/release-images:4.17.16-x86_64: Get
"https://central-quay-registry.apps.ocp-hub.intern.tpit.test/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority ).
```

**Error-01: (If the Agent.service is not running and its restarting - sudo journalctl -u agent.service -> Unable to pull the image registry.redhat.io/multicluster-engine/assisted-installer-agent-rhel8@sha: a8798fs876fs78 )**
```
-> Check the quay registry status if require, restart all the pods
-> Manually pull the images to test the quay connectivity
   $ podman pull quay.example.com/multicluster-engine/assisted-installer-agent-rhel8@sha: a8798fs876fs78
-> Check the SHA:dww7667fcs78c68scs  - availableon quay registry or not.
-> If everythigs look ok then it should work expected. 
-> I got this error due to quay not healthy - due to s3 connection issues - s3 pods is not running state on openshift-storage ns.
```



**Error-02: (Nodes are not getting Up)**
If your mirror registry having some SYNC issues or images are not available on the "registry.mirror" path
```
    [[registry]]
       prefix = ""
       location = "registry.redhat.io/multicluster-engine"
       mirror-by-digest-only = true
       [[registry.mirror]]
       location = "central-quay-registry.apps.ocp-hub.intern.example.com/multicluster-engine"
```
```
ps -ef | grep agent

sudo journalctl -u agent.service

sudo journalctl TAG=agent | less

sudo su
podman ps -a | grep assisted-installer
podman logs <container id>

sudo journalctl -u bootkube
```



**Error-03: (Cluster installation is not moving futher, Due to below error.)**

Container images availability: Failed to fetch container images needed for installation from 

quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ec4767f65cbd7acddcb107607245d7f701e769fd35ece8f141bdac67105bdcc6,

quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:3fd8ae627bdf3482a0a1d735a50c65a4f3e6a985af224bea279a62694bf2c118. 

This may be due to a network hiccup. Retry to install again. If this problem persists, check your network settings to make sure you’re not blocked.

Fix it using below details and check the sync status 
```
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
```
