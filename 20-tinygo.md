# Chapter 20: TinyGo

## What is TinyGo?

TinyGo is an alternative Go compiler designed for embedded systems, WebAssembly, and resource-constrained environments. It produces dramatically smaller binaries than the official Go compiler.

**Key advantages**:
- **Much smaller binaries**: 10-100KB vs 2-3MB
- **Better WASI support**: File I/O, networking (proposal)
- **Faster compilation**: Optimized for WASM
- **Still Go**: Familiar syntax and tooling

**Tradeoffs**:
- Not 100% Go compatible
- Smaller standard library
- Some reflection limitations
- Newer, less mature

## Installation

**macOS (Homebrew)**:
```bash
brew tap tinygo-org/tools
brew install tinygo
```

**Linux (binary)**:
```bash
wget https://github.com/tinygo-org/tinygo/releases/download/v0.30.0/tinygo_0.30.0_amd64.deb
sudo dpkg -i tinygo_0.30.0_amd64.deb
```

**Verify**:
```bash
tinygo version
```

## Basic Compilation

### For Browsers

**hello.go**:
```go
package main

func main() {
    println("Hello from TinyGo!")
}
```

**Compile**:
```bash
tinygo build -target wasm -o main.wasm hello.go
```

**Size comparison**:
```bash
# Official Go
GOOS=js GOARCH=wasm go build -o go-main.wasm hello.go
ls -lh go-main.wasm  # ~2.1 MB

# TinyGo
tinygo build -target wasm -o tiny-main.wasm hello.go
ls -lh tiny-main.wasm  # ~7 KB
```

That's a **300x size reduction**!

### For WASI

```bash
tinygo build -target wasi -o program.wasm hello.go
wasmtime program.wasm
```

## JavaScript Interop

TinyGo uses the same `syscall/js` package:

```go
package main

import (
    "syscall/js"
)

func add(this js.Value, args []js.Value) interface{} {
    return args[0].Int() + args[1].Int()
}

func greet(this js.Value, args []js.Value) interface{} {
    name := args[0].String()
    return "Hello, " + name + "!"
}

func main() {
    js.Global().Set("add", js.FuncOf(add))
    js.Global().Set("greet", js.FuncOf(greet))

    println("TinyGo functions ready")

    // Keep running
    select {}
}
```

**Compile**:
```bash
tinygo build -target wasm -o main.wasm main.go
```

**Use wasm_exec.js from TinyGo**:
```bash
cp $(tinygo env TINYGOROOT)/targets/wasm_exec.js .
```

**HTML**:
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>TinyGo WASM</title>
</head>
<body>
    <script src="wasm_exec.js"></script>
    <script>
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject)
            .then((result) => {
                go.run(result.instance);

                // Call TinyGo functions
                console.log(add(10, 32));        // 42
                console.log(greet("TinyGo"));    // "Hello, TinyGo!"
            });
    </script>
</body>
</html>
```

## DOM Manipulation

```go
package main

import (
    "syscall/js"
    "strconv"
)

func setupUI() {
    document := js.Global().Get("document")

    // Create elements
    button := document.Call("createElement", "button")
    button.Set("textContent", "Click Me")

    counter := document.Call("createElement", "div")
    counter.Set("id", "counter")
    counter.Set("textContent", "Clicks: 0")

    body := document.Get("body")
    body.Call("appendChild", button)
    body.Call("appendChild", counter)

    // Add event listener
    count := 0
    callback := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        count++
        counter.Set("textContent", "Clicks: "+strconv.Itoa(count))
        return nil
    })

    button.Call("addEventListener", "click", callback)
}

func main() {
    setupUI()
    select {}
}
```

## Goroutines

Goroutines work in TinyGo WASM:

```go
package main

import (
    "syscall/js"
    "time"
)

