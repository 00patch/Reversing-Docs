# Windows Kernel Concepts

The Windows Kernel is the core component of the operating system, responsible for managing system resources, hardware abstraction, and providing a secure environment for user-mode applications.

## Architecture Overview

Windows uses a **Hybrid Kernel** architecture. It combines the speed of a monolithic kernel with the modularity of a microkernel.

```text
[ User Mode ]
  - Applications
  - Subsystem DLLs (kernel32.dll, user32.dll)
  - NTDLL.dll (Gateway)
--------| (Syscall Interface) --------
[ Kernel Mode ]
  - Executive (Ntoskrnl.exe) -> High-level services
  - Kernel (Ntoskrnl.exe)    -> Low-level scheduling/interrupts
  - Device Drivers (.sys)
  - HAL (Hal.dll)           -> Hardware Abstraction
```

---

## 1. Memory Management

### Virtual Memory and Paging
Windows implements a demand-paged virtual memory system. Each process has its own **Virtual Address Space (VAS)**, isolated from other processes.

- **Paging**: The process of translating a Virtual Address (VA) to a Physical Address (PA).
- **Page Tables (x64)**: Uses a 4-level hierarchy:
    - `PML4` (Page Map Level 4)
    - `PDPT` (Page Directory Pointer Table)
    - `PD` (Page Directory)
    - `PT` (Page Table)
- **VAD (Virtual Address Descriptor)**: The kernel uses a balanced tree (AVL) of VADs to track which ranges of virtual memory are allocated within a process.

### Memory Pools
The kernel manages memory in two primary "pools":
- **Non-paged Pool**: Memory that is guaranteed to stay in physical RAM and never paged to disk. Used for critical structures (e.g., IRPs, spinlocks) accessed at high IRQL.
- **Paged Pool**: Virtual memory that can be paged to disk when physical RAM is low.

### 64-BIT Address Space Layout (Canonical Addresses)
On x64, addresses are 64 bits, but only 48 bits are currently used.
- **User Space**: `0x00000000'00000000` to `0x00007FFF'FFFFFFFF`
- **Kernel Space**: `0xFFFF8000'00000000` to `0xFFFFFFFF'FFFFFFFF`
- **Sign Extension**: Bits 48-63 must match bit 47. If they don't, the address is **non-canonical**, triggering a General Protection Fault.

---

## 2. Processor Privilege Levels (Rings)

Windows utilizes the CPU's hardware-enforced protection levels:
- **Ring 3 (User Mode)**: Restricted access. Applications cannot touch hardware or kernel memory directly.
- **Ring 0 (Kernel Mode)**: Unrestricted access. The OS and drivers have full control over the CPU and memory.

### Transition: User to Kernel
Applications transition to kernel mode via **System Calls (Syscalls)**.
1.  Application calls an API (e.g., `CreateFile`).
2.  `kernel32.dll` calls `ntdll.dll`.
3.  `ntdll.dll` executes the `syscall` instruction (x64) or `int 2E` (older x86).
4.  The CPU switches to Ring 0 and jumps to the **System Service Dispatcher** (`nt!KiSystemCall64`).

---

## 3. System Service Descriptor Table (SSDT)

The SSDT is the lookup table the kernel uses to find the function corresponding to a syscall number (index).

### x64 SSDT Relative Offsets
Unlike x86 (which used direct pointers), x64 SSDT (`nt!KiServiceTable`) uses 32-bit relative offsets to mitigate "SSDT Hooking".

**To find the absolute address in WinDbg:**
```text
; 1. Find the offset (example index 0x33)
kd> dd /c1 nt!KiServiceTable + (0x33 * 4) L1

; 2. Calculate: Address = TableBase + (Offset >> 4)
kd> u nt!KiServiceTable + (Offset >> 4)
```

---

## 4. Interrupts and IRQL

### IDT (Interrupt Descriptor Table)
The IDT is an array of entry points for interrupts and exceptions. Each logical processor has its own IDT, pointed to by the `IDTR` register.

