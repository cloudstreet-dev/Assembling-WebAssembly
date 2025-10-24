# Glossary

A comprehensive reference of WebAssembly terms, concepts, and abbreviations used throughout "Assembling WebAssembly".

## A

**ABI (Application Binary Interface)**
The low-level interface between compiled code modules, defining calling conventions, data types, and memory layout. WebAssembly doesn't mandate a specific ABI, allowing flexibility across languages.

**ADSR (Attack, Decay, Sustain, Release)**
An envelope generator pattern used in audio synthesis to shape sound over time. Defines how a sound starts, evolves, and ends.

**AES (Advanced Encryption Standard)**
A symmetric encryption algorithm widely used for securing data. Commonly implemented in WebAssembly for performance-critical cryptography.

**Alignment**
The memory address boundary requirements for load/store operations. For example, `i32.load` requires 4-byte alignment (addresses divisible by 4).

**AMDGPU**
Graphics processing unit architecture by AMD. WebAssembly SIMD operations can map to AMDGPU vector instructions.

**AOT (Ahead-Of-Time Compilation)**
Compilation that happens before runtime. WebAssembly can be AOT-compiled to native machine code for faster startup compared to JIT compilation.

**API (Application Programming Interface)**
A set of functions and types exposed by a module for other code to use. WebAssembly modules expose APIs through exports.

**AssemblyScript**
A TypeScript-like language that compiles to WebAssembly. Designed specifically for WebAssembly with a syntax familiar to JavaScript/TypeScript developers.

**Async/Await**
JavaScript pattern for handling asynchronous operations. Often used when loading and instantiating WebAssembly modules.

**Atomic Operations**
Memory operations that execute as a single, indivisible unit. Essential for thread-safe programming with shared memory. WebAssembly provides atomic load, store, and read-modify-write operations.

**AudioWorklet**
A modern Web Audio API for processing audio in a separate thread with low latency. Ideal for WebAssembly audio processing.

## B

**Big-Endian**
A byte ordering where the most significant byte comes first. WebAssembly uses little-endian exclusively.

**Binary Format**
The actual `.wasm` file format, a compact binary encoding of WebAssembly modules. More efficient than text format for transmission and parsing.

**Binaryen**
A compiler infrastructure for WebAssembly, providing optimization passes and tooling. Used by Emscripten and other toolchains.

**Biquad Filter**
A second-order IIR (Infinite Impulse Response) digital filter used in audio processing. Commonly implemented for low-pass, high-pass, and band-pass filtering.

**Blazor**
Microsoft's framework for building web applications using C# and .NET, compiled to WebAssembly via Mono.

**Block**
A structured control flow construct that groups instructions and provides a branch target. Can have a result type.

**Branch**
A control flow operation that jumps to a labeled block. WebAssembly supports conditional (`br_if`) and unconditional (`br`) branches.

**Browser Support**
The availability of WebAssembly features across different browsers. Core WebAssembly has universal support; newer features vary.

**Buffer**
A region of memory used for data storage. In WebAssembly context, often refers to ArrayBuffer or SharedArrayBuffer.

## C

**Call Stack**
The runtime structure tracking function calls and local variables. WebAssembly manages this internally, separate from linear memory.

**Canonical ABI**
The standard interface for the Component Model, defining how high-level types map to WebAssembly core types.

**Canvas**
HTML element for drawing graphics. Often used as the rendering target for WebAssembly graphics applications.

**CDN (Content Delivery Network)**
A distributed network of servers for delivering web content. WASM files are often served via CDN for performance.

**CGO**
Go's mechanism for calling C code. Relevant when discussing Go's WebAssembly implementation limitations.

**Chorus Effect**
An audio effect that simulates multiple voices by modulating delayed copies of a signal. Often implemented in WebAssembly for performance.

**Clang**
LLVM's C/C++ compiler, used to compile C/C++ to WebAssembly via the LLVM backend.

**Cloudflare Workers**
A serverless platform that supports WebAssembly, allowing code to run at the edge close to users.

**Code Section**
A section of a WebAssembly module containing function bodies (implementation code).

**Component Model**
A WebAssembly standard for composable, portable modules with rich interfaces defined via WIT (WebAssembly Interface Types).

