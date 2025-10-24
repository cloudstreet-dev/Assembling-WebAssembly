# Chapter 26: Swift and SwiftWasm

## SwiftWasm Project

SwiftWasm is a port of the Swift toolchain that targets WebAssembly. It brings Swift's powerful features to the web and WASM environments.

**Status**: Experimental but functional
**Use cases**: Web applications, shared Swift codebases, iOS/web code sharing

## Installation

```bash
# Download SwiftWasm toolchain
# Visit https://book.swiftwasm.org/getting-started/setup.html

# Install via swiftenv
swiftenv install wasm-5.9-SNAPSHOT-2023-12-01-a
swiftenv global wasm-5.9-SNAPSHOT-2023-12-01-a

# Verify
swift --version
```

## Basic Compilation

**hello.swift**:
```swift
@_cdecl("add")
func add(_ a: Int32, _ b: Int32) -> Int32 {
    return a + b
}

@_cdecl("greet")
func greet() {
    print("Hello from Swift!")
}
```

**Compile**:
```bash
swiftc -target wasm32-unknown-wasi hello.swift -o hello.wasm
```

**Run**:
```bash
wasmtime hello.wasm
```

## JavaScript Interop with JavaScriptKit

JavaScriptKit enables Swift ↔ JavaScript communication:

**Package.swift**:
```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "MyWasmApp",
    products: [
        .executable(name: "MyWasmApp", targets: ["MyWasmApp"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftwasm/JavaScriptKit", from: "0.18.0")
    ],
    targets: [
        .executableTarget(
            name: "MyWasmApp",
            dependencies: [
                .product(name: "JavaScriptKit", package: "JavaScriptKit")
            ]
        )
    ]
)
```

**Sources/MyWasmApp/main.swift**:
```swift
import JavaScriptKit

let alert = JSObject.global.alert.function!
alert("Hello from Swift!")

let console = JSObject.global.console
console.log("Swift says hello")

// Access DOM
let document = JSObject.global.document
let div = document.createElement("div")
div.innerHTML = "Created by Swift!"
```

## Type Conversion

```swift
import JavaScriptKit

// Swift to JavaScript
let number: Int = 42
let jsNumber = JSValue(number)

let string: String = "Hello"
let jsString = JSValue(string)

let array: [Int] = [1, 2, 3]
let jsArray = JSValue(array)

// JavaScript to Swift
let jsVal = JSObject.global.someValue
if let num = jsVal.number {
    print("Number: \(num)")
}

if let str = jsVal.string {
    print("String: \(str)")
}
```

## Functions and Closures

```swift
import JavaScriptKit

// Export function to JavaScript
@_cdecl("swiftAdd")
func swiftAdd(_ a: Int32, _ b: Int32) -> Int32 {
    return a + b
}

// Callbacks
func setupCallback() {
    let button = JSObject.global.document.getElementById("myButton")
    let closure = JSClosure { _ in
        JSObject.global.console.log("Button clicked!")
        return .undefined
    }
    button.addEventListener("click", closure)
}

// Async operations
func fetchData() {
    let fetch = JSObject.global.fetch.function!
    let promise = fetch("https://api.example.com/data")

    let then = JSClosure { args in
        guard let response = args.first else { return .undefined }
        JSObject.global.console.log("Fetch complete:", response)
        return .undefined
    }

    _ = promise.then(then)
}
```

## Swift Features in WASM

### Optionals

```swift
@_cdecl("safeDivide")
func safeDivide(_ a: Int32, _ b: Int32) -> Int32 {
    guard b != 0 else {
        return -1  // Error sentinel
    }
    return a / b
}

func findFirst(_ array: [Int], matching condition: (Int) -> Bool) -> Int? {
    return array.first(where: condition)
}
```

### Protocols

```swift
protocol Shape {
    func area() -> Double
}

struct Circle: Shape {
    let radius: Double

    func area() -> Double {
        return Double.pi * radius * radius
    }
}

struct Rectangle: Shape {
    let width: Double
    let height: Double

    func area() -> Double {
        return width * height
    }
}

@_cdecl("calculateArea")
func calculateArea(_ shapeType: Int32, _ param1: Double, _ param2: Double) -> Double {
    let shape: Shape
    if shapeType == 0 {
        shape = Circle(radius: param1)
    } else {
        shape = Rectangle(width: param1, height: param2)
    }
    return shape.area()
}
```

