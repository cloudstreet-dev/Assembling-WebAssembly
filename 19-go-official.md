# Chapter 19: Go Official WASM Support

## Go and WebAssembly

Go added WebAssembly support in Go 1.11 (2018). The official Go compiler can target WASM, bringing Go's simplicity, goroutines, and standard library to the web.

**Key characteristics**:
- Full Go language support
- Goroutines work in WASM
- Comprehensive standard library
- Garbage collection
- But: large binary sizes and requires JavaScript glue code

## Compiling Go to WASM

### Basic Setup

**hello.go**:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, WebAssembly!")
}
```

**Compile**:

```bash
GOOS=js GOARCH=wasm go build -o main.wasm hello.go
```

**Required files**:

1. **main.wasm** - Your compiled WASM module
2. **wasm_exec.js** - Go's JavaScript support file

Get wasm_exec.js:

```bash
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .
```

### HTML Setup

**index.html**:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Go WASM</title>
</head>
<body>
    <script src="wasm_exec.js"></script>
    <script>
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject)
            .then((result) => {
                go.run(result.instance);
            });
    </script>
</body>
</html>
```

**Serve**:

```bash
# Simple server
python3 -m http.server 8080

# Or use goexec
go install github.com/shurcooL/goexec@latest
goexec 'http.ListenAndServe(`:8080`, http.FileServer(http.Dir(`.`)))'
```

Visit http://localhost:8080 and check the browser console for "Hello, WebAssembly!"

## Interacting with JavaScript

### Calling JavaScript from Go

```go
package main

import (
    "syscall/js"
)

func main() {
    // Get global objects
    window := js.Global()
    document := window.Get("document")

    // Alert
    window.Call("alert", "Hello from Go!")

    // Console.log
    console := window.Get("console")
    console.Call("log", "Logging from Go")

    // Create element
    div := document.Call("createElement", "div")
    div.Set("innerHTML", "Created by Go!")

    body := document.Get("body")
    body.Call("appendChild", div)

    // Prevent program from exiting
    select {}
}
```

### Exporting Functions to JavaScript

```go
package main

import (
    "syscall/js"
)

func add(this js.Value, args []js.Value) interface{} {
    return args[0].Int() + args[1].Int()
}

func multiply(this js.Value, args []js.Value) interface{} {
    return args[0].Int() * args[1].Int()
}

func greet(this js.Value, args []js.Value) interface{} {
    name := args[0].String()
    return "Hello, " + name + "!"
}

func main() {
    // Register functions
    js.Global().Set("add", js.FuncOf(add))
    js.Global().Set("multiply", js.FuncOf(multiply))
    js.Global().Set("greet", js.FuncOf(greet))

    println("Go functions registered")

    // Keep program running
    select {}
}
```

**JavaScript usage**:

```javascript
// After WASM loads
console.log(add(5, 3));          // 8
console.log(multiply(4, 7));     // 28
console.log(greet("World"));     // "Hello, World!"
```

## Working with the DOM

### DOM Manipulation

```go
package main

import (
    "strconv"
    "syscall/js"
)

func setupCounter() {
    document := js.Global().Get("document")

    // Create button
    button := document.Call("createElement", "button")
    button.Set("innerHTML", "Click me!")
    button.Set("id", "counterButton")

    // Create counter display
    counter := document.Call("createElement", "div")
    counter.Set("id", "counter")
    counter.Set("innerHTML", "Count: 0")

    // Append to body
    body := document.Get("body")
    body.Call("appendChild", button)
    body.Call("appendChild", counter)

    // Add click handler
    count := 0
    cb := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        count++
        counter.Set("innerHTML", "Count: "+strconv.Itoa(count))
        return nil
    })

    button.Call("addEventListener", "click", cb)
}

func main() {
    setupCounter()
    select {}  // Keep running
}
```

### Form Handling

```go
package main

import (
    "syscall/js"
)

func handleSubmit(this js.Value, args []js.Value) interface{} {
    event := args[0]
    event.Call("preventDefault")

    document := js.Global().Get("document")
    input := document.Call("getElementById", "nameInput")
    name := input.Get("value").String()

    output := document.Call("getElementById", "output")
    output.Set("innerHTML", "Hello, "+name+"!")

    return nil
}

func main() {
    document := js.Global().Get("document")

    form := document.Call("getElementById", "myForm")
    form.Call("addEventListener", "submit", js.FuncOf(handleSubmit))

    select {}
}
```

## Goroutines in WASM

Goroutines work in Go WASM:

