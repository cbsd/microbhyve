# Microbhyve: generate a bootable working minimal installation of FreeBSD for bhyve VM

## Introduction

This article describes how to generate a minimalistic image of the (FreeBSD OS)[https://www.freebsd.org] (architecture: amd64) for the (bhyve)[https://en.wikipedia.org/wiki/Bhyve] virtual machine using the `jail2iso` script of the (CBSD project)[https://github.com/cbsd/cbsd] as an example. As a result of these works, we get a **12MB** distribution kit and a working network stack with the ability to remotely access via SSH, as well as a set of elementary Unix utilities. The consumption of RAM by such an installation in multi-user mode does not exceed **80MB**:

