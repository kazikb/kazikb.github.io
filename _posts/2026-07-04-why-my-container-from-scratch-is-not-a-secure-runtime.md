---
layout: post
title:  "Why My Container From Scratch Is Not a Secure Runtime"
date:   2026-07-04 21:32:48 +0200
categories: linux containers
tags: [security, linux, containers]
---

## Introduction

In one of my previous posts, [containers-from-scratch](/linux/containers/2026/05/22/containers-from-scratch.html), I created a minimal “container” with shell commands. The goal was to show that a container is not a single kernel feature. My version used Linux namespaces, cgroups, and a prepared root filesystem to create a container-like environment.

This form of isolation still has major security gaps that real container runtimes commonly mitigate with additional kernel features and runtime configuration. To make the difference visible, I will compare it with an official Alpine image started with rootless Podman.

I already covered building blocks such as OverlayFS, `pivot_root`, a minimal `/dev`, `/proc`, networking, and cgroups v2 in the previous post, so I will skip them here. This post focuses only on the hardening mechanisms that were intentionally skipped: user namespaces, capability reduction, seccomp filtering, and AppArmor confinement. I expected my minimal container to show basic namespace isolation, while Podman would add hardening layers that my shell-based setup did not implement.

## Shell-based Container vs Rootless Podman

I ran the tests in this environment:

* Host OS: Ubuntu 24.04
* Kernel: 6.17.0-35-generic
* Podman version: 4.9.3
* OCI runtime: crun
* Podman mode: rootless
* AppArmor: enabled
* User namespace restriction: `kernel.apparmor_restrict_unprivileged_userns=1`

The first step in this lab was to create the shell-based container as described in my previous post, [containers-from-scratch](/linux/containers/2026/05/22/containers-from-scratch.html). The command below is the main `unshare` fragment from that setup. The tool is built around the `unshare(2)` system call and creates new namespaces before starting the shell.

> **Safety note:** The `unshare` command below comes from the full lab in the previous post, but it is only a fragment. It changes mount, filesystem, device, `/proc`, and network state. Do not run it on a production system or on your daily-use host. If you want to try it yourself, use a disposable VM and follow the complete setup from the previous article.

```bash
unshare \
    --uts \
    --pid \
    --mount \
    --net \
    --ipc \
    --cgroup \
    --fork \
    /bin/bash -c "
        set -e
        echo '  [+] Setup container side'

        echo '  [+] Setup root filesystem'
        cd /tmp/container1/merged
        mount --make-rprivate /
        mkdir old_root
        pivot_root . old_root
        umount -l /old_root
        rm -rf /old_root

        echo '  [+] Mount additional filesystems and devices'
        /bin/mount -t tmpfs tmpfs /dev
        /bin/mkdir -p /dev/{pts,shm}
        /bin/mount -t devpts devpts /dev/pts
        /bin/mount -t tmpfs tmpfs /dev/shm
        /bin/mount -t tmpfs tmpfs /run
        /bin/mount -t proc proc /proc

        /bin/mknod -m 666 /dev/null c 1 3
        /bin/mknod -m 666 /dev/zero c 1 5
        /bin/mknod -m 666 /dev/random c 1 8
        /bin/mknod -m 666 /dev/urandom c 1 9
        /bin/mknod -m 666 /dev/tty c 5 0

        echo '  [+] Setup hostname and loopback interface'
        hostname container1
        echo 'PS1=\"\h # \"' >> /etc/profile
        ip addr add 127.0.0.1/8 dev lo
        ip link set lo up

        echo '  [+] Switch to new shell'
        exec /bin/sh -l
    "
```

The command creates several namespaces:

* **UTS namespace (`--uts`)**
  Allows the container to have its own hostname independent of the host.

* **PID namespace (`--pid`)**
  Provides an independent process ID space. The first process started here becomes PID 1 inside the container, even though it has a completely different PID on the host.

* **Mount namespace (`--mount`)**
  Isolates the mount table. Changes to mounts inside the container do not affect the host (once propagation is handled correctly).

* **Network namespace (`--net`)**
  Creates a completely separate network stack: interfaces, routing tables, firewall rules, etc.

* **IPC namespace (`--ipc`)**
  Isolates shared memory and other inter-process communication mechanisms.

