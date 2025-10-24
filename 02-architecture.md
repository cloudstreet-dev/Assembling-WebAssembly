# Chapter 2: Architecture

## The Stack Machine Model

WebAssembly is a stack-based virtual machine. Understanding this is fundamental to everything that follows.

### Why Stack-Based?

Most physical CPUs are register machines—they have named registers (RAX, RBX, etc.) and instructions explicitly specify which registers to use. WASM instead uses an implicit operand stack.

Consider adding two numbers:

**Register machine (x86):**
```asm
mov eax, 5      ; Put 5 in register eax
mov ebx, 10     ; Put 10 in register ebx
add eax, ebx    ; Add ebx to eax, result in eax
```

**Stack machine (WASM):**
```wasm
i32.const 5     ;; Push 5 onto stack
i32.const 10    ;; Push 10 onto stack
i32.add         ;; Pop two values, push sum
```

Stack-based architectures have advantages for a portable VM:

1. **Compact encoding** - No need to encode register numbers in instructions
2. **Simple validation** - Type checking is straightforward
3. **Easy compilation** - Modern compilers excel at register allocation from stack code
4. **Small bytecode** - Instructions are often single bytes

The performance cost is negligible. JIT compilers translate stack operations to register operations during compilation. By the time WASM executes, it's using physical registers.

### The Value Stack

WASM functions operate on an implicit value stack. Instructions push values onto the stack and pop values from it. The stack is strongly typed—we always know what type each stack slot contains.

Example: `(a + b) * c`

```wasm
local.get 0   ;; Push local variable 'a' [a]
local.get 1   ;; Push local variable 'b' [a, b]
i32.add       ;; Pop two, push sum     [a+b]
local.get 2   ;; Push local variable 'c' [a+b, c]
i32.mul       ;; Pop two, push product [(a+b)*c]
```

The stack annotations show what's on the stack after each instruction.

### Structured Control Flow

WASM doesn't have goto or arbitrary jumps. Instead, it has structured control flow: blocks, loops, and conditionals. Each control structure is a region with a defined entry point and exit points.

```wasm
(block $break
  (loop $continue
    ;; Loop body
    local.get $i
    i32.const 10
    i32.lt_s
    br_if $continue  ;; Continue if i < 10
  )
)
```

This makes validation fast and enables optimizations. It's impossible to jump into the middle of a block from outside—control flow is well-defined.

## The Instruction Set

WASM's instruction set is minimal and orthogonal. There are roughly 180 instructions organized into categories:

### Numeric Instructions

- **Integer arithmetic**: `i32.add`, `i64.sub`, `i32.mul`, `i64.div_s` (signed), `i64.div_u` (unsigned)
- **Floating-point**: `f32.add`, `f64.div`, `f32.sqrt`, `f64.min`, `f64.max`
- **Bitwise**: `i32.and`, `i64.or`, `i32.xor`, `i32.shl`, `i64.shr_u`
- **Comparison**: `i32.eq`, `i64.lt_s`, `f32.le`, `f64.ge`

Instructions are typed by operands: `i32` (32-bit integer), `i64` (64-bit integer), `f32` (32-bit float), `f64` (64-bit float).

### Memory Instructions

- **Load**: `i32.load`, `i64.load`, `f32.load`, `f64.load`
- **Store**: `i32.store`, `i64.store`, `f32.store`, `f64.store`
- **Partial loads**: `i32.load8_s` (load 8 bits, sign-extend), `i32.load16_u` (load 16 bits, zero-extend)
- **Memory operations**: `memory.size`, `memory.grow`

Memory is byte-addressable. Loads and stores can specify alignment.

### Control Flow

- **Blocks**: `block`, `loop`, `if`/`else`/`end`
- **Branches**: `br` (unconditional), `br_if` (conditional), `br_table` (switch/case)
- **Calls**: `call`, `call_indirect`
- **Returns**: `return`

### Variable Access

- **Locals**: `local.get`, `local.set`, `local.tee` (set and leave value on stack)
- **Globals**: `global.get`, `global.set`

### Parametric Instructions

- **Stack manipulation**: `drop` (discard top value), `select` (ternary operator)

### Table Operations

- **Tables**: `table.get`, `table.set`, `table.size`, `table.grow`

Used for function references and indirect calls.

### SIMD (Vector Instructions)

- **Vector operations**: `v128.load`, `i32x4.add`, `f32x4.mul`, etc.

128-bit vector operations for data parallelism.

### Atomic Instructions

- **Atomic operations**: `i32.atomic.load`, `i64.atomic.rmw.add`, `memory.atomic.notify`

For threading and shared memory.

## Types

WASM has a simple type system:

### Value Types

- `i32` - 32-bit integer (signed or unsigned, depends on operation)
- `i64` - 64-bit integer
- `f32` - 32-bit IEEE 754 floating-point
- `f64` - 64-bit IEEE 754 floating-point
- `v128` - 128-bit vector (SIMD)
- `funcref` - reference to a function
- `externref` - opaque reference to host object

