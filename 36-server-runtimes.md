# Chapter 36: Server-Side Runtimes

## WebAssembly Beyond the Browser

While WASM started in browsers, it's increasingly used on servers for plugins, sandboxed execution, edge computing, and more. Server-side runtimes provide WASM execution environments optimized for backend workloads.

## Major Server-Side Runtimes

### Wasmtime

**Status**: Production-ready, Bytecode Alliance
**Language**: Rust
**Focus**: Security, standards compliance, embedding

**Installation**:
```bash
# CLI tool
curl https://wasmtime.dev/install.sh -sSf | bash

# Or via package manager
brew install wasmtime  # macOS
```

**Command-line usage**:
```bash
# Run WASI module
wasmtime run program.wasm

# With arguments
wasmtime run program.wasm -- arg1 arg2

# Map directories
wasmtime run program.wasm --dir=/host/path::guest/path

# Set environment variables
wasmtime run program.wasm --env KEY=value

# Limit resources
wasmtime run program.wasm --max-memory-size 100m
```

**Embedding in Rust**:
```rust
use wasmtime::*;

fn main() -> Result<()> {
    // Create engine and store
    let engine = Engine::default();
    let mut store = Store::new(&engine, ());

    // Load module
    let module = Module::from_file(&engine, "module.wasm")?;

    // Create instance
    let instance = Instance::new(&mut store, &module, &[])?;

    // Get exported function
    let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;

    // Call function
    let result = add.call(&mut store, (5, 3))?;
    println!("5 + 3 = {}", result);

    Ok(())
}
```

**Providing imports**:
```rust
use wasmtime::*;

struct HostState {
    counter: i32,
}

fn main() -> Result<()> {
    let engine = Engine::default();
    let mut store = Store::new(&engine, HostState { counter: 0 });

    // Define host function
    let log_func = Func::wrap(&mut store, |caller: Caller<HostState>, arg: i32| {
        println!("WASM called log with: {}", arg);
        caller.data_mut().counter += 1;
    });

    let mut linker = Linker::new(&engine);
    linker.define("env", "log", log_func)?;

    let module = Module::from_file(&engine, "module.wasm")?;
    let instance = linker.instantiate(&mut store, &module)?;

    let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;
    run.call(&mut store, ())?;

    println!("Log was called {} times", store.data().counter);

    Ok(())
}
```

**WASI support**:
```rust
use wasmtime::*;
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder};

fn main() -> Result<()> {
    let engine = Engine::default();

    // Build WASI context
    let wasi = WasiCtxBuilder::new()
        .inherit_stdio()
        .inherit_args()?
        .env("NAME", "World")?
        .preopened_dir(
            Dir::open_ambient_dir(".", ambient_authority())?,
            "."
        )?
        .build();

    let mut store = Store::new(&engine, wasi);

    // Add WASI to linker
    let mut linker = Linker::new(&engine);
    wasmtime_wasi::add_to_linker(&mut linker, |s| s)?;

    let module = Module::from_file(&engine, "wasi-program.wasm")?;
    let instance = linker.instantiate(&mut store, &module)?;

    // Call _start (WASI entry point)
    let start = instance.get_typed_func::<(), ()>(&mut store, "_start")?;
    start.call(&mut store, ())?;

    Ok(())
}
```

### Wasmer

**Status**: Production-ready
**Language**: Rust
**Focus**: Portability, embedding, package management

**Installation**:
```bash
curl https://get.wasmer.io -sSfL | sh
```

**Command-line usage**:
```bash
# Run WASI module
wasmer run program.wasm

# With package manager (WAPM)
wasmer run cowsay "Hello!"

# Map directories
wasmer run program.wasm --mapdir=/host:/guest

# Set environment
wasmer run program.wasm --env KEY=value
```

**Embedding in Rust**:
```rust
use wasmer::{Store, Module, Instance, Value, imports};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut store = Store::default();

    // Load module
    let module = Module::from_file(&store, "module.wasm")?;

    // Create imports
    let import_object = imports! {
        "env" => {
            "log" => Function::new_native(&mut store, |x: i32| {
                println!("WASM says: {}", x);
            }),
        }
    };

    // Instantiate
    let instance = Instance::new(&mut store, &module, &import_object)?;

    // Call function
    let add = instance.exports.get_function("add")?;
    let result = add.call(&mut store, &[Value::I32(5), Value::I32(3)])?;

    println!("Result: {:?}", result);

    Ok(())
}
```

