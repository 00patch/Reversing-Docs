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
struct _DEVICE_OBJECT{
  //.....
    NTSTATUS (*MajorFunction[28])(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2); //0x70
}

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
### Device Types
```c
#define FILE_DEVICE_BEEP                    0x00000001
#define FILE_DEVICE_CD_ROM                  0x00000002
#define FILE_DEVICE_CD_ROM_FILE_SYSTEM      0x00000003
#define FILE_DEVICE_CONTROLLER              0x00000004
#define FILE_DEVICE_DATALINK                0x00000005
#define FILE_DEVICE_DFS                     0x00000006
#define FILE_DEVICE_DISK                    0x00000007
#define FILE_DEVICE_DISK_FILE_SYSTEM        0x00000008
#define FILE_DEVICE_FILE_SYSTEM             0x00000009
#define FILE_DEVICE_INPORT_PORT             0x0000000a
#define FILE_DEVICE_KEYBOARD                0x0000000b
#define FILE_DEVICE_MAILSLOT                0x0000000c
#define FILE_DEVICE_MIDI_IN                 0x0000000d
#define FILE_DEVICE_MIDI_OUT                0x0000000e
#define FILE_DEVICE_MOUSE                   0x0000000f
#define FILE_DEVICE_MULTI_UNC_PROVIDER      0x00000010
#define FILE_DEVICE_NAMED_PIPE              0x00000011
#define FILE_DEVICE_NETWORK                 0x00000012
#define FILE_DEVICE_NETWORK_BROWSER         0x00000013
#define FILE_DEVICE_NETWORK_FILE_SYSTEM     0x00000014
#define FILE_DEVICE_NULL                    0x00000015
#define FILE_DEVICE_PARALLEL_PORT           0x00000016
#define FILE_DEVICE_PHYSICAL_NETCARD        0x00000017
#define FILE_DEVICE_PRINTER                 0x00000018
#define FILE_DEVICE_SCANNER                 0x00000019
#define FILE_DEVICE_SERIAL_MOUSE_PORT       0x0000001a
#define FILE_DEVICE_SERIAL_PORT             0x0000001b
#define FILE_DEVICE_SCREEN                  0x0000001c
#define FILE_DEVICE_SOUND                   0x0000001d
#define FILE_DEVICE_STREAMS                 0x0000001e
#define FILE_DEVICE_TAPE                    0x0000001f
#define FILE_DEVICE_TAPE_FILE_SYSTEM        0x00000020
#define FILE_DEVICE_TRANSPORT               0x00000021
#define FILE_DEVICE_UNKNOWN                 0x00000022
#define FILE_DEVICE_VIDEO                   0x00000023
#define FILE_DEVICE_VIRTUAL_DISK            0x00000024
#define FILE_DEVICE_WAVE_IN                 0x00000025
#define FILE_DEVICE_WAVE_OUT                0x00000026
#define FILE_DEVICE_8042_PORT               0x00000027
#define FILE_DEVICE_NETWORK_REDIRECTOR      0x00000028
#define FILE_DEVICE_BATTERY                 0x00000029
#define FILE_DEVICE_BUS_EXTENDER            0x0000002a
#define FILE_DEVICE_MODEM                   0x0000002b
#define FILE_DEVICE_VDM                     0x0000002c
#define FILE_DEVICE_MASS_STORAGE            0x0000002d
#define FILE_DEVICE_SMB                     0x0000002e
#define FILE_DEVICE_KS                      0x0000002f
#define FILE_DEVICE_CHANGER                 0x00000030
#define FILE_DEVICE_SMARTCARD               0x00000031
#define FILE_DEVICE_ACPI                    0x00000032
#define FILE_DEVICE_DVD                     0x00000033
#define FILE_DEVICE_FULLSCREEN_VIDEO        0x00000034
#define FILE_DEVICE_DFS_FILE_SYSTEM         0x00000035
#define FILE_DEVICE_DFS_VOLUME              0x00000036
#define FILE_DEVICE_SERENUM                 0x00000037
#define FILE_DEVICE_TERMSRV                 0x00000038
#define FILE_DEVICE_KSEC                    0x00000039
#define FILE_DEVICE_FIPS                    0x0000003A
#define FILE_DEVICE_INFINIBAND              0x0000003B
#define FILE_DEVICE_VMBUS                   0x0000003E
#define FILE_DEVICE_CRYPT_PROVIDER          0x0000003F
#define FILE_DEVICE_WPD                     0x00000040
#define FILE_DEVICE_BLUETOOTH               0x00000041
#define FILE_DEVICE_MT_COMPOSITE            0x00000042
#define FILE_DEVICE_MT_TRANSPORT            0x00000043
#define FILE_DEVICE_BIOMETRIC               0x00000044
#define FILE_DEVICE_PMI                     0x00000045
#define FILE_DEVICE_EHSTOR                  0x00000046
#define FILE_DEVICE_DEVAPI                  0x00000047
#define FILE_DEVICE_GPIO                    0x00000048
#define FILE_DEVICE_USBEX                   0x00000049
#define FILE_DEVICE_CONSOLE                 0x00000050
#define FILE_DEVICE_NFP                     0x00000051
#define FILE_DEVICE_SYSENV                  0x00000052
#define FILE_DEVICE_VIRTUAL_BLOCK           0x00000053
#define FILE_DEVICE_POINT_OF_SERVICE        0x00000054
#define FILE_DEVICE_STORAGE_REPLICATION     0x00000055
#define FILE_DEVICE_TRUST_ENV               0x00000056
#define FILE_DEVICE_UCM                     0x00000057
#define FILE_DEVICE_UCMTCPCI                0x00000058
#define FILE_DEVICE_PERSISTENT_MEMORY       0x00000059
#define FILE_DEVICE_NVDIMM                  0x0000005a
#define FILE_DEVICE_HOLOGRAPHIC             0x0000005b
#define FILE_DEVICE_SDFXHCI                 0x0000005c
#define FILE_DEVICE_UCMUCSI                 0x0000005d
#define FILE_DEVICE_PRM                     0x0000005e
#define FILE_DEVICE_EVENT_COLLECTOR         0x0000005f
#define FILE_DEVICE_USB4                    0x00000060
#define FILE_DEVICE_SOUNDWIRE               0x00000061
#define FILE_DEVICE_FABRIC_NVME             0x00000062
#define FILE_DEVICE_SVM                     0x00000063
#define FILE_DEVICE_HARDWARE_ACCELERATOR    0x00000064
#define FILE_DEVICE_I3C                     0x00000065
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
follow kernel addr region
![image](images/screenshot_20260505_184412.png)
