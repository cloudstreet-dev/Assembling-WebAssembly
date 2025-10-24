# Chapter 32: JavaScript Interop

## Bridging JavaScript and WebAssembly

Effective WASM applications require seamless communication between JavaScript and WASM. This chapter covers patterns, techniques, and best practices for JS↔WASM interop.

## Calling WASM from JavaScript

### Basic Function Calls

```javascript
const { instance } = await WebAssembly.instantiateStreaming(fetch('math.wasm'));

// Call exported functions
const sum = instance.exports.add(10, 20);        // 30
const product = instance.exports.multiply(5, 6); // 30
```

### Parameter Passing

WASM functions accept only numeric types (i32, i64, f32, f64):

```javascript
// ✓ These work directly
instance.exports.add(5, 3);                    // i32 params
instance.exports.multiply_f64(2.5, 4.0);       // f64 params

// ✗ These don't work directly
instance.exports.process("string");            // Error: not a number
instance.exports.handle({ data: 42 });         // Error: not a number
instance.exports.callback(() => {});           // Error: not a number
```

## Passing Complex Data

### Passing Strings

**JavaScript to WASM**:

```javascript
class WasmString {
    constructor(instance) {
        this.instance = instance;
        this.encoder = new TextEncoder();
        this.decoder = new TextDecoder();
    }

    // Allocate and write string to WASM memory
    write(str) {
        const bytes = this.encoder.encode(str + '\0');
        const ptr = this.instance.exports.alloc(bytes.length);

        const memory = new Uint8Array(this.instance.exports.memory.buffer);
        memory.set(bytes, ptr);

        return ptr;
    }

    // Read string from WASM memory
    read(ptr) {
        const memory = new Uint8Array(this.instance.exports.memory.buffer);

        let end = ptr;
        while (memory[end] !== 0) end++;

        const bytes = memory.subarray(ptr, end);
        return this.decoder.decode(bytes);
    }

    // Free string memory
    free(ptr) {
        // Requires WASM exports a free function
        this.instance.exports.free(ptr);
    }
}

// Usage
const wasmStr = new WasmString(instance);

const ptr = wasmStr.write("Hello, WASM!");
instance.exports.process_string(ptr);
wasmStr.free(ptr);

const resultPtr = instance.exports.get_result();
const result = wasmStr.read(resultPtr);
console.log(result);
wasmStr.free(resultPtr);
```

### Passing Arrays

```javascript
class WasmArray {
    constructor(instance) {
        this.instance = instance;
    }

    // Write typed array to WASM
    writeFloat32Array(arr) {
        const ptr = this.instance.exports.alloc(arr.length * 4);
        const memory = new Float32Array(this.instance.exports.memory.buffer);
        memory.set(arr, ptr / 4);
        return { ptr, length: arr.length };
    }

    // Read typed array from WASM
    readFloat32Array(ptr, length) {
        const memory = new Float32Array(this.instance.exports.memory.buffer);
        return memory.slice(ptr / 4, ptr / 4 + length);
    }

    free(ptr) {
        this.instance.exports.free(ptr);
    }
}

// Usage
const wasmArr = new WasmArray(instance);

const input = new Float32Array([1.0, 2.0, 3.0, 4.0]);
const { ptr, length } = wasmArr.writeFloat32Array(input);

instance.exports.process_array(ptr, length);

const output = wasmArr.readFloat32Array(ptr, length);
console.log(output);

wasmArr.free(ptr);
```

### Passing Objects (JSON)

```javascript
class WasmJSON {
    constructor(instance, wasmStr) {
        this.instance = instance;
        this.wasmStr = wasmStr;
    }

    // Serialize object to JSON string, pass to WASM
    write(obj) {
        const json = JSON.stringify(obj);
        return this.wasmStr.write(json);
    }

    // Read JSON string from WASM, parse to object
    read(ptr) {
        const json = this.wasmStr.read(ptr);
        return JSON.parse(json);
    }
}

// Usage
const wasmJSON = new WasmJSON(instance, wasmStr);

const data = { name: "Alice", age: 30, scores: [85, 90, 95] };
const ptr = wasmJSON.write(data);

instance.exports.process_json(ptr);

const resultPtr = instance.exports.get_json_result();
const result = wasmJSON.read(resultPtr);
console.log(result);

wasmStr.free(ptr);
wasmStr.free(resultPtr);
```

## Calling JavaScript from WASM

### Import Functions

WASM can import JavaScript functions:

**JavaScript**:
```javascript
const imports = {
    env: {
        log: (value) => console.log('WASM says:', value),

        alert_message: (strPtr) => {
            const str = wasmStr.read(strPtr);
            alert(str);
        },

        get_timestamp: () => Date.now(),

        random_range: (min, max) => {
            return Math.floor(Math.random() * (max - min + 1)) + min;
        }
    }
};

const { instance } = await WebAssembly.instantiate(module, imports);
```

