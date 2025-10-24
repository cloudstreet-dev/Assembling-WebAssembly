# Chapter 17: Rust Fundamentals for WASM

## Why Rust for WebAssembly?

Rust has emerged as the premier language for WebAssembly development:

- **Memory safety without GC**: No runtime overhead, predictable performance
- **Zero-cost abstractions**: High-level code compiles to efficient WASM
- **First-class tooling**: wasm-pack, wasm-bindgen, cargo integration
- **Active ecosystem**: Libraries designed for WASM
- **Small output**: Optimized binaries are tiny
- **Official support**: `wasm32-unknown-unknown` is a tier 2 target

If you're starting a new WASM project, Rust should be your default choice unless you have specific reasons otherwise.

## Rust Compilation Targets for WASM

### wasm32-unknown-unknown

The most common target for web applications:

```bash
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown
```

**Characteristics**:
- No operating system
- No standard C library
- Minimal std library (no threading, no filesystem)
- Perfect for browser environments

**Use cases**: Web applications, browser-based tools

### wasm32-wasi

For applications needing system access:

```bash
rustup target add wasm32-wasi
cargo build --target wasm32-wasi
```

**Characteristics**:
- WASI system interface
- File I/O, environment variables, stdio
- More of std library available
- Runs in wasmtime, wasmer, etc.

**Use cases**: CLI tools, server-side WASM, edge computing

## Basic Setup

### Project Structure

```bash
cargo new --lib my-wasm-project
cd my-wasm-project
```

**Cargo.toml**:

```toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # Create dynamic library for WASM

[dependencies]
# Add dependencies here

[profile.release]
opt-level = "z"      # Optimize for size
lto = true           # Link-time optimization
codegen-units = 1    # Single codegen unit for better optimization
```

### Minimal Example

**src/lib.rs**:

```rust
// Disable standard library (not needed for wasm32-unknown-unknown)
#![no_std]

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

**Build**:

```bash
cargo build --target wasm32-unknown-unknown --release
```

**Output**: `target/wasm32-unknown-unknown/release/my_wasm_project.wasm`

### With Standard Library

For most projects, you'll use std:

```rust
// src/lib.rs
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

The `#[no_mangle]` attribute prevents Rust from mangling the function name, making it accessible from JavaScript.

## Memory Management

Rust's ownership system ensures memory safety in WASM:

```rust
#[no_mangle]
pub extern "C" fn process_data(len: usize) -> *mut u8 {
    let mut vec = Vec::with_capacity(len);
    vec.resize(len, 0);

    // Transfer ownership to caller
    let ptr = vec.as_mut_ptr();
    std::mem::forget(vec);  // Don't drop the Vec
    ptr
}

#[no_mangle]
pub extern "C" fn free_data(ptr: *mut u8, len: usize) {
    unsafe {
        // Reconstruct Vec to drop it properly
        let _ = Vec::from_raw_parts(ptr, len, len);
    }
}
```

**From JavaScript**:

```javascript
const instance = await WebAssembly.instantiateStreaming(fetch('module.wasm'));

const len = 1024;
const ptr = instance.exports.process_data(len);

// Access memory
const memory = new Uint8Array(instance.exports.memory.buffer, ptr, len);

// Use memory...

// Free when done
instance.exports.free_data(ptr, len);
```

## Exporting Memory

Export memory for host access:

```rust
use std::alloc::{alloc, dealloc, Layout};

// Global allocator setup
#[global_allocator]
static ALLOCATOR: std::alloc::System = std::alloc::System;

#[no_mangle]
pub extern "C" fn alloc_bytes(size: usize) -> *mut u8 {
    unsafe {
        let layout = Layout::from_size_align_unchecked(size, 1);
        alloc(layout)
    }
}

#[no_mangle]
pub extern "C" fn dealloc_bytes(ptr: *mut u8, size: usize) {
    unsafe {
        let layout = Layout::from_size_align_unchecked(size, 1);
        dealloc(ptr, layout);
    }
}
```

## String Handling

Passing strings between Rust and JavaScript:

### Rust to JavaScript

```rust
use std::ffi::CString;
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn get_greeting() -> *mut c_char {
    let s = CString::new("Hello from Rust!").unwrap();
    s.into_raw()  // Transfer ownership
}

#[no_mangle]
pub extern "C" fn free_string(ptr: *mut c_char) {
    unsafe {
        if !ptr.is_null() {
            let _ = CString::from_raw(ptr);  // Reconstruct and drop
        }
    }
}
```

**JavaScript**:

```javascript
function readCString(memory, ptr) {
    const view = new Uint8Array(memory.buffer);
    const end = view.indexOf(0, ptr);  // Find null terminator
    const bytes = view.slice(ptr, end);
    return new TextDecoder().decode(bytes);
}

const ptr = instance.exports.get_greeting();
const greeting = readCString(instance.exports.memory, ptr);
console.log(greeting);  // "Hello from Rust!"
instance.exports.free_string(ptr);
```

### JavaScript to Rust

```rust
use std::slice;
use std::str;

#[no_mangle]
pub extern "C" fn process_string(ptr: *const u8, len: usize) -> usize {
    let bytes = unsafe { slice::from_raw_parts(ptr, len) };

    match str::from_utf8(bytes) {
        Ok(s) => {
            // Process string
            s.len()
        }
        Err(_) => 0,
    }
}
```

