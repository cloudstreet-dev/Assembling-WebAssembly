# Chapter 25: Zig

## Zig and WebAssembly

Zig is a modern systems programming language designed as a better C. It has excellent WebAssembly support with:

- Manual memory management (no GC)
- C interop (can compile C code)
- Compile-time execution
- Small, fast binaries
- Cross-compilation built-in

**Key strengths for WASM**:
- Tiny output (competitive with Rust)
- No runtime overhead
- Excellent C/C++ integration
- Fast compilation

## Installation

```bash
# Download from https://ziglang.org/download/

# Or use package manager
brew install zig  # macOS
snap install zig --classic  # Linux

# Verify
zig version
```

## Basic Compilation

### Simple Example

**add.zig**:
```zig
export fn add(a: i32, b: i32) i32 {
    return a + b;
}

export fn multiply(a: f64, b: f64) f64 {
    return a * b;
}
```

**Compile**:
```bash
zig build-lib add.zig -target wasm32-freestanding -dynamic -rdynamic
```

**Use in JavaScript**:
```javascript
const { instance } = await WebAssembly.instantiateStreaming(fetch('add.wasm'));
console.log(instance.exports.add(5, 3));  // 8
```

### WASI Target

**hello.zig**:
```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello from Zig!\n", .{});
}
```

**Compile**:
```bash
zig build-exe hello.zig -target wasm32-wasi
```

**Run**:
```bash
wasmtime hello.wasm
```

## Memory Management

Zig gives full control over memory:

```zig
const std = @import("std");

export fn allocate(size: usize) ?[*]u8 {
    const allocator = std.heap.page_allocator;
    const memory = allocator.alloc(u8, size) catch return null;
    return memory.ptr;
}

export fn deallocate(ptr: [*]u8, size: usize) void {
    const allocator = std.heap.page_allocator;
    const slice = ptr[0..size];
    allocator.free(slice);
}
```

## Comptime (Compile-Time Execution)

Zig's killer feature - code runs at compile time:

```zig
fn fibonacci(n: u32) u32 {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

export fn getFib10() u32 {
    return comptime fibonacci(10);  // Computed at compile time!
}

// Generate lookup table at compile time
const fib_table = comptime blk: {
    var table: [20]u32 = undefined;
    for (table) |*item, i| {
        item.* = fibonacci(@intCast(u32, i));
    }
    break :blk table;
};

export fn fibFromTable(n: u32) u32 {
    return fib_table[n];
}
```

## Error Handling

Zig uses explicit error handling:

```zig
const std = @import("std");

const DivisionError = error{
    DivisionByZero,
};

export fn safeDivide(a: i32, b: i32) i32 {
    const result = divide(a, b) catch |err| {
        return -1;  // Error sentinel
    };
    return result;
}

fn divide(a: i32, b: i32) DivisionError!i32 {
    if (b == 0) return error.DivisionByZero;
    return @divTrunc(a, b);
}
```

## Arrays and Slices

```zig
export fn sumArray(ptr: [*]const i32, len: usize) i32 {
    const slice = ptr[0..len];
    var sum: i32 = 0;
    for (slice) |value| {
        sum += value;
    }
    return sum;
}

export fn fillArray(ptr: [*]i32, len: usize, value: i32) void {
    const slice = ptr[0..len];
    for (slice) |*item| {
        item.* = value;
    }
}
```

## Structs

```zig
const Point = struct {
    x: f64,
    y: f64,

    pub fn distance(self: Point) f64 {
        return @sqrt(self.x * self.x + self.y * self.y);
    }

    pub fn add(self: Point, other: Point) Point {
        return Point{
            .x = self.x + other.x,
            .y = self.y + other.y,
        };
    }
};

export fn createPoint(x: f64, y: f64) Point {
    return Point{ .x = x, .y = y };
}

export fn pointDistance(p: Point) f64 {
    return p.distance();
}
```

## Interfacing with C

Zig can compile C code directly:

**main.zig**:
```zig
const c = @cImport({
    @cInclude("math.h");
});

export fn useCMath(x: f64) f64 {
    return c.sqrt(x);
}
```

Compile C alongside Zig:
```bash
zig build-lib main.zig math.c -target wasm32-freestanding
```

## SIMD

```zig
const std = @import("std");

export fn vectorAdd(a: [*]const f32, b: [*]const f32, result: [*]f32, len: usize) void {
    var i: usize = 0;
    while (i < len) : (i += 4) {
        const va = @Vector(4, f32){ a[i], a[i+1], a[i+2], a[i+3] };
        const vb = @Vector(4, f32){ b[i], b[i+1], b[i+2], b[i+3] };
        const vr = va + vb;

        result[i] = vr[0];
        result[i+1] = vr[1];
        result[i+2] = vr[2];
        result[i+3] = vr[3];
    }
}
```