* **Cgroup namespace (`--cgroup`)**
  Provides a private view of the cgroup hierarchy (useful for tools inspecting `/proc/self/cgroup`).

* **`--fork`**
  Spawns a new process that becomes PID 1 inside the new PID namespace.

The important namespace missing from my previous implementation is the user namespace. From a container security perspective this is important because it controls how UIDs and GIDs inside the container map to UIDs and GIDs outside the container.

To verify the namespace setup, I compared the container process with host PID 1 and checked which namespaces were different. Entries in `/proc/<pid>/ns` are symbolic links that expose namespace identifiers. If two processes have the same namespace type and inode number, they are members of the same namespace. To confirm this, first let’s find the PID of my namespaced process.

```bash
ps auxf
.
.
xor         3670  0.0  0.1  29852 11164 ?        Ss   21:28   0:01  \_ tmux
xor         3671  0.0  0.1  25152 10304 pts/1    Ss+  21:28   0:00  |   \_ -bash
xor         6943  0.0  0.1  23612  9456 pts/4    Ss   21:43   0:00  |   \_ -bash
root        8285  0.0  0.1  28368  8268 pts/4    S+   21:46   0:00  |   |   \_ sudo -i
root        8288  0.0  0.0  28368  2732 pts/5    Ss   21:47   0:00  |   |       \_ sudo -i
root        8289  0.0  0.0  20088  5900 pts/5    S    21:47   0:00  |   |           \_ -bash
root        8477  0.0  0.0  16968  2076 pts/5    S    21:50   0:00  |   |               \_ unshare --uts --pid --mount --ne
root        8478  0.0  0.0   1748  1232 pts/5    S+   21:50   0:00  |   |                   \_ /bin/sh -l
.
.
```

From the process list, I can identify `8478` as the host PID of the shell running inside the new namespaces. To make the comparison easier, I will collect this information from the host perspective with a simple loop and show the host and container values side by side.

```bash
# List all namespaces associated with the process
ls -l /proc/1/ns
total 0
lrwxrwxrwx 1 root root 0 Jun 27 21:25 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jun 27 21:25 mnt -> 'mnt:[4026531832]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 net -> 'net:[4026531833]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 27 23:27 uts -> 'uts:[4026531838]'

# Loop for comparing host <-> container namespace ID
HOST_PID=1
CONTAINER_PID=8478

for ns in pid mnt net uts ipc cgroup time user; do
  echo "== $ns =="
  echo -n "host:      "
  readlink /proc/$HOST_PID/ns/$ns
  echo -n "container: "
  readlink /proc/$CONTAINER_PID/ns/$ns
done
```
```bash
# Output
== pid ==
host:      pid:[4026531836]
container: pid:[4026532915]
== mnt ==
host:      mnt:[4026531832]
container: mnt:[4026532453]
== net ==
host:      net:[4026531833]
container: net:[4026532917]
== uts ==
host:      uts:[4026531838]
container: uts:[4026532913]
== ipc ==
host:      ipc:[4026531839]
container: ipc:[4026532914]
== cgroup ==
host:      cgroup:[4026531835]
container: cgroup:[4026532916]
== time ==
host:      time:[4026531834]
container: time:[4026531834]
== user ==
host:      user:[4026531837]
container: user:[4026531837]

# Check if CONTAINER_PID 8478 on host maps to pid 1 in pid namespace
grep NSpid /proc/$CONTAINER_PID/status
NSpid:  8478    1
```

Compared with host PID 1, the container process has separate PID, mount, network, UTS, IPC, and cgroup namespaces, but it still shares the user and time namespaces with the host. The shared user namespace is important because it means UID/GID mappings were not isolated. The time namespace is outside the scope of this post; it virtualizes offsets for `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME`, but I do not treat it as one of the main container security boundaries here.

Now let’s compare this with a simple container created by a real container runtime. I used Podman in rootless mode, started as my normal user. Since I used an Alpine minirootfs for my shell-based container, I used the official Alpine image for the Podman comparison and kept Podman’s default settings.

