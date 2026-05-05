# IDA Pro Cheat Sheet
Essential shortcuts and commands for navigating and reversing with IDA Pro.

## Navigation
| Key | Action |
| :--- | :--- |
| **G** | Jump to address or name. |
| **Esc** | Go back (previous location). |
| **Ctrl + Enter** | Go forward. |
| **Space** | Switch between Graph and Text view. |
| **Ctrl + P** | Open Functions window. |
| **Ctrl + L** | Open Names window. |
| **Ctrl + S** | Open Segments window. |

## Analysis and Decompilation
| Key | Action |
| :--- | :--- |
| **F5** | Decompile current function (Hex-Rays). |
| **Tab** | Switch between disassembly and pseudocode. |
| **N** | Rename variable, function, or label. |
| **Y** | Set variable or function type (e.g., `void*`, `int`). |
| **/** | Add a comment in pseudocode. |
| **;** | Add a non-repeatable comment in disassembly. |
| **:** | Add a repeatable comment in disassembly. |
| **X** | List cross-references (Xrefs) to a symbol. |

## Data Manipulation
| Key | Action |
| :--- | :--- |
| **C** | Mark as Code. |
| **D** | Mark as Data (cycle through db, dw, dd, dq). |
| **U** | Undefine (convert back to raw bytes). |
| **A** | Convert to ASCII string. |
| **O** | Convert to Offset (pointer). |
| **H** | Toggle Hex/Decimal view for a constant. |
| **M** | Convert to Enum member. |

## Searching
| Key | Action |
| :--- | :--- |
| **Alt + T** | Search for text. |
| **Alt + B** | Search for a sequence of bytes (hex). |
| **Ctrl + T** | Search next. |

## Tips
- **Right-click -> Synchronize with** : Keep the hex view or decompiler in sync with the disassembly.
- **Edit -> Plugins**: Access installed plugins like Keypatch or LazyIDA.
- **Shift + F12**: Open Strings window to find interesting text (URLs, error messages).
