---
title: "Namespace and Mount-Based Sandboxing"
---

these notes came out of working through pwn.college's sandboxing challenges, that's why a lot of the explanations are shaped the way they are: each pitfall below is something i actually had to reason through and use, not just theory copied from a manual.

## namespaces

- **The Global View vs. The Local View:** Namespaces determine what system resources a process can "see." Without namespaces, every process sees the global state of the system.

- **PID Namespace:** Hides processes from each other. A process inside a new PID namespace thinks it is PID 1.

- **NET Namespace:** Provides private routing tables, firewall rules, and IP addresses.

- **MNT Namespace:** Gives a process its own independent "tree" of mounts. Changes here don't affect the host.

- **UTS Namespace:** Allows setting a private hostname and domain name for the sandbox.

- **USER Namespace:** Maps a "Root" user inside the container to a "Normal User" outside, preventing actual host root access if the sandbox is breached.

## filesystem isolation

### chroot

- **The `chroot` (Change Root) System Call**
    
    - _Concept:_ Changes the apparent root directory (`/`) for a current process and its children.

    - _Limitations:_ `chroot` was not built for security. It is easily bypassed if the process has certain privileges or file descriptors left open.


### pivot_root

`pivot_root` allows us to change the root file system of the current process (and all processes in its namespace) to a new directory, while simultaneously stowing the old root away in a different location.

To do this, we need to have a mounted folder:

```bash
mkdir /tmp/new_root
mount --bind /tmp/new_root /tmp/new_root
```

Inside the new root, create a directory where the current host system will be tucked away:

```bash
mkdir /tmp/new_root/old_root
```

Now it's time to swap. After we `cd` into `/tmp/new_root`, we execute:

```bash
pivot_root . old_root
```

Finally, we must clean up to ensure the old root is completely detached:

```bash
umount -l /old_root
rmdir /old_root
```

Now, all the processes in the namespace see `/tmp/new_root` as the absolute `/` directory.

> **Crucial Note:** The Linux kernel almost never lets you do this in the default namespace. You must unshare the Mount namespace first.

### chroot vs pivot_root

#### 1. The "Pointer" vs. The "Mount"

In the Linux kernel, every process has a `struct fs_struct` that contains a field called `root`.

- **When you `chroot`:** The kernel simply updates that `root` pointer for that process to a new directory. It's like giving someone a pair of binoculars that only lets them see one room. The rest of the house is still there; they just can't see it through the lenses.
    
- **When you `pivot_root`:** You are physically moving the furniture. You take the "Mount" (the actual disk partition) that was at `/` and move it to `/old_root`, then take a different "Mount" and put it at `/`.
    

#### 2. Why the distinction matters

Because `chroot` is just a per-process pointer, it is **inherited** but not **global**.

1. **Inheritance:** If a chrooted process starts a child, that child is also chrooted.
    
2. **No Isolation:** If you `chroot` Process A, Process B (the host) can still see everything Process A is doing. More importantly, Process A can still "see" the host's mount points if it knows how to look (e.g., via `/proc` or by using the `fchdir` breakout).
    

With `pivot_root`, because it happens at the **namespace** level:

- You can safely tuck away the host system into an "Old Root" mount point.
    
- You can **unmount** the old root entirely.
    
- Once unmounted, that data is **physically unreachable** for the process. There is no pointer in the world that can take you back to the host's files because they aren't in your "namespace universe" anymore. This makes `pivot_root` significantly more secure than `chroot`.
    

## common pitfalls when using pivot_root

### 1. Failing to Unmount the Old Root

If you `pivot_root` but forget to unmount the old directory, you haven't actually restricted access; you've just moved the host filesystem to a new folder. An attacker can simply traverse into the old root and read sensitive files.

> turns out `pivot_root` on its own doesn't remove anything, it just relocates it. if the old root isn't explicitly unmounted afterward, it's sitting right there as an ordinary directory, fully readable.

### 2. The Procfs Portal & PID 1 Root Access

Even if a sandbox is seemingly secure, `procfs` exposes running processes in a way that lets you access their view of the filesystem. If you can mount `procfs`, `/proc/1/root` acts as a direct portal back to the actual root of the main container or host environment.

> the takeaway: `procfs` breaks isolation almost by design, since its whole purpose is exposing process internals. if you're able to mount it yourself, `/proc/1/root` hands you a direct symlink back to whatever the host's actual root is, permissions on that path aside, the portal itself doesn't care about your jail.

### 3. Leaked External File Descriptors

If a process is allowed to inherit an open file descriptor (FD) that points outside the jail (e.g., FD 3 attached to `/`), filesystem isolation is entirely bypassed. You can use shellcode with `openat` starting from that external FD, and then use `sendfile` to push the data directly to `stdout`.

> this is the same file-descriptor lesson as chroot, just at the namespace level too: inheriting an open handle to somewhere outside the jail makes every filesystem restriction irrelevant, `openat` against that handle just ignores the boundary entirely.

### 4. Unrestricted Mount Privileges in Bind Mounts

If you are dropped into a jail with only specific directories bind-mounted in, but you retain the capability to `mkdir` and `mount`, you can build your own escape route. By creating a new directory, mounting `procfs` to it, and reading from `/proc/1/root/flag`, you can reach back out to the real root.

