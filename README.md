# Assembling WebAssembly

A comprehensive guide to building WebAssembly from multiple programming languages. This book is designed for advanced developers who want to master WASM across web, server, embedded, and edge computing contexts.

## About This Book

This book assumes you already know the programming languages we'll be using. Our focus is on teaching you WebAssembly itself—how to compile to it, optimize for it, and deploy it effectively across different environments.

**Status**: Complete - All 52 chapters written, covering foundations through advanced topics to real-world practice projects.

## Table of Contents

### Part 1: Foundations

1. [Introduction](01-introduction.md) - Why WASM exists, use cases, and the problems it solves
2. [Architecture](02-architecture.md) - Stack machine model, instruction set architecture, specification overview
3. [Text Format](03-text-format.md) - Deep dive into WAT (WebAssembly Text format) syntax
4. [Binary Format](04-binary-format.md) - Understanding WASM bytecode structure and encoding
5. [Type System](05-type-system.md) - Value types, function signatures, and type validation
6. [Memory Model](06-memory-model.md) - Linear memory, pages, limits, and memory operations
7. [Module Anatomy](07-module-anatomy.md) - Module sections, structure, and validation rules

### Part 2: Runtime & Execution

8. [Instantiation](08-instantiation.md) - Module loading, compilation, and linking process
9. [Imports and Exports](09-imports-exports.md) - Interface patterns and module composition
10. [Tables](10-tables.md) - Indirect calls, function references, and dynamic dispatch
11. [Security Model](11-security-model.md) - Sandboxing, capability-based security, and isolation

### Part 3: Toolchain & Ecosystem

12. [Compilers Overview](12-compilers-overview.md) - Landscape of WASM compilation targets
13. [Runtimes](13-runtimes.md) - wasmtime, wasmer, wasm3, and browser engines
14. [WASI](14-wasi.md) - WebAssembly System Interface specification
15. [Debugging](15-debugging.md) - Tools, source maps, profiling, and troubleshooting
16. [Testing](16-testing.md) - Unit testing WASM modules and integration testing strategies

### Part 4: Language Deep Dives

17. [Rust Fundamentals](17-rust-fundamentals.md) - Using rustc with wasm32 target, basics
18. [Rust Advanced](18-rust-advanced.md) - wasm-bindgen, wasm-pack, and optimization techniques
19. [Go Official](19-go-official.md) - Go's official WASM support and its limitations
20. [TinyGo](20-tinygo.md) - Better Go WASM compilation and tradeoffs
21. [C/C++](21-c-cpp.md) - Emscripten ecosystem and toolchain
22. [Clang WASM](22-clang-wasm.md) - Direct LLVM-based compilation without Emscripten
23. [AssemblyScript](23-assemblyscript.md) - TypeScript-like language designed for WASM
24. [.NET and Blazor](24-dotnet-blazor.md) - C# and the .NET approach to WebAssembly

### Part 5: Language Survey

25. [Zig](25-zig.md) - Modern systems language with excellent WASM support
26. [Swift](26-swift.md) - SwiftWasm project and Swift's WASM capabilities
27. [Kotlin](27-kotlin.md) - Kotlin/Wasm and the JVM ecosystem approach
28. [Python](28-python-wasm.md) - Pyodide, limitations, and Python in the browser
29. [Experimental Languages](29-experimental-languages.md) - Grain, Moonbit, and emerging WASM-first languages
30. [Barely There](30-barely-there.md) - Languages with minimal or experimental WASM support

### Part 6: Web Integration

31. [Browser APIs](31-browser-apis.md) - The WebAssembly JavaScript API
32. [JavaScript Interop](32-js-interop.md) - Calling conventions and data marshalling
33. [DOM Access](33-dom-access.md) - Patterns for DOM manipulation from WASM
34. [Web Workers](34-web-workers.md) - Threading and parallel execution in browsers
35. [Streaming](35-streaming.md) - Streaming compilation and instantiation

### Part 7: Server & CLI

36. [WASI Deep Dive](36-wasi-deep-dive.md) - Filesystem, networking, and system interfaces
37. [CLI Tools](37-cli-tools.md) - Building command-line tools with WASM
38. [Server-Side](38-server-side.md) - Fastly Compute, Cloudflare Workers, and server runtimes
39. [Plugin Systems](39-plugin-systems.md) - Extending applications with WASM plugins

### Part 8: Embedded & Edge

40. [Embedded Runtimes](40-embedded-runtimes.md) - wasm3, WAMR for IoT and microcontrollers
41. [Edge Computing](41-edge-computing.md) - CDN edge, distributed computing patterns
42. [Resource Constraints](42-resource-constraints.md) - Optimizing for memory and CPU-constrained targets

### Part 9: Advanced Topics

43. [SIMD](43-simd.md) - Vector instructions and parallel data processing with v128 types
44. [Threads](44-threads.md) - Shared memory threading model and atomics
45. [Component Model](45-component-model.md) - Next-generation composition with WIT interfaces
46. [Garbage Collection](46-garbage-collection.md) - GC proposal for managed languages (Java, Kotlin, Dart)
47. [Exception Handling](47-exceptions.md) - First-class exceptions with try/catch/throw
48. [Tail Calls](48-tail-calls.md) - Tail call optimization for functional programming patterns

### Part 10: Practice

49. [Building an Image Processor](49-building-image-processor.md) - Complete image processing app with SIMD and Web Workers
50. [Building a Game Engine](50-building-game-engine.md) - 2D platformer with ECS architecture
51. [Cryptography and Data Processing](51-crypto-mining.md) - SHA-256, PBKDF2, AES-256 implementation
52. [Real-Time Audio Synthesis](52-real-time-audio.md) - Polyphonic synthesizer with AudioWorklet

### Appendices

- [Glossary](glossary.md) - WASM terminology reference
- [Contributing](CONTRIBUTING.md) - Guidelines for contributing to this book

## License

See [LICENSE](LICENSE) for details.

## How to Read This Book

- **Sequential reading**: Parts 1-3 build foundational knowledge. Read these in order.
- **Language chapters**: Pick the languages relevant to your work (Parts 4-5).
- **Context-based reading**: Choose Part 6 (Web), Part 7 (Server), or Part 8 (Embedded) based on your deployment target.
- **Advanced readers**: Parts 9-10 can be read independently once you understand the foundations.

## Prerequisites

This book assumes:
- Strong programming background in at least one systems or application language
- Familiarity with concepts like memory management, types, and compilation
- Basic understanding of how code executes on hardware
- No prior WASM knowledge required
