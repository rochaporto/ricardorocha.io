+++
author = "Ricardo Rocha"
date = "2021-01-11T14:00:00+00:00"
description = "A simple and declarative way to maintain minimal and reproducible cluster node images"
tags = ["kubernetes", "docker", "containerd", "linuxkit"]
title = "Building Immutable Cluster Images with LinuxKit"
+++

One of the challenges of running large scale infrastructures is avoiding
configuration drift and making sure nodes are kept consistent and up to date. One
way of achieving this is relying on systems like CFEngine, Puppet or Chef, regularly
(re)applying the expected configuration on the nodes ([SnowFlakes](https://martinfowler.com/bliki/SnowflakeServer.html)).
A limitation of this approach is that drift will only be spotted in areas defined under these tools' control and
for long lived instances it's common to find inconsistencies in other areas -
practice tells us there will always be some.

+ Issue with tools not running properly from time to time?

An alternative is to rely on immutable infrastructure, where nodes
([PhoenixServers](https://martinfowler.com/bliki/PhoenixServer.html)) are launched
from a base image and never changed. Updates are done directly on the base
image (possibly created using the tools above, most often not) and nodes are recreated from
scratch, ensuring any configuration drift is removed. It also improves
reproducibility and makes deployments more predictable - including more reliable rollbacks
when needed.

> *Uptime bragging rights will be lost, but probably not missed by many...*

The latter is a popular approach for containerized environments with the nodes
being set with a small image containing only the minimal set of dependencies to
launch additional service and application containers. There are multiple
advantages to this approach which are covered below.

## Containers and Immutable Infrastructure

Orchestration systems like Kubernetes assume nodes are by default disposable and
allow workloads to express their scheduling and placement restrictions in a declarative
way - in the case of Kubernetes these include
things like [Affinities](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity),
[TopologyContraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/), [PodDisruptionBudgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets), etc. Node cordoning and draining allow workloads to be safely terminated when a node is refreshed, making these setups well suited for an immutable infrastructure.

Tools that explore this in the Kubernetes and container area include [RancherOS](https://rancher.com/docs/os/v1.x/en/),
[Flatcar](https://www.flatcar-linux.org/) (a fork from pre-RH CoreOS), [Fedora
CoreOS](https://getfedora.org/en/coreos?stream=stable) or GCP's [Container-Optimized OS](https://cloud.google.com/container-optimized-os/docs). This post looks at what can be done in [LinuxKit](https://github.com/linuxkit/linuxkit), but the same layout
is valid for the other tools.


The picture above shows a mixed environment.

* Speed of updates

* Separation between base node and application workloads - containerd is the
  interface

* Containerization improves the situation as nodes do not necessarily need to be recreated, as long as only the containers should be upgraded.

- Tradeoffs (state, local data ... needs to be externalized or properly replicated)

* Levels:
  * Node dependencies
  * Containers / applications


## Maintaining a Base Image with LinuxKit

Here's a full sample 
- Difference in image size


## Adding a New Component


