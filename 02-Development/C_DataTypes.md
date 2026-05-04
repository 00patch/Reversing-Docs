# C Data Types in Reversing
Understanding how C data types are represented in memory and manipulated at the assembly level is fundamental for reconstructing code during reverse engineering.

## Arrays
An array is a contiguous block of memory containing elements of the same type.
- **In Memory**: Elements are placed back-to-back.
- **In Assembly**: Accessed using a base address and an index.
    - Example: `arr[i]` often looks like `[base + index * sizeof(type)]`.
    - If `eax` is the index and `int arr[10]` starts at `ebp-0x40`, access might be `mov edx, [ebp + eax*4 - 0x40]`.

## Structs (Structures)
A struct is a contiguous block of memory that can contain different data types.
- **In Memory**: Fields are stored in the order they are defined.
- **Alignment/Padding**: The compiler often adds "padding" bytes between fields to ensure they align with the CPU's word size (e.g., 4 or 8 bytes).
- **In Assembly**: Fields are accessed via offsets from the base address of the struct.
    - `struct.field2` -> `[base + offset_of_field2]`.
    - Identifying a struct: Look for multiple `mov` instructions to different offsets from the same base pointer (e.g., `[ebx+0]`, `[ebx+4]`, `[ebx+8]`).

## Unions
A union is a memory location that is shared by several different variables, which can be of different types.
- **In Memory**: Only enough space for the largest member is allocated. All members start at the same memory address.
- **In Assembly**: You will see the same memory address being treated as different types (e.g., accessed as a `float` in one place and an `int` in another).

## Pointers
A pointer is a variable that stores the memory address of another value.
- **In Memory**: On a 32-bit system, a pointer is 4 bytes. On a 64-bit system, it is 8 bytes.
- **In Assembly**:
    - **Single Pointer**: `mov eax, [ebp-4]` (where `ebp-4` holds the address).
    - **Dereferencing**: `mov eax, [ebp-4]` then `mov edx, [eax]` (gets the value at the address).
    - **Pointer to Pointer**: `mov eax, [ebp-4]`, `mov edx, [eax]`, `mov ecx, [edx]`.

## Summary Table: Identifying Types
| C Construction | Common Assembly Pattern |
| :--- | :--- |
| **Array Access** | `[base + reg * scale]` (Scale = 1, 2, 4, 8) |
| **Struct Member** | `[reg + constant_offset]` (e.g., `[esi+0x10]`) |
| **Pointer Deref** | `mov reg1, [mem]` followed by `mov reg2, [reg1]` |
| **Local Variable** | `[ebp - offset]` or `[esp + offset]` |
| **Global Variable** | `[absolute_address]` (e.g., `[0x00403000]`) |
