# Chapter 35: Streaming and Progressive Loading

## Why Streaming Matters

Traditional WASM loading requires downloading the entire module before compilation begins. For large modules (10MB+), this creates noticeable delays. Streaming compilation starts compiling while bytes are still downloading, dramatically reducing time-to-interactive.

## Streaming Compilation API

### instantiateStreaming()

The recommended approach for loading WASM:

```javascript
// Best practice: streaming instantiation
const { instance, module } = await WebAssembly.instantiateStreaming(
    fetch('large-module.wasm'),
    {
        env: {
            memory: new WebAssembly.Memory({ initial: 256 }),
            // ... other imports
        }
    }
);

// Start using immediately
const result = instance.exports.compute();
```

### compileStreaming()

For cases where you want to compile once, instantiate multiple times:

```javascript
// Compile while downloading
const module = await WebAssembly.compileStreaming(fetch('module.wasm'));

// Instantiate multiple times with different imports
const instance1 = await WebAssembly.instantiate(module, imports1);
const instance2 = await WebAssembly.instantiate(module, imports2);
```

## Performance Comparison

### Non-Streaming (Slow)

```javascript
// ✗ Bad: Sequential operations
const response = await fetch('module.wasm');       // Download
const bytes = await response.arrayBuffer();        // Wait for all bytes
const { instance } = await WebAssembly.instantiate(bytes, imports);  // Then compile

// For a 10MB module:
// - Download: 2s
// - Compile: 1s
// Total: 3s
```

### Streaming (Fast)

```javascript
// ✓ Good: Parallel operations
const { instance } = await WebAssembly.instantiateStreaming(
    fetch('module.wasm'),
    imports
);

// For the same 10MB module:
// - Download + compile overlap
// Total: ~2.2s (25-30% faster)
```

## Advanced Streaming Patterns

### Custom Response Handling

```javascript
async function loadWasmWithFallback(primaryUrl, fallbackUrl) {
    try {
        return await WebAssembly.instantiateStreaming(
            fetch(primaryUrl),
            imports
        );
    } catch (error) {
        console.warn('Primary WASM failed, trying fallback:', error);
        return await WebAssembly.instantiateStreaming(
            fetch(fallbackUrl),
            imports
        );
    }
}

const { instance } = await loadWasmWithFallback(
    'optimized.wasm',
    'compatible.wasm'
);
```

### Response Modification

```javascript
// Modify response before instantiation
async function loadWasmWithHeaders(url) {
    const response = await fetch(url);

    // Verify MIME type
    const contentType = response.headers.get('Content-Type');
    if (contentType !== 'application/wasm') {
        console.warn('Incorrect MIME type:', contentType);

        // Create new response with correct MIME type
        const corrected = new Response(response.body, {
            headers: {
                'Content-Type': 'application/wasm'
            }
        });

        return await WebAssembly.instantiateStreaming(corrected, imports);
    }

    return await WebAssembly.instantiateStreaming(response, imports);
}
```

### Conditional Loading

```javascript
// Load different modules based on feature detection
async function loadOptimizedWasm() {
    const features = {
        simd: await detectSIMD(),
        threads: typeof SharedArrayBuffer !== 'undefined'
    };

    let moduleUrl;
    if (features.simd && features.threads) {
        moduleUrl = 'module-simd-threads.wasm';
    } else if (features.simd) {
        moduleUrl = 'module-simd.wasm';
    } else {
        moduleUrl = 'module-baseline.wasm';
    }

    return await WebAssembly.instantiateStreaming(
        fetch(moduleUrl),
        imports
    );
}

async function detectSIMD() {
    try {
        // Test bytes for v128 type (SIMD)
        return WebAssembly.validate(new Uint8Array([
            0, 97, 115, 109, 1, 0, 0, 0,  // Magic + version
            1, 5, 1, 96, 0, 1, 123         // Function type with v128
        ]));
    } catch {
        return false;
    }
}
```

## Progressive Loading Strategies

### Split Modules by Functionality

Instead of one large module, split into smaller functional units:

```javascript
// Core module loads first
const { instance: core } = await WebAssembly.instantiateStreaming(
    fetch('core.wasm'),
    imports
);

// App is interactive now!
console.log('App ready');

// Load additional features in background
const features = [
    'image-processing.wasm',
    'video-encoding.wasm',
    'audio-filters.wasm'
].map(url =>
    WebAssembly.instantiateStreaming(fetch(url), imports)
);

// Wait for all features
const modules = await Promise.all(features);

console.log('All features loaded');
```

### Lazy Loading on Demand

