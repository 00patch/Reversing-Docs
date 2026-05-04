# Assembler Instructions
Assembly language is a low-level programming language that provides a symbolic representation of the machine code instructions used by a specific CPU architecture (e.g., x86, x64).

## Instruction Structure
An assembly instruction typically consists of an **Opcode** and one or more **Operands**.

### Opcodes (Operation Codes)
The opcode is the part of the instruction that specifies the operation to be performed (e.g., `MOV`, `ADD`, `PUSH`). 
- **At the low level**: Opcodes are represented by specific byte values. For example, `0x90` is the opcode for `NOP` (No Operation) in x86.
- **Prefixes**: Opcodes can have prefixes (like `0x66` for operand-size override) that modify their behavior.

### Operands
Operands are the data or locations on which the opcode operates. There are three main types:
1.  **Immediate**: A constant value hardcoded into the instruction.
    - Example: `MOV EAX, 10` (10 is an immediate value).
2.  **Register**: A reference to one of the CPU's internal registers.
    - Example: `MOV EAX, EBX` (Both are register operands).
3.  **Memory**: A reference to a memory address, often enclosed in brackets `[]`.
    - Example: `MOV EAX, [0x00403000]` (Memory operand).
    - Example: `MOV EAX, [EBX + 4]` (Relative memory access).

## Operand Formats (Intel Syntax)
In Intel syntax (common in Windows debuggers like x64dbg):
- `OPCODE DESTINATION, SOURCE`
- Example: `MOV EAX, EBX` (Copies EBX into EAX).
- **Size Directives**: When the size of an operand is ambiguous, directives are used:
    - `BYTE PTR [addr]` (1 byte)
    - `WORD PTR [addr]` (2 bytes)
    - `DWORD PTR [addr]` (4 bytes)
    - `QWORD PTR [addr]` (8 bytes)