**JavaScript**:

```javascript
function writeString(memory, str) {
    const encoder = new TextEncoder();
    const bytes = encoder.encode(str);

    // Allocate space
    const ptr = instance.exports.alloc_bytes(bytes.length);
    const view = new Uint8Array(instance.exports.memory.buffer);

    // Copy string to WASM memory
    view.set(bytes, ptr);

    return { ptr, len: bytes.length };
}

const { ptr, len } = writeString(instance.exports.memory, "Hello, Rust!");
const result = instance.exports.process_string(ptr, len);
instance.exports.dealloc_bytes(ptr, len);
```

## Error Handling

### Panics

By default, panics abort in WASM. You can configure panic behavior:

**Cargo.toml**:

```toml
[profile.release]
panic = "abort"  # Default for WASM
```

Or unwind (larger binary):

```toml
[profile.release]
panic = "unwind"
```

### Result Types

Return error codes to JavaScript:

```rust
#[no_mangle]
pub extern "C" fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        -1  // Error code
    } else {
        a / b
    }
}
```

Or use sentinel values:

```rust
#[no_mangle]
pub extern "C" fn safe_divide(a: i32, b: i32, result: *mut i32) -> bool {
    if b == 0 {
        false  // Indicates error
    } else {
        unsafe {
            *result = a / b;
        }
        true  // Success
    }
}
```

## Using External Crates

Many Rust crates work in WASM with feature flags:

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"], default-features = false }
serde_json = { version = "1.0", default-features = false, features = ["alloc"] }
```

**Code**:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Config {
    name: String,
    value: i32,
}

#[no_mangle]
pub extern "C" fn parse_config(ptr: *const u8, len: usize) -> i32 {
    let bytes = unsafe { std::slice::from_raw_parts(ptr, len) };

    match serde_json::from_slice::<Config>(bytes) {
        Ok(config) => config.value,
        Err(_) => -1,
    }
}
```

## Optimization Techniques

### Size Optimization

**Cargo.toml**:

```toml
[profile.release]
opt-level = "z"          # Optimize for size
lto = true               # Link-time optimization
codegen-units = 1        # Better optimization
strip = true             # Strip symbols (Rust 1.59+)
panic = "abort"          # Smaller panic handling
```

**Additional tools**:

```bash
# wasm-opt from binaryen
wasm-opt -Oz -o optimized.wasm input.wasm

# wasm-snip: Remove unused functions
wasm-snip input.wasm -o snipped.wasm
```

### Performance Optimization

**Cargo.toml**:

```toml
[profile.release]
opt-level = 3            # Maximum optimization
lto = "fat"              # Aggressive LTO
codegen-units = 1
```

**Code-level optimizations**:

```rust
// Inline critical functions
#[inline(always)]
pub fn hot_function() {
    // ...
}

// Use iterators (they optimize well)
pub fn sum_array(arr: &[i32]) -> i32 {
    arr.iter().sum()  // Better than manual loop
}

// Avoid bounds checks when safe
pub fn process_slice(arr: &[i32]) {
    if arr.len() >= 10 {
        // Compiler knows these won't panic
        let _ = arr[0];
        let _ = arr[9];
    }
}
```

## Testing Rust WASM

### Unit Tests (Native)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

Run: `cargo test`

### Conditional Compilation

```rust
#[cfg(target_arch = "wasm32")]
fn wasm_specific_function() {
    // WASM-only code
}

#[cfg(not(target_arch = "wasm32"))]
fn native_specific_function() {
    // Native-only code
}
```

## Advanced Patterns

### Lazy Static Initialization

```rust
use std::sync::Once;

static INIT: Once = Once::new();
static mut DATA: Option<Vec<i32>> = None;

fn get_data() -> &'static Vec<i32> {
    unsafe {
        INIT.call_once(|| {
            DATA = Some(vec![1, 2, 3, 4, 5]);
        });
        DATA.as_ref().unwrap()
    }
}
```

### Generic Functions

```rust
fn process<T: std::fmt::Display>(value: T) -> String {
    format!("Value: {}", value)
}

#[no_mangle]
pub extern "C" fn process_i32(value: i32) -> *mut u8 {
    let result = process(value);
    let bytes = result.into_bytes();
    let ptr = bytes.as_ptr() as *mut u8;
    std::mem::forget(bytes);
    ptr
}
```

## Common Pitfalls

### 1. Name Mangling

**Problem**: Rust mangles function names

**Solution**: Use `#[no_mangle]`

### 2. Memory Leaks

**Problem**: Forgetting to free allocated memory

**Solution**: Provide explicit free functions

### 3. Stack Overflow

**Problem**: Deep recursion or large stack allocations

**Solution**: Use heap allocation, increase stack size

### 4. ABI Incompatibility

**Problem**: Rust types don't match C ABI

**Solution**: Use `extern "C"` and C-compatible types

### 5. Floating-Point Non-Determinism

**Problem**: NaN bit patterns vary

**Solution**: Normalize NaN values if determinism is needed

## Next Steps

We've covered Rust fundamentals for WASM—basic compilation, memory management, string handling, and optimization. In the next chapter, we'll explore advanced Rust WASM with wasm-bindgen and wasm-pack, which dramatically simplify JavaScript interop and enable rich web applications.
