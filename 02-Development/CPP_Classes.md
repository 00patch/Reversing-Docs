# C++ Classes and Inheritance Under the Hood
In reverse engineering, understanding how C++ objects are laid out in memory is crucial for identifying class structures, virtual functions, and inheritance hierarchies.

## Polymorphism and the `vftable`
When a class defines or inherits at least one `virtual` function, the compiler creates a **Virtual Function Table (vftable)**.

- **vptr (Virtual Pointer)**: Every instance of a class with virtual functions contains a hidden pointer (usually at the very beginning of the object) that points to the `vftable`.
- **vftable**: A static array of function pointers stored in the `.rdata` section.

### Memory Layout (Single Inheritance)
```cpp
class Base {
public:
    int a;
    virtual void func1();
};

class Derived : public Base {
public:
    int b;
    virtual void func1() override; // Overwrites Base::func1 in vftable
    virtual void func2();          // Added to the end of the vftable
};
```

**Object Layout in Memory:**
```text
+-----------------------+
| vptr (4/8 bytes)      | ---> [ vftable ]
+-----------------------+      | &Derived::func1 |
| Base::a (4 bytes)     |      | &Derived::func2 |
+-----------------------+
| Derived::b (4 bytes)  |
+-----------------------+
```

## Multiple Inheritance
When a class inherits from multiple base classes, it will have multiple `vptr`s—one for each base class that has virtual functions.

### Memory Layout (Multiple Inheritance)
```text
+-----------------------+
| vptr_Base1            | ---> [ vftable_Base1 ]
+-----------------------+
| Base1 Data            |
+-----------------------+
| vptr_Base2            | ---> [ vftable_Base2 ]
+-----------------------+
| Base2 Data            |
+-----------------------+
| Derived Data          |
+-----------------------+
```

## Virtual Inheritance and the Diamond Problem
The **Diamond Problem** occurs when two classes (B and C) inherit from the same base class (A), and a fourth class (D) inherits from both B and C. Without virtual inheritance, class D would contain two separate copies of class A's data.

### `vbtable` (Virtual Base Table)
Virtual inheritance ensures only one copy of the base class exists. This is achieved using a **vbtable**.

- **vbptr**: A pointer to the `vbtable`, which contains offsets to the shared virtual base class data.

### Memory Layout (Virtual Inheritance)
```text
+-----------------------+
| vbptr                 | ---> [ vbtable ]
+-----------------------+      | Offset to Virtual Base |
| Class Data            |
+-----------------------+
| ...                   |
+-----------------------+
| Virtual Base Data     | <--- Shared copy
+-----------------------+
```

## RTTI (Run-Time Type Information)
RTTI allows the program to retrieve the actual type of an object at runtime (used by `dynamic_cast` and `typeid`).

- **Complete Object Locator (COL)**: In MSVC, the `vftable` is preceded by a pointer to a `RTTICompleteObjectLocator` structure. This structure contains metadata about the class hierarchy and the type name.

### Identifying RTTI in a Debugger
Look for a pointer at `[vftable - 4]` (x86) or `[vftable - 8]` (x64). This leads to the type descriptor string (e.g., `.?.AVDerived@@`).

## References
- [C++ Reversing: The MSVC ABI](https://www.openrce.org/articles/files/jangrayhood.pdf)
- [MSVC Virtual Inheritance: Obscure Members](https://blog.litneet64.com/post/2026/03/msvc-virtual-inheritance-obscure-members/)
- [Virtual Inheritance Case Study: Chimera](https://blog.litneet64.com/post/2026/04/virtual-inheritance-case-study-chimera/)
