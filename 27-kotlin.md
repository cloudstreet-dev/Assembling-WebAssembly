# Chapter 27: Kotlin/Wasm

Kotlin/Wasm brings JetBrains' Kotlin language to WebAssembly, leveraging the GC proposal for efficient memory management. As of 2024, it's in Alpha but shows JetBrains' serious commitment to WASM as a first-class compilation target alongside Kotlin/JVM, Kotlin/JS, and Kotlin/Native.

## Why Kotlin/Wasm?

### The Multiplatform Vision

Kotlin's true power lies in its multiplatform capabilities. With Kotlin/Wasm, you can now share code across:

- **Android** (Kotlin/JVM)
- **iOS** (Kotlin/Native)
- **Web** (Kotlin/Wasm)
- **Backend** (Kotlin/JVM)
- **Desktop** (Kotlin/JVM or Kotlin/Native)

This is the dream: write business logic once, run it everywhere.

### WebAssembly GC Integration

Unlike Rust or C++, Kotlin/Wasm uses WebAssembly's GC proposal. This means:

- **No manual memory management**: Kotlin's garbage collector works naturally
- **Smaller binaries**: No need to bundle a GC implementation
- **Better JS interop**: Objects can be shared efficiently with JavaScript
- **Natural language semantics**: Kotlin code works as expected without WASM-specific constraints

## Status and Browser Support

**Current State**: Alpha (Kotlin 1.9.20+)
**Browser Support**:
- Chrome 119+ (GC proposal support)
- Firefox 120+
- Safari 17.4+
- Edge 119+

**Production Readiness**: Not yet recommended for production, but rapidly maturing.

## Project Setup

### Build Configuration

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.20"
}

repositories {
    mavenCentral()
}

kotlin {
    wasm {
        binaries.executable()
        browser {
            commonWebpackConfig {
                outputFileName = "app.js"
            }
        }
    }

    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
            }
        }

        val wasmMain by getting {
            dependencies {
                // Wasm-specific dependencies
            }
        }
    }
}
```

### Project Structure

```
my-kotlin-wasm-app/
├── build.gradle.kts
├── src/
│   ├── commonMain/
│   │   └── kotlin/
│   │       └── common/
│   │           └── BusinessLogic.kt    # Shared code
│   ├── wasmMain/
│   │   ├── kotlin/
│   │   │   └── Main.kt                # Wasm entry point
│   │   └── resources/
│   │       └── index.html
│   ├── jvmMain/                        # Optional: Android/backend
│   │   └── kotlin/
│   └── nativeMain/                     # Optional: iOS
│       └── kotlin/
└── gradle.properties
```

## Basic Example

### Hello World

```kotlin
// src/wasmMain/kotlin/Main.kt

@JsExport
fun main() {
    console.log("Hello from Kotlin/Wasm!")

    // Call exported function
    greet("WebAssembly")
}

@JsExport
fun add(a: Int, b: Int): Int = a + b

@JsExport
fun greet(name: String): String = "Hello, $name from Kotlin!"

@JsExport
fun fibonacci(n: Int): Long {
    return if (n < 2) n.toLong()
    else fibonacci(n - 1) + fibonacci(n - 2)
}
```

### HTML Integration

```html
<!-- src/wasmMain/resources/index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Kotlin/Wasm Demo</title>
</head>
<body>
    <h1>Kotlin/Wasm Example</h1>
    <div id="output"></div>

    <script type="module">
        import { instantiate } from './app.mjs';

        const { exports } = await instantiate();

        // Call Kotlin functions from JavaScript
        console.log('2 + 3 =', exports.add(2, 3));
        console.log(exports.greet('World'));
        console.log('Fib(10) =', exports.fibonacci(10));

        // Kotlin main() already executed during instantiation
    </script>
</body>
</html>
```

### Build and Run

```bash
# Development server with hot reload
./gradlew wasmBrowserRun

# Production build
./gradlew wasmBrowserDistribution

# Output in build/dist/wasmBrowser/productionExecutable/
```

## JavaScript Interop

### Calling JavaScript from Kotlin

```kotlin
// Define JavaScript API
external fun alert(message: String)

external object console {
    fun log(message: String)
    fun error(message: String)
    fun warn(message: String)
}

external object Math {
    fun random(): Double
    fun sqrt(x: Double): Double
    fun pow(base: Double, exponent: Double): Double
}

// DOM Access
external val document: Document
external val window: Window