**WASM (Rust example)**:
```rust
extern "C" {
    fn log(value: i32);
    fn alert_message(ptr: *const u8);
    fn get_timestamp() -> f64;
    fn random_range(min: i32, max: i32) -> i32;
}

#[no_mangle]
pub extern "C" fn do_something() {
    unsafe {
        log(42);

        let timestamp = get_timestamp();
        log(timestamp as i32);

        let random = random_range(1, 100);
        log(random);
    }
}
```

### Callbacks

Pass JavaScript callbacks to WASM:

```javascript
const callbacks = new Map();
let nextCallbackId = 0;

const imports = {
    env: {
        // WASM calls this to invoke a JS callback
        invoke_callback: (callbackId, arg) => {
            const callback = callbacks.get(callbackId);
            if (callback) {
                callback(arg);
            }
        }
    }
};

// Register callback, get ID
function registerCallback(func) {
    const id = nextCallbackId++;
    callbacks.set(id, func);
    return id;
}

// Usage
const callbackId = registerCallback((value) => {
    console.log('Callback received:', value);
});

instance.exports.set_callback(callbackId);
instance.exports.trigger_callback(42);  // Calls JS callback with 42
```

### Promises and Async

WASM functions can't be async, but you can use callbacks:

```javascript
const pendingPromises = new Map();
let nextPromiseId = 0;

const imports = {
    env: {
        // WASM calls this to create a promise
        create_promise: () => {
            const id = nextPromiseId++;
            const promise = new Promise((resolve, reject) => {
                pendingPromises.set(id, { resolve, reject });
            });
            return id;
        },

        // WASM calls this when promise resolves
        resolve_promise: (id, value) => {
            const pending = pendingPromises.get(id);
            if (pending) {
                pending.resolve(value);
                pendingPromises.delete(id);
            }
        },

        // Async operation (fetch example)
        fetch_data: (urlPtr, promiseId) => {
            const url = wasmStr.read(urlPtr);

            fetch(url)
                .then(response => response.json())
                .then(data => {
                    // Process data, write to WASM memory
                    const dataPtr = wasmJSON.write(data);
                    instance.exports.handle_fetch_result(promiseId, dataPtr);
                })
                .catch(error => {
                    console.error('Fetch failed:', error);
                    instance.exports.handle_fetch_error(promiseId);
                });
        }
    }
};
```

## Performance Optimization

### Minimize Boundary Crossings

```javascript
// ✗ Bad: many crossings
for (let i = 0; i < 10000; i++) {
    instance.exports.process_one(i);  // 10,000 calls
}

// ✓ Good: one crossing
const ptr = instance.exports.alloc(10000 * 4);
const view = new Int32Array(instance.exports.memory.buffer);
for (let i = 0; i < 10000; i++) {
    view[ptr / 4 + i] = i;
}
instance.exports.process_batch(ptr, 10000);  // 1 call
instance.exports.free(ptr);
```

### Cache Memory Views

```javascript
// ✗ Bad: recreate every time
function processData(ptr, len) {
    const view = new Uint8Array(instance.exports.memory.buffer);
    for (let i = 0; i < len; i++) {
        view[ptr + i] = i;
    }
}

// ✓ Good: reuse view (but watch for growth!)
let memoryView = new Uint8Array(instance.exports.memory.buffer);

function processData(ptr, len) {
    // Refresh if memory grew
    if (memoryView.buffer !== instance.exports.memory.buffer) {
        memoryView = new Uint8Array(instance.exports.memory.buffer);
    }

    for (let i = 0; i < len; i++) {
        memoryView[ptr + i] = i;
    }
}
```

### Use Bulk Memory Operations

```javascript
// ✗ Slow: element by element
for (let i = 0; i < data.length; i++) {
    view[ptr + i] = data[i];
}

// ✓ Fast: bulk copy
view.set(data, ptr);
```

## Advanced Patterns

### Object Handles

Pass JavaScript objects to WASM via handles:

```javascript
const objectRegistry = new Map();
let nextHandle = 1;

const imports = {
    env: {
        // Create handle for JS object
        create_handle: (type) => {
            const handle = nextHandle++;
            let obj;

            if (type === 0) {
                obj = new Map();
            } else if (type === 1) {
                obj = new Set();
            } else if (type === 2) {
                obj = [];
            }

            objectRegistry.set(handle, obj);
            return handle;
        },

        // Operate on object via handle
        map_set: (handle, keyPtr, valuePtr) => {
            const map = objectRegistry.get(handle);
            const key = wasmStr.read(keyPtr);
            const value = wasmStr.read(valuePtr);
            map.set(key, value);
        },

        map_get: (handle, keyPtr) => {
            const map = objectRegistry.get(handle);
            const key = wasmStr.read(keyPtr);
            const value = map.get(key);
            return wasmStr.write(value || "");
        },

        // Clean up handle
        destroy_handle: (handle) => {
            objectRegistry.delete(handle);
        }
    }
};

// WASM can now use JS Maps, Sets, Arrays via handles
```

