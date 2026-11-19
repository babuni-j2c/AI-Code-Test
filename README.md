# OpenShift Logging with LokiStack and ODF

A practical guide to configuring OpenShift Logging with LokiStack and OpenShift Data Foundation (ODF) object storage.

## What this repository contains

| File | Description |
|---|---|
| [test.md](./test.md) | Full technical walkthrough with prerequisites, architecture, YAML manifests, CLI commands, validation, and troubleshooting |
| [linkedin-post.md](./linkedin-post.md) | A polished LinkedIn article/post draft introducing the lab implementation |

## Scope

The technical walkthrough focuses on:

- OpenShift Logging 6.x
- LokiStack
- OpenShift Data Foundation (ODF) NooBaa object storage
- ObjectBucketClaim (OBC)
- ClusterLogForwarder using `observability.openshift.io/v1`
- OpenShift console logging view using UIPlugin

## Before you begin

You need:

- An OpenShift cluster with administrator access.
- OpenShift Data Foundation configured for object storage.
- Red Hat OpenShift Logging Operator.
- Loki Operator.
- Cluster Observability Operator for the console logging view.
- OpenShift CLI (`oc`).

## Important compatibility note

OpenShift Logging has an independent release cycle from OpenShift Container Platform. Check the supported operator/platform combination and the APIs available in your environment.

The examples in this repository use the Logging 6.x ClusterLogForwarder API (`observability.openshift.io/v1`). They are not intended for legacy Logging 5.x configurations that use `logging.openshift.io` resources.

The LokiStack size `1x.demo` is for demonstration only. Choose a supported production size based on official sizing guidance.

## Quick start

Read the complete procedure in [test.md](./test.md). It covers:

1. Creating the ODF ObjectBucketClaim.
2. Retrieving bucket configuration and credentials.
3. Creating the Loki object-storage Secret.
4. Deploying LokiStack.
5. Creating the collector service account and granting permissions.
6. Configuring ClusterLogForwarder.
7. Enabling the logging UIPlugin.
8. Validating log collection and troubleshooting.

## Security

- Never commit real access keys, tokens, kubeconfigs, or passwords.
- Do not paste Secret values into issues, pull requests, screenshots, or public documentation.
- Use placeholders for environment-specific values.
- Apply least-privilege RBAC and rotate credentials if they are exposed.

## Validation status

These are reference examples and should be validated against the exact OpenShift, OpenShift Logging, Loki Operator, and Cluster Observability Operator versions in your environment before production use.

## Official documentation

- [OpenShift Logging 6.2 – Configuring logging](https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.2/html-single/configuring_logging/index)
- [OpenShift Logging 6.2 – Installing logging](https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.2/html-single/installing_logging/index)

## Contributing

Corrections, version-specific notes, and tested improvements are welcome. Please include the OpenShift and operator versions used when submitting changes.