**Continuation-Passing Style (CPS)**
A programming style where control flow is explicit via function continuations. Enabled by tail call optimization.

**Control Flow**
The order in which instructions execute. WebAssembly uses structured control flow (blocks, loops, if) rather than arbitrary goto.

**CORS (Cross-Origin Resource Sharing)**
Browser security policy restricting cross-origin requests. Affects loading WebAssembly modules from different domains.

**Core Specification**
The fundamental WebAssembly specification (MVP + standardized extensions), as opposed to proposals in progress.

**Cranelift**
A code generator used in wasmtime and other runtimes for fast, secure compilation of WebAssembly to native code.

**Cryptography**
The practice of secure communication. WebAssembly is often used for crypto operations like hashing, encryption, and signature verification due to performance requirements.

**CSP (Content Security Policy)**
Browser security mechanism that can restrict WebAssembly execution. Requires proper configuration for WASM applications.

## D

**Data Segment**
A section of a WebAssembly module containing initial data for linear memory, loaded at instantiation time.

**Data URL**
A URL scheme (`data:`) that embeds data inline. Can be used to embed small WASM modules directly in HTML/JavaScript.

**Denormal Numbers**
Floating-point values very close to zero. Can cause performance issues in audio processing; often flushed to zero.

**Deno**
A JavaScript/TypeScript runtime with WebAssembly support, offering a secure sandbox for WASM execution.

**Digital Signal Processing (DSP)**
Mathematical manipulation of signals, commonly audio. WebAssembly excels at DSP due to performance requirements.

**Direct Calls**
Function calls where the target is known at compile time, as opposed to indirect calls through a table.

**DOM (Document Object Model)**
Browser API for manipulating HTML/CSS. WebAssembly can access the DOM via JavaScript glue code or web-sys bindings.

**Dynamic Linking**
Loading and linking code modules at runtime. WebAssembly supports dynamic linking via imports/exports.

## E

**ECS (Entity Component System)**
An architectural pattern for game engines separating data (components) from behavior (systems). Well-suited to WebAssembly game development.

**Edge Computing**
Running code close to end users rather than in centralized data centers. WebAssembly is popular for edge deployments.

**Element Section**
A section of a WebAssembly module initializing function tables with function references.

**Emscripten**
A complete toolchain for compiling C/C++ to WebAssembly, including a comprehensive libc implementation and JavaScript glue code.

**Envelope**
In audio synthesis, a time-varying modifier that shapes sound characteristics. ADSR envelopes are common.

**ESP32**
A popular microcontroller with WiFi/Bluetooth. WebAssembly (via WAMR) can run on ESP32 for IoT applications.

**Exception Handling**
The WebAssembly proposal (now standardized) for first-class exceptions with try/catch/throw semantics.

**Export**
A function, memory, table, or global made available to the host environment or other modules. Listed in the export section.

**Export Section**
A section of a WebAssembly module listing exported entities and their names.

**Expression**
In WebAssembly, a sequence of instructions that produces a typed value on the stack.

**Externref**
A WebAssembly reference type for holding host references (e.g., JavaScript objects). Opaque to WebAssembly code.

## F

**Fastly Compute**
An edge computing platform with WebAssembly support, using the Compute runtime for running WASM at the edge.

**FFI (Foreign Function Interface)**
A mechanism for calling functions written in another language. WebAssembly imports/exports serve as FFI boundaries.

**FFT (Fast Fourier Transform)**
An algorithm for frequency domain analysis. Commonly implemented in WebAssembly for signal processing.

**FIFO (First-In, First-Out)**
A queue data structure. Often used in WebAssembly audio processing buffers.

**Filter (Audio)**
A process that removes or enhances certain frequencies. Digital filters (low-pass, high-pass, etc.) are common in WebAssembly audio applications.

**Floating-Point**
Number representation supporting fractional values. WebAssembly has f32 (32-bit) and f64 (64-bit) types.

**Function Index**
The numeric identifier for a function within a module, used by call instructions and table elements.

**Function Section**
A section of a WebAssembly module declaring function signatures (types).

