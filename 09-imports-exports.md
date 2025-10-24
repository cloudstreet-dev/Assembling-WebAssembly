# Chapter 9: Imports and Exports

## Interface Design in WebAssembly

Imports and exports define a module's interface—what it needs and what it provides. Well-designed interfaces make modules:

- **Reusable** - Work in different contexts with different hosts
- **Testable** - Easy to mock imports for testing
- **Composable** - Modules can work together
- **Secure** - Minimal surface area, explicit dependencies

Unlike traditional linking (which often pulls in entire libraries), WASM imports are fine-grained and explicit.

## Exports: The Module's Public API

Exports make module internals accessible to the host:

```wasm
(module
  (func $add (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )

  (export "add" (func $add))
)
```

### Exporting Functions

Functions are the primary export:

```wasm
(func (export "initialize") ...)
(func (export "process") (param i32) (result i32) ...)
(func (export "cleanup") ...)
```

Or export separately:

```wasm
(func $init ...)
(export "initialize" (func $init))
```

**Multiple names for same function:**

```wasm
(func $add ...)
(export "add" (func $add))
(export "plus" (func $add))
(export "_add" (func $add))  ;; For different naming conventions
```

### Exporting Memory

Share linear memory with the host:

```wasm
(memory 1)
(export "memory" (memory 0))
```

This enables:
- **Zero-copy I/O** - Host writes directly to WASM memory
- **String passing** - Write strings to memory, pass pointers
- **Large data transfer** - Avoid slow parameter passing
- **Shared buffers** - Multiple modules access same memory

Example in JavaScript:

```javascript
const instance = await WebAssembly.instantiate(module);
const memory = instance.exports.memory;
const buffer = memory.buffer;  // ArrayBuffer
const view = new Uint8Array(buffer);

// Write data to WASM memory
const data = [1, 2, 3, 4, 5];
view.set(data, 0);

// Call WASM function that reads from memory
instance.exports.process_buffer(0, data.length);
```

### Exporting Tables

Share tables for dynamic linking:

```wasm
(table 100 funcref)
(export "table" (table 0))
```

Multiple modules can register functions in a shared table.

### Exporting Globals

Export constants or mutable state:

```wasm
(global $version i32 (i32.const 100))
(export "version" (global $version))

(global $counter (mut i32) (i32.const 0))
(export "counter" (global $counter))
```

Immutable globals are constants. Mutable globals provide shared state:

```javascript
const instance = await WebAssembly.instantiate(module);
console.log(instance.exports.version.value);  // 100

const counter = instance.exports.counter;
console.log(counter.value);  // 0
counter.value = 10;           // Update from JavaScript
console.log(counter.value);  // 10
```

## Imports: Module Dependencies

Imports declare what a module needs from the host:

```wasm
(module
  (import "env" "log" (func $log (param i32)))
  (import "env" "memory" (memory 1))
  (import "env" "table" (table 10 funcref))
  (import "env" "max_size" (global $max_size i32))

  (func (export "example")
    i32.const 42
    call $log
  )
)
```

### Importing Functions

Functions bring host capabilities into WASM:

```wasm
(import "console" "log" (func $log (param i32)))
(import "math" "random" (func $random (result f64)))
(import "file" "read" (func $read (param i32 i32 i32) (result i32)))
```

The host provides implementations:

```javascript
const importObject = {
  console: {
    log: (value) => console.log(value)
  },
  math: {
    random: () => Math.random()
  },
  file: {
    read: (fd, buf, len) => {
      // Read from file descriptor into buffer
      return bytesRead;
    }
  }
};
```

### Function Signature Matching

Import signatures must match exactly:

```wasm
(import "env" "func" (func (param i32 f64) (result i32)))
```

```javascript
// ✓ Correct: two parameters, returns number
env.func = (a, b) => Math.floor(a + b);

// ✗ Wrong: returns nothing
env.func = (a, b) => { console.log(a, b); };

// ✗ Wrong: wrong number of parameters
env.func = (a) => a;
```

JavaScript functions receive WASM values as JavaScript numbers. The WASM runtime handles conversion.

### Importing Memory

Import shared memory from host:

```wasm
(import "env" "memory" (memory 1))
```

```javascript
const memory = new WebAssembly.Memory({ initial: 1 });
const importObject = { env: { memory } };
```

