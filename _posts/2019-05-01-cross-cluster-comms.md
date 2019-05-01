---
layout: post
title: "Dynamic Cross Cluster Route Propagation using GoBGP"
date: "2019-05-01 11:32:02 +0100"
categories: kubernetes
---

# Dynamic Cross Cluster Route Propagation using GoBGP

## Topology

This assumes 2 kubernetes clusters with flat network, so that host in one network can reach host in the other. In particular we have:


- 1 cluster deployed in aws europe-west-1 zone with nodes in cidr 10.66.21.0/24 and pod cidr 10.2.0.0/16
- 1 cluster deployed in gcp europe-west2 zone with nodes in cidr 10.22.20.0/24 and pod cidr 10.4.0.0/16
- Both clusters running calico version > 3.2.0 for cni (to be able to set felix `ExternalNodesCIDRList` variable and allow remote traffic) configured to use `bgp` to exchange local routes and ipip enabled for pod to pod communication.

Flat network is implemented via `interconnects` using Megaport, and firewall rules in both sides are in place so that:

```
core@ip-10-66-21-57 ~ $ ping 10.22.20.82
PING 10.22.20.82 (10.22.20.82) 56(84) bytes of data.
64 bytes from 10.22.20.82: icmp_seq=1 ttl=62 time=25.0 ms
64 bytes from 10.22.20.82: icmp_seq=2 ttl=62 time=23.7 ms
64 bytes from 10.22.20.82: icmp_seq=3 ttl=62 time=23.6 ms
64 bytes from 10.22.20.82: icmp_seq=4 ttl=62 time=33.9 ms
```

## Deploy GoBGP instances as Route Reflectors

Adding a bgp route reflector will give us the following example network topology:

![bgp-simple](/png/bgp_simple.png){:class="img-responsive"}

In each cluster we deploy 3 `gobgp` instances (one per availability zone for redundancy) and we use static ips for each one of them. 

We can do that with a simple systemd service:
```
Description=GoBGP Router
Requires=docker.socket
After=docker.socket
[Service]
Environment=GOBGP_VERSION=v2.1.0
ExecStartPre=-/bin/sh -c 'docker pull quay.io/utilitywarehouse/sys-gobgp:$GOBGP_VERSION'
ExecStartPre=-/bin/sh -c 'docker rm "$(docker ps -q --filter=name=gobgp_router)"'
ExecStart=/bin/sh -c '\
                docker run --rm \
                -p 179:179 \
                -v /etc/gobgp:/etc/gobgp \
                --name=gobgp_router \
                quay.io/utilitywarehouse/sys-gobgp:$GOBGP_VERSION gobgpd -f /etc/gobgp/gobgp.conf'
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

We also need to allow all the cluster nodes to communicate with the `gobgp` instances (protocol tcp, port 179 on both ends)

Configuration lives under `/etc/gobgp/gobgp.conf`. `GoBGP` allows dynamic peers from within a range and will use a passive approach for contacting them (waits for connection to be initiated from the peer). So a simple configuration to accept incoming bgp connections from the kube cluster nodes will be:

```
[global.config]
  as = 64513
  router-id = "10.22.20.6"

[[peer-groups]]
  [peer-groups.config]
    peer-group-name = "k8s-bgp-clients"
    peer-as = 64513
  [peer-groups.graceful-restart.config]
    enabled = true
  [peer-groups.route-reflector.config]
    route-reflector-client = true
    route-reflector-cluster-id = "10.22.20.6"
[[dynamic-neighbors]]
  [dynamic-neighbors.config]
    prefix = "10.22.20.0/24"
    peer-group = "k8s-bgp-clients"
```

* We have enabled graceful-restart to keep routes from nodes while calico bird instances might need to restart.

That will run a bgp router with id `10.22.20.6,` which in that case is the same as the static ip we assigned to the node, and will allow iBGP connections from peers in `10.22.22.0/24` using AS `64513`.

Now we need to point calico to the new static bgp routers that we created and configure it to use those for route advertisement and discovery instead of a full mesh.

Disable mesh and set asNumber:


```
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Error
  nodeToNodeMeshEnabled: false
  asNumber: 64513
```

Point to a static global peer:

```
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: bgppeer-global-6
spec:
  peerIP: 10.22.20.6
  asNumber: 64513