external interface Document {
    fun getElementById(id: String): Element?
    fun createElement(tag: String): Element
    fun querySelector(selector: String): Element?
}

external interface Element {
    var innerHTML: String
    var textContent: String
    var className: String
    fun appendChild(child: Element): Element
    fun addEventListener(event: String, handler: () -> Unit)
}

external interface Window {
    fun setTimeout(handler: () -> Unit, timeout: Int): Int
    fun setInterval(handler: () -> Unit, interval: Int): Int
    fun fetch(url: String): JsPromise<Response>
}

external interface Response {
    fun text(): JsPromise<String>
    fun json(): JsPromise<JsAny>
}

// Use it
fun demo() {
    console.log("Starting demo")

    val randomValue = Math.random()
    console.log("Random: $randomValue")

    // DOM manipulation
    val div = document.createElement("div")
    div.textContent = "Created by Kotlin!"
    document.getElementById("output")?.appendChild(div)

    // Event handling
    div.addEventListener("click") {
        alert("Div clicked!")
    }
}
```

### Calling Kotlin from JavaScript

```kotlin
// Kotlin side
@JsExport
class Calculator {
    @JsExport
    fun add(a: Double, b: Double): Double = a + b

    @JsExport
    fun multiply(a: Double, b: Double): Double = a * b

    @JsExport
    fun power(base: Double, exp: Double): Double = Math.pow(base, exp)
}

@JsExport
data class Point(
    val x: Double,
    val y: Double
) {
    @JsExport
    fun distanceTo(other: Point): Double {
        val dx = x - other.x
        val dy = y - other.y
        return Math.sqrt(dx * dx + dy * dy)
    }
}
```

```javascript
// JavaScript side
import { Calculator, Point } from './app.mjs';

const calc = new Calculator();
console.log('2 + 3 =', calc.add(2, 3));
console.log('4 * 5 =', calc.multiply(4, 5));

const p1 = new Point(0.0, 0.0);
const p2 = new Point(3.0, 4.0);
console.log('Distance:', p1.distanceTo(p2));  // 5.0
```

## Kotlin Multiplatform Patterns

### Shared Business Logic

```kotlin
// src/commonMain/kotlin/common/User.kt
data class User(
    val id: String,
    val name: String,
    val email: String
)

// src/commonMain/kotlin/common/UserRepository.kt
interface UserRepository {
    suspend fun getUser(id: String): User?
    suspend fun saveUser(user: User)
    suspend fun listUsers(): List<User>
}

// src/commonMain/kotlin/common/UserService.kt
class UserService(private val repository: UserRepository) {
    suspend fun validateAndSave(user: User): Result<User> {
        return when {
            user.name.isBlank() -> Result.failure(Exception("Name required"))
            !user.email.contains("@") -> Result.failure(Exception("Invalid email"))
            else -> {
                repository.saveUser(user)
                Result.success(user)
            }
        }
    }

    suspend fun searchUsers(query: String): List<User> {
        return repository.listUsers().filter {
            it.name.contains(query, ignoreCase = true) ||
            it.email.contains(query, ignoreCase = true)
        }
    }
}
```

### Platform-Specific Implementation

```kotlin
// src/wasmMain/kotlin/UserRepositoryWasm.kt
class UserRepositoryWasm : UserRepository {
    private val storage = mutableMapOf<String, User>()

    override suspend fun getUser(id: String): User? {
        return storage[id]
    }

    override suspend fun saveUser(user: User) {
        storage[user.id] = user
        // Could save to localStorage via JS interop
        saveToLocalStorage(user.id, JSON.stringify(user))
    }

    override suspend fun listUsers(): List<User> {
        return storage.values.toList()
    }
}

external fun saveToLocalStorage(key: String, value: String)
external fun loadFromLocalStorage(key: String): String?

// Provide in JavaScript:
// window.saveToLocalStorage = (key, value) => localStorage.setItem(key, value);
// window.loadFromLocalStorage = (key) => localStorage.getItem(key);
```

## Coroutines Support

Kotlin's coroutines work seamlessly in WASM:

```kotlin
import kotlinx.coroutines.*

@JsExport
class AsyncDemo {
    private val scope = CoroutineScope(Dispatchers.Default)

    @JsExport
    fun startAsyncWork(callback: (String) -> Unit) {
        scope.launch {
            val result = performAsyncTask()
            callback(result)
        }
    }

    private suspend fun performAsyncTask(): String {
        delay(1000)  // Simulate async operation
        return "Task completed"
    }

