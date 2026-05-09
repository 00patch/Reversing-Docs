## Core Kernel Structures (Structural Definition)

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
---
## DRIVERS

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
    NTSTATUS (*MajorFunction[28])(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2); //0x70
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

### IRP (I/O Request Packet)
```c
//0xd0 bytes (sizeof)
struct _IRP
{
    SHORT Type;                                                             //0x0
    USHORT Size;                                                            //0x2
    struct _MDL* MdlAddress;                                                //0x8
    ULONG Flags;                                                            //0x10
    union
    {
        struct _IRP* MasterIrp;                                             //0x18
        LONG IrpCount;                                                      //0x18
        VOID* SystemBuffer;                                                 //0x18
    } AssociatedIrp;                                                        //0x18
    struct _LIST_ENTRY ThreadListEntry;                                     //0x20
    struct _IO_STATUS_BLOCK IoStatus;                                       //0x30
    CHAR RequestorMode;                                                     //0x40
    UCHAR PendingReturned;                                                  //0x41
    CHAR StackCount;                                                        //0x42
    CHAR CurrentLocation;                                                   //0x43
    UCHAR Cancel;                                                           //0x44
    UCHAR CancelIrql;                                                       //0x45
    CHAR ApcEnvironment;                                                    //0x46
    UCHAR AllocationFlags;                                                  //0x47
    struct _IO_STATUS_BLOCK* UserIosb;                                      //0x48
    struct _KEVENT* UserEvent;                                              //0x50
    union
    {
        struct
        {
            union
            {
                VOID (*UserApcRoutine)(VOID* arg1, struct _IO_STATUS_BLOCK* arg2, ULONG arg3); //0x58
                VOID* IssuingProcess;                                       //0x58
            };
            VOID* UserApcContext;                                           //0x60
        } AsynchronousParameters;                                           //0x58
        union _LARGE_INTEGER AllocationSize;                                //0x58
    } Overlay;                                                              //0x58
    VOID (*CancelRoutine)(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2);  //0x68
    VOID* UserBuffer;                                                       //0x70
    union
    {
        struct
        {
            union
            {
                struct _KDEVICE_QUEUE_ENTRY DeviceQueueEntry;               //0x78
                VOID* DriverContext[4];                                     //0x78
            };
            struct _ETHREAD* Thread;                                        //0x98
            CHAR* AuxiliaryBuffer;                                          //0xa0
            struct _LIST_ENTRY ListEntry;                                   //0xa8
            union
            {
                struct _IO_STACK_LOCATION* CurrentStackLocation;            //0xb8
                ULONG PacketType;                                           //0xb8
            };
            struct _FILE_OBJECT* OriginalFileObject;                        //0xc0
        } Overlay;                                                          //0x78
        struct _KAPC Apc;                                                   //0x78
        VOID* CompletionKey;                                                //0x78
    } Tail;                                                                 //0x78
}; 
```

`UserBuffer`
Contains the address of an output buffer if both of the following conditions apply:
The major function code in the I/O stack location is `IRP_MJ_DEVICE_CONTROL` or `IRP_MJ_INTERNAL_DEVICE_CONTROL`
The I/O control code was defined with `METHOD_NEITHER` or `METHOD_BUFFERED`.
For `METHOD_BUFFERED`, the driver should use the buffer pointed to by Irp->AssociatedIrp.SystemBuffer as the output buffer. When the driver completes the request, the I/O manager copies the contents of this buffer to the output buffer that is pointed to by Irp->UserBuffer. The driver should not write directly to the buffer pointed to by Irp->UserBuffer.
For more information, see [Buffer Descriptions for I/O Control Codes](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/buffer-descriptions-for-i-o-control-codes).

