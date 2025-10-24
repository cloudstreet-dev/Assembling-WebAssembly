# Chapter 34: Web Workers and Threading

## WebAssembly in Web Workers

Web Workers enable running JavaScript (and WebAssembly) in background threads, preventing UI blocking for compute-intensive tasks.

## Basic Web Worker Setup

### Main Thread

```javascript
// main.js
const worker = new Worker('worker.js');

// Send message to worker
worker.postMessage({ action: 'compute', data: [1, 2, 3, 4, 5] });

// Receive results
worker.addEventListener('message', (event) => {
    console.log('Result from worker:', event.data);
});

// Handle errors
worker.addEventListener('error', (error) => {
    console.error('Worker error:', error.message);
});
```

### Worker Script

```javascript
// worker.js
let wasmInstance = null;

// Load WASM in worker
async function loadWasm() {
    const { instance } = await WebAssembly.instantiateStreaming(
        fetch('compute.wasm')
    );
    wasmInstance = instance;
    postMessage({ type: 'ready' });
}

loadWasm();

// Handle messages
self.addEventListener('message', (event) => {
    const { action, data } = event.data;

    if (action === 'compute' && wasmInstance) {
        const result = wasmInstance.exports.process(data);
        postMessage({ type: 'result', result });
    }
});
```

## Passing Data to Workers

### Structured Cloning

Most data is cloned when passed:

```javascript
// Main thread
worker.postMessage({
    numbers: [1, 2, 3],
    config: { iterations: 1000 }
});

// Worker receives copies, not references
```

### Transferable Objects (Zero-Copy)

ArrayBuffers can be transferred (not cloned):

```javascript
// Main thread
const buffer = new ArrayBuffer(1024 * 1024);  // 1MB
const view = new Uint32Array(buffer);

// Fill with data
for (let i = 0; i < view.length; i++) {
    view[i] = i;
}

// Transfer ownership to worker
worker.postMessage({ buffer }, [buffer]);

// buffer is now unusable in main thread!
console.log(buffer.byteLength);  // 0
```

**Worker**:
```javascript
self.addEventListener('message', (event) => {
    const { buffer } = event.data;
    const view = new Uint32Array(buffer);

    // Process data...

    // Transfer back
    postMessage({ buffer }, [buffer]);
});
```

## WASM Worker Patterns

### Heavy Computation

**worker.js**:
```javascript
let wasm = null;

async function init() {
    const { instance } = await WebAssembly.instantiateStreaming(
        fetch('matrix.wasm')
    );
    wasm = instance;
    postMessage({ ready: true });
}

init();

self.addEventListener('message', async (event) => {
    const { action, data } = event.data;

    switch (action) {
        case 'multiply_matrices':
            const { a, b, size } = data;

            // Allocate in WASM
            const aPtr = wasm.exports.alloc(size * size * 8);
            const bPtr = wasm.exports.alloc(size * size * 8);
            const resultPtr = wasm.exports.alloc(size * size * 8);

            // Copy data to WASM memory
            const memory = new Float64Array(wasm.exports.memory.buffer);
            memory.set(a, aPtr / 8);
            memory.set(b, bPtr / 8);

            // Compute
            wasm.exports.multiply_matrices(aPtr, bPtr, resultPtr, size);

            // Read result
            const result = memory.slice(resultPtr / 8, resultPtr / 8 + size * size);

            // Free
            wasm.exports.free(aPtr);
            wasm.exports.free(bPtr);
            wasm.exports.free(resultPtr);

            postMessage({ action: 'result', result: Array.from(result) });
            break;
    }
});
```

### Image Processing

**Main thread**:
```javascript
const worker = new Worker('image-worker.js');

// Get image data from canvas
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

// Send to worker (transfer pixel data)
worker.postMessage({
    action: 'process',
    imageData,
    width: canvas.width,
    height: canvas.height
}, [imageData.data.buffer]);

worker.addEventListener('message', (event) => {
    const { imageData } = event.data;

    // Put processed image back
    ctx.putImageData(imageData, 0, 0);
});
```

