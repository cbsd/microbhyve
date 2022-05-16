# Microbhyve: generate a bootable working minimal installation of FreeBSD for bhyve VM

## Introduction

This article describes how to generate a minimalistic image of the [FreeBSD OS](https://www.freebsd.org) (architecture: amd64) for the [bhyve](https://en.wikipedia.org/wiki/Bhyve) virtual machine using the `jail2iso` script of the [CBSD project](https://github.com/cbsd/cbsd) as an example. As a result of these works, we get a **12MB** distribution kit and a working network stack with the ability to remotely access via SSH, as well as a set of elementary Unix utilities. The consumption of RAM by such an installation in multi-user mode does not exceed **80MB**:

```
rctl -hu process:20349 |grep memoryuse
memoryuse=78M
vmemoryuse=569M
```
![microtop](https://user-images.githubusercontent.com/926409/168559735-b07c9c92-afd9-4376-b841-dd8503a8804a.png)

Why is it needed and to whom:

- the script can be used as firmware when creating embedded products and services;
- research activities, studying how FreeBSD boot works;

For example, the initial microcontainer image with SSH access is used in the [MyBee project](https://github.com/myb-project/guide) when creating a [Kubernetes](https://kubernetes.io/) cluster. In this case, the [FreeBSD jail](https://docs.freebsd.org/en/books/handbook/jails/) is created as a jump-host environment for accessing a Kubernetes cluster, which has utilities for working with K8S: `kubectl`, `helm`, `k9s`. Here we go a little further and use CBSD `jail2iso` script to get the VM image. At the same time, the minimalism of the kernel is achieved by eliminating all options and drivers that are not required when working as a VM. Basically, only the network stack and the virtio driver remain in the kernel, which allows you to get a **2MB** kernel.

## Preparation for work

We assume that we are getting a "bhyve" (and jail) image of the same version of FreeBSD we are working on. For the purposes of this article, this is FreeBSD 13.1-RELEASE. If you are using a different version, use the appropriate numbering instead of **13.1**.

There are many ways to get a custom lightweight FreeBSD environment. One option is to use KNOBS (the *WITH/WITHOUT_* option in [src.conf](https://man.freebsd.org/src.conf/5), which can be used to control which components to include (or exclude) when building. Another option is to have a full installation of the base system and only copy the files we need. We will use the second option because it allows us to minimize the environment as much as possible - we will control each file.

To do this, create an empty CBSD container in rootfs called micro1 with RW rights:

> cbsd jcreate jname=micro1 baserw=1 ver=empty applytpl=0

Now in the *~cbsd/jails-data/micro1-data* directory (which is the rootfs for the created container), we need to create an environment hierarchy that will allow us to start the container, the SSH service and provide authorization/login. For the `jail2iso` script, CBSD has the `copy-binlib` script, which copies the necessary files by index and, if it is an executable file, libraries for it (search through the [ldd](https://man.freebsd.org/ldd/1) utility).

If you want to relax and get the environment as soon as possible, skip this chapter, as the work described below is already part of CBSD. For the even more impatient, this repository contains the script that you can just run.

Generating such an index is an exploratory activity: you can know by heart all the files involved and needed to boot FreeBSD (in which case you are an incredible FreeBSD hacker!), or, as the author of this article, find it in practice. To do this, you can use any file access tracking methods, such as DTRACE. But we will consider two other options. The first is to track access to files by the "atime" attribute of the file system. To do this, let's create a full-fledged CBSD container:

> cbsd jcreate jname=test1 runasap=1 baserw=1

Since we are working on ZFS, a separate named dataset is created for the container:

```
zfs list | grep test1
jails/test1    632M   550G   632M  /usr/jails/jails-data/test1-data
```

Enable the atime attribute for jails/test1:

> zfs set atime=on  jails/test1

Now that you have configured the SSH service and adding user for login inside the container, you can restart the container and login via SSH. After that, the [find](https://man.freebsd.org/find/1) command will show files that "participated" in this activity (pay attention to the `-atime -5m` modifier - it shows the changed time for the last 5 minutes, so the interval between entering the jail and running `find` command should be no more than these values:

```
find ~cbsd/jails-data/test1-data -type f -atime -5m -print
/usr/jails/jails-data/test1-data/var/run/sshd.pid
/usr/jails/jails-data/test1-data/var/run/motd
/usr/jails/jails-data/test1-data/var/run/ld-elf.so.hints
/usr/jails/jails-data/test1-data/var/run/utx.active
/usr/jails/jails-data/test1-data/var/log/utx.lastlogin
/usr/jails/jails-data/test1-data/root/.cshrc
/usr/jails/jails-data/test1-data/root/.history
/usr/jails/jails-data/test1-data/root/.login
/usr/jails/jails-data/test1-data/lib/libcrypto.so.111
/usr/jails/jails-data/test1-data/bin/pwd
...
```

Obviously, these files must be present in our 'gold' micro-container.

or **second method**: Another convenient option for monitoring the required files is [AUDITD](https://man.freebsd.org/auditd/8). Set up AUDIT to monitor files on the host system:

```
cat > /etc/security/audit_control <<EOF
dir:/var/audit
flags:+fc,+fd,+fw
minfree:5
naflags:+fc,+fd,+fw
policy:cnt,argv
filesz:1M
expire-after:10M
EOF
```

then:
> service auditd onestart

Relogin to host and restart the `test1` container (AUDITD does not log events in sessions started **before** AUDITD started). Now, through the [praudit](https://man.freebsd.org/praudit/1) utility, you will be able to see all the files that are being read and written. Filter the output to get files related to the `test1` container only:

> praudit /dev/auditpipe | grep ^path| grep test1

## Copy file structure to micro-container