```

As a result calico bird instances will contact the new bgp instances. From a `gobgp` node we can see the cluster peers:

```
$ gobgp neighbor
Peer           AS      Up/Down State       |#Received  Accepted
10.22.20.79 64513 34d 09:36:30 Establ      |        1         1
10.22.20.86 64513 34d 13:27:10 Establ      |        1         1
10.22.20.80 64513 34d 13:35:02 Establ      |        1         1
10.22.20.83 64513 34d 09:40:40 Establ      |        1         1
10.22.20.92 64513 18d 17:18:44 Establ      |        1         1
```

And the routes:

```
gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.4.0.0/24          10.22.20.90                               34d 13:23:24 [{Origin: i} {LocalPref: 100}]
*> 10.4.1.0/24          10.22.20.93                               18d 17:20:13 [{Origin: i} {LocalPref: 100}]
...
```

We can also do the same via bird instances running in calico-node containers:

```
kubectl --context=dev-aws --namespace=kube-system exec -ti calico-node-2d89s -- birdcl -s /var/run/calico/bird.ctl show protocols
```

```
kubectl --context=dev-aws --namespace=kube-system exec -ti calico-node-2d89s -- birdcl -s /var/run/calico/bird.ctl show route
```

And finally routes are in each node’s routing table: `ip route` 

After that our clusters should look like:


Aws:
 
- Hosts cidr 10.66.21.0/24
- Pod cidr 10.2.0.0/16
- GoBGP RRs 10.66.21.6, 10.66.21.69, 10.66.21.133
- iBGP as number 64512

Gcp:
 
- Hosts cidr 10.22.20.0/24
- Pod cidr 10.4.0.0/16
- GoBGP RRs 10.22.20.6, 10.22.20.69, 10.22.20.133
- iBGP as number 64513

* The network IP addresses should be considered arbitrary and are just picked to satisfy restrictions/concepts in our local network setups.

* Because we use dynamic peer discovery creating and terminating nodes will make gobgp automatically pick up changes and route propagation will still work fine. So we can scale the cluster up and down without the need to alter configuration.

## GoBGP External Peering for Cross Cluster Route Propagation

Now that we have 2 different autonomous bgp clusters we can pair them by establishing eBGP connections between the static gobgp instances of each side.
In order to do that and as we do not actually want to route any packets between hosts (remember we have a flat network and host reachability) we need to use the route server feature (https://github.com/osrg/gobgp/blob/master/docs/sources/route-server.md).

![bgp-multi](/png/bgp_multi.png){:class="img-responsive"}

For that purpose we add configuration on all gobgp instances to inform about the respective ones on the other cluster and filters to allow advertisement only of the pod networks.

An example configuration will look like:

```

[global.config]
  as = 64513
  router-id = "10.22.20.6"

[[peer-groups]]
  [peer-groups.config]
    peer-group-name = "k8s-bgp-clients"
    peer-as = 64513
  [peer-groups.graceful-restart.config]
    enabled = true
  [peer-groups.route-reflector.config]
    route-reflector-client = true
    route-reflector-cluster-id = "10.22.20.6"
  [peer-groups.route-server.config]
    route-server-client = true
[[dynamic-neighbors]]
  [dynamic-neighbors.config]
    prefix = "10.22.20.0/24"
    peer-group = "k8s-bgp-clients"

[[peer-groups]]
  [peer-groups.config]
    peer-group-name = "aws-bgp-clients"
    peer-as = 64512
  # https://github.com/osrg/gobgp/issues/1932
  [peer-groups.timers.config]
    connect-retry = 5
  [peer-groups.apply-policy.config]
    import-policy-list = ["import-from-aws"]
    default-import-policy = "reject-route"
    export-policy-list = ["export-from-gcp"]
    default-export-policy = "reject-route"
  [peer-groups.route-server.config]
    route-server-client = true