    @JsExport
    fun fetchDataFromAPI(url: String, callback: (String) -> Unit) {
        scope.launch {
            try {
                val data = fetchData(url)
                callback("Success: $data")
            } catch (e: Exception) {
                callback("Error: ${e.message}")
            }
        }
    }

    private suspend fun fetchData(url: String): String = suspendCancellableCoroutine { continuation ->
        window.fetch(url)
            .then { response -> response.text() }
            .then { text -> continuation.resume(text) }
            .catch { error -> continuation.resumeWithException(Exception(error.toString())) }
    }
}
```

## Compose for Web (Experimental)

Kotlin's Jetpack Compose is being adapted for web via Kotlin/Wasm:

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.20"
    id("org.jetbrains.compose") version "1.5.10"
}

dependencies {
    implementation(compose.html.core)
    implementation(compose.runtime)
}
```

```kotlin
import androidx.compose.runtime.*
import org.jetbrains.compose.web.dom.*
import org.jetbrains.compose.web.css.*

@Composable
fun CounterApp() {
    var count by remember { mutableStateOf(0) }

    Div({
        style {
            padding(20.px)
            fontFamily("sans-serif")
        }
    }) {
        H1 { Text("Counter: $count") }

        Div({
            style {
                display(DisplayStyle.Flex)
                gap(10.px)
            }
        }) {
            Button(attrs = {
                onClick { count++ }
            }) {
                Text("Increment")
            }

            Button(attrs = {
                onClick { count-- }
            }) {
                Text("Decrement")
            }

            Button(attrs = {
                onClick { count = 0 }
            }) {
                Text("Reset")
            }
        }

        P {
            Text(when {
                count < 0 -> "Negative!"
                count == 0 -> "Zero"
                count > 10 -> "Getting high!"
                else -> "Positive"
            })
        }
    }
}

@Composable
fun TodoApp() {
    var todos by remember { mutableStateOf(listOf<String>()) }
    var input by remember { mutableStateOf("") }

    Div({
        style {
            maxWidth(600.px)
            margin(0.px, "auto")
            padding(20.px)
        }
    }) {
        H2 { Text("Todo List") }

        Div({
            style {
                display(DisplayStyle.Flex)
                gap(10.px)
                marginBottom(20.px)
            }
        }) {
            Input(type = InputType.Text, attrs = {
                value(input)
                onInput { input = it.value }
                placeholder("Enter todo...")
            })

            Button(attrs = {
                onClick {
                    if (input.isNotBlank()) {
                        todos = todos + input
                        input = ""
                    }
                }
            }) {
                Text("Add")
            }
        }

        Ul {
            todos.forEachIndexed { index, todo ->
                Li({
                    style {
                        marginBottom(10.px)
                    }
                }) {
                    Span { Text(todo) }
                    Button(attrs = {
                        onClick { todos = todos.filterIndexed { i, _ -> i != index } }
                        style {
                            marginLeft(10.px)
                        }
                    }) {
                        Text("Delete")
                    }
                }
            }
        }
    }
}

fun main() {
    renderComposable(rootElementId = "root") {
        CounterApp()
        TodoApp()
    }
}
```

## Advanced Example: Data Visualization

```kotlin
import kotlin.math.*

@JsExport
class DataProcessor {
    @JsExport
    data class Statistics(
        val mean: Double,
        val median: Double,
        val stdDev: Double,
        val min: Double,
        val max: Double
    )

    @JsExport
    fun calculateStatistics(data: Array<Double>): Statistics {
        val sorted = data.sorted()
        val mean = data.average()
        val median = if (sorted.size % 2 == 0) {
            (sorted[sorted.size / 2 - 1] + sorted[sorted.size / 2]) / 2
        } else {
            sorted[sorted.size / 2]
        }

        val variance = data.map { (it - mean).pow(2) }.average()
        val stdDev = sqrt(variance)

        return Statistics(
            mean = mean,
            median = median,
            stdDev = stdDev,
            min = sorted.first(),
            max = sorted.last()
        )
    }

    @JsExport
    fun generateHistogram(data: Array<Double>, bins: Int): Array<Int> {
        val min = data.minOrNull() ?: 0.0
        val max = data.maxOrNull() ?: 0.0
        val binWidth = (max - min) / bins

        val histogram = IntArray(bins)

        for (value in data) {
            val binIndex = min(((value - min) / binWidth).toInt(), bins - 1)
            histogram[binIndex]++
        }

        return histogram.toTypedArray()
    }

    @JsExport
    fun movingAverage(data: Array<Double>, windowSize: Int): Array<Double> {
        return data.indices.map { i ->
            val start = maxOf(0, i - windowSize + 1)
            val end = i + 1
            data.slice(start until end).average()
        }.toTypedArray()
    }
}
```

