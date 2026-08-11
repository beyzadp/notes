---
title: "8. Device Drivers & Interrupts: Talking to Hardware"
---

part of the [[index|os-development]] series.

This section describes how the kernel reaches the outside world without letting slow, stateful devices stall the whole system. The CPU is fast and predictable; devices are slow, asynchronous, and full of side effects. A driver exists to absorb that mismatch and present a tiny, deterministic interface to the rest of the kernel.

## The Role of the Driver

*What the driver must solve:*

* *Trust boundary:* Only the kernel touches device registers. User code requests services; the kernel arbitrates and talks to hardware.
* *Timing mismatch:* Devices complete work on their own schedule. The driver prevents the CPU from wasting cycles while it waits.
* *State and ordering:* Devices require specific register sequences. The driver encodes those sequences so callers do not need to know them.
* *Failure visibility:* Devices signal errors through status bits. The driver turns those into clear kernel-level outcomes.

*The minimal control loop:*

1. Configure the device and select how it will notify the CPU.
2. Issue work by writing command/data registers.
3. Wait using polling or interrupts, based on latency and cost.
4. Read results, translate status, and return a stable kernel result.

This keeps the kernel responsive while still making forward progress on slow I/O, and it preserves isolation by keeping hardware access in one place.


## Memory-Mapped I/O (MMIO)

MMIO is how the CPU talks to devices without special instructions. Device registers live at fixed physical addresses, and a normal load/store to that address becomes a hardware action instead of a RAM access. This is why drivers treat these addresses as volatile state: reads can change the device state, and writes can trigger real work.

**What this implies for a tiny kernel**

- **Address knowledge:** The driver must know the exact MMIO base and offsets for a device.
- **No caching:** MMIO regions must not be cached or reordered, or the device will see stale or out-of-order commands.
- **One writer:** Only the kernel touches MMIO so device state stays coherent.

MMIO gives us the doorbell and status registers that connect the CPU to the device's internal state machine, which is exactly what the virtqueue example below relies on.
In practice, you often communicate by flipping specific bits in a single MMIO status register in a very exact order.

*Implementation Notes:*

* Accessing MMIO registers is not the same as accessing normal memory.
* Use volatile so the compiler does not optimize away reads/writes.
* MMIO reads/writes can trigger side effects (for example, sending a command to the device).
* When accessing hardware registers, `#define` constants act as fixed offsets added to the device base address.
* For shared memory layouts, use C `structs` so the compiler computes field offsets, and mark them `__attribute__((packed))` to avoid hidden padding that would misalign the device view.


## VirtIO Device Initialization

We will focus on the virtio-blk device. It is a virtual disk in QEMU, but it uses the same queue-based interface you would see in real hardware. So we can explain the driver once, and the same ideas carry over.

### Why we map the VirtIO MMIO page

When paging is enabled, the kernel cannot touch physical addresses directly. Every access goes through the page table. The VirtIO MMIO registers live at physical `0x10001000`, so the kernel must map that address before it can read or write the device.

This line creates an identity mapping for the MMIO page and grants read/write permission:

```c
map_page(page_table, VIRTIO_BLK_PADDR, VIRTIO_BLK_PADDR, PAGE_R | PAGE_W);
```


* It maps virtual = physical for the device base.
* It grants `PAGE_R | PAGE_W` so the kernel can flip device bits.
* It omits `PAGE_X` because we never execute from device registers.

Without this mapping, any MMIO access would fault as an unmapped virtual address.

### Initializing the Status Register

The device status register lives at physical address `0x10001070`. Writing to `0x10001070` is MMIO: the CPU sends the write to QEMU's virtual device, which updates its internal state machine instead of RAM.

*Initialization Steps:*

1. Reset the device (not required on first boot).
2. Set ACKNOWLEDGE: the driver has noticed the device.
3. Set DRIVER: the driver knows how to drive the device.
4. Do device-specific setup: read feature bits, discover virtqueues, optional MSI-X, read/write config space.
5. Write back the subset of feature bits the driver accepts.
6. Set DRIVER_OK.

*Status Bits (Binary):*

* `VIRTIO_STATUS_ACK` = 1 (0000 0001), bit 0
* `VIRTIO_STATUS_DRIVER` = 2 (0000 0010), bit 1
* `VIRTIO_STATUS_DRIVER_OK` = 4 (0000 0100), bit 2

*Examples of register offsets and packed shared-memory structs:*

