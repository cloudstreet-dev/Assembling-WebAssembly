# Chapter 22: Clang Direct WASM Compilation

## Beyond Emscripten

While Emscripten is comprehensive, sometimes you want direct WASM compilation without JavaScript glue code. Clang can target WASM directly via LLVM's WASM backend.

**When to use direct Clang**:
- Minimal binaries without JavaScript dependencies
- WASI applications
- Embedded/constrained environments
- Full control over the runtime
- Integration with non-browser WASM runtimes

## Setup

### wasi-sdk

The easiest way to get Clang targeting WASM:

```bash
# Download wasi-sdk
wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-20/wasi-sdk-20.0-linux.tar.gz
tar xf wasi-sdk-20.0-linux.tar.gz

# Add to PATH
export WASI_SDK_PATH=/path/to/wasi-sdk-20.0
export PATH="$WASI_SDK_PATH/bin:$PATH"
```

### System Clang

If your system Clang supports WASM:

```bash
clang --version  # Check for wasm support
clang --print-targets | grep wasm
```

## Basic Compilation

### Minimal Example

**hello.c**:
```c
int add(int a, int b) {
    return a + b;
}
```

**Compile**:
```bash
clang --target=wasm32 -nostdlib -Wl,--no-entry \
    -Wl,--export-all -o hello.wasm hello.c
```

**Flags explained**:
- `--target=wasm32`: Target 32-bit WASM
- `-nostdlib`: Don't link standard library
- `-Wl,--no-entry`: No main() entry point
- `-Wl,--export-all`: Export all functions

### With Standard Library (WASI)

```c
#include <stdio.h>

int main() {
    printf("Hello from WASI!\n");
    return 0;
}
```

**Compile with wasi-sdk**:
```bash
$WASI_SDK_PATH/bin/clang hello.c -o hello.wasm
```

**Run**:
```bash
wasmtime hello.wasm
```

## Memory Management

### Manual Memory Layout

```c
// Declare memory
__attribute__((export_name("memory")))
unsigned char memory[65536];  // 64KB

__attribute__((export_name("alloc")))
void* alloc(unsigned int size) {
    static unsigned int offset = 0;
    void* ptr = memory + offset;
    offset += size;
    return ptr;
}
```

### Using System Allocator (with WASI)

```c
#include <stdlib.h>

__attribute__((export_name("create_buffer")))
void* create_buffer(int size) {
    return malloc(size);
}

__attribute__((export_name("free_buffer")))
void free_buffer(void* ptr) {
    free(ptr);
}
```

## Function Exports

### Export Attributes

```c
// Export with original name
__attribute__((export_name("add")))
int my_add_function(int a, int b) {
    return a + b;
}

// Multiple exports
__attribute__((export_name("multiply")))
__attribute__((visibility("default")))
int mul(int a, int b) {
    return a * b;
}
```

### Linker Exports

```bash
clang --target=wasm32 -nostdlib -Wl,--no-entry \
    -Wl,--export=add -Wl,--export=multiply \
    -o math.wasm math.c
```

## Imports

### Declaring Imports

```c
// Import function from host
__attribute__((import_module("env"), import_name("log")))
void host_log(int value);

__attribute__((import_module("env"), import_name("get_time")))
double host_get_time(void);

// Use imports
void my_function() {
    host_log(42);
    double time = host_get_time();
}
```

## Linear Memory

### Accessing Memory

```c
__attribute__((export_name("write_data")))
void write_data(unsigned char* ptr, int offset, int value) {
    ptr[offset] = value;
}

__attribute__((export_name("read_data")))
int read_data(unsigned char* ptr, int offset) {
    return ptr[offset];
}
```

### Growing Memory

```c
__attribute__((export_name("get_memory_size")))
int get_memory_size() {
    return __builtin_wasm_memory_size(0) * 65536;  // Pages to bytes
}

__attribute__((export_name("grow_memory")))
int grow_memory(int pages) {
    return __builtin_wasm_memory_grow(0, pages);
}
```

## SIMD Support

Clang supports WASM SIMD intrinsics:

```c
#include <wasm_simd128.h>

void vector_add(float* a, float* b, float* result, int count) {
    for (int i = 0; i < count; i += 4) {
        v128_t va = wasm_v128_load(&a[i]);
        v128_t vb = wasm_v128_load(&b[i]);
        v128_t vr = wasm_f32x4_add(va, vb);
        wasm_v128_store(&result[i], vr);
    }
}
```

**Compile**:
```bash
clang --target=wasm32-wasi -msimd128 -o simd.wasm simd.c
```

## Optimization

### Size Optimization

```bash
clang --target=wasm32 -Os -flto \
    -Wl,--lto-O3 -Wl,--strip-all \
    -o optimized.wasm program.c

# Further optimization with wasm-opt
wasm-opt -Oz -o final.wasm optimized.wasm
```

### Performance Optimization

```bash
clang --target=wasm32-wasi -O3 -flto \
    -march=wasm32 -mtune=generic \
    -o fast.wasm program.c
```

## WASI System Calls

### File I/O

```c
#include <stdio.h>
#include <string.h>

int main() {
    FILE* f = fopen("output.txt", "w");
    if (!f) {
        perror("fopen");
        return 1;
    }

    fprintf(f, "Hello from WASI!\n");
    fclose(f);

    return 0;
}
```