```bash
# Start a rootless Podman container
podman run -it --name alpine docker.io/library/alpine:3.23.3 /bin/sh

# Find container process
ps auxf
.
.
xor         3406  0.0  0.1  27156  8324 ?        Ss   20:42   0:00  \_ tmux
xor         3407  0.0  0.1  23744  9624 pts/1    Ss   20:42   0:00  |   \_ -bash
xor         7748  0.3  0.5 1935184 42504 pts/1   Sl+  20:50   0:00  |   |   \_ podman run -it --name alpine docker.io/library/alpine:3.23.3 /bin/sh
xor         7760  0.0  0.0   6204  3632 pts/1    S    20:50   0:00  |   |       \_ /usr/bin/slirp4netns --disable-host-loopback --mtu=65520 --enable-sandbox --enable-seccomp --enable-ipv6 -c -r 3 -e 4 --netns-type=path /run/user/1000/netns/netns-cf
xor         7811  0.3  0.1  23480  9264 pts/3    Ss   20:51   0:00  |   \_ -bash
xor         7916  0.0  0.0  22556  5008 pts/3    R+   20:51   0:00  |       \_ ps auxf
xor         5902  0.0  0.0 149320  2960 ?        Sl   20:45   0:00  \_ /snap/firefox/8568/usr/lib/firefox/crashhelper 5828 9 /tmp/ 11
xor         6059  0.0  0.3 1840332 25260 ?       Sl   20:45   0:00  \_ /usr/bin/snap userd
xor         7428  0.0  0.0   1136   808 ?        S    20:49   0:00  \_ catatonit -P
xor         7766  0.0  0.0  23456  2556 ?        Ss   20:50   0:00  \_ /usr/bin/conmon --api-version 1 -c 5f1ff0cde30f11ad6b86654efed7f37e6b4b3513182af65f15dcddefcdf1479d -u 5f1ff0cde30f11ad6b86654efed7f37e6b4b3513182af65f15dcddefcdf1479d -r /usr/b
xor         7768  0.0  0.0   1744  1220 pts/0    Ss+  20:50   0:00  |   \_ /bin/sh
.
.
```

From the process tree, I can identify the shell running inside the Podman container, just like I did for the shell created with `unshare`. Podman also exposes the container PID directly through `podman inspect`.

{% raw %}
```bash
podman inspect --format '{{.State.Pid}}' alpine
7768
```
{% endraw %}

I used the same namespace comparison loop for the Podman process.

```bash
HOST_PID=1
PODMAN_CONTAINER_PID=7768

for ns in pid mnt net uts ipc cgroup time user; do
  echo "== $ns =="
  echo -n "host:      "
  readlink /proc/$HOST_PID/ns/$ns
  echo -n "container: "
  readlink /proc/$PODMAN_CONTAINER_PID/ns/$ns
done
```
```bash
# Output
== pid ==
host:      pid:[4026531836]
container: pid:[4026533201]
== mnt ==
host:      mnt:[4026531832]
container: mnt:[4026533197]
== net ==
host:      net:[4026531833]
container: net:[4026533141]
== uts ==
host:      uts:[4026531838]
container: uts:[4026533199]
== ipc ==
host:      ipc:[4026531839]
container: ipc:[4026533200]
== cgroup ==
host:      cgroup:[4026531835]
container: cgroup:[4026533202]
== time ==
host:      time:[4026531834]
container: time:[4026531834]
== user ==
host:      user:[4026531837]
container: user:[4026533139]
```

The output shows that this rootless Podman process has different PID, mount, network, UTS, IPC, cgroup, and user namespace identifiers than host PID 1. The time namespace is still shared with the host. This behavior is specific to this default Podman configuration on this host. It is not a universal rule, because Podman can also be configured to share or join selected namespaces.

### 1. Root is not safely remapped

Without a separate user namespace, my shell-based container still uses the host’s initial user namespace. I started it as root, so `root` inside the container is still `root` from the host’s point of view.

The process has a narrowed view of the system because of the other namespaces, but its UID is not safely remapped. If another mistake exposes a sensitive host path, leaves the old root reachable, or allows interaction with host resources, UID 0 still has host-root meaning.

A user namespace would reduce this risk by mapping container UID 0 to an unprivileged host UID or to a subordinate UID range. The process could still look like root inside the container, but it would not have the same authority over host resources.

We can verify this by comparing the UID/GID maps from inside the container and from the host.

First, let’s check what this looks like from inside the container.

```bash
id
uid=0(root) gid=0(root) groups=0(root)

# UID map
cat /proc/self/uid_map
         0          0 4294967295

# GID map
cat /proc/self/gid_map
         0          0 4294967295

# user namespace
readlink /proc/self/ns/user
user:[4026531837]
```

