# Chapter 4: Binary Format

## Why a Binary Format?

WebAssembly is designed to be fast to decode, validate, and compile. A text format wouldn't achieve this. The binary format is carefully designed for:

- **Compact size** - Smaller than equivalent JavaScript, even minified
- **Fast parsing** - Single-pass, no backtracking
- **Stream compilation** - Can begin compiling before entire module is received
- **Deterministic validation** - No ambiguity, fast type checking

The binary format is the canonical form. WAT is derived from it, not the other way around.

## Module Structure

A WASM binary module starts with a magic number and version:

```
0x00 0x61 0x73 0x6D  ; "\0asm" magic number
0x01 0x00 0x00 0x00  ; Version 1
```

These 8 bytes identify every WASM file. The magic number is `\0asm` in ASCII (the null byte prevents text editors from treating it as text).

After the header come sections, each with:

- **Section ID** - 1 byte identifying section type
- **Section size** - Variable-length unsigned integer (LEB128)
- **Section content** - Size bytes of data

## Section Types

Sections must appear in this order (all optional except potentially Custom):

| ID  | Section       | Purpose                                  |
| --- | ------------- | ---------------------------------------- |
| 0   | Custom        | Custom data (debug info, names, etc.)    |
| 1   | Type          | Function type signatures                 |
| 2   | Import        | Imported functions, tables, memory, globals |
| 3   | Function      | Function type indices (declarations)     |
| 4   | Table         | Table definitions                        |
| 5   | Memory        | Memory definitions                       |
| 6   | Global        | Global variable definitions              |
| 7   | Export        | Exported functions, tables, memory, globals |
| 8   | Start         | Start function index                     |
| 9   | Element       | Table initialization data                |
| 10  | Code          | Function bodies (implementation)         |
| 11  | Data          | Memory initialization data               |
| 12  | DataCount     | Number of data segments (for validation) |

Custom sections (ID 0) can appear anywhere and are ignored by the runtime unless a tool understands them. They're used for:

- Debug information
- Function and local names (for better debugging)
- Source maps
- Custom toolchain metadata

## Variable-Length Encoding (LEB128)

WASM uses LEB128 (Little Endian Base 128) for encoding integers. This saves space for small numbers while supporting large ones.

### Unsigned LEB128

Each byte encodes 7 bits of data. The high bit indicates whether more bytes follow:

- If high bit is 1: more bytes follow
- If high bit is 0: this is the last byte

Encoding 624485:

```
624485 = 0x98765 = 1001 1000 0111 0110 0101

Split into 7-bit chunks (right to left):
  0110 0101 = 0x65
  1110 0110 = 0xE6
  0001 1000 = 0x18
  0000 0100 = 0x04

Encoded (reverse order, set high bits except last):
  11100101 = 0xE5  (0x65 | 0x80)
  11001100 = 0xCC  (0x4C | 0x80)
  10011000 = 0x98  (0x18 | 0x80)
  00000100 = 0x04
```

Small numbers encode compactly:
- 0-127: 1 byte
- 128-16383: 2 bytes
- 16384-2097151: 3 bytes

### Signed LEB128

For signed integers, use two's complement and sign-extend the last byte:

- Positive numbers have 0 in the sign bit of the last 7-bit chunk
- Negative numbers have 1 in the sign bit and are sign-extended

Example: -123456

```
-123456 in 32-bit two's complement: 0xFFFE1DC0
In binary: 11111111 11111110 00011101 11000000

Extract 7-bit chunks from right:
  1000000 (0x40)
  0111011 (0x3B)
  1111000 (0x78)
  1111111 (0x7F)

Encoded (set high bits except last):
  11000000 = 0xC0
  10111011 = 0xBB
  11111000 = 0xF8
  01111111 = 0x7F
```

## Type Section

Defines all function signatures used in the module:

```
Section ID: 0x01
Size: <LEB128>
Count: <LEB128>  ; Number of type definitions
[Types]:
  Form: 0x60     ; Function type
  Param count: <LEB128>
  [Param types]: <value type> ...
  Result count: <LEB128>
  [Result types]: <value type> ...
```