```go
package main

import (
    "fmt"
    "syscall/js"
    "time"
)

func counter(id int, console js.Value) {
    for i := 0; i < 5; i++ {
        console.Call("log", fmt.Sprintf("Goroutine %d: %d", id, i))
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    console := js.Global().Get("console")

    console.Call("log", "Starting goroutines...")

    // Launch multiple goroutines
    for i := 0; i < 3; i++ {
        go counter(i, console)
    }

    // Wait for goroutines
    time.Sleep(3 * time.Second)
    console.Call("log", "Done!")

    select {}
}
```

Goroutines are implemented using the JavaScript event loop, not true threads.

## Callbacks and Promises

### Returning Promises

```go
package main

import (
    "syscall/js"
    "time"
)

func asyncOperation(this js.Value, args []js.Value) interface{} {
    handler := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        resolve := args[0]
        reject := args[1]

        go func() {
            // Simulate async work
            time.Sleep(2 * time.Second)

            // Resolve promise
            resolve.Invoke("Operation completed!")
        }()

        return nil
    })

    // Return Promise
    promiseConstructor := js.Global().Get("Promise")
    return promiseConstructor.New(handler)
}

func main() {
    js.Global().Set("asyncOperation", js.FuncOf(asyncOperation))
    select {}
}
```

**JavaScript**:

```javascript
asyncOperation().then(result => {
    console.log(result);  // "Operation completed!" after 2 seconds
});
```

### Handling JavaScript Promises

```go
package main

import (
    "fmt"
    "syscall/js"
)

func callAsyncJS() {
    // Get a promise from JavaScript
    promise := js.Global().Call("fetch", "https://api.example.com/data")

    // Handle promise
    then := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        response := args[0]
        fmt.Println("Fetch completed")

        // Get response body as text
        textPromise := response.Call("text")

        textThen := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
            text := args[0].String()
            fmt.Println("Body:", text)
            return nil
        })

        textPromise.Call("then", textThen)
        return nil
    })

    catchFunc := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        err := args[0]
        fmt.Println("Error:", err.String())
        return nil
    })

    promise.Call("then", then).Call("catch", catchFunc)
}

func main() {
    callAsyncJS()
    select {}
}
```

## Canvas and Graphics

```go
package main

import (
    "math"
    "syscall/js"
    "time"
)

func drawCircle(ctx js.Value, x, y, radius float64, color string) {
    ctx.Call("beginPath")
    ctx.Call("arc", x, y, radius, 0, 2*math.Pi)
    ctx.Set("fillStyle", color)
    ctx.Call("fill")
}

func animate() {
    document := js.Global().Get("document")
    canvas := document.Call("getElementById", "myCanvas")
    ctx := canvas.Call("getContext", "2d")

    width := canvas.Get("width").Float()
    height := canvas.Get("height").Float()

    x := width / 2
    y := height / 2
    angle := 0.0

    ticker := time.NewTicker(16 * time.Millisecond)  // ~60 FPS

    go func() {
        for range ticker.C {
            // Clear canvas
            ctx.Call("clearRect", 0, 0, width, height)

            // Calculate position
            offsetX := math.Cos(angle) * 100
            offsetY := math.Sin(angle) * 100

            // Draw circle
            drawCircle(ctx, x+offsetX, y+offsetY, 20, "blue")

            angle += 0.05
        }
    }()
}

func main() {
    animate()
    select {}
}
```

## Memory Management

### Accessing WASM Memory

Go's WASM memory is managed by the GC. You can share data:

```go
package main

import (
    "syscall/js"
    "unsafe"
)

func getBuffer(this js.Value, args []js.Value) interface{} {
    size := args[0].Int()

    // Allocate buffer
    data := make([]byte, size)

    // Fill with data
    for i := range data {
        data[i] = byte(i % 256)
    }

    // Create JS Uint8Array view
    uint8Array := js.Global().Get("Uint8Array")
    buffer := uint8Array.New(size)

    js.CopyBytesToJS(buffer, data)

    return buffer
}

func main() {
    js.Global().Set("getBuffer", js.FuncOf(getBuffer))
    select {}
}
```

### CopyBytesToGo and CopyBytesToJS

```go
package main

import (
    "syscall/js"
)

func processBuffer(this js.Value, args []js.Value) interface{} {
    input := args[0]  // Uint8Array from JavaScript

    size := input.Length()
    data := make([]byte, size)

    // Copy from JS to Go
    js.CopyBytesToGo(data, input)

    // Process data
    for i := range data {
        data[i] = data[i] * 2
    }

    // Copy back to JS
    output := js.Global().Get("Uint8Array").New(size)
    js.CopyBytesToJS(output, data)

    return output
}

func main() {
    js.Global().Set("processBuffer", js.FuncOf(processBuffer))
    select {}
}
```

## Standard Library Usage

