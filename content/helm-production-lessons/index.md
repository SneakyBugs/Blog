---
title: "Helm in production: lessons and gotchas"
date: 2026-02-23
description: "Practical lessons from running Helm in production: CRD management, health checks, dry runs, schema validation, and OCI registries."
avatar: tie-orange
images:
  ogPath: 1x1.png
  ldPaths:
    - 1x1.png
    - 4x3.png
    - 16x9.png
---

While not always the most intuitive, Helm is the de-facto standard templating tool for Kubernetes manifests.
I collected my experience with Helm into lessons that will help you make better use of Helm.

## Manage CRDs yourself because Helm won't

[Helm only installs CRDs on initial chart installation and never updates, rolls back or deletes CRDs.](https://helm.sh/docs/topics/charts/#limitations-on-crds)
If you don't manage CRDs yourself you never get CRD updates.

It also means that a chart with CRDs is not idempotent with respect to version: upgrading from `1.0.0` to `1.1.0` produces a different result than installing `1.1.0` directly.

## Wait doesn't always wait

Up until Helm 4 the `--wait` flag only waited for Pods and hooks.
Since Helm 4 [the flag is using kstatus](https://github.com/kubernetes-sigs/cli-utils/blob/master/pkg/kstatus/README.md)
to check resource health, but even kstatus isn't guaranteed to work on custom resources.

If your chart deploys custom resources the `--wait` flag might not work as intended.
Helm may return successfully without any indication the application is not ready.

## Dry run is wetter than it sounds

When running `helm upgrade` with `--dry-run` and the existing version fails Helm's health check, the dry run upgrade also fails.

Avoid using `--dry-run` in PR/MR pipelines because it prevents merging if the existing installation is unhealthy.

## Validate your values

Charts can have a `values.schema.json` file with JSON schema for chart values validation.
The schema allows specifying additional restrictions for string value lengths, enum types, property names and more.

The schema can be loaded into `yaml-language-server` to provide schema validation when editing values files:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/cloudnative-pg/charts/refs/heads/main/charts/cluster/values.schema.json
backups: true # Incorrect type. Expected "object".
```

## OCI registries for everything

Helm has been moving to OCI registries instead of the traditional Helm repo format.

Charts can be uploaded to any OCI compatible registry, allowing you to use the same registry provider for your charts and container images.

With OCI registries you must specify the full image URL on install. This removes the need for `repo add`, and no more worries about getting old versions because you forgot to run `repo update`.
Prefer OCI over traditional Helm repos where possible.