Value types are encoded as:
- `0x7F` = i32
- `0x7E` = i64
- `0x7D` = f32
- `0x7C` = f64
- `0x7B` = v128
- `0x70` = funcref
- `0x6F` = externref

Example: `(func (param i32 i64) (result f32))`

```
0x60              ; Function type
0x02              ; 2 parameters
0x7F 0x7E         ; i32, i64
0x01              ; 1 result
0x7D              ; f32
```

## Function and Code Sections

Functions are split across two sections:

**Function section (3):** Lists type indices for each function
```
Section ID: 0x03
Size: <LEB128>
Count: <LEB128>
[Type indices]: <LEB128> ...  ; Index into type section
```

**Code section (10):** Contains function bodies
```
Section ID: 0x0A
Size: <LEB128>
Count: <LEB128>
[Function bodies]:
  Body size: <LEB128>
  Local count: <LEB128>
  [Locals]:
    Count: <LEB128>      ; Number of consecutive locals of this type
    Type: <value type>
  [Instructions]: <byte> ...
  End: 0x0B
```

Why split? It allows parallel compilation. Engines can allocate space and plan compilation while still receiving the code section.

## Instructions

Instructions are encoded compactly. Common instructions are single bytes:

- `0x20` = local.get
- `0x21` = local.set
- `0x6A` = i32.add
- `0x6B` = i32.sub
- `0x6C` = i32.mul

Some instructions have immediate operands:

```wasm
i32.const 42
```

Encodes as:
```
0x41        ; i32.const opcode
0x2A        ; Value 42 in signed LEB128
```

```wasm
local.get 5
```

Encodes as:
```
0x20        ; local.get opcode
0x05        ; Local index 5
```

## Control Flow Encoding

Blocks have a type indicating what they produce:

```wasm
(block (result i32) ... end)
```

Encodes as:
```
0x02        ; block opcode
0x7F        ; Result type: i32
...         ; Block contents
0x0B        ; end opcode
```

If a block produces nothing:
```
0x02        ; block opcode
0x40        ; Empty type
...
0x0B        ; end
```

Branches encode label depth:

```wasm
br 2  ; Branch to 2nd enclosing block
```

```
0x0C        ; br opcode
0x02        ; Depth 2
```

Branch tables:

```wasm
br_table $case0 $case1 $default
```

```
0x0E                    ; br_table opcode
0x02                    ; Number of targets (not including default)
0x00 0x01              ; Target depths
0x02                    ; Default target depth
```

## Memory and Data Sections

**Memory section (5):**
```
Section ID: 0x05
Size: <LEB128>
Count: <LEB128>  ; Usually 1 (multi-memory is proposal)
[Memories]:
  Limits flag: <byte>   ; 0x00 = min only, 0x01 = min and max
  Min: <LEB128>         ; Minimum pages
  [Max: <LEB128>]       ; Maximum pages (if flag = 0x01)
```

**Data section (11):**
```
Section ID: 0x0B
Size: <LEB128>
Count: <LEB128>
[Data segments]:
  Mode: <byte>          ; 0 = active, 1 = passive, 2 = active with memidx
  [Memory index: <LEB128>]  ; If mode = 2
  [Offset expr]: ...    ; If mode = 0 or 2
    i32.const <offset>
    end
  Size: <LEB128>
  [Data]: <byte> ...
```

Active segments initialize memory at instantiation. Passive segments can be copied later using `memory.init`.

## Table and Element Sections

**Table section (4):**
```
Section ID: 0x04
Size: <LEB128>
Count: <LEB128>
[Tables]:
  Type: <reftype>       ; 0x70 = funcref, 0x6F = externref
  Limits flag: <byte>
  Min: <LEB128>
  [Max: <LEB128>]
```

**Element section (9):**
```
Section ID: 0x09
Size: <LEB128>
Count: <LEB128>
[Element segments]:
  Mode: <byte>
  [Table index: <LEB128>]
  [Offset expr]: ...
  Type: <reftype>
  Count: <LEB128>
  [Init exprs]: ...
```

Element segments initialize table entries with function references or external references.

## Import Section