This is common when:
- Multiple modules share memory
- Host pre-populates memory with data
- Host needs to inspect memory after WASM runs

### Importing Tables

Import tables for dynamic linking:

```wasm
(import "env" "table" (table 100 funcref))
```

```javascript
const table = new WebAssembly.Table({ initial: 100, element: 'anyfunc' });
const importObject = { env: { table } };
```

### Importing Globals

Import constants or mutable globals:

```wasm
(import "env" "heap_base" (global $heap_base i32))
(import "env" "debug" (global $debug (mut i32)))
```

```javascript
const importObject = {
  env: {
    heap_base: 65536,  // Plain number for immutable global
    debug: new WebAssembly.Global({ value: 'i32', mutable: true }, 0)
  }
};
```

## Namespacing Imports

Import names have two parts: **module** and **field**:

```wasm
(import "module_name" "field_name" ...)
```

This creates a two-level namespace. Common conventions:

- `env.*` - Environment functions (standard in Emscripten, WASI)
- `wasi_snapshot_preview1.*` - WASI system calls
- `console.*` - Console/logging functions
- `js.*` - JavaScript interop functions
- `app.*` - Application-specific functions

Example:

```wasm
(import "env" "memory" (memory 1))
(import "console" "log" (func $log (param i32)))
(import "console" "error" (func $error (param i32)))
(import "wasi_snapshot_preview1" "fd_write" (func $fd_write ...))
```

```javascript
const importObject = {
  env: {
    memory: new WebAssembly.Memory({ initial: 1 })
  },
  console: {
    log: (x) => console.log(x),
    error: (x) => console.error(x)
  },
  wasi_snapshot_preview1: {
    fd_write: (...args) => wasiImplementation.fd_write(...args)
  }
};
```

## Interface Patterns

### Callback Pattern

Pass callbacks from host to WASM:

```wasm
(import "env" "callback" (func $callback (param i32)))

(func (export "process") (param $data i32)
  local.get $data
  call $callback  ;; Call host-provided callback
)
```

```javascript
const importObject = {
  env: {
    callback: (value) => console.log('Callback:', value)
  }
};
```

### Memory-Based Interface

Pass complex data via shared memory:

```wasm
(import "env" "memory" (memory 1))

;; Write result to memory[0..3]
(func (export "compute") (param $x i32) (param $y i32)
  i32.const 0     ;; Address
  local.get $x
  local.get $y
  i32.add
  i32.store       ;; Store result
)
```

```javascript
const memory = new WebAssembly.Memory({ initial: 1 });
const importObject = { env: { memory } };
const instance = await WebAssembly.instantiate(module, importObject);

instance.exports.compute(10, 32);

const view = new Int32Array(memory.buffer);
console.log('Result:', view[0]);  // 42
```

### String Passing

Strings as pointers to UTF-8 in memory:

```wasm
(import "env" "memory" (memory 1))
(import "env" "print_string" (func $print (param i32 i32)))  ;; ptr, len

(data (i32.const 0) "Hello, WASM!")

(func (export "greet")
  i32.const 0   ;; String pointer
  i32.const 12  ;; String length
  call $print
)
```

```javascript
const memory = new WebAssembly.Memory({ initial: 1 });

const importObject = {
  env: {
    memory,
    print_string: (ptr, len) => {
      const bytes = new Uint8Array(memory.buffer, ptr, len);
      const str = new TextDecoder().decode(bytes);
      console.log(str);
    }
  }
};
```

### Error Handling Pattern

Return error codes or use imported error handler:

```wasm
(import "env" "on_error" (func $on_error (param i32)))

(func (export "divide") (param $a i32) (param $b i32) (result i32)
  local.get $b
  i32.eqz
  (if
    (then
      i32.const 1  ;; Error code: division by zero
      call $on_error
      i32.const 0
      return
    )
  )

  local.get $a
  local.get $b
  i32.div_s
)
```

```javascript
const importObject = {
  env: {
    on_error: (code) => {
      throw new Error(`WASM error code: ${code}`);
    }
  }
};
```

## Multi-Module Composition

### Shared Memory Between Modules

Module A exports memory, Module B imports it:

```wasm
;; Module A
(module
  (memory 1)
  (export "memory" (memory 0))

  (func (export "write") (param $addr i32) (param $value i32)
    local.get $addr
    local.get $value
    i32.store
  )
)
```