**Function Signature**
The type of a function, specifying parameter types and return types.

**Function Table**
See "Table".

**Funcref**
A WebAssembly reference type for function references, used in tables for indirect calls.

## G

**Garbage Collection (GC)**
Automatic memory management. WebAssembly GC proposal adds struct/array types with automatic memory management, enabling languages like Java, Kotlin, and Dart.

**GC Proposal**
The WebAssembly garbage collection proposal, now standardized (Phase 4), adding managed memory types.

**Global**
A mutable or immutable variable stored outside linear memory, accessible by index. Can be imported or exported.

**Global Section**
A section of a WebAssembly module declaring global variables.

**Glue Code**
JavaScript code that bridges WebAssembly and JavaScript, handling type conversions and API differences. Generated by tools like Emscripten or wasm-bindgen.

**Go**
A programming language with WebAssembly support. Standard Go's WASM output is large; TinyGo produces smaller binaries.

**GPU (Graphics Processing Unit)**
Hardware accelerator for graphics and parallel computation. WebGPU enables WebAssembly to leverage GPU compute.

## H

**Hash Function**
A function mapping data to fixed-size values. Cryptographic hashes (SHA-256, etc.) are often implemented in WebAssembly.

**Heap**
Dynamic memory region for allocations. In WebAssembly, allocations come from linear memory managed by the runtime or allocator.

**Host**
The environment running WebAssembly modules (browser, Node.js, wasmtime, etc.). The host provides imports and manages execution.

**Host Bindings**
Language-specific wrappers for calling host (JavaScript) APIs from WebAssembly. Examples: wasm-bindgen (Rust), Emscripten (C/C++).

**Host Function**
A function provided by the host environment and imported into a WebAssembly module.

## I

**i32, i64**
WebAssembly's integer types: 32-bit and 64-bit signed/unsigned integers. The `i` stands for "integer".

**i31ref**
A WebAssembly GC proposal type representing a 31-bit integer value as a reference type.

**IEEE 754**
The standard for floating-point arithmetic. WebAssembly f32 and f64 follow IEEE 754.

**IIR Filter (Infinite Impulse Response)**
A type of digital filter using feedback. Biquad filters are common IIR implementations in audio DSP.

**Import**
A function, memory, table, or global provided by the host or another module. Declared in the import section.

**Import Object**
JavaScript object passed to `WebAssembly.instantiate()` providing imported functions, memories, tables, and globals.

**Import Section**
A section of a WebAssembly module declaring required imports.

**Indirect Call**
A function call through a table, where the target function is determined at runtime via an index.

**Inline Assembly**
Assembly code embedded in higher-level source code. Some languages allow inline WAT in WASM targeting.

**Instantiation**
The process of creating a runnable instance of a WebAssembly module, resolving imports and initializing memory/tables.

**Instruction**
The basic unit of WebAssembly code, like `i32.add`, `local.get`, or `call`.

**Integer**
Whole number type. WebAssembly has i32 and i64 integer types.

**Interface Types**
See "Component Model" and "WIT".

**Interoperability**
The ability for different languages/modules to work together. WebAssembly enables interop between languages and with JavaScript.

## J

**JavaScript API**
The JavaScript interface for loading, compiling, instantiating, and interacting with WebAssembly modules.

**JIT (Just-In-Time Compilation)**
Compilation that happens at runtime. Browsers typically JIT-compile WebAssembly for optimal performance.

**JSPI (JavaScript Promise Integration)**
A WebAssembly proposal for suspending and resuming execution around JavaScript promises. Enables natural async/await in WASM.

## K

**Kotlin**
A JVM language with WebAssembly support via Kotlin/Wasm, leveraging the GC proposal for memory management.

## L

**Label**
A name given to a block, loop, or if construct, used as a branch target.

**Latency**
The delay between input and output. Critical in real-time applications like audio processing, where WebAssembly excels due to predictable performance.

**LFO (Low-Frequency Oscillator)**
An oscillator used for modulation in audio synthesis (e.g., vibrato, tremolo, chorus). Typically < 20 Hz.

**Linear Memory**
A contiguous, mutable array of bytes accessible to WebAssembly modules. The primary mechanism for storing data.