```
Section ID: 0x02
Size: <LEB128>
Count: <LEB128>
[Imports]:
  Module name length: <LEB128>
  Module name: <UTF-8 bytes>
  Field name length: <LEB128>
  Field name: <UTF-8 bytes>
  Kind: <byte>          ; 0 = func, 1 = table, 2 = memory, 3 = global
  [Type]:
    If func: type index <LEB128>
    If table: table type
    If memory: limits
    If global: value type + mutability
```

Example: `(import "env" "log" (func $log (param i32)))`

```
0x03 "env"              ; Module name (length + bytes)
0x03 "log"              ; Field name
0x00                    ; Kind: function
0x00                    ; Type index 0
```

## Export Section

```
Section ID: 0x07
Size: <LEB128>
Count: <LEB128>
[Exports]:
  Name length: <LEB128>
  Name: <UTF-8 bytes>
  Kind: <byte>          ; 0 = func, 1 = table, 2 = memory, 3 = global
  Index: <LEB128>       ; Index into respective index space
```

Example: `(export "add" (func 0))`

```
0x03 "add"              ; Name
0x00                    ; Kind: function
0x00                    ; Function index 0
```

## Custom Sections

Custom sections have:

```
Section ID: 0x00
Size: <LEB128>
Name length: <LEB128>
Name: <UTF-8 bytes>
[Content]: <arbitrary bytes>
```

The "name" custom section is standard and provides names for debugging:

```
Name: "name"
Subsections:
  0: Module name
  1: Function names
  2: Local names
  ...
```

This is how debuggers know to show `$add` instead of `func[0]`.

## Parsing Strategy

WASM is designed for single-pass parsing:

1. Read header (magic + version)
2. For each section:
   - Read section ID
   - Read section size
   - Read/skip section content
3. Validate
4. Compile

Sections can be compiled in parallel once validated. Unknown sections are safely skipped using the size field.

## Size Optimization

The binary format optimizes for size:

- **LEB128 encoding** - Small numbers use fewer bytes
- **Shared type definitions** - Functions with same signature reference one type
- **Implicit typing** - Stack types are inferred, not encoded
- **Dense instruction encoding** - Common instructions are 1 byte

A simple add function compiles to ~20 bytes total, including headers.

## Validation

The binary format enables fast validation:

- **Type checking**: Stack types are known from instruction signatures
- **Bounds checking**: Indices are validated against counts
- **Control flow**: Block nesting is validated during parsing
- **No forward references**: Everything is defined before use

Invalid modules are rejected before execution begins.

## Binary Example

Here's `(func (param i32) (result i32) local.get 0 i32.const 1 i32.add)` in full binary:

```
00 61 73 6D    ; Magic number
01 00 00 00    ; Version 1

01             ; Type section ID
06             ; Section size
01             ; 1 type
60             ; Function type
01 7F          ; 1 param: i32
01 7F          ; 1 result: i32

03             ; Function section ID
02             ; Section size
01             ; 1 function
00             ; Type index 0

0A             ; Code section ID
09             ; Section size
01             ; 1 function body
07             ; Body size
00             ; 0 locals
20 00          ; local.get 0
41 01          ; i32.const 1
6A             ; i32.add
0B             ; end
```

Total: 31 bytes for a complete module.

## Streaming Compilation

Because sections have size fields, engines can:

1. Allocate memory for code as soon as they read the Code section header
2. Begin compiling functions as they arrive
3. Validate and compile in parallel

By the time downloading finishes, compilation is often complete. This is why WASM loads so much faster than JavaScript.

## Tools

**wabt** (WebAssembly Binary Toolkit) provides:
- `wasm-objdump` - Disassemble WASM binaries
- `wasm-strip` - Remove debug information
- `wasm2wat` / `wat2wasm` - Convert between formats

**wasm-opt** (from Binaryen) optimizes WASM binaries.

Example:
```bash
wasm-objdump -d module.wasm     # Disassemble
wasm-strip module.wasm          # Strip names/debug info
wasm-opt -O3 module.wasm -o optimized.wasm
```

## Next Steps

Understanding the binary format demystifies WASM. You can now reason about module size, understand how parsers work, and debug issues at the bytecode level. Next, we'll explore the type system in depth—how WASM ensures memory safety through static typing.
