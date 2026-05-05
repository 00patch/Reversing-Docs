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

### Core Objects
- **DRIVER_OBJECT**: Represents the loaded driver. Contains the **MajorFunction** table (dispatchers for Create, Read, Write, etc.).
- **DEVICE_OBJECT**: Represents a device instance.
- **IRP (I/O Request Packet)**: The container for an I/O request. It includes a "Stack Location" (`IO_STACK_LOCATION`) for each driver in the chain.

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

## 8. Essential WinDbg Kernel Commands

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
