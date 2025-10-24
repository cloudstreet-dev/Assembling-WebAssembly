# Chapter 15: Debugging

## The Challenge of Debugging WASM

WebAssembly bytecode isn't human-readable. Without proper tooling, debugging WASM is like debugging assembly—possible, but painful. Fortunately, modern tooling has made WASM debugging approachable.

Challenges:
- Binary format is opaque
- Source code is in a different language (Rust, C++, etc.)
- Multiple compilation stages obscure the connection
- Stack traces show WASM function indices, not names

Solutions:
- Source maps
- DWARF debug information
- Browser DevTools integration
- Standalone debuggers

## Source Maps

Source maps link compiled WASM back to original source code. When enabled, debuggers can:
- Show original source, not WASM bytecode
- Set breakpoints in source files
- Step through source lines
- Show variable names and values

### Generating Source Maps

**Rust**:

```toml
[profile.dev]
debug = true  # Include debug info

[profile.release]
debug = true  # Even in release builds
```

Or via command line:

```bash
cargo build --target wasm32-unknown-unknown
# Debug info is included by default in debug builds
```

**C/C++ (Emscripten)**:

```bash
emcc -g4 -o output.js source.c
# -g4 includes full DWARF debug information
```

**C/C++ (clang)**:

```bash
clang --target=wasm32-unknown-unknown -g source.c -o module.wasm
```

**AssemblyScript**:

```bash
asc module.ts --sourceMap --outFile module.wasm
```

### Using Source Maps in Browsers

Modern browsers automatically use source maps when available:

1. Compile with debug info
2. Load WASM in browser
3. Open DevTools
4. Navigate to Sources tab
5. See your original source code

**Chrome DevTools**:
- Sources panel shows WASM modules
- Set breakpoints in source code
- Step through code
- Inspect variables
- View call stack

**Firefox DevTools**:
- Debugger panel shows WASM
- Excellent WASM support
- Sometimes first to support new features

## DWARF Debug Information

DWARF is a standardized debug data format embedded in WASM modules. It contains:
- Function names
- Variable names and types
- Line number mappings
- Type information
- Inline function data

### Inspecting DWARF

```bash
# Using llvm-dwarfdump
llvm-dwarfdump module.wasm

# Using wasm-objdump from WABT
wasm-objdump --details module.wasm
```

Example output:

```
.debug_info contents:
0x0000000b: DW_TAG_compile_unit
              DW_AT_producer ("rustc version 1.70.0")
              DW_AT_language (DW_LANG_Rust)
              DW_AT_name ("main.rs")

0x00000023: DW_TAG_subprogram
              DW_AT_name ("add")
              DW_AT_decl_file ("main.rs")
              DW_AT_decl_line (5)
```

### Stripping Debug Info

Debug info increases binary size significantly:

```bash
# Original with debug info
-rw-r--r--  1 user  staff  1.2M  module.wasm

# Stripped
wasm-strip module.wasm
-rw-r--r--  1 user  staff  120K  module.wasm
```

**When to strip**:
- Production builds where size matters
- After testing/debugging

**When to keep**:
- Development
- Post-mortem debugging
- When size isn't critical

## Browser Debugging

### Chrome DevTools

**Opening DevTools**: F12 or Cmd+Option+I (Mac)

**Debugging workflow**:

1. **Load WASM module**:
```javascript
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm')
);
```

2. **Open Sources panel**:
- See WASM modules listed
- If debug info exists, see original source
- Otherwise, see disassembled WAT

3. **Set breakpoints**:
- Click line numbers in source
- Or use `debugger;` in JavaScript before calling WASM

4. **Inspect state**:
- Hover over variables
- Use Console to evaluate expressions
- Check call stack
- View memory (Memory panel)

**Memory inspection**:

```javascript
// In console after pausing at breakpoint
const memory = instance.exports.memory;
const bytes = new Uint8Array(memory.buffer);
console.log(bytes.slice(0, 100));  // First 100 bytes
```

**Linear memory viewer**:
- Go to Memory tab
- Select WASM memory
- View as hex dump or array

### Firefox DevTools

Firefox has excellent WASM debugging:

1. **Better WASM display**: Often renders WASM more clearly
2. **Inline variables**: Shows local variables inline
3. **Type information**: Better type display

**Enabling WASM debugging**:
Usually enabled by default. Check `about:config`:
- `devtools.debugger.features.wasm` should be `true`

### Safari Web Inspector

Safari supports WASM debugging with similar features:

1. **Web Inspector**: Cmd+Option+I
2. **Sources tab**: Shows WASM modules
3. **Set breakpoints**: Click line numbers
4. **Inspect state**: Hover, console, call stack

## Logging and Tracing

### Console Logging from WASM

Import console functions:

**Rust**:

```rust
extern "C" {
    fn log(x: i32);
}

#[no_mangle]
pub extern "C" fn debug_function() {
    unsafe { log(42); }
}
```

JavaScript side:

```javascript
const importObject = {
  env: {
    log: (x) => console.log(`WASM logged: ${x}`)
  }
};
```

**Better: wasm-bindgen**:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    log(&format!("Hello, {}!", name));
}
```

### printf-style Debugging

**C/C++ with Emscripten**:

```c
#include <stdio.h>

int main() {
    printf("Debug: value = %d\n", 42);
    return 0;
}
```

Emscripten redirects printf to console.log.

**C with WASI**:

```c
#include <stdio.h>

int main() {
    fprintf(stderr, "Debug output\n");  // To stderr
    printf("Regular output\n");          // To stdout
    return 0;
}
```

```bash
wasmtime module.wasm 2>debug.log  # Redirect stderr
```

### Structured Logging

**Rust with log crate**:

```rust
use log::{info, warn, error};

pub fn complex_function() {
    info!("Starting complex operation");

    let result = do_something();

    if result.is_err() {
        error!("Operation failed: {:?}", result);
    } else {
        info!("Operation succeeded");
    }
}
```

Configure logger for WASM:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn init() {
    console_log::init_with_level(log::Level::Debug).unwrap();
}
```

## Assertions and Panics

### Rust Panics

Panics in Rust WASM show stack traces:

```rust
pub fn might_panic(x: i32) {
    assert!(x > 0, "x must be positive, got {}", x);
    // Or:
    if x <= 0 {
        panic!("x must be positive, got {}", x);
    }
}
```

In browser console:

```
panicked at 'x must be positive, got -5', src/lib.rs:42:5
```

With debug info, you get file and line number.

### Controlled Traps

You can trigger traps intentionally:

**WAT**:
```wasm
(func $trap_here
  unreachable  ;; Immediate trap
)
```

**Rust**:
```rust
pub fn trap_here() {
    std::panic!("Intentional trap for debugging");
}
```

## Standalone Debuggers

### wasm-gdb

GDB with WASM support (experimental):

```bash
# Requires GDB built with WASM support
wasm-gdb --args wasmtime module.wasm

(gdb) break add_function
(gdb) run
(gdb) step
(gdb) print variable_name
```

Limited support, but improving.

### LLDB

LLDB has some WASM support:

```bash
lldb -- wasmtime module.wasm

(lldb) breakpoint set --name add_function
(lldb) run
(lldb) step
```

## Debugging Rust WASM

### wasm-pack with Debugging

```bash
wasm-pack build --dev  # Debug mode

# Or with release optimizations but debug info:
wasm-pack build --profiling
```

### Common Rust Issues

**Issue**: "RuntimeError: unreachable executed"

**Cause**: Panic occurred, but panic handler trapped

**Solution**: Check console for panic message, examine source at that line

**Issue**: "TypeError: Cannot read property 'X' of undefined"

**Cause**: JavaScript trying to access WASM export that doesn't exist

**Solution**: Check wasm-bindgen generated code, ensure function is marked `#[wasm_bindgen]`

**Issue**: Memory access out of bounds

**Cause**: Buffer overflow, incorrect pointer arithmetic

**Solution**: Use Rust's safe abstractions, avoid `unsafe` unless necessary

### rust-analyzer

Use rust-analyzer in VS Code for:
- Inline type hints
- Go to definition (works across WASM boundary with wasm-bindgen)
- Error checking before compilation

## Debugging C/C++ WASM

### Emscripten Debugging

```bash
# Full debug build
emcc -g4 -s ASSERTIONS=1 -o module.js module.c

# ASSERTIONS=1 enables runtime checks
# Catches buffer overflows, null pointer dereferences, etc.
```

**Sanitizers** (partial support):

```bash
emcc -fsanitize=address module.c  # AddressSanitizer (partial)
```

### Common C/C++ Issues

**Issue**: Segmentation fault / trap

**Cause**: Null pointer dereference, buffer overflow, use-after-free

**Solution**: Enable assertions, use AddressSanitizer, check pointers

**Issue**: Stack overflow

**Cause**: Deep recursion, large local arrays

**Solution**: Increase stack size with `-s STACK_SIZE=5MB` or move data to heap

## Performance Profiling

