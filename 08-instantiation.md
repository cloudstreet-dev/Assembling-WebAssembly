# Chapter 8: Instantiation

## What is Instantiation?

Instantiation is the process of transforming a compiled WebAssembly module into a running instance with its own state. While a module is static (immutable bytecode), an instance is dynamic (mutable state).

Think of it like object-oriented programming:
- **Module** = Class definition (code, types, structure)
- **Instance** = Object (state, memory, globals)

One module can create many instances, each with independent state.

## The Instantiation Pipeline

Instantiation follows a well-defined sequence:

### 1. Module Compilation

Before instantiation, the module must be compiled:

```javascript
// Browser or Node.js
const response = await fetch('module.wasm');
const bytes = await response.arrayBuffer();
const module = await WebAssembly.compile(bytes);
```

Compilation transforms binary WASM into optimized machine code. This step:
- Parses the binary format
- Validates types, control flow, and constraints
- Generates native machine code (JIT compilation)
- Does not allocate memory or run any code

Compilation is separate from instantiation, enabling:
- Compile once, instantiate many times
- Parallel compilation during download (streaming)
- Caching compiled modules

### 2. Import Resolution

The module declares what it imports. The host must provide matching imports:

```wasm
(module
  (import "env" "memory" (memory 1))
  (import "env" "log" (func $log (param i32)))
  (import "env" "table" (table 10 funcref))
)
```

The host supplies these via an import object:

```javascript
const importObject = {
  env: {
    memory: new WebAssembly.Memory({ initial: 1 }),
    log: (value) => console.log(value),
    table: new WebAssembly.Table({ initial: 10, element: 'anyfunc' })
  }
};

const instance = await WebAssembly.instantiate(module, importObject);
```

Import resolution:
- Checks that each import is provided
- Validates types match (function signatures, memory/table limits)
- Links imported items into the instance

If imports are missing or types don't match, instantiation fails.

### 3. Allocation

The runtime allocates resources for the instance:

**Memory** - If the module declares memory (not imported), allocate linear memory:

```wasm
(memory 2)  ;; Allocate 2 pages (128KB)
```

This creates an ArrayBuffer (in browsers) or equivalent backing store.

**Tables** - Allocate tables for function references:

```wasm
(table 100 funcref)  ;; Allocate 100 slots
```

**Globals** - Allocate and initialize global variables:

```wasm
(global $counter (mut i32) (i32.const 0))
```

At this point, memory is zeroed, tables are null, globals have default values.

### 4. Data and Element Initialization

**Data segments** copy bytes into linear memory:

```wasm
(data (i32.const 0) "Hello, WASM!")
```

This writes the string to memory at offset 0.

**Element segments** copy function references into tables:

```wasm
(elem (i32.const 0) $func0 $func1 $func2)
```

This writes function references to table slots 0, 1, 2.

Initialization runs in declaration order. If an offset is out of bounds, instantiation traps.

### 5. Start Function Execution

If a start function is declared, it runs now:

```wasm
(start $init)
```

The start function:
- Takes no arguments
- Returns no value
- Runs once, before exports are accessible
- Can initialize data structures, perform setup, etc.

If the start function traps, instantiation fails.

### 6. Export Access

After successful initialization, the instance's exports are available:

```javascript
const instance = await WebAssembly.instantiate(module, importObject);

// Access exports
instance.exports.add(2, 3);  // Call exported function
instance.exports.memory;      // Access exported memory
```

## Instantiation APIs

### JavaScript: WebAssembly.instantiate()

**From compiled module:**

```javascript
const module = await WebAssembly.compile(bytes);
const instance = await WebAssembly.instantiate(module, importObject);
```

**From bytes (compile + instantiate):**

```javascript
const { instance, module } = await WebAssembly.instantiate(bytes, importObject);
```

The second form is shorthand for compiling then instantiating.

### JavaScript: WebAssembly.Instance()

Synchronous instantiation (module must already be compiled):

```javascript
const module = await WebAssembly.compile(bytes);
const instance = new WebAssembly.Instance(module, importObject);
```

Use this when you need synchronous control flow.

### Streaming Instantiation

Browsers support streaming compilation and instantiation:

```javascript
const { instance, module } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm'),
  importObject
);
```

This begins compilation as bytes arrive over the network. By the time download completes, compilation is often done, reducing time-to-execution significantly.

## Import Object Structure

The import object is a nested JavaScript object matching the module's imports:

```wasm
(import "math" "add" (func (param i32 i32) (result i32)))
(import "math" "pi" (global f64))
(import "env" "memory" (memory 1))
```

```javascript
const importObject = {
  math: {
    add: (a, b) => a + b,
    pi: 3.141592653589793
  },
  env: {
    memory: new WebAssembly.Memory({ initial: 1 })
  }
};
```

### Importing Functions

JavaScript functions become WASM functions:

```javascript
const importObject = {
  console: {
    log: (value) => console.log(value)
  }
};
```

The function signature must match:

```wasm
(import "console" "log" (func $log (param i32)))
```

Type mismatches cause instantiation to fail.

### Importing Memory

Create and share memory:

```javascript
const memory = new WebAssembly.Memory({ initial: 10, maximum: 100 });

const importObject = {
  env: { memory }
};

const instance = await WebAssembly.instantiate(module, importObject);

// Access shared memory
const buffer = memory.buffer;  // ArrayBuffer
const view = new Uint8Array(buffer);
view[0] = 42;  // Write to WASM memory from JavaScript
```

This enables zero-copy data sharing.

### Importing Tables

Share tables for dynamic linking:

