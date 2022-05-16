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

Using the CBSD `copy-binlib` script and the already existing index file for a minimal FreeBSD + SSH environment in the `test1` container:

> cbsd copy-binlib basedir=/ chaselibs=1 dstdir=/usr/jails/jails-data/micro1-data filelist=/usr/local/cbsd/share/FreeBSD-microbhyve.txt.xz

Note: make sure */usr/jails* is the correct path for the CBSD environment in your installation.

## Micro (gold) container customization

With a minimal structure, we can make a number of necessary settings, such as allowing SSH access for the root user, configuring the network, and so on. We will configure the virtual machine network through the */etc/rc.local* file - are you striving for a minimal structure? ;-) so don't copy the entire */etc/rc.d* directory:

```
cbsd sysrc jname=micro1 \
        sshd_flags="-oUseDNS=no -oPermitRootLogin=yes" \
        root_rw_mount="YES" \
        sshd_enable=YES \
        rc_startmsgs="YES" 

cat > /usr/jails/jails-data/micro1-data/etc/rc.local <<EOF
/sbin/ifconfig vtnet0 inet 10.0.100.10/24 up
/sbin/route add default 10.0.100.1
EOF

cp -a /etc/ssh /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/gss /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/pam.d /usr/jails/jails-data/micro1-data/etc/
mkdir -p /usr/jails/jails-data/micro1-data/var/empty /usr/jails/jails-data/micro1-data/var/log /usr/jails/jails-data/micro1-data/var/run /usr/jails/jails-data/micro1-data/root /usr/jails/jails-data/micro1-data/dev
chmod 0700 /usr/jails/jails-data/micro1-data/var/empty
pw -R /usr/jails/jails-data/micro1-data usermod root -s /bin/sh

# strip debug info via strip(1)
find /usr/jails/jails-data/micro1-data/ -type f -perm +111 -exec strip {} \;
```

