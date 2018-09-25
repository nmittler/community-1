---
kep-number: TBD
title: Sidecar Lifecycle
authors:
  - "@nmittler"
owning-sig: sig-apps
participating-sigs:
  - sig-apps
  - sig-node
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2018-09-25
last-updated: 2018-09-25
status: provisional
---

# Sidecar Containers

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints-optional)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Alternatives](#alternatives-optional)

## Summary

This proposal modifies the lifecyle management for containers within the same Pod in order to address various issues when using sidecar containers. The goal is to control container startup and shutdown in a way that will work for the majority of sidecar-based applications, without requiring custom logic or configuration in the application containers.

## Motivation

Sidecars containers have existed for a while but have never formally been identified as first-class citizens. Since sidecars are "just containers", there are no guarantees with respect to startup/shutdown ordering.

Since more and more applications have been written which rely (either directly or indirectly) on the sidecar in order to function, application developers have had to resort to hacking their Docker images in order to manually script lifecycle events This becomes much more difficult when dealing with third-party images.

Some of the issues include:

### Startup

* An application using a sidecar proxy needs to connect to a remote service at startup. In this case the outbound connection will fail until the proxy is fully operational. If this takes a while, the application may crash-loop and significantly delay application startup. This was seen when running an [NGINX controller](https://github.com/istio/istio/issues/3533#issuecomment-366261085), which requires communication with Kube API Server. Readiness probes/gates are not sufficient to address this problem, since readiness only affects inbound traffic.

### Shutdown

* An application using a sidecar proxy needs to inform another service that it's going down. Since there are no guarantees on container shutdown ordering, it's a race as to whether this message gets sent or not.
* A Job with two containers where one is actually doing the main processing and the other is just facilitating it. If the main process finishes, the sidecar container will carry on running so the job will never finish.


### Goals

Solve the following issues so that sidecars can be easily handled without needing modifications to the container itself:

* [25908](https://github.com/kubernetes/kubernetes/issues/25908)
* [65502](https://github.com/kubernetes/kubernetes/issues/65502)

### Non-Goals

// TODO(nmittler)

## Proposal

Provide a way of identifying containers as sidecars with an additional field to the Container Spec: `sidecar: true`. For example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: myapp
    command: ['do something']
  - name: sidecar
    image: sidecar-image
    sidecar: true
    command: ["do something to help my app"]

```


The lifcycle of containers within the same Pod will be modified in the following manner:

### Pod Startup

1. Sidecars are started (order unspecified).
2. Non-sidecars are started (order unspecified) after all sidecars have been started and are ready (if any/all sidecars define readiness probes).

### Pod Shutdown

1. Before any container is terminated, `preStop` hooks for all containers are executed (sidecars first, followed by non-sidecars). This provides all containers with an opportunity to begin a graceful shutdown before receiving SIGTERM.
    - A use case for this is when running a sidecar proxy. At the beginning of Pod shutdown, it's desirable to have the sidecar enter a "lameduck" mode, where it immediately rejects any inbound requests, while still allowing outbound connections from the application container.
2. Non-sidecars are terminated (order unspecified).
3. Sidecars are terminated (order unspecified).

### Implementation Details/Notes/Constraints

As this is a change to the Container spec we will be using feature gating, you will be required to explicitly enable this feature on the api server as recommended [here](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#adding-unstable-features-to-stable-versions).

### Risks and Mitigations

You could set all containers to be `sidecar: true`, this seems wrong, so maybe the api should do a validation check that at least one container is not a sidecar.

Init containers would be able to have `sidecar: true` applied to them as it's an additional field to the container spec, this doesn't currently make sense as init containers are ran sequentially. We could get around this by having the api throw a validation error if you try to use this field on an init container or just ignore the field.

## Graduation Criteria

// TODO(nmittler)

## Implementation History

// TODO(nmittler)

## Alternatives

// TODO(nmittler)