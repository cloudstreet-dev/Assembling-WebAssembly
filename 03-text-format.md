# Chapter 3: Text Format (WAT)

## What is WAT?

WebAssembly Text Format (WAT) is a human-readable representation of WASM binary. While you'll rarely write WAT by hand in production, understanding it is crucial for:

- Debugging compiled output
- Understanding how languages compile to WASM
- Writing small test modules
- Reading compiler output
- Learning WASM instruction semantics

WAT uses S-expressions (like Lisp). Everything is wrapped in parentheses, which makes the structure explicit and parsing unambiguous.

## Basic Structure

A minimal WAT module:

```wasm
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add
  )
  (export "add" (func $add))
)
```

This defines a module with one function that adds two integers.

Breaking it down:

- `(module ...)` - The module container
- `(func ...)` - Function definition
- `$add` - Function identifier (a name, not a number)
- `(param $a i32)` - Parameter named `$a` of type `i32`
- `(result i32)` - Function returns an `i32`
- `local.get $a` - Push parameter `$a` onto the stack
- `i32.add` - Pop two i32s, push their sum
- `(export "add" (func $add))` - Export function as "add"

## S-Expression Syntax

WAT uses two equivalent syntaxes: folded and linear.

**Linear (stack-based):**
```wasm
local.get $a
local.get $b
i32.add
local.get $c
i32.mul
```

**Folded (nested):**
```wasm
(i32.mul
  (i32.add
    (local.get $a)
    (local.get $b))
  (local.get $c))
```

Both produce identical bytecode. Folded form mirrors the expression structure. Linear form mirrors the stack operations. Use whichever is clearer for the context.

## Types

### Value Types

- `i32` - 32-bit integer
- `i64` - 64-bit integer
- `f32` - 32-bit float
- `f64` - 64-bit float
- `v128` - 128-bit SIMD vector
- `funcref` - function reference
- `externref` - external reference

### Function Types

Define separately for reuse:

```wasm
(module
  (type $binary_op (func (param i32 i32) (result i32)))

  (func $add (type $binary_op)
    local.get 0
    local.get 1
    i32.add
  )

  (func $mul (type $binary_op)
    local.get 0
    local.get 1
    i32.mul
  )
)
```

Or inline:

```wasm
(func (param i32 i32) (result i32)
  ;; function body
)
```

## Functions

### Parameters and Results

Parameters are indexed from 0:

```wasm
(func $example (param i32) (param i64) (result f32)
  local.get 0  ;; First parameter (i32)
  local.get 1  ;; Second parameter (i64)
  ;; ... conversion and arithmetic ...
  f32.const 42.0
)
```

Named parameters are clearer:

```wasm
(func $lerp (param $a f32) (param $b f32) (param $t f32) (result f32)
  ;; result = a + (b - a) * t
  local.get $a
  local.get $b
  local.get $a
  f32.sub
  local.get $t
  f32.mul
  f32.add
)
```

### Local Variables

Functions can declare local variables:

```wasm
(func $sum_to_n (param $n i32) (result i32)
  (local $sum i32)
  (local $i i32)

  i32.const 0
  local.set $sum

  i32.const 0
  local.set $i

  (block $break
    (loop $continue
      ;; sum += i
      local.get $sum
      local.get $i
      i32.add
      local.set $sum

      ;; i++
      local.get $i
      i32.const 1
      i32.add
      local.tee $i  ;; Set $i and leave value on stack

      ;; if (i <= n) continue
      local.get $n
      i32.le_s
      br_if $continue
    )
  )

  local.get $sum
)
```

Note `local.tee`: it sets a local variable but leaves the value on the stack. It's shorthand for `local.set $x` followed by `local.get $x`.

## Control Flow

### Blocks

Blocks define label targets for branches:

```wasm
(block $outer
  ;; code
  (block $inner
    ;; code
    br $outer  ;; Break to $outer, skipping rest of both blocks
  )
  ;; skipped
)
;; execution continues here
```

Blocks can produce values:

```wasm
(block (result i32)
  i32.const 42
)
;; 42 is now on the stack
```

### Loops

Loops branch to the beginning:

```wasm
(loop $repeat
  ;; code
  local.get $condition
  br_if $repeat  ;; If condition true, jump to start of loop
)
```

The difference between `block` and `loop`: `br` to a block exits it, `br` to a loop re-enters it.

### If/Else

Conditional execution:

```wasm
local.get $x
i32.const 0
i32.gt_s
(if (result i32)
  (then
    i32.const 1   ;; x is positive
  )
  (else
    i32.const -1  ;; x is non-positive
  )
)
;; result (-1 or 1) is on stack
```

Without result type:

```wasm
local.get $x
(if
  (then
    ;; code when true
  )
  (else
    ;; code when false
  )
)
```

### Branch Tables

Switch/case via `br_table`:

```wasm
local.get $value
(br_table $case0 $case1 $case2 $default)

(block $default
  (block $case0
    (block $case1
      (block $case2
        ;; value not 0, 1, or 2 - falls through to default
      )
      ;; case 2
      ;; ...
    )
    ;; case 1
    ;; ...
  )
  ;; case 0
  ;; ...
)
;; default case
```

## Memory

Declare memory:

```wasm
(module
  (memory 1)  ;; 1 page minimum (64KB)
  (memory 1 10)  ;; 1 page min, 10 pages max
)
```

Load from memory:

```wasm
i32.const 0      ;; Address
i32.load         ;; Load 4 bytes from address 0
```

Store to memory:

```wasm
i32.const 8      ;; Address
i32.const 42     ;; Value
i32.store        ;; Store 42 at address 8
```

Loads and stores have alignment and offset:

