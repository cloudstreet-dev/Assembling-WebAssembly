# Chapter 31: Browser APIs

## The WebAssembly JavaScript API

Browsers expose WebAssembly through a comprehensive JavaScript API. Understanding this API is crucial for effective web integration.

## Core Objects

### WebAssembly.Module

Represents compiled WASM code:

```javascript
// Compile from bytes
const module = await WebAssembly.compile(bytes);

// Validate before compiling
if (WebAssembly.validate(bytes)) {
    const module = await WebAssembly.compile(bytes);
}

// Inspect module
const imports = WebAssembly.Module.imports(module);
const exports = WebAssembly.Module.exports(module);

console.log('Imports:', imports);
// [{module: "env", name: "memory", kind: "memory"}, ...]

console.log('Exports:', exports);
// [{name: "add", kind: "function"}, ...]
```

### WebAssembly.Instance

Represents an instantiated module:

```javascript
const instance = await WebAssembly.instantiate(module, importObject);

// Access exports
const { add, memory } = instance.exports;

// Call exported function
const result = add(5, 3);
```

### WebAssembly.Memory

Linear memory object:

```javascript
// Create memory
const memory = new WebAssembly.Memory({
    initial: 1,      // 1 page (64KB)
    maximum: 10,     // 10 pages max (640KB)
    shared: false    // Not shared (for threading)
});

// Access buffer
const buffer = memory.buffer;  // ArrayBuffer
const view = new Uint8Array(buffer);

// Write to memory
view[0] = 42;
view[1] = 100;

// Grow memory
memory.grow(2);  // Add 2 pages (128KB)
console.log('Pages:', memory.buffer.byteLength / 65536);
```

### WebAssembly.Table

Function reference table:

```javascript
// Create table
const table = new WebAssembly.Table({
    initial: 10,
    maximum: 20,
    element: 'anyfunc'  // or 'funcref'
});

// Get/set elements
const func = table.get(0);
table.set(1, someFunction);

// Grow table
table.grow(5);  // Add 5 slots
```

### WebAssembly.Global

Global variables:

```javascript
// Immutable global
const immutable = new WebAssembly.Global({
    value: 'i32',
    mutable: false
}, 42);

console.log(immutable.value);  // 42
// immutable.value = 43;  // Error: global is immutable

// Mutable global
const mutable = new WebAssembly.Global({
    value: 'i32',
    mutable: true
}, 0);

mutable.value = 100;
console.log(mutable.value);  // 100
```

## Compilation APIs

### Synchronous Compilation

```javascript
// compile() - returns Promise
const module = await WebAssembly.compile(bytes);

// compileStreaming() - compile while downloading
const module = await WebAssembly.compileStreaming(fetch('module.wasm'));

// instantiate() - compile and instantiate
const { instance, module } = await WebAssembly.instantiate(bytes, imports);

// instantiateStreaming() - fastest
const { instance, module } = await WebAssembly.instantiateStreaming(
    fetch('module.wasm'),
    imports
);
```

### Streaming Compilation (Recommended)

```javascript
// Best practice: use streaming
async function loadWasm(url) {
    const { instance } = await WebAssembly.instantiateStreaming(
        fetch(url),
        {
            env: {
                memory: new WebAssembly.Memory({ initial: 1 }),
                // other imports...
            }
        }
    );

    return instance.exports;
}

const wasm = await loadWasm('optimized.wasm');
```

Streaming compilation starts compiling as bytes arrive, dramatically reducing load time.

## Error Handling

### Compilation Errors

```javascript
try {
    const module = await WebAssembly.compile(invalidBytes);
} catch (error) {
    if (error instanceof WebAssembly.CompileError) {
        console.error('Compilation failed:', error.message);
    }
}
```

### Instantiation Errors

```javascript
try {
    const instance = await WebAssembly.instantiate(module, {});
} catch (error) {
    if (error instanceof WebAssembly.LinkError) {
        console.error('Missing import:', error.message);
    }
}
```

### Runtime Errors

```javascript
try {
    instance.exports.might_trap();
} catch (error) {
    if (error instanceof WebAssembly.RuntimeError) {
        console.error('WASM trapped:', error.message);
        // Common: "unreachable executed", "out of bounds memory access"
    }
}
```

## Memory Management

### Accessing Memory

```javascript
const memory = instance.exports.memory;
const buffer = memory.buffer;

// Create views
const u8 = new Uint8Array(buffer);
const u32 = new Uint32Array(buffer);
const f64 = new Float64Array(buffer);

// Read/write
u8[0] = 255;
u32[10] = 0xDEADBEEF;
f64[5] = 3.14159;
```

### Handling Memory Growth

```javascript
let memory = instance.exports.memory;
let buffer = memory.buffer;
let view = new Uint8Array(buffer);

// After WASM calls memory.grow, buffer is detached!
instance.exports.grow_memory();

// Must recreate views
buffer = memory.buffer;
view = new Uint8Array(buffer);
```

### Safe Memory Access Helper

```javascript
function getMemoryView(instance) {
    const memory = instance.exports.memory;
    return {
        u8: new Uint8Array(memory.buffer),
        u16: new Uint16Array(memory.buffer),
        u32: new Uint32Array(memory.buffer),
        i32: new Int32Array(memory.buffer),
        f32: new Float32Array(memory.buffer),
        f64: new Float64Array(memory.buffer),
    };
}

// Usage
let views = getMemoryView(instance);

// After potential growth
views = getMemoryView(instance);
```

## String Handling

### Writing Strings to WASM Memory