**Little-Endian**
A byte ordering where the least significant byte comes first. WebAssembly memory is always little-endian.

**LLVM**
A compiler infrastructure used by many languages to target WebAssembly (Rust, C/C++, Swift, etc.).

**Load**
An instruction that reads from linear memory, like `i32.load` or `f64.load`.

**Local**
A function-scoped variable, including parameters. Accessed by index via `local.get`, `local.set`, and `local.tee`.

**Lock-Free**
Algorithms that guarantee progress without traditional locks. WebAssembly atomics enable lock-free data structures.

**Loop**
A control flow construct for iteration. Can be branched to from within for repetition.

**LTO (Link-Time Optimization)**
Optimization across compilation units at link time. Enabled in Rust/C++ for smaller, faster WebAssembly binaries.

## M

**Magic Number**
The first four bytes of a WASM file: `0x00 0x61 0x73 0x6d` (ASCII: "\0asm"). Identifies the file as WebAssembly.

**Memory**
See "Linear Memory".

**Memory Section**
A section of a WebAssembly module declaring memory requirements (initial and optional maximum size).

**Memory64**
A WebAssembly proposal extending linear memory addressing from 32-bit to 64-bit, enabling > 4GB memory.

**MIDI (Musical Instrument Digital Interface)**
A protocol for electronic musical instruments. WebAssembly synthesizers often support MIDI input via Web MIDI API.

**Mono**
The .NET runtime used by Blazor to run C# in WebAssembly.

**Module**
A WebAssembly program unit, containing code, data, and declarations. The `.wasm` file is a compiled module.

**Multi-Memory**
A WebAssembly proposal allowing multiple linear memories per module.

**Multi-Value**
A WebAssembly feature allowing functions and blocks to return multiple values. Now part of the core spec.

**Mutual Recursion**
Functions calling each other recursively. Enabled efficiently by tail calls.

**MVP (Minimum Viable Product)**
The initial WebAssembly specification released in 2017. Now supplemented by many standardized extensions.

## N

**NaN (Not a Number)**
A special floating-point value representing undefined or unrepresentable results (e.g., 0/0). WebAssembly follows IEEE 754 NaN semantics.

**Native Module**
In Node.js, a compiled addon (e.g., WebAssembly) that can be required like JavaScript modules.

**Node.js**
A JavaScript runtime with WebAssembly support, enabling WASM on servers and in command-line tools.

**Non-Determinism**
Behavior that can vary across executions. WebAssembly minimizes non-determinism but allows it in specific cases (e.g., NaN canonicalization).

**Numeric Types**
WebAssembly's value types: i32, i64, f32, f64, and vector types (v128).

## O

**Opcode**
A binary code representing an instruction. Each WebAssembly instruction has a unique opcode (e.g., 0x6A for `i32.add`).

**Optimization**
Transformations improving code performance or size. Tools like wasm-opt provide aggressive optimization passes.

**Oscillator**
In audio synthesis, a component generating periodic waveforms (sine, square, sawtooth, etc.).

**OTA (Over-The-Air)**
Wireless software updates. WebAssembly enables OTA updates for embedded devices via downloadable WASM modules.

## P

**Page (Memory)**
A 64KB unit of linear memory. Memory sizes are specified in pages.

**Parameter**
A function input value. WebAssembly functions can have multiple typed parameters.

**PBKDF2 (Password-Based Key Derivation Function 2)**
A key derivation function used for password hashing. Often implemented in WebAssembly for security applications.

**Phase 4**
The final stage of the WebAssembly standardization process, indicating a feature is fully standardized and recommended for implementation.

**Polyfill**
Code providing modern features in older environments. Some WebAssembly features can be polyfilled for broader support.

**Polyphony**
In audio synthesis, the ability to play multiple notes simultaneously. WebAssembly synthesizers typically support 8-32 voice polyphony.

**Proposal**
A potential WebAssembly feature working through the standardization process (Phases 0-4).

**PWASM (Polkadot WebAssembly)**
WebAssembly used in the Polkadot blockchain for smart contracts.

**Pyodide**
A Python distribution for WebAssembly, including NumPy, SciPy, and other scientific packages.

