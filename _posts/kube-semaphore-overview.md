---
layout: post
title: "Kubernetes Semaphore: A lightweight framework for cross cluster communication"
date: ""
categories: kubernetes, multicluster
---

Kube-Semaphore is a lightweight framework to provide easy, reliable and secure
communication between deployments that run in different Kubernetes clusters,
without requiring any changes to the applications code or sidecar containers.

This is not intended to implement a service mesh model, but aims to provide
service endpoints and firewall rules for workloads that reside in a remote
cluster.

It is implemented as a set of tools that will target the following:

- Encryption on traffic between Kubernetes clusters.

- Exposing Kubernetes services from one cluster to another to avoid going
  through external load balancers.

- Firewall rules on cross cluster traffic on pod to pod communication level.

# How it works - Overview

The solution works via deploying a set of small applications, each of which will
be in charge of implementing one of the above targets. Networking between hosts
in different Kubernetes clusters is handled via WireGuard, thus offering a node
to node traffic encryption model by default. Services and firewall are
implemented via watching resources in a target cluster and mirroring all the
needed information locally, to be able to handle everything with native
Kubernetes or Calico resources. The only thing needed in a remote cluster for
the local controllers to work is a set of service accounts that will allow
watching the resources of interest.

In order to be as small, lightweight, and safe as possible, the components are
written in Golang and use the respective Kubernetes and Calico client
implementations.

# Encryption

[Semaphore-Wireguard](https://github.com/utilitywarehouse/semaphore-wireguard) is responsible for handling encryption between nodes of
different clusters. It is essentially a WireGuard peer manager that runs on
every node in all clusters and automates the peering between them. It is
responsible for generating local keys and discovering all the remote keys and
endpoints to configure successful peering between all the nodes. If combined
with WireGuard for in cluster traffic (offered by Calico), the end result will
be a full mesh between all nodes in our clusters and all traffic travelling
between nodes via the created WireGuard network.

# Services

[Semaphore-Service-Mirror](https://github.com/utilitywarehouse/semaphore-service-mirror) is a controller responsible for mirroring services
from one Kubernetes cluster to another. We define as a service mirror a local
Kubernetes service with endpoints that live in a remote cluster. More in
particular, the mirroring controller will create local services in a cluster and
will update the list of endpoints for each service with the IP addresses of pods
from a remote cluster. That way we will have the same benefits of using a
Kubernetes ClusterIP service (e.t.c. implementation on the host via iptables or
ipvs) but we will be effectively targeting remote workloads when sending traffic
towards a mirrored service. Additionally, a small [tweak](https://github.com/utilitywarehouse/semaphore-service-mirror#coredns-config-example) will be needed
in order to allow us to use CoreDNS and resolve the new services using friendly
names.

# Policy

[Semaphore-Policy](https://github.com/utilitywarehouse/semaphore-policy) is the component responsible for populating the needed
resources to allow us creating firewall rules for traffic originated from a
remote cluster. The objective here is to create sets of IPs that will be used
in Calico Network Policies to define which traffic should be allowed. The
controller has only one task at hand, to watch remote pods based on a label and
create local NetworkSets with all the discovered IP addresses. Then we can use
simple labels to describe those sets inside a Calico Network Policy and
effectively implement cross cluster firewall rules.

# Next Step

A deep dive post is available with a lot more details on how all the above work
[here]({{site.url}}/blog/_posts/kube-semaphore.md).
