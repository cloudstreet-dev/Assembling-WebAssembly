# Chapter 21: C/C++ with Emscripten

## Emscripten Overview

Emscripten is the most mature and feature-complete toolchain for compiling C/C++ to WebAssembly. It provides:

- Full C/C++ language support
- POSIX-like APIs
- OpenGL/WebGL support
- SDL (Simple DirectMedia Layer)
- Comprehensive standard library (libc, C++ STL)
- JavaScript glue code generation

**Use cases**: Porting games, scientific libraries, media codecs, legacy C/C++ code

## Installation

```bash
# Clone
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk

# Install latest
./emsdk install latest
./emsdk activate latest

# Set up environment
source ./emsdk_env.sh

# Verify
emcc --version
```

## Basic Compilation

### Hello World

**hello.c**:
```c
#include <stdio.h>

int main() {
    printf("Hello, WebAssembly!\n");
    return 0;
}
```

**Compile**:
```bash
emcc hello.c -o hello.html
```

**Generates**:
- `hello.wasm` - The WebAssembly module
- `hello.js` - JavaScript glue code
- `hello.html` - Test HTML page

**Run**:
```bash
emrun hello.html
# Opens browser with built-in server
```

### Output Formats

```bash
# HTML + JS + WASM
emcc hello.c -o hello.html

# JS + WASM (for integration)
emcc hello.c -o hello.js

# Just WASM (no glue code)
emcc hello.c -o hello.wasm
```

## Exporting Functions

### Simple Exports

```c
#include <emscripten.h>

EMSCRIPTEN_KEEPALIVE
int add(int a, int b) {
    return a + b;
}

EMSCRIPTEN_KEEPALIVE
float multiply(float a, float b) {
    return a * b;
}
```

**Compile**:
```bash
emcc math.c -o math.js -s EXPORTED_FUNCTIONS='["_add", "_multiply"]' \
    -s EXPORTED_RUNTIME_METHODS='["ccall", "cwrap"]'
```

**Use in JavaScript**:
```javascript
const add = Module.cwrap('add', 'number', ['number', 'number']);
const result = add(5, 3);
console.log(result);  // 8

// Or with ccall
const result2 = Module.ccall('multiply', 'number', ['number', 'number'], [4.5, 2.0]);
console.log(result2);  // 9.0
```

### Automatic Export

```bash
# Export all functions
emcc math.c -o math.js -s EXPORTED_FUNCTIONS='_*' \
    -s EXPORTED_RUNTIME_METHODS='["ccall", "cwrap"]'
```

## Memory Management

### Allocating from JavaScript

```javascript
// Allocate memory
const ptr = Module._malloc(1024);  // 1KB

// Use Uint8Array view
const buffer = new Uint8Array(Module.HEAPU8.buffer, ptr, 1024);

// Write data
for (let i = 0; i < 1024; i++) {
    buffer[i] = i % 256;
}

// Call WASM function with pointer
Module._process_buffer(ptr, 1024);

// Free memory
Module._free(ptr);
```

### Allocating from C

```c
#include <emscripten.h>
#include <stdlib.h>
#include <string.h>

EMSCRIPTEN_KEEPALIVE
char* create_string(const char* input) {
    char* result = (char*)malloc(strlen(input) + 10);
    sprintf(result, "Hello, %s!", input);
    return result;
}

EMSCRIPTEN_KEEPALIVE
void free_string(char* ptr) {
    free(ptr);
}
```

**JavaScript**:
```javascript
const str = "World";
const strPtr = Module.allocateUTF8(str);
const resultPtr = Module._create_string(strPtr);

const result = Module.UTF8ToString(resultPtr);
console.log(result);  // "Hello, World!"

Module._free_string(resultPtr);
Module._free(strPtr);
```

## String Handling

### Helper Functions

```c
#include <emscripten.h>
#include <string.h>
#include <ctype.h>

EMSCRIPTEN_KEEPALIVE
char* to_uppercase(const char* input) {
    int len = strlen(input);
    char* result = (char*)malloc(len + 1);

    for (int i = 0; i < len; i++) {
        result[i] = toupper(input[i]);
    }
    result[len] = '\0';

    return result;
}
```

**Compile with string utilities**:
```bash
emcc strings.c -o strings.js \
    -s EXPORTED_FUNCTIONS='["_to_uppercase"]' \
    -s EXPORTED_RUNTIME_METHODS='["ccall", "cwrap", "UTF8ToString", "allocateUTF8"]'
```

**Use**:
```javascript
const to_uppercase = Module.cwrap('to_uppercase', 'number', ['string']);

const inputStr = "hello world";
const resultPtr = to_uppercase(inputStr);
const result = Module.UTF8ToString(resultPtr);

console.log(result);  // "HELLO WORLD"

Module._free(resultPtr);
```

