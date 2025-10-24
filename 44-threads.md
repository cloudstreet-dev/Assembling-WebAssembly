# Chapter 44: Threads and Atomics

## WebAssembly Threading

WebAssembly threads enable true parallel execution across multiple CPU cores using shared memory and atomic operations. This is crucial for CPU-intensive tasks that benefit from parallelization.

**Key Components**:
- **SharedArrayBuffer**: Shared memory between threads
- **Atomic operations**: Thread-safe memory operations
- **Web Workers**: Thread execution in browsers
- **pthreads**: POSIX threads in compiled C/C++

## Browser Support

```javascript
const supportsThreads = typeof SharedArrayBuffer !== 'undefined';
const supportsAtomics = typeof Atomics !== 'undefined';

console.log('Threads supported:', supportsThreads && supportsAtomics);
```

**Required HTTP headers** (for security):
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## Shared Memory Basics

### Creating Shared Memory

```javascript
// Create shared memory (must be SharedArrayBuffer)
const sharedMemory = new WebAssembly.Memory({
    initial: 10,      // 10 pages (640KB)
    maximum: 100,     // 100 pages (6.4MB)
    shared: true      // Enable sharing
});

// Create shared Int32Array view
const sharedView = new Int32Array(sharedMemory.buffer);

// Write from main thread
sharedView[0] = 42;

// Worker can read/write the same memory
worker.postMessage({ memory: sharedMemory });
```

### Atomic Operations

```javascript
const sharedArray = new Int32Array(new SharedArrayBuffer(4));

// Atomic read/write
Atomics.store(sharedArray, 0, 42);
const value = Atomics.load(sharedArray, 0);

// Atomic add
Atomics.add(sharedArray, 0, 10);  // 42 + 10 = 52

// Compare and exchange
const oldValue = Atomics.compareExchange(sharedArray, 0, 52, 100);
// If sharedArray[0] == 52, set to 100; returns old value

// Wait/notify for synchronization
Atomics.wait(sharedArray, 0, 100);  // Wait until sharedArray[0] != 100
Atomics.notify(sharedArray, 0, 1);  // Wake one waiting thread
```

## Rust with Threads

### Basic Threading

```rust
use std::sync::atomic::{AtomicI32, Ordering};
use std::sync::Arc;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Counter {
    value: Arc<AtomicI32>,
}

#[wasm_bindgen]
impl Counter {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Counter {
        Counter {
            value: Arc::new(AtomicI32::new(0)),
        }
    }

    pub fn increment(&self) {
        self.value.fetch_add(1, Ordering::SeqCst);
    }

    pub fn get(&self) -> i32 {
        self.value.load(Ordering::SeqCst)
    }
}
```

### Parallel Array Processing

```rust
use rayon::prelude::*;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parallel_sum(data: &[i32]) -> i32 {
    // Rayon provides data parallelism
    data.par_iter().sum()
}

#[wasm_bindgen]
pub fn parallel_map(data: &mut [i32]) {
    data.par_iter_mut().for_each(|x| {
        *x = *x * 2;
    });
}

#[wasm_bindgen]
pub fn parallel_filter_count(data: &[i32], threshold: i32) -> usize {
    data.par_iter()
        .filter(|&&x| x > threshold)
        .count()
}
```

**Cargo.toml**:
```toml
[dependencies]
wasm-bindgen = "0.2"
rayon = "1.7"
wasm-bindgen-rayon = "1.0"

[profile.release]
opt-level = 3
```

### Thread Pool Pattern

```rust
use std::sync::mpsc;
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.send(job).unwrap();
    }
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv();

            match job {
                Ok(job) => {
                    println!("Worker {} got a job", id);
                    job();
                }
                Err(_) => {
                    println!("Worker {} disconnected", id);
                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}

// Usage
#[wasm_bindgen]
pub fn parallel_process(data: Vec<i32>) -> Vec<i32> {
    let pool = ThreadPool::new(4);
    let (tx, rx) = mpsc::channel();

    let chunk_size = data.len() / 4;

    for (i, chunk) in data.chunks(chunk_size).enumerate() {
        let tx = tx.clone();
        let chunk = chunk.to_vec();

        pool.execute(move || {
            let result: i32 = chunk.iter().map(|x| x * 2).sum();
            tx.send((i, result)).unwrap();
        });
    }

    drop(tx);

    let mut results = vec![0; 4];
    for (i, result) in rx {
        results[i] = result;
    }

    results
}
```

## C/C++ with pthreads

### Emscripten Threading