## Q

**Quantization**
Reducing the precision of values. Common in audio/graphics processing and machine learning.

**Queue**
A FIFO data structure. WebAssembly implementations often use queues for worker communication or task scheduling.

## R

**RAII (Resource Acquisition Is Initialization)**
A C++ pattern tying resource lifetime to object lifetime. Translates well to WebAssembly with proper compiler support.

**Reference Types**
WebAssembly types for holding references: funcref, externref, and GC types. Enable interaction with host objects.

**Register Allocation**
Compiler optimization assigning variables to registers. WebAssembly is stack-based but runtimes allocate registers during compilation.

**Relaxed SIMD**
A WebAssembly proposal for non-deterministic SIMD operations that can use faster platform-specific instructions.

**Result**
A function's return value(s). WebAssembly functions can return 0, 1, or multiple values (multi-value feature).

**Return**
An instruction terminating function execution and returning value(s) to the caller.

**Reverb**
An audio effect simulating room acoustics. Computationally expensive, often implemented in WebAssembly.

**Rust**
A systems programming language with excellent WebAssembly support via rustc, wasm-bindgen, and wasm-pack.

**Runtime**
An environment executing WebAssembly: browser engine, wasmtime, wasmer, Node.js, etc.

## S

**Sandbox**
A secure, isolated execution environment. WebAssembly runs in a sandbox with no direct access to host resources.

**Sample Rate**
Audio samples per second (typically 44,100 Hz or 48,000 Hz). Determines audio quality and processing requirements.

**Saturating Arithmetic**
Operations that clamp to min/max values instead of wrapping on overflow. Useful in graphics and audio processing.

**Section**
A part of a WebAssembly module containing specific information (types, functions, code, etc.). Modules consist of multiple sections.

**SHA-256**
A cryptographic hash function producing 256-bit digests. Commonly implemented in WebAssembly for performance.

**SharedArrayBuffer**
JavaScript buffer type allowing shared memory between threads. Essential for WebAssembly threading.

**SIMD (Single Instruction, Multiple Data)**
Parallel processing of multiple values with one instruction. WebAssembly supports SIMD via v128 vector types.

**Source Maps**
Files mapping compiled code back to source, enabling debugging. WebAssembly supports source maps for debugging WASM.

**Spec (Specification)**
The formal definition of WebAssembly semantics, maintained by the W3C WebAssembly Working Group.

**Spin**
A framework by Fermyon for building serverless applications with WebAssembly and WASI.

**Stack**
The operand stack holding intermediate values during computation. WebAssembly is a stack machine.

**Stack Machine**
A computational model where operations work on a stack. WebAssembly uses a typed stack machine.

**Standard Library**
Built-in functionality provided by a language. Languages compiled to WASM often include subsets of their standard libraries.

**State Machine**
A pattern for managing state transitions. Efficiently implemented in WebAssembly using tail calls or explicit loops.

**Store**
An instruction writing to linear memory, like `i32.store` or `f64.store`.

**Streaming**
Processing data incrementally as it arrives. WebAssembly supports streaming compilation via `WebAssembly.compileStreaming()`.

**Structured Control Flow**
Control flow using nested blocks, loops, and conditionals rather than arbitrary jumps. WebAssembly enforces structured control flow.

**Subtyping**
A type system feature where one type is compatible with another. WebAssembly GC proposal includes subtyping for struct/array types.

**WASI (WebAssembly System Interface)**
A standardized API for WebAssembly accessing system resources (files, network, time, etc.). Enables portable WASM outside browsers.

**Sustain**
In ADSR envelopes, the level maintained while a note is held.

## T

**Table**
An array of function references (funcref) or external references (externref), used for indirect calls and storing host references.

**Table Section**
A section of a WebAssembly module declaring tables and their sizes.

**Tail Call**
A function call that reuses the caller's stack frame. WebAssembly tail call proposal enables O(1) space recursion via `return_call`.

**TCO (Tail Call Optimization)**
See "Tail Call".

**Text Format (WAT)**
The human-readable representation of WebAssembly, using S-expressions. Files use `.wat` extension.

