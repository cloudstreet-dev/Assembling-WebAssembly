# Chapter 6: Memory Model

## Linear Memory

WebAssembly uses a linear memory model: a contiguous, byte-addressable array of memory. Think of it as a giant `uint8_t[]` that your WASM code can read from and write to.

Key characteristics:

- **Byte-addressable** - Individual bytes can be accessed
- **Zero-indexed** - First byte is at address 0
- **Little-endian** - Multi-byte values store least-significant byte first
- **Sandboxed** - Each module instance has its own isolated memory
- **Growable** - Memory can expand at runtime (but never shrink)

## Memory Pages

Memory is measured in **pages**, where one page is exactly **64KB** (65,536 bytes). This matches common OS page sizes and makes memory management efficient.

When you declare memory:

```wasm
(memory 1)      ;; 1 page minimum (64KB)
(memory 2 10)   ;; 2 pages minimum (128KB), 10 pages maximum (640KB)
```

The first number is the minimum number of pages. The optional second number is the maximum. Without a maximum, memory can grow up to the engine's limit (often 4GB, due to 32-bit addressing).

### Why Pages?

- **Efficiency** - OS memory management aligns with 4KB or 64KB pages
- **Security** - Guard pages can be placed at boundaries
- **Granularity** - 64KB is a good balance between flexibility and overhead

## Memory Declaration

A module can have at most one linear memory (multi-memory is in proposal stage):

```wasm
(module
  (memory 1)    ;; Declare memory

  (func $store_example
    i32.const 0      ;; Address
    i32.const 42     ;; Value
    i32.store        ;; Store value at address
  )
)
```

Alternatively, memory can be imported:

```wasm
(module
  (import "env" "memory" (memory 1))  ;; Import memory from host

  ;; Use imported memory
  (func $use_memory
    i32.const 100
    i32.load
  )
)
```

Or exported:

```wasm
(module
  (memory 1)
  (export "memory" (memory 0))  ;; Export memory to host
)
```

Exporting memory allows the host (e.g., JavaScript) to access WASM's memory directly. This is common for passing large data efficiently.

## Memory Instructions

### Load Instructions

Load values from memory onto the stack:

```wasm
i32.load      ;; Load 4 bytes as i32
i64.load      ;; Load 8 bytes as i64
f32.load      ;; Load 4 bytes as f32
f64.load      ;; Load 8 bytes as f64
```

Loads require an address on the stack:

```wasm
i32.const 100   ;; Address
i32.load        ;; Load i32 from address 100
```

### Partial Loads

Load fewer bytes and extend:

```wasm
i32.load8_s     ;; Load 1 byte, sign-extend to i32
i32.load8_u     ;; Load 1 byte, zero-extend to i32
i32.load16_s    ;; Load 2 bytes, sign-extend to i32
i32.load16_u    ;; Load 2 bytes, zero-extend to i32

i64.load8_s     ;; Load 1 byte, sign-extend to i64
i64.load16_u    ;; Load 2 bytes, zero-extend to i64
i64.load32_s    ;; Load 4 bytes, sign-extend to i64
i64.load32_u    ;; Load 4 bytes, zero-extend to i64
```

Example:

```wasm
i32.const 50
i32.load8_u     ;; Load byte at address 50, zero-extend to i32
```

If memory[50] contains `0xFF`, this loads `255` (unsigned). With `i32.load8_s`, it would load `-1` (signed).

### Store Instructions

Store values from stack to memory:

```wasm
i32.store      ;; Store i32 (4 bytes)
i64.store      ;; Store i64 (8 bytes)
f32.store      ;; Store f32 (4 bytes)
f64.store      ;; Store f64 (8 bytes)
```

Stores require an address and a value:

```wasm
i32.const 100   ;; Address
i32.const 42    ;; Value
i32.store       ;; Store 42 at address 100
```

### Partial Stores

Store fewer bytes:

```wasm
i32.store8     ;; Store low 8 bits
i32.store16    ;; Store low 16 bits

i64.store8     ;; Store low 8 bits
i64.store16    ;; Store low 16 bits
i64.store32    ;; Store low 32 bits
```

Example:

```wasm
i32.const 200
i32.const 0x12345678
i32.store8      ;; Store only 0x78 at address 200
```