func worker(id int, console js.Value) {
    for i := 0; i < 5; i++ {
        console.Call("log", "Worker", id, "iteration", i)
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    console := js.Global().Get("console")
    console.Call("log", "Starting goroutines")

    // Launch goroutines
    for i := 0; i < 3; i++ {
        go worker(i, console)
    }

    // Keep running
    select {}
}
```

## WASI Support

TinyGo has excellent WASI support:

**fileops.go**:
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Write file
    err := os.WriteFile("output.txt", []byte("Hello from TinyGo!"), 0644)
    if err != nil {
        fmt.Println("Error writing:", err)
        return
    }

    // Read file
    data, err := os.ReadFile("output.txt")
    if err != nil {
        fmt.Println("Error reading:", err)
        return
    }

    fmt.Println("Read:", string(data))
}
```

**Compile and run**:
```bash
tinygo build -target=wasi -o program.wasm fileops.go
wasmtime --dir=. program.wasm
# Hello from TinyGo!
```

## Standard Library Support

TinyGo supports much of Go's standard library:

**Working**:
- `fmt`
- `strings`
- `encoding/json`
- `encoding/xml`
- `math`
- `os` (with WASI)
- `time`
- `sync` (basic)

**Limited or not working**:
- `net` (limited, improving)
- `reflect` (partial)
- `runtime` (subset)
- `plugin`

### JSON Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "syscall/js"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func parseJSON(this js.Value, args []js.Value) interface{} {
    jsonStr := args[0].String()

    var person Person
    err := json.Unmarshal([]byte(jsonStr), &person)
    if err != nil {
        return map[string]interface{}{
            "error": err.Error(),
        }
    }

    return map[string]interface{}{
        "name": person.Name,
        "age":  person.Age,
    }
}

func main() {
    js.Global().Set("parseJSON", js.FuncOf(parseJSON))
    select {}
}
```

## Memory Efficiency

TinyGo uses a different garbage collector optimized for size:

```go
package main

import (
    "runtime"
    "syscall/js"
)

func getMemStats(this js.Value, args []js.Value) interface{} {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    return map[string]interface{}{
        "alloc":      m.Alloc,
        "totalAlloc": m.TotalAlloc,
        "sys":        m.Sys,
    }
}

func main() {
    js.Global().Set("getMemStats", js.FuncOf(getMemStats))
    select {}
}
```

## Optimization

### Size Optimization

```bash
# Already optimized by default
tinygo build -target wasm -o main.wasm main.go

# Further optimization
tinygo build -target wasm -no-debug -o main.wasm main.go

# Use wasm-opt
wasm-opt -Oz -o optimized.wasm main.wasm
```

### Performance Optimization

```bash
# Enable scheduler (for better goroutine performance)
tinygo build -target wasm -scheduler=asyncify -o main.wasm main.go
```

**Schedulers**:
- `none`: No scheduler (simplest, smallest)
- `tasks`: Cooperative scheduling (default)
- `asyncify`: Full scheduler using Binaryen asyncify

## Real-World Example: Game of Life

```go
package main

import (
    "syscall/js"
    "math/rand"
    "time"
)

type Grid struct {
    width, height int
    cells         []bool
}

func NewGrid(width, height int) *Grid {
    return &Grid{
        width:  width,
        height: height,
        cells:  make([]bool, width*height),
    }
}

func (g *Grid) Get(x, y int) bool {
    if x < 0 || x >= g.width || y < 0 || y >= g.height {
        return false
    }
    return g.cells[y*g.width+x]
}

func (g *Grid) Set(x, y int, alive bool) {
    if x >= 0 && x < g.width && y >= 0 && y < g.height {
        g.cells[y*g.width+x] = alive
    }
}

func (g *Grid) CountNeighbors(x, y int) int {
    count := 0
    for dy := -1; dy <= 1; dy++ {
        for dx := -1; dx <= 1; dx++ {
            if dx == 0 && dy == 0 {
                continue
            }
            if g.Get(x+dx, y+dy) {
                count++
            }
        }
    }
    return count
}

func (g *Grid) Step() {
    next := make([]bool, len(g.cells))

    for y := 0; y < g.height; y++ {
        for x := 0; x < g.width; x++ {
            neighbors := g.CountNeighbors(x, y)
            alive := g.Get(x, y)

            if alive && (neighbors == 2 || neighbors == 3) {
                next[y*g.width+x] = true
            } else if !alive && neighbors == 3 {
                next[y*g.width+x] = true
            }
        }
    }

    g.cells = next
}

func (g *Grid) Randomize() {
    for i := range g.cells {
        g.cells[i] = rand.Float32() < 0.3
    }
}

var grid *Grid

func init_game(this js.Value, args []js.Value) interface{} {
    width := args[0].Int()
    height := args[1].Int()

    rand.Seed(time.Now().UnixNano())
    grid = NewGrid(width, height)
    grid.Randomize()

    return nil
}

func step(this js.Value, args []js.Value) interface{} {
    if grid != nil {
        grid.Step()
    }
    return nil
}