### _IO_STACK_LOCATION
```c
//0x48 bytes (sizeof)
struct _IO_STACK_LOCATION
{
    UCHAR MajorFunction;                                                    //0x0
    UCHAR MinorFunction;                                                    //0x1
    UCHAR Flags;                                                            //0x2
    UCHAR Control;                                                          //0x3
    union
    {
        struct
        {
            struct _IO_SECURITY_CONTEXT* SecurityContext;                   //0x8
            ULONG Options;                                                  //0x10
            USHORT FileAttributes;                                          //0x18
            USHORT ShareAccess;                                             //0x1a
            ULONG EaLength;                                                 //0x20
        } Create;                                                           //0x8
        struct
        {
            struct _IO_SECURITY_CONTEXT* SecurityContext;                   //0x8
            ULONG Options;                                                  //0x10
            USHORT Reserved;                                                //0x18
            USHORT ShareAccess;                                             //0x1a
            struct _NAMED_PIPE_CREATE_PARAMETERS* Parameters;               //0x20
        } CreatePipe;                                                       //0x8
        struct
        {
            struct _IO_SECURITY_CONTEXT* SecurityContext;                   //0x8
            ULONG Options;                                                  //0x10
            USHORT Reserved;                                                //0x18
            USHORT ShareAccess;                                             //0x1a
            struct _MAILSLOT_CREATE_PARAMETERS* Parameters;                 //0x20
        } CreateMailslot;                                                   //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            ULONG Key;                                                      //0x10
            ULONG Flags;                                                    //0x14
            union _LARGE_INTEGER ByteOffset;                                //0x18
        } Read;                                                             //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            ULONG Key;                                                      //0x10
            ULONG Flags;                                                    //0x14
            union _LARGE_INTEGER ByteOffset;                                //0x18
        } Write;                                                            //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            struct _UNICODE_STRING* FileName;                               //0x10
            enum _FILE_INFORMATION_CLASS FileInformationClass;              //0x18
            ULONG FileIndex;                                                //0x20
        } QueryDirectory;                                                   //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            ULONG CompletionFilter;                                         //0x10
        } NotifyDirectory;                                                  //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            ULONG CompletionFilter;                                         //0x10
            enum _DIRECTORY_NOTIFY_INFORMATION_CLASS DirectoryNotifyInformationClass; //0x18
        } NotifyDirectoryEx;                                                //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            enum _FILE_INFORMATION_CLASS FileInformationClass;              //0x10
        } QueryFile;                                                        //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            enum _FILE_INFORMATION_CLASS FileInformationClass;              //0x10
            struct _FILE_OBJECT* FileObject;                                //0x18
            union
            {
                struct
                {
                    UCHAR ReplaceIfExists;                                  //0x20
                    UCHAR AdvanceOnly;                                      //0x21
                };
                ULONG ClusterCount;                                         //0x20
                VOID* DeleteHandle;                                         //0x20
            };
        } SetFile;                                                          //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            VOID* EaList;                                                   //0x10
            ULONG EaListLength;                                             //0x18
            ULONG EaIndex;                                                  //0x20
        } QueryEa;                                                          //0x8
        struct
        {
            ULONG Length;                                                   //0x8
        } SetEa;                                                            //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            enum _FSINFOCLASS FsInformationClass;                           //0x10
        } QueryVolume;                                                      //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            enum _FSINFOCLASS FsInformationClass;                           //0x10
        } SetVolume;                                                        //0x8
        struct
        {
            ULONG OutputBufferLength;                                       //0x8
            ULONG InputBufferLength;                                        //0x10
            ULONG FsControlCode;                                            //0x18
            VOID* Type3InputBuffer;                                         //0x20
        } FileSystemControl;                                                //0x8
        struct
        {
            union _LARGE_INTEGER* Length;                                   //0x8
            ULONG Key;                                                      //0x10
            union _LARGE_INTEGER ByteOffset;                                //0x18
        } LockControl;                                                      //0x8
        struct
        {
            ULONG OutputBufferLength;                                       //0x8
            ULONG InputBufferLength;                                        //0x10
            ULONG IoControlCode;                                            //0x18
            VOID* Type3InputBuffer;                                         //0x20
        } DeviceIoControl;                                                  //0x8
        struct
        {
            ULONG SecurityInformation;                                      //0x8
            ULONG Length;                                                   //0x10
        } QuerySecurity;                                                    //0x8
        struct
        {
            ULONG SecurityInformation;                                      //0x8
            VOID* SecurityDescriptor;                                       //0x10
        } SetSecurity;                                                      //0x8
        struct
        {
            struct _VPB* Vpb;                                               //0x8
            struct _DEVICE_OBJECT* DeviceObject;                            //0x10
        } MountVolume;                                                      //0x8
        struct
        {
            struct _VPB* Vpb;                                               //0x8
            struct _DEVICE_OBJECT* DeviceObject;                            //0x10
        } VerifyVolume;                                                     //0x8
        struct
        {
            struct _SCSI_REQUEST_BLOCK* Srb;                                //0x8
        } Scsi;                                                             //0x8
        struct
        {
            ULONG Length;                                                   //0x8
            VOID* StartSid;                                                 //0x10
            struct _FILE_GET_QUOTA_INFORMATION* SidList;                    //0x18
            ULONG SidListLength;                                            //0x20
        } QueryQuota;                                                       //0x8
        struct
        {
            ULONG Length;                                                   //0x8
        } SetQuota;                                                         //0x8
        struct
        {
            enum _DEVICE_RELATION_TYPE Type;                                //0x8
        } QueryDeviceRelations;                                             //0x8
        struct
        {
            struct _GUID* InterfaceType;                                    //0x8
            USHORT Size;                                                    //0x10
            USHORT Version;                                                 //0x12
            struct _INTERFACE* Interface;                                   //0x18
            VOID* InterfaceSpecificData;                                    //0x20
        } QueryInterface;                                                   //0x8
        struct
        {
            struct _DEVICE_CAPABILITIES* Capabilities;                      //0x8
        } DeviceCapabilities;                                               //0x8
        struct
        {
            struct _IO_RESOURCE_REQUIREMENTS_LIST* IoResourceRequirementList; //0x8
        } FilterResourceRequirements;                                       //0x8
        struct
        {
            ULONG WhichSpace;                                               //0x8
            VOID* Buffer;                                                   //0x10
            ULONG Offset;                                                   //0x18
            ULONG Length;                                                   //0x20
        } ReadWriteConfig;                                                  //0x8
        struct
        {
            UCHAR Lock;                                                     //0x8
        } SetLock;                                                          //0x8
        struct
        {
            enum BUS_QUERY_ID_TYPE IdType;                                  //0x8
        } QueryId;                                                          //0x8
        struct
        {
            enum DEVICE_TEXT_TYPE DeviceTextType;                           //0x8
            ULONG LocaleId;                                                 //0x10
        } QueryDeviceText;                                                  //0x8
        struct
        {
            UCHAR InPath;                                                   //0x8
            UCHAR Reserved[3];                                              //0x9
            enum _DEVICE_USAGE_NOTIFICATION_TYPE Type;                      //0x10
        } UsageNotification;                                                //0x8
        struct
        {
            enum _SYSTEM_POWER_STATE PowerState;                            //0x8
        } WaitWake;                                                         //0x8
        struct
        {
            struct _POWER_SEQUENCE* PowerSequence;                          //0x8
        } PowerSequence;                                                    //0x8
        struct
        {
            union
            {
                ULONG SystemContext;                                        //0x8
                struct _SYSTEM_POWER_STATE_CONTEXT SystemPowerStateContext; //0x8
            };
            enum _POWER_STATE_TYPE Type;                                    //0x10
            union _POWER_STATE State;                                       //0x18
            enum POWER_ACTION ShutdownType;                                 //0x20
        } Power;                                                            //0x8
        struct
        {
            struct _CM_RESOURCE_LIST* AllocatedResources;                   //0x8
            struct _CM_RESOURCE_LIST* AllocatedResourcesTranslated;         //0x10
        } StartDevice;                                                      //0x8
        struct
        {
            ULONGLONG ProviderId;                                           //0x8
            VOID* DataPath;                                                 //0x10
            ULONG BufferSize;                                               //0x18
            VOID* Buffer;                                                   //0x20
        } WMI;                                                              //0x8
        struct
        {
            VOID* Argument1;                                                //0x8
            VOID* Argument2;                                                //0x10
            VOID* Argument3;                                                //0x18
            VOID* Argument4;                                                //0x20
        } Others;                                                           //0x8
    } Parameters;                                                           //0x8
    struct _DEVICE_OBJECT* DeviceObject;                                    //0x28
    struct _FILE_OBJECT* FileObject;                                        //0x30
    LONG (*CompletionRoutine)(struct _DEVICE_OBJECT* arg1, struct _IRP* arg2, VOID* arg3); //0x38
    VOID* Context;                                                          //0x40
}; 
```
---

### Major_codes
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
### DEVICE_FLAGS
```c
#define DO_VERIFY_VOLUME	                  0x00000002
#define DO_BUFFERED_IO	                    0x00000004
#define DO_EXCLUSIVE		                    0x00000008
#define DO_DIRECT_IO		                    0x00000010
#define DO_MAP_IO_BUFFER	                  0x00000020
#define DO_DEVICE_INITIALIZING             	0x00000080
#define DO_SHUTDOWN_REGISTERED             	0x00000800
#define DO_BUS_ENUMERATED_DEVICE	        	0x00001000
#define DO_POWER_PAGABLE		                0x00002000
#define DO_POWER_INRUSH	                   	0x00004000
#define DO_DEVICE_TO_BE_RESET	              0x04000000
#define DO_DAX_VOLUME		                    0x10000000
```
#### References
[Vergilius Project](https://www.vergiliusproject.com/)
[WDM Header](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/#structures)