**Worker**:
```javascript
let wasm = null;

async function init() {
    const { instance } = await WebAssembly.instantiateStreaming(
        fetch('image.wasm')
    );
    wasm = instance;
}

init();

self.addEventListener('message', async (event) => {
    const { action, imageData, width, height } = event.data;

    if (action === 'process') {
        const pixels = imageData.data;

        // Allocate in WASM
        const ptr = wasm.exports.alloc(pixels.length);

        // Copy to WASM
        const memory = new Uint8ClampedArray(wasm.exports.memory.buffer);
        memory.set(pixels, ptr);

        // Process (e.g., apply filter)
        wasm.exports.apply_filter(ptr, width, height);

        // Read back
        const processed = memory.slice(ptr, ptr + pixels.length);

        // Free
        wasm.exports.free(ptr);

        // Send back (transfer)
        const newImageData = new ImageData(processed, width, height);
        postMessage({ imageData: newImageData }, [newImageData.data.buffer]);
    }
});
```

## Shared Memory and Atomics

Browsers supporting `SharedArrayBuffer` enable true multi-threading:

### Creating Shared Memory

**Main thread**:
```javascript
// Create shared memory
const shared = new SharedArrayBuffer(1024 * 1024);  // 1MB
const sharedView = new Int32Array(shared);

// Create worker
const worker = new Worker('worker.js');
worker.postMessage({ shared });

// Both threads can access simultaneously
sharedView[0] = 42;

// Wait for worker using Atomics
Atomics.wait(sharedView, 1, 0);  // Wait until sharedView[1] changes
console.log('Worker done, result:', sharedView[0]);
```

**Worker**:
```javascript
let sharedView = null;

self.addEventListener('message', (event) => {
    const { shared } = event.data;
    sharedView = new Int32Array(shared);

    // Do work
    const value = Atomics.load(sharedView, 0);
    const result = value * 2;
    Atomics.store(sharedView, 0, result);

    // Notify main thread
    Atomics.store(sharedView, 1, 1);
    Atomics.notify(sharedView, 1);
});
```

### WASM with Shared Memory

**Compile WASM with shared memory**:
```bash
# Rust
cargo build --target wasm32-unknown-unknown --release \
    -Z build-std=panic_abort,std \
    -Z build-std-features=panic_immediate_abort

# Clang
clang --target=wasm32 -pthread -o threaded.wasm threaded.c
```

**Use in worker**:
```javascript
const shared = new SharedArrayBuffer(65536 * 100);  // 100 pages
const memory = new WebAssembly.Memory({
    initial: 100,
    maximum: 100,
    shared: true
});

const { instance } = await WebAssembly.instantiate(module, {
    env: {
        memory: memory
    }
});
```

## Worker Pool Pattern

Manage multiple workers for parallel processing:

```javascript
class WorkerPool {
    constructor(workerScript, poolSize) {
        this.workers = [];
        this.taskQueue = [];
        this.activeWorkers = new Set();

        for (let i = 0; i < poolSize; i++) {
            const worker = new Worker(workerScript);
            worker.addEventListener('message', (event) => {
                this.handleResult(worker, event.data);
            });
            this.workers.push(worker);
        }
    }

    async execute(task) {
        return new Promise((resolve, reject) => {
            this.taskQueue.push({ task, resolve, reject });
            this.processQueue();
        });
    }

    processQueue() {
        // Find available worker
        const availableWorker = this.workers.find(
            w => !this.activeWorkers.has(w)
        );

        if (availableWorker && this.taskQueue.length > 0) {
            const { task, resolve, reject } = this.taskQueue.shift();

            this.activeWorkers.add(availableWorker);
            availableWorker.currentResolve = resolve;
            availableWorker.currentReject = reject;

            availableWorker.postMessage(task);
        }
    }

    handleResult(worker, result) {
        this.activeWorkers.delete(worker);

        if (worker.currentResolve) {
            worker.currentResolve(result);
            worker.currentResolve = null;
            worker.currentReject = null;
        }

        // Process next task
        this.processQueue();
    }

    terminate() {
        this.workers.forEach(w => w.terminate());
    }
}

// Usage
const pool = new WorkerPool('compute-worker.js', 4);  // 4 workers

const tasks = Array.from({ length: 100 }, (_, i) => ({
    action: 'process',
    data: i
}));

const results = await Promise.all(
    tasks.map(task => pool.execute(task))
);

console.log('All tasks complete:', results);
pool.terminate();
```

