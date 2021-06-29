---
layout: post
title: "Kubernetes Semaphore: A lightweight framework for cross cluster communication"
date: "2021-06-29 16:26:00 +0100"
categories: kubernetes, multicluster
---

# The problem

Our environment consists of three Kubernetes clusters that are hosted in AWS,
GCP and on premises. We have a flat layer 3 network between clusters, which
allows communication between all nodes. Each cluster allocates nodes in a
dedicated subnet:

- AWS: 10.66.21.0/24
- GCP: 10.22.20.0/24
- On-prem: 10.88.0.0/24

Equally important, the subnets from which pods are assigned IP addresses are:

- AWS: 10.2.0.0/16
- GCP: 10.4.0.0/16
- On-prem: 10.6.0.0/16

We want applications running in different clusters to be able to communicate.
More specifically our objectives are:

- Encryption on Pod to Pod communication.
- Targeting a Kubernetes service that lives in a remote cluster.
- Adding rules to allow certain applications from a remote cluster talk to local
  endpoints.

# The Limitation

We run Calico as CNI in every cluster, and thus we have built a solution that
relies on it.

# Existing Solutions

We have reviewed Istio, Linkerd, Consul and also played with our own configurer
for envoy proxy directly. Even though each of these solutions was able to
provide us most or all the above goals, we have decided that none fits our
environment well enough to make the investment worthy. We were not necessarily
interested in a service mesh between applications in different clusters so we
would not benefit from a great amount of the functionality offered by these
frameworks. Moreover, we wanted to avoid using sidecar proxies and the extra
overhead because of them and try to make sure that our applications remain
agnostic of the underlying solution to talk outside of the cluster.

# The design

Adding to the above mentioned decision to avoid using sidecar proxies, we wanted
to solve these problems in a simple way, both from operational and user point of
view. Ideally, each goal should be achieved in isolation, so for example if
users need only encryption for their pod communication, they should be able to
deploy just that. Also, we consider important to require the minimum buy in for
new users, so that they need minimal configuration to try the solution and an
easy way to revert.

# The solution

Kube-Semaphore is a lightweight framework to provide easy, reliable and secure
communication between deployments that run in different Kubernetes clusters,
without requiring any changes to the applications code or sidecar containers.

This is not intended to implement a service mesh model, but aims to provide
service endpoints and firewall rules for workloads that reside in a remote
cluster.

It is implemented as a set of 3 independent tools as follows:

- Semapore-Wireguard: Responsible for the encryption on traffic between
  Kubernetes clusters.

- Semaphore-Service-Mirror: Responsible for exposing Kubernetes services from
  one cluster to another to avoid going through external load balancers.

- Semaphore-policy: Responsible for creating firewall rules on cross cluster
  traffic on pod to pod communication level.

In order to be as small, lightweight, and safe as possible, the components are
written in Golang and use the respective Kubernetes and Calico client
implementations. Also, the footprint on a remote cluster is minimal, as the only
thing needed for local controllers to work is a set of service accounts that
will alllow watching the resources of interest.

# Encryption