## Build System

**build.zig**:
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const target = b.standardTargetOptions(.{
        .default_target = .{
            .cpu_arch = .wasm32,
            .os_tag = .freestanding,
        },
    });

    const lib = b.addSharedLibrary("mylib", "src/main.zig", .unversioned);
    lib.setTarget(target);
    lib.setBuildMode(.ReleaseSmall);
    lib.install();
}
```

**Build**:
```bash
zig build
```

## Real Example: Game of Life

```zig
const std = @import("std");

const Grid = struct {
    width: usize,
    height: usize,
    cells: []bool,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, width: usize, height: usize) !Grid {
        const cells = try allocator.alloc(bool, width * height);
        @memset(cells, false);
        return Grid{
            .width = width,
            .height = height,
            .cells = cells,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Grid) void {
        self.allocator.free(self.cells);
    }

    pub fn get(self: Grid, x: usize, y: usize) bool {
        if (x >= self.width or y >= self.height) return false;
        return self.cells[y * self.width + x];
    }

    pub fn set(self: *Grid, x: usize, y: usize, alive: bool) void {
        if (x >= self.width or y >= self.height) return;
        self.cells[y * self.width + x] = alive;
    }

    pub fn countNeighbors(self: Grid, x: usize, y: usize) u8 {
        var count: u8 = 0;
        var dy: i32 = -1;
        while (dy <= 1) : (dy += 1) {
            var dx: i32 = -1;
            while (dx <= 1) : (dx += 1) {
                if (dx == 0 and dy == 0) continue;
                const nx = @intCast(i32, x) + dx;
                const ny = @intCast(i32, y) + dy;
                if (nx >= 0 and nx < self.width and ny >= 0 and ny < self.height) {
                    if (self.get(@intCast(usize, nx), @intCast(usize, ny))) {
                        count += 1;
                    }
                }
            }
        }
        return count;
    }

    pub fn step(self: *Grid) !void {
        const next = try self.allocator.alloc(bool, self.cells.len);
        defer self.allocator.free(next);

        for (0..self.height) |y| {
            for (0..self.width) |x| {
                const neighbors = self.countNeighbors(x, y);
                const alive = self.get(x, y);
                const idx = y * self.width + x;

                next[idx] = if (alive)
                    neighbors == 2 or neighbors == 3
                else
                    neighbors == 3;
            }
        }

        @memcpy(self.cells, next);
    }
};

var gpa = std.heap.GeneralPurposeAllocator(.{}){};
var grid: ?Grid = null;

export fn init_game(width: usize, height: usize) bool {
    const allocator = gpa.allocator();
    grid = Grid.init(allocator, width, height) catch return false;
    return true;
}

export fn step_game() void {
    if (grid) |*g| {
        g.step() catch {};
    }
}

export fn get_cell(x: usize, y: usize) bool {
    if (grid) |g| {
        return g.get(x, y);
    }
    return false;
}

export fn set_cell(x: usize, y: usize, alive: bool) void {
    if (grid) |*g| {
        g.set(x, y, alive);
    }
}
```

## Testing

```zig
const std = @import("std");
const testing = std.testing;

fn add(a: i32, b: i32) i32 {
    return a + b;
}

test "add function" {
    try testing.expectEqual(@as(i32, 5), add(2, 3));
    try testing.expectEqual(@as(i32, 0), add(-1, 1));
}

test "arrays" {
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    try testing.expectEqual(@as(usize, 5), arr.len);
    try testing.expectEqual(@as(i32, 3), arr[2]);
}
```

**Run tests**:
```bash
zig test mycode.zig
```

## Optimizations

```bash
# Size optimization
zig build-lib code.zig -target wasm32-freestanding -O ReleaseSmall

# Speed optimization
zig build-lib code.zig -target wasm32-freestanding -O ReleaseFast

# Both size and speed
zig build-lib code.zig -target wasm32-freestanding -O ReleaseSafe
```

## Cross-Compilation

Zig makes cross-compilation trivial:

```bash
# Compile for WASM from any platform
zig build-exe app.zig -target wasm32-wasi

# Compile C code for WASM
zig cc program.c -target wasm32-wasi -o program.wasm
```

## When to Use Zig for WASM

**Good fit**:
- Need small binaries
- Want C interop without fighting build systems
- Manual memory control
- Compile-time code generation
- Cross-platform development

**Consider alternatives**:
- Large existing Rust ecosystem needed
- Prefer automatic memory management
- Language still evolving (pre-1.0)

## Next Steps

Zig brings modern systems programming to WebAssembly with excellent C interop, tiny binaries, and powerful compile-time features. It's an excellent choice for performance-critical WASM modules and porting C codebases. Next, we'll explore Swift's WebAssembly support via the SwiftWasm project.