**Compile with threads**:
```bash
emcc -pthread -O3 threaded.c -o threaded.js \
    -s PTHREAD_POOL_SIZE=4 \
    -s TOTAL_MEMORY=67108864
```

**C code**:
```c
#include <pthread.h>
#include <emscripten.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int* data;
    int start;
    int end;
    int result;
} ThreadData;

void* sum_range(void* arg) {
    ThreadData* td = (ThreadData*)arg;
    int sum = 0;

    for (int i = td->start; i < td->end; i++) {
        sum += td->data[i];
    }

    td->result = sum;
    return NULL;
}

EMSCRIPTEN_KEEPALIVE
int parallel_sum(int* data, int length, int num_threads) {
    pthread_t threads[num_threads];
    ThreadData thread_data[num_threads];

    int chunk_size = length / num_threads;

    // Create threads
    for (int i = 0; i < num_threads; i++) {
        thread_data[i].data = data;
        thread_data[i].start = i * chunk_size;
        thread_data[i].end = (i == num_threads - 1) ? length : (i + 1) * chunk_size;

        pthread_create(&threads[i], NULL, sum_range, &thread_data[i]);
    }

    // Wait for completion
    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }

    // Combine results
    int total = 0;
    for (int i = 0; i < num_threads; i++) {
        total += thread_data[i].result;
    }

    return total;
}
```

### Mutex and Synchronization

```c
#include <pthread.h>

static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static int shared_counter = 0;

void* increment_counter(void* arg) {
    int iterations = *(int*)arg;

    for (int i = 0; i < iterations; i++) {
        pthread_mutex_lock(&lock);
        shared_counter++;
        pthread_mutex_unlock(&lock);
    }

    return NULL;
}

EMSCRIPTEN_KEEPALIVE
int test_mutex(int num_threads, int iterations_per_thread) {
    pthread_t threads[num_threads];

    shared_counter = 0;

    for (int i = 0; i < num_threads; i++) {
        pthread_create(&threads[i], NULL, increment_counter, &iterations_per_thread);
    }

    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }

    return shared_counter;
}
```

### Condition Variables

```c
#include <pthread.h>
#include <stdbool.h>

typedef struct {
    int* buffer;
    int size;
    int count;
    int in;
    int out;
    pthread_mutex_t mutex;
    pthread_cond_t not_full;
    pthread_cond_t not_empty;
} BoundedBuffer;

void buffer_init(BoundedBuffer* buf, int size) {
    buf->buffer = malloc(size * sizeof(int));
    buf->size = size;
    buf->count = 0;
    buf->in = 0;
    buf->out = 0;
    pthread_mutex_init(&buf->mutex, NULL);
    pthread_cond_init(&buf->not_full, NULL);
    pthread_cond_init(&buf->not_empty, NULL);
}

void buffer_put(BoundedBuffer* buf, int item) {
    pthread_mutex_lock(&buf->mutex);

    while (buf->count == buf->size) {
        pthread_cond_wait(&buf->not_full, &buf->mutex);
    }

    buf->buffer[buf->in] = item;
    buf->in = (buf->in + 1) % buf->size;
    buf->count++;

    pthread_cond_signal(&buf->not_empty);
    pthread_mutex_unlock(&buf->mutex);
}

int buffer_get(BoundedBuffer* buf) {
    pthread_mutex_lock(&buf->mutex);

    while (buf->count == 0) {
        pthread_cond_wait(&buf->not_empty, &buf->mutex);
    }

    int item = buf->buffer[buf->out];
    buf->out = (buf->out + 1) % buf->size;
    buf->count--;

    pthread_cond_signal(&buf->not_full);
    pthread_mutex_unlock(&buf->mutex);

    return item;
}

// Producer-Consumer example
void* producer(void* arg) {
    BoundedBuffer* buf = (BoundedBuffer*)arg;

    for (int i = 0; i < 100; i++) {
        buffer_put(buf, i);
    }

    return NULL;
}

void* consumer(void* arg) {
    BoundedBuffer* buf = (BoundedBuffer*)arg;

    for (int i = 0; i < 100; i++) {
        int item = buffer_get(buf);
        // Process item
    }

    return NULL;
}
```

## JavaScript Worker Coordination

### Main Thread