### Command-Line Arguments

```c
#include <stdio.h>

int main(int argc, char* argv[]) {
    printf("Program: %s\n", argv[0]);

    for (int i = 1; i < argc; i++) {
        printf("Arg %d: %s\n", i, argv[i]);
    }

    return 0;
}
```

## Embedding Clang WASM

### In JavaScript (Browser)

```javascript
// Minimal WASM loading
const response = await fetch('module.wasm');
const buffer = await response.arrayBuffer();
const { instance } = await WebAssembly.instantiate(buffer, {
    env: {
        // Provide imports if needed
    }
});

const result = instance.exports.add(5, 3);
console.log(result);  // 8
```

### In Wasmtime (Rust)

```rust
use wasmtime::*;

fn main() -> Result<()> {
    let engine = Engine::default();
    let module = Module::from_file(&engine, "module.wasm")?;
    let mut store = Store::new(&engine, ());
    let instance = Instance::new(&mut store, &module, &[])?;

    let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
    let result = add.call(&mut store, (5, 3))?;

    println!("Result: {}", result);
    Ok(())
}
```

## Custom Sections

Add custom data to WASM modules:

```c
__attribute__((section(".custom_section"), used))
const char custom_data[] = "Custom metadata";
```

## Debugging

### DWARF Debug Info

```bash
clang --target=wasm32-wasi -g -o debug.wasm program.c
```

### Source Maps

```bash
clang --target=wasm32-wasi -g -fdebug-prefix-map=$(pwd)=. \
    -o program.wasm program.c
```

## Interop with Other Languages

### Calling from Rust

```rust
// Rust calling Clang-compiled WASM
use wasmtime::*;

let engine = Engine::default();
let module = Module::from_file(&engine, "clang_module.wasm")?;
let mut store = Store::new(&engine, ());
let instance = Instance::new(&mut store, &module, &[])?;

let func = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
let result = func.call(&mut store, (10, 20))?;
```

### Calling Rust from Clang WASM

Compile both to WASM and link:

```bash
# Compile Rust library
cargo build --target wasm32-wasi --release

# Compile C code
clang --target=wasm32-wasi -c main.c -o main.o

# Link together
clang --target=wasm32-wasi main.o \
    target/wasm32-wasi/release/libmyrust.a \
    -o combined.wasm
```

## Advanced Linker Options

### Custom Memory

```bash
clang --target=wasm32 -nostdlib \
    -Wl,--initial-memory=1048576 \  # 1MB initial
    -Wl,--max-memory=16777216 \     # 16MB max
    -o program.wasm program.c
```

### Shared Memory

```bash
clang --target=wasm32 -nostdlib \
    -Wl,--shared-memory \
    -Wl,--max-memory=16777216 \
    -o shared.wasm program.c
```

### Import Memory

```bash
clang --target=wasm32 -nostdlib \
    -Wl,--import-memory \
    -o program.wasm program.c
```

## Practical Example: Math Library

**math_lib.c**:
```c
#include <math.h>

__attribute__((export_name("fast_sqrt")))
double fast_sqrt(double x) {
    return sqrt(x);
}

__attribute__((export_name("fast_sin")))
double fast_sin(double x) {
    return sin(x);
}

__attribute__((export_name("fast_pow")))
double fast_pow(double base, double exp) {
    return pow(base, exp);
}

__attribute__((export_name("vector_magnitude")))
double vector_magnitude(double x, double y, double z) {
    return sqrt(x*x + y*y + z*z);
}
```

**Compile**:
```bash
clang --target=wasm32-wasi -O3 -o math_lib.wasm math_lib.c -lm
```

**Use**:
```javascript
const { instance } = await WebAssembly.instantiateStreaming(fetch('math_lib.wasm'));

console.log(instance.exports.fast_sqrt(16));           // 4
console.log(instance.exports.fast_sin(Math.PI / 2));  // 1
console.log(instance.exports.vector_magnitude(3, 4, 0)); // 5
```

## Comparison: Emscripten vs Direct Clang

| Feature | Emscripten | Direct Clang |
|---------|------------|--------------|
| Binary size | Larger | Smaller |
| JS glue code | Included | None |
| Standard library | Comprehensive | Minimal/WASI |
| Browser APIs | Full access | Manual bindings |
| WASI support | Limited | Excellent |
| Learning curve | Moderate | Steeper |
| Use case | Web apps | CLI tools, servers |

## Best Practices

1. **Use wasi-sdk**: Simplifies WASI development
2. **Optimize for size**: Use `-Os` and wasm-opt
3. **Export explicitly**: Only export what's needed
4. **Test in target runtime**: wasmtime, wasmer, browser
5. **Leverage SIMD**: Use SIMD intrinsics for performance
6. **Profile**: Use browser DevTools or wasmtime profiling
7. **Version carefully**: WASM evolves; test compatibility

## Next Steps

Direct Clang compilation gives you full control over WASM output without Emscripten's abstractions. It's ideal for WASI applications, minimal web modules, and integration with non-browser runtimes. Next, we'll explore AssemblyScript, a TypeScript-like language designed specifically for WebAssembly.
