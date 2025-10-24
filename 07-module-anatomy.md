# Chapter 7: Module Anatomy

## What is a Module?

A WebAssembly module is the unit of deployment, loading, and compilation. It's a self-contained package that:

- Declares what it needs (imports)
- Declares what it provides (exports)
- Contains code, data, and type definitions
- Can be instantiated multiple times
- Is validated before execution

Think of a module as analogous to a shared library (.so, .dll) or a compiled executable, but portable and sandboxed.

## Module Structure Overview

A complete module contains these components:

```wasm
(module
  ;; Type definitions
  (type $binary_op (func (param i32 i32) (result i32)))

  ;; Imports
  (import "env" "log" (func $log (param i32)))
  (import "js" "memory" (memory 1))

  ;; Function declarations
  (func $add (type $binary_op) ...)
  (func $mul (type $binary_op) ...)

  ;; Tables
  (table 10 funcref)

  ;; Memory
  (memory 1 10)

  ;; Globals
  (global $counter (mut i32) (i32.const 0))

  ;; Exports
  (export "add" (func $add))
  (export "memory" (memory 0))

  ;; Start function
  (start $init)

  ;; Element segments (table initialization)
  (elem (i32.const 0) $add $mul)

  ;; Data segments (memory initialization)
  (data (i32.const 0) "Hello")
)
```

Let's dissect each component.

## Types Section

Defines function signatures used throughout the module:

```wasm
(type $void_func (func))
(type $predicate (func (param i32) (result i32)))
(type $comparator (func (param i32 i32) (result i32)))
```

Types are referenced by index or identifier. Defining types once and reusing them saves space in the binary encoding.

Example using types:

```wasm
(module
  (type $binary_op (func (param i32 i32) (result i32)))

  (func $add (type $binary_op)
    local.get 0
    local.get 1
    i32.add
  )

  (func $sub (type $binary_op)
    local.get 0
    local.get 1
    i32.sub
  )
)
```

Both `$add` and `$sub` share the same type definition.

## Imports Section

Imports bring functionality from the host environment:

### Importing Functions

```wasm
(import "env" "print" (func $print (param i32)))
(import "Math" "random" (func $random (result f64)))
```

The import has:
- **Module name** - "env", "Math" (namespace)
- **Field name** - "print", "random" (specific import)
- **Type** - Function signature

The host must provide these at instantiation.

### Importing Memory

```wasm
(import "js" "memory" (memory 1 100))
```

Import memory from the host. This enables shared memory between host and WASM—useful for passing large data without copying.

### Importing Tables

```wasm
(import "env" "table" (table 10 funcref))
```

Import a table from the host. Multiple modules can share a table for dynamic linking.

### Importing Globals

```wasm
(import "env" "PI" (global $pi f64))
```

Import a constant or variable from the host.

Mutable globals can also be imported:

```wasm
(import "env" "counter" (global $counter (mut i32)))
```

### Import Ordering

Imports define an index space:

```wasm
(import "a" "f1" (func))   ;; Function index 0
(import "a" "f2" (func))   ;; Function index 1
(func $local ...)          ;; Function index 2
```

Imported items come first in their respective index spaces (functions, tables, memories, globals).

## Functions Section

Functions are the executable code:

```wasm
(func $add (param $a i32) (param $b i32) (result i32)
  local.get $a
  local.get $b
  i32.add
)
```

Functions can be:
- **Private** - Not exported, only callable within the module
- **Public** - Exported for external use

### Function Index Space

Functions have indices starting from 0:

```wasm
(import "env" "log" (func))   ;; Index 0 (imported)
(func $add ...)               ;; Index 1
(func $mul ...)               ;; Index 2
```

You can call functions by index:

```wasm
call 2  ;; Call $mul
```

Or by identifier:

```wasm
call $mul
```

### Local Variables

Functions can declare local variables beyond parameters:

```wasm
(func $example (param $n i32) (result i32)
  (local $i i32)
  (local $sum i32)
  (local $temp f64)

  ;; Function body uses locals
  ...
)
```

Locals are zero-initialized at function entry.

Multiple locals of the same type can be declared together:

```wasm
(local $x i32)
(local $y i32)
(local $z i32)
```

Or:

```wasm
(local i32 i32 i32)  ;; Three i32 locals
```

## Tables Section

Tables hold references—either function references or external references:

```wasm
(table $dispatch 100 funcref)
(table $objects 50 externref)
```

Tables enable:
- **Indirect function calls** - Call functions dynamically
- **Dynamic linking** - Add functions at runtime
- **Host object references** - Store JavaScript objects, etc.

### Table Initialization

Tables can be initialized via element segments:

```wasm
(module
  (table 10 funcref)

  (func $f0 ...)
  (func $f1 ...)
  (func $f2 ...)

  ;; Initialize table[0..2] with $f0, $f1, $f2
  (elem (i32.const 0) $f0 $f1 $f2)
)
```

Active element segments run at instantiation. Passive segments can be applied later:

```wasm
(elem $funcs funcref (ref.func $f0) (ref.func $f1))

(func $init
  i32.const 5         ;; Destination index in table
  i32.const 0         ;; Source offset in segment
  i32.const 2         ;; Number of elements
  table.init $funcs   ;; Copy to table
  elem.drop $funcs    ;; Drop segment
)
```

### Indirect Calls

Call functions through the table:

```wasm
(module
  (type $callback (func (param i32) (result i32)))
  (table 10 funcref)

  (func $double (param i32) (result i32)
    local.get 0
    i32.const 2
    i32.mul
  )

  (elem (i32.const 0) $double)

  (func $call_indirect_example (param $index i32) (param $value i32) (result i32)
    local.get $value
    local.get $index
    call_indirect (type $callback)  ;; Call table[$index]($value)
  )
)
```

The type annotation ensures the function in the table has the expected signature.

## Memory Section

Declare linear memory:

```wasm
(memory 1)        ;; 1 page minimum
(memory 2 100)    ;; 2 pages minimum, 100 pages maximum
```

Modules can have at most one memory currently (multi-memory is in proposal).

### Memory Initialization

Data segments initialize memory:

```wasm
(module
  (memory 1)

  ;; Write string at offset 0
  (data (i32.const 0) "Hello, WASM!")

  ;; Write bytes at offset 100
  (data (i32.const 100) "\01\02\03\04")
)
```

Data is written before the start function runs and before any exported functions are callable.

## Globals Section

Globals are module-level variables:

```wasm
(global $const i32 (i32.const 42))               ;; Immutable
(global $var (mut i32) (i32.const 0))            ;; Mutable
(global $pi f64 (f64.const 3.141592653589793))  ;; Immutable
```

Immutable globals are constant expressions evaluated at instantiation:

```wasm
(import "env" "base" (global $base i32))
(global $offset i32 (global.get $base))  ;; Computed from imported global
```

Mutable globals can be imported, exported, and modified:

```wasm
(global $counter (mut i32) (i32.const 0))

(func $increment
  global.get $counter
  i32.const 1
  i32.add
  global.set $counter
)
```

## Exports Section

Exports make module contents visible to the host:

```wasm
(export "add" (func $add))
(export "memory" (memory 0))
(export "table" (table 0))
(export "counter" (global $counter))
```

Export names are arbitrary strings (UTF-8). Multiple exports can refer to the same item:

```wasm
(func $add ...)
(export "add" (func $add))
(export "plus" (func $add))  ;; Same function, different name
```

### Export as Module Interface

Exports define your module's public API:

```wasm
(module
  ;; Private helper
  (func $helper ...)

  ;; Public API
  (func $init (export "init") ...)
  (func $process (export "process") ...)
  (func $cleanup (export "cleanup") ...)

  ;; Share memory
  (memory 1)
  (export "memory" (memory 0))
)
```

Only exported items are accessible from outside.

## Start Section

Optionally, specify a start function that runs automatically at instantiation:

```wasm
(module
  (func $init
    ;; Initialization code
    ;; Set up data structures, etc.
  )

  (start $init)
)
```

The start function:
- Takes no parameters
- Returns no results
- Runs once, before any exports are callable
- Runs after data and element segments are initialized

Use it for:
- Initializing global state
- Setting up memory structures
- Performing one-time setup

## Element Section

Element segments initialize tables with function references or external references:

```wasm
(elem (i32.const 0) $f0 $f1 $f2)  ;; Active: runs at instantiation

(elem $passive funcref (ref.func $f3) (ref.func $f4))  ;; Passive: apply later
```

Active segments are like data segments for tables. Passive segments are applied via `table.init`.

## Data Section

Data segments initialize linear memory:

```wasm
(data (i32.const 0) "static string")  ;; Active

(data $lazy "lazy data")  ;; Passive
```

Active segments copy data to memory at instantiation. Passive segments are copied via `memory.init`.

### Offset Expressions

Data and element segments use constant expressions for offsets:

```wasm
(global $data_start i32 (i32.const 1024))
(data (global.get $data_start) "data")  ;; Offset from global
```

This allows computing offsets from imported globals.

## Complete Example: Calculator Module

