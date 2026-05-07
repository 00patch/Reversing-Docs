# Windows Driver Model (WDM)
Drivers are organized in **Device Stacks**. An I/O request travels from the top (Filter) to the bottom (Function/Bus) driver.

## Driver Lifecycle

The Windows driver lifecycle involves complex transitions between User-Mode (Ring 3) and Kernel-Mode (Ring 0).

### 1. Registration
Before loading, a driver must be registered in the Windows Registry under `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\<DriverName>`. Key values include:
- `ImagePath`: Path to the driver file (e.g., `\SystemRoot\System32\drivers\my.sys`).
- `Type`: Specifies the driver type (e.g., `0x1` for Kernel-mode).
- `Start`: Controls boot-time loading (e.g., `0` for Boot Start, `3` for Manual Start).

### 2. Loading Process
User-mode applications or the OS loader invoke the `ZwLoadDriver` system service:

1.  **User Request**: Application calls `ntdll!ZwLoadDriver`.
2.  **Syscall**: Transition to Ring 0 via the `syscall` instruction. The `EAX` register is set to the System Service Descriptor Table **`SSDT`** for `NtLoadDriver`.
3.  **Kernel Execution**: `nt!NtLoadDriver` is invoked. It locates the driver image on disk.
4.  **Memory Mapping**: The kernel maps the driver image into **Non-Paged Pool** memory, enabling it to be accessed even at high IRQL.
5.  **Initialization**: `nt!IopLoadDriver` is called. It parses the PE file, resolves dependencies, and finally invokes the driver's `DriverEntry`.

```text
[ User Mode ]           [ Kernel Mode (ntoskrnl.exe) ]
    |                               |
    |-- ZwLoadDriver() ------------>|-- NtLoadDriver()
                                    |-- [IopLoadDriver]
                                    |-- Load driver to Non-Paged Pool
                                    |-- DriverEntry(DriverObject, RegistryPath)
```

### 3. DriverEntry
`DriverEntry` is the entry point. It is responsible for:
- Creating the `DEVICE_OBJECT` (via `IoCreateDevice`).
- Setting up the `MajorFunction` dispatch table in the `DRIVER_OBJECT`.
- Creating a Symbolic Link (if user-mode access is needed).

### 4. Dispatch Mechanism
The `MajorFunction` table in the `DRIVER_OBJECT` holds function pointers to the driver's routines. When a user application calls `DeviceIoControl`, the I/O Manager:
1.  Creates an IRP (`IRP_MJ_DEVICE_CONTROL`).
2.  Looks up the function pointer in the `DRIVER_OBJECT->MajorFunction[IRP_MJ_DEVICE_CONTROL]` table.
3.  Calls the routine (Dispatch callback).

**Dispatch Flow:**
```text
[ User App ] --DeviceIoControl--> [ I/O Manager ]
                                      |
                                  (Look up MajorFunction table)
                                      |
                                  [ DispatchDeviceControl() ]
```
*Note: This is a direct function call from the I/O Manager to the driver's dispatch routine, not a syscall, once the IRP is already inside the kernel.*

### 5. Driver Unload
When the service is stopped, `nt!IopUnloadDriver` is called:
1.  **Cleanup**: Calls the `DriverUnload` routine defined in the `DRIVER_OBJECT`.
2.  **Resource Release**: The driver must delete the `DEVICE_OBJECT` (`IoDeleteDevice`) and delete any symbolic links.
3.  **Unload**: The system frees the memory allocated in the Non-Paged Pool.

### Alternative Loading
While `ZwLoadDriver` is standard, drivers can also be loaded via:
- **SCM (Service Control Manager)**: `StartService` (higher-level, wraps `ZwLoadDriver`).
- **Manual Mapping**: Malicious drivers often bypass the loader entirely by mapping themselves directly into kernel memory (bypassing signature checks/PatchGuard).

## IOCTL (Input/Output Control)
IOCTL codes are the primary mechanism for user-mode applications to send device-specific requests to a driver.

### Defining IOCTLs (`CTL_CODE` Macro)
Codes are defined using the `CTL_CODE` macro, which encodes:
- **Device Type**: (e.g., `FILE_DEVICE_UNKNOWN`)
- **Function Code**: User-defined value.
- **Method**: How the buffer is accessed (`METHOD_BUFFERED`, `METHOD_IN_DIRECT`, `METHOD_NEITHER` etc.).
- **Access**: Required permissions (`FILE_ANY_ACCESS`).

### Communication Workflows

#### User-Mode Communication
Applications use `DeviceIoControl`. The I/O Manager creates an `IRP_MJ_DEVICE_CONTROL` and passes it to the driver.

```text
[ User App ] --(DeviceIoControl)--> [ I/O Manager ] --(IRP_MJ_DEVICE_CONTROL)--> [ Driver ]
```