```javascript
// JavaScript usage
import { DataProcessor } from './app.mjs';

const processor = new DataProcessor();

// Generate sample data
const data = Array.from({ length: 1000 }, () => Math.random() * 100);

// Calculate statistics
const stats = processor.calculateStatistics(data);
console.log('Statistics:', stats);

// Generate histogram
const histogram = processor.generateHistogram(data, 10);
console.log('Histogram:', histogram);

// Calculate moving average
const smoothed = processor.movingAverage(data, 5);

// Visualize with Chart.js or similar
drawChart(smoothed);
```

## Performance Comparison

### Kotlin/Wasm vs Kotlin/JS

**Benchmark: Fibonacci (n=40)**

```kotlin
@JsExport
fun fibonacciRecursive(n: Int): Long {
    return if (n < 2) n.toLong()
    else fibonacciRecursive(n - 1) + fibonacciRecursive(n - 2)
}

@JsExport
fun fibonacciIterative(n: Int): Long {
    if (n < 2) return n.toLong()
    var a = 0L
    var b = 1L
    repeat(n - 1) {
        val temp = a + b
        a = b
        b = temp
    }
    return b
}
```

**Results**:
```
Fibonacci(40) recursive:
- Kotlin/Wasm: 850ms
- Kotlin/JS: 1200ms
- JavaScript: 1150ms
- Speedup: 1.4x over Kotlin/JS

Fibonacci(40) iterative:
- Kotlin/Wasm: 0.02ms
- Kotlin/JS: 0.08ms
- JavaScript: 0.05ms
- Speedup: 4x over Kotlin/JS
```

### Array Processing

```kotlin
@JsExport
fun processArray(size: Int): Double {
    val array = DoubleArray(size) { it.toDouble() }

    // Map-reduce operation
    return array
        .map { it * 2 }
        .map { sqrt(it) }
        .reduce { acc, d -> acc + d }
}
```

**Results (size=1,000,000)**:
```
- Kotlin/Wasm: 45ms
- Kotlin/JS: 120ms
- JavaScript: 95ms
- Speedup: 2.7x over Kotlin/JS
```

## GC Proposal Integration

Kotlin/Wasm uses WebAssembly's GC proposal, which provides:

### Struct Types

```wat
;; Generated by Kotlin/Wasm compiler
(type $User (struct
  (field $id (ref string))
  (field $name (ref string))
  (field $age i32)
))

(func $createUser (param $id (ref string)) (param $name (ref string)) (param $age i32)
                   (result (ref $User))
  (struct.new $User
    (local.get $id)
    (local.get $name)
    (local.get $age)
  )
)
```

### Benefits

1. **Smaller binaries**: No bundled GC (5-10% size reduction)
2. **Better JS interop**: Objects can be shared without serialization
3. **Native memory model**: Works like JVM/native Kotlin
4. **Faster allocation**: Browser's GC is highly optimized

## Production Deployment

### Build Optimization

```kotlin
// build.gradle.kts
kotlin {
    wasm {
        binaries.executable()
        browser {
            commonWebpackConfig {
                // Production optimizations
                mode = "production"

                devServer = devServer?.copy(
                    open = false,
                    port = 3000
                )
            }
        }

        // Enable compiler optimizations
        compilations.all {
            kotlinOptions {
                freeCompilerArgs += listOf(
                    "-Xopt-in=kotlin.ExperimentalStdlibApi",
                    "-Xopt-in=kotlinx.coroutines.ExperimentalCoroutinesApi"
                )
            }
        }
    }
}
```

### Size Optimization

```bash
# Production build
./gradlew wasmBrowserProductionWebpack

# Output analysis
ls -lh build/dist/wasmBrowser/productionExecutable/

# Typical sizes (optimized):
# - app.wasm: 150-500KB (depends on code)
# - app.js: 20-50KB (JS glue)
# - Gzipped: 40-150KB total
```

