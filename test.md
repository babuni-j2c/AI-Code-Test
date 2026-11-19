# Configure Loki for cluster logging 


https://access.redhat.com/articles/7089751

https://access.redhat.com/articles/6991632

https://access.redhat.com/articles/7006275

https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/logging/logging-6-2

Labels;


https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/logging/logging-6-0#filters-and-pipeline-configuration

https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/logging/logging-6-0#log6x-input-spec-filter-labels-expressions_logging-6x

https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/logging/logging-6-0#log6x-input-spec-filter-namespace-container_logging-6x


https://access.redhat.com/solutions/5928241?band=se&seSessionId=1fe11905-f080-4007-94d6-269150e36d76&seSource=Recommendation&seResourceOriginID=8e79a701-f59a-4bd6-8166-cfc1b0e0f812

https://access.redhat.com/articles/7020079?band=se&seSessionId=1fe11905-f080-4007-94d6-269150e36d76&seSource=Recommendation&seResourceOriginID=8e79a701-f59a-4bd6-8166-cfc1b0e0f812



# OCP Logging stack configuration with Loki  

https://docs.openshift.com/container-platform/4.14/logging/log_storage/installing-log-storage.html

https://docs.openshift.com/container-platform/4.15/observability/logging/log_storage/installing-log-storage.html

https://docs.openshift.com/container-platform/4.15/observability/logging/log_storage/cluster-logging-loki.html

# OCP Logging stack configuration with Loki 

https://docs.openshift.com/container-platform/4.17/observability/logging/logging-6.1/log6x-about-6.1.html


# OCP 4.16 or above version::


```
1) Install logging operator
2) Install loki operator
```

3) Create a OBC
   
```
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: loki-bucket-odf
  namespace: openshift-logging
spec:
  generateBucketName: loki-bucket-odf
  storageClassName: openshift-storage.noobaa.io
```

4) Get the OBC details

```
$ BUCKET_HOST=$(oc get -n openshift-logging \
  configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_HOST}'); \
  BUCKET_NAME=$(oc get -n openshift-logging configmap loki-bucket-odf \
  -o jsonpath='{.data.BUCKET_NAME}'); \
  BUCKET_PORT=$(oc get -n openshift-logging configmap loki-bucket-odf \
  -o jsonpath='{.data.BUCKET_PORT}'); \
  ACCESS_KEY_ID=$(oc get -n openshift-logging secret loki-bucket-odf \
  -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d); \
  SECRET_ACCESS_KEY=$(oc get -n openshift-logging secret loki-bucket-odf \
  -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
```
5) Create a secret using above details.
```
oc create secret generic logging-loki-odf \
  -n openshift-logging \
  --from-literal=access_key_id=${ACCESS_KEY_ID} \
  --from-literal=access_key_secret=${SECRET_ACCESS_KEY} \
  --from-literal=bucketnames=${BUCKET_NAME} \
  --from-literal=endpoint=https://${BUCKET_HOST}:${BUCKET_PORT}
```
6) Configure Loki Instance 

```
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.demo   ##Based on your requirement change the size
  storage:
    secret:
      name: logging-loki-odf
      type: s3
    tls:
      caName: openshift-service-ca.crt  ## oc get cm you will get this 
  storageClassName: ocs-external-storagecluster-ceph-rbd
  tenants:
    mode: openshift-logging
```

7) Create cluster loggin instance

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  managementState: Managed
  logStore:
    type: lokistack
    lokistack:
      name: logging-loki
  collection:
    type: vector
  visualization:
    type: ocp-console    ##Kibana and ocp-console both are supported if you are working on Elasticksearch
```

8) Enable the console pluggin, if not enabled during the logging operator installation

9) Validate the logs
```
   > OpenShift Web UI > Observe > logs
                                      > Infrastructure / Application / Audit (Default Audit logs are not available)
```
10) Configure Audit logs

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  pipelines:
  - name: all-to-default
    inputRefs:
    - infrastructure
    - application
    - audit
    outputRefs:
    - default
```

11) Validate the logs
```
   > OpenShift Web UI > Observe > logs
                                      > Infrastructure / Application / Audit 
```





# Advance Configuration.



# Loki Instance
```
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  limits:
    global:
      retention:
        days: 30
        streams:
          - selector: '{kubernetes_namespace_name="build-ci"}'
            priority: 1
            days: 7
  size: 1x.demo
  storage:
    tls:
      caName: openshift-service-ca.crt
    secret:
      name: logging-loki-odf
      type: s3
  storageClassName: ocs-external-storagecluster-ceph-rbd
  tenants:
    mode: openshift-logging
```




# CL Instance Configuration

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  managementState: Managed
  logStore:
    type: lokistack
    lokistack:
      name: logging-loki
  collection:
    type: vector
  visualization:
    type: ocp-console
```




# CLF Instance Configuration

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  inputs:
    - name: production-apps
      application:
        selector:
          matchLabels:
            environment: production
    - name: ci
      application:
        namespaces:
        - build-ci
  outputs:
    - name: audit-syslog
      type: syslog
      url: tcp://utility.lab.example.com:514
      syslog:
        msgID: audit
        appName: ocp-lab
        facility: user
        procID: vector
        rfc: RFC5424
        severity: informational
  pipelines:
    - name: to-syslog
      inputRefs:
        - audit
      outputRefs:
        - audit-syslog
    - name: to-loki
      inputRefs:
        - infrastructure
        - ci
        - production-apps
      outputRefs:
        - default
```

```
$ oc adm policy add-cluster-role-to-group cluster-logging-application-view ocpdevs
```


