## PE Format
The Portable Executable (PE) format is a data structure that tells the Windows OS loader how to manage the wrapped executable code. It is used for `.exe`, `.dll`, `.sys`, and `.obj` files. Understanding the PE format is crucial for reverse engineering as it reveals how a program is loaded into memory, its entry point, and its dependencies.

### PE Structure Overview
1.  **DOS Header (`IMAGE_DOS_HEADER`)**: The first 64 bytes of every PE file. 
    - `e_magic`: Always `MZ` (0x5A4D), named after Mark Zbikowski.
    - `e_lfanew`: A 4-byte offset at 0x3C that points to the start of the NT Headers.
2.  **DOS Stub**: A small program that runs if the file is executed in DOS mode, typically printing "This program cannot be run in DOS mode."
3.  **NT Headers (`IMAGE_NT_HEADERS`)**:
    - **Signature**: `PE\0\0` (0x00004550).
    - **File Header (`IMAGE_FILE_HEADER`)**: Contains basic info like target architecture (Machine) and number of sections.
    - **Optional Header (`IMAGE_OPTIONAL_HEADER`)**: Despite its name, it's mandatory for executables. It contains the **AddressOfEntryPoint**, **ImageBase**, **SizeOfImage**, and the **Data Directory** (pointers to Import/Export tables).
4.  **Section Table**: An array of `IMAGE_SECTION_HEADER` structures, each describing a section's properties (name, virtual size, virtual address, raw data size, and characteristics like Read/Write/Execute).

### Executable Files
When a PE file is executed, the loader maps the file from disk (Raw) to memory (Virtual). The alignment on disk (`FileAlignment`) is usually smaller than the alignment in memory (`SectionAlignment`).

Sections layout:
```text
+--------------+  <--- Header size (usually 0x400 or 0x1000 Bytes)
|    Header    |  (DOS, NT, Section Table)
+--------------+
|    .text     |  Executable code (Read/Execute)
+--------------+
|    .data     |  Initialized Global/Static Data (Read/Write)
+--------------+
| .idata/.rdata|  Import Table / Read-only Data (Read)
+--------------+
|    .reloc    |  Base Relocation Table (Used for ASLR)
+--------------+
|    .rsrc     |  Resources (Icons, Strings, etc.)
+--------------+
```

## Memory
In modern operating systems, processes do not access physical memory directly. Instead, they use **Virtual Memory**, an abstraction provided by the CPU (MMU) and the Kernel.

### Key Concepts:
- **Isolation**: Each process has its own private virtual address space, preventing it from interfering with other processes.
- **Abstraction**: The OS can map virtual addresses to physical RAM, disk (swap), or even hardware registers.
- **Paging**: Memory is divided into small blocks called "Pages" (typically 4KB). The OS uses **Page Tables** to translate Virtual Addresses (VA) to Physical Addresses (PA).

### Virtual Address Space (x64)
On 64-bit Windows, virtual addresses are 64 bits wide, but only the lower 48 bits are currently used for translation. This creates a "canonical" address space.

- **User Space**: `0x00000000'00000000` to `0x00007FFF'FFFFFFFF` (8 TB). Where user applications live.
- **Non-canonical Gap**: Addresses from `0x00008000'00000000` to `0xFFFF7FFF'FFFFFFFF` are invalid and will cause a fault if accessed.
- **Kernel Space**: `0xFFFF8000'00000000` to `0xFFFFFFFF'FFFFFFFF` (8 TB). Where the OS kernel, drivers, and system-wide structures reside.

```text
+---------------------+  0xFFFFFFFF'FFFFFFFF
|    Kernel Space     |  (Ring 0)
+---------------------+  0xFFFF8000'00000000
|                     |
|  Non-canonical Gap  |  (Illegal Addresses)
|                     |
+---------------------+  0x00007FFF'FFFFFFFF
|     User Space      |  (Ring 3)
+---------------------+  0x00000000'00000000
```

Translation Path:
Virtual Address -> [MMU / Page Tables] -> Physical RAM / Swap / Other Storage