### IRQL (Interrupt Request Level)
IRQL is a Windows-specific mechanism to prioritize interrupts.
- **PASSIVE_LEVEL (0)**: Normal thread execution. Paged memory is accessible.
- **APC_LEVEL (1)**: Asynchronous Procedure Calls.
- **DISPATCH_LEVEL (2)**: Thread scheduler and DPCs. **Cannot access paged memory** (causes a bugcheck).
- **DIRQL (3+)**: Device interrupts.

---

## 5. Windows Driver Model (WDM)

Drivers are organized in **Device Stacks**. An I/O request travels from the top (Filter) to the bottom (Function/Bus) driver.

### Major Function Table
The `MajorFunction` array in the `DRIVER_OBJECT` contains entry points for different I/O requests.

```c
enum Major_Codes : unsigned __int8
{
  IRP_MJ_CREATE = 0x0,
  IRP_MJ_CREATE_NAMED_PIPE = 0x1,
  IRP_MJ_CLOSE = 0x2,
  IRP_MJ_READ = 0x3,
  IRP_MJ_WRITE = 0x4,
  IRP_MJ_QUERY_INFORMATION = 0x5,
  IRP_MJ_SET_INFORMATION = 0x6,
  IRP_MJ_QUERY_EA = 0x7,
  IRP_MJ_SET_EA = 0x8,
  IRP_MJ_FLUSH_BUFFERS = 0x9,
  IRP_MJ_QUERY_VOLUME_INFORMATION = 0xA,
  IRP_MJ_SET_VOLUME_INFORMATION = 0xB,
  IRP_MJ_DIRECTORY_CONTROL = 0xC,
  IRP_MJ_FILE_SYSTEM_CONTROL = 0xD,
  IRP_MJ_DEVICE_CONTROL = 0xE,
  IRP_MJ_INTERNAL_DEVICE_CONTROL = 0xF,
  IRP_MJ_SHUTDOWN = 0x10,
  IRP_MJ_LOCK_CONTROL = 0x11,
  IRP_MJ_CLEANUP = 0x12,
  IRP_MJ_CREATE_MAILSLOT = 0x13,
  IRP_MJ_QUERY_SECURITY = 0x14,
  IRP_MJ_SET_SECURITY = 0x15,
  IRP_MJ_POWER = 0x16,
  IRP_MJ_SYSTEM_CONTROL = 0x17,
  IRP_MJ_DEVICE_CHANGE = 0x18,
  IRP_MJ_QUERY_QUOTA = 0x19,
  IRP_MJ_SET_QUOTA = 0x1A,
  IRP_MJ_PNP = 0x1B,
  IRP_MJ_PNP_POWER = 0x1C,
  IRP_MJ_MAXIMUM_FUNCTION = 0x1D,
};
```

### Buffer Access Methods
- **Buffered I/O**: Kernel copies data between user and a system-allocated buffer.
- **Direct I/O**: Kernel locks user pages in RAM and maps them via an **MDL** (Memory Descriptor List).
- **Neither I/O**: Driver accesses user addresses directly (highly dangerous).

---

## 6. Kernel Protections

Modern Windows implements several layers of security to prevent kernel exploitation:

- **KASLR (Kernel ASLR)**: Randomizes the base address of `ntoskrnl.exe` and drivers on every boot.
- **SMEP (Supervisor Mode Execution Prevention)**: CPU feature that prevents the kernel from executing code in user-mode pages.
- **SMAP (Supervisor Mode Access Prevention)**: Prevents the kernel from accidentally reading/writing user-mode data.
- **PatchGuard (KPP)**: Periodically checks the integrity of critical kernel structures (SSDT, IDT, GDT) and triggers a Blue Screen (BSOD) if tampering is detected.
- **VBS / HVCI**: Uses the Hypervisor to protect kernel memory, ensuring only signed code can be executed.

---

## 7. Exploitation: Token Stealing

A classic privilege escalation technique where an exploit replaces the current process's token with the **SYSTEM** process token.