### Browser Profiler

**Chrome DevTools**:

1. Open Performance tab
2. Click Record
3. Execute WASM code
4. Stop recording
5. Examine flame graph

WASM functions appear in the flame graph. With debug info, see function names.

**Firefox Profiler**:

1. Open Performance tab (or use profiler.firefox.com)
2. Start profiling
3. Run WASM code
4. Stop and analyze

Shows WASM functions with names if debug info is available.

### wasmtime Profiling

```bash
# Generate perf map for Linux perf tool
wasmtime --profile=perfmap module.wasm

# Use with perf
perf record wasmtime --profile=perfmap module.wasm
perf report

# JitDump format
wasmtime --profile=jitdump module.wasm
```

### Custom Profiling

**Timing specific functions**:

```rust
use std::time::Instant;

pub fn timed_function() {
    let start = Instant::now();

    // Do work
    expensive_computation();

    let duration = start.elapsed();
    log(&format!("Function took {:?}", duration));
}
```

**Sampling profiler** (import from host):

```javascript
const samples = new Map();

const importObject = {
  env: {
    profile_sample: (functionId) => {
      samples.set(functionId, (samples.get(functionId) || 0) + 1);
    }
  }
};
```

## Memory Debugging

### Visualizing Memory Layout

```javascript
const memory = instance.exports.memory;
const buffer = memory.buffer;

// View as different types
const u8 = new Uint8Array(buffer);
const u32 = new Uint32Array(buffer);
const f64 = new Float64Array(buffer);

console.log('First 64 bytes (u8):', u8.slice(0, 64));
console.log('First 16 words (u32):', u32.slice(0, 16));
```

### Heap Corruption Detection

**Rust**: Use debug allocator

```rust
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

**C/C++**: Enable SAFE_HEAP in Emscripten

```bash
emcc -s SAFE_HEAP=1 module.c
```

This adds checks for:
- Out-of-bounds memory access
- Use-after-free
- Double-free

### Memory Leak Detection

Track allocations:

```javascript
let allocations = 0;

const importObject = {
  env: {
    malloc: (size) => {
      allocations++;
      return instance.exports.__malloc(size);
    },
    free: (ptr) => {
      allocations--;
      instance.exports.__free(ptr);
    }
  }
};

// After operations, check allocations
console.log('Outstanding allocations:', allocations);
```

## Debugging Strategies

### Binary Search for Bugs

1. Comment out half the code
2. If bug persists, it's in remaining half
3. If bug disappears, it's in commented half
4. Repeat until narrowed down

### Differential Testing

Compare WASM output with native:

```bash
# Compile native
gcc -o native program.c
./native > native_output.txt

# Compile WASM
emcc -o wasm.js program.c
node wasm.js > wasm_output.txt

# Compare
diff native_output.txt wasm_output.txt
```

### Minimal Reproduction

Reduce bug to smallest possible test case:

1. Remove unrelated code
2. Simplify inputs
3. Isolate failing function
4. Create standalone test

Easier to debug and report.

## Debugging Imports/Exports

### Missing Imports

**Error**: "LinkError: import object field 'X' is not a Function"

**Cause**: WASM expects import, but it's not provided or wrong type

**Solution**: Check import object structure:

```javascript
console.log('Module imports:', WebAssembly.Module.imports(module));

// Ensure imports match:
const importObject = {
  env: {
    X: /* correct implementation */
  }
};
```

### Type Mismatches

**Error**: "LinkError: imported function does not match signature"

**Cause**: Import signature doesn't match expected signature

**Solution**: Verify types match:

```bash
wasm-objdump -x module.wasm | grep "import"
```

Ensure JavaScript function signature matches.

## Debugging Tips

1. **Start small**: Test individual functions before integration
2. **Use assertions**: Validate assumptions with assert! or assert_eq!
3. **Log liberally**: Add logging at key points
4. **Check boundaries**: Validate array indices, pointer arithmetic
5. **Test in multiple browsers**: WASM implementations differ slightly
6. **Enable all debug features**: Debug builds, assertions, sanitizers
7. **Read the error message**: WASM errors are usually informative
8. **Check the docs**: Language-specific WASM quirks are documented
9. **Use the community**: Ask on forums, Discord, Stack Overflow
10. **Reproduce reliably**: Intermittent bugs are hardest; make them consistent

## Next Steps

Debugging is essential for productive WASM development. With proper tooling—source maps, browser DevTools, logging, profiling—you can diagnose issues effectively. Next, we'll explore testing strategies for WASM modules.
