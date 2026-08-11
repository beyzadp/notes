---
title: "9. File System Operations: Bridging Raw Storage and User Space"
---

part of the [[index|os-development]] series.

A hard drive is just a massive array of dumb bytes. It has no concept of files, folders, boundaries, or names. The file system is the translator that imposes structure on this chaos. In our OS, we built a custom, flat file system (`MYFS`) to act as this translator, handling the journey from a human-readable string like `"suvari.txt"` down to physical 512-byte blocks on a VirtIO disk.

The core problem is mapping variable-length files into fixed-size hardware sectors, and then safely transferring that data across the strict hardware trust boundary between the kernel and the user shell.

## The MYFS Architecture: Linked Allocation

To structure the raw disk, `MYFS` divides the storage into three distinct zones. It relies on a linked-list approach to file storage, ensuring that files can grow without needing contiguous space.

- **The Anchor (Sector 0 - Superblock):** The very first sector holds the `superblock`. This contains the magic signature (`"MYFS"`), the total disk size, and the index boundaries indicating where the directories end and the data begins. If this sector is corrupted or missing, the OS formats the disk.
    
- **The Catalog (Sectors 1 to 4 - Directory Blocks):** These blocks contain `dir_entry` structures. Each entry acts as a file's ID badge, holding a boolean `in_use` flag, the 19-character filename, the total file size, and crucially, the `start_block` index.
    
- **The Payload (Sectors 5+ - Data Blocks):** The actual file contents live here. Because a file might be larger than a single 512-byte sector, we reserve the last 4 bytes of every sector as a `next_block` pointer. This leaves 508 bytes (`DATA_SIZE`) for actual text.
    

## The Scavenger Hunt (Read Logic)

When the kernel is asked to read a file, it acts as a detective reconstructing a shredded document.

First, it executes a **Directory Scan**. It loops through the directory sectors, parsing every 32-byte `dir_entry` until `strcmp` finds a match for the requested filename. If found, it extracts the `start_block`.

Next, it begins the **Block Traversal**. The file system jumps to the `start_block`, reads the sector via the `virtio-blk` driver, and copies the 508 bytes of payload into a buffer. It then reads the 4-byte pointer at the end of the sector to find the next hop. This cycle repeats, hopping from block to block, until the pointer reads `END_OF_FILE (0xFFFFFFFF)`.

## The Virtual Memory Barrier & Bounce Buffers

Reconstructing the file in the kernel is only half the battle; the data must then be handed to the user process.

Because the shell runs with virtual memory enabled, the kernel cannot blindly copy the disk data into a memory pointer provided by the user. If the user passes a pointer to unmapped memory (like a dynamic stack variable), the hardware's Memory Management Unit (MMU) will instantly trigger a `Load Page Fault (scause=0000000d)`.

To solve this, we use the **Bounce Buffer Strategy**:

The kernel never does I/O directly into a user pointer. Instead, it reads the file from the disk into a safe, isolated `static` array inside kernel space. Once the file is fully assembled, the kernel uses a hardware privilege override to carefully "bounce" the data into the user's mapped `.bss` memory.

## The File Read Pipeline

Here is the exact sequence of events from the moment the user types `cat` to the text appearing on screen.

1. **User code initiates request:** Runs in U-Mode.
    
    The shell isolates the filename and passes it, along with a statically allocated `.bss` buffer, to the `SYS_READFILE` wrapper, triggering an `ecall`.
    
2. **Kernel secures the parameters:** CPU transitions to S-Mode.
    
    The kernel catches the trap. It temporarily flips the `SUM` (Supervisor User Memory) bit in the `sstatus` CSR to bypass the hardware privilege lock, copying the requested filename into a secure kernel string (`k_filename`), then immediately disables `SUM`.
    
3. **VirtIO Disk I/O:** Hostile data assumption.
    
    The file system driver (`fs_read_file`) takes over, asking the VirtIO device to fetch the directory sectors and the linked data blocks. The assembled text is written into an isolated kernel bounce buffer (`k_data_buf`).
    
4. **Memory Transfer:**
    
    If the file was found, the kernel flips the `SUM` bit back on. It executes a `memcpy` to push the text from the secure kernel bounce buffer into the user's destination buffer. `SUM` is disabled immediately after.
    
5. **Return to User:** Hardware drops back to U-Mode.
    
    The kernel writes the number of bytes read into `a0`, restores the user's trap frame, and executes `sret`. The shell resumes and prints the populated buffer.
    