```javascript
function writeString(memory, str) {
    const encoder = new TextEncoder();
    const bytes = encoder.encode(str + '\0');  // Null-terminated

    // Assume we have an alloc function
    const ptr = instance.exports.alloc(bytes.length);

    const view = new Uint8Array(memory.buffer);
    view.set(bytes, ptr);

    return ptr;
}

// Usage
const ptr = writeString(instance.exports.memory, "Hello, WASM!");
instance.exports.process_string(ptr);
instance.exports.free(ptr);
```

### Reading Strings from WASM Memory

```javascript
function readString(memory, ptr) {
    const view = new Uint8Array(memory.buffer);

    // Find null terminator
    let end = ptr;
    while (view[end] !== 0) end++;

    // Decode UTF-8
    const bytes = view.subarray(ptr, end);
    const decoder = new TextDecoder();
    return decoder.decode(bytes);
}

// Usage
const strPtr = instance.exports.get_string();
const str = readString(instance.exports.memory, strPtr);
console.log(str);
```

## Working with Typed Arrays

### Zero-Copy Data Transfer

```javascript
// Create data in JavaScript
const input = new Float32Array([1.0, 2.0, 3.0, 4.0, 5.0]);

// Allocate in WASM
const ptr = instance.exports.alloc(input.length * 4);  // 4 bytes per f32

// Copy to WASM memory
const wasmMemory = new Float32Array(instance.exports.memory.buffer);
wasmMemory.set(input, ptr / 4);  // Divide by 4 (f32 size)

// Process in WASM
instance.exports.process_floats(ptr, input.length);

// Read results
const output = wasmMemory.slice(ptr / 4, ptr / 4 + input.length);

// Free
instance.exports.free(ptr);
```

## Caching Compiled Modules

### IndexedDB Caching

```javascript
async function getCachedModule(url) {
    const cacheName = 'wasm-cache-v1';
    const cache = await caches.open(cacheName);

    // Try cache first
    const cached = await cache.match(url);
    if (cached) {
        const bytes = await cached.arrayBuffer();
        return await WebAssembly.compile(bytes);
    }

    // Not cached, fetch and cache
    const response = await fetch(url);
    await cache.put(url, response.clone());

    const bytes = await response.arrayBuffer();
    return await WebAssembly.compile(bytes);
}

// Usage
const module = await getCachedModule('my-module.wasm');
```

### Structured Clone for Modules

```javascript
// Modules can be cloned (cheap)
const module = await WebAssembly.compile(bytes);

// Store in IndexedDB
const db = await openDB('wasm-store');
await db.put('modules', module, 'my-module');

// Retrieve
const storedModule = await db.get('modules', 'my-module');
const instance = await WebAssembly.instantiate(storedModule);
```

## Feature Detection

```javascript
// Check WASM support
if (typeof WebAssembly === 'object') {
    console.log('WebAssembly supported');
}

// Check specific features
const features = {
    streaming: typeof WebAssembly.instantiateStreaming === 'function',
    threads: typeof SharedArrayBuffer === 'function',
    simd: WebAssembly.validate(new Uint8Array([
        0, 97, 115, 109, 1, 0, 0, 0,  // Magic + version
        1, 5, 1, 96, 0, 1, 123         // v128 result type
    ])),
    bulkMemory: WebAssembly.validate(new Uint8Array([
        0, 97, 115, 109, 1, 0, 0, 0,
        1, 4, 1, 96, 0, 0,
        3, 2, 1, 0,
        10, 7, 1, 5, 0, 252, 10, 0, 0, 11  // memory.copy instruction
    ]))
};

console.log('Features:', features);
```

## Performance Best Practices

### 1. Use Streaming APIs

```javascript
// ✓ Good: streaming
WebAssembly.instantiateStreaming(fetch('module.wasm'));

// ✗ Bad: sequential
fetch('module.wasm')
    .then(r => r.arrayBuffer())
    .then(bytes => WebAssembly.instantiate(bytes));
```

### 2. Cache Compiled Modules

```javascript
// Compile once, instantiate many times
const module = await WebAssembly.compileStreaming(fetch('module.wasm'));

const instance1 = await WebAssembly.instantiate(module);
const instance2 = await WebAssembly.instantiate(module);
```

### 3. Minimize JS↔WASM Calls

```javascript
// ✗ Bad: many calls
for (let i = 0; i < 1000; i++) {
    wasm.process(i);
}

// ✓ Good: batch processing
const ptr = wasm.alloc(1000 * 4);
// ... write data to memory
wasm.process_batch(ptr, 1000);
wasm.free(ptr);
```

### 4. Use Typed Arrays for Data Transfer

```javascript
// ✗ Bad: element by element
for (let i = 0; i < data.length; i++) {
    view[ptr + i] = data[i];
}

// ✓ Good: bulk copy
view.set(data, ptr);
```

## Debugging in DevTools

### Console Output

```javascript
// In import object
const imports = {
    env: {
        console_log: (x) => console.log('WASM:', x)
    }
};
```

### Breakpoints and Stepping

1. Open DevTools (F12)
2. Sources tab → WASM modules listed
3. Set breakpoints in WASM code
4. Inspect stack, locals, memory

### Memory Inspector

```javascript
// From console during breakpoint
memory = instance.exports.memory
new Uint8Array(memory.buffer).slice(0, 100)  // First 100 bytes
```

## Next Steps

The Browser API provides the foundation for all web-based WASM. Understanding these APIs—memory management, compilation options, error handling—is essential for building robust web applications. Next, we'll explore JavaScript interop patterns for seamless integration between JS and WASM.
