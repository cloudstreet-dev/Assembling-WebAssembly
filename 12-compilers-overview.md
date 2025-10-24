# Chapter 12: Compilers Overview

## The Compilation Landscape

WebAssembly is a compilation target, not a source language. Dozens of languages can compile to WASM, each using different toolchains with different tradeoffs. Understanding the compilation landscape helps you choose the right tool for your project.

## Compiler Architectures

### Three-Stage Architecture

Most compilers follow a three-stage architecture:

**Frontend** → **Optimizer** → **Backend**

1. **Frontend**: Parse source language, generate intermediate representation (IR)
2. **Optimizer**: Transform IR for performance and size
3. **Backend**: Generate target code (WASM bytecode)

Examples:
- **Rust**: rustc → LLVM IR → LLVM → WASM
- **C/C++**: clang → LLVM IR → LLVM → WASM
- **Go**: Go compiler → Go IR → wasm backend

### LLVM-Based Toolchains

LLVM has become the dominant infrastructure for WASM compilation:

```
Source → Frontend → LLVM IR → LLVM Optimizer → WASM Backend → .wasm
```

Languages using LLVM for WASM:
- **Rust** (rustc uses LLVM)
- **C/C++** (clang)
- **Swift** (Swift compiler uses LLVM)
- **Zig** (self-hosted, but can use LLVM)
- **Julia** (LLVM-based)

**Advantages**:
- Mature, well-optimized IR
- Excellent optimization passes
- Shared improvements benefit all languages
- Good WASM backend

