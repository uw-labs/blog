---
layout: post
title: "Kubernetes Semaphore: A simple framework to automate cross cluster communication"
date: ""
categories: kubernetes
---

# Kubernetes Semaphore

This is a collection of components we deploy under the same namespace we called
`semaphore`, in order to support communication between deployments in different
Kubernetes clusters.

## The Challenge

`Semaphore` components try to address the following concerns we faced on pod
to pod communication between pods which live in different clusters:

- Encryption on traffic between Kubernetes clusters.
- Exposing Kubernetes services from one cluster to the other to avoid going
  through external load balancers.
- NetworkPolicies for traffic coming from another cluster.

For each one of the above, we developed an independent app called respectively:
- semaphore-wireguard
- semaphore-service-mirror
- semaphore-policy

## Limitations/Requirements

- The aim of the project is not to provide networking between nodes in different
  clusters or providers. It strictly tries to tackle the above mentioned
  challenges and assumes that node to node network connectivity is something
  already achieved.
- We use Calico as a CNI solution in our clusters and we rely on Calico CRDs in
  order to implement cross cluster network policies. Thus running Calico in all
  clusters is a requirement for this to work.
- Nodes' pod subnets should be assigned using Kubernetes node `spec.PodCIDR`
  along with the hostLocal IPAM.

## Semaphore-Wireguard