```javascript
const table = new WebAssembly.Table({ initial: 10, element: 'anyfunc' });

const importObject = {
  env: { table }
};
```

Multiple instances can share the same table.

### Importing Globals

Import constants or mutable globals:

```javascript
const importObject = {
  env: {
    maxSize: 1000,  // Immutable global (plain number)
    counter: new WebAssembly.Global({ value: 'i32', mutable: true }, 0)
  }
};
```

Mutable globals require `WebAssembly.Global` objects. Immutable globals can be plain numbers.

## Multiple Instances

You can instantiate the same module multiple times:

```javascript
const module = await WebAssembly.compile(bytes);

const instance1 = await WebAssembly.instantiate(module, importObject);
const instance2 = await WebAssembly.instantiate(module, importObject);
```

Each instance has its own:
- Memory (unless shared via imports)
- Tables (unless shared)
- Globals
- Function state (locals, etc.)

They share:
- Compiled code
- Type definitions
- Static structure

This is efficient—code is compiled once, but state is per-instance.

## Instance Lifetime

Instances are garbage collected when no longer referenced:

```javascript
let instance = await WebAssembly.instantiate(module, importObject);
// Use instance...
instance = null;  // Instance becomes eligible for GC
```

The runtime reclaims:
- Memory buffers
- Table storage
- Global storage
- Compiled code (if no other instances exist)

## Error Handling

Instantiation can fail at several points:

### Missing Imports

```javascript
// Module imports "env.log" but it's not provided
const importObject = { env: {} };  // Missing 'log'
await WebAssembly.instantiate(module, importObject);  // Throws LinkError
```

### Type Mismatch

```wasm
(import "env" "func" (func (param i32)))
```

```javascript
const importObject = {
  env: {
    func: (a, b) => a + b  // Wrong signature: expects 2 params
  }
};
await WebAssembly.instantiate(module, importObject);  // Throws LinkError
```

### Out-of-Bounds Initialization

```wasm
(memory 1)  ;; 65536 bytes
(data (i32.const 70000) "data")  ;; Offset beyond memory
```

Instantiation traps at the data initialization step.

### Start Function Trap

```wasm
(func $init
  unreachable  ;; Trap!
)
(start $init)
```

Instantiation fails when the start function traps.

### Handling Errors

```javascript
try {
  const instance = await WebAssembly.instantiate(module, importObject);
} catch (error) {
  if (error instanceof WebAssembly.LinkError) {
    console.error('Import mismatch:', error.message);
  } else if (error instanceof WebAssembly.RuntimeError) {
    console.error('Instantiation trapped:', error.message);
  } else {
    console.error('Unknown error:', error);
  }
}
```

## Deterministic Instantiation

Instantiation is deterministic if:
- Imports are deterministic
- Data and element segments have constant offsets
- Start function is deterministic

This makes WASM suitable for blockchain and reproducible builds.

Non-determinism can come from:
- Host imports (e.g., `Math.random()`)
- Computed offsets using non-deterministic globals
- Floating-point operations (NaN bit patterns)

## Instantiation in Non-Browser Environments

### Node.js

```javascript
import fs from 'fs';
const bytes = fs.readFileSync('module.wasm');
const { instance } = await WebAssembly.instantiate(bytes, importObject);
```

### Deno

```javascript
const bytes = await Deno.readFile('module.wasm');
const { instance } = await WebAssembly.instantiate(bytes, importObject);
```

### wasmtime (Rust)

```rust
use wasmtime::*;

let engine = Engine::default();
let module = Module::from_file(&engine, "module.wasm")?;
let mut store = Store::new(&engine, ());
let instance = Instance::new(&mut store, &module, &[])?;
```

### wasmer (Rust/C/Python/etc.)

```rust
use wasmer::{Store, Module, Instance};

let store = Store::default();
let module = Module::from_file(&store, "module.wasm")?;
let instance = Instance::new(&module, &imports)?;
```

Each runtime has its own API, but the semantics are standardized.

## Pre-Instantiation

Some runtimes support pre-instantiation or snapshotting:

1. Instantiate a module
2. Run initialization code
3. Snapshot the state
4. Restore from snapshot for fast subsequent starts

This is useful for serverless functions or CLI tools that start frequently.

## Lazy Initialization

You can defer initialization by using passive data and element segments:

```wasm
(data $lazy "expensive data")

(func (export "init")
  i32.const 0
  i32.const 0
  i32.const 1000
  memory.init $lazy  ;; Initialize on demand
  data.drop $lazy
)
```

This reduces instantiation time when data isn't always needed.

## Debugging Instantiation Issues

Tools and techniques:

**Check imports carefully:**
```javascript
console.log('Module imports:', WebAssembly.Module.imports(module));
console.log('Module exports:', WebAssembly.Module.exports(module));
```

**Inspect module structure:**
```bash
wasm-objdump -x module.wasm  # Show all sections
```

**Validate separately:**
```bash
wasm-validate module.wasm
```

**Use browser DevTools:**
- Set breakpoints in imported functions
- Inspect memory after instantiation
- Check console for LinkError details

## Optimizing Instantiation

Techniques to reduce instantiation time:

1. **Stream compilation** - Use `instantiateStreaming()` in browsers
2. **Cache compiled modules** - Store compiled modules in IndexedDB
3. **Minimize data segments** - Large static data slows initialization
4. **Use passive segments** - Defer copying until needed
5. **Avoid expensive start functions** - Move work to first function call
6. **Share imports** - Reuse memory, tables between instances

## Next Steps

Now that we understand instantiation, we'll explore imports and exports in depth—how to design effective module interfaces, patterns for interfacing with host environments, and techniques for building modular WASM applications.
