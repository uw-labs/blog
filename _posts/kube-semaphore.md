---
layout: post
title: "Kubernetes Semaphore: A simple framework to automate cross cluster communication"
date: ""
categories: kubernetes
---

# Kubernetes Semaphore

This is a collection of components we deploy under the same namespace we called
`semaphore`, in order to support communication between deployments in different
kubernetes clusters.

## The Challenge

`Semaphore` components try to address the following concerns we faced on pod
to pod communication between pods which live in different clusters:

- Encryption on traffic between kubernetes clusters.
- Exposing kubernetes services from one cluster to the other to avoid going
  through external load balancers.
- Network policies for traffic coming from another cluster.

For each one of the above, we developed an independent app called respectively:
- semaphore-wireguard
- semaphore-service-mirror
- semaphore-policy

## Limitations/Requirements

- The aim of the project is not to provide networking between nodes in different
  clusters or providers. It strictly tries to tackle the above mentioned
  challenges and assumes that node to node network connectivity is something
  already achieved.
- We use calico as a cni solution in our clusters and we rely on calico CRDs in
  order to implement cross cluster network policies. Thus running calico in all
  clusters is a requirement for this to work.
- Nodes' pod subnets should be assigned using kubernetes node `spec.PodCIDR`
  along with the hostLocal ipam.

## Semaphore-Wireguard

[Semaphore-Wireguard]( wireguard peer manager th) is a wireguard peer manager that is meant to run as a
daemonset in a kubernetes cluster. It has the following tasks to manage:
- Create wireguard devices on the local host and store the public keys and
  wireguard endpoints in local node's kubernetes annotations
- Watch nodes from remote clusters and update wireguard devices peers.
- Make sure there is a route for each remote cluster pod ip range via the
  respective wireguard interface.

As briefly mentioned above the following limitation applies here:
- Semaphore-Wireguard currently discovers the remove pod subnets for each node
  based on the node's `spec.PodCIDR`, thus it can only work in environments
  where hostLocal ipam is used to assign addresses to pods based on that range.

### How it works

Semaphore-wireguard expects a json configuration file, where we pass a name for
the local cluster and a list of remote clusters, plus the needed configuration
to talk to their apiservers. A more detailed list of all the configurable
varables for semaphore-wireguard can be found [here](https://github.com/utilitywarehouse/semaphore-wireguard#config)

For the following, let's assume 2 clusters, one deployed in aws and one deployed
in gcp, with only one node each. More in detail:

aws:
- pod ip net: 10.2.0.0/16
- 1 node with ip address 10.66.23.31 and local pod subnet 10.2.3.0/24
gcp:
- pod ip net: 10.4.0.0/16
- 1 node with ip address 10.22.22.27 and local pod subnet 10.4.7.0/24

Running semaphore-wireguard on aws will create a new wireguard device named:
`wireguard.gcp` to peer with the discovered gcp peers, and generate a new key
pair. Also, it will configure a static route for 10.4.0.0/16 on the host via
`wireguard.gcp`.
```
$ ip route show 10.4.0.0/16
10.4.0.0/16 dev wireguard.gcp scope link
```

Then it will expose the public key and endpoint to talk to the local wireguard
instance under the host node's kubernetes annotations using the following keys:
- `gcp.wireguard.semaphore.uw.io/pubKey`
- `gcp.wireguard.semaphore.uw.io/endpoint`
The `gcp` prefix is for watcher to be able to identify which annotations are
meant for the cluster it belongs and talk to the correct wireguard device.
Following that, the agent will start watching all gcp nodes, expecting that the
semaphore-wireguard agents are working there, and looking for the following
annotations:
- `aws.wireguard.semaphore.uw.io/pubKey`
- `aws.wireguard.semaphore.uw.io/endpoint`
in order to configure local peers.

If everything described above works as intended the local wireguard device will
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

Expanding the above scenario, here is how a full wireguard mesh between 3
clusters running semaphore-wireguard would look like:
![full-mesh]({{site.url}}/blog/png/semaphore-wireguard-full-mesh-example.png){:class="img-responsive"}