```javascript
class WasmModuleManager {
    constructor() {
        this.modules = new Map();
        this.loading = new Map();
    }

    async load(name, url) {
        // Return if already loaded
        if (this.modules.has(name)) {
            return this.modules.get(name);
        }

        // Return existing promise if currently loading
        if (this.loading.has(name)) {
            return this.loading.get(name);
        }

        // Start loading
        const promise = WebAssembly.instantiateStreaming(
            fetch(url),
            this.createImports(name)
        );

        this.loading.set(name, promise);

        try {
            const { instance } = await promise;
            this.modules.set(name, instance);
            this.loading.delete(name);
            return instance;
        } catch (error) {
            this.loading.delete(name);
            throw error;
        }
    }

    createImports(moduleName) {
        return {
            env: {
                log: (msg) => console.log(`[${moduleName}]`, msg),
                // ... module-specific imports
            }
        };
    }
}

// Usage
const manager = new WasmModuleManager();

// Load on demand
document.getElementById('processImage').addEventListener('click', async () => {
    const processor = await manager.load('imageProc', 'image.wasm');
    processor.exports.process();
});

document.getElementById('encodeVideo').addEventListener('click', async () => {
    const encoder = await manager.load('videoEnc', 'video.wasm');
    encoder.exports.encode();
});
```

## Caching Strategies

### Cache API Integration

```javascript
const CACHE_NAME = 'wasm-cache-v1';

async function loadWasmWithCache(url) {
    const cache = await caches.open(CACHE_NAME);

    // Try cache first
    let response = await cache.match(url);

    if (!response) {
        // Not in cache, fetch and cache
        response = await fetch(url);

        // Clone before caching (response can only be read once)
        await cache.put(url, response.clone());
    }

    return await WebAssembly.instantiateStreaming(response, imports);
}

// Usage
const { instance } = await loadWasmWithCache('module.wasm');
```

### Versioned Caching

```javascript
const VERSION = '1.2.3';
const CACHE_NAME = `wasm-v${VERSION}`;

async function loadWithVersionedCache(url) {
    // Open versioned cache
    const cache = await caches.open(CACHE_NAME);

    let response = await cache.match(url);

    if (!response) {
        response = await fetch(url);
        await cache.put(url, response.clone());

        // Clean up old caches
        const cacheNames = await caches.keys();
        await Promise.all(
            cacheNames
                .filter(name => name.startsWith('wasm-v') && name !== CACHE_NAME)
                .map(name => caches.delete(name))
        );
    }

    return await WebAssembly.instantiateStreaming(response, imports);
}
```

### IndexedDB for Compiled Modules

```javascript
// Store compiled modules directly (faster than re-compiling)
class WasmModuleCache {
    constructor(dbName = 'wasm-cache', version = 1) {
        this.dbName = dbName;
        this.version = version;
        this.db = null;
    }

    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.version);

            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                this.db = request.result;
                resolve();
            };

            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                if (!db.objectStoreNames.contains('modules')) {
                    db.createObjectStore('modules');
                }
            };
        });
    }

    async getModule(url) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['modules'], 'readonly');
            const store = transaction.objectStore('modules');
            const request = store.get(url);

            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }

    async putModule(url, module) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['modules'], 'readwrite');
            const store = transaction.objectStore('modules');
            const request = store.put(module, url);

            request.onsuccess = () => resolve();
            request.onerror = () => reject(request.error);
        });
    }

    async load(url, imports) {
        if (!this.db) await this.init();

        // Try cache first
        let module = await this.getModule(url);

        if (module) {
            console.log('Module loaded from IndexedDB');
            return await WebAssembly.instantiate(module, imports);
        }

        // Not cached, compile and cache
        console.log('Compiling module...');
        const response = await fetch(url);
        module = await WebAssembly.compileStreaming(response);

        // WebAssembly.Module can be stored via structured clone
        await this.putModule(url, module);

        return await WebAssembly.instantiate(module, imports);
    }
}

// Usage
const cache = new WasmModuleCache();
const { instance } = await cache.load('module.wasm', imports);
```

## Progress Tracking

### Download Progress