**Thread**
An independent execution context. WebAssembly supports threads via atomics and SharedArrayBuffer.

**Threading Proposal**
WebAssembly extension (now standardized) adding atomic operations and instructions for thread synchronization.

**TinyGo**
A Go compiler targeting small devices and WebAssembly, producing much smaller binaries than standard Go.

**Toolchain**
A collection of tools for building software. WebAssembly toolchains include compilers, linkers, and optimization tools.

**Trampoline**
A pattern for iterative recursion simulation. WebAssembly tail calls eliminate the need for trampolines.

**Trap**
A runtime error causing immediate execution halt (e.g., division by zero, out-of-bounds access). Traps are catchable by the host.

**Try/Catch**
Exception handling constructs. WebAssembly exception handling proposal (now standardized) adds first-class try/catch.

**Type**
A classification of values. WebAssembly has numeric types (i32, i64, f32, f64), vector types (v128), and reference types (funcref, externref, GC types).

**Type Section**
A section of a WebAssembly module declaring function signatures (types).

## U

**Undefined Behavior (UB)**
Operations with unspecified semantics. WebAssembly minimizes UB compared to languages like C, defining behavior for most operations.

**Unwinding**
The process of propagating exceptions up the call stack, cleaning up resources. Supported by WebAssembly exception handling.

**Userspace**
Code running without privileged system access. WebAssembly is a userspace technology running in a sandbox.

## V

**v128**
WebAssembly's 128-bit SIMD vector type, containing four f32/i32 values, two f64/i64 values, or sixteen i8 values.

**Validation**
Checking that a WebAssembly module is well-formed and type-safe. Performed before instantiation.

**Vector**
A fixed-size array of values processed together. WebAssembly SIMD uses v128 vectors.

**Virtual Machine (VM)**
A software emulation of a computer. WebAssembly runtimes are VMs executing WASM bytecode.

**Voice**
In audio synthesis, an independent sound generator. Polyphonic synthesizers manage multiple voices.

**Voice Stealing**
Reassigning active voices when polyphony limit is reached, typically stealing the oldest or quietest voice.

## W

**W3C (World Wide Web Consortium)**
The standards organization maintaining the WebAssembly specification.

**WAGI (WebAssembly Gateway Interface)**
A specification for running WebAssembly as CGI-like programs, used by some serverless platforms.

**WAMR (WebAssembly Micro Runtime)**
A lightweight standalone WebAssembly runtime for embedded systems (ESP32, Arduino, etc.).

**wasm-bindgen**
A Rust library/tool for high-level bindings between Rust-generated WebAssembly and JavaScript.

**wasm-opt**
An optimizer from the Binaryen toolkit, providing aggressive size/speed optimizations for WebAssembly binaries.

**wasm-pack**
A Rust tool for building, testing, and publishing WebAssembly packages for npm and the web.

**wasm2wat**
A tool converting WebAssembly binary format (.wasm) to text format (.wat) for inspection.

**WABT (WebAssembly Binary Toolkit)**
A suite of tools for working with WebAssembly: wat2wasm, wasm2wat, wasm-objdump, etc.

**Wasm3**
A fast, lightweight WebAssembly interpreter, useful for embedded systems and low-memory environments.

**wasmtime**
A standalone WebAssembly runtime from the Bytecode Alliance, with strong security focus and WASI support.

**Wasmer**
A WebAssembly runtime supporting multiple backends (LLVM, Cranelift, Singlepass) for different performance/compilation time tradeoffs.

**WASI (WebAssembly System Interface)**
A modular system interface for WebAssembly, enabling portable access to system resources outside browsers.

**WASI Preview 1**
The first stable release of WASI, providing file I/O, environment variables, and basic system calls.

**WASI Preview 2**
The upcoming WASI version based on the Component Model, with richer types and asynchronous I/O.

**wat2wasm**
A tool converting WebAssembly text format (.wat) to binary format (.wasm).

**WAT (WebAssembly Text Format)**
See "Text Format".

**Wavetable Synthesis**
An audio synthesis technique using recorded waveforms. Efficient to implement in WebAssembly.

**Web Audio API**
JavaScript API for processing and synthesizing audio in browsers. Works well with WebAssembly for performance-critical processing.