## OffscreenCanvas

Render in workers with OffscreenCanvas:

**Main thread**:
```javascript
const canvas = document.getElementById('canvas');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('render-worker.js');
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

**Worker**:
```javascript
let canvas, ctx, wasm;

self.addEventListener('message', async (event) => {
    if (event.data.canvas) {
        canvas = event.data.canvas;
        ctx = canvas.getContext('2d');

        // Load WASM
        const { instance } = await WebAssembly.instantiateStreaming(
            fetch('renderer.wasm')
        );
        wasm = instance;

        // Start render loop
        requestAnimationFrame(render);
    }
});

function render() {
    // WASM computes next frame
    wasm.exports.update();

    // Get pixel data from WASM
    const ptr = wasm.exports.get_pixels();
    const pixels = new Uint8ClampedArray(
        wasm.exports.memory.buffer,
        ptr,
        canvas.width * canvas.height * 4
    );

    // Draw to canvas
    const imageData = new ImageData(pixels, canvas.width, canvas.height);
    ctx.putImageData(imageData, 0, 0);

    requestAnimationFrame(render);
}
```

## Communicating Between Workers

Workers can't directly communicate, but can via main thread:

```javascript
// Main thread coordinates
const worker1 = new Worker('worker1.js');
const worker2 = new Worker('worker2.js');

worker1.addEventListener('message', (event) => {
    // Forward to worker2
    worker2.postMessage({ from: 'worker1', data: event.data });
});

worker2.addEventListener('message', (event) => {
    // Forward to worker1
    worker1.postMessage({ from: 'worker2', data: event.data });
});
```

Or use `SharedArrayBuffer` + `Atomics` for direct communication.

## Error Handling

```javascript
const worker = new Worker('worker.js');

worker.addEventListener('error', (event) => {
    console.error('Worker error:', {
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno
    });
});

worker.addEventListener('messageerror', (event) => {
    console.error('Message error (deserialization failed)');
});

// Worker can also handle its own errors
// worker.js
self.addEventListener('error', (event) => {
    console.error('Internal error:', event.message);
    // Can't prevent default, but can log
});
```

## Performance Tips

1. **Transfer, don't clone**: Use Transferable objects
2. **Batch operations**: Send multiple tasks at once
3. **Reuse workers**: Worker pool pattern
4. **Minimize messages**: Send less frequently
5. **Use SharedArrayBuffer**: For lock-free algorithms
6. **Profile**: Use DevTools Performance tab

## Browser Support

Check for features:

```javascript
const supportsWorkers = typeof Worker !== 'undefined';
const supportsSharedArrayBuffer = typeof SharedArrayBuffer !== 'undefined';
const supportsOffscreenCanvas = typeof OffscreenCanvas !== 'undefined';

console.log({
    workers: supportsWorkers,
    sharedMemory: supportsSharedArrayBuffer,
    offscreenCanvas: supportsOffscreenCanvas
});
```

## Next Steps

Web Workers enable running WASM in background threads, crucial for keeping UIs responsive during heavy computation. With Shared ArrayBuffer and atomics, you can even implement true multi-threaded algorithms. Next, we'll explore streaming compilation and instantiation for faster WASM loading.