### Shared Memory Buffer

For performance-critical data sharing:

```javascript
// Allocate persistent buffer in WASM
const BUFFER_SIZE = 1024 * 1024;  // 1MB
const bufferPtr = instance.exports.alloc(BUFFER_SIZE);

// Create reusable view
const sharedBuffer = new Uint8Array(
    instance.exports.memory.buffer,
    bufferPtr,
    BUFFER_SIZE
);

// JS and WASM can both access this memory directly
function updateBuffer() {
    // JS writes
    sharedBuffer[0] = 42;
    sharedBuffer[1] = 100;

    // WASM processes
    instance.exports.process_shared_buffer(bufferPtr, BUFFER_SIZE);

    // JS reads results
    console.log(sharedBuffer[0], sharedBuffer[1]);
}
```

### Virtual Method Dispatch

Implement polymorphism across JS/WASM boundary:

```javascript
// Vtable in JavaScript
class VTable {
    constructor() {
        this.methods = new Map();
    }

    register(methodId, func) {
        this.methods.set(methodId, func);
    }

    call(methodId, ...args) {
        const method = this.methods.get(methodId);
        if (!method) throw new Error(`Method ${methodId} not found`);
        return method(...args);
    }
}

const vtable = new VTable();

// Register methods
vtable.register(0, (x) => x * 2);
vtable.register(1, (x) => x * x);
vtable.register(2, (x) => Math.sqrt(x));

const imports = {
    env: {
        call_method: (methodId, arg) => {
            return vtable.call(methodId, arg);
        }
    }
};

// WASM can call different JS functions via method IDs
```

## Error Handling Across Boundary

### WASM Exceptions to JS

```javascript
try {
    instance.exports.might_trap();
} catch (error) {
    if (error instanceof WebAssembly.RuntimeError) {
        console.error('WASM trapped:', error.message);
        // Handle: unreachable, out of bounds, etc.
    }
}
```

### JS Exceptions to WASM

WASM can't catch JS exceptions directly. Use return codes:

```javascript
const imports = {
    env: {
        risky_operation: (arg) => {
            try {
                // Might throw
                const result = somethingRisky(arg);
                return result;
            } catch (error) {
                console.error('Error in JS:', error);
                return -1;  // Error sentinel
            }
        }
    }
};

// WASM checks return value
// if (result == -1) { handle_error(); }
```

## Debugging Interop

### Logging Helper

```javascript
function createDebugImports(imports) {
    const proxy = {};

    for (const [module, funcs] of Object.entries(imports)) {
        proxy[module] = {};
        for (const [name, func] of Object.entries(funcs)) {
            proxy[module][name] = (...args) => {
                console.log(`WASM → JS: ${module}.${name}(${args.join(', ')})`);
                const result = func(...args);
                console.log(`WASM ← JS: ${result}`);
                return result;
            };
        }
    }

    return proxy;
}

// Usage
const debugImports = createDebugImports(imports);
const { instance } = await WebAssembly.instantiate(module, debugImports);
```

### Memory Inspector

```javascript
function inspectMemory(instance, ptr, length) {
    const view = new Uint8Array(instance.exports.memory.buffer);
    const bytes = view.slice(ptr, ptr + length);

    console.log('Memory at', ptr.toString(16) + ':');
    console.log('Hex:', Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join(' '));
    console.log('Decimal:', Array.from(bytes).join(' '));
    console.log('ASCII:', String.fromCharCode(...bytes.filter(b => b >= 32 && b < 127)));
}

// Usage
inspectMemory(instance, 0x1000, 64);
```

## Best Practices

1. **Minimize calls**: Batch operations when possible
2. **Use typed arrays**: Efficient data transfer
3. **Cache views**: But refresh after memory.grow
4. **Handle errors**: Check return values, catch exceptions
5. **Document types**: Clear comments on data layouts
6. **Validate data**: Check bounds, null pointers
7. **Profile**: Use DevTools to find bottlenecks
8. **Use helpers**: Create reusable interop utilities

## Next Steps

Effective JavaScript interop is crucial for WASM web applications. With these patterns—string/array passing, callbacks, object handles—you can build rich interactions while maintaining performance. Next, we'll explore DOM access and manipulation from WebAssembly.
