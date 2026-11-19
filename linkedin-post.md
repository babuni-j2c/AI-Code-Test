# Configure OpenShift Logging with LokiStack and OpenShift Data Foundation

How do you build a centralized logging solution on OpenShift using LokiStack and OpenShift Data Foundation?

In my lab, I worked through an implementation pattern that brings together OpenShift Logging, LokiStack, and ODF NooBaa object storage.

The configuration covers the complete workflow:

🔹 Deploy the required OpenShift Logging and Loki operators  
🔹 Provision S3-compatible object storage using an ObjectBucketClaim (OBC)  
🔹 Configure LokiStack to use ODF object storage  
🔹 Configure ClusterLogForwarder for application and infrastructure logs  
🔹 Enable the logging view in the OpenShift web console  
🔹 Validate log collection and review troubleshooting steps  

One important detail: OpenShift Logging 6.x uses the `observability.openshift.io/v1` ClusterLogForwarder API. This differs from the legacy logging APIs, so it is important to check the operator and OpenShift version compatibility before applying manifests.

I have documented the architecture, YAML examples, CLI commands, prerequisites, validation steps, and troubleshooting notes in the repository.

📘 Technical guide: https://github.com/babuni-j2c/AI-Code-Test/blob/main/test.md

💻 Repository: https://github.com/babuni-j2c/AI-Code-Test

This is a lab-oriented reference implementation. Review the official Red Hat documentation and validate the configuration in a non-production environment before using it in production.

I welcome feedback, suggestions, and discussions from the OpenShift community.

#OpenShift #RedHatOpenShift #OpenShiftLogging #Loki #LokiStack #OpenShiftDataFoundation #ODF #CloudNative #Kubernetes #DevOps