Now I can compare those values with what the host sees.

```bash
CONTAINER_PID=8478

# Host view of container process IDs
grep -E 'Uid|Gid' /proc/$CONTAINER_PID/status
Uid:    0       0       0       0
Gid:    0       0       0       0

# Host view of UID/GID maps
cat /proc/$CONTAINER_PID/uid_map
         0          0 4294967295

cat /proc/$CONTAINER_PID/gid_map
         0          0 4294967295

# Compare user namespace with host init
readlink /proc/1/ns/user
user:[4026531837]

readlink /proc/$CONTAINER_PID/ns/user
user:[4026531837]
```

The UID/GID map has three columns: the starting ID inside the namespace, the starting ID outside the namespace, and the length of the mapping.

Here the mapping is `0 0 4294967295`. This is the initial user namespace mapping. It means that UID 0 maps to UID 0, and the range covers all normal UID values except the special unmapped value `4294967295`.

Now I can compare this with the Podman container.

```bash
# Podman container side
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)

# UID map
cat /proc/self/uid_map
         0       1000          1
         1     100000      65536

# GID map
cat /proc/self/gid_map
         0       1000          1
         1     100000      65536

# user namespace
readlink /proc/self/ns/user
user:[4026533139]

# Host side
PODMAN_CONTAINER_PID=7768

# Host view of container process IDs
grep -E 'Uid|Gid' /proc/$PODMAN_CONTAINER_PID/status
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000

# Host view of UID/GID maps
cat /proc/$PODMAN_CONTAINER_PID/uid_map
         0       1000          1
         1     100000      65536

cat /proc/$PODMAN_CONTAINER_PID/gid_map
         0       1000          1
         1     100000      65536

# Compare user namespace with host init
readlink /proc/1/ns/user
user:[4026531837]

readlink /proc/$PODMAN_CONTAINER_PID/ns/user
user:[4026533139]
```

For this rootless container, Podman created a separate user namespace. Inside the container, the shell runs as UID 0, but from the host’s point of view the same process runs as UID 1000. Other container UIDs are mapped into the subordinate UID/GID range starting at 100000.

This mapping is visible in `/proc/<pid>/uid_map` and `/proc/<pid>/gid_map`. Files such as `/etc/subuid` and `/etc/subgid` define which subordinate UID/GID ranges a normal user is allowed to use when tools like `newuidmap` and `newgidmap` configure UID and GID mappings for a user namespace.

### 2. Capabilities are not reduced

Another security mechanism I omitted is capability reduction. Linux capabilities split root privileges into smaller units. For example, a service that only needs to bind to a traditionally privileged port below 1024 may need `CAP_NET_BIND_SERVICE`, not full root.

Capabilities are evaluated relative to a user namespace. A process may have capabilities inside its own user namespace, but that does not automatically give it the same authority in the initial host user namespace. This matters here because my shell-based container did not create a separate user namespace.

In my shell-based container, I did not drop capabilities after the setup phase. I also did not reduce the bounding set. This means the shell and its child processes keep more privileges than most workloads need, and a future `execve()` can still gain supported capabilities from file capabilities if such files exist in the container filesystem.

Even when namespaces limit what the process can see, a broad capability set still gives the process more authority inside those namespaces. Capabilities such as `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, or `CAP_DAC_OVERRIDE` can significantly change what a process is allowed to do.

We can verify this by inspecting the capability fields from the running process.

```bash
# capabilities of process inside a container
cat /proc/self/status | grep Cap
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000

# capabilities of process from host perspective
CONTAINER_PID=8478

cat /proc/$CONTAINER_PID/status | grep Cap
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000

# Decoded values
capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read,cap_perfmon,cap_bpf,cap_checkpoint_restore
```

The most important fields here are `CapEff`, `CapPrm`, and `CapBnd`.

`CapEff` is the set of capabilities the kernel checks when the process tries to perform privileged operations. `CapPrm` is the set of capabilities the thread is allowed to make effective. `CapBnd` is the upper limit for capabilities that can be gained through future `execve()` paths, especially from file capabilities.

The values are the same inside the container and from the host view. They also decode to the full capability set available on this kernel. I can verify the highest supported capability by reading `/proc/sys/kernel/cap_last_cap`.

```bash
cat /proc/sys/kernel/cap_last_cap
40
```

With `cap_last_cap=40`, the kernel supports capability IDs from 0 to 40, which means 41 capabilities in total. This matches the decoded capability set from my shell-based container.

Now I can compare this with the capability set in the rootless Podman container. The process running inside that container has a much smaller capability set.

```bash
# capabilities of process inside a Podman container
cat /proc/self/status | grep Cap
CapInh: 0000000000000000
CapPrm: 00000000800405fb
CapEff: 00000000800405fb
CapBnd: 00000000800405fb
CapAmb: 0000000000000000