```wasm
i32.const 100    ;; Base address
i32.load offset=4 align=4  ;; Load from address 100+4 with 4-byte alignment
```

Partial loads and stores:

```wasm
i32.const 0
i32.load8_s      ;; Load 1 byte, sign-extend to i32

i32.const 0
i32.const 255
i32.store8       ;; Store only the low byte
```

Initialize memory with data segments:

```wasm
(module
  (memory 1)
  (data (i32.const 0) "Hello, WASM!")  ;; Write string at offset 0
)
```

Active data segments run at instantiation. Passive segments can be copied later:

```wasm
(module
  (memory 1)
  (data $msg "Hello")  ;; Passive segment

  (func $init
    i32.const 0       ;; Destination offset
    i32.const 0       ;; Source offset within segment
    i32.const 5       ;; Length
    memory.init $msg  ;; Copy data
    data.drop $msg    ;; Drop segment (save memory)
  )
)
```

## Tables

Tables hold references:

```wasm
(module
  (table 10 funcref)  ;; Table of 10 function references

  (func $f1 (result i32) i32.const 1)
  (func $f2 (result i32) i32.const 2)

  (elem (i32.const 0) $f1 $f2)  ;; Initialize table[0] = $f1, table[1] = $f2

  (type $thunk (func (result i32)))

  (func $call_indirect (param $index i32) (result i32)
    local.get $index
    call_indirect (type $thunk)  ;; Call table[$index]
  )
)
```

## Globals

Globals are module-level variables:

```wasm
(module
  (global $counter (mut i32) (i32.const 0))  ;; Mutable global, initialized to 0
  (global $max i32 (i32.const 100))          ;; Immutable global

  (func $increment
    global.get $counter
    i32.const 1
    i32.add
    global.set $counter
  )

  (func $get_counter (result i32)
    global.get $counter
  )
)
```

Immutable globals are constant expressions evaluated at instantiation. Mutable globals can be read and written.

## Imports and Exports

### Imports

Modules can import functions, memory, tables, and globals:

```wasm
(module
  (import "env" "log" (func $log (param i32)))
  (import "js" "memory" (memory 1))
  (import "js" "table" (table 10 funcref))
  (import "config" "max_size" (global $max_size i32))

  (func $use_imports
    i32.const 42
    call $log  ;; Call imported function

    global.get $max_size  ;; Read imported global
    ;; ...
  )
)
```

Imports are satisfied during instantiation. The host provides the actual implementations.

### Exports

Modules export their functions, memory, tables, and globals:

```wasm
(module
  (memory 1)
  (func $add (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )

  (export "add" (func $add))
  (export "memory" (memory 0))
)
```

Exports make module contents accessible to the host.

## Start Function

Optionally, a module can specify a start function that runs automatically at instantiation:

```wasm
(module
  (func $init
    ;; Initialization code
  )

  (start $init)
)
```

Use this for setup that must run before any exported functions are called.

## Comments

Line comments with `;;`:

```wasm
i32.const 42  ;; The answer
```

Block comments with `(; ... ;)`:

```wasm
(; This is a
   multi-line comment ;)
```

## Complete Example

Here's a module that computes Fibonacci numbers:

```wasm
(module
  (func $fib (param $n i32) (result i32)
    (local $a i32)
    (local $b i32)
    (local $temp i32)
    (local $i i32)

    ;; Handle base cases
    local.get $n
    i32.const 2
    i32.lt_u
    (if (result i32)
      (then
        local.get $n  ;; fib(0) = 0, fib(1) = 1
      )
      (else
        ;; Iterative computation
        i32.const 0
        local.set $a

        i32.const 1
        local.set $b

        i32.const 2
        local.set $i

        (block $break
          (loop $continue
            ;; temp = a + b
            local.get $a
            local.get $b
            i32.add
            local.set $temp

            ;; a = b
            local.get $b
            local.set $a

            ;; b = temp
            local.get $temp
            local.set $b

            ;; i++
            local.get $i
            i32.const 1
            i32.add
            local.tee $i

            ;; if (i <= n) continue
            local.get $n
            i32.le_u
            br_if $continue
          )
        )

        local.get $b
      )
    )
  )

  (export "fib" (func $fib))
)
```

## Converting WAT to WASM

Use the WebAssembly Binary Toolkit (WABT):

```bash
wat2wasm module.wat -o module.wasm
```

And back:

```bash
wasm2wat module.wasm -o module.wat
```

These tools are essential for development and debugging.

## Folded Form Deep Dive

The folded form is often more readable for complex expressions. Compare these:

**Linear:**
```wasm
local.get $a
local.get $b
i32.add
local.get $c
local.get $d
i32.add
i32.mul
```

**Folded:**
```wasm
(i32.mul
  (i32.add (local.get $a) (local.get $b))
  (i32.add (local.get $c) (local.get $d)))
```

The folded form makes the tree structure explicit: `(a + b) * (c + d)`.

## Type Annotations

Sometimes you need to annotate types for clarity or when the type checker needs help:

```wasm
(block $outer (result i32)
  (block $inner (result i64)
    i64.const 42
    br $outer  ;; Error: wrong type
  )
)
```

The type system tracks what type each label expects.

## Practical WAT Writing

While you won't write WAT for production applications, you will:

- **Read it** when debugging compiler output
- **Write snippets** for testing specific WASM features
- **Use it as a compilation target** for DSLs or code generators
- **Understand it** to optimize higher-level language compilation

Being fluent in WAT makes you effective at debugging and understanding WASM at a deep level.

## Next Steps

Now that you understand the text format, we'll dive into the binary format. Understanding the binary encoding helps you reason about module size, parsing performance, and how WASM is transmitted and stored.