[Semaphore-Wireguard](https://github.com/utilitywarehouse/semaphore-wireguard) is responsible for handling encryption between nodes of
different clusters. It is essentially a WireGuard peer manager that runs on
every node in all clusters and automates the peering between them. It is
responsible for generating local keys and discovering all the remote keys and
endpoints to configure successful peering between all the nodes. If combined
with WireGuard for in cluster traffic (offered by Calico), the end result will
be a full mesh between all nodes in our clusters and all traffic travelling
between nodes via the created WireGuard network.

A deployment example can be found [here](https://github.com/utilitywarehouse/semaphore-wireguard/tree/main/deploy/example)

The following diagram illustrates the created WireGuard mesh between our hosts,
where our on-prem cluster is named "merit":
![full-mesh]({{site.url}}/blog/png/semaphore-wireguard-full-mesh-example.png){:class="img-responsive"}

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

A deployment example can be found [here](https://github.com/utilitywarehouse/semaphore-service-mirror/tree/master/deploy/example)

For example, assuming that we have a Service resource in `AWS` cluster as:
```
kubectl --context=aws --namespace=sys-log get service fluentd
NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
fluentd   ClusterIP   10.3.88.18   <none>        8888/TCP,8889/TCP   164d
```
and the respective Endpoints:
```
kubectl --context=aws --namespace=sys-log get endpoints fluentd
NAME      ENDPOINTS                                                  AGE
fluentd   10.2.3.19:8889,10.2.4.19:8889,10.2.7.18:8889 + 3 more...   164d
```

The mirror controller will create the respective service and endpoints under the
configured namespace, in this case `sys-semaphore`:
```
kubectl --context=gcp --namespace=sys-semaphore get service | grep fluentd
aws-sys-log-73736d-fluentd   ClusterIP   10.5.184.192   <none>        8888/TCP,8889/TCP   25d
```
```
kubectl --context=gcp --namespace=sys-semaphore get endpoints | grep fluentd
aws-sys-log-73736d-fluentd   10.2.3.19:8889,10.2.4.19:8889,10.2.7.18:8889 + 3 more...   17d
```

Looking at the end result we have a Kubernetes Service with endpoints that point
to pod IPs from a remote cluster:
```
kubectl --context=gcp --namespace=sys-semaphore describe service aws-sys-log-73736d-fluentd | grep Endpoints
Endpoints:         10.2.3.19:8888,10.2.4.19:8888,10.2.7.18:8888
Endpoints:         10.2.3.19:8889,10.2.4.19:8889,10.2.7.18:8889
```

Since our controller will be watching the remote resources and updating on any
event, the mirrored service should always have up to date information regarding
the available endpoints. Moreover, the mirrored service is a local Kubernetes
cluster-IP service and will be implemented based on how the cluster is
configured to deploy services (for example ipvs or iptables).

Finally, if we follow this CoreDNS [trick](https://github.com/utilitywarehouse/semaphore-service-mirror#coredns-config-example), we will be able to resolve the
mirrored service in a rational manner:

```
# drill fluentd.sys-log.svc.cluster.aws
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 51067
;; flags: qr aa rd ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; fluentd.sys-log.svc.cluster.aws.     IN      A

;; ANSWER SECTION:
fluentd.sys-log.svc.cluster.aws.        5       IN      A       10.5.184.192
```
So our pods can use human friendly names and would not require any knowledge of
the mirroring mechanism.

# Policy

[Semaphore-Policy](https://github.com/utilitywarehouse/semaphore-policy) is the component responsible for populating the needed
resources to allow us creating firewall rules for traffic originated from a
remote cluster. The objective here is to create sets of IPs that will be used
in Calico Network Policies to define which traffic should be allowed. The
controller has only one task at hand, to watch remote pods based on a label and
create local NetworkSets with all the discovered IP addresses. Then we can use
simple labels to describe those sets inside a Calico Network Policy and
effectively implement cross cluster firewall rules.

A deployment example can be found [here](https://github.com/utilitywarehouse/semaphore-policy/tree/main/deploy)

For example, let's consider the following deployment in our `GCP` cluster:
```
$ kubectl --context=gcp --namespace=sys-log get po -o wide -l policy.semaphore.uw.io/name=forwarder
NAME              READY   STATUS    RESTARTS   AGE     IP          NODE                                      NOMINATED NODE   READINESS GATES
forwarder-4jdm6   1/1     Running   0          3d20h   10.4.1.3    worker-k8s-exp-1-4l87.c.uw-dev.internal   <none>           <none>
forwarder-6ztl4   1/1     Running   0          3d20h   10.4.0.13   worker-k8s-exp-1-2868.c.uw-dev.internal   <none>           <none>
forwarder-klxdc   1/1     Running   0          4h27m   10.4.4.2    master-k8s-exp-1-j5f8.c.uw-dev.internal   <none>           <none>
forwarder-m9k27   1/1     Running   0          4h27m   10.4.5.2    master-k8s-exp-1-fc0b.c.uw-dev.internal   <none>           <none>
forwarder-n6nsn   1/1     Running   0          4h27m   10.4.3.3    master-k8s-exp-1-31rv.c.uw-dev.internal   <none>           <none>
forwarder-n8vnj   1/1     Running   0          3d20h   10.4.2.4    worker-k8s-exp-1-mdd7.c.uw-dev.internal   <none>           <none>
```
Which shows a daemonset named `forwarder` under `sys-log` namespace. In order
for the policy controller to create the needed resources in a remote cluster, we
need to make sure that all the pods of the above daemonset are labelled as: 
`policy.semaphore.uw.io/name=forwarder`. That will trigger the aws controller to
create the respective GlobalNetworkSet as described above:
```
 $ kubectl --context=aws describe GlobalNetworkSet gcp-sys-log-forwarder
Name:         gcp-sys-log-forwarder
Namespace:
Labels:       managed-by=semaphore-policy
              policy.semaphore.uw.io/cluster=gcp
              policy.semaphore.uw.io/name=forwarder
              policy.semaphore.uw.io/namespace=sys-log
Annotations:  projectcalico.org/metadata: {"uid":"c7569765-a47d-424c-9533-80e4a7c201d6","creationTimestamp":"2021-04-09T15:04:43Z"}
API Version:  crd.projectcalico.org/v1
Kind:         GlobalNetworkSet
Spec:
  Nets:
    10.4.5.2/32
    10.4.4.2/32
    10.4.1.3/32
    10.4.0.13/32
    10.4.3.3/32
    10.4.2.4/32
Events:  <none>
```

That is now a set which presents the remote deployments IP addresses and can be
used in a namespaced Calico NetworkPolicy as follows:
```
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: allow-to-fluentd
spec:
  selector: app.kubernetes.io/name == 'fluentd'
  types:
    - Ingress
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: >-
          policy.semaphore.uw.io/name == 'forwarder' &&
          policy.semaphore.uw.io/namespace == 'sys-log' &&
          policy.semaphore.uw.io/cluster == 'gcp'
        namespaceSelector: global()
      destination:
        ports:
          - 8889
```

The above rule will allow traffic from our remote `forwarder` to our local