```javascript
async function loadWithProgress(url, onProgress) {
    const response = await fetch(url);
    const contentLength = response.headers.get('Content-Length');

    if (!contentLength) {
        // No content length, can't track progress
        return await WebAssembly.instantiateStreaming(response, imports);
    }

    const total = parseInt(contentLength, 10);
    let loaded = 0;

    // Create custom readable stream
    const reader = response.body.getReader();
    const stream = new ReadableStream({
        async start(controller) {
            while (true) {
                const { done, value } = await reader.read();

                if (done) {
                    controller.close();
                    break;
                }

                loaded += value.length;
                onProgress(loaded, total);
                controller.enqueue(value);
            }
        }
    });

    // Create new response with our stream
    const newResponse = new Response(stream, {
        headers: response.headers
    });

    return await WebAssembly.instantiateStreaming(newResponse, imports);
}

// Usage
const { instance } = await loadWithProgress(
    'large-module.wasm',
    (loaded, total) => {
        const percent = ((loaded / total) * 100).toFixed(1);
        console.log(`Loading: ${percent}% (${loaded}/${total} bytes)`);
        // Update progress bar
        document.getElementById('progress').style.width = `${percent}%`;
    }
);
```

### Compilation Progress (Approximate)

```javascript
async function loadWithEstimatedProgress(url, onProgress) {
    const startTime = performance.now();

    // Estimate: 40% download, 60% compile
    const response = await fetch(url);
    const contentLength = parseInt(
        response.headers.get('Content-Length') || '0',
        10
    );

    let downloadProgress = 0;

    const reader = response.body.getReader();
    const stream = new ReadableStream({
        async start(controller) {
            while (true) {
                const { done, value } = await reader.read();

                if (done) {
                    onProgress(40, 100);  // Download complete
                    controller.close();
                    break;
                }

                downloadProgress += value.length;
                const percent = (downloadProgress / contentLength) * 40;
                onProgress(percent, 100);

                controller.enqueue(value);
            }
        }
    });

    const newResponse = new Response(stream, {
        headers: response.headers
    });

    // Simulate compilation progress
    const progressInterval = setInterval(() => {
        const elapsed = performance.now() - startTime;
        // Rough estimate: 1s per MB for compilation
        const estimatedCompileTime = (contentLength / 1024 / 1024) * 1000;
        const compileProgress = Math.min(
            60,
            (elapsed / estimatedCompileTime) * 60
        );
        onProgress(40 + compileProgress, 100);
    }, 100);

    try {
        const result = await WebAssembly.instantiateStreaming(
            newResponse,
            imports
        );
        clearInterval(progressInterval);
        onProgress(100, 100);
        return result;
    } catch (error) {
        clearInterval(progressInterval);
        throw error;
    }
}

// Usage
await loadWithEstimatedProgress('module.wasm', (current, total) => {
    console.log(`Overall progress: ${current.toFixed(1)}%`);
});
```

## Preloading Strategies

### Preload Link Header

```html
<!-- In HTML head -->
<link rel="preload" href="module.wasm" as="fetch" crossorigin>
<link rel="modulepreload" href="module.js">
```

**Server response header**:
```
Link: </module.wasm>; rel=preload; as=fetch; crossorigin
```

### Programmatic Preloading

```javascript
// Preload during idle time
if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
        fetch('heavy-feature.wasm', { priority: 'low' });
    });
} else {
    // Fallback
    setTimeout(() => {
        fetch('heavy-feature.wasm');
    }, 1000);
}
```

### Predictive Preloading

```javascript
// Preload based on user behavior
const preloadManager = {
    preloadQueue: new Set(),

    async preload(url) {
        if (this.preloadQueue.has(url)) return;

        this.preloadQueue.add(url);

        try {
            // Fetch with low priority
            const response = await fetch(url, { priority: 'low' });

            // Store in cache
            const cache = await caches.open('wasm-preload');
            await cache.put(url, response);

            console.log(`Preloaded: ${url}`);
        } catch (error) {
            console.warn(`Preload failed: ${url}`, error);
        }
    }
};

// Preload on hover
document.getElementById('imageEditorBtn').addEventListener('mouseenter', () => {
    preloadManager.preload('image-editor.wasm');
});

document.getElementById('videoPlayerBtn').addEventListener('mouseenter', () => {
    preloadManager.preload('video-decoder.wasm');
});
```

## Service Worker Integration

### Background Compilation

```javascript
// service-worker.js
self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url);

    if (url.pathname.endsWith('.wasm')) {
        event.respondWith(handleWasmRequest(event.request));
    }
});

async function handleWasmRequest(request) {
    const cache = await caches.open('wasm-compiled');

    // Check cache
    let response = await cache.match(request);

    if (response) {
        return response;
    }

    // Fetch and cache
    response = await fetch(request);

    // Cache the response
    await cache.put(request, response.clone());

    // Optionally: compile in background
    compileInBackground(request.url, response.clone());

    return response;
}

async function compileInBackground(url, response) {
    try {
        const module = await WebAssembly.compileStreaming(response);

        // Store compiled module (could use IndexedDB)
        // This makes subsequent loads even faster
        console.log('Background compilation complete:', url);
    } catch (error) {
        console.error('Background compilation failed:', error);
    }
}
```