### Offset and Alignment

Loads and stores can specify an offset and alignment hint:

```wasm
i32.const 100
i32.load offset=4 align=4   ;; Load from address 100+4 with 4-byte alignment
```

The `offset` is added to the address. This is useful for struct field access:

```wasm
;; Struct at address in $ptr:
;;   int32 field1;   // Offset 0
;;   int32 field2;   // Offset 4
;;   float field3;   // Offset 8

local.get $ptr
i32.load offset=0    ;; Load field1

local.get $ptr
i32.load offset=4    ;; Load field2

local.get $ptr
f32.load offset=8    ;; Load field3
```

The `align` is a hint (in bytes, as power of 2: align=4 means 4-byte alignment). WASM handles unaligned access correctly, but aligned access may be faster.

## Bounds Checking

Every memory access is bounds-checked. Accessing memory beyond the current size traps:

```wasm
(memory 1)    ;; 65536 bytes (0 to 65535)

i32.const 100000  ;; Address beyond memory
i32.load          ;; TRAP: out of bounds
```

Bounds checking prevents buffer overflows. There's no way to access memory you don't own.

The bounds check cost is minimal:

- Modern CPUs make bounds checks fast
- JIT compilers can optimize consecutive accesses
- Memory is guarded by OS-level guard pages

## Little-Endian Byte Order

WASM uses little-endian encoding: the least-significant byte is stored at the lowest address.

Storing `0x12345678` as i32 at address 0:

```
Address:  0    1    2    3
Value:    0x78 0x56 0x34 0x12
```

This matches x86/x64, ARM (in little-endian mode), and most modern architectures. It simplifies implementation and improves performance on common hardware.

## Memory Growth

Memory can grow at runtime using the `memory.grow` instruction:

```wasm
i32.const 1      ;; Number of pages to grow
memory.grow      ;; Returns previous size in pages, or -1 on failure
```

Example:

```wasm
;; Current size is 2 pages (128KB)
i32.const 3      ;; Grow by 3 pages
memory.grow      ;; Returns 2 (previous size), now size is 5 pages (320KB)
```

Growth can fail if:

- It would exceed the maximum (if specified)
- The host can't allocate more memory
- Implementation limits are reached

Check the result:

```wasm
i32.const 10
memory.grow
i32.const -1
i32.eq
(if
  (then
    ;; Growth failed
  )
  (else
    ;; Growth succeeded
  )
)
```

### Memory Growth Performance

Memory growth is relatively expensive:

- May require reallocating and copying
- May invalidate JIT-compiled code that made assumptions about memory size
- May trigger garbage collection in the host

Minimize growth by:

- Declaring sufficient initial memory
- Growing in larger increments (e.g., double the size)
- Avoiding growth in hot paths

## Memory Size Query

Get current memory size in pages:

```wasm
memory.size      ;; Returns current size in pages as i32
```

To get size in bytes:

```wasm
memory.size
i32.const 65536
i32.mul         ;; Size in bytes
```

## Memory Initialization

### Data Segments

Initialize memory at module instantiation:

```wasm
(module
  (memory 1)

  (data (i32.const 0) "Hello, WASM!")  ;; Write string at offset 0
  (data (i32.const 100) "\00\01\02\03")  ;; Write bytes at offset 100
)
```

The string/bytes are written to memory when the module is instantiated, before any code runs.

### Active vs. Passive Data Segments

**Active segments** (default) initialize memory automatically:

```wasm
(data (i32.const 0) "data")  ;; Runs at instantiation
```

**Passive segments** don't run automatically. You copy them explicitly:

```wasm
(module
  (memory 1)
  (data $mydata "Hello")  ;; Passive segment

  (func $init
    i32.const 0       ;; Destination offset in memory
    i32.const 0       ;; Source offset within segment
    i32.const 5       ;; Number of bytes
    memory.init $mydata  ;; Copy segment to memory
    data.drop $mydata ;; Drop segment (free the copy)
  )
)
```

Passive segments are useful for:

- Lazy initialization
- Copying data conditionally
- Reusing segments in multiple places

## Memory Isolation and Security

Each WASM instance has its own linear memory. Instances can't access each other's memory unless memory is explicitly shared via imports/exports.

This provides strong isolation:

- Untrusted code can't leak data
- Bugs in one module can't corrupt another's memory
- Host memory is inaccessible (unless via imports)

### Shared Memory (Threads)

For multi-threading, memory can be shared between instances. Shared memory is declared with the `shared` flag:

```wasm
(memory 1 10 shared)  ;; Shared memory
```

Shared memory enables threads but requires atomic operations to avoid data races (covered in Chapter 45).

## Memory Layout Strategies

Since WASM doesn't have a built-in memory allocator, you must manage memory yourself or use a library allocator.

Common strategies:

### Static Allocation

Allocate memory at compile time:

```wasm
(global $heap_ptr (mut i32) (i32.const 65536))  ;; Start at 64KB

(func $alloc (param $size i32) (result i32)
  global.get $heap_ptr     ;; Current heap pointer
  global.get $heap_ptr
  local.get $size
  i32.add
  global.set $heap_ptr     ;; Bump pointer
)
```

This "bump allocator" is fast but can't free memory.

### Stack Allocation

Use a separate "stack" for temporary allocations:

```wasm
(global $stack_ptr (mut i32) (i32.const 1048576))  ;; Start at 1MB

(func $push_i32 (param $value i32)
  global.get $stack_ptr
  local.get $value
  i32.store

  global.get $stack_ptr
  i32.const 4
  i32.add
  global.set $stack_ptr
)

(func $pop_i32 (result i32)
  global.get $stack_ptr
  i32.const 4
  i32.sub
  global.set $stack_ptr

  global.get $stack_ptr
  i32.load
)
```

### Dynamic Allocation

Use a real allocator (like dlmalloc, compiled to WASM). Languages like Rust and C++ bring their own allocators.

Typical memory layout:

```
0x00000000: Static data (global variables, constants)
0x00010000: Heap (dynamically allocated)
0x00100000: Stack (grows downward from high address)
```

## Memory Access Patterns and Performance

### Cache-Friendly Access

Modern CPUs cache memory. Sequential access is fast:

```wasm
;; Good: sequential access
local.get $base
i32.load offset=0

local.get $base
i32.load offset=4

local.get $base
i32.load offset=8
```

```wasm
;; Bad: random access
local.get $random_addr1
i32.load

local.get $random_addr2
i32.load

local.get $random_addr3
i32.load
```

### Alignment

Aligned access is faster on many architectures:

```wasm
;; Fast: 4-byte aligned
i32.const 0
i32.load align=4

;; Potentially slower: unaligned
i32.const 1
i32.load align=4  ;; Address 1 is not 4-byte aligned
```

WASM handles unaligned access correctly, but it may be slower.

## Common Patterns

### Strings

C-style null-terminated strings:

```wasm
(data (i32.const 0) "Hello\00")

(func $strlen (param $ptr i32) (result i32)
  (local $len i32)
  i32.const 0
  local.set $len

  (block $break
    (loop $continue
      local.get $ptr
      local.get $len
      i32.add
      i32.load8_u
      i32.eqz
      br_if $break      ;; Break if null byte

      local.get $len
      i32.const 1
      i32.add
      local.set $len
      br $continue
    )
  )

  local.get $len
)
```

Or length-prefixed strings:

```
Offset 0: Length (4 bytes)
Offset 4: Character data
```

### Arrays

Store length followed by elements:

```
Offset 0: Length (4 bytes)
Offset 4: Element 0
Offset 8: Element 1
...
```

### Structs

Pack fields sequentially:

```c
// C struct
struct Point {
    int32_t x;    // Offset 0
    int32_t y;    // Offset 4
};
```

```wasm
;; Access Point.x
local.get $point_ptr
i32.load offset=0

;; Access Point.y
local.get $point_ptr
i32.load offset=4
```

## Debugging Memory Issues

Common issues:

**Out-of-bounds access** - Check addresses before load/store
**Uninitialized memory** - Data segments ensure initialization
**Memory leaks** - Use proper allocator with free()
**Alignment issues** - Ensure addresses are properly aligned

Tools like `wasm-objdump` can inspect memory contents in WASM files. Debuggers in browsers show memory as a byte array.

## Next Steps

We've covered how memory works at a low level. Next, we'll examine module anatomy—how all the pieces (types, functions, memory, tables, imports, exports) fit together in a complete WASM module.