## Assembler Logics
| Instruction | Description |
| :--- | :--- |
| **AND** | Performs a bitwise AND between two operands. |
| **OR** | Performs a bitwise inclusive OR between two operands. |
| **XOR** | Performs a bitwise exclusive OR between two operands. |
| **NOT** | Performs a bitwise NOT (one's complement) on a single operand. |

## Assembler Instructions (Data Movement)
| Instruction | Description |
| :--- | :--- |
| **MOV** | Copies the content of the source to the destination. Format: `MOV Dest, Src`. |
| **MOVSX** | Move with Sign-Extension. Copies the source to the destination and fills the remaining bits with the sign bit. |
| **MOVZX** | Move with Zero-Extend. Copies the source to the destination and fills the remaining bits with zeros. |
| **LEA** | Load Effective Address. Calculates the address of the source and stores it in the destination without accessing memory. |
| **XCHG** | Exchanges the values between two operands. |

## Assembler: Arithmetic Instructions
| Instruction | Description |
| :--- | :--- |
| **INC** | Increments the operand by 1. |
| **DEC** | Decrements the operand by 1. |
| **ADD** | Adds two operands and stores the result in the first. |
| **SUB** | Subtracts the second operand from the first and stores the result in the first. |
| **MUL** | Unsigned Multiply. Multiplies RAX by the operand; result in RDX:RAX. |
| **IMUL** | Signed Multiply. Multiplies operands and stores the result in a register. |
| **DIV** | Unsigned Divide. Divides RDX:RAX by the operand; result in RAX (quotient), RDX (remainder). |
| **IDIV** | Signed Divide. Similar to DIV but considers the sign. |
| **CMP** | Compares two operands by subtracting them (does not store result) and sets flags. |
| **TEST** | Logical Compare. Performs bitwise AND (does not store result) and sets flags (ZF, SF, PF). Often used to check if a register is 0 (e.g., `TEST EAX, EAX`). |

## Assembler: Jump Instructions
| Instruction | Description |
| :--- | :--- |
| **JMP** | Unconditional Jump to a specified address. |
| **JE / JZ** | Jump if Equal / Jump if Zero (ZF=1). |
| **JNE / JNZ** | Jump if Not Equal / Jump if Not Zero (ZF=0). |
| **JA** | Jump if Above (Unsigned comparison). |
| **JB** | Jump if Below (Unsigned comparison). |
| **JG** | Jump if Greater (Signed comparison). |
| **JL** | Jump if Less (Signed comparison). |
| **SHL** | Shift Left. Multiplies operand by 2 per shift. |
| **SHR** | Shift Right (Unsigned). Divides operand by 2 per shift. |

## Assembler: Stack Instructions
| Instruction | Description |
| :--- | :--- |
| **PUSH** | Pushes a value onto the stack. Decrements ESP/RSP. |
| **POP** | Pops a value from the stack into the operand. Increments ESP/RSP. |
| **PUSHAD** | Pushes all general-purpose registers (EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI) onto the stack (x86). |
| **POPAD** | Pops all general-purpose registers from the stack in reverse order (x86). |
| **PUSHFQ** | Pushes the RFLAGS register onto the stack (x64). |
| **POPFQ** | Pops a value from the stack into the RFLAGS register (x64). |
| **CALL** | Calls a procedure. Pushes the return address onto the stack and jumps to the destination. |
| **RET** | Returns from a procedure. Pops the return address from the stack and jumps to it. |

## Assembler: Loop and String Instructions
| Instruction | Description |
| :--- | :--- |
| **LOOP** | Decrements RCX and jumps if RCX != 0. |
| **LOOPE / LOOPZ** | Loop while RCX != 0 and ZF=1. |
| **LOOPNE / LOOPNZ** | Loop while RCX != 0 and ZF=0. |
| **REP** | Repeat prefix. Repeats the following string operation RCX times. |
| **REPE / REPZ** | Repeat while Equal/Zero. Repeats while RCX != 0 and ZF=1. |
| **REPNE / REPNZ** | Repeat while Not Equal/Not Zero. Repeats while RCX != 0 and ZF=0. |
| **MOVS** | Move Data from String to String. Copies from `[RSI]` to `[RDI]`. (`MOVSB/W/D/Q`) |
| **LODS** | Load String. Copies from `[RSI]` to `RAX`. (`LODSB/W/D/Q`) |
| **STOS** | Store String. Copies from `RAX` to `[RDI]`. (`STOSB/W/D/Q`) |
| **CMPS** | Compare String Operands. Compares `[RSI]` with `[RDI]` and sets flags. |
| **SCAS** | Scan String. Compares `RAX` with `[RDI]` and sets flags. |


## References
- [x86/x64 Instruction Set Reference (Opcodes)](http://ref.x86asm.net/coder64.html)

# Breakpoints
Breakpoints are strategic interruption points that allow a reverse engineer to stop the execution of a program to inspect its state (registers, memory, stack).

## How Breakpoints Work
At the CPU level, a breakpoint is a mechanism that triggers an exception, which the operating system then passes to the debugger.

### 1. Software Breakpoints (`INT 3`)
These are the most common type of breakpoints.
- **Mechanism**: The debugger replaces the first byte of an instruction with the opcode `0xCC` (the `INT 3` instruction).
- **Execution**: When the CPU hits `0xCC`, it generates a "Breakpoint Exception" (Exception 3). The debugger catches this, restores the original byte, and pauses execution.
- **Pros**: Unlimited number of breakpoints.
- **Cons**: Modifies the code in memory (can be detected by anti-debugging checks like checksums).

### 2. Hardware Breakpoints
Hardware breakpoints use dedicated debug registers within the CPU (`DR0`, `DR1`, `DR2`, and `DR3`).
- **Mechanism**: The memory address is stored in one of the debug registers. The `DR7` register controls the condition (Read, Write, or Execute) and the size (1, 2, 4, or 8 bytes).
- **Conditions**:
    - **On Execution**: Triggers when the EIP/RIP reaches the specified address.
    - **On Write**: Triggers only when the memory at the address is modified.
    - **On Access (Read/Write)**: Triggers when the memory is either read or written.
- **Pros**: Does not modify the code (invisible to checksums). Can break on data access, not just execution.
- **Cons**: Limited to 4 breakpoints per thread.

### 3. Memory Breakpoints
These are implemented by the debugger using the OS's memory protection features (e.g., `VirtualProtect` on Windows).
- **Mechanism**: The debugger marks a page of memory as "No Access" or "Guard Page".
- **Execution**: When the program tries to access any address in that page, a "Page Fault" or "Guard Page Violation" occurs. The debugger catches the exception and determines if the specific address matches the breakpoint.
- **Pros**: Can cover large areas of memory.
- **Cons**: Slower than hardware breakpoints as they trigger for any access to the entire page.

## Summary Table: Breakpoint Comparison
| Type | Detection Difficulty | Limit | Can Break on Data? |
| :--- | :--- | :--- | :--- |
| **Software** | Easy (Code change) | Unlimited | No |
| **Hardware** | Moderate | 4 | Yes |
| **Memory** | High | Implementation-defined| Yes |
