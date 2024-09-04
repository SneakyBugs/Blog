---
title: "Level up your Packer templates: A best practices guide"
date: 2024-09-04
description: Looking to improve your Packer templates? This collection of tips will make your Packer templates faster, easier to use and easier to maintain.
avatar: baseball-blue
images:
  ogPath: 1x1.png
  ldPaths:
    - 1x1.png
    - 4x3.png
    - 16x9.png
---

Over the past few years I've been using Packer a lot.
When it works, it works.
But most of the time the experience is not very smooth.
Over time I made many mistakes, and slowly figured out how to create Packer templates that are easier to use and maintain.

I collected my experiences with Packer into practices that will help you learn from my mistakes and create Packer templates that are easier to use, test and maintain.

## Test the images

> Test images automatically.

Automated testing is hands down the best productivity boost I found for Packer.
Manually testing images is a long and tedious process.
Automated testing makes the development feedback loop much faster, making it much easier to maintain templates.

Testing Packer templates means building the template, cloning it and performing checks on the cloned virtual machine.
[Terratest has utilities for running Packer](https://pkg.go.dev/github.com/gruntwork-io/terratest@v0.47.1/modules/packer),
and I find it very useful because it also has utilities for running Terraform and running commands over SSH.

[Check out my Kubernetes image builder to see an example of testing with Terratest.](https://github.com/SneakyBugs/Kubernetes-Image-Builder/tree/main/test)

## Minimize the boot command

> Do as little as possible in the boot command.

Boot commands are slow and inconsistent.
Automating an interactive process with one-way communication tends to do that.

Avoid performing unnecessary actions in the boot command.
Only perform actions required for Packer to connect with SSH, everything else will be easier, faster and more consistent with provisioners.

[Alpine Linux cloud images are a great example](https://gitlab.alpinelinux.org/alpine/cloud/alpine-cloud-images/-/blob/main/alpine.pkr.hcl) of doing the bare minimum, connecting to SSH first and installing the OS with provisioners.

If possible, it is best to use a disk image of an OS rather than an installer `iso`.
Using a disk image removes the need for boot commands, and is much faster since you don't need to install the OS itself.

## Reduce usage of Variables

> Only use variables for what users need to change.

Only use variables for options users need to customize on a per-build basis.
Use locals to avoid repeating yourself if needed.
Use multiple `source` and `build` blocks for building images of multiple operating systems and versions.

Variables are the interface between the template and it's users.
Reducing the amount of variables makes templates easier to use, develop, and test.

The effects of using too many variables
[can be seen in the official Kubernetes image builder.](https://github.com/kubernetes-sigs/image-builder/tree/main/images/capi/packer/qemu)
It makes the template very hard to understand, and does not reduce code repetition much compared to multiple `source` and `build` blocks.

## Avoid Packer's HTTP server

> Avoid Packer's HTTP server if you plan on running in CI.

Using Packer's HTTP server limits you to run Packer only where you can run an HTTP server reachable by the virtual machine.
This is problematic for running Packer inside CI runners which are not configured to accept incoming HTTP traffic.
An example of working around this limitation
[can be seen in my Debian PVE template.](https://github.com/LKummer/packer-debian)
Where I had to work around it by uploading the needed files as GitLab pipeline artifacts in the `upload-preseed` job.

When not utilizing Packer's HTTP server, you can build templates everywhere.
No need for messy workarounds.

By avoiding the HTTP server you also avoid Windows related issues.
Specifically CRLF line endings causing issues when cloning on Windows and networking problems when running inside WSL.
