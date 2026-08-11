---
title: "chroot and Seccomp Sandboxing"
---

these notes came out of working through pwn.college's sandboxing challenges, that's why a lot of the explanations are shaped the way they are: each pitfall or bypass technique below is something i actually had to reason through and use, not just theory copied from a manual.

# chroot

## What is chroot?

The `chroot()` system call is a feature in Unix-like operating systems that changes the root directory (`/`) for the calling process and its children. After invoking `chroot()`, all absolute filesystem paths used by the process are interpreted relative to the new root directory (commonly called the "chroot jail"). This mechanism is used for process isolation, software testing, system recovery, and as a primitive form of sandboxing.

For example, if `chroot()` is called with `/tmp/jail`:
- The process will see `/tmp/jail` as its filesystem root.
- Paths such as `/etc/passwd` will resolve to `/tmp/jail/etc/passwd` in the actual filesystem.

However, `chroot()` only changes the apparent root. It does **not** inherently provide strong security isolation on its own, and there are important caveats and pitfalls to its correct use.

---

## Common Pitfalls When Using chroot

### 1. Retaining the Current Working Directory Outside the Jail

Although `chroot()` redefines the filesystem root, it does **not** modify the process's current working directory (`cwd`). This means that if the process's working directory is **outside** the new root at the time `chroot()` is called, the process will retain access to files outside of the intended jail via relative paths or open directory handles.

**Example:**

Suppose your process starts in `/home/user`. You run:
```bash
cd /home/user
sudo chroot /tmp/jail
```
Now the process is in `/tmp/jail`, but the cwd is still `/home/user` (which **does not exist** inside the jail). The process can reference files via relative paths that resolve outside the jail, as long as it keeps this cwd.

**Remedy:** Immediately call `chdir("/")` after `chroot()`, or ensure the cwd is inside the new root before chrooting.

> turns out this isn't just a cwd problem, it's a general lesson about trusting input. if the program that does the chrooting doesn't validate the path you hand it (or the path your own shellcode constructs), stacking enough `../` walks you straight back out, no exotic syscalls needed. the simplest possible jail break is just: the code never checked.

---

### 2. File Descriptor-based Escape: openat(), execveat(), and dirfd

Modern Unix-like systems provide system calls such as `openat()` and `execveat()`, which allow a process to operate relative to an **open directory file descriptor** (`dirfd`). If a process retains a directory file descriptor (opened before `chroot()` or from the cwd outside the jail), it can use these syscalls to access files *outside* the chroot jail, bypassing the intended isolation.

**Example in C:**

```c
int fd = open("/", O_DIRECTORY); // Open root before chroot
chroot("/tmp/jail");
chdir("/");
int file = openat(fd, "etc/passwd", O_RDONLY); // This accesses the real /etc/passwd!
```

Or with `execveat()`:
```c
int fd = open("/", O_DIRECTORY);
chroot("/tmp/jail");
chdir("/");
execveat(fd, "bin/bash", ...); // Runs /bin/bash from the real root
```

**Remedy:** Always close all file descriptors to directories outside the jail before or immediately after calling `chroot()`.

> the real lesson here: a chroot is only as strong as the file descriptors that existed before it. if something handed the process an FD to the real root before locking it down, `openat` against that FD walks straight past the jail. this doesn't need any path tricks at all, just a directory handle that was never meant to survive the transition.

---

### 3. Repeated or Nested chroot() Pitfall

Calling `chroot()` multiple times within a process, especially from within an already chrooted environment, can introduce subtle and dangerous issues:

- If a process chroots into one jail, then (inside that jail) performs another `chroot()` to a different directory (which may point to a location within the previous jail or elsewhere, depending on jail setup), this may enable escalation or escape scenarios.
- Some chroot implementations contain exploitable bugs when chroot is called repeatedly, particularly by root-privileged processes.
- If privileges are not dropped after the first chroot, nested or repeated chroot calls can potentially allow a process to "walk out" of a jail, for example, by chrooting to a directory mounted or available inside the current jail that actually maps outside.

**Example:**

