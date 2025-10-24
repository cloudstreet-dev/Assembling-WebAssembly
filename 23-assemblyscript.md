# Chapter 23: AssemblyScript

## What is AssemblyScript?

AssemblyScript is a TypeScript-like language that compiles directly to WebAssembly. It's designed specifically for WASM, not ported from another platform.

**Key features**:
- TypeScript-like syntax (familiar to JS/TS developers)
- Compiles directly to WASM (no JavaScript runtime)
- Manual memory management (explicit control)
- Small binaries
- Fast compilation

**Not TypeScript**: Despite the similarity, AssemblyScript has important differences—it's statically typed, requires explicit types, and manages memory manually.

## Installation

```bash
npm install -g assemblyscript

# Or in a project
npm install --save-dev assemblyscript

# Initialize AssemblyScript project
npx asinit .
```

**Project structure**:
```
project/
├── assembly/
│   ├── index.ts          # AssemblyScript source
│   └── tsconfig.json     # AS compiler config
├── build/
│   └── module.wasm       # Compiled output
└── package.json
```

## Basic Syntax

### Simple Functions

**assembly/index.ts**:
```typescript
export function add(a: i32, b: i32): i32 {
    return a + b;
}

export function multiply(a: f64, b: f64): f64 {
    return a * b;
}

export function greet(name: string): string {
    return "Hello, " + name + "!";
}
```

**Compile**:
```bash
npm run asbuild
```

**Use in JavaScript**:
```javascript
import { instantiate } from "./build/module.js";

const { add, multiply, greet } = await instantiate();

console.log(add(5, 3));          // 8
console.log(multiply(2.5, 4.0)); // 10.0
console.log(greet("World"));     // "Hello, World!"
```

## Type System

AssemblyScript's types map directly to WASM:

### Integer Types
```typescript
let a: i8 = 127;        // 8-bit signed integer
let b: u8 = 255;        // 8-bit unsigned integer
let c: i16 = 32767;     // 16-bit signed
let d: u16 = 65535;     // 16-bit unsigned
let e: i32 = 2147483647;  // 32-bit signed
let f: u32 = 4294967295;  // 32-bit unsigned
let g: i64 = 9223372036854775807; // 64-bit signed
let h: u64 = 18446744073709551615; // 64-bit unsigned
```

### Floating-Point Types
```typescript
let x: f32 = 3.14159;   // 32-bit float
let y: f64 = 2.71828;   // 64-bit float
```

### Boolean
```typescript
let flag: bool = true;
```

### Arrays
```typescript
let numbers: i32[] = [1, 2, 3, 4, 5];
let floats: f64[] = new Array<f64>(100);

export function sum_array(arr: i32[]): i32 {
    let total: i32 = 0;
    for (let i = 0; i < arr.length; i++) {
        total += arr[i];
    }
    return total;
}
```

### Strings
```typescript
export function string_ops(input: string): string {
    return input.toUpperCase();
}

export function string_length(s: string): i32 {
    return s.length;
}
```

## Memory Management

AssemblyScript has manual memory management:

### Allocating Objects

```typescript
class Point {
    x: f64;
    y: f64;

    constructor(x: f64, y: f64) {
        this.x = x;
        this.y = y;
    }

    distance(): f64 {
        return Math.sqrt(this.x * this.x + this.y * this.y);
    }
}

export function create_point(x: f64, y: f64): Point {
    return new Point(x, y);
}
```

### Manual Memory

```typescript
import { memory } from "env";

export function alloc(size: usize): usize {
    return heap.alloc(size);
}

export function free(ptr: usize): void {
    heap.free(ptr);
}
```

## Classes and Objects

```typescript
export class Counter {
    private value: i32;

    constructor(initial: i32 = 0) {
        this.value = initial;
    }

    increment(): void {
        this.value++;
    }

    decrement(): void {
        this.value--;
    }

    getValue(): i32 {
        return this.value;
    }

    reset(): void {
        this.value = 0;
    }
}
```

**JavaScript usage**:
```javascript
const { Counter } = await instantiate();

const counter = new Counter(10);
counter.increment();
counter.increment();
console.log(counter.getValue());  // 12
```

## Standard Library

AssemblyScript includes useful built-ins:

### Math
```typescript
export function math_operations(x: f64): f64 {
    let a = Math.sqrt(x);
    let b = Math.pow(x, 2);
    let c = Math.sin(x);
    let d = Math.max(a, b);
    return d;
}
```

### Array Methods
```typescript
export function array_ops(): i32[] {
    let arr = [1, 2, 3, 4, 5];

    arr.push(6);
    arr.pop();

    let doubled = arr.map<i32>((x: i32) => x * 2);
    let filtered = arr.filter((x: i32) => x > 2);

    return filtered;
}
```

### String Methods
```typescript
export function string_ops(input: string): string {
    return input
        .trim()
        .toLowerCase()
        .replace("hello", "goodbye");
}
```

## Importing JavaScript Functions