[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.66.21.6"
    peer-group = "aws-bgp-clients"
  [neighbors.ebgp-multihop.config]
    enabled = true

[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.66.21.69"
    peer-group = "aws-bgp-clients"
  [neighbors.ebgp-multihop.config]
    enabled = true

[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.66.21.133"
    peer-group = "aws-bgp-clients"
  [neighbors.ebgp-multihop.config]
    enabled = true

[[defined-sets.neighbor-sets]]
  neighbor-set-name = "aws-neighbors"
  neighbor-info-list = [
    "10.66.21.6",
    "10.66.21.69",
    "10.66.21.133"
  ]

[[defined-sets.prefix-sets]]
  prefix-set-name = "aws-pod-subnet"
  [[defined-sets.prefix-sets.prefix-list]]
    ip-prefix = "10.2.0.0/16"
    masklength-range = "24..32"

[[defined-sets.prefix-sets]]
  prefix-set-name = "gcp-pod-subnet"
  [[defined-sets.prefix-sets.prefix-list]]
    ip-prefix = "10.4.0.0/16"
    masklength-range = "24..32"

[[policy-definitions]]
  name = "import-from-aws"
  [[policy-definitions.statements]]
    name = "import-from-aws-stmt"
    [policy-definitions.statements.conditions.match-prefix-set]
      prefix-set = "aws-pod-subnet"
    [policy-definitions.statements.conditions.match-neighbor-set]
      neighbor-set = "aws-neighbors"
    [policy-definitions.statements.actions]
      route-disposition = "accept-route"

[[policy-definitions]]
  name = "export-from-gcp"
  [[policy-definitions.statements]]
    name = "export-from-gcp-stmt"
    [policy-definitions.statements.conditions.match-prefix-set]
      prefix-set = "gcp-pod-subnet"
    [policy-definitions.statements.conditions.match-neighbor-set]
      neighbor-set = "aws-neighbors"
    [policy-definitions.statements.actions]
      route-disposition = "accept-route"
```

Notes on the above `gobgp` configuration example:

- cluster peers have both `route-reflector` and `route-server` functionality enabled
- `gobgp` peers have `route-server` functionality enabled
- filters allow `gobgp` peers to only advertise addresses from the pod range that belongs to their cluster and accept routes from the remote cluster's pod range

## Configure Calico to Allow External Routes

Also we need to make calico aware of the pod cidr of the other cluster. To do that we need a new pool:
```
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: dev-gcp-pods
spec:
  cidr: 10.4.0.0/16
  ipipMode: CrossSubnet
  disabled: true
```
The disabled label is needed to prevent calico from using this pool to assign addresses to pods and the `ipip` is the encapsulation that we need for the pod to pod communication.

* Firewall rules need to be in place to allow ipip (4) from 10.66.21.0/24 to 10.22.20.0/24 and vice versa.

Also need to configure calico-node to allow routes that will go outside the nodes of its own cluster that already knows about.

```
     env:
           - name: FELIX_EXTERNALNODESCIDRLIST
              value: '10.22.20.0/24'
```

That will add the range in the ipset that calico uses to keep track of known nodes.

```
kubectl --context=dev-aws --namespace=kube-system exec calico-node-2d89s -- ipset list cali40all-hosts-net
Name: cali40all-hosts-net
Type: hash:net
Revision: 6
Header: family inet hashsize 1024 maxelem 1048576
Size in memory: 2840
References: 2
Number of entries: 39
Members:
10.66.21.58
10.66.21.104
10.66.21.157
10.22.20.0/24

```

Now we can see the new static neighbors from the remote cluster:

```
gobgp neighbor
Peer            AS      Up/Down State       |#Received  Accepted
10.22.20.88  64513 14d 21:33:51 Establ      |        1         1
10.22.20.78  64513 14d 21:41:25 Establ      |        1         1
10.22.20.81  64513 14d 21:34:01 Establ      |        1         1
10.22.20.84  64513 14d 21:34:00 Establ      |        1         1
10.22.20.79  64513 14d 21:42:07 Establ      |        1         1
10.66.21.6   64512  4d 17:35:55 Establ      |       38        38
10.66.21.69  64512 35d 05:01:47 Establ      |       38        38
10.66.21.133 64512 34d 05:42:16 Establ      |       38        38
```

And for  each neighbor we get a local rib with all the routes:

```
 gobgp neighbor 10.22.20.78 local
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.2.0.0/24          10.66.21.52                               15d 02:29:21 [{Origin: i} {LocalPref: 100}]
*  10.2.0.0/24          10.66.21.52                               15d 02:29:21 [{Origin: i} {LocalPref: 100}]
*  10.2.0.0/24          10.66.21.52                               4d 17:37:09 [{Origin: i} {LocalPref: 100}]
*> 10.2.1.0/24          10.66.21.104                              10d 07:02:57 [{Origin: i} {LocalPref: 100}]
*  10.2.1.0/24          10.66.21.104                              10d 07:02:57 [{Origin: i} {LocalPref: 100}]
*  10.2.1.0/24          10.66.21.104                              4d 17:37:09 [{Origin: i} {LocalPref: 100}]
*> 10.2.2.0/24          10.66.21.8                                15d 02:39:51 [{Origin: i} {LocalPref: 100}]
*  10.2.2.0/24          10.66.21.8                                15d 02:39:51 [{Origin: i} {LocalPref: 100}]
*  10.2.2.0/24          10.66.21.8                                4d 17:37:09 [{Origin: i} {LocalPref: 100}]
…
*> 10.4.3.0/24          10.22.20.77                               14d 21:42:40 [{Origin: i} {LocalPref: 100}]
*> 10.4.4.0/24          10.22.20.79                               14d 21:43:20 [{Origin: i} {LocalPref: 100}]
*> 10.4.5.0/24          10.22.20.81                               14d 21:35:14 [{Origin: i} {LocalPref: 100}]
*> 10.4.6.0/24          10.22.20.82                               14d 21:35:08 [{Origin: i} {LocalPref: 100}]
*> 10.4.7.0/24          10.22.20.83                               14d 21:35:09 [{Origin: i} {LocalPref: 100}]
*> 10.4.9.0/24          10.22.20.85                               14d 21:35:08 [{Origin: i} {LocalPref: 100}]
*> 10.4.11.0/24         10.22.20.84                               14d 21:35:14 [{Origin: i} {LocalPref: 100}]

```

Here we have 3 identical routes for each route that was advertised from the remote cluster because of the 3 gobgp instances that we are pairing with. We can again check that the routes are propagated down to the `bird` peers:

```
kubectl --context=dev-aws --namespace=kube-system exec -ti calico-node-2d89s -- birdcl -s /var/run/calico/bird.ctl show route
```


And the hosts of the remote cluster:

```
ip route
```

## Cross Cluster Service Advertisement

Since calico v3.4.0 it is possible to advertise service addresses. That can be done by setting `CALICO_ADVERTISE_CLUSTER_IPS` environment variable to our kubernetes services cidr (https://docs.projectcalico.org/v3.4/usage/service-advertisement). We will also need to configure bgp filters to allow receiving and advertising routes inside services ranges (same as we did above for pod ranges). For classic services that will result in all the nodes in the cluster advertising that they could route the whole range (here we use `10.3.0.0/16` in one cluster and `10.5.0.0/16` in the other). Since a bgp client will only pick the shortest route and advertise only this one, gobgp instances will pick a node and advertise it on the remote cluster and that node will act as a Gateway for accessing services from the remote cluster. We don’t really care that the 3 gobgp instances might pick a different node to advertise (as all the routes have the same weight) since on each remote node only one node will be available for routing `10.3.0.0/16` after all.

In order for that to work `kube-proxy` `--cluster-cidr` flag must be set:
> The CIDR range of pods in the cluster. When configured, traffic sent to a Service cluster IP from outside this range will be masqueraded and traffic sent from pods to an external LoadBalancer IP will be directed to the respective cluster IP instead

So that traffic that arrive to the node that acts as services gateway can be masquaraded and redirected to the proper destination.

Because we have graceful restart enabled on all peers:

```
  [peer-groups.graceful-restart.config]
    enabled = true
```

which is needed in order to avoid `calico-node` pod restarts causing blinks on the network, that introduces a problem. If the node that acts as a gateway for services dies, it will still remain advertised for as long as the `bird` instances asked as a graceful restart period (by default 2mins) and all requests to services will fail.
That may be solved by using long lived graceful restart bgp feature (https://github.com/osrg/gobgp/blob/master/docs/sources/graceful-restart.md#long-lived-graceful-restart) 

This feature will still allow the route to the failing node to be in the rib, but since the node is not responding the route will be marked as stale and get de-selected as the prefered one to be advertised downstream.

Sadly long lived graceful restart is not available through calico-node bird instance at the moment and the above setup will be possible after this: https://github.com/projectcalico/calico/issues/2470#issuecomment-483370379 is resolved.

## Limitations - Network Policies

Obviously in order to allow cross cluster pod to pod communication you will need to tune network policies (in case they are used) to allow the whole pod cidr of the other cluster talk to your pod, since the local calico deployment cannot know anything about the remote cluster. That means that you open your pod to the other cluster and access limiting becomes now a work that should be handled on pod level rather than network/iptables level (the way it is done with network policies).