### Update Strategy

```javascript
// service-worker.js
const VERSION = '1.0.0';
const WASM_CACHE = `wasm-v${VERSION}`;

self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(WASM_CACHE).then((cache) => {
            return cache.addAll([
                '/core.wasm',
                '/ui.wasm'
                // ... critical modules
            ]);
        })
    );
});

self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames
                    .filter(name => name.startsWith('wasm-v') && name !== WASM_CACHE)
                    .map(name => caches.delete(name))
            );
        })
    );
});
```

## Error Handling in Streaming

### Graceful Degradation

```javascript
async function loadWasmRobustly(url, imports) {
    try {
        // Try streaming first
        return await WebAssembly.instantiateStreaming(fetch(url), imports);
    } catch (streamError) {
        console.warn('Streaming failed, falling back to non-streaming:', streamError);

        try {
            // Fall back to non-streaming
            const response = await fetch(url);
            const bytes = await response.arrayBuffer();
            return await WebAssembly.instantiate(bytes, imports);
        } catch (fallbackError) {
            console.error('All loading methods failed:', fallbackError);
            throw fallbackError;
        }
    }
}
```

### Network Retry Logic

```javascript
async function loadWithRetry(url, imports, maxRetries = 3) {
    let lastError;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            console.log(`Loading attempt ${attempt + 1}/${maxRetries}`);

            return await WebAssembly.instantiateStreaming(
                fetch(url),
                imports
            );
        } catch (error) {
            lastError = error;

            if (attempt < maxRetries - 1) {
                // Exponential backoff
                const delay = Math.pow(2, attempt) * 1000;
                console.log(`Retry in ${delay}ms...`);
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }

    throw new Error(`Failed after ${maxRetries} attempts: ${lastError}`);
}
```

## Optimization Best Practices

### 1. Always Use Streaming

```javascript
// ✓ Good
WebAssembly.instantiateStreaming(fetch('module.wasm'), imports);

// ✗ Bad
fetch('module.wasm')
    .then(r => r.arrayBuffer())
    .then(bytes => WebAssembly.instantiate(bytes, imports));
```

### 2. Compile Once, Instantiate Many

```javascript
// ✓ Good: Share compiled module
const module = await WebAssembly.compileStreaming(fetch('shared.wasm'));

const worker1Instance = await WebAssembly.instantiate(module, imports1);
const worker2Instance = await WebAssembly.instantiate(module, imports2);

// ✗ Bad: Compile multiple times
const instance1 = await WebAssembly.instantiateStreaming(
    fetch('shared.wasm'),
    imports1
);
const instance2 = await WebAssembly.instantiateStreaming(
    fetch('shared.wasm'),
    imports2
);
```

### 3. Cache Aggressively

```javascript
const moduleCache = new Map();

async function getCachedModule(url) {
    if (moduleCache.has(url)) {
        return moduleCache.get(url);
    }

    const module = await WebAssembly.compileStreaming(fetch(url));
    moduleCache.set(url, module);
    return module;
}
```

### 4. Lazy Load Non-Critical Modules

```javascript
// Core functionality loads immediately
const core = await loadCore();

// Non-critical features load in background
setTimeout(() => {
    loadFeature('advanced-editor.wasm');
    loadFeature('export-formats.wasm');
}, 2000);
```

### 5. Use HTTP/2 Push

**Server configuration** (nginx):
```nginx
location /app {
    http2_push /core.wasm;
    http2_push /ui.wasm;
}
```

### 6. Compress WASM Files

**Server configuration**:
```nginx
location ~ \.wasm$ {
    gzip on;
    gzip_types application/wasm;
    # Or better: use Brotli
    brotli on;
    brotli_types application/wasm;
}
```

Typical compression ratios:
- **gzip**: 30-40% reduction
- **Brotli**: 40-50% reduction

### 7. Set Correct MIME Type

```nginx
types {
    application/wasm wasm;
}
```

**Or in JavaScript**:
```javascript
const response = await fetch('module.wasm');
if (response.headers.get('Content-Type') !== 'application/wasm') {
    // Fix MIME type
    const corrected = new Response(response.body, {
        headers: { 'Content-Type': 'application/wasm' }
    });
    return await WebAssembly.instantiateStreaming(corrected, imports);
}
```

