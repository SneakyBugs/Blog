---
title: 'Cluster API to production: from Cluster API to GitOps with Argo CD and Kyverno'
date: 2025-01-20
description: Discover practical patterns for managing Cluster API tenant clusters with GitOps. Step-by-step guide to implementing Argo CD for automated cluster configuration.
avatar: party-blue
series: capi-to-production
images:
  ogPath: 1x1.png
  ldPaths:
    - 1x1.png
    - 4x3.png
    - 16x9.png
---

Cluster API provisions barebones clusters, leaving us to decide how to install and manaage addons and workloads.
While there are solutions such as the
[Cluster API addon provider for Helm,](https://github.com/kubernetes-sigs/cluster-api-addon-provider-helm)
I prefer managing addons with Argo CD as it is much better
featured for maintaining the addons over time.

For Argo CD to deploy resources in tenant clusters we first need to configure the
clusters in Argo CD.
This guide goes over automatically generating Argo CD cluster credentials secrets
using Kyverno.
By the end of this guide, we will be able to deploy addons to Cluster API tenant
clusters with Argo CD from the management cluster.

Deploying addons with Argo CD from the management cluster has a significant advantage of
being able to manage both clusters and addons as resources on the management
cluster.
Enabling us to deploy fully configured clusters in a single `kubectl apply` or
`helm install` command, or with Argo CD.

While this article is targeted at Cluster API provider KubeVirt clusters, the steps
described will work for any provider with minimal modification.

## Why Argo CD cluster credentials need to be generated

Our goal is to deploy Argo CD applications targeting the tenant cluster.
For that to work, we must configure the tenant cluster credentials in Argo CD.
[Cluster credentials are configured in Argo CD via secrets with a special label.](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)

Problem is: we don't know the cluster credentials before applying the Cluster API
`Cluster` resource. Meaning we can't template the Argo CD cluster secret with Helm
(or other templating solution).

## Why Kyverno is the solution

There are a couple of Kubernetes controllers available that listen for Cluster API
clusters and create secrets for Argo CD.
[The Capi2Argo operator](https://github.com/dntosas/capi2argo-cluster-operator)
seems to be the most popular.

However, I chose Kyverno to do the Argo CD cluster secret generation.
Primarily because Kyverno is useful for other cluster management tasks.
Running controllers just for Argo CD secret generation seems redundant when
Kyverno has wider use.

## Kyverno policy for generating Argo CD cluster credentials

This guide assumes you have Argo CD, Kyverno, and Cluster API with Cluster API
provider KubeVirt installed on the management cluster.
The policy will work with other Cluster API providers with minimal modification.

Using the following Kyverno policy we can automatically create Argo CD cluster
credentials from kubeconfig secrets created by Cluster API:

```yml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: argo-cluster-generation-from-capi
spec:
  rules:
    - name: source-capi-secret
      match:
        all:
          - resources:
              kinds:
                - v1/Secret
              names:
                - "*-kubeconfig"
              selector:
                matchLabels:
                  cluster.x-k8s.io/cluster-name: "*"
      context:
        - name: clusterName
          variable:
            jmesPath: 'request.object.metadata.labels."cluster.x-k8s.io/cluster-name"'
        - name: kubeconfig
          variable:
            jmesPath: "request.object.data.value | base64_decode(@) | parse_yaml(@)"
        - name: dataConfig
          variable:
            value: |
              {
                "tlsClientConfig": {
                  "caData": "{{ kubeconfig.clusters[0].cluster."certificate-authority-data" }}",
                  "certData": "{{ kubeconfig.users[0].user."client-certificate-data" }}",
                  "keyData": "{{ kubeconfig.users[0].user."client-key-data" }}"
                }
              }
      generate:
        synchronize: true
        apiVersion: v1
        kind: Secret
        namespace: argocd
        name: "{{ request.object.metadata.namespace }}-{{ clusterName }}-cluster"
        data:
          metadata:
            labels:
              argocd.argoproj.io/secret-type: cluster
          type: Opaque
          stringData:
            name: "{{ request.object.metadata.namespace }}/{{ clusterName }}"
            server: "{{ kubeconfig.clusters[0].cluster.server }}"
            config: "{{ dataConfig }}"
```

Let's go over it step-by-step:
the policy starts by matching `Secret` resources created by Cluster API containing the
kubeconfig of tenant clusters:

```yaml
spec:
  rules:
    - name: source-capi-cluster-and-secret
      match:
        all:
          - resources:
              kinds:
                - v1/Secret
              names:
                - "*-kubeconfig"
              selector:
                matchLabels:
                  cluster.x-k8s.io/cluster-name: "*"
```

Why not match the `Cluster` and look up the kubeconfig secret?
When creating a `Cluster` resource, the kubeconfig secret is not created
immediately, causing the policy to fail until the kubeconfig secret is created.
Matching the kubeconfig secret directly avoids this failure and simplifies the
policy.

The policy continues by decoding and parsing the tenant cluster's kubeconfig from the
secret, and templating
[the config Argo CD requires for cluster credentials:](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)

```yaml
spec:
  rules:
    - name: source-capi-cluster-and-secret
      context:
        - name: clusterName
          variable:
            jmesPath: 'request.object.metadata.labels."cluster.x-k8s.io/cluster-name"'
        - name: kubeconfig
          variable:
            jmesPath: "request.object.data.value | base64_decode(@) | parse_yaml(@)"
        - name: dataConfig
          variable:
            value: |
              {
                "tlsClientConfig": {
                  "caData": "{{ kubeconfig.clusters[0].cluster."certificate-authority-data" }}",
                  "certData": "{{ kubeconfig.users[0].user."client-certificate-data" }}",
                  "keyData": "{{ kubeconfig.users[0].user."client-key-data" }}"
                }
              }              
```

Lastly, the policy generates a secret in Argo CD's namespace with the cluster credentials.
Note the cluster credentials secret's `name` property is set to `namespace/cluster-name`.

```yaml
spec:
  rules:
    - name: source-capi-cluster-and-secret
      generate:
        synchronize: true
        apiVersion: v1
        kind: Secret
        namespace: argocd
        name: "{{ request.object.metadata.namespace }}-{{ clusterName }}-cluster"
        data:
          metadata:
            labels:
              argocd.argoproj.io/secret-type: cluster
          type: Opaque
          stringData:
            name: "{{ request.object.metadata.namespace }}/{{ clusterName }}"
            server: "{{ kubeconfig.clusters[0].cluster.server }}"
            config: "{{ dataConfig }}"
```

## Deploying an Application in the tenant cluster

Argo CD applications can now target the tenant cluster.
Set the destination `name` to `namespace/cluster-name` where `namespace` is the
namespace of the cluster in the management cluster, and `cluster-name` is the name
of the cluster.

```yml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  # ...
  destination:
    name: namespace/cluster-name
    namespace: namespace-in-tenant
```

Note `spec.destination.namespace` refers to the namespace the application will be
installed **inside the tenant cluster**.

For a full example
[see the cluster chart from my infrastructure charts repository.](https://github.com/SneakyBugs/Helm-Charts/tree/main/charts/cluster)
Specifically [the argo-applications templates subdirectory](https://github.com/SneakyBugs/Helm-Charts/tree/main/charts/cluster/templates/argo-applications)
containing Argo CD application definitions for addons to be deployed on the tenant
cluster.

Another notable option for deploying addons that can be used after following this
guide is
[using Argo CD ApplicationSets with the cluster generator.](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)
It's out of scope for this guide, but deserves a honorable mention.
