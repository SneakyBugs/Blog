---
title: "Terraform meets Ansible: Seamless inventory integration with the new certified collection"
date: 2024-08-21
description: Still templating your Ansible inventory for Terraform provisioned machines? In this blogpost we're taking a look at the new official integration from Red Hat.
avatar: tie-orange
images:
  ogPath: 1x1.png
  ldPaths:
    - 1x1.png
    - 4x3.png
    - 16x9.png
---

You probably know the drill.

Terraform is great for resource provisioning, and Ansible is great for configuration management.
A very common workflow is to combine both to automatically provision and configure virtual machines.
Terraform is used to provision virtual machines, and Ansible is then used to configure them.
However, for this to work you need to *define Ansible inventory based on Terraform outputs*.

If you go around searching for ways to populate Ansible inventory from Terraform, you'd mostly see solutions templating inventory files.
Templating and inventory scripts based solutions are outdated at this point.
Inventory plugins are the most recent and capable way to manage dynamic inventory in Ansible.

[In late 2022 Red Hat announced the *cloud.terraform* certified collection.](https://www.ansible.com/blog/walking-on-clouds-with-ansible)
Later on in 2023 Red Hat released [a Terraform provider for Ansible](https://registry.terraform.io/providers/ansible/ansible)
and [a matching inventory plugin](https://github.com/ansible-collections/cloud.terraform/blob/main/docs/cloud.terraform.terraform_provider_inventory.rst)
for defining inventory as Terraform resources.

I'm migrating from my custom inventory plugins to this new official inventory plugin and Terraform provider duo, and it felt like there's not enough information about it around.

## The instructions

In this guide we'll define Ansible inventory using Terraform, and configure Ansible to use the inventory resources.

This guide assumes you have Ansible 8.5 or later, and Terraform 1.6 or later installed.
It assumes your repository has a `terraform` directory containing a Terraform module, and a `playbook` directory containing an Ansible playbook.

Add the `ansible/ansible` Terraform provider to the provider requirements:

```hcl
# Inside terraform/versions.tf
terraform {
  required_providers {
    ansible = {
      source  = "ansible/ansible"
      version = "1.3.0"
    }
  }
}
```

Initialize Terraform to download the provider:

```
terraform init
```

Create Ansible inventory group and host resources:

```hcl
# Inside terraform/main.tf
resource "ansible_group" "cluster" {
  name     = "k8s_cluster"
  children = ["kube_control_plane", "kube_node"]
}

resource "ansible_host" "debug" {
  name   = "rocky-debug"
  groups = ["kube_control_plane", "kube_node", "etcd"]
  variables = {
    ansible_host = libvirt_domain.debug.network_interface[0].addresses[0]
  }
}
```

Apply the module to create the inventory resources:

```
terraform apply
```

Moving over to the `playbook` directory, add the `cloud.terraform` collection requirement:

```yml
# Inside playbook/requirements.yml
collections:
  - name: cloud.terraform
    version: 3.0.0
```

Install collection requirements:

```
ansible-galaxy collection install --requirements requirements.yml
```

Create an inventory configuration file:

```yml
# Inside playbook/inventory/nodes/hosts.yml
plugin: cloud.terraform.terraform_provider
project_path: ../terraform
```

Verify the Ansible inventory is actually present:

```
ansible-inventory -i inventory/nodes/hosts.yml --graph
```

Output should look like this:

```
@all:
  |--@ungrouped:
  |--@k8s_cluster:
  |  |--@kube_control_plane:
  |  |  |--rocky-debug
  |  |--@kube_node:
  |  |  |--rocky-debug
  |--@etcd:
  |  |--rocky-debug
```

## Run it

Only thing left is to run your playbook.

You now know how to use the latest and greatest Terraform integration Ansible has to offer.
How will you be using it?