```wasm
(module
  ;; Type for binary operations
  (type $binop (func (param i32 i32) (result i32)))

  ;; Import logging function
  (import "console" "log" (func $log (param i32)))

  ;; Memory for storing results
  (memory 1)
  (export "memory" (memory 0))

  ;; Global for result count
  (global $result_count (mut i32) (i32.const 0))

  ;; Addition
  (func $add (type $binop)
    local.get 0
    local.get 1
    i32.add
  )

  ;; Subtraction
  (func $sub (type $binop)
    local.get 0
    local.get 1
    i32.sub
  )

  ;; Multiplication
  (func $mul (type $binop)
    local.get 0
    local.get 1
    i32.mul
  )

  ;; Division
  (func $div (type $binop)
    local.get 0
    local.get 1
    i32.div_s
  )

  ;; Table for dynamic dispatch
  (table 4 funcref)
  (elem (i32.const 0) $add $sub $mul $div)

  ;; Execute operation by index
  (func $execute (param $op i32) (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    local.get $op
    call_indirect (type $binop)

    ;; Increment result count
    global.get $result_count
    i32.const 1
    i32.add
    global.set $result_count
  )

  ;; Initialization
  (func $init
    ;; Log initialization
    i32.const 42
    call $log
  )
  (start $init)

  ;; Exports
  (export "execute" (func $execute))
  (export "add" (func $add))
  (export "sub" (func $sub))
  (export "mul" (func $mul))
  (export "div" (func $div))
  (export "result_count" (global $result_count))
)
```

This module demonstrates:
- Type definitions
- Imports (logging function)
- Multiple functions
- Tables for dynamic dispatch
- Memory export
- Globals (result counter)
- Start function
- Multiple exports

## Module Validation

Before a module can execute, it must be validated:

1. **Type checking** - All operations have correct types
2. **Function signatures** - Calls match declarations
3. **Memory/table bounds** - Indices are valid
4. **Control flow** - Well-structured, stack-balanced
5. **Constants** - Constant expressions are valid
6. **Imports** - Declared with valid types

Invalid modules are rejected. Valid modules are safe to execute.

## Module Instantiation

Creating a module instance:

1. **Parse** - Decode binary format
2. **Validate** - Check all constraints
3. **Allocate** - Create memory, tables, globals
4. **Initialize** - Run data and element segments
5. **Run start function** - Execute initialization code
6. **Return instance** - Exports are now callable

Each instantiation creates a fresh instance with its own state.

## Multiple Instantiation

A module can be instantiated multiple times:

```javascript
const module = await WebAssembly.compile(bytes);
const instance1 = await WebAssembly.instantiate(module, imports);
const instance2 = await WebAssembly.instantiate(module, imports);
```

Each instance has separate memory, tables, and globals. They share the compiled code but not state.

## Module Linking

Modules can be linked by:

- Importing exports from other instances
- Sharing memory between instances
- Sharing tables for dynamic linking

Example:

```javascript
// Module A exports memory
const instanceA = await WebAssembly.instantiate(moduleA);

// Module B imports A's memory
const instanceB = await WebAssembly.instantiate(moduleB, {
  env: {
    memory: instanceA.exports.memory
  }
});
```

This enables modular composition.

## Module Composition Patterns

### Shared Memory Pattern

Multiple modules access shared memory:

```wasm
;; Module A
(module
  (memory 1)
  (export "memory" (memory 0))
  (func (export "write") (param i32 i32) ...)
)

;; Module B
(module
  (import "env" "memory" (memory 1))
  (func (export "read") (param i32) (result i32) ...)
)
```

### Dynamic Linking Pattern

Modules share a function table:

```wasm
;; Module A provides functions
(module
  (import "env" "table" (table 10 funcref))
  (func $f1 (export "f1") ...)
  (func $f2 (export "f2") ...)
  ;; Register functions in shared table
)

;; Module B calls via table
(module
  (import "env" "table" (table 10 funcref))
  (func $call_dynamic (param $index i32)
    local.get $index
    call_indirect ...
  )
)
```

### Service Pattern

One module provides services to others:

```wasm
;; Service module
(module
  (func (export "malloc") (param i32) (result i32) ...)
  (func (export "free") (param i32) ...)
  (global (export "heap_base") i32 (i32.const 4096))
)

;; Client module
(module
  (import "allocator" "malloc" (func $malloc (param i32) (result i32)))
  (import "allocator" "free" (func $free (param i32)))
  ;; Use allocation services
)
```

## Next Steps

We've completed the foundations of WebAssembly—architecture, formats, types, memory, and modules. In Part 2, we'll explore runtime concerns: how modules are instantiated, how imports and exports work in practice, and how the security model protects against malicious code.
