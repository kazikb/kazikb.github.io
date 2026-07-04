---
layout: post
title:  "Containers from Scratch"
date:   2026-05-22 23:45:13 +0200
categories: linux containers
tags: [networking, linux, containers]
---
## Introduction

My first technical post documents a minimal container runtime implemented entirely in Bash using native Linux kernel primitives. The goal was not to replace tools like Docker or Podman, but to expose the underlying mechanisms they orchestrate.

The implementation was inspired by a talk from [Michael Kerrisk at NDC Security 2025](https://www.youtube.com/watch?v=4RUiVAlJE2w), as well as the excellent write-up by [Michal Pitr Linux container from scratch](https://michalpitr.substack.com/p/linux-container-from-scratch). Both resources emphasize an important point: containers are not a single kernel feature, but a composition of existing ones.

Containers are foundational to modern infrastructure, but they are also useful as lightweight isolation boundaries for example, when running AI agents or development environments. While higher-level tooling abstracts this away, the runtime behavior ultimately depends on standard kernel subsystems: process isolation, filesystem layering, networking, and resource control.

This post intentionally omits advanced hardening mechanisms such as seccomp filters, Linux capabilities management, and user namespaces. These are critical in production systems, but would obscure the core concepts demonstrated here.

It is also important to note that this example does **not** use user namespaces. As a result, processes running as `root` inside the container are also `root` on the host system.


## What “Containers” Actually Are

First, let's try to explain what "containers" really are. The Linux kernel does not implement “containers” as a first-class abstraction. A container is best understood as a set of isolated processes created by combining multiple kernel features.

It is also important to distinguish containers from virtual machines. A virtual machine emulates hardware and runs a full guest operating system on top of that abstraction. Containers, in contrast, share the host kernel and isolate processes within it.

The isolation model relies on three primary mechanisms:

* **Namespaces**
  Provide isolation of global system resources. They control what a process can “see” (e.g. PIDs, network interfaces, mount points).

* **Control Groups (cgroups)**
  Enforce resource limits and accounting (CPU, memory, I/O).

* **OverlayFS**
  Enables layered filesystems, allowing multiple containers to share a common base image while maintaining independent writable layers.

Together, these features create the illusion of independent environments, even though all processes are managed by the same kernel.


## Host Environment Preparation

***Do not run these commands on a production system without fully understanding their impact.***

The objective is to create a minimal environment where multiple containers can:

1. Communicate with each other over a private network
2. Access external resources (e.g. download packages)

The diagram below shows the target architecture we are going to build. Each container runs in its own network namespace and is connected to the host via a veth pair. The host side of these interfaces is attached to a Linux bridge (`minidocker-br`), which acts as a virtual switch. Traffic leaving the bridge is routed through the host network interface and translated using NAT (MASQUERADE), allowing containers to reach external networks.

![00-network-diagram]({{ "/assets/images/2026-05-22-containers-from-scratch/00-network-diagram.svg" | relative_url }})

### Base Root Filesystem

Instead of constructing a root filesystem manually, I will use a prebuilt minimal image. The Alpine Linux mini root filesystem provides a compact and clean base layer.

```bash
curl -sSfLo /tmp/alpine-minirootfs-3.23.3-x86_64.tar.gz https://dl-cdn.alpinelinux.org/alpine/v3.23/releases/x86_64/alpine-minirootfs-3.23.3-x86_64.tar.gz
```

Now we will switch to root. All subsequent commands are executed as root.

```bash
sudo -i

mkdir -p /tmp/base-image
tar -xf /tmp/alpine-minirootfs-3.23.3-x86_64.tar.gz -C /tmp/base-image
```

This directory will serve as the shared read-only base layer for all containers.

### Network Bridge

To enable inter-container communication, create a Linux bridge. A bridge operates as a Layer 2 switch inside the kernel, allowing multiple network interfaces to communicate on the same broadcast domain.

```bash
ip link add minidocker-br type bridge
ip link set minidocker-br up
ip addr add 172.17.10.1/24 dev minidocker-br
```

Each container will later attach a virtual Ethernet (veth) interface to this bridge.

### Packet Forwarding

IP forwarding must be enabled to allow traffic to traverse between interfaces.

```bash
sysctl -w net.ipv4.ip_forward=1
```

### NAT Configuration

To allow containers to reach external networks, outbound traffic is translated using NAT (MASQUERADE). This is implemented using iptables with the nftables backend on my host system, the Internet connected interface is `enp0s3`.

```bash
iptables -t nat -A POSTROUTING -s 172.17.10.0/24 -o enp0s3 -j MASQUERADE
iptables -A FORWARD -i minidocker-br -o enp0s3 -j ACCEPT
iptables -A FORWARD -i enp0s3 -o minidocker-br -m state --state RELATED,ESTABLISHED -j ACCEPT
```

To check what backend is used on a system you can run this command:
```bash
iptables --version

iptables v1.8.10 (nf_tables)
```

On many modern distributions (like Ubuntu 24.04 which I'm using), iptables commands are implemented through the nftables compatibility layer.

## Creating the First Container

With the host prepared, we can start building the first container. The process is incremental: first we assemble a filesystem, then apply resource limits, and finally isolate the process tree.

### Preparing the Container Filesystem

We begin by creating the directory layout required for OverlayFS:

```bash
mkdir -p /tmp/container1/{lower,upper,work,merged,lower/etc}
```

The goal is to layer container-specific configuration on top of the shared base image that we have already prepared. The `lower` directory is intentionally placed *before* the base image in the stack, so it can override selected files without modifying the base layer.

We inject two files:

```bash
cat << EOF > /tmp/container1/lower/etc/container_info
CONTAINER=container1
IP=172.17.10.10
GATEWAY=172.17.10.1
MASK=24
DATE=$(date +"%d-%m-%Y %H:%M:%S")
EOF

cat << EOF > /tmp/container1/lower/etc/resolv.conf
nameserver 1.1.1.1
EOF
```

Now mount the overlay filesystem:

```bash
cd /tmp/container1

mount -t overlay overlay merged \
    -o lowerdir=lower:/tmp/base-image \
    -o upperdir=upper \
    -o workdir=work
```

The order of `lowerdir` is important:

```
lower:/tmp/base-image
```

From a conceptual point of view, OverlayFS can be understood as a layered filesystem where multiple read-only directories are stacked on top of each other and presented as a single coherent view. These read-only layers are defined by `lowerdir`, and their ordering is significant the leftmost layer has the highest precedence, meaning it overrides files with the same path in layers to its right. So, in my example, any file or directory located in `lower` directory will override those in `base-image`.

On top of these read-only layers sits a single writable layer, defined by `upperdir`. Any modifications performed at runtime whether creating, modifying, or deleting files are recorded exclusively in this layer, leaving the underlying layers untouched. Internally, the kernel uses `workdir` as temporary scratch space to coordinate operations between layers, particularly for tasks like copy-up.

The final result is exposed through the mount point (`merged`), which presents a unified filesystem view. From the perspective of processes inside the container, this layered structure is completely transparent they interact with it as if it were a regular filesystem, even though it is composed of multiple underlying directories.

This is exactly the same mechanism used by Docker when stacking image layers — here we are just doing it manually using directories.

### Setting Up cgroups v2

Next, we prepare a cgroup v2 hierarchy to constrain CPU and memory usage. Control groups (cgroups) are a Linux kernel mechanism for organizing processes into hierarchical groups and applying resource limits, accounting, and isolation to them. The unified cgroup v2 model simplifies this by exposing a single, consistent hierarchy where controllers such as CPU and memory are enabled explicitly and applied uniformly to descendant processes. In my example, I will only enable two cgroup controllers: memory and CPU.

```bash
mkdir -p /sys/fs/cgroup/minidocker.slice/container1
echo "+memory +cpu" > /sys/fs/cgroup/minidocker.slice/cgroup.subtree_control
```

#### CPU Limits

The `cpu.max` interface expects two values:

```
quota period
```

Both are expressed in microseconds. The first value defines how much CPU time all processes within the cgroup can collectively use (the quota), while the second value defines the length of the time period in which that quota applies.

```bash
echo "10000 100000" > /sys/fs/cgroup/minidocker.slice/container1/cpu.max
```

In this case, processes in the `container1` cgroup are allowed to run for 10,000 microseconds (10 ms) within every 100,000 microseconds (100 ms) period. This effectively limits the container to about 10% of a single CPU core worth of runtime.

#### Memory Limits

```bash
echo "512M" > /sys/fs/cgroup/minidocker.slice/container1/memory.max
echo "0" > /sys/fs/cgroup/minidocker.slice/container1/memory.swap.max
```

This limits memory to 512 MB and prevents processes in the cgroup from using swap.

Finally, we attach the current shell process to the newly created cgroup:

```bash
echo $$ > /sys/fs/cgroup/minidocker.slice/container1/cgroup.procs
```

Here, `$$` expands to the PID of the current shell. By writing it to `cgroup.procs`, we move this process into the target cgroup. This approach is intentionally simple: any child processes spawned from this shell including the container init process will automatically inherit membership in the same cgroup. As a result, all resource limits (CPU, memory) applied to the cgroup are enforced on the entire process tree without needing to manage individual PIDs explicitly.

## Creating Namespaces

We now define the container’s isolation boundary using `unshare`, which is a thin wrapper around the `unshare(2)` system call. At the kernel level, this does not create a separate entity, but instead places the calling process into newly created namespaces, effectively changing how it sees selected global resources. From this point on, any child processes inherit these namespace contexts, meaning they operate within the same isolated view of the system. This is the moment where the container begins to take shape.

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

Each flag enables a different namespace:

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

### Switching the Root Filesystem

The first step inside the container is to replace the root filesystem:

```bash
echo '  [+] Setup root filesystem'
cd /tmp/container1/merged
mount --make-rprivate /
mkdir old_root
pivot_root . old_root
umount -l /old_root
rm -rf /old_root
```

There are several important details here.

First, `mount --make-rprivate /` ensures that mount events inside the container (which have its own mount namespace) do not propagate back to peer mount namespaces if they belong to a shared propagation group. Without this, mount operations performed in the container could leak outside of the namespace, which is almost never what we want.

At this stage, even though we are already inside a mount namespace, the process still sees the host filesystem. Namespaces isolate *future changes*, but they do not automatically replace what is already mounted. The actual switch happens with `pivot_root`.

The `pivot_root` call replaces the current root filesystem (`/`) with the directory we are currently in (`merged`) and moves the previous root to `/old_root`. It requires the new root filesystem to be a mount point. In my example, `merged` satisfies that requirement because it is the mount target of the OverlayFS filesystem. From that point on, the container operates entirely within its own filesystem tree, but the old root is still temporarily accessible.

To fully detach from the host, we lazily unmount the old root using `umount -l`, which immediately disconnects it and cleans up once no processes are referencing it. After that, the `old_root` directory is removed entirely.

This step is both critical and easy to get wrong. During development I misspelled the umount command, which meant the old root was never detached. The result was havoc on my host filesystem, and this is exactly why doing this kind of work inside a virtual machine with proper snapshots can save the day.

### Mounting Minimal Runtime Filesystems

Next, we construct a minimal runtime environment inside the container. Since `/dev` is mounted as a fresh `tmpfs`, we must manually create essential device nodes:

```bash
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
```

After switching the root filesystem, the container starts in a very minimal state. Many of the standard kernel interfaces and device nodes simply do not exist. Without recreating them, even basic user space tools would fail.

We begin by mounting essential virtual filesystems:

* `/proc` provides a view into the process tree and kernel state, scoped to the current PID namespace
* `/dev/pts` enables pseudo-terminals, which are required for interactive shells
* `tmpfs` mounts (`/dev/shm`, `/run`, `/dev`) provide in-memory storage for runtime data and device nodes

Since `/dev` is mounted as a fresh `tmpfs`, it starts empty. We therefore need to manually create a minimal set of device nodes using `mknod`. These are character devices that act as interfaces to kernel functionality:

* `/dev/null` — discards all written data and returns EOF on read; commonly used to suppress output
* `/dev/zero` — provides an infinite stream of zero bytes; often used for memory initialization
* `/dev/random` and `/dev/urandom` — provide random data from the kernel entropy pool (blocking and non-blocking respectively)
* `/dev/tty` — represents the controlling terminal for the current process

This setup is intentionally minimal, but sufficient to support a functional user space environment inside the container. For simplicity, `/sys` is intentionally omitted.

### Container Identity and Loopback

```bash
echo '  [+] Setup hostname and loopback interface'
hostname container1
echo 'PS1=\"\h # \"' >> /etc/profile

ip addr add 127.0.0.1/8 dev lo
ip link set lo up
```

We assign a dedicated hostname and adjust the shell prompt so it is immediately clear when operating inside the container. The loopback interface is then configured and brought up, which is required for many basic networking operations that rely on local communication.

### Init Process

```bash
echo '  [+] Switch to new shell'
exec /bin/sh -l
"
```

Finally, we replace the current process with a shell using `exec`. This ensures that the shell becomes PID 1 inside the container, rather than being spawned as a child process. As a result, the process tree remains minimal and there is no additional wrapper process between the container init and the user shell.


## Connecting the Container to the Network

At this point, the container has its own network namespace, but no external connectivity. We now attach it to the host network so it can communicate both with other containers and the outside world.

To do this, we need to perform part of the configuration from the host. Open a second terminal on the host system and identify the PID of the process running inside the container. Since the shell is currently acting as PID 1 inside the container, we can locate it using `ps`:

```bash
ps auxf

.
.
xor         3737  0.0  0.1  27816  9184 ?        Ss   21:31   0:00  \_ tmux
xor         3738  0.0  0.0  20060  5744 pts/2    Ss   21:31   0:00      \_ -bash
root        3782  0.0  0.1  28364  8264 pts/2    S+   21:34   0:00      |   \_ sudo -i
root        3783  0.0  0.0  28364  2716 pts/3    Ss   21:34   0:00      |       \_ sudo -i
root        3784  0.0  0.0  19956  5772 pts/3    S    21:34   0:00      |           \_ -bash
root        3890  0.0  0.0  16968  2080 pts/3    S    21:37   0:00      |               \_ unshare --uts --pid --mount --ne
root        3891  0.0  0.0   1748  1336 pts/3    S+   21:37   0:00      |                   \_ /bin/sh -l
xor         4040  0.0  0.0  20060  5768 pts/4    Ss   21:40   0:00      \_ -bash
xor         4133  0.0  0.0  22424  4832 pts/4    R+   21:43   0:00          \_ ps auxf
.
.
```

In this output, the shell running inside the container appears as a child of the unshare process. From the host perspective, it has PID 3891. We assign this value to a variable for later use:

```bash
container_pid=3891
```

This PID is critical, it serves as a handle to the container’s namespaces. In particular, it allows us to move network interfaces into the container’s network namespace and configure them from the host side.

### Creating and Wiring a Virtual Link

From the host console, we create a veth pair, which behaves like a virtual Ethernet cable with two endpoints:

```bash
sudo ip link add con1-host type veth peer name con1-ns
```

Conceptually, this looks like:

```
host namespace           container namespace
   con1-host  <------->   con1-ns
```

By default, both ends of the veth pair are created in the host namespace. We move one end into the container’s network namespace using the previously captured PID:

```bash
sudo ip link set con1-ns netns ${container_pid}
```

The other end remains on the host and is attached to the bridge:

```bash
sudo ip link set con1-host master minidocker-br
sudo ip link set con1-host up
```

At this point, `con1-host` is connected to `minidocker-br`, which acts as a Layer 2 switch. Any interface attached to this bridge participates in the same broadcast domain, allowing containers to communicate directly.

### Configuring Networking Inside the Container

With the host-side configuration complete, we return to the container shell and configure the interface inside the network namespace:

```bash
ip addr add 172.17.10.10/24 dev con1-ns
ip link set con1-ns up
ip route add default via 172.17.10.1
```

Here, `con1-ns` becomes the primary network interface for the container, and `172.17.10.1` (the bridge on the host) is used as the default gateway.

## Entering the Container from Another Console

```bash
nsenter -t ${container_pid} -a /bin/sh -l
```

We can attach an additional shell to the container by joining all of its namespaces using `nsenter`. The `-a` flag ensures that the new process runs with the same execution context as the container process. This works, but there is one important caveat, especially visible when using containers created with the `container-setup.sh` script from my [GitHub repository](https://github.com/kazikb/containers-from-scratch).

In that setup, the container is started with a simple long-running process (`sleep`) as PID 1, and additional shells are attached from the host using `nsenter`. While this keeps the implementation minimal, it exposes how `nsenter` actually behaves. Even though the new shell runs inside the container’s namespaces, it is not created inside the container’s process tree. From the kernel’s point of view, the process still originates from the host and only joins the namespace, so it is not a child of the container’s init process (PID 1).

## Creating a Second Container

We will now create a second container. All the steps are the same as described above. Since the current shell is already attached to the first container, you should start a new terminal session and run the same commands, replacing `container1` with `container2` and the IP `172.17.10.10` with `172.17.10.11`.


### Preparing Filesystem

```bash
# switch to root in a new console
sudo -i

mkdir -p /tmp/container2/{lower,upper,work,merged,lower/etc}

cat << EOF > /tmp/container2/lower/etc/container_info
CONTAINER=container2
IP=172.17.10.11
GATEWAY=172.17.10.1
MASK=24
DATE=$(date +"%d-%m-%Y %H:%M:%S")
EOF

cat << EOF > /tmp/container2/lower/etc/resolv.conf
nameserver 1.1.1.1
EOF

cd /tmp/container2

mount -t overlay overlay merged \
  -o lowerdir=lower:/tmp/base-image \
  -o upperdir=upper \
  -o workdir=work
```

### cgroup Configuration

```bash
mkdir -p /sys/fs/cgroup/minidocker.slice/container2

echo "+cpu +memory" > /sys/fs/cgroup/minidocker.slice/cgroup.subtree_control
echo "10000 100000" > /sys/fs/cgroup/minidocker.slice/container2/cpu.max
echo "512M" > /sys/fs/cgroup/minidocker.slice/container2/memory.max
echo "0" > /sys/fs/cgroup/minidocker.slice/container2/memory.swap.max
```

Attach current shell:

```bash
echo $$ > /sys/fs/cgroup/minidocker.slice/container2/cgroup.procs
```

### Creating Namespaces

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
    cd /tmp/container2/merged
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
    hostname container2
    echo 'PS1=\"\h # \"' >> /etc/profile
    ip addr add 127.0.0.1/8 dev lo
    ip link set lo up

    echo '  [+] Switch to new shell'
    exec /bin/sh -l
  "
```

Capture PID:

```bash
ps auxf
.
.
xor         3737  0.0  0.1  31796 13112 ?        Ss   21:31   0:00  \_ tmux
xor         3738  0.0  0.0  20060  5744 pts/2    Ss   21:31   0:00      \_ -bash
root        3782  0.0  0.1  28364  8264 pts/2    S+   21:34   0:00      |   \_ sudo -i
root        3783  0.0  0.0  28364  2716 pts/3    Ss   21:34   0:00      |       \_ sudo -i
root        3784  0.0  0.0  19956  5772 pts/3    S    21:34   0:00      |           \_ -bash
root        3890  0.0  0.0  16968  2080 pts/3    S    21:37   0:00      |               \_ unshare --uts --pid --mount --ne
root        3891  0.0  0.0   1748  1336 pts/3    S+   21:37   0:00      |                   \_ /bin/sh -l
xor         4585  0.0  0.0  20060  5752 pts/5    Ss+  21:55   0:00      \_ -bash
xor        21322  0.0  0.0  20060  5748 pts/4    Ss   22:06   0:00      \_ -bash
root       21344  0.0  0.1  28368  8252 pts/4    S+   22:06   0:00      |   \_ sudo -i
root       21345  0.0  0.0  28368  2692 pts/6    Ss   22:06   0:00      |       \_ sudo -i
root       21346  0.0  0.0  19956  5772 pts/6    S    22:06   0:00      |           \_ -bash
root       21376  0.0  0.0  16968  2088 pts/6    S    22:07   0:00      |               \_ unshare --uts --pid --mount --ne
root       21377  0.0  0.0   1748  1184 pts/6    S+   22:07   0:00      |                   \_ /bin/sh -l
xor        21398  0.2  0.0  20060  5756 pts/7    Ss   22:08   0:00      \_ -bash
xor        21422  0.0  0.0  22424  4868 pts/7    R+   22:08   0:00          \_ ps auxf

.
.

container2_pid=21377
```

### Networking

Start a new console on the host side and configure networking.

```bash
sudo ip link add con2-host type veth peer name con2-ns
sudo ip link set con2-ns netns ${container2_pid}
sudo ip link set con2-host master minidocker-br
sudo ip link set con2-host up
```

Back to the container shell, complete the network configuration:

```bash
ip addr add 172.17.10.11/24 dev con2-ns
ip link set con2-ns up
ip route add default via 172.17.10.1
```

## Testing isolation and connectivity

With both containers up and running, we can now validate that the environment behaves as expected. The goal here is not only to confirm connectivity, but also to observe how isolation mechanisms (network, filesystem, and process model) behave in practice.

We start with basic network communication. Each container should be able to reach the other as well as access external resources through the host NAT configuration.

<img class="screenshot" src="{{ "/assets/images/2026-05-22-containers-from-scratch/01-network.png" | relative_url }}" alt="load">

Next, we verify that the container provides a usable user space environment. We install a simple package and create some files inside the container. This confirms that the filesystem is writable and that package management works correctly within the isolated environment.

<img class="screenshot" src="{{ "/assets/images/2026-05-22-containers-from-scratch/02-creating-file.png" | relative_url }}" alt="load">

<img class="screenshot" src="{{ "/assets/images/2026-05-22-containers-from-scratch/03-installing-app.png" | relative_url }}" alt="load">

From the host perspective, it is interesting to observe how these changes are reflected in the filesystem. Since the container is built using OverlayFS, all modifications should appear in the writable layer (upperdir) while the base image remains untouched.

<img class="screenshot" src="{{ "/assets/images/2026-05-22-containers-from-scratch/04-filesystem-tree.png" | relative_url }}" alt="load">

Finally, we can stress the container by running CPU-intensive workloads to see how the configured cgroup limits behave.

<img class="screenshot" src="{{ "/assets/images/2026-05-22-containers-from-scratch/05-testing-load.png" | relative_url }}" alt="load">

## Cleanup

After all tests are complete, we should clean up all created resources.

First, terminate all processes running inside the container cgroups. Once the processes are stopped, we can unmount the OverlayFS filesystems, remove the cgroup hierarchy, and finally delete the container directories.

```bash
# Kill container processes
echo 1 | sudo tee /sys/fs/cgroup/minidocker.slice/container1/cgroup.kill
echo 1 | sudo tee /sys/fs/cgroup/minidocker.slice/container2/cgroup.kill

# Unmount container OverlayFS
sudo umount -v /tmp/container1/merged
sudo umount -v /tmp/container2/merged

# Remove container cgroups
sudo rmdir -v /sys/fs/cgroup/minidocker.slice/container1
sudo rmdir -v /sys/fs/cgroup/minidocker.slice/container2
sudo rmdir -v /sys/fs/cgroup/minidocker.slice

# Remove container files
sudo rm -rf /tmp/container1
sudo rm -rf /tmp/container2
```

Finally, remove the iptables rules, delete the network bridge, and disable packet forwarding.

```bash
# Disable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=0

# Remove NAT for container network
sudo iptables -t nat -D POSTROUTING -s 172.17.10.0/24 -o enp0s3 -j MASQUERADE
sudo iptables -D FORWARD -i minidocker-br -o enp0s3 -j ACCEPT
sudo iptables -D FORWARD -i enp0s3 -o minidocker-br -m state --state RELATED,ESTABLISHED -j ACCEPT

# Remove network bridge for containers
sudo ip link delete minidocker-br
```

## Conclusion

As a final step, I prepared a set of scripts that automate everything described in this post. They are available in the accompanying [GitHub repository](https://github.com/kazikb/containers-from-scratch). After manually assembling these building blocks, it becomes much easier to understand what tools like Docker or Podman actually do under the hood.

While the abstractions provided by these tools are extremely valuable, they do not eliminate the need to understand what happens underneath. When things stop working as expected, understanding the underlying mechanisms becomes irreplaceable.

The talks and write-ups referenced earlier are excellent starting points, but hands-on experimentation provides a level of intuition that is difficult to gain otherwise.

## Follow-up

**Update, July 2026:** I wrote a follow-up article that checks the missing security layers in this shell-based container: user namespaces, capabilities, seccomp, and AppArmor.

Read it here: [Why My Container From Scratch Is Not a Secure Runtime](/linux/containers/2026/07/04/why-my-container-from-scratch-is-not-a-secure-runtime.html)

## Sources

- [https://www.youtube.com/watch?v=4RUiVAlJE2w](https://www.youtube.com/watch?v=4RUiVAlJE2w)
- [https://michalpitr.substack.com/p/linux-container-from-scratch](https://michalpitr.substack.com/p/linux-container-from-scratch)
- [https://man7.org/linux/man-pages/man7/cgroups.7.html](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [https://man7.org/linux/man-pages/man7/namespaces.7.html](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [https://man7.org/linux/man-pages/man7/network_namespaces.7.html](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
- [https://man7.org/linux/man-pages/man7/pid_namespaces.7.html](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)
- [https://man7.org/linux/man-pages/man7/uts_namespaces.7.html](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)
- [https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html](https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)
- [https://man7.org/linux/man-pages/man8/mount.8.html](https://man7.org/linux/man-pages/man8/mount.8.html)
- [https://man7.org/linux/man-pages/man8/ip.8.html](https://man7.org/linux/man-pages/man8/ip.8.html)
- [https://man7.org/linux/man-pages/man8/ip-link.8.html](https://man7.org/linux/man-pages/man8/ip-link.8.html)
- [https://man7.org/linux/man-pages/man1/unshare.1.html](https://man7.org/linux/man-pages/man1/unshare.1.html)
- [https://man7.org/linux/man-pages/man1/nsenter.1.html](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [https://docs.kernel.org/filesystems/overlayfs.html](https://docs.kernel.org/filesystems/overlayfs.html)
- [https://github.com/kazikb/containers-from-scratch](https://github.com/kazikb/containers-from-scratch)