In addition, you can optionally disable the pause in [loader.conf](https://man.freebsd.org/loader.conf/5) before loading the kernel through the `autoboot_delay` parameter and the *~cbsd/jails-system/micro1/loader.conf* file, which is processed by the CBSD `jail2iso` script. In this case, you are unlikely to be able to see the loading process of FreeBSD OS - everything will happen instantly ;-)

## Getting the FreeBSD microkernel for bhyve

We need the FreeBSD sources (/usr/src) to compile a custom FreeBSD kernel for our bhyve environment. There is already a microkernel configuration file in CBSD called 'BHYVE', so we will get the FreeBSD source code (if not already) and compile the kernel:

```
cbsd srcup
cbsd kernel name=BHYVE
```

As a result of these commands, the FreeBSD micro-kernel will be waiting for us in the *~cbsd/basejail/FreeBSD-kernel_BHYVE_amd64_XX* directory, which we will specify when running the CBSD `jail2iso` script. At the same time, some of the modules may not be useful to us when working in bhyve, and we can even pack the core itself via gzip:

```
ls -1 /usr/jails/basejail/FreeBSD-kernel_BHYVE_amd64_13.1/boot/kernel/
if_vtnet.ko
kernel
linker.hints
virtio.ko
virtio_balloon.ko
virtio_blk.ko
virtio_console.ko
virtio_pci.ko
virtio_random.ko
virtio_scsi.ko

rm -f /usr/jails/basejail/FreeBSD-kernel_BHYVE_amd64_13.1/boot/kernel/{virtio_balloon.ko,virtio_console.ko,virtio_scsi.ko}
strip /usr/jails/basejail/FreeBSD-kernel_BHYVE_amd64_13.1/boot/kernel/*.ko
gzip -9 /usr/jails/basejail/FreeBSD-kernel_BHYVE_amd64_13.1/boot/kernel/kernel
```

## CBSD jail2iso magic: get the image!

We just have to get the image from the `micro1` container. Let the image be generated into the **/tmp** directory. The `freesize` parameter controls the amount of free disk space:

> cbsd jail2iso name=BHYVE jname=micro1 dstdir=/tmp media=bhyve freesize=4m efi=1 ver=native

As a result, a small image **/tmp/micro1-13.1_amd64.img** will be generated:

```
du -sh /tmp/micro1-13.1_amd64.img
12M    /tmp/micro1-13.1_amd64.img
```

which you can check immediately via the `bhyve` startup script:

> sh /usr/share/examples/bhyve/vmrun.sh -d /tmp/micro1-13.1_amd64.img micro1

However, it is better to create a CBSD virtual machine to set up the network and so on. To do this, stop the running VM (if started):

```
ps axfwww | grep "bhyve: micro1" |grep -v grep
kill <PID_OF_BHYVE>
```

Create VM. Here we can create a disk of any size - we will overwrite it anyway:

> cbsd bcreate jname=micro vm_os_type=freebsd vm_os_profile=FreeBSD-x64-13.1 imgtype=md imgsize=1g ip4_addr=10.0.100.10

Let's overwrite the disk created by the `bcreate` with a `micro` file from CBSD `jail2iso`:

> cp -a /tmp/micro1-13.1_amd64.img ~cbsd/jails-data/micro-data/dsk1.vhd

Now we can start the VM and by calling `ssh root@10.0.100.10` you can log into the VM using the password of the 'root' user from your host system, because we copied the files from the host system ( do you remember `cbsd copy-binlib basedir=/` ? ).

## Errata

The kernel config file is really minimal. You can view its contents: *~cbsd/etc/defaults/FreeBSD-kernel-BHYVE-amd64-13.1*.

If you open the VNC console on bhyve terminal, apart from the boot menu, you will not see the boot process and the console, since the devices responsible for outputting information in UEFI mode are also **turned off**. You can rebuild the kernel by uncommenting the entries:

```
device         vt
device         vt_efifb
```


:bangbang: | :warning: Attention! Do not edit the *~cbsd/etc/defaults/FreeBSD-kernel-BHYVE-amd64-13.1* file, instead copy it or create your own 'FreeBSD-kernel-XXXX-amd64-13.1' file in the *~cbsd/etc/* directory because the files in *~cbsd/etc/defaults* is overwritten by `cbsd initenv`.
:---: | :---

## Afterword, instead of conclusion.

[CBSD](https://github.com/cbsd/cbsd) is not only a jail and VM management tool, but also a framework that includes various auxiliary utilities for working with jail and VM, since they reuse the lots of common code of basic functions and maintaining them separately from CBSD is inappropriate. One of these scripts is `jail2iso`, with which you can get various bonuses from **CBSD**. For example, we all know such LiveCD distributions as [NomadBSD](https://nomadbsd.org/), [GhostBSD](https://ghostbsd.org/), [helloSystem](https://github.com/helloSystem/hello), etc. Using the CBSD `jail2iso` script, you can easily create your own LiveCD distributions based on FreeBSD, preparing and checking the settings and services in the jail container in advance. In a similar way, you can generate the most stripped-down FreeBSD distribution, where there is only your service and nothing more.

Since we used compression only for the kernel, but not rootfs, it is possible to get an image size of up to **5MB** without changing the file structure (for example, with specialized busybox-like projects) by using `uzip/geom_uzip`, but more on that in another article.

_
- **About FreeBSD OS**: Freely distributed under the most liberal BSD license, a general-purpose OS, the code of which is not affiliated with any companies;
- **About the CBSD project**: Founded in 2013, the project aims to create a framework for working with FreeBSD virtual environments to facilitate the creation of cluster and cloud solutions based on FreeBSD;
- **About the 'jail2iso' script**: the script was written and included in **CBSD** since 2014, the purpose of the script is to convert the jail of the FreeBSD operating system into a bootable media: ISO, memstick or virtual machine image. It is convenient for creating custom liveCD builds of FreeBSD - you can pre-configure and test all services in jail and make sure that they are correct and working, get a working ISO image or VM image with one command;
- **About bhyve**: a type 2 hypervisor included with FreeBSD that can run modern operating systems;

