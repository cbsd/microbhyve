# Microbhyve: generate a bootable working minimal installation of FreeBSD for bhyve VM

## Introduction

This article describes how to generate a minimalistic image of the (FreeBSD OS)[https://www.freebsd.org] (architecture: amd64) for the (bhyve)[https://en.wikipedia.org/wiki/Bhyve] virtual machine using the `jail2iso` script of the (CBSD project)[https://github.com/cbsd/cbsd] as an example. As a result of these works, we get a **12MB** distribution kit and a working network stack with the ability to remotely access via SSH, as well as a set of elementary Unix utilities. The consumption of RAM by such an installation in multi-user mode does not exceed **80MB**:

```
rctl -hu process:20349 |grep memoryuse
memoryuse=78M
vmemoryuse=569M
```
![microtop](https://user-images.githubusercontent.com/926409/168559735-b07c9c92-afd9-4376-b841-dd8503a8804a.png)

Why is it needed and to whom:

- the script can be used as firmware when creating embedded products and services;
- research activities, studying how FreeBSD boot works;

For example, the initial microcontainer image with SSH access is used in the (MyBee project)[https://github.com/myb-project/guide] when creating a (Kubernetes)[https://kubernetes.io/] cluster. In this case, the (FreeBSD jail)[https://docs.freebsd.org/en/books/handbook/jails/] is created as a jump-host environment for accessing a Kubernetes cluster, which has utilities for working with K8S: `kubectl`, `helm`, `k9s`. Here we go a little further and use CBSD `jail2iso` script to get the VM image. At the same time, the minimalism of the kernel is achieved by eliminating all options and drivers that are not required when working as a VM. Basically, only the network stack and the virtio driver remain in the kernel, which allows you to get a **2MB** kernel.