Suppose the initial jail is `/tmp/jail`. Inside `/tmp/jail`, you have a symlink or bind-mount to `/` of the real filesystem as `/tmp/jail/escape`. From within the chrooted environment, you now execute:
```c
chroot("/escape");
chdir("/");
```
If `/escape` is actually a mount pointing to the real root, you have effectively escaped out of the original jail.

**Remedy:** Only perform `chroot()` once per process, and always immediately drop root privileges after the operation. Never allow untrusted code to run with the ability to call `chroot()` a second time.

---

# seccomp

## What is Seccomp?

**Seccomp** (Secure Computing mode) is a Linux kernel security feature that restricts the system calls a process can make. Instead of isolating the _filesystem_ (like `chroot`), Seccomp reduces the process's access to the _kernel attack surface_.

In modern implementations (Seccomp-BPF), Berkeley Packet Filter (BPF) rules are used to examine syscall numbers and their arguments before the kernel executes them. If a process attempts a disallowed system call, the kernel can immediately terminate the process (SIGSYS), return a spoofed error, or trigger a trap. However, if the filter is too permissive, fails to account for alternative architectures, or ignores side channels, the sandbox can be broken.

---

## Common Bypass Techniques and Pitfalls

### 1. Exploiting Permissive Whitelists (openat & sendfile)

A sandbox is only as strong as its tightest restriction. If a sandbox allows `openat`, it permits opening files relative to an existing directory file descriptor (FD). If a process is given an FD that points to the real system root (e.g., FD 3 pointing to `/`) before the sandbox is locked down, `openat(3, "flag", 0)` tells the kernel to bypass the current working directory and resolve "flag" starting from the real root.

Furthermore, if standard `read` and `write` are blocked to prevent data exfiltration, `sendfile` can often bypass this. `sendfile` operates entirely within kernel space, directly copying data from one file descriptor (the opened flag) to another (stdout, FD 1), effectively dumping the file without ever needing to read it into user-space memory.

> the takeaway: a syscall whitelist doesn't mean much on its own, it's only as safe as the state that already exists when those syscalls run. an allowed `openat` combined with a leftover FD, plus `sendfile` to move the data out without ever touching `read`/`write`, was enough by itself. tightening the syscall list further wouldn't have mattered if that FD was already there.

---

### 2. Cross-Directory Linking with linkat()

Sometimes a sandbox restricts direct `open` calls on specific sensitive paths (like `/flag`) but leaves other filesystem metadata syscalls intact. `linkat` creates a hard link, a new directory entry that points to the exact same underlying inode (data) on the disk.

By calling `linkat(3, "flag", -100, "my_flag", 0)`, you instruct the kernel to use the external directory handle (FD 3) to find "flag", and create a new hard link to it named "my_flag" inside your current, permitted directory (`-100` or `AT_FDCWD`). The kernel checks permissions at the time of linking, but once the link is created, "my_flag" is just a local file in your allowed jail. You can then `open` and read it normally, as the path restrictions no longer apply to the new name.

> worth remembering: blocking `open` on a specific path doesn't block every syscall that can reach it. `linkat` only checks permissions at the moment the link is created, after that, the new name is just a normal file inside your allowed directory, with none of the original path's restrictions attached to it.

---

### 3. Current Working Directory (CWD) Manipulation via fchdir()

While `chroot` changes the perceived root of the filesystem, it does not automatically change the Current Working Directory (CWD). Conversely, `fchdir` changes the process's CWD to the location of an open directory file descriptor.

If you possess a file descriptor pointing outside the jail (like an FD to `/`), calling `fchdir(FD)` physically moves your process's CWD to that external directory. Because relative path lookups (like `open("flag", O_RDONLY)`) start from the CWD, the kernel resolves the path in the outside filesystem, entirely ignoring the `chroot` jail boundaries.

> this taught me that `chroot` changing the root and `chdir` changing the cwd are two completely separate operations, and a filter that only thinks in terms of paths forgets that. `fchdir` moves your cwd using a file descriptor instead of a path, so it sidesteps path-based restrictions entirely, once your cwd is outside, every relative lookup after that just works.