**The Attack Logic:**
1.  Use a vulnerability to get an **Arbitrary Read/Write** in kernel mode.
2.  Find the `EPROCESS` of the exploit process (via `gs:[188h]` -> `KTHREAD` -> `KPROCESS`).
3.  Walk the `ActiveProcessLinks` list until `UniqueProcessId == 4` (System).
4.  Copy the `Token` pointer from System `EPROCESS` to the exploit `EPROCESS`.

---

## 8. Core Kernel Structures (Structural Definition)

For reversing and exploit development, understanding the field offsets in these structures is essential. You can dump these structures in WinDbg using `dt nt!_STRUCTURE_NAME`.

### EPROCESS (Process Object)
```c
struct _EPROCESS {
    struct _KPROCESS Pcb;            // Core Kernel Process structure
    struct _EX_PUSH_LOCK ProcessLock;
    union _LARGE_INTEGER CreateTime;
    union _LARGE_INTEGER ExitTime;
    struct _EX_RUNDOWN_REF RundownProtect;
    void* UniqueProcessId;           // Process ID (PID)
    struct _LIST_ENTRY ActiveProcessLinks; // Doubly-linked list of processes
    void* Token;                     // Security Token (for Token Stealing)
    void* VirtualProcessSize;        // Process VAS size
};
```

### KTHREAD (Kernel Thread Object)
```c
struct _KTHREAD {
    struct _DISPATCHER_HEADER Header;
    void* InitialStack;
    void* StackLimit;
    void* KernelStack;              // Base of kernel stack
    unsigned long ThreadFlags;
    unsigned char Priority;
    unsigned char BasePriority;
    void* Teb;                      // Pointer to Thread Environment Block (User-mode)
};
```

### DRIVER_OBJECT
```c
struct _DRIVER_OBJECT {
    short Type;
    short Size;
    struct _DEVICE_OBJECT *DeviceObject; // Head of device list
    unsigned long Flags;
    void *DriverStart;
    unsigned long DriverSize;
    void *DriverSection;
    struct _DRIVER_EXTENSION *DriverExtension;
    struct _UNICODE_STRING DriverName;
    void *HardwareDatabase;
    struct _FAST_IO_DISPATCH *FastIoDispatch;
    long (*DriverInit)(struct _DRIVER_OBJECT *, struct _UNICODE_STRING *);
    void (*DriverStartIo)(struct _DEVICE_OBJECT *, struct _IRP *);
    void (*DriverUnload)(struct _DRIVER_OBJECT *);
    void *MajorFunction[28];        // IRP Dispatch Table
};
```

### DEVICE_OBJECT
```c
struct _DEVICE_OBJECT {
    short Type;
    unsigned short Size;
    long ReferenceCount;
    struct _DRIVER_OBJECT *DriverObject;
    struct _DEVICE_OBJECT *NextDevice;   // Pointer to next device in stack
    struct _DEVICE_OBJECT *AttachedDevice;
    struct _IRP *CurrentIrp;
    unsigned long Flags;
    unsigned long Characteristics;
};
```

### IRP (I/O Request Packet)
```c
struct _IRP {
    short Type;
    unsigned short Size;
    struct _MDL *MdlAddress;         // Memory Descriptor List
    unsigned long Flags;
    union {
        struct _IRP *MasterIrp;
        long IrpCount;
    } AssociatedIrp;                 // SystemBuffer is here
    struct _IO_STATUS_BLOCK IoStatus;
    char RequestorMode;
    // ...
    struct _IO_STACK_LOCATION *Tail.Overlay.CurrentStackLocation;
};
```

---

## 9. Essential WinDbg Kernel Commands

| Command | Description |
| :--- | :--- |
| **!process 0 0** | List all active processes. |
| **!process <addr> 1** | Show detailed info for a specific process (VADs, threads). |
| **dt nt!_EPROCESS <addr>** | Dump the EPROCESS structure. |
| **!address <addr>** | Show info about a specific memory address. |
| **!idt** | Dump the Interrupt Descriptor Table. |
| **!pcr** | Show the Processor Control Region (KPCR). |
| **!devnode 0 1** | Show the device tree. |
| **!irp <addr>** | Dump an IRP structure. |
