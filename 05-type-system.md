# Chapter 5: Type System

## Why Types Matter in WebAssembly

WASM's type system isn't an afterthought—it's central to its security model, performance, and correctness guarantees. The type system ensures:

- **Memory safety** - No type confusion attacks, no buffer overflows via typing
- **Fast validation** - Type checking in a single pass
- **JIT optimization** - Types enable aggressive optimization
- **Interface contracts** - Imports and exports are type-checked

Unlike JavaScript's dynamic typing or C's weak typing, WASM is statically and strongly typed. Every value has a known type, and type mismatches are impossible.

## Value Types

WASM has seven value types in the current specification:

### Numeric Types

**i32** - 32-bit integer
- Used for booleans (0 = false, non-zero = true)
- Used for small integers
- Used for pointers in linear memory (memory is 32-bit addressable currently)
- Signed and unsigned operations available (e.g., `i32.div_s` vs `i32.div_u`)

**i64** - 64-bit integer
- Large integers
- Used for timestamps, IDs
- Not directly usable in JavaScript (JS Number is 53-bit precise)

**f32** - 32-bit IEEE 754 floating-point
- Single-precision float
- Graphics, games, scientific computing where precision/speed tradeoff matters

**f64** - 64-bit IEEE 754 floating-point
- Double-precision float
- Corresponds to JavaScript Number
- Default for floating-point when precision matters

### Vector Type

**v128** - 128-bit SIMD vector
- Can hold 16×i8, 8×i16, 4×i32, 2×i64, 4×f32, or 2×f64
- Operations process all lanes in parallel
- Covered in detail in Chapter 44

### Reference Types

**funcref** - Reference to a function
- Opaque reference
- Can be stored in tables
- Can't be forged or manipulated as integers
- Enables indirect function calls securely

**externref** - Reference to a host object
- Opaque to WASM
- Managed by host (e.g., JavaScript object in browser)
- Can be passed around but not inspected by WASM
- Enables interfacing with host without copying

## Type Signedness

WASM integers are signless—the type system doesn't distinguish signed from unsigned. The interpretation depends on the operation:

```wasm
i32.const -1    ;; All bits set: 0xFFFFFFFF
i32.const 1
i32.div_s       ;; Signed division: -1 / 1 = -1
```

```wasm
i32.const -1    ;; 0xFFFFFFFF interpreted as 4294967295
i32.const 1
i32.div_u       ;; Unsigned division: 4294967295 / 1 = 4294967295
```

This is different from languages like C, where `int` and `unsigned int` are distinct types. WASM shifts the signed/unsigned distinction to operations, not values.

Comparisons also come in signed and unsigned flavors:
- `i32.lt_s` - less than, signed
- `i32.lt_u` - less than, unsigned
- `i32.gt_s` - greater than, signed
- `i32.gt_u` - greater than, unsigned

## Function Types

Function types specify parameter and result types:

```wasm
(type $callback (func (param i32 f64) (result i32)))
```

This type describes functions that:
- Take an i32 and f64 as parameters
- Return an i32

Function types are structural—two functions with the same signature have the same type, even if defined separately.

### Multiple Results

WASM supports multiple return values (since MVP 1.1):

```wasm
(func $divmod (param $a i32) (param $b i32) (result i32 i32)
  local.get $a
  local.get $b
  i32.div_s    ;; Quotient

  local.get $a
  local.get $b
  i32.rem_s    ;; Remainder
)
;; Calling this leaves two i32 values on the stack
```

This is efficient for functions that logically return multiple values, avoiding memory allocation or struct packing.

## Table Types

Tables have a type specifying:
- Element type (funcref or externref)
- Limits (minimum and optionally maximum size)

```wasm
(table 10 funcref)          ;; Min 10 entries, no max
(table 5 20 externref)      ;; Min 5, max 20 entries
```

Tables are homogeneous—all entries have the same type.

## Memory Type

Memory has a type consisting of limits:

```wasm
(memory 1)      ;; Min 1 page (64KB), no max
(memory 2 100)  ;; Min 2 pages (128KB), max 100 pages (6.4MB)
```

Memory is untyped at the byte level—it's just an array of bytes. Types come from load/store operations:

```wasm
i32.const 0
i32.load        ;; Load 4 bytes as i32

i32.const 0
f64.load        ;; Load 8 bytes as f64
```

The same memory location can be loaded as different types. This is intentional—it allows implementing unions, type punning, and low-level data structures.

## Global Types

Globals have a type consisting of:
- Value type (i32, i64, f32, f64, v128, funcref, externref)
- Mutability (mutable or immutable)

```wasm
(global $const i32 (i32.const 100))              ;; Immutable
(global $var (mut f64) (f64.const 0.0))          ;; Mutable
```

Immutable globals are constants evaluated at instantiation. Mutable globals can be read and written.

## Type Checking and the Stack

The type checker validates that operations receive the correct types. This is done by tracking stack types during validation.

Example:

```wasm
i32.const 5     ;; Stack: [i32]
f64.const 2.5   ;; Stack: [i32, f64]
f64.add         ;; Error: f64.add expects [f64, f64], but top is [i32, f64]
```

The validation fails before execution. Invalid modules never run.

Correct version:

```wasm
f64.const 5.0   ;; Stack: [f64]
f64.const 2.5   ;; Stack: [f64, f64]
f64.add         ;; Stack: [f64]  ✓
```

## Block Types

Blocks can produce values:

```wasm
(block (result i32)
  i32.const 42
)
;; Stack: [i32]
```

The block type specifies what's left on the stack when the block exits. This must match for all paths through the block:

```wasm
(block (result i32)
  ;; All branches must leave exactly one i32 on the stack
  (if (result i32)
    (i32.const 1)
    (then (i32.const 2))
    (else (i32.const 3))
  )
)
```