---

### 4. The "Double Chroot" or Recursive Escape

This is a classic kernel logic abuse relying on how the OS tracks directory roots. The escape requires the ability to create a directory and call `chroot`.

1. You create a new directory (`mkdir("escape_pod")`).

2. You `chroot("escape_pod")` into it.

Crucially, you _do not_ change your CWD into the pod first. This means your new root is inside the pod, but your CWD is technically _outside_ of it. The kernel's path resolution logic dictates that if a process tries to traverse up (`..`) and hits its filesystem root, it stops. However, because your CWD is outside the newly defined root, the kernel never hits the boundary check. Repeatedly calling `chdir("..")` forces the kernel to walk all the way up the directory tree to the real, physical system root, escaping the sandbox entirely.

> what stuck with me: chroot only isolates you as long as your cwd is actually inside the new root. if you chroot into a directory you just created without first `chdir`ing into it, your cwd stays outside the jail you built, and repeated `chdir("..")` walks you all the way up to the real root. the fix isn't really about the chroot call itself, it's about never separating the root swap from actually moving into it.

---

### 5. File Descriptor Injection (External Handles)

When a binary strictly locks down its environment and doesn't explicitly leave any "handles" (like an open directory FD) for you to exploit, you can inject one from the outside.

In Linux, file descriptors are inherited by child processes unless explicitly flagged with `O_CLOEXEC` (close-on-exec). By running a shell command like `(exec 3</ /challenge_binary)`, the shell opens `/` and assigns it to FD 3, then executes the binary. The binary inherits FD 3. Once the binary drops into its sandbox, it unwittingly possesses a valid, open handle to the outside filesystem, which can then be used with `openat` or `fchdir`.

> the actual insight: when a binary doesn't leave you any handle to abuse, the shell running it can hand you one anyway. file descriptors survive `exec` unless explicitly marked close-on-exec, so opening something from the shell before launching the sandboxed binary quietly smuggles that access in with it.

---

### 6. Architecture Confusion (x86_32 vs. x64)

Seccomp-BPF rules often check the syscall _number_ to determine if an action is allowed. However, syscall numbers vary drastically between CPU architectures. On 64-bit Linux (x86_64), syscall `5` is `fstat`. On 32-bit Linux (x86_32), syscall `5` is `open`.

If the Seccomp filter strictly whitelists syscall `5` thinking it's safely allowing `fstat`, but forgets to check the architecture flag (`AUDIT_ARCH_X86_64`), you can switch the CPU's execution mode. By using the 32-bit interrupt instruction (`int 0x80`) instead of the 64-bit `syscall` instruction, the kernel interprets the registers using the 32-bit table. This allows you to execute powerful syscalls like `open`, `read`, and `write` while the sandbox thinks you are executing harmless file stat queries.

> what i took from this: filtering by syscall number without checking the architecture is a real gap, not a theoretical one. the same number means a completely different syscall depending on whether you entered through the 64-bit or 32-bit calling convention, so a filter that allows number 5 thinking it's one thing can be silently allowing something else entirely.

---

### 7. Side-Channel: Data Leakage via Exit Status

**How it works:** When standard output methods are entirely blocked, you have to extract data indirectly. If `read` and `exit` are allowed, you can use the process's exit code as a communication channel.

The shellcode reads the target file into memory, isolates a single byte (e.g., the first character of the flag), and passes that exact byte's numerical value as the argument to the `exit(status)` syscall. A parent script outside the sandbox monitors the child process, catches the exit status code, and converts that number back into an ASCII character. Repeating this loop by incrementing the read offset reconstructs the entire file.

> the takeaway: with almost nothing allowed, the exit code itself becomes a channel. if `read` and `exit` are all you get, you can still leak one byte at a time by reading it into a register and exiting with that value as the status, an outside process just has to be watching for it.

---

### 8. Side-Channel: Timing-Based Inference

**How it works:** If `exit` is blocked or restricted but `nanosleep` is allowed, time becomes the exfiltration channel. This requires breaking the data down to the binary level (bits).