```typescript
@external("env", "console.log")
declare function consoleLog(s: string): void;

@external("env", "Math.random")
declare function random(): f64;

export function demo_imports(): void {
    consoleLog("Hello from AssemblyScript!");

    let r = random();
    consoleLog("Random: " + r.toString());
}
```

**Provide imports**:
```javascript
const imports = {
    env: {
        "console.log": (str) => console.log(str),
        "Math.random": () => Math.random()
    }
};

const { demo_imports } = await instantiate(imports);
demo_imports();
```

## Performance Optimization

### Use Appropriate Types

```typescript
// Good: specific types
function fast_add(a: i32, b: i32): i32 {
    return a + b;
}

// Avoid: generic types require more overhead
function slow_add<T>(a: T, b: T): T {
    return a + b;  // Less efficient
}
```

### Inline Functions

```typescript
@inline
function square(x: f64): f64 {
    return x * x;
}

export function sum_of_squares(a: f64, b: f64): f64 {
    return square(a) + square(b);  // Inlined
}
```

### Manual Loop Unrolling

```typescript
export function vector_add(a: Float64Array, b: Float64Array, result: Float64Array): void {
    let len = a.length;

    for (let i = 0; i < len; i += 4) {
        result[i] = a[i] + b[i];
        result[i + 1] = a[i + 1] + b[i + 1];
        result[i + 2] = a[i + 2] + b[i + 2];
        result[i + 3] = a[i + 3] + b[i + 3];
    }
}
```

## Real-World Example: Image Processing

```typescript
export class ImageProcessor {
    width: i32;
    height: i32;
    data: Uint8ClampedArray;

    constructor(width: i32, height: i32) {
        this.width = width;
        this.height = height;
        this.data = new Uint8ClampedArray(width * height * 4);
    }

    grayscale(): void {
        for (let i = 0; i < this.data.length; i += 4) {
            let r = this.data[i];
            let g = this.data[i + 1];
            let b = this.data[i + 2];

            let gray = u8(0.299 * f32(r) + 0.587 * f32(g) + 0.114 * f32(b));

            this.data[i] = gray;
            this.data[i + 1] = gray;
            this.data[i + 2] = gray;
            // Alpha unchanged
        }
    }

    invert(): void {
        for (let i = 0; i < this.data.length; i += 4) {
            this.data[i] = 255 - this.data[i];
            this.data[i + 1] = 255 - this.data[i + 1];
            this.data[i + 2] = 255 - this.data[i + 2];
        }
    }

    brightness(amount: f32): void {
        for (let i = 0; i < this.data.length; i += 4) {
            this.data[i] = u8(Mathf.min(f32(this.data[i]) * amount, 255));
            this.data[i + 1] = u8(Mathf.min(f32(this.data[i + 1]) * amount, 255));
            this.data[i + 2] = u8(Mathf.min(f32(this.data[i + 2]) * amount, 255));
        }
    }
}
```

## Configuration

**asconfig.json**:
```json
{
  "targets": {
    "debug": {
      "outFile": "build/debug.wasm",
      "textFile": "build/debug.wat",
      "sourceMap": true,
      "debug": true
    },
    "release": {
      "outFile": "build/release.wasm",
      "textFile": "build/release.wat",
      "sourceMap": true,
      "optimizeLevel": 3,
      "shrinkLevel": 2,
      "converge": true,
      "noAssert": true
    }
  },
  "options": {
    "bindings": "esm"
  }
}
```

## Testing

**tests/index.spec.ts**:
```typescript
import { add, multiply } from "../assembly";

describe("Math functions", () => {
    it("should add numbers", () => {
        expect(add(2, 3)).toBe(5);
    });

    it("should multiply numbers", () => {
        expect(multiply(4, 5)).toBe(20);
    });
});
```

## Differences from TypeScript

### Must Specify Types

```typescript
// TypeScript (works)
function add(a, b) {
    return a + b;
}

// AssemblyScript (requires types)
function add(a: i32, b: i32): i32 {
    return a + b;
}
```

### No 'any' Type

```typescript
// Not allowed
let x: any = 42;

// Must be specific
let x: i32 = 42;
```

### No Union Types

```typescript
// Not allowed
function process(x: string | number) { }

// Use function overloading or generics
```

### No Undefined

```typescript
// Not allowed
let x: number | undefined;

// Use nullable
let x: i32 | null = null;
```

## When to Use AssemblyScript

**Good fit**:
- Developers familiar with TypeScript
- Need small, fast WASM modules
- Don't need TypeScript's full type system
- Want manual memory control

**Not ideal**:
- Need full TypeScript compatibility
- Heavy use of existing TypeScript libraries
- Complex type system requirements

## Next Steps

AssemblyScript provides a familiar syntax for TypeScript developers while compiling directly to efficient WASM. It's ideal for performance-critical web modules, games, and computational tasks where you want TypeScript-like syntax without JavaScript runtime overhead. Next, we'll explore .NET and Blazor, Microsoft's approach to running C# in WebAssembly.