### Function Types

Functions are typed by their signature: `(param i32 i32) (result i64)` means "takes two i32 parameters, returns one i64".

Function types enable static validation. You can't call a function with the wrong types or wrong number of arguments.

### Limits

Used for memory and tables: `{ min: 1, max: 10 }` means minimum 1 page/entry, maximum 10.

## Memory Model

WASM has a linear memory model—a contiguous array of bytes. Memory is measured in pages (64KB each). Modules can have at most one memory (though multi-memory is in proposal stage).

Memory is sandboxed. Each module instance has its own memory space. Modules can't access each other's memory unless memory is explicitly shared via imports/exports.

### Addressing

Memory is byte-addressed. Loads and stores specify an offset and alignment:

```wasm
i32.const 0      ;; Base address
i32.load offset=4 align=4  ;; Load from address 0+4 with 4-byte alignment
```

Alignment is a hint for optimization. The runtime will still handle unaligned accesses correctly, just potentially slower.

### Bounds Checking

All memory accesses are bounds-checked. Accessing memory beyond the current size traps (throws an exception). This prevents buffer overflows and memory corruption.

### Growth

Memory can grow at runtime:

```wasm
i32.const 1       ;; Number of pages to grow
memory.grow       ;; Returns previous size or -1 on failure
```

Growth can fail if it would exceed the maximum limit or if the runtime can't allocate more memory.

## Tables

Tables hold references—either function references or external references. They enable indirect function calls and interfacing with host objects.

Why not put function references in memory? Because that would enable forging function pointers, breaking security. Tables ensure function references are valid.

```wasm
i32.const 2       ;; Index into table
call_indirect (type $sig)  ;; Call function at table[2] with signature $sig
```

Tables are also bounds-checked. Tables can grow like memory.

## Modules

A WASM module is the unit of deployment. It contains:

- **Type definitions** - Function signatures
- **Function declarations** - Which functions exist, their types
- **Tables** - Function or external reference tables
- **Memory** - Linear memory definition
- **Globals** - Global variables
- **Exports** - What this module exposes
- **Imports** - What this module requires
- **Code** - The actual function bodies
- **Data segments** - Initial memory contents
- **Element segments** - Initial table contents

Modules are validated before execution. Invalid modules are rejected—they never run.

## Validation

WASM is validated in a single linear pass over the bytecode. Validation ensures:

- All types match correctly
- Stack is balanced at the end of functions
- Control flow is well-structured
- Memory and table accesses are within bounds expressions
- Functions are called with correct signatures

If validation passes, execution is safe. No type errors, no memory corruption, no undefined behavior.

This is enforced by the specification. You can't have a conforming WASM engine that skips validation.

## Compilation Pipeline

Here's what happens when a WASM module is loaded:

1. **Parse** - Decode the binary format
2. **Validate** - Check all constraints
3. **Compile** - Generate machine code (JIT or AOT)
4. **Instantiate** - Allocate memory, initialize globals, run start function
5. **Execute** - Run exported functions

Modern engines stream-compile: they begin compiling as bytes arrive. By the time downloading finishes, compilation is mostly done.

## Determinism

WASM is mostly deterministic—the same inputs produce the same outputs. There are exceptions:

- Non-deterministic features are explicitly marked (like floating-point NaN patterns)
- Host imports can be non-deterministic
- Out-of-resources errors (out of memory) aren't deterministic

This makes WASM suitable for blockchain smart contracts and other contexts requiring reproducibility.

## The Embedding API

WASM isn't standalone—it embeds in a host environment. The embedding API defines:

- How modules are loaded and compiled
- How instances are created
- How functions are called between host and WASM
- How memory is accessed
- Error handling

Different hosts (browsers, Node.js, wasmtime) implement this API differently, but the semantics are standardized.

## Version 1 vs. Future

The current version (MVP - Minimal Viable Product) is widely deployed. Future features in various stages of proposal:

- **Multiple memories** - More than one linear memory per module
- **Garbage collection** - Direct support for managed languages
- **Threads and atomics** - Shipped, but not in original MVP
- **SIMD** - Shipped in most engines
- **Tail calls** - Efficient recursion
- **Exception handling** - Structured exception handling
- **Component model** - High-level composition

We'll cover these in detail in later chapters.

## Why This Design?

Every design decision has a rationale:

- **Stack machine** - Compact, easy to validate, easy to compile
- **Structured control flow** - Fast validation, enables optimizations
- **Strong typing** - Memory safety, no type confusion attacks
- **Linear memory** - Simple, efficient, matches native code
- **Explicit imports/exports** - Security, capability-based design
- **Validation before execution** - Safety guarantees

WASM is designed to be fast, safe, and portable. The architecture reflects these goals at every level.

## Next Steps

We've covered the conceptual architecture. Next, we'll dive into the text format (WAT), which makes these concepts concrete. Understanding WAT is essential for debugging and for understanding how higher-level languages compile to WASM.