Most of Go's standard library works:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "syscall/js"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func parseJSON(this js.Value, args []js.Value) interface{} {
    jsonStr := args[0].String()

    var user User
    err := json.Unmarshal([]byte(jsonStr), &user)
    if err != nil {
        return map[string]interface{}{
            "error": err.Error(),
        }
    }

    return map[string]interface{}{
        "id":   user.ID,
        "name": user.Name,
    }
}

func main() {
    js.Global().Set("parseJSON", js.FuncOf(parseJSON))
    select {}
}
```

**Limitations**:
- No `net` package (TCP/UDP sockets)
- No `os/exec` (process spawning)
- No threading (goroutines use JS event loop)
- File I/O doesn't work (no filesystem access)

## Size Considerations

Go WASM binaries are large:

```bash
# Minimal "Hello World"
GOOS=js GOARCH=wasm go build -o main.wasm hello.go
ls -lh main.wasm
# ~2-3 MB
```

**Optimization**:

```bash
# With optimization flags
GOOS=js GOARCH=wasm go build -ldflags="-s -w" -o main.wasm hello.go
# ~1.8 MB

# Further compression with wasm-opt
wasm-opt -Oz -o optimized.wasm main.wasm
# ~1.5 MB
```

**Why so large?**
- Go runtime included
- Garbage collector
- Standard library
- Reflection support

For smaller binaries, consider TinyGo (next chapter).

## Debugging Go WASM

### Console Logging

```go
package main

import (
    "fmt"
    "syscall/js"
)

func main() {
    console := js.Global().Get("console")

    // Different log levels
    console.Call("log", "Regular log")
    console.Call("warn", "Warning message")
    console.Call("error", "Error message")

    // Using fmt.Println
    fmt.Println("This goes to console.log too")
}
```

### Panic Handling

```go
package main

import (
    "syscall/js"
)

func mightPanic(this js.Value, args []js.Value) interface{} {
    defer func() {
        if r := recover(); r != nil {
            console := js.Global().Get("console")
            console.Call("error", "Panic recovered:", r)
        }
    }()

    // Risky operation
    panic("Something went wrong!")
}

func main() {
    js.Global().Set("mightPanic", js.FuncOf(mightPanic))
    select {}
}
```

## Performance Tips

1. **Minimize JS↔Go calls**: Crossing the boundary is expensive
2. **Batch operations**: Pass arrays instead of calling repeatedly
3. **Use typed arrays**: `js.CopyBytesToGo` is faster than element-by-element
4. **Avoid reflection**: It's included in the binary even if unused
5. **Profile with DevTools**: Use browser profiler to find hotspots

## Real-World Example: Image Processor

```go
package main

import (
    "syscall/js"
)

func grayscale(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    data := imageData.Get("data")

    length := data.Length()
    pixels := make([]byte, length)

    js.CopyBytesToGo(pixels, data)

    // Process pixels
    for i := 0; i < length; i += 4 {
        r := float32(pixels[i])
        g := float32(pixels[i+1])
        b := float32(pixels[i+2])

        gray := uint8(0.299*r + 0.587*g + 0.114*b)

        pixels[i] = gray
        pixels[i+1] = gray
        pixels[i+2] = gray
        // pixels[i+3] is alpha, leave unchanged
    }

    js.CopyBytesToJS(data, pixels)

    return imageData
}

func main() {
    js.Global().Set("grayscale", js.FuncOf(grayscale))

    console := js.Global().Get("console")
    console.Call("log", "Go image processor ready")

    select {}
}
```

**HTML/JavaScript**:

```html
<canvas id="canvas"></canvas>
<button onclick="applyGrayscale()">Grayscale</button>

<script>
async function applyGrayscale() {
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');

    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    grayscale(imageData);  // Go function
    ctx.putImageData(imageData, 0, 0);
}
</script>
```

## Limitations of Official Go WASM

1. **Large binary size**: 2-3 MB minimum
2. **No true threading**: Goroutines use JS event loop
3. **No networking**: `net` package doesn't work
4. **Slow compilation**: Go compiler isn't optimized for WASM
5. **GC pauses**: Can affect UI responsiveness

## When to Use Official Go WASM

**Good fit**:
- You have existing Go code to port
- You need goroutines
- Standard library is valuable
- Binary size isn't critical

**Not ideal**:
- Need smallest possible binary
- Targeting embedded/edge
- Performance-critical applications
- Need true threading

For better size and performance, see the next chapter on TinyGo.

## Next Steps

Official Go WASM support brings Go's simplicity and concurrency to the web, but with tradeoffs in binary size. It's production-ready and works well for porting existing Go applications. Next, we'll explore TinyGo, an alternative Go compiler that produces much smaller WASM binaries.