## Stack
The stack is a dedicated region of memory used to store temporary data, such as function parameters, local variables, and return addresses. It operates on the LIFO (Last In, First Out) principle, meaning the last value added to the stack is the first one to be removed—similar to a stack of plates. In 32-bit systems, the stack is also used to pass function arguments when invoking a function.

The top of the stack, often referred to as the “stack pointer”, is tracked by the **ESP/RSP** (Extended Stack Pointer) register in 32/64 bits systems. The value in the ESP/RSP register changes dynamically as data is pushed onto or popped off the stack, ensuring efficient memory management during program execution.

**Note:** The stack grows **downwards** in memory (from higher addresses to lower addresses).

### Stack Frame Example
When a function is called, a new "Stack Frame" (or Activation Record) is created.

```text
Address     Data              Description
----------  ----------------  --------------------------
0x0019FFCC  0x00401234        Return Address (Where to go back)
0x0019FFC8  0x0019FFD8        Saved EBP (Previous frame's base)
0x0019FFC4  [Local Var 1]     Stored at EBP-4
0x0019FFC0  [Local Var 2]     Stored at EBP-8
0x0019FFBC  [Temporary]       Current Top of Stack (ESP)
```

1.  **Arguments** are pushed (in x86).
2.  **CALL** instruction pushes the Return Address.
3.  **Function Prologue**:
    - `push ebp` (Saves caller's base pointer)
    - `mov ebp, esp` (Sets current frame's base)
    - `sub esp, 0x10` (Allocates space for local variables)

## Heap
In Windows, the heap is a region of memory managed by the Heap Manager, which allows dynamic memory allocation for applications. It provides functions like `HeapAlloc()` and `HeapFree()` to allocate and deallocate memory blocks efficiently. Each process can have multiple heaps, including a default heap created at startup. The heap helps optimize memory usage by reducing fragmentation and improving performance. It is a core component of Windows memory management, sitting between low-level virtual memory and high-level allocators like the C runtime’s `malloc()`.


## Endianness
Endianness refers to the order in which bytes are stored in memory.

- **Little Endian (LE)**: The least significant byte (LSB) is stored at the lowest memory address. Used by x86 and x64 architectures.
- **Big Endian (BE)**: The most significant byte (MSB) is stored at the lowest memory address. Used by some network protocols and PowerPC/ARM (in BE mode).

### Examples (Storage in Memory)
Value: `0x12345678` (4-byte integer)

| Address | Little Endian (x86) | Big Endian |
| :--- | :--- | :--- |
| `0x1000` | `0x78` (LSB) | `0x12` (MSB) |
| `0x1001` | `0x56` | `0x34` |
| `0x1002` | `0x34` | `0x56` |
| `0x1003` | `0x12` (MSB) | `0x78` (LSB) |

- **String**: ` "ABC"` is always stored as `0x41, 0x42, 0x43, 0x00` in order, regardless of endianness (since each `char` is 1 byte).
- **Hex Address**: `0x00007FF712340000` in Little Endian memory would look like `00 00 34 12 F7 7F 00 00`.


## Registers
Registers are small, high-speed storage locations located directly inside the CPU. They are used to hold data that the processor is currently working on, such as operands for arithmetic, addresses for memory access, or state information.

### 32 bits (x86)
In 32-bit systems, there are 8 general-purpose registers (GPRs), each 4 bytes wide.

- **EAX**: Accumulator - used for arithmetic and storing function return values.
- **ECX**: Counter - used for loop iterations and string operations.
- **EDX**: Data - extension of EAX for 64-bit arithmetic; also used for I/O.
- **EBX**: Base - pointer to data in the DS segment.
- **ESI**: Source Index - used as a pointer for source strings/arrays.
- **EDI**: Destination Index - used as a pointer for destination strings/arrays.
- **EBP**: Base Pointer - points to the base of the current stack frame.
- **ESP**: Stack Pointer - points to the top of the stack.
- **EIP**: Instruction Pointer - contains the address of the next instruction to execute.

#### Register Splitting
32-bit registers can be accessed as smaller sub-registers for backward compatibility.

```text
   32 bits (EAX)
 /               \
+-------+---------+
| (H)   |    AX   | < WORD (2 Bytes)
+-------+----+----+
        | AH | AL | < BYTE (1 Byte each)
        +----+----+
```

### 64 bits (x64)
x64 expands the 32-bit registers to 64 bits (prefixed with 'R') and adds 8 new general-purpose registers (`R8`-`R15`).

- **RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP** (64-bit versions of the above).
- **R8 - R15**: Additional general-purpose registers.
- **RIP**: 64-bit Instruction Pointer.

**Register Splitting in 64-bit:**
```text
               64 bits (RAX)
 /                                       \
+-----------------------+-----------------+
|          (H)          |       EAX       | < DWORD (4 Bytes)
+-----------------------+-------+---------+
                        |  (H)  |    AX   | < WORD (2 Bytes)
                        +-------+----+----+
                                | AH | AL | < BYTE (1 Byte each)
                                +----+----+
```
- `RAX` (64 bits) -> `EAX` (32 bits) -> `AX` (16 bits) -> `AH`/`AL` (8 bits).
- `R8` (64 bits) -> `R8D` (32 bits) -> `R8W` (16 bits) -> `R8B` (8 bits).

## Call convention 32 vs 64 bits
Calling conventions define how functions receive parameters and how they return results.

### 32-bit (x86)
Common conventions include:
- **cdecl**: Arguments passed on stack (right-to-left), **Caller** cleans the stack.
- **stdcall**: Arguments passed on stack (right-to-left), **Callee** cleans the stack.
- **fastcall**: First two arguments in `ECX` and `EDX`, others on stack.

### 64-bit (Windows x64)
Windows uses a single "Fastcall-like" convention:
- **Integer/Pointer Arguments**: Passed in `RCX`, `RDX`, `R8`, and `R9` (in that order).
- **Additional Arguments**: Passed on the stack (right-to-left).
- **Return Value**: Stored in `RAX` (or `XMM0` for floats).
- **Shadow Space (Home Space)**: The caller must allocate 32 bytes of "shadow space" on the stack before the call, even if the function takes fewer than 4 arguments. This allows the callee to save the register arguments back to the stack if needed.

#### Example Call (x64)
```asm
sub rsp, 28h      ; Allocate shadow space (32 bytes) + alignment
mov rcx, 1        ; Arg 1
mov rdx, 2        ; Arg 2
mov r8, 3         ; Arg 3
mov r9, 4         ; Arg 4
call MyFunction
add rsp, 28h      ; Clean up shadow space
```


## Signed & Unsigned
In reverse engineering, distinguishing between signed and unsigned integers is critical for understanding comparison logic (`JZ`, `JG`, `JA`, etc.) and identifying potential integer overflows.

- **Unsigned**: Represents only non-negative values (0 and positive).
- **Signed**: Uses Two's Complement representation to store both positive and negative values. The most significant bit (MSB) acts as the sign bit (1 for negative, 0 for positive).

### Data Type Ranges

| Size (Bits) | Type | Range (Decimal) | Hex Range |
| :--- | :--- | :--- | :--- |
| **8** | Unsigned (Byte) | 0 to 255 | `0x00` to `0xFF` |
| **8** | Signed (SByte) | -128 to 127 | `0x80` to `0x7F` |
| **16** | Unsigned (Word) | 0 to 65,535 | `0x0000` to `0xFFFF` |
| **16** | Signed (SWord) | -32,768 to 32,767 | `0x8000` to `0x7FFF` |
| **32** | Unsigned (DWord) | 0 to 4,294,967,295 | `0x00000000` to `0xFFFFFFFF` |
| **32** | Signed (SDWord) | -2,147,483,648 to 2,147,483,647 | `0x80000000` to `0x7FFFFFFF` |
| **64** | Unsigned (QWord) | 0 to 1.84 x 10^19 | `0x00...00` to `0xFF...FF` |
| **64** | Signed (SQWord) | -9.22 x 10^18 to 9.22 x 10^18 | `0x80...00` to `0x7F...FF` |

#### Comparison Instructions Hint
- **Unsigned Comparisons**: Use `JA` (Above), `JB` (Below).
- **Signed Comparisons**: Use `JG` (Greater), `JL` (Less).