**Disadvantages**:
- Large dependency (LLVM is huge)
- Compilation can be slow
- May not fit language semantics perfectly (e.g., Go's goroutines)

### Language-Specific Backends

Some compilers have custom WASM backends:

**Go**: Native Go compiler with WASM target
- Direct from Go AST to WASM
- Better integration with Go runtime
- But: larger output, some limitations

**AssemblyScript**: Custom compiler, TypeScript-like syntax
- Designed specifically for WASM
- No JavaScript runtime overhead
- Tight, efficient output

**TinyGo**: Alternative Go compiler using LLVM
- Smaller output than official Go
- Better WASM support
- Tradeoff: not 100% Go compatible

### JIT/AOT Compilation

**Ahead-of-Time (AOT)**: Compile to WASM during build
- Most common
- Optimized for size and speed
- No runtime compilation cost

**Just-in-Time (JIT)**: Compile WASM to machine code at runtime
- What engines do (V8, SpiderMonkey, etc.)
- Fast startup (streaming compilation)
- Can optimize based on runtime profile

## Major Toolchains

### Emscripten (C/C++)

The original and most mature WASM toolchain.

**What it does**:
- Compiles C/C++ to WASM via LLVM
- Provides POSIX-like APIs
- Includes libc, C++ STL
- Generates JavaScript glue code

**Architecture**:
```
C/C++ → clang → LLVM IR → LLVM → WASM
                              ↓
                         JavaScript glue
```

**Strengths**:
- Extremely mature
- Large ecosystem of ported libraries (SDL, OpenGL, etc.)
- Comprehensive documentation
- Good optimization

**Weaknesses**:
- JavaScript dependency (glue code)
- Larger output than raw LLVM
- Complex build system for large projects

**Use cases**:
- Porting existing C/C++ code to web
- Graphics/games (Unity, Unreal)
- Scientific computing libraries

**Example**:
```bash
emcc hello.c -o hello.html  # Generates .wasm + .js + .html
emcc hello.c -o hello.js    # Generates .wasm + .js
```

### wasm32-unknown-unknown (Direct LLVM)

Clang can target WASM directly without Emscripten:

```bash
clang --target=wasm32-unknown-unknown -o module.wasm main.c
```

**Strengths**:
- No JavaScript glue code
- Minimal output
- Full control
- Good for WASI targets

**Weaknesses**:
- No libc by default (bring your own)
- Manual setup for system APIs
- Less beginner-friendly

**Use cases**:
- Small, standalone modules
- WASI applications
- Embedded systems
- When you want minimal dependencies

### Rust (rustc + wasm-bindgen)

Rust's official compiler has excellent WASM support:

```rust
// Compile to WASM
rustc --target wasm32-unknown-unknown main.rs
```

**wasm-bindgen**: Tool for Rust ↔ JavaScript interop

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Generates:
- WASM module
- JavaScript bindings
- TypeScript definitions

**Strengths**:
- Memory safety carries to WASM
- Excellent tooling (wasm-pack, wasm-bindgen)
- Growing ecosystem
- Zero-cost abstractions

**Weaknesses**:
- Learning curve for Rust
- Compilation can be slow
- Output size can be large without optimization

**Use cases**:
- New projects where memory safety matters
- Performance-critical web apps
- Safe systems programming

### AssemblyScript

TypeScript-like language that compiles directly to WASM:

```typescript
export function add(a: i32, b: i32): i32 {
  return a + b;
}
```

```bash
asc module.ts --outFile module.wasm
```

**Strengths**:
- Familiar syntax for JavaScript/TypeScript developers
- Designed for WASM (no runtime baggage)
- Fast compilation
- Small output

**Weaknesses**:
- Not actually TypeScript (subtle differences)
- Smaller ecosystem than Rust/C++
- Manual memory management

**Use cases**:
- Developers comfortable with TypeScript
- Greenfield WASM projects
- When JS interop is important

### Go (Official and TinyGo)

**Official Go**:

```bash
GOOS=js GOARCH=wasm go build -o main.wasm
```

**Strengths**:
- Full Go language support
- Goroutines work
- Standard library available

**Weaknesses**:
- Large output (2-3 MB minimum)
- Requires JavaScript glue code
- GC overhead

**TinyGo**:

```bash
tinygo build -target wasm -o main.wasm
```

**Strengths**:
- Much smaller output
- Still supports goroutines
- Better WASI support

**Weaknesses**:
- Not all Go features supported
- Smaller standard library
- Newer, less mature

**Use cases**:
- Porting Go code to web/WASI
- Serverless functions (TinyGo)
- Embedded systems (TinyGo)

### .NET (Blazor)

C# compilation to WASM via Blazor:

```bash
dotnet new blazorwasm -o MyApp
dotnet build
```

**Strengths**:
- Full .NET framework
- Mature ecosystem
- Good tooling (Visual Studio)
- Razor components for UI

**Weaknesses**:
- Large download size (but improving)
- Startup time
- Limited to web (not WASI yet)

**Use cases**:
- Enterprise web applications
- Teams already using C#/.NET
- Full-stack .NET development

## Optimization Flags

### Size Optimization

**LLVM-based (Rust, C++)**:
```bash
-O z          # Optimize for size
-O s          # Optimize for size, less aggressive
--strip-all   # Remove debug symbols
```

**Rust**:
```toml
[profile.release]
opt-level = "z"  # Optimize for size
lto = true       # Link-time optimization
```

**Emscripten**:
```bash
emcc -Oz           # Maximum size optimization
emcc --closure 1   # Closure compiler on JS glue
```

### Speed Optimization

```bash
-O 2          # Standard optimization
-O 3          # Aggressive optimization
```

### Debug Builds

```bash
-g            # Include debug info
-g4           # Full debug info (DWARF)
```

## Binary Size Comparison

For a simple "Hello World":

| Toolchain | Unoptimized | Optimized (-Oz) |
|-----------|-------------|-----------------|
| Emscripten | ~15 KB | ~1 KB |
| Rust | ~200 KB | ~300 bytes |
| TinyGo | ~50 KB | ~5 KB |
| Go | ~2 MB | ~1.8 MB |
| AssemblyScript | ~2 KB | ~300 bytes |

*Note: Real apps are larger, but relative sizes hold*

## Cross-Compilation Considerations

### Target Triples

WASM targets use target triples:

- `wasm32-unknown-unknown` - No OS, no standard library
- `wasm32-unknown-emscripten` - Emscripten environment
- `wasm32-wasi` - WASI system interface

Example in Rust:

```bash
rustup target add wasm32-unknown-unknown
rustup target add wasm32-wasi

cargo build --target wasm32-unknown-unknown  # Minimal
cargo build --target wasm32-wasi             # WASI support
```

### System Dependencies

Different targets provide different APIs:

**unknown-unknown**: Nothing. You provide everything.

**wasi**: File I/O, environment variables, sockets (proposal).

**emscripten**: POSIX APIs, OpenGL, SDL, etc.

Choose based on what you need.

## Toolchain Selection Guide

| Use Case | Recommended | Alternative |
|----------|-------------|-------------|
| Porting C/C++ to web | Emscripten | wasm32-unknown |
| New, safe web project | Rust + wasm-bindgen | AssemblyScript |
| TypeScript familiarity | AssemblyScript | Rust |
| Enterprise .NET team | Blazor | - |
| Go code to web | TinyGo | Official Go |
| CLI/server tools | Rust (wasi) | TinyGo (wasi) |
| Minimal size | Rust (-Oz) | AssemblyScript |
| Fastest compilation | AssemblyScript | Go |

## Debugging and Profiling

### Source Maps

Most toolchains support source maps:

```bash
# Rust
cargo build --target wasm32-unknown-unknown --release

# Emscripten
emcc -g4 -o output.js input.c

# AssemblyScript
asc --sourceMap input.ts
```

Source maps let debuggers show original source, not WASM bytecode.

### DWARF Debug Info

LLVM-based toolchains embed DWARF:

```bash
clang -g -target wasm32-unknown-unknown input.c
```

Debuggers can use this for:
- Breakpoints in source
- Variable inspection
- Stack traces

## Future of WASM Compilation

### Emerging Toolchains

- **Zig**: Excellent C/C++ interop, compiles to WASM
- **Kotlin**: Kotlin/Wasm in preview
- **Python**: Pyodide (Python in browser via WASM)
- **Swift**: SwiftWasm making progress

### Proposals Affecting Compilation

**GC proposal**: Better support for managed languages (Java, C#, Scheme)

**Exception handling**: Native exceptions instead of setjmp/longjmp

**Tail calls**: Efficient recursion for functional languages

**Component model**: Better module composition

These will improve compilation for many languages.

## Next Steps

Understanding the compiler landscape helps you choose the right tool. In the next chapter, we'll explore WASM runtimes—the engines that execute WASM code in different environments.