**Web Crypto API**
JavaScript API for cryptographic operations. WebAssembly can implement custom crypto beyond what Web Crypto provides.

**WebGPU**
A modern graphics/compute API for the web. WebAssembly can use WebGPU for GPU-accelerated computation.

**Web MIDI API**
JavaScript API for MIDI input/output. Enables WebAssembly synthesizers to respond to MIDI controllers.

**WebSocket**
A protocol for bidirectional communication. WebAssembly applications can use WebSocket for networking.

**Web Workers**
JavaScript API for background threads. Often used with WebAssembly for parallel processing.

**WIT (WebAssembly Interface Types)**
The interface definition language for the Component Model, defining rich types and interfaces between components.

**Working Group**
The W3C WebAssembly Community Group that develops the specification through a formal process.

**World**
In the Component Model, a description of a component's imports and exports, defined in WIT.

## X

**XOR**
Exclusive OR operation. WebAssembly doesn't have a dedicated XOR instruction but can simulate with `i32.xor` (via other operations, though actually it does have `i32.xor`).

## Y

**Yew**
A Rust framework for building web applications, compiling to WebAssembly similar to React in JavaScript.

## Z

**Zero-Cost Abstraction**
A programming principle where high-level abstractions have no runtime overhead. WebAssembly enables zero-cost abstractions for languages like Rust.

**Zero-Cost Exceptions**
Exception handling with no overhead on the happy path, only when exceptions are thrown. WebAssembly exception handling follows this model.

**Zig**
A systems programming language with WebAssembly support, emphasizing performance and low-level control.

---

## Symbols

**$**
In WAT, the `$` prefix denotes a named identifier (e.g., `$add` for a function name, `$x` for a parameter).

**( )**
S-expression delimiters in WAT. Everything in WAT is enclosed in parentheses.

**.wasm**
File extension for WebAssembly binary modules.

**.wat**
File extension for WebAssembly text format.

**0x00asm**
The magic number at the start of every WebAssembly binary file (hex bytes: 00 61 73 6D).

---

## Acronyms Quick Reference

- **ABI**: Application Binary Interface
- **ADSR**: Attack, Decay, Sustain, Release
- **AES**: Advanced Encryption Standard
- **AOT**: Ahead-Of-Time Compilation
- **API**: Application Programming Interface
- **CDN**: Content Delivery Network
- **CORS**: Cross-Origin Resource Sharing
- **CPS**: Continuation-Passing Style
- **CSP**: Content Security Policy
- **DOM**: Document Object Model
- **DSP**: Digital Signal Processing
- **ECS**: Entity Component System
- **FFI**: Foreign Function Interface
- **FFT**: Fast Fourier Transform
- **FIFO**: First-In, First-Out
- **GC**: Garbage Collection
- **GPU**: Graphics Processing Unit
- **IIR**: Infinite Impulse Response
- **JIT**: Just-In-Time Compilation
- **JSPI**: JavaScript Promise Integration
- **LFO**: Low-Frequency Oscillator
- **LLVM**: Low Level Virtual Machine
- **LTO**: Link-Time Optimization
- **MIDI**: Musical Instrument Digital Interface
- **MVP**: Minimum Viable Product
- **NaN**: Not a Number
- **OTA**: Over-The-Air (updates)
- **PBKDF2**: Password-Based Key Derivation Function 2
- **RAII**: Resource Acquisition Is Initialization
- **SHA**: Secure Hash Algorithm
- **SIMD**: Single Instruction, Multiple Data
- **TCO**: Tail Call Optimization
- **UB**: Undefined Behavior
- **VM**: Virtual Machine
- **W3C**: World Wide Web Consortium
- **WAGI**: WebAssembly Gateway Interface
- **WAMR**: WebAssembly Micro Runtime
- **WASI**: WebAssembly System Interface
- **WAT**: WebAssembly Text Format
- **WABT**: WebAssembly Binary Toolkit
- **WIT**: WebAssembly Interface Types

---

This glossary covers the essential terminology used throughout "Assembling WebAssembly". For more detailed explanations, refer to the specific chapters where these concepts are introduced and explained in depth.