# capabilities of process from host perspective
PODMAN_CONTAINER_PID=7768

cat /proc/$PODMAN_CONTAINER_PID/status | grep Cap
CapInh: 0000000000000000
CapPrm: 00000000800405fb
CapEff: 00000000800405fb
CapBnd: 00000000800405fb
CapAmb: 0000000000000000

capsh --decode=00000000800405fb
0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
```

The key difference is not only the smaller capability set. The scope of those capabilities is different too. In my shell-based container, the process still runs in the host’s initial user namespace, so its capabilities have host-level meaning. In the rootless Podman container, the process has capabilities inside its own user namespace. The same hexadecimal capability fields are still visible from the host through `/proc/<pid>/status`, but they are not host-root capabilities in the initial user namespace. The process can look privileged inside the container without having the same authority over the host.

### 3. No seccomp filtering

Another missing mechanism is seccomp filtering. In my shell-based container, no seccomp filter is installed for the shell process. The Linux kernel exposes hundreds of system calls, but most applications need only a subset of them. Seccomp allows a process to restrict which system calls it can make. This reduces the kernel attack surface available to the application. We can check these settings for the process inside my container:

```bash
# seccomp filter status inside a container
cat /proc/self/status | grep -E 'NoNewPrivs|Seccomp'
NoNewPrivs:     0
Seccomp:        0
Seccomp_filters:        0
```

* `Seccomp: 0` means seccomp is disabled.
* `Seccomp_filters: 0` means no seccomp filters are installed.

The output shows that seccomp is not enabled and no filters are installed for this process. If the parent process had already been seccomp-filtered, the child would inherit that restriction, but that is not the case here.

I also want to highlight `NoNewPrivs`. This flag connects seccomp with the earlier discussion about capabilities and user namespaces. When `no_new_privs` is set, the process cannot gain additional privileges by later calling `execve()`. Once set, the flag is inherited by child processes and cannot be unset.

For seccomp, this matters because an unprivileged process normally needs `no_new_privs` before it can install a seccomp filter. Without it, the process needs `CAP_SYS_ADMIN` in its user namespace.

We will now compare this with the values set for this default rootless Podman container.

```bash
# seccomp filter status inside a Podman container
cat /proc/self/status | grep -E 'NoNewPrivs|Seccomp'
NoNewPrivs:     0
Seccomp:        2
Seccomp_filters:        1
```

* `Seccomp: 2` means the process is running in seccomp filter mode.
* `Seccomp_filters: 1` means one seccomp filter is installed.

For this container, Podman uses its default seccomp profile. On this host, that profile is stored at `/usr/share/containers/seccomp.json`.

One important detail is that `NoNewPrivs` is not set. This can be changed by starting the container with `--security-opt no-new-privileges`. Because this process has `NoNewPrivs` set to `0` and does not have `CAP_SYS_ADMIN` in its capability set, it cannot install a new seccomp filter in its current state. It would first need to set `no_new_privs`, assuming the existing seccomp policy allows that path.

Another important detail is that adding a new seccomp filter does not replace the old one. If multiple filters exist, all of them are evaluated, and the kernel applies the action with the highest precedence. A later filter cannot remove restrictions from an earlier filter.

### 4. No restrictive AppArmor confinement

Namespaces, capabilities, and seccomp are not the only security layers used around containers. On Ubuntu, the default Linux Security Module used for mandatory access control is AppArmor. AppArmor can restrict what a process is allowed to access through profiles loaded on the host.

My shell-based container does not explicitly load or apply an AppArmor profile. Depending on how the parent process was started, it may still inherit an existing AppArmor label or profile from the host. In this lab, the process is unconfined.

To verify this, I checked the current AppArmor label of the container process:

```bash
# Inside container
cat /proc/self/attr/current
unconfined