```c
#define VIRTIO_REG_MAGIC         0x00
#define VIRTIO_REG_VERSION       0x04
#define VIRTIO_REG_DEVICE_ID     0x08

struct virtq_used {
    uint16_t flags;
    uint16_t index;
    struct virtq_used_elem ring[VIRTQ_ENTRY_NUM];
} __attribute__((packed));

```

### Virtqueue Initialization

1. Write the virtqueue index to Queue Select (first queue is 0).
2. Read Queue Size; it is always a power of 2 and determines the queue capacity. If it is 0, the virtqueue does not exist.
3. Allocate and zero the virtqueue in contiguous physical memory with 4096-byte alignment.
4. Write the physical address divided by 4096 to Queue Address.

## Handling I/O Requests: VirtIO Block Read/Write

### Descriptor Layout (Read)

| index | points to      | permission | state before device runs |
|-------|----------------|------------|--------------------------|
| 0     | request header | read-only  | OS filled sector and op  |
| 1     | data buffer    | write-only | empty                    |
| 2     | status byte    | write-only | empty                    |

### Request Header Shape (`virtio_blk_req`)

```c
struct virtio_blk_req {
    // First descriptor: read-only from the device
    uint32_t type;
    uint32_t reserved;
    uint64_t sector;

    // Second descriptor: writable by the device if it's a read operation
    // (VIRTQ_DESC_F_WRITE)
    uint8_t data[512];

    // Third descriptor: writable by the device (VIRTQ_DESC_F_WRITE)
    uint8_t status;
} __attribute__((packed));
```

This struct is the on-wire message the device reads from descriptor 0. It is packed so the layout matches the device's expected bytes exactly.

- **`type`**: read or write operation selector.
- **`reserved`**: unused padding required by the device format.
- **`sector`**: the target disk sector number.
- **`data[512]`**: the data payload. For reads, the device writes into it; for writes, the device reads from it.
- **`status`**: 1-byte result written by the device (`0` means success).

### The Read Path, Step by Step

1. *Setup:* The driver builds the descriptor chain and fills the request header.
2. *Queue:* It publishes the head index in the available ring.
3. *Notify:* It writes the MMIO "doorbell" to tell the device work is ready.
4. *Process:* The device DMA-copies disk data into the buffer in descriptor 1.
5. *Status:* The device writes 0 (success) into the status byte in descriptor 2.
6. *Complete:* The device pushes the head index into the used ring; the driver can now consume the buffer.

### The Write Path

Write is the same shape as the read path, with inverted permissions:

* Descriptor 1 becomes read-only so the device can read the OS-provided data.
* The device still writes the status byte to confirm success.

## Sending an I/O Request (Shared Memory + Doorbell)

Sending a request means writing a formatted message into shared RAM and ringing the doorbell so the device processes it.

1. *Prepare the request header:* Fill `virtio_blk_req` with read/write and the target sector.
2. *Build the descriptor chain:* Link three descriptors: request header, data buffer, and a 1-byte status flag.
3. *Update the available ring:* Place the head descriptor index into the next slot so the device sees the request.
4. *Kick the device:* MMIO write to `VIRTIO_REG_QUEUE_NOTIFY` to trigger processing.
5. *Busy-wait (simple kernel):* Spin until the device writes the head index into the used ring.
6. *Check status:* If the status byte is 0, the transfer succeeded and the buffer is valid.

## summary


> There is a fixed address for the device status register, and the virtio driver writes to this address to report its state to QEMU or the virtio device. There are three main bits: bit 0 is for **ACKNOWLEDGE**, which means the driver reports that it found the device; bit 1 is for **DRIVER**, which means the OS recognizes the device and knows how to drive it; and bit 2 is **DRIVER_OK**. Between setting the DRIVER bit and the DRIVER_OK bit, the driver performs its setup and feature negotiation.


> During the initialization phase, the driver selects the virtqueue and checks its size to determine its capacity and ensure it actually exists. Next, the driver allocates and zeros out contiguous physical memory for the queue, handles the required memory alignment, and registers that physical address with the device so device can also know where the ring is.


> In the I/O request handling phase, there are two rings involved, and work is read from one and written to the other. For example, during a read operation, a higher-level part of the OS (like the file system) sends a request down to the driver. The driver then builds the descriptor chain and fills out the request, translating the OS request into memory addresses that the device can actually understand and process. After filling the descriptor chain, the driver pushes the head index to the **Available Ring** and notifies the device that there is work to do.

> we basically act like virtio is the real hardware and it handles the driver stuff.