### CDN Deployment

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Kotlin/Wasm App</title>
</head>
<body>
    <div id="root"></div>

    <script type="module">
        // Load from CDN
        const wasmUrl = 'https://cdn.example.com/app.wasm';
        const module = await WebAssembly.compileStreaming(fetch(wasmUrl));

        // Import JS glue
        const { instantiate } = await import('https://cdn.example.com/app.mjs');
        const { exports } = await instantiate({ module });

        // App is now running
    </script>
</body>
</html>
```

## Comparison: Kotlin/Wasm vs Alternatives

### vs Kotlin/JS

**Kotlin/Wasm Advantages**:
- 1.5-3x faster execution
- Predictable performance
- Smaller bundle sizes (with GC proposal)
- Better for compute-heavy tasks

**Kotlin/JS Advantages**:
- More mature (production-ready)
- Better IDE support
- Larger ecosystem
- No browser version requirements

### vs Rust/WASM

**Kotlin Advantages**:
- Garbage collection (easier development)
- Multiplatform code sharing (Android/iOS/Web)
- Familiar JVM ecosystem
- Less steep learning curve

**Rust Advantages**:
- Better performance (2-3x faster)
- Smaller binaries
- No GC pauses
- More mature tooling

## Best Practices

### 1. Minimize JS Interop Overhead

```kotlin
// ✗ Bad: Frequent JS calls
@JsExport
fun processItems(items: Array<Item>) {
    items.forEach { item ->
        console.log("Processing: ${item.name}")  // JS call per item!
    }
}

// ✓ Good: Batch JS calls
@JsExport
fun processItems(items: Array<Item>) {
    val messages = items.map { "Processing: ${it.name}" }
    console.log(messages.joinToString("\n"))  // Single JS call
}
```

### 2. Use Coroutines for Async

```kotlin
// ✓ Good: Idiomatic Kotlin
suspend fun fetchUserData(userId: String): User {
    val response = window.fetch("/api/users/$userId").await()
    val json = response.json().await()
    return parseUser(json)
}
```

### 3. Share Code Effectively

```kotlin
// commonMain: Business logic
expect class PlatformStorage() {
    fun save(key: String, value: String)
    fun load(key: String): String?
}

// wasmMain: localStorage
actual class PlatformStorage {
    actual fun save(key: String, value: String) {
        window.localStorage.setItem(key, value)
    }

    actual fun load(key: String): String? {
        return window.localStorage.getItem(key)
    }
}
```

## Current Limitations

1. **Alpha status**: API may change
2. **Browser requirements**: Needs GC proposal support (Chrome 119+, Firefox 120+)
3. **No DOM library yet**: Must define JS interop manually
4. **Limited ecosystem**: Fewer WASM-specific libraries than Rust
5. **Larger than Rust**: Even with GC proposal, bigger than Rust WASM
6. **No WASI support yet**: Browser-only for now

## Future Roadmap

- **Compose for Web**: Full multiplatform UI framework
- **WASI support**: Enable server-side Kotlin/Wasm
- **Better tooling**: Improved debugging, profiling
- **Kotlin/Wasm libraries**: Growing ecosystem
- **Performance improvements**: Further optimizations
- **Stable release**: Move from Alpha to production-ready

## When to Use Kotlin/Wasm

**Consider Kotlin/Wasm if**:
- You have existing Kotlin codebase (Android/backend)
- You want multiplatform code sharing
- You prefer garbage collection over manual memory management
- You're building data-heavy applications
- You can target modern browsers

**Choose alternatives if**:
- You need maximum performance (use Rust)
- You need broad browser support (use Kotlin/JS)
- You need mature ecosystem (wait or use Kotlin/JS)
- Binary size is critical (use Rust or plain JS)

## Resources

- **Official docs**: https://kotlinlang.org/docs/wasm-overview.html
- **Compose for Web**: https://compose-web.ui.pages.jetbrains.team/
- **kotlin-wrappers**: https://github.com/JetBrains/kotlin-wrappers
- **Samples**: https://github.com/Kotlin/kotlin-wasm-examples
- **Slack**: #webassembly channel in Kotlin Slack

Kotlin/Wasm is evolving rapidly. As it matures and browsers adopt the GC proposal universally, expect Kotlin/Wasm to become a compelling choice for web development, especially for teams already invested in the Kotlin ecosystem. The combination of type safety, coroutines, multiplatform capabilities, and WebAssembly performance makes it a powerful option for modern web applications.

## Next Steps

In the next chapter, we'll explore Python in WebAssembly via Pyodide, seeing how dynamic languages can leverage WASM for browser deployment despite very different runtime characteristics from compiled languages like Kotlin.