```wasm
;; Module B
(module
  (import "env" "memory" (memory 1))

  (func (export "read") (param $addr i32) (result i32)
    local.get $addr
    i32.load
  )
)
```

```javascript
const instanceA = await WebAssembly.instantiate(moduleA);

const importObjectB = {
  env: {
    memory: instanceA.exports.memory  // Share A's memory
  }
};
const instanceB = await WebAssembly.instantiate(moduleB, importObjectB);

instanceA.exports.write(0, 42);
console.log(instanceB.exports.read(0));  // 42
```

### Shared Tables for Dynamic Linking

```wasm
;; Module A: provides functions
(module
  (import "env" "table" (table 10 funcref))

  (func $f1 (export "f1") (result i32) i32.const 1)
  (func $f2 (export "f2") (result i32) i32.const 2)

  (func (export "register")
    ;; Register functions in shared table
    i32.const 0
    ref.func $f1
    table.set

    i32.const 1
    ref.func $f2
    table.set
  )
)
```

```wasm
;; Module B: calls via table
(module
  (import "env" "table" (table 10 funcref))

  (type $thunk (func (result i32)))

  (func (export "call_dynamic") (param $index i32) (result i32)
    local.get $index
    call_indirect (type $thunk)
  )
)
```

```javascript
const table = new WebAssembly.Table({ initial: 10, element: 'anyfunc' });

const importObject = { env: { table } };

const instanceA = await WebAssembly.instantiate(moduleA, importObject);
const instanceB = await WebAssembly.instantiate(moduleB, importObject);

instanceA.exports.register();  // Register A's functions
console.log(instanceB.exports.call_dynamic(0));  // 1 (calls f1)
console.log(instanceB.exports.call_dynamic(1));  // 2 (calls f2)
```

## Import Polymorphism

Different hosts can provide different implementations:

```wasm
(import "env" "log" (func $log (param i32)))
```

**Browser:**

```javascript
const importObject = {
  env: {
    log: (value) => console.log(value)
  }
};
```

**Node.js:**

```javascript
const importObject = {
  env: {
    log: (value) => process.stdout.write(value.toString())
  }
};
```

**Testing:**

```javascript
const logs = [];
const importObject = {
  env: {
    log: (value) => logs.push(value)  // Capture for assertions
  }
};
```

This makes modules portable and testable.

## Performance Considerations

### Import Call Overhead

Calling imported functions has some overhead:

- Transition from WASM to host
- Parameter marshalling
- Context switching

For hot paths, minimize import calls:

```wasm
;; Bad: call import in tight loop
(loop $continue
  local.get $value
  call $imported_log
  br $continue
)

;; Better: batch calls
(loop $continue
  ;; Accumulate data
  br $continue
)
call $imported_log  ;; Log once after loop
```

### Memory Export Overhead

Exporting memory has minimal overhead. The host accesses memory directly via an ArrayBuffer.

### Table Overhead

Indirect calls via tables are slightly slower than direct calls, but much faster than calling through imports.

## Versioning and Evolution

Design for evolution:

```wasm
;; Version 1
(export "api_v1_add" (func $add))

;; Version 2 (add new, keep old)
(export "api_v1_add" (func $add))
(export "api_v2_add_checked" (func $add_checked))
```

Or use version globals:

```wasm
(global (export "api_version") i32 (i32.const 2))
```

Hosts can check versions and adapt.

## Best Practices

1. **Minimize imports** - Fewer dependencies = more portable
2. **Use descriptive names** - `env.log` is clearer than `e.l`
3. **Document signatures** - Comment parameter meanings
4. **Group related imports** - Use namespaces logically
5. **Export memory when needed** - Enables efficient data transfer
6. **Version your API** - Plan for evolution
7. **Validate imported functions** - Check behavior in tests
8. **Keep interfaces simple** - Avoid complex calling conventions

## Debugging Import/Export Issues

**Check module metadata:**

```javascript
console.log(WebAssembly.Module.imports(module));
console.log(WebAssembly.Module.exports(module));
```

**Use browser DevTools:**
- Set breakpoints in imported functions
- Inspect parameters and return values
- Watch memory/table contents

**Validate manually:**

```bash
wasm-objdump -x module.wasm | grep -A5 "Import\|Export"
```

## Next Steps

We've covered how modules interface with their environment. Next, we'll explore tables in depth—how they enable indirect function calls, dynamic linking, and polymorphism in WebAssembly.