## Real-World Loading Strategy

Complete example combining best practices:

```javascript
class WasmLoader {
    constructor() {
        this.cache = new Map();
        this.loading = new Map();
        this.dbCache = null;
    }

    async init() {
        // Initialize IndexedDB cache
        this.dbCache = new WasmModuleCache();
        await this.dbCache.init();
    }

    async load(url, imports, options = {}) {
        const {
            useCache = true,
            priority = 'auto',
            onProgress = null
        } = options;

        // Return if already loaded
        if (this.cache.has(url)) {
            return this.cache.get(url);
        }

        // Deduplicate concurrent loads
        if (this.loading.has(url)) {
            return this.loading.get(url);
        }

        const loadPromise = this._loadInternal(
            url,
            imports,
            useCache,
            priority,
            onProgress
        );

        this.loading.set(url, loadPromise);

        try {
            const instance = await loadPromise;
            this.cache.set(url, instance);
            this.loading.delete(url);
            return instance;
        } catch (error) {
            this.loading.delete(url);
            throw error;
        }
    }

    async _loadInternal(url, imports, useCache, priority, onProgress) {
        // Try IndexedDB cache first
        if (useCache && this.dbCache) {
            try {
                const { instance } = await this.dbCache.load(url, imports);
                return instance;
            } catch (error) {
                console.warn('Cache load failed, fetching:', error);
            }
        }

        // Fetch with progress tracking
        let response;

        if (onProgress) {
            response = await this._fetchWithProgress(url, onProgress);
        } else {
            response = await fetch(url, { priority });
        }

        // Verify MIME type
        const contentType = response.headers.get('Content-Type');
        if (contentType !== 'application/wasm') {
            response = new Response(response.body, {
                headers: { 'Content-Type': 'application/wasm' }
            });
        }

        // Instantiate with streaming
        try {
            const { instance, module } = await WebAssembly.instantiateStreaming(
                response,
                imports
            );

            // Cache compiled module
            if (useCache && this.dbCache) {
                await this.dbCache.putModule(url, module);
            }

            return instance;
        } catch (error) {
            console.error('Streaming instantiation failed:', error);
            throw error;
        }
    }

    async _fetchWithProgress(url, onProgress) {
        const response = await fetch(url);
        const contentLength = parseInt(
            response.headers.get('Content-Length') || '0',
            10
        );

        if (!contentLength) {
            return response;
        }

        let loaded = 0;
        const reader = response.body.getReader();

        const stream = new ReadableStream({
            async start(controller) {
                while (true) {
                    const { done, value } = await reader.read();

                    if (done) {
                        controller.close();
                        break;
                    }

                    loaded += value.length;
                    onProgress(loaded, contentLength);
                    controller.enqueue(value);
                }
            }
        });

        return new Response(stream, { headers: response.headers });
    }

    preload(url) {
        // Preload in background
        if ('requestIdleCallback' in window) {
            requestIdleCallback(() => {
                fetch(url, { priority: 'low' });
            });
        } else {
            setTimeout(() => fetch(url), 100);
        }
    }

    clearCache() {
        this.cache.clear();
    }
}

// Usage
const loader = new WasmLoader();
await loader.init();

// Preload on hover
document.getElementById('editorBtn').addEventListener('mouseenter', () => {
    loader.preload('editor.wasm');
});

// Load with progress
const instance = await loader.load('editor.wasm', imports, {
    onProgress: (loaded, total) => {
        const percent = (loaded / total) * 100;
        updateProgressBar(percent);
    }
});
```

## Measuring Performance

```javascript
async function measureLoadTime(url, imports) {
    const marks = {
        fetchStart: performance.now()
    };

    const response = await fetch(url);
    marks.fetchEnd = performance.now();

    const { instance } = await WebAssembly.instantiateStreaming(
        response,
        imports
    );
    marks.instantiateEnd = performance.now();

    console.log('Performance metrics:', {
        fetchTime: marks.fetchEnd - marks.fetchStart,
        compileTime: marks.instantiateEnd - marks.fetchEnd,
        totalTime: marks.instantiateEnd - marks.fetchStart
    });

    return instance;
}
```

## Next Steps

Streaming compilation and progressive loading are essential for production WASM applications. By leveraging `instantiateStreaming()`, caching strategies, and lazy loading, you can minimize time-to-interactive even for large modules. Combined with preloading and service workers, you can create near-instant loading experiences.

Next, in **Part 7**, we'll explore WASM on the server side—building CLI tools, server applications, and cloud functions with WebAssembly outside the browser.