> what stuck with me: being confined to a few bind-mounted directories means nothing if you still have permission to `mkdir` and `mount`. mounting `procfs` yourself from inside a restricted jail rebuilds the exact same portal back to the real root that unrestricted access would have given you.

## mount magic

### what is mounting?

**Mounting** is the process of attaching a storage device (like a hard drive, USB stick, or network share) to a specific directory in a tree so the system can actually talk to it. **Unmounting** is the clean "handshake" where you tell the system to finish writing data and let go of the device so it can be safely removed.

Mounting is like making a side door to a directory. If we mount `/bin` to `/tmp/bin`, `/tmp/bin` acts exactly like `/bin`. Changes in one reflect in the other. If you mount over a directory that already has files, those underlying files disappear until you unmount the "side door."

**Why is this necessary?**

1. **Support different formats:** Seamlessly integrate Windows-formatted (NTFS) drives and Linux-native (ext4) drives in one tree.

2. **Security:** Mount a drive as "Read-Only," locking the gate so even malware cannot delete files.

3. **Stability:** Unmounting forces background "lazy writing" to finish, preventing data corruption.


#### Pseudo-Filesystems

Mounting goes further than just physical disks. In Linux, you can mount **"pseudo-filesystems"** that only exist in RAM. You always need a "folder" (the mount point) to see the info, but you don't need a physical "device."

|Mount Point|What it actually is|
|---|---|
|`/proc`|A "window" into the **Kernel's brain**. When you read files here, the kernel is generating text on the fly.|
|`/dev`|A list of your **Hardware**. Files here represent your mouse, keyboard, disks, etc.|
|`/sys`|A way to talk to **Drivers**. You can configure hardware (like screen brightness) by writing to these files.|

Because `/proc` is a **Virtual Filesystem**, when you mount it, you are telling the Kernel: _"Take your internal status reports and map them to this folder so I can read them like files."_

```bash
# 1. Create the empty folder (the "portal")
sudo mkdir /my_kernel_info

# 2. Mount the 'proc' type to that folder
sudo mount -t proc proc /my_kernel_info
```

#### Real-World Example: Arch Linux Installation

During installation, you manually construct the mount tree:

1. `mount /dev/sda3 /mnt`: Mounts the main partition to `/mnt`.
2. `mkdir /mnt/boot`: Prepares the boot directory.
3. `mount /dev/sda1 /mnt/boot`: Attaches the tiny boot partition exactly on top of the `/mnt/boot` folder so critical startup files are routed directly to the motherboard's expected location.
4. `genfstab -U /mnt >> /mnt/etc/fstab`, then `arch-chroot /mnt`.

#### Advanced Mount Concepts

- **What is a Mount Point?:** The folder acting as the gateway to the mounted filesystem.
    
- **Bind Mounts (`mount --bind`):** Mirroring a directory into another location without needing a separate partition.
    
- **Propagations:** Shared, Private, and Slave mounts dictate if mount events inside a namespace propagate back to the host.
    
- **The `rootfs` (Root Filesystem):** The actual directories and binaries packed inside the container.
    
- **Layered Filesystems (OverlayFS):** How Docker "stacks" changes so containers can modify files without touching the original read-only image.


## resource control (cgroups)

- **Control Groups (v1 vs v2):** Used for limiting hardware usage like CPU, RAM, and Disk I/O for a specific namespace.

- **The OOM Killer:** Defines what happens when the "jail" runs out of its allocated memory limit (the Out-Of-Memory killer terminates processes).


## Putting it All Together: The "Container"

- **The Workflow:** 1. Unshare Namespaces
    
    2. Setup Mounts
    
    3. Pivot Root
    
    4. Exec the process.
    
- **Security Hardening:** Combine namespaces with Capabilities (dropping root privileges), Seccomp (filtering syscalls), and AppArmor/SELinux (mandatory access control).


## Conclusion: The Sandbox Illusion

Just like with Seccomp, filesystem sandboxing relies heavily on preventing context and state leaks. Relying purely on `pivot_root` or `chroot` creates an illusion of security that is easily shattered if the environment is not meticulously sanitized.

A few critical lessons stand out from working through these:

1. **Incomplete Cleanup is Fatal:** Moving the root is meaningless if the old root is left mounted. If you don't aggressively `umount -l` the host filesystem, it remains a heavily populated folder sitting right next to the user.

2. **Pseudo-Filesystems are Skeleton Keys:** `procfs` is designed for transparency, making it the enemy of isolation. Allowing a sandboxed process the privileges to mount pseudo-filesystems guarantees they will access `/proc/1/root` to reach the physical host layer.

3. **File Descriptors Transcend Paths:** Sandboxes restrict _path resolution_, not kernel file handles. If an external directory FD is leaked into the sandbox via `fork/exec`, an attacker doesn't need to break out of the filesystem tree; they just ask the kernel to resolve their `openat` queries starting from the outside handle.

True containerization requires a perfect symphony. A container isn't a physical box; it's just a regular Linux process where Namespaces lie to it about its surroundings, Cgroups restrict its diet, and Seccomp tapes its mouth shut. If even one of those lies fails, the sandbox collapses.
