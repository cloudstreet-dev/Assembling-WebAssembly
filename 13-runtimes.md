# Chapter 13: Runtimes

## What is a WASM Runtime?

A WebAssembly runtime is an execution environment that:
- **Loads** WASM modules (parse binary format)
- **Validates** modules (type checking, safety verification)
- **Compiles** to native code (JIT or AOT)
- **Instantiates** modules (allocate memory, initialize state)
- **Executes** WASM code
- **Interfaces** with the host (provide imports, call exports)

Runtimes exist for many environments: browsers, servers, embedded systems, edge platforms.

## Browser Runtimes

### V8 (Chrome, Edge, Node.js, Deno)

**Used in**: Chrome, Edge, Node.js, Deno, Electron

**Compilation strategy**: Liftoff (baseline JIT) + TurboFan (optimizing JIT)

**How it works**:
1. **Streaming compilation**: Begins compiling while downloading
2. **Liftoff**: Fast baseline compiler, quick startup
3. **TurboFan**: Optimizing compiler, triggered for hot code
4. **Tier-up**: Code transitions from Liftoff to TurboFan

**Strengths**:
- Extremely fast execution
- Sophisticated optimization
- Streaming compilation
- Excellent debugging support

**Features**:
- SIMD support
- Threads (SharedArrayBuffer)
- Tail calls
- Multi-value returns
- Reference types

**Example**:
```javascript
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm')
);
instance.exports.main();
```

### SpiderMonkey (Firefox)

**Used in**: Firefox

**Compilation strategy**: Baseline (fast) + Ion (optimizing)

**How it works**:
Similar to V8: baseline compiler for fast startup, optimizing compiler for hot code.

**Strengths**:
- Fast compilation
- Good performance
- Standards compliance
- Strong debugging

**Notable**: Often first to implement experimental WASM features

### JavaScriptCore (Safari)

**Used in**: Safari, WebKit-based browsers

**Compilation strategy**: BBQ (baseline) + OMG (optimizing)
- BBQ: "Baseline compiler with quick-and-dirty optimizations"
- OMG: "Optimizing Machine-code Generator"

**Strengths**:
- Low memory overhead
- Good mobile performance
- Efficient startup

**Features**: Generally implements standard features, sometimes slower to adopt experimental ones

## Standalone Runtimes

### Wasmtime

**Repository**: https://github.com/bytecodealliance/wasmtime

**Written in**: Rust

**Compilation**: Cranelift (AOT and JIT)

**Primary use cases**:
- Server-side WASM
- CLI applications
- Embedded in other applications
- WASI development

**Installation**:
```bash
curl https://wasmtime.dev/install.sh -sSf | bash
```

**Running modules**:
```bash
wasmtime module.wasm
wasmtime --dir=. module.wasm  # Grant directory access
```

**Strengths**:
- Production-ready
- Excellent WASI support
- Secure by default
- Good performance
- Embedding API (Rust, C, Python, .NET, etc.)

**Embedding example (Rust)**:
```rust
use wasmtime::*;

let engine = Engine::default();
let module = Module::from_file(&engine, "module.wasm")?;
let mut store = Store::new(&engine, ());
let instance = Instance::new(&mut store, &module, &[])?;

let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
let result = add.call(&mut store, (5, 3))?;
println!("{}", result);  // 8
```

**Configuration**:
```rust
let mut config = Config::new();
config.wasm_threads(true);
config.wasm_simd(true);
config.cache_config_load_default()?;  // Enable compilation cache
let engine = Engine::new(&config)?;
```

### Wasmer

**Repository**: https://github.com/wasmerio/wasmer

**Written in**: Rust

**Compilers**: Multiple backends
- **Singlepass**: Fast compilation, lower performance
- **Cranelift**: Balance of compilation speed and runtime performance
- **LLVM**: Slow compilation, best runtime performance

**Primary use cases**:
- Server-side execution
- Blockchain (used by NEAR, CosmWasm)
- Plugin systems
- Cross-platform binaries

**Installation**:
```bash
curl https://get.wasmer.io -sSfL | sh
```

**Running modules**:
```bash
wasmer run module.wasm
wasmer run --dir=. module.wasm
```