The shellcode reads a byte into memory and isolates a specific bit using bitwise shifts.

- If the bit is `1`, the shellcode triggers `nanosleep` for a noticeable duration (e.g., 0.5 seconds).

- If the bit is `0`, it deliberately crashes or finishes instantly.

An external script times how long the process lives. A long lifespan equals a `1`; a short lifespan equals a `0`. By iterating through all 8 bits of every byte, you can fully rebuild the file purely by observing how long the kernel keeps the process alive.

> turns out even without exit codes, time itself leaks information. sleeping conditionally on a single bit and measuring how long the process stayed alive from the outside is slow, but it's a fully working channel with nothing more than `nanosleep`.

---

### 9. Side-Channel: Error Signals and Deliberate Faults

**How it works:** When you only have the `read` syscall and absolutely no outward communication, you can use memory access violations (like a Segmentation Fault) as a boolean true/false signal.

The shellcode reads a byte of the flag and compares it to a hardcoded "guess" character.

- If the guess is correct, the shellcode intentionally does something illegal, like attempting to read from a NULL pointer (`0x0`), which forces the kernel to kill the process with a `SIGSEGV`.
    
- If the guess is wrong, the shellcode loops infinitely or exits cleanly.
    

An outside script tries every possible character (brute-force). The character that causes the process to crash is the correct byte.

> worth remembering: a crash is also a signal. with only `read` available, comparing a guessed byte and deliberately dereferencing null on a match turns a segfault into a boolean answer, brute-forcing every possible byte value this way is slow but it works with almost no syscalls at all.

---

### 10. Breaking Process Isolation (IPC Abuse)

**How it works:** Some sandboxes isolate a worker (child) process and use an un-sandboxed parent process to handle specific, privileged operations via Inter-Process Communication (IPC).

If the communication protocol isn't strictly validated, the child can send crafted messages to trick the parent. For instance, sending a `read_file` command with a target path forces the privileged parent to load the target into its own memory. Following up with a `print_msg` command manipulates the parent into writing its buffer back to you. The exploit works because you aren't escaping the sandbox yourself; you are weaponizing the privileged broker that exists outside of it.

> the real lesson: you don't have to escape a sandbox if something outside it will do the work for you. if a privileged parent process trusts messages from its sandboxed child without validating them carefully, you can just ask it to read the file and hand the contents back, the sandbox around the child was never the actual boundary.

## Conclusion: The Sandbox Illusion

At the end of the day, Seccomp is a surface-area reduction tool, not a magic bullet. In both CTFs and real-world vulnerability research, the failure points almost never involve finding a zero-day in the Linux kernel itself. Instead, the actual "problems" in modern sandboxing boil down to logic gaps left by the developers who implemented the filters.

Looking at the bypass techniques above, a few core themes stand out:

- **Context and State Leaks:** Whitelisting syscalls like `openat` or `sendfile` might seem necessary for basic program functionality, but developers often forget about the surrounding state. If a file descriptor pointing outside the jail is accidentally inherited, leaked, or injected, the sandbox is useless. The kernel rigidly enforces the BPF rules, but it doesn't know that FD 3 was never supposed to be there in the first place.

- **The Architecture Blindspot:** Relying solely on the syscall number without explicitly validating the architecture (`AUDIT_ARCH`) is a fatal, yet incredibly common, mistake. It proves a basic security concept: if you don't verify _how_ a syscall is being executed (32-bit vs 64-bit), you can't trust what it's trying to do.

- **Side Channels are Unstoppable:** Probably the biggest headache for defenders is realizing that an attacker doesn't need a `write` syscall or a network connection to exfiltrate data. As long as a process can execute and eventually terminate, it can communicate. Whether it's timing execution delays, throwing deliberate segfaults, or manipulating exit codes, an attacker will always find a way to leak a flag bit by bit.

Ultimately, building a secure Seccomp profile isn't just about compiling a strict whitelist of "safe" syscalls. It requires a deep understanding of filesystem edge cases, CPU architecture quirks, and the reality that attackers will weaponize absolutely anything, even errors and time, to break out.