[Semaphore-Wireguard](https://github.com/utilitywarehouse/semaphore-wireguard) is a WireGuard peer manager
that is meant to run as a daemonset in a Kubernetes cluster. It has the
following tasks to manage:
- Create WireGuard devices on the local host and store the public keys and
  WireGuard endpoints in local node's Kubernetes annotations
- Watch nodes from remote clusters and update WireGuard devices peers.
- Make sure there is a route for each remote cluster pod IP range via the
  respective WireGuard interface.

As briefly mentioned above the following limitation applies here:
- Semaphore-Wireguard currently discovers the remote pod subnets for each node
  based on the node's `spec.PodCIDR`, thus it can only work in environments
  where hostLocal ipam is used to assign addresses to pods based on that range.

### How it works

Semaphore-wireguard expects a json configuration file, where we pass a name for
the local cluster and a list of remote clusters, plus the needed configuration
to talk to their apiservers. A more detailed list of all the configurable
varables for semaphore-wireguard can be found
[here](https://github.com/utilitywarehouse/semaphore-wireguard#config)

In the following example, let's assume two clusters, one deployed in AWS and
one deployed in GCP, with only one node each. More in detail:

AWS:
- pod IP net: 10.2.0.0/16
- 1 node with ip address 10.66.23.31 and local pod subnet 10.2.3.0/24

GCP:
- pod IP net: 10.4.0.0/16
- 1 node with ip address 10.22.22.27 and local pod subnet 10.4.7.0/24

Running semaphore-wireguard on AWS will create a new WireGuard device named:
`wireguard.gcp` to peer with the discovered GCP peers, and generate a new key
pair. Also, it will configure a static route for 10.4.0.0/16 on the host via
`wireguard.gcp`.
```
$ ip route show 10.4.0.0/16
10.4.0.0/16 dev wireguard.gcp scope link
```

Then it will expose the public key and endpoint to talk to the local WireGuard
instance under the host node's Kubernetes annotations using the following keys:
- `gcp.wireguard.semaphore.uw.io/pubKey`
- `gcp.wireguard.semaphore.uw.io/endpoint`
The `gcp` prefix is for watcher to be able to identify which annotations are
meant for the cluster it belongs and talk to the correct WireGuard device.
Following that, the agent will start watching all GCP nodes, expecting that the
semaphore-wireguard agents are working there, and looking for the following
annotations:
- `aws.wireguard.semaphore.uw.io/pubKey`
- `aws.wireguard.semaphore.uw.io/endpoint`
in order to configure local peers.

If everything described above works as intended the local WireGuard device will
look like:
```
$ wg show wireguard.gcp
interface: wireguard.gcp
  public key: 20+DPsIsMA79+kkXNG1Raoe0hi0bnRf+xUJWhBtB6h8=
  private key: (hidden)
  listening port: 51821

peer: ZIaHH+/lKD6Qd4cGGQnQcBWBKkasTnRUJ7m30oCgcAA=
  endpoint: 10.22.22.27:51821
  allowed ips: 10.4.7.0/24
  latest handshake: 30 seconds ago
  transfer: 1.72 MiB received, 523.89 KiB sent
  persistent keepalive: every 25 seconds

```

Here is a detailed sketch of what was described above:
![node-to-node]({{site.url}}/blog/png/semaphore-wireguard-node-to-node.png){:class="img-responsive"}

Expanding the above scenario, here is how a full WireGuard mesh between 3
clusters running semaphore-wireguard would look like:
![full-mesh]({{site.url}}/blog/png/semaphore-wireguard-full-mesh-example.png){:class="img-responsive"}

## Semaphore-Service-Mirror

[Semaphore-Service-Mirror](https://github.com/utilitywarehouse/semaphore-service-mirror) is an controller responsible for mirroring services
from one Kubernetes cluster to another. We describe service mirroring as:
- Creating a local Kubernetes Cluster-IP service
- Adding a custom local Endpoints resource for the above service which includes
  the addresses from the mirrored service endpoints.
The controller mirrros services based on a label. Since Kubernetes allows
defining Endpoints for a Service resource which are outside of the local
cluster's pod subnet, the above approach will result in a local service with
endpoints on remote pod IP addresses.

### How it works
Let's consider the following Kubernetes clusters again:

AWS:
- pod IP net: 10.2.0.0/16
- services IP range: 10.3.0.0/16

GCP:
- pod IP net: 10.4.0.0/16
- services IP range: 10.5.0.0/16

Now assuming that we have a Service resource in `AWS` cluster as:
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

The controller running on `GCP` will mirror that service under semaphore
namespace (`sys-semaphore` in our example) generating a name following the
pattern:
`<cluster>-<namespace>-73736d-<service-name>`, where the separator comes from
```
echo -n ssm | xxd -p
73736d
```
(ssm=semaphore-service-mirror) and is a "hack" to allow more flexibility
regarding namespace and service names, without risking collisions.

The result in `GCP` cluster will look like:
```
kubectl --context=gcp --namespace=sys-semaphore get service | grep fluentd
aws-sys-log-73736d-fluentd   ClusterIP   10.5.184.192   <none>        8888/TCP,8889/TCP   25d
```
```
kubectl --context=gcp --namespace=sys-semaphore get endpoints | grep fluentd
aws-sys-log-73736d-fluentd   10.2.3.19:8889,10.2.4.19:8889,10.2.7.18:8889 + 3 more...   17d
```

So, the end result is a Kubernetes Service with endpoints that point to pod IPs
from a remote cluster:
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

### The DNS trick to resolve
In order for the above service to be accessed via a rational DNS address, we
use CoreDNS `rewrite` plugin as follows:
```
    rewrite continue {
      name regex ([\w-]*\.)?([\w-]*)\.([\w-]*)\.svc\.cluster\.aws {1}{4}-{3}-73736d-{2}.sys-semaphore.svc.cluster.local
      answer name ([\w-]*\.)?aws-([\w-]*)-73736d-([\w-]*)\.sys-semaphore\.svc\.cluster\.local {1}{4}.{3}.svc.cluster.{2}
    }
```
that would match a query for:
`fluentd.sys-log.svc.cluster.aws` and rewrite it to look for
-> `aws-sys-log-73736d-fluentd.sys-semaphore.svc.cluster.local`

Finally, the `answer name` section will rewrite the DNS answer back for the
original address in question.
As a result our queries for the remote cluster will resolve to the mirrored
service, for example:
```
# drill fluentd.sys-log.svc.cluster.aws
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 51067
;; flags: qr aa rd ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; fluentd.sys-log.svc.cluster.aws.     IN      A

;; ANSWER SECTION:
fluentd.sys-log.svc.cluster.aws.        5       IN      A       10.5.184.192
```

A full CoreDNS configuration example for the above `GCP` cluster would look
like:
```
cluster.aws {
    errors
    health
    rewrite continue {
      name regex ([\w-]*\.)?([\w-]*)\.([\w-]*)\.svc\.cluster\.(aws|gcp) {1}{4}-{3}-73736d-{2}.sys-semaphore.svc.cluster.local
      answer name ([\w-]*\.)?(aws|gcp)-([\w-]*)-73736d-([\w-]*)\.sys-semaphore\.svc\.cluster\.local {1}{4}.{3}.svc.cluster.{2}
    }
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      endpoint_pod_names
      upstream
      fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
.:53 {
    errors
    health
    kubernetes cluster.local cluster.gcp in-addr.arpa ip6.arpa {
      pods insecure
      endpoint_pod_names
      upstream
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### EndpointSlice Mirroring

The above would also work with EndpointSlices thanks to the Kubernetes
[EndpointSliceMirroring controller](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/#endpointslice-mirroring).
The controller will mirror the Endpoints copied over from a remote cluster and
make sure that they are depicted in EndpointSlices.

## Semaphore-Policy

The last component of this framework is our policy controller, called
[semaphore-policy](https://github.com/utilitywarehouse/semaphore-policy).
This component has a hard dependency on Calico CNI being deployed on all the
clusters, as it uses the related CRDs to perform the task at hand.
The main idea is watching Pod resources in a remote cluster and creating sets
of remote IPs under GlobalNetworkSet resources. Then these can be used with
Calico Network Policies to allow traffic from the IPs in a particular set.

### How it works

The policy operator performs the following tasks:
- Watch for pods labelled with `policy.semaphore.uw.io/name` on a remote cluster
  and extract a name from the label.
- Generate a name for the local resource to be created based on the above name,
  the pods namespace and the cluster name. That will derive from the pattern:
  `<cluster>-<namespace>-<name>`
- Create a Calico GlobalNetworkSet resource in the local cluster and add the
  remote pod's IP as member of the addresses set. If the set already exists then
  update the addresses.

As a result, for each remote deployment's pods which we chose to label, we will
have a local GlobalNetworkSet resource that will depict the IP addresses of the
deployment members. We can then use the generated sets in local Calico Network
Policies in order to allow access from the remote deployments. One important
thing to note here is that the controller is creating GlobalNetworkSets, which
are cluster scoped resources, in order to be agnostic of the local cluster
namespaces. Then users can use `namespaceSelector: global()` attribute in their
local Network Policies to refer to the global sets.

Let's consider the following example in our `GCP` cluster:
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
Which shows a daemonset named `forwarder` under `sys-log` namespace,  where all
the pods are labelled as: `policy.semaphore.uw.io/name=forwarder`.
That will trigger the aws controller to create the respective GlobalNetworkSet
as described above:
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

That is now a set that presents the remote deployments IP addresses and can be
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
`fluentd` deployment.