### Generics

```swift
func swap<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

struct Stack<Element> {
    private var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        return items.isEmpty ? nil : items.removeLast()
    }
}
```

## Memory Management (ARC)

Swift uses Automatic Reference Counting:

```swift
class Counter {
    var value: Int = 0

    func increment() {
        value += 1
    }

    func getValue() -> Int {
        return value
    }

    deinit {
        print("Counter deallocated")
    }
}

@_cdecl("createCounter")
func createCounter() -> UnsafeMutableRawPointer {
    let counter = Counter()
    return Unmanaged.passRetained(counter).toOpaque()
}

@_cdecl("useCounter")
func useCounter(_ ptr: UnsafeMutableRawPointer) -> Int32 {
    let counter = Unmanaged<Counter>.fromOpaque(ptr).takeUnretainedValue()
    counter.increment()
    return Int32(counter.getValue())
}

@_cdecl("releaseCounter")
func releaseCounter(_ ptr: UnsafeMutableRawPointer) {
    Unmanaged<Counter>.fromOpaque(ptr).release()
}
```

## Building a Web App

**Simple web app structure**:

```swift
import JavaScriptKit

let document = JSObject.global.document

func setupUI() {
    // Create elements
    let container = document.createElement("div")
    container.setAttribute("id", "app")

    let title = document.createElement("h1")
    title.innerHTML = "Swift WebAssembly App"

    let button = document.createElement("button")
    button.innerHTML = "Click Me"

    // Add click handler
    let clickHandler = JSClosure { _ in
        JSObject.global.alert("Hello from Swift!")
        return .undefined
    }
    button.addEventListener("click", clickHandler)

    // Build DOM
    container.appendChild(title)
    container.appendChild(button)

    let body = document.body
    body.appendChild(container)
}

setupUI()
```

## Tokamak - SwiftUI for Web

Tokamak is a SwiftUI-compatible framework for SwiftWasm:

**Package.swift**:
```swift
dependencies: [
    .package(url: "https://github.com/TokamakUI/Tokamak", from: "0.11.0")
]
```

**App.swift**:
```swift
import TokamakDOM

@main
struct App: App {
    var body: some Scene {
        WindowGroup("Tokamak App") {
            ContentView()
        }
    }
}

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
                .font(.largeTitle)

            Button("Increment") {
                count += 1
            }

            Button("Reset") {
                count = 0
            }
        }
        .padding()
    }
}
```

This looks like SwiftUI but renders to DOM!

## Performance Considerations

Swift WASM is still experimental. Performance tips:

1. **Avoid excessive JS interop**: Crossing boundaries is expensive
2. **Use value types**: Structs are faster than classes
3. **Minimize allocations**: Reuse objects when possible
4. **Profile**: Use browser DevTools

## Limitations

- **Experimental**: Not production-ready
- **Binary size**: Larger than Rust/C++ (includes Swift runtime)
- **Limited ecosystem**: Fewer WASM-specific libraries
- **Threading**: No thread support yet
- **Incomplete stdlib**: Some features unavailable

## Use Cases

**Good fit**:
- Sharing code between iOS and web
- Swift developers exploring web
- Proof-of-concept projects

**Not recommended**:
- Production web apps (use mature toolchains)
- Performance-critical applications
- Small binary size requirements

## Building and Deploying

```bash
# Build
swift build --triple wasm32-unknown-wasi

# Find output
ls .build/wasm32-unknown-wasi/debug/

# Serve
python3 -m http.server
```

## Future of Swift WASM

SwiftWasm continues to evolve:
- Better optimization
- Smaller binaries
- More complete standard library
- Improved JavaScript interop
- Potential official Swift support

## Next Steps

Swift WebAssembly brings Apple's language to the web, enabling code sharing between iOS and web applications. While still experimental, it shows promise for Swift developers wanting to target WebAssembly. Next, we'll explore Kotlin/Wasm, JetBrains' approach to bringing Kotlin to WebAssembly.
