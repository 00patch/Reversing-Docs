# WinDbg Cheat Sheet

WinDbg is the primary debugger for Windows, capable of both user-mode and kernel-mode debugging.

---

## 1. Execution Control
| Command | Action | Keyboard |
| :--- | :--- | :--- |
| **g** | Go (Continue execution) | **F5** |
| **p** | Step Over (Execute next instruction) | **F10** |
| **t** | Step Into (Enter function call) | **F11** |
| **gu** | Go Up (Execute until current function returns) | **Shift+F11**|
| **pct** | Step until next Call or Return | |
| **q** | Quit Debugging | |

---

## 2. Breakpoints
### Software Breakpoints
- **bp <addr>**: Set breakpoint at address or symbol.
- **bl**: List all current breakpoints.
- **bc <id>**: Clear (delete) a specific breakpoint.
- **bd <id>**: Disable a breakpoint.
- **be <id>**: Enable a breakpoint.

### Hardware Breakpoints (Processor Access)
- **ba <type><size> <addr>**: Break on Access.
    - Types: **e** (execute), **r** (read/write), **w** (write only).
    - Sizes: **1, 2, 4, 8** (bytes).
    - Example: `ba w4 00401234` (Break when 4 bytes at address are written).

---

## 3. Memory and Registers
### Registers
- **r**: Show all general-purpose registers.
- **r @rax**: Show value of RAX.
- **r @rax=1**: Modify value of RAX to 1.

### Memory Inspection
- **db / dw / dd / dq**: Display bytes / words / dwords / qwords.
- **da / du**: Display ASCII / Unicode strings.
- **dyb**: Display binary.
- **dps**: Display pointers and symbols (useful for stack/vftables).
- **!address <addr>**: Show memory region properties (Protect, State, Type).

---

## 4. Modules and Symbols
- **lm**: List loaded modules.
- **lm m <pattern>**: List modules matching a name (e.g., `lm m nt*`).
- **x <module>!<symbol>**: Examine symbols (e.g., `x ntdll!NtCreate*`).
- **.reload /f**: Force-reload symbols.
- **.reload /u <module>**: Unload symbols for a module.
- **.symfix <dir>**: Setup directory symbols.
- **.sympath**: Show the current symbols path.

---

## 5. Kernel-Mode Specific
- **!process 0 0**: List all active processes in the system.
- **!process -1 0**: Show current process context.
- **!thread**: Show current thread info.
- **.process /i <addr>**: Switch context to a specific process (requires `g` to complete).
- **!pcr**: Show Processor Control Region (KPCR).
- **!idt**: Show Interrupt Descriptor Table.
- **!devnode 0 1**: Show device tree.

---

## 6. Visualizing WinDbg Output

### Address Space Separation
When listing modules (`lm`) or inspecting memory, the address prefix indicates the context:

```text
Address Range              Context
-------------------------  --------------------------
00000000`00000000 - ...    User-Mode (Ring 3)
ffff8000`00000000 - ...    Kernel-Mode (Ring 0)
```

### Current Process Context (`!process -1 0`)
Shows the metadata of the process currently being debugged:

```text
PROCESS ffff9d8b74681080
    SessionId: 1  Cid: 04a4    Peb: 004c3000  ParentCid: 0254
    DirBase: 106ea002  ObjectTable: ffffd20b41046700  HandleCount: 154.
    Image: notepad.exe
```
- **PROCESS**: Address of the `EPROCESS` structure.
- **Cid**: Process ID (PID) in hex.
- **DirBase**: Page Directory Base (CR3 register).

### Changing Context (`.process /i`)
To debug a user-mode process from a kernel session, you must switch context:

1. Find the EPROCESS: `!process 0 0 myapp.exe`
2. Set context: `.process /i <EPROCESS_ADDR>`
3. Continue: `g` (The debugger needs to execute a small amount of code to perform the swap).

---

## 7. Useful Extensions and Shortcuts
- **!gle**: Show GetLastError() value for the current thread.
- **!teb / !peb**: Show Thread/Process Environment Block.
- **.cls**: Clear screen.
- **?? <expr>**: Evaluate C++ expression (e.g., `?? sizeof(nt!_EPROCESS)`).

---
*Reference: [WinDbg CheatSheet (f1zm0)](https://github.com/f1zm0/WinDBG-Cheatsheet)*