```javascript
// Load WASM with shared memory
const memory = new WebAssembly.Memory({
    initial: 10,
    maximum: 100,
    shared: true
});

const { instance } = await WebAssembly.instantiate(wasmModule, {
    env: { memory }
});

// Create workers
const numWorkers = navigator.hardwareConcurrency || 4;
const workers = [];

for (let i = 0; i < numWorkers; i++) {
    const worker = new Worker('worker.js');
    worker.postMessage({
        module: wasmModule,
        memory: memory,
        workerId: i,
        numWorkers: numWorkers
    });

    workers.push(worker);
}

// Distribute work
const data = new Int32Array(1000000);
for (let i = 0; i < data.length; i++) {
    data[i] = i;
}

// Copy to shared memory
const dataPtr = instance.exports.alloc(data.length * 4);
const sharedData = new Int32Array(memory.buffer, dataPtr, data.length);
sharedData.set(data);

// Tell workers to process
const chunkSize = Math.ceil(data.length / numWorkers);

workers.forEach((worker, i) => {
    worker.postMessage({
        cmd: 'process',
        start: i * chunkSize,
        end: Math.min((i + 1) * chunkSize, data.length),
        dataPtr: dataPtr
    });
});

// Collect results
let completed = 0;
const results = [];

workers.forEach((worker, i) => {
    worker.onmessage = (e) => {
        if (e.data.cmd === 'done') {
            results[i] = e.data.result;
            completed++;

            if (completed === numWorkers) {
                const total = results.reduce((a, b) => a + b, 0);
                console.log('Total:', total);
            }
        }
    };
});
```

### Worker Thread

```javascript
// worker.js
let wasmInstance;
let memory;
let workerId;

self.onmessage = async (e) => {
    if (e.data.module) {
        // Initialize worker
        memory = e.data.memory;
        workerId = e.data.workerId;

        const { instance } = await WebAssembly.instantiate(e.data.module, {
            env: { memory }
        });

        wasmInstance = instance;

        self.postMessage({ cmd: 'ready' });
    } else if (e.data.cmd === 'process') {
        // Process chunk
        const result = wasmInstance.exports.sum_range(
            e.data.dataPtr,
            e.data.start,
            e.data.end
        );

        self.postMessage({
            cmd: 'done',
            result: result
        });
    }
};
```

## Parallel Algorithms

### Parallel Reduction

```rust
use rayon::prelude::*;

#[wasm_bindgen]
pub fn parallel_reduce(data: &[i32]) -> i32 {
    data.par_iter()
        .cloned()
        .reduce(|| 0, |a, b| a + b)
}

// Custom reduction
#[wasm_bindgen]
pub fn parallel_max(data: &[i32]) -> Option<i32> {
    data.par_iter()
        .cloned()
        .reduce(|| i32::MIN, |a, b| a.max(b))
        .into()
}
```

### Parallel Sort

```rust
#[wasm_bindgen]
pub fn parallel_sort(data: &mut [i32]) {
    data.par_sort_unstable();
}

// Custom comparison
#[wasm_bindgen]
pub fn parallel_sort_by(data: &mut [i32]) {
    data.par_sort_unstable_by(|a, b| b.cmp(a));  // Descending
}
```

### Parallel Search

```rust
#[wasm_bindgen]
pub fn parallel_find(data: &[i32], target: i32) -> Option<usize> {
    data.par_iter()
        .position_any(|&x| x == target)
}

#[wasm_bindgen]
pub fn parallel_all(data: &[i32], predicate: i32) -> bool {
    data.par_iter().all(|&x| x > predicate)
}

#[wasm_bindgen]
pub fn parallel_any(data: &[i32], predicate: i32) -> bool {
    data.par_iter().any(|&x| x > predicate)
}
```

## Lock-Free Data Structures

### Lock-Free Queue

```rust
use std::sync::atomic::{AtomicUsize, AtomicPtr, Ordering};
use std::ptr;

pub struct LockFreeQueue<T> {
    head: AtomicPtr<Node<T>>,
    tail: AtomicPtr<Node<T>>,
}

struct Node<T> {
    data: Option<T>,
    next: AtomicPtr<Node<T>>,
}

impl<T> LockFreeQueue<T> {
    pub fn new() -> Self {
        let dummy = Box::into_raw(Box::new(Node {
            data: None,
            next: AtomicPtr::new(ptr::null_mut()),
        }));

        LockFreeQueue {
            head: AtomicPtr::new(dummy),
            tail: AtomicPtr::new(dummy),
        }
    }

    pub fn enqueue(&self, data: T) {
        let node = Box::into_raw(Box::new(Node {
            data: Some(data),
            next: AtomicPtr::new(ptr::null_mut()),
        }));

        loop {
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*tail).next.load(Ordering::Acquire) };

            if next.is_null() {
                if unsafe {
                    (*tail).next.compare_exchange(
                        ptr::null_mut(),
                        node,
                        Ordering::Release,
                        Ordering::Acquire
                    ).is_ok()
                } {
                    let _ = self.tail.compare_exchange(
                        tail,
                        node,
                        Ordering::Release,
                        Ordering::Acquire
                    );
                    return;
                }
            } else {
                let _ = self.tail.compare_exchange(
                    tail,
                    next,
                    Ordering::Release,
                    Ordering::Acquire
                );
            }
        }
    }

    pub fn dequeue(&self) -> Option<T> {
        loop {
            let head = self.head.load(Ordering::Acquire);
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*head).next.load(Ordering::Acquire) };

            if head == tail {
                if next.is_null() {
                    return None;  // Queue is empty
                }

                let _ = self.tail.compare_exchange(
                    tail,
                    next,
                    Ordering::Release,
                    Ordering::Acquire
                );
            } else {
                if self.head.compare_exchange(
                    head,
                    next,
                    Ordering::Release,
                    Ordering::Acquire
                ).is_ok() {
                    let data = unsafe { (*next).data.take() };
                    unsafe { Box::from_raw(head) };  // Free old head
                    return data;
                }
            }
        }
    }
}
```