Multi-value blocks are also allowed:

```wasm
(block (result i32 i64)
  i32.const 42
  i64.const 100
)
;; Stack: [i32, i64]
```

## Type Polymorphism

Some instructions are polymorphic:

**unreachable** - Has type `[] -> [t*]` where `t*` is any type sequence
- Used to mark code that should never execute
- After unreachable, any instructions are valid (they're never reached)

**drop** - Has type `[t] -> []` where `t` is any single type
- Discards the top stack value, regardless of its type

**select** - Has type `[t, t, i32] -> [t]` where `t` is any non-reference type
- Ternary operator: picks first or second value based on third

These polymorphic types make the type system more flexible without sacrificing safety.

## Reference Type Restrictions

Reference types (funcref, externref) have restrictions:

- Can't be stored in linear memory (only in tables for funcref, or as locals/globals)
- Can't be manipulated as integers
- Can't be forged

This maintains security. If funcref could be stored in memory, you could forge function pointers by writing arbitrary values. By restricting them to tables, WASM ensures all function references are valid.

## Type Conversions

WASM provides explicit conversion instructions:

### Integer to Integer

```wasm
i32.wrap_i64         ;; i64 -> i32 (truncate high bits)
i64.extend_i32_s     ;; i32 -> i64 (sign-extend)
i64.extend_i32_u     ;; i32 -> i64 (zero-extend)
```

### Float to Float

```wasm
f32.demote_f64       ;; f64 -> f32 (round to nearest)
f64.promote_f32      ;; f32 -> f64 (exact)
```

### Integer to Float

```wasm
f32.convert_i32_s    ;; i32 -> f32 (signed)
f32.convert_i32_u    ;; i32 -> f32 (unsigned)
f64.convert_i64_s    ;; i64 -> f64 (signed)
f64.convert_i64_u    ;; i64 -> f64 (unsigned)
```

### Float to Integer

```wasm
i32.trunc_f32_s      ;; f32 -> i32 (truncate toward zero, signed)
i32.trunc_f32_u      ;; f32 -> i32 (truncate toward zero, unsigned)
i64.trunc_f64_s      ;; f64 -> i64 (signed)
i64.trunc_f64_u      ;; f64 -> i64 (unsigned)
```

Truncation traps if the float is NaN or out of range for the target integer type.

### Saturating Conversions

Newer WASM versions add saturating conversions that don't trap:

```wasm
i32.trunc_sat_f32_s  ;; f32 -> i32, saturate on overflow
```

If the float is too large, it saturates to INT_MAX. If too small, to INT_MIN. NaN becomes 0.

### Reinterpretation

Reinterpretation casts bits without conversion:

```wasm
i32.reinterpret_f32  ;; Treat f32 bits as i32
f32.reinterpret_i32  ;; Treat i32 bits as f32
i64.reinterpret_f64  ;; Treat f64 bits as i64
f64.reinterpret_i64  ;; Treat i64 bits as f64
```

This is like C's type punning or reinterpret_cast. Useful for bit manipulation and low-level operations.

## Type Validation Algorithm

WASM's type validation is algorithmic:

1. **Initialize** with function's parameter types on the stack
2. **For each instruction**:
   - Pop required input types from stack (error if stack doesn't match)
   - Push result types onto stack
3. **At end of function**, stack must match result type

Control flow complicates this—branches can exit blocks early. The validator tracks:

- Current stack types
- Expected types for each label (block, loop, if)
- Reachability (after unreachable, validation continues but doesn't enforce stack types)

Example:

```wasm
(func (result i32)
  i32.const 1
  (if (result i32)
    (then
      i32.const 2
      return        ;; Exits function early, stack: [i32]
      i64.const 999 ;; Unreachable, any type OK
    )
    (else
      i32.const 3   ;; Stack: [i32]
    )
  )
  ;; Both branches produce i32, merge OK
  ;; Stack: [i32]
)
```

The validator ensures both branches of the `if` produce the same type.

## Benefits of WASM's Type System

### Security

Type confusion attacks are impossible. You can't treat a pointer as a function reference or vice versa. Memory safety is enforced by separating pointers (i32 indices into memory) from references (funcref/externref).

### Performance

Knowing types enables optimizations:
- Register allocation based on types
- Inlining based on function types
- SIMD operations based on vector types
- Elimination of runtime type checks

### Correctness

Type errors are caught before execution. Invalid modules are rejected at load time, not when the buggy code path is hit.

### Interface Contracts

Imports and exports are type-checked. If you export a function with signature `(param i32) (result f64)`, callers must provide an i32 and will receive an f64. No runtime surprises.

## Limitations and Extensions

Current limitations:

- **No user-defined types** - You can't define structs or enums in WASM itself
- **Limited reference types** - Only funcref and externref, not arbitrary GC'd objects
- **No parametric polymorphism** - No generics or templates at the WASM level

Future extensions (in proposal stage):

- **GC types** - Structs, arrays, and other GC'd types
- **Type imports/exports** - Share type definitions between modules
- **Subtyping** - Type hierarchies for GC'd objects

These will enable better support for high-level languages like Java, C#, and functional languages.

## Practical Implications

When compiling to WASM:

- **Language types must map to WASM types** - High-level types (strings, objects) are encoded in memory with i32 pointers
- **Type conversions must be explicit** - No implicit coercion
- **References are opaque** - Can't inspect externref contents from WASM

Understanding the type system helps you:
- Debug type errors in compiler output
- Optimize by choosing appropriate types
- Design efficient interfaces between WASM and host

## Next Steps

Now that we understand types, we'll explore the memory model—how WASM manages linear memory, how memory is allocated and accessed, and the security guarantees around memory isolation.