**WASI support**:
```rust
use wasmer::{Store, Module, Instance};
use wasmer_wasi::WasiState;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut store = Store::default();

    // Build WASI state
    let mut wasi_env = WasiState::new("program")
        .args(&["arg1", "arg2"])
        .env("KEY", "value")
        .map_dir("/host/path", ".")?
        .finalize(&mut store)?;

    let module = Module::from_file(&store, "wasi-program.wasm")?;

    let import_object = wasi_env.import_object(&mut store, &module)?;
    let instance = Instance::new(&mut store, &module, &import_object)?;

    // Run
    let start = instance.exports.get_function("_start")?;
    start.call(&mut store, &[])?;

    Ok(())
}
```

### WasmEdge

**Status**: Production-ready, CNCF project
**Language**: C++
**Focus**: Edge computing, cloud-native, AI/ML

**Installation**:
```bash
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash
```

**Command-line usage**:
```bash
# Run WASI module
wasmedge run program.wasm

# With networking
wasmedge run --env LISTEN_ADDR=0.0.0.0:8080 server.wasm

# With Tensorflow
wasmedge run --dir .:. tf-model.wasm
```

**Embedding in Rust**:
```rust
use wasmedge_sdk::{
    config::{CommonConfigOptions, ConfigBuilder},
    Vm, WasmVal,
};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Configure runtime
    let config = ConfigBuilder::new(CommonConfigOptions::default())
        .with_wasi(true)
        .build()?;

    // Create VM
    let mut vm = Vm::new(Some(config))?;

    // Load module
    vm.load_wasm_from_file("module.wasm")?;
    vm.validate()?;
    vm.instantiate()?;

    // Run function
    let result = vm.run_func(Some("main"), "add", [
        WasmVal::from_i32(5),
        WasmVal::from_i32(3),
    ])?;

    println!("Result: {:?}", result);

    Ok(())
}
```

### wasm3

**Status**: Production-ready
**Language**: C
**Focus**: Embedded systems, minimal footprint

**Key Features**:
- Interpreter (not JIT)
- Extremely small binary (100KB)
- Low memory usage
- No dependencies

**Embedding in C**:
```c
#include "wasm3.h"
#include "m3_env.h"

// Host function
m3ApiRawFunction(m3_log) {
    m3ApiGetArg(int32_t, value);
    printf("WASM says: %d\n", value);
    m3ApiSuccess();
}

int main() {
    // Read WASM file
    FILE* f = fopen("module.wasm", "rb");
    fseek(f, 0, SEEK_END);
    size_t fsize = ftell(f);
    fseek(f, 0, SEEK_SET);

    uint8_t* wasm = malloc(fsize);
    fread(wasm, 1, fsize, f);
    fclose(f);

    // Create environment
    IM3Environment env = m3_NewEnvironment();
    IM3Runtime runtime = m3_NewRuntime(env, 64*1024, NULL);
    IM3Module module;

    // Parse and load
    m3_ParseModule(env, &module, wasm, fsize);
    m3_LoadModule(runtime, module);

    // Link host function
    m3_LinkRawFunction(module, "env", "log", "v(i)", &m3_log);

    // Find and call function
    IM3Function func;
    m3_FindFunction(&func, runtime, "add");

    const char* args[] = { "5", "3" };
    m3_CallArgv(func, 2, args);

    int32_t result;
    m3_GetResultsV(func, &result);
    printf("Result: %d\n", result);

    // Cleanup
    m3_FreeRuntime(runtime);
    m3_FreeEnvironment(env);
    free(wasm);

    return 0;
}
```

### WAMR (WebAssembly Micro Runtime)

**Status**: Production-ready, Bytecode Alliance
**Language**: C
**Focus**: Embedded, IoT, minimal footprint

**Compilation modes**:
- **Interpreter**: Smallest, slowest
- **AOT**: Fastest, platform-specific
- **JIT**: Balance of speed and portability