## Performance Benchmarking

```javascript
async function benchmarkThreading() {
    const size = 10000000;  // 10M elements
    const data = new Int32Array(size);

    for (let i = 0; i < size; i++) {
        data[i] = i;
    }

    // Single-threaded
    console.time('Single-threaded');
    const resultSingle = instance.exports.sum_sequential(dataPtr, size);
    console.timeEnd('Single-threaded');

    // Multi-threaded (4 threads)
    console.time('Multi-threaded');
    const resultMulti = await sumParallel(dataPtr, size, 4);
    console.timeEnd('Multi-threaded');

    console.log('Results match:', resultSingle === resultMulti);

    // Typical results on 4-core CPU:
    // Single-threaded: 45ms
    // Multi-threaded: 13ms
    // ~3.5x speedup
}
```

## Common Pitfalls

### Race Conditions

```rust
// ✗ Bad: Race condition
static mut COUNTER: i32 = 0;

#[wasm_bindgen]
pub fn increment_unsafe() {
    unsafe {
        COUNTER += 1;  // Not thread-safe!
    }
}

// ✓ Good: Use atomics
use std::sync::atomic::{AtomicI32, Ordering};

static COUNTER: AtomicI32 = AtomicI32::new(0);

#[wasm_bindgen]
pub fn increment_safe() {
    COUNTER.fetch_add(1, Ordering::SeqCst);
}
```

### Deadlocks

```rust
// ✗ Bad: Can deadlock
use std::sync::Mutex;

static LOCK_A: Mutex<i32> = Mutex::new(0);
static LOCK_B: Mutex<i32> = Mutex::new(0);

fn thread1() {
    let _a = LOCK_A.lock().unwrap();
    let _b = LOCK_B.lock().unwrap();  // Might wait forever
}

fn thread2() {
    let _b = LOCK_B.lock().unwrap();
    let _a = LOCK_A.lock().unwrap();  // Might wait forever
}

// ✓ Good: Consistent lock order
fn thread1_fixed() {
    let _a = LOCK_A.lock().unwrap();
    let _b = LOCK_B.lock().unwrap();
}

fn thread2_fixed() {
    let _a = LOCK_A.lock().unwrap();  // Same order
    let _b = LOCK_B.lock().unwrap();
}
```

## Best Practices

1. **Use atomic operations**: For simple shared state
2. **Minimize locking**: Coarse-grained locks over fine-grained
3. **Avoid deadlocks**: Consistent lock ordering
4. **Test thoroughly**: Race conditions are hard to reproduce
5. **Profile first**: Not all code benefits from parallelization
6. **Consider overhead**: Thread creation/synchronization costs
7. **Use thread pools**: Reuse threads instead of creating new ones
8. **Handle errors**: Thread panics can be tricky
9. **Document thread safety**: Clear comments on shared state
10. **Benchmark**: Measure actual speedup

## When to Use Threading

**Good candidates**:
- Large dataset processing
- Independent computations
- CPU-intensive tasks
- Embarrassingly parallel problems

**Poor candidates**:
- Small datasets (overhead > benefit)
- Sequential dependencies
- I/O-bound tasks
- Simple operations

## Next Steps

Threading in WebAssembly enables true parallel execution, leveraging multiple CPU cores for performance-critical workloads. With shared memory, atomics, and thread pools, you can build highly concurrent applications that scale with available hardware.

Next, we'll explore the WebAssembly Component Model, a new paradigm for composable, language-agnostic WebAssembly modules with rich type systems and interface definitions.
