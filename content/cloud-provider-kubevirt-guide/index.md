---
title: 'Cluster API to production: implementing LoadBalancer services on Cluster API KubeVirt clusters'
date: 2025-01-09
description: Had trouble getting load balancer services working on Cluster API KubeVirt clusters? This guide will get you sorted out.
avatar: tie-blue
images:
  ogPath: 1x1.png
  ldPaths:
    - 1x1.png
    - 4x3.png
    - 16x9.png
---

This article is the beginning of a series on taking Cluster API managed clusters
on KubeVirt from where the documentation leaves you to fully functioning
production clusters.
Make sure to check out the next parts in the future.

Recently I've been moving my personal infrastructure to Kubernetes clusters
managed by Cluster API on KubeVirt.
After going through the Cluster API documentation, and getting my first clusters
up and running I encountered a problem: how do I get a working load balancer
implementation?

After going through the Kubernetes Slack I found the
[Cloud Provider KubeVirt](https://github.com/kubevirt/cloud-provider-kubevirt)
project. Yet it lacks documentation, and it is not clear how to get it working.
This post will explain how it works and how to set it up.

Since terminology with Cluster API is a bit confusing, this article will refer to
the cluster running KubeVirt and Cluster API as the management cluster, and to
clusters managed by Cluster API as tenant clusters.

## The problem with load balancer implementations

Using MetalLB, Kube VIP or other ARP/BGP based load balancer implementations does
not work inside KubeVirt Cluster API tenant clusters. Tenant Nodes are connected to the management cluster network without
direct access to the outside network, preventing load balancers from functioning.

## The solution to provisioning load balancer services

KubeVirt offers a cloud controller manager that provisions LoadBalancer type
services on the management cluster.

Cloud Provider KubeVirt runs on the management cluster and watches for Service resources
of type LoadBalancer in the tenant cluster.
When a Service of type LoadBalancer is created in the tenant cluster, Cloud Provider
KubeVirt creates a matching service of type LoadBalancer in the management cluster and
connects it to the service in the tenant via node ports.

## Installing the cloud controller manager

The Cluster API Provider KubeVirt project supplies templates with the cloud
controller manager. To use them specify `--flavor lb-kccm` when generating
manifests with `clusterctl`.
`kccm`, `passt-kccm` and `persistent-storage-kccm` template flavors are also
avaiable.
In this example we'll use the `lb-kccm` template.

Set environment variables configuring `clusterctl` manifest templating:

```sh
export NODE_VM_IMAGE_TEMPLATE='quay.io/capk/ubuntu-2204-container-disk:v1.30.1'
export CRI_PATH='/var/run/containerd/containerd.sock'
```

Generate manifests with `clusterctl`:

```sh
clusterctl generate cluster capi-quickstart --infrastructure kubevirt:v0.1.9 --flavor lb-kccm --kubernetes-version v1.30.1 --control-plane-machine-count 1 --worker-machine-count 1 > capi-quickstart.yml
```

Apply the manifests:

```sh
kubectl apply -f capi-quickstart.yml
```

Installing a network plugin, creating a Pod and LoadBalancer type service have
been omitted for brevity.

For a full example [see the cluster chart from my infrastructure charts
repository](https://github.com/SneakyBugs/Helm-Charts/tree/main/charts/cluster).
Specifically [the cloud-controller-manager templates subdirectory.](https://github.com/SneakyBugs/Helm-Charts/tree/main/charts/cluster/templates/cloud-controller-manager)
The chart includes installation of a network plugin and ingress exposed over a
LoadBalancer type service.