func get_cell(this js.Value, args []js.Value) interface{} {
    if grid == nil {
        return false
    }

    x := args[0].Int()
    y := args[1].Int()

    return grid.Get(x, y)
}

func main() {
    js.Global().Set("init_game", js.FuncOf(init_game))
    js.Global().Set("step", js.FuncOf(step))
    js.Global().Set("get_cell", js.FuncOf(get_cell))

    println("Game of Life ready")
    select {}
}
```

**JavaScript/Canvas rendering**:
```javascript
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

const width = 100;
const height = 100;
const cellSize = 5;

canvas.width = width * cellSize;
canvas.height = height * cellSize;

// Initialize
init_game(width, height);

function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
            if (get_cell(x, y)) {
                ctx.fillStyle = 'black';
                ctx.fillRect(x * cellSize, y * cellSize, cellSize, cellSize);
            }
        }
    }
}

function gameLoop() {
    step();
    render();
    requestAnimationFrame(gameLoop);
}

gameLoop();
```

## CLI Tools with TinyGo

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Println("Usage: program <filename>")
        os.Exit(1)
    }

    filename := os.Args[1]

    file, err := os.Open(filename)
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    defer file.Close()

    // Count lines and words
    scanner := bufio.NewScanner(file)
    lines := 0
    words := 0

    for scanner.Scan() {
        lines++
        words += len(strings.Fields(scanner.Text()))
    }

    fmt.Printf("Lines: %d\n", lines)
    fmt.Printf("Words: %d\n", words)
}
```

**Compile and run**:
```bash
tinygo build -target=wasi -o wordcount.wasm wordcount.go
wasmtime --dir=. wordcount.wasm input.txt
```

## Limitations of TinyGo

### Language Features

**Not supported**:
- Full reflection (limited `reflect` package)
- `go://` imports
- Calling C code from Go (cgo)
- Some `unsafe` patterns
- Build tags (partial support)

**Partially supported**:
- `defer` (works but limited in some contexts)
- `panic`/`recover` (works but simpler than official Go)
- Closures (work but may have size overhead)

### Standard Library

Check compatibility:
```bash
tinygo build -target=wasm -size=short main.go
```

If a package doesn't compile, it's likely not supported.

## Debugging TinyGo WASM

### Enable Debugging Info

```bash
tinygo build -target wasm -o main.wasm main.go
# Debug info included by default

# Remove for smaller binary
tinygo build -target wasm -no-debug -o main.wasm main.go
```

### Print Debugging

```go
println("Debug:", value)  // Works without imports

// Or with fmt
import "fmt"
fmt.Println("Debug:", value)
```

### Stack Traces

TinyGo provides stack traces on panic:

```go
package main

func problematic() {
    panic("Something went wrong!")
}

func intermediate() {
    problematic()
}

func main() {
    intermediate()
}
```

Console shows the call stack.

## When to Use TinyGo

**Choose TinyGo when**:
- Binary size matters (web, embedded, edge)
- You need WASI support
- Faster compilation is important
- Your code uses a supported subset of Go

**Stick with official Go when**:
- You need full reflection
- You need the complete standard library
- You need cgo
- You need guaranteed compatibility

## Real-World Use Cases

### Fastly Compute@Edge
TinyGo is used for edge computing:
```go
package main

import (
    "fmt"
    "github.com/fastly/compute-sdk-go/fsthttp"
)

func main() {
    fsthttp.ServeFunc(func(ctx context.Context, w fsthttp.ResponseWriter, r *fsthttp.Request) {
        w.Header().Set("Content-Type", "text/plain")
        fmt.Fprintln(w, "Hello from TinyGo at the edge!")
    })
}
```

### Embedded Systems
TinyGo targets microcontrollers:
```go
package main

import (
    "machine"
    "time"
)

func main() {
    led := machine.LED
    led.Configure(machine.PinConfig{Mode: machine.PinOutput})

    for {
        led.Low()
        time.Sleep(time.Millisecond * 500)

        led.High()
        time.Sleep(time.Millisecond * 500)
    }
}
```

Compile for hardware:
```bash
tinygo flash -target=arduino-nano33 blink.go
```

## Next Steps

TinyGo brings Go to size-constrained environments with dramatically smaller binaries. It's the Go compiler of choice for WebAssembly when size matters—browsers, serverless, edge, embedded. Next, we'll explore C and C++ compilation to WASM via Emscripten, the most mature and comprehensive toolchain.