## Calling JavaScript from C

### EM_JS Macro

```c
#include <emscripten.h>

EM_JS(void, show_alert, (const char* msg), {
    alert(UTF8ToString(msg));
});

EM_JS(double, get_random, (), {
    return Math.random();
});

EM_JS(void, console_log, (const char* msg), {
    console.log(UTF8ToString(msg));
});

int main() {
    console_log("Starting program");

    double r = get_random();
    printf("Random: %f\n", r);

    show_alert("Hello from C!");

    return 0;
}
```

### emscripten_run_script

```c
#include <emscripten.h>

int main() {
    emscripten_run_script("console.log('Hello from C via script')");

    int result = emscripten_run_script_int("2 + 2");
    printf("2 + 2 = %d\n", result);  // 4

    return 0;
}
```

## File System

### Emscripten Virtual File System

```c
#include <stdio.h>
#include <emscripten.h>

int main() {
    // Write file
    FILE* f = fopen("/tmp/test.txt", "w");
    fprintf(f, "Hello, file system!\n");
    fclose(f);

    // Read file
    f = fopen("/tmp/test.txt", "r");
    char buffer[256];
    fgets(buffer, sizeof(buffer), f);
    fclose(f);

    printf("Read: %s", buffer);

    return 0;
}
```

**Preloading files**:
```bash
emcc program.c -o program.js --preload-file data/
```

This embeds `data/` into a `.data` file loaded at runtime.

### Persistent Storage (IndexedDB)

```bash
emcc program.c -o program.js -s FORCE_FILESYSTEM=1 -lidbfs.js
```

**Mount IDBFS in C**:
```c
#include <emscripten.h>

EM_JS(void, setup_idbfs, (), {
    FS.mkdir('/data');
    FS.mount(IDBFS, {}, '/data');

    FS.syncfs(true, function(err) {
        if (err) console.error(err);
        console.log('IDBFS loaded');
    });
});

int main() {
    setup_idbfs();

    // Use /data for persistent storage
    FILE* f = fopen("/data/persistent.txt", "w");
    fprintf(f, "This persists across sessions\n");
    fclose(f);

    // Sync to IndexedDB
    emscripten_run_script("FS.syncfs(false, function(){})");

    return 0;
}
```

## C++ Features

### Classes and Objects

```cpp
#include <emscripten/bind.h>
#include <string>

class Counter {
private:
    int value;

public:
    Counter() : value(0) {}

    void increment() { value++; }
    void decrement() { value--; }
    int getValue() const { return value; }
    void setValue(int v) { value = v; }
};

EMSCRIPTEN_BINDINGS(counter_module) {
    emscripten::class_<Counter>("Counter")
        .constructor<>()
        .function("increment", &Counter::increment)
        .function("decrement", &Counter::decrement)
        .function("getValue", &Counter::getValue)
        .function("setValue", &Counter::setValue);
}
```

**Compile**:
```bash
em++ counter.cpp -o counter.js --bind
```

**JavaScript**:
```javascript
const counter = new Module.Counter();
counter.increment();
counter.increment();
console.log(counter.getValue());  // 2
counter.setValue(10);
console.log(counter.getValue());  // 10
```

### STL Containers

```cpp
#include <emscripten/bind.h>
#include <vector>
#include <string>

std::vector<int> create_vector() {
    return {1, 2, 3, 4, 5};
}

int sum_vector(const std::vector<int>& vec) {
    int sum = 0;
    for (int x : vec) sum += x;
    return sum;
}

EMSCRIPTEN_BINDINGS(vector_module) {
    emscripten::function("create_vector", &create_vector);
    emscripten::function("sum_vector", &sum_vector);

    emscripten::register_vector<int>("VectorInt");
}
```

## OpenGL/WebGL

Emscripten translates OpenGL ES to WebGL:

```c
#include <GLES3/gl3.h>
#include <emscripten/html5.h>

void render() {
    glClearColor(0.3f, 0.3f, 0.8f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    // Drawing code...

    emscripten_webgl_commit_frame();
}

int main() {
    EmscriptenWebGLContextAttributes attrs;
    emscripten_webgl_init_context_attributes(&attrs);

    EMSCRIPTEN_WEBGL_CONTEXT_HANDLE ctx =
        emscripten_webgl_create_context("#canvas", &attrs);
    emscripten_webgl_make_context_current(ctx);

    emscripten_set_main_loop(render, 0, 1);

    return 0;
}
```

**Compile**:
```bash
emcc opengl.c -o opengl.html -s USE_WEBGL2=1 -s FULL_ES3=1
```

## SDL (Simple DirectMedia Layer)

Emscripten includes SDL for graphics, audio, and input:

```c
#include <SDL2/SDL.h>
#include <emscripten.h>

SDL_Window* window;
SDL_Renderer* renderer;

void main_loop() {
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        if (event.type == SDL_QUIT) {
            emscripten_cancel_main_loop();
        }
    }

    SDL_SetRenderDrawColor(renderer, 100, 149, 237, 255);
    SDL_RenderClear(renderer);

    // Drawing code...

    SDL_RenderPresent(renderer);
}

int main() {
    SDL_Init(SDL_INIT_VIDEO);
    SDL_CreateWindowAndRenderer(800, 600, 0, &window, &renderer);

    emscripten_set_main_loop(main_loop, 0, 1);

    return 0;
}
```

**Compile**:
```bash
emcc sdl_app.c -o sdl_app.html -s USE_SDL=2
```

## Optimization

### Size Optimization

```bash
# Optimize for size
emcc program.c -o program.js -O z

# Enable closure compiler
emcc program.c -o program.js -O3 --closure 1

# Remove assertions
emcc program.c -o program.js -O3 -s ASSERTIONS=0

# Minimize runtime
emcc program.c -o program.js -O3 -s MINIMAL_RUNTIME=1
```

### Performance Optimization

```bash
# Optimize for speed
emcc program.c -o program.js -O3

# Enable SIMD
emcc program.c -o program.js -O3 -msimd128

# Enable LTO
emcc program.c -o program.js -O3 -flto
```

## Debugging

### Enable Debugging

```bash
# Debug build
emcc program.c -o program.js -g

# With source maps
emcc program.c -o program.js -g4

# Enable assertions
emcc program.c -o program.js -g -s ASSERTIONS=2

# Safe heap (detect memory errors)
emcc program.c -o program.js -g -s SAFE_HEAP=1
```

### AddressSanitizer

```bash
emcc program.c -o program.js -fsanitize=address
```

Detects memory errors: buffer overflows, use-after-free, etc.

## Real-World Example: Image Processing

```c
#include <emscripten.h>
#include <stdlib.h>
#include <math.h>

typedef struct {
    unsigned char r, g, b, a;
} Pixel;

EMSCRIPTEN_KEEPALIVE
void grayscale(Pixel* pixels, int width, int height) {
    int total = width * height;

    for (int i = 0; i < total; i++) {
        float gray = 0.299f * pixels[i].r +
                     0.587f * pixels[i].g +
                     0.114f * pixels[i].b;

        unsigned char g = (unsigned char)gray;
        pixels[i].r = g;
        pixels[i].g = g;
        pixels[i].b = g;
    }
}

EMSCRIPTEN_KEEPALIVE
void blur(Pixel* pixels, int width, int height) {
    Pixel* temp = (Pixel*)malloc(width * height * sizeof(Pixel));

    for (int y = 1; y < height - 1; y++) {
        for (int x = 1; x < width - 1; x++) {
            int r = 0, g = 0, b = 0;

            for (int dy = -1; dy <= 1; dy++) {
                for (int dx = -1; dx <= 1; dx++) {
                    int idx = (y + dy) * width + (x + dx);
                    r += pixels[idx].r;
                    g += pixels[idx].g;
                    b += pixels[idx].b;
                }
            }

            int idx = y * width + x;
            temp[idx].r = r / 9;
            temp[idx].g = g / 9;
            temp[idx].b = b / 9;
            temp[idx].a = pixels[idx].a;
        }
    }

    for (int i = 0; i < width * height; i++) {
        pixels[i] = temp[i];
    }

    free(temp);
}
```

**JavaScript integration**:
```javascript
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
const pixels = imageData.data;

// Copy to WASM memory
const pixelPtr = Module._malloc(pixels.length);
Module.HEAPU8.set(pixels, pixelPtr);

// Process
Module._grayscale(pixelPtr, canvas.width, canvas.height);

// Copy back
const processed = Module.HEAPU8.subarray(pixelPtr, pixelPtr + pixels.length);
imageData.data.set(processed);

ctx.putImageData(imageData, 0, 0);

Module._free(pixelPtr);
```

## Porting Existing Libraries

Emscripten makes porting C/C++ libraries straightforward:

```bash
# Many libraries just work
emcc library.c -o library.js

# With configure/make
emconfigure ./configure
emmake make
```

**Examples of ported libraries**:
- zlib (compression)
- libpng, libjpeg (image formats)
- Ogg Vorbis (audio)
- SQLite (database)
- Box2D (physics)

## Next Steps

Emscripten is the most mature WASM toolchain, with comprehensive C/C++ support, extensive libraries, and powerful features. It's the tool of choice for porting existing C/C++ codebases and building high-performance web applications. Next, we'll explore direct Clang compilation to WASM without Emscripten's JavaScript layer.