> [!INFO]
> 
> **The SUM Bit Reality Check:** Flipping the `SUM` bit (Bit 18 of `sstatus`) allows the kernel to access memory marked with the User (`PAGE_U`) flag. However, it only bypasses _privilege_ checks, not reality checks. The destination buffer in user space must still physically exist and be mapped in the page table (e.g., as a global/static variable in the `.bss` segment), or the MMU will still fault.

## Host vs. Guest: The Disk Injection Problem

When building an operating system, there is a strict physical boundary between the **Host** (your development machine) and the **Guest** (your custom kernel running in QEMU).

Because the Guest uses a custom file system (`MYFS`), it cannot simply mount or read the host's native directory structure. To the Guest, the hard drive (`disk.img`) is just a raw, unformatted sequence of bytes attached to a VirtIO bus.

Initially, this creates a massive development bottleneck: how do you put a file like `suvari.txt` onto the Guest's disk? You cannot just copy it into a folder. You would theoretically have to manually write the file's bytes, calculate the sector offsets, update the `MYFS` superblock, and manually link the data blocks using a hex editor. This is a messy, unmaintainable approach.

> [!INFO]
> 
> **The Bare Metal Reality Check:** It is easy to assume we need a bridging tool because we are running in a virtual machine (QEMU), but the true barrier is the custom file system itself. If you were booting this OS on a physical RISC-V motherboard with a real USB drive, you would still need these tools. Your native Linux machine has built-in drivers for standard file systems like FAT32 or ext4, but it has no idea how to read or write your custom `MYFS` structure. The tooling exists to translate standard files into your OS's native storage language, regardless of whether the destination is a virtual `disk.img` or a physical thumb drive.

To solve this, we treat the file system like a compilation step. Just as we compile `.c` files into `.elf` executables, we must "compile" a standard host directory (`disk/`) into a raw `MYFS` binary image (`disk.img`) before booting the OS.

#### The Tooling Solution: `mkfs` and `extractfs`

To bridge the gap between the host and the guest, we built two standalone C programs in the `tools/` directory. Crucially, **these tools do not run in the custom OS.** They run on the host Linux/macOS machine, allowing them to use the standard C library (`<stdio.h>`, `<dirent.h>`) to manipulate files.

- **The Injector (`mkfs.c`):** This tool creates the raw `disk.img` file. It formats the first sector with the `MYFS` superblock, zeros out the directory sectors, and marks all data blocks as free. It then uses the host's `opendir()` to scan the local `disk/` folder. For every file it finds (like `suvari.txt`), it automatically finds a free directory entry, splits the file into 508-byte chunks, links the blocks, and writes them into the image.
    
- **The Extractor (`extractfs.c`):** This is the reverse pipeline, used primarily for debugging. It parses a compiled `disk.img` file, reads the custom `MYFS` directory entries, reconstructs the linked data blocks, and dumps the files back out to the host file system.
    

#### The Orchestration Pipeline

This entire process is automated by the `Makefile`, ensuring the disk image is always synchronized with the `disk/` folder before QEMU boots. Here is the exact sequence of events triggered when you run `make run`:

1. **Dual Compilation:**
    
    The Makefile uses two completely different compilers. It uses the cross-compiler (`CC = clang --target=riscv32-unknown-elf`) to build the kernel and shell, but it uses the native host compiler (`HOST_CC = gcc`) to compile `tools/mkfs.c` into an executable utility on your local machine.
    
2. **Blank Canvas Generation:**
    
    The Makefile uses the `dd` command (`dd if=/dev/zero of=$@ bs=512 count=2048`) to generate a 1MB file (`disk.img`) filled entirely with zeros. This represents our blank, unformatted hard drive.
    
3. **Disk Formatting and Injection:**
    
    The Makefile executes the newly built host tool (`./build/mkfs disk.img disk/`). The tool opens the zeroed image, lays down the `MYFS` structure, reads `suvari.txt` from the host folder, and injects it into the binary image.
    
4. **Booting the Guest:**
    
    Finally, QEMU is launched. The `QEMU_FLAGS` pass the populated `disk.img` to the virtual machine as a raw VirtIO block device (`-drive id=drive0,file=disk.img,format=raw`). When the kernel boots and calls `fs_init()`, it successfully detects the `MYFS` magic string and can read the files.
    

> [!INFO]
> 
> **The Dependency Tree:** In the Makefile, `disk.img` depends on `$(BUILD_DIR)/mkfs` and the contents of `$(wildcard disk/*)`. This means if you edit `suvari.txt` or add a new file to the `disk/` folder, the Makefile is smart enough to detect the change, re-run `mkfs` to pack a fresh `disk.img`, and pass the updated drive to QEMU on the next build.