**Embedding**:
```c
#include "wasm_export.h"

int main() {
    // Initialize runtime
    wasm_runtime_init();

    // Load module
    FILE* f = fopen("module.wasm", "rb");
    fseek(f, 0, SEEK_END);
    size_t size = ftell(f);
    fseek(f, 0, SEEK_SET);

    uint8_t* buffer = malloc(size);
    fread(buffer, 1, size, f);
    fclose(f);

    // Parse module
    char error_buf[128];
    wasm_module_t module = wasm_runtime_load(
        buffer, size,
        error_buf, sizeof(error_buf)
    );

    // Instantiate
    wasm_module_inst_t instance = wasm_runtime_instantiate(
        module, 64*1024,  // stack size
        64*1024,          // heap size
        error_buf, sizeof(error_buf)
    );

    // Find function
    wasm_function_inst_t func = wasm_runtime_lookup_function(
        instance, "add", NULL
    );

    // Call function
    uint32_t args[2] = { 5, 3 };
    uint32_t result;

    if (wasm_runtime_call_wasm(instance, NULL, func, 2, args)) {
        result = args[0];  // Result in first arg
        printf("Result: %u\n", result);
    }

    // Cleanup
    wasm_runtime_deinstantiate(instance);
    wasm_runtime_unload(module);
    wasm_runtime_destroy();
    free(buffer);

    return 0;
}
```

## Embedding in Other Languages

### Python (wasmtime-py)

```python
from wasmtime import Store, Module, Instance, Func, FuncType

# Create store
store = Store()

# Load module
module = Module.from_file(store.engine, "module.wasm")

# Define host function
def log_func(val):
    print(f"WASM says: {val}")

log = Func(store, FuncType([ValType.i32()], []), log_func)

# Instantiate with imports
instance = Instance(store, module, [log])

# Call exported function
add = instance.exports(store)["add"]
result = add(store, 5, 3)
print(f"5 + 3 = {result}")
```

**WASI support**:
```python
from wasmtime import Store, Module, Instance
from wasmtime_wasi import WasiConfig, WasiInstance

store = Store()

# Configure WASI
wasi_config = WasiConfig()
wasi_config.argv = ["program", "arg1"]
wasi_config.env = [("KEY", "value")]
wasi_config.inherit_stdout()
wasi_config.inherit_stderr()

wasi = WasiInstance(store, "wasi_snapshot_preview1", wasi_config)

module = Module.from_file(store.engine, "wasi-program.wasm")
instance = Instance(store, module, wasi.bind_imports())

# Run
start = instance.exports(store)["_start"]
start(store)
```

### Node.js (wasmtime)

```javascript
const { Store, Module, Instance } = require('@bytecodealliance/wasmtime');
const fs = require('fs');

async function main() {
    const store = new Store();

    // Load module
    const bytes = fs.readFileSync('module.wasm');
    const module = await Module.deserialize(store.engine, bytes);

    // Define imports
    const imports = {
        env: {
            log: (val) => console.log(`WASM says: ${val}`)
        }
    };

    // Instantiate
    const instance = new Instance(store, module, imports);

    // Call function
    const add = instance.exports.add;
    const result = add(5, 3);
    console.log(`5 + 3 = ${result}`);
}

main();
```

### Go

```go
package main

import (
    "fmt"
    "os"
    wasmtime "github.com/bytecodealliance/wasmtime-go"
)

func main() {
    // Load WASM
    wasm, _ := os.ReadFile("module.wasm")

    // Create engine and store
    engine := wasmtime.NewEngine()
    store := wasmtime.NewStore(engine)

    // Compile module
    module, _ := wasmtime.NewModule(engine, wasm)

    // Define imports
    logType := wasmtime.NewFuncType(
        []*wasmtime.ValType{wasmtime.NewValType(wasmtime.KindI32)},
        []*wasmtime.ValType{},
    )

    log := wasmtime.NewFunc(
        store,
        logType,
        func(caller *wasmtime.Caller, args []wasmtime.Val) ([]wasmtime.Val, *wasmtime.Trap) {
            fmt.Printf("WASM says: %d\n", args[0].I32())
            return []wasmtime.Val{}, nil
        },
    )

    // Instantiate
    instance, _ := wasmtime.NewInstance(store, module, []wasmtime.AsExtern{log})

    // Call function
    add := instance.GetExport(store, "add").Func()
    result, _ := add.Call(store, 5, 3)
    fmt.Printf("5 + 3 = %d\n", result.(int32))
}
```

## Use Cases

### 1. Plugin Systems

Execute untrusted user plugins safely:

```rust
use wasmtime::*;
use std::time::Duration;

struct PluginRuntime {
    engine: Engine,
}

impl PluginRuntime {
    fn new() -> Self {
        let mut config = Config::new();
        config.consume_fuel(true);  // Enable fuel metering

        Self {
            engine: Engine::new(&config).unwrap(),
        }
    }

    fn run_plugin(&self, plugin_bytes: &[u8], input: &str) -> Result<String> {
        let mut store = Store::new(&self.engine, ());

        // Add 1 million fuel units (limits execution)
        store.add_fuel(1_000_000)?;

        let module = Module::from_binary(&self.engine, plugin_bytes)?;

        // Create imports
        let mut linker = Linker::new(&self.engine);

        // Provide safe host functions
        linker.func_wrap(
            "env",
            "get_input",
            |mut caller: Caller<'_, ()>| -> i32 {
                // Return pointer to input string
                0  // Simplified
            }
        )?;

        let instance = linker.instantiate(&mut store, &module)?;

        // Call plugin
        let process = instance
            .get_typed_func::<(), i32>(&mut store, "process")?;

        let result_ptr = process.call(&mut store, ())?;

        // Read result from memory
        let memory = instance.get_memory(&mut store, "memory").unwrap();
        let result = read_string_from_memory(&store, &memory, result_ptr)?;

        Ok(result)
    }
}

// Usage
let runtime = PluginRuntime::new();
let output = runtime.run_plugin(&plugin_wasm, "input data")?;
println!("Plugin output: {}", output);
```

### 2. Serverless Functions

Execute user functions in isolated contexts:

```rust
use wasmtime::*;
use std::collections::HashMap;

struct FunctionInvoker {
    engine: Engine,
    module_cache: HashMap<String, Module>,
}

impl FunctionInvoker {
    fn invoke(&mut self, function_id: &str, event: serde_json::Value) -> Result<serde_json::Value> {
        let module = self.module_cache.get(function_id).unwrap();

        let mut store = Store::new(&self.engine, ());

        // Provide event as JSON
        let mut linker = Linker::new(&self.engine);
        linker.func_wrap("env", "get_event", |caller: Caller<()>| -> i32 {
            // Return pointer to serialized event
            0
        })?;

        let instance = linker.instantiate(&mut store, &module)?;

        let handler = instance.get_typed_func::<(), i32>(&mut store, "handler")?;
        let result_ptr = handler.call(&mut store, ())?;

        // Deserialize result
        let memory = instance.get_memory(&mut store, "memory").unwrap();
        let result_json = read_json_from_memory(&store, &memory, result_ptr)?;

        Ok(result_json)
    }
}
```

### 3. Data Processing Pipelines

Execute transformation logic safely:

```rust
struct DataPipeline {
    stages: Vec<Module>,
    engine: Engine,
}

impl DataPipeline {
    fn process(&self, mut data: Vec<u8>) -> Result<Vec<u8>> {
        let mut store = Store::new(&self.engine, ());

        for (i, stage) in self.stages.iter().enumerate() {
            let instance = Instance::new(&mut store, stage, &[])?;

            // Allocate input in WASM memory
            let alloc = instance.get_typed_func::<i32, i32>(&mut store, "alloc")?;
            let input_ptr = alloc.call(&mut store, data.len() as i32)?;

            let memory = instance.get_memory(&mut store, "memory").unwrap();
            memory.write(&mut store, input_ptr as usize, &data)?;

            // Process
            let process = instance.get_typed_func::<(i32, i32), i32>(
                &mut store,
                "process"
            )?;
            let output_ptr = process.call(&mut store, (input_ptr, data.len() as i32))?;

            // Read output
            let output_len = read_i32(&store, &memory, output_ptr)?;
            data = read_bytes(&store, &memory, output_ptr + 4, output_len as usize)?;

            println!("Stage {} complete: {} bytes", i, data.len());
        }

        Ok(data)
    }
}
```

### 4. Multi-Tenant Isolation

Safely execute code from multiple tenants:

```rust
struct TenantRuntime {
    engine: Engine,
}

impl TenantRuntime {
    fn execute_for_tenant(&self, tenant_id: &str, code: &[u8]) -> Result<()> {
        let module = Module::from_binary(&self.engine, code)?;

        let mut config = Config::new();
        config.max_wasm_stack(128 * 1024);  // 128KB stack limit

        let mut store_limits = StoreLimits::default();
        store_limits.memory_size(10 * 1024 * 1024);  // 10MB memory limit
        store_limits.instances(10);  // Max 10 instances

        let limiter = StoreLimitsBuilder::new()
            .memory_size(10 * 1024 * 1024)
            .build();

        let mut store = Store::new(&self.engine, limiter);
        store.limiter(|s| s);

        // Execute with limits
        let instance = Instance::new(&mut store, &module, &[])?;

        let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;
        run.call(&mut store, ())?;

        println!("Executed for tenant: {}", tenant_id);

        Ok(())
    }
}
```