# From host
CONTAINER_PID=8478

cat /proc/$CONTAINER_PID/attr/current
unconfined
```

The result is `unconfined`, so AppArmor does not add a restrictive policy around my shell-based container.

I checked the same value for the Podman container:

{% raw %}
```bash
# Inside Podman container
cat /proc/self/attr/current
crun (unconfined)

# From host
PODMAN_CONTAINER_PID=7768

cat /proc/$PODMAN_CONTAINER_PID/attr/current
crun (unconfined)

# Verify what container runtime is used by Podman
podman info --format '{{.Host.OCIRuntime.Name}}'
crun
```
{% endraw %}

For the Podman container, the label is `crun (unconfined)`. This gives the process an AppArmor profile name, but the profile is still marked as unconfined, so it does not restrict the container process.

The profile file explains why it exists:

```bash
cat /etc/apparmor.d/crun
# This profile allows everything and only exists to give the
# application a name instead of having the label "unconfined"

abi <abi/4.0>,
include <tunables/global>

profile crun /usr/bin/crun flags=(unconfined) {
  userns,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/crun>
}
```

The important parts are `flags=(unconfined)` and the `userns,` rule. On Ubuntu 24.04, AppArmor can restrict unprivileged user namespace creation. In this environment, the `crun` profile allows the runtime to create a user namespace under Ubuntu’s AppArmor-based restriction model, but it does not add a restrictive AppArmor policy around the container process.

A more restrictive AppArmor profile can still be loaded on the host and selected for a Podman container with `--security-opt apparmor=<profile-name>`.

So AppArmor is the exception in this comparison. Podman applies several security defaults that my shell-based container does not: rootless user namespace mapping, a reduced capability set, and a seccomp filter. But in this Ubuntu lab, neither container is restricted by a meaningful AppArmor policy.

## Conclusion

After this comparison, the main lesson is clear: my shell-based container had real namespace isolation, but it was still not a secure runtime.

The reason was not one missing option. It was the missing security stack around the process. Root was not remapped through a user namespace. Capabilities were not reduced. Seccomp was not filtering syscalls. AppArmor did not add a restrictive profile.

Comparing it with a rootless Podman container made these gaps easier to see. Podman added several layers that my shell-based setup did not implement: a separate user namespace, a smaller capability set, and a seccomp filter. AppArmor was the useful exception here, because it showed that runtime defaults still depend on the host distribution and local security configuration.

This was the main takeaway for me: creating a container-like environment and running it securely are different problems. The first one can be demonstrated with namespaces, cgroups, mounts, and a root filesystem. The second one needs deliberate runtime decisions about identity, privileges, syscalls, and AppArmor policy.

The implementation will differ between tools, but the questions stay the same: what can the process see, what can it modify, what privileges does it keep, and which kernel interfaces can it call?

## Sources

* [https://kazikb.github.io/linux/containers/2026/05/22/containers-from-scratch.html](https://kazikb.github.io/linux/containers/2026/05/22/containers-from-scratch.html)
* [https://docs.kernel.org/userspace-api/no_new_privs.html](https://docs.kernel.org/userspace-api/no_new_privs.html)
* [https://specs.opencontainers.org/runtime-spec/config/?v=v1.0.2](https://specs.opencontainers.org/runtime-spec/config/?v=v1.0.2)
* [https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces)
* [https://docs.podman.io/en/v4.9.3/markdown/podman-run.1.html](https://docs.podman.io/en/v4.9.3/markdown/podman-run.1.html)
* [https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html](https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html)
* [https://www.apparmor.net/](https://www.apparmor.net/)
* [https://man7.org/linux/man-pages/man7/namespaces.7.html](https://man7.org/linux/man-pages/man7/namespaces.7.html)
* [https://man7.org/linux/man-pages/man7/user_namespaces.7.html](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)
* [https://man7.org/linux/man-pages/man7/capabilities.7.html](https://man7.org/linux/man-pages/man7/capabilities.7.html)
* [https://man7.org/linux/man-pages/man2/seccomp.2.html](https://man7.org/linux/man-pages/man2/seccomp.2.html)
* [https://man7.org/linux/man-pages/man5/proc_pid_status.5.html](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html)

> **Editorial note:** This article is based on my own lab work and technical verification. GPT-5.5 Thinking was used as an editing assistant for structure and English clarity.