**Strengths**:
- Multiple compiler backends (choose your tradeoff)
- Excellent caching
- WAPM (package manager for WASM)
- Strong plugin ecosystem
- Embedding APIs (Rust, C, C++, Python, Go, PHP, Ruby, Java, C#)

**Embedding example (Python)**:
```python
from wasmer import engine, Store, Module, Instance

store = Store()
module = Module(store, open('module.wasm', 'rb').read())
instance = Instance(module)

result = instance.exports.add(5, 3)
print(result)  # 8
```

**Compiler selection**:
```rust
use wasmer::{Store, Module, Instance};
use wasmer_compiler_llvm::LLVM;

let compiler = LLVM::new();
let mut store = Store::new(compiler);
let module = Module::from_file(&store, "module.wasm")?;
```

### wasm3

**Repository**: https://github.com/wasm3/wasm3

**Written in**: C

**Execution**: Interpreter (not JIT)

**Primary use cases**:
- Embedded systems
- Microcontrollers
- IoT devices
- Low-memory environments

**Strengths**:
- Tiny footprint (~64KB)
- No JIT required (works on systems without executable memory)
- Fast for an interpreter
- Portable (runs everywhere)

**Weaknesses**:
- Slower than JIT runtimes (10-50x)
- Limited WASI support

**Embedding example (C)**:
```c
#include "wasm3.h"
#include "m3_env.h"

IM3Environment env = m3_NewEnvironment();
IM3Runtime runtime = m3_NewRuntime(env, 64*1024, NULL);
IM3Module module;
m3_ParseModule(env, &module, wasm_bytes, wasm_size);
m3_LoadModule(runtime, module);

IM3Function func;
m3_FindFunction(&func, runtime, "add");
const char* args[] = { "5", "3" };
m3_CallWithArgs(func, 2, args);
```

**Use case**: ESP32, ARM Cortex-M, environments where JIT isn't possible

### WAMR (WebAssembly Micro Runtime)

**Repository**: https://github.com/bytecodealliance/wasm-micro-runtime

**Written in**: C

**Execution modes**:
- **Interpreter**: Lowest memory, slowest
- **Fast JIT**: Quick compilation, moderate performance
- **AOT**: Pre-compile to native code, fastest

**Primary use cases**:
- Embedded systems
- IoT
- Edge computing
- Resource-constrained environments

**Strengths**:
- Multiple execution modes
- Small footprint
- Good performance for embedded
- WASI support

**Example (AOT)**:
```bash
wamrc -o module.aot module.wasm  # Compile ahead-of-time
iwasm module.aot                  # Run AOT module
```

## Specialized Runtimes

### Node.js and Deno

**Node.js** and **Deno** use V8's WASM implementation:

```javascript
// Node.js
const fs = require('fs');
const bytes = fs.readFileSync('module.wasm');
const { instance } = await WebAssembly.instantiate(bytes);

// Deno
const bytes = await Deno.readFile('module.wasm');
const { instance } = await WebAssembly.instantiate(bytes);
```

Both provide WASI support via polyfills or native modules.

### WasmEdge

**Repository**: https://github.com/WasmEdge/WasmEdge

**Written in**: C++

**Focus**: Cloud-native, serverless, edge computing

**Strengths**:
- AOT compiler
- Excellent performance
- Extended capabilities (TensorFlow, database access)
- WASI support
- Used in production (Docker, Kubernetes)

**Use cases**:
- Edge functions
- Serverless
- Service mesh sidecars

### Lucet (Archived)

Fastly's WASM runtime, now archived in favor of Wasmtime. Mentioned for historical context—pioneered AOT compilation for WASM.

## Runtime Comparison

### Performance

**Fastest** (optimized JIT):
1. V8 (browser)
2. Wasmtime (Cranelift)
3. Wasmer (LLVM backend)

**Fast startup**:
1. Wasmer (Singlepass)
2. V8 (Liftoff)
3. Wasmtime (Cranelift)

**Embedded** (low resource):
1. wasm3
2. WAMR (interpreter mode)

### Memory Usage

**Lowest**:
- wasm3: ~64 KB runtime
- WAMR: ~100 KB
- Wasmtime: ~several MB
- V8: ~10+ MB

### WASI Support

**Best**:
1. Wasmtime (reference implementation)
2. Wasmer
3. WasmEdge

**Limited**:
- wasm3
- Browser runtimes (WASI not applicable)

## Embedding WASM Runtimes

### Rust (wasmtime)

```rust
use wasmtime::*;

let engine = Engine::default();
let mut store = Store::new(&engine, ());
let module = Module::from_file(&engine, "module.wasm")?;

// Define host function
let log = Func::wrap(&mut store, |x: i32| {
    println!("WASM called log: {}", x);
});

// Provide imports
let imports = [log.into()];
let instance = Instance::new(&mut store, &module, &imports)?;

// Call exported function
let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;
run.call(&mut store, ())?;
```

### Python (wasmer)

```python
from wasmer import Store, Module, Instance, ImportObject, Function

def log_func(x: int):
    print(f"WASM called log: {x}")

store = Store()
module = Module(store, open('module.wasm', 'rb').read())

import_object = ImportObject()
import_object.register("env", {
    "log": Function(store, log_func)
})

instance = Instance(module, import_object)
instance.exports.run()
```

### JavaScript (Node.js)

```javascript
const fs = require('fs');
const bytes = fs.readFileSync('module.wasm');

const importObject = {
  env: {
    log: (x) => console.log(`WASM called log: ${x}`)
  }
};

WebAssembly.instantiate(bytes, importObject).then(({ instance }) => {
  instance.exports.run();
});
```

## Runtime Features Comparison

| Feature | Wasmtime | Wasmer | wasm3 | WAMR | V8 |
|---------|----------|--------|-------|------|-----|
| WASI | ✓ | ✓ | Partial | ✓ | N/A |
| SIMD | ✓ | ✓ | ✗ | ✓ | ✓ |
| Threads | ✓ | ✓ | ✗ | ✓ | ✓ |
| Tail calls | ✓ | ✓ | ✗ | ✗ | ✓ |
| Multi-memory | ✓ | ✓ | ✗ | ✗ | ✓ |
| GC (proposal) | In progress | In progress | ✗ | ✗ | ✓ |

## Choosing a Runtime

### For Web Applications
→ Use the browser's built-in runtime (V8, SpiderMonkey, JSC)

### For Server/CLI Applications
→ **Wasmtime** (best WASI support, production-ready)
→ **Wasmer** (if you need multiple compiler backends)

### For Embedded Systems
→ **wasm3** (smallest footprint, no JIT required)
→ **WAMR** (better performance, still small)

### For Edge Computing
→ **WasmEdge** (designed for this)
→ **Wasmtime** (also excellent)

### For Blockchain
→ **Wasmer** (used by NEAR, CosmWasm)
→ **Wasmtime** (used by some chains)

## Performance Tuning

### Compilation Caching

**Wasmtime**:
```rust
let mut config = Config::new();
config.cache_config_load_default()?;  // Enable cache
```

**Wasmer**:
Uses filesystem cache automatically.

**Browser**: Structured clone or IndexedDB

### Memory Limits

**Wasmtime**:
```rust
let mut config = Config::new();
config.max_wasm_stack(1024 * 1024);  // 1 MB stack
```

**WAMR**:
```c
RuntimeInitArgs init_args;
init_args.mem_alloc_type = Alloc_With_Pool;
init_args.mem_alloc_option.pool.heap_buf = heap_buf;
init_args.mem_alloc_option.pool.heap_size = heap_size;
```

### Profiling

Most runtimes support profiling:

**Wasmtime**:
```bash
wasmtime --profile=jitdump module.wasm
```

**Browser DevTools**: Built-in profiler shows WASM in call stacks

## Security Considerations

All major runtimes enforce WASM's security model:
- Memory isolation
- Bounds checking
- Type safety

Additional security features:

**Wasmtime**:
- Limits on memory, table size, stack
- Disabled speculative execution mitigations available

**Wasmer**:
- Metering (limit execution time)
- Gas metering (for blockchain use)

## Next Steps

We've explored the runtime landscape—from browser engines to standalone runtimes to embedded interpreters. Next, we'll dive into WASI, the WebAssembly System Interface that enables WASM to interact with operating systems securely and portably.