## Performance Considerations

### AOT Compilation

Pre-compile WASM for faster startup:

```bash
# Wasmtime
wasmtime compile module.wasm -o module.cwasm

# Wasmer
wasmer compile module.wasm -o module.wasmu
```

**Loading AOT modules**:
```rust
use wasmtime::*;

let engine = Engine::default();

// Deserialize pre-compiled module (fast)
let module = unsafe {
    Module::deserialize_file(&engine, "module.cwasm")?
};

// vs. compiling from WASM (slower)
let module = Module::from_file(&engine, "module.wasm")?;
```

### Module Caching

Cache compiled modules in production:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

struct ModuleCache {
    engine: Engine,
    cache: Arc<Mutex<HashMap<String, Module>>>,
}

impl ModuleCache {
    fn get_or_compile(&self, path: &str) -> Result<Module> {
        let mut cache = self.cache.lock().unwrap();

        if let Some(module) = cache.get(path) {
            return Ok(module.clone());
        }

        let module = Module::from_file(&self.engine, path)?;
        cache.insert(path.to_string(), module.clone());

        Ok(module)
    }
}
```

### Instance Pooling

Reuse instances for better performance:

```rust
use wasmtime::*;

struct InstancePool {
    engine: Engine,
    module: Module,
    pool: Vec<Instance>,
}

impl InstancePool {
    fn get_instance(&mut self) -> Result<Instance> {
        if let Some(instance) = self.pool.pop() {
            // Reuse existing instance
            Ok(instance)
        } else {
            // Create new instance
            let mut store = Store::new(&self.engine, ());
            Instance::new(&mut store, &self.module, &[])
        }
    }

    fn return_instance(&mut self, instance: Instance) {
        self.pool.push(instance);
    }
}
```

## Security Best Practices

### 1. Resource Limits

```rust
let mut config = Config::new();
config.max_wasm_stack(256 * 1024);  // 256KB stack
config.consume_fuel(true);

let mut store = Store::new(&engine, ());
store.add_fuel(10_000_000)?;  // Limit execution time
store.limiter(|_| &mut limits);  // Memory limits
```

### 2. Capability-Based Security

Only provide necessary imports:

```rust
// ✗ Bad: Expose everything
linker.define("env", "read_file", read_file_func)?;
linker.define("env", "write_file", write_file_func)?;
linker.define("env", "network_request", network_func)?;

// ✓ Good: Minimal capabilities
linker.define("env", "log", log_func)?;
// Don't expose file or network access unless needed
```

### 3. Validate Modules

```rust
let module = Module::from_file(&engine, "untrusted.wasm")?;

// Module is validated automatically, but you can also:
let module_binary = std::fs::read("untrusted.wasm")?;
if !Module::validate(&engine, &module_binary)? {
    return Err(anyhow!("Invalid module"));
}
```

### 4. Timeout Execution

```rust
use std::time::Duration;
use std::thread;

let handle = thread::spawn(move || {
    instance.get_func("run").unwrap().call(&[]).unwrap()
});

match handle.join_timeout(Duration::from_secs(5)) {
    Ok(_) => println!("Completed"),
    Err(_) => println!("Timeout!"),
}
```

## Monitoring and Observability

```rust
use wasmtime::*;

struct InstrumentedStore {
    store: Store<()>,
    call_count: usize,
    fuel_consumed: u64,
}

impl InstrumentedStore {
    fn call_function(&mut self, func: &Func, args: &[Val]) -> Result<Box<[Val]>> {
        let fuel_before = self.store.fuel_consumed().unwrap_or(0);

        let result = func.call(&mut self.store, args)?;

        let fuel_after = self.store.fuel_consumed().unwrap_or(0);

        self.call_count += 1;
        self.fuel_consumed += fuel_after - fuel_before;

        println!(
            "Function call #{}: {} fuel units",
            self.call_count,
            fuel_after - fuel_before
        );

        Ok(result)
    }

    fn report(&self) {
        println!("Total calls: {}", self.call_count);
        println!("Total fuel consumed: {}", self.fuel_consumed);
    }
}
```

## Next Steps

Server-side WASM runtimes enable safe, fast, and portable execution of untrusted code. With wasmtime, wasmer, and others, you can build plugin systems, serverless platforms, and multi-tenant applications with strong isolation guarantees.

Next, we'll explore building CLI tools with WebAssembly, creating standalone executable tools that leverage WASM's portability.