#### Kernel-Mode Communication
Drivers communicate using `IoBuildDeviceIoControlRequest`.

```text
[ Driver A ] --(IoBuildDeviceIoControlRequest)--> [ I/O Manager ] --(IRP)--> [ Driver B ]
```

### IOCTL Processing
Inside the `DispatchDeviceControl` routine, the driver switches on the `IoControlCode`.

```c
// Example: Buffer handling in DispatchDeviceControl
switch (IoControlCode) {
    case MY_IOCTL_CODE:
        // Access buffer via Irp->AssociatedIrp.SystemBuffer
        break;
}
```

## Buffer Access Methods for IOCTLs
The transfer method defined in `CTL_CODE` determines how the I/O manager handles buffers:

| Method | Description |
| :--- | :--- |
| **METHOD_BUFFERED** | I/O Manager allocates a system buffer; data is copied between user and system space. |
| **METHOD_IN_DIRECT** | User buffer is probed and locked; an MDL is created to map it to kernel space. |
| **METHOD_OUT_DIRECT** | Same as IN_DIRECT, but for output. t
| **METHOD_NEITHER** | Driver accesses user virtual memory directly (Danger: Driver must validate addresses). |

---

## Common structures

### DRIVER_OBJECT
```c
//0x150 bytes (sizeof)
struct _DRIVER_OBJECT
{
    SHORT Type;                                                             //0x0
    SHORT Size;                                                             //0x2
    struct _DEVICE_OBJECT* DeviceObject;                                    //0x8
    ULONG Flags;                                                            //0x10
    VOID* DriverStart;                                                      //0x18
    ULONG DriverSize;                                                       //0x20
    VOID* DriverSection;                                                    //0x28
    struct _DRIVER_EXTENSION* DriverExtension;                              //0x30
    struct _UNICODE_STRING DriverName;                                      //0x38
    struct _UNICODE_STRING* HardwareDatabase;                               //0x48
    struct _FAST_IO_DISPATCH* FastIoDispatch;                               //0x50
    LONG (*DriverInit)(struct _DRIVER_OBJECT* arg1, struct _UNICODE_STRING* arg2); //0x58
    VOID (*DriverStartIo)(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2);  //0x60
    VOID (*DriverUnload)(struct _DRIVER_OBJECT* arg1);                      //0x68
    LONG (*MajorFunction[28])(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2); //0x70
};
```
### DEVICE_OBJECT
```c
//0x150 bytes (sizeof)
struct _DEVICE_OBJECT
{
    SHORT Type;                                                             //0x0
    USHORT Size;                                                            //0x2
    LONG ReferenceCount;                                                    //0x4
    struct _DRIVER_OBJECT* DriverObject;                                    //0x8
    struct _DEVICE_OBJECT* NextDevice;                                      //0x10
    struct _DEVICE_OBJECT* AttachedDevice;                                  //0x18
    struct _IRP* CurrentIrp;                                                //0x20
    struct _IO_TIMER* Timer;                                                //0x28
    ULONG Flags;                                                            //0x30
    ULONG Characteristics;                                                  //0x34
    struct _VPB* Vpb;                                                       //0x38
    VOID* DeviceExtension;                                                  //0x40
    ULONG DeviceType;                                                       //0x48
    CHAR StackSize;                                                         //0x4c
    union
    {
        struct _LIST_ENTRY ListEntry;                                       //0x50
        struct _WAIT_CONTEXT_BLOCK Wcb;                                     //0x50
    } Queue;                                                                //0x50
    ULONG AlignmentRequirement;                                             //0x98
    struct _KDEVICE_QUEUE DeviceQueue;                                      //0xa0
    struct _KDPC Dpc;                                                       //0xc8
    ULONG ActiveThreadCount;                                                //0x108
    VOID* SecurityDescriptor;                                               //0x110
    struct _KEVENT DeviceLock;                                              //0x118
    USHORT SectorSize;                                                      //0x130
    USHORT Spare1;                                                          //0x132
    struct _DEVOBJ_EXTENSION* DeviceObjectExtension;                        //0x138
    VOID* Reserved;                                                         //0x140
}; 
```

### IO_STACK_LOCATION
```c
struct _IO_STACK_LOCATION
{
    UCHAR MajorFunction;                                                    //0x0
    UCHAR MinorFunction;                                                    //0x1
    UCHAR Flags;                                                            //0x2
    UCHAR Control;                                                          //0x3
    //... union 
    struct _DEVICE_OBJECT* DeviceObject;                                    //0x28
    struct _FILE_OBJECT* FileObject;                                        //0x30
    LONG (*CompletionRoutine)(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2, VOID* arg3); //0x38
    VOID* Context;                                                          
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
### References
- [Introduction to I/O Control Codes](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/introduction-to-i-o-control-codes)
- [Buffer Descriptions for I/O Control Codes](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/buffer-descriptions-for-i-o-control-codes)
