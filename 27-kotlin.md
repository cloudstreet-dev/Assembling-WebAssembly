# Chapter 27: Kotlin/Wasm

Kotlin/Wasm brings JetBrains' Kotlin language to WebAssembly. Currently in Alpha, it shows JetBrains' commitment to WASM as a compilation target.

## Status and Features

**Current state**: Alpha (as of 2024)
**Target**: Browser-based applications
**Interop**: Excellent JavaScript interop via kotlin-wrappers

## Quick Start

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.20"
}

kotlin {
    wasm {
        binaries.executable()
        browser {}
    }

    sourceSets {
        val wasmMain by getting
    }
}
```

**Main.kt**:
```kotlin
fun main() {
    console.log("Hello from Kotlin/Wasm!")
}

@JsExport
fun add(a: Int, b: Int): Int = a + b

@JsExport
fun greet(name: String): String = "Hello, $name!"
```

**Build and run**:
```bash
./gradlew wasmBrowserRun
```

## JavaScript Interop

```kotlin
// Call JavaScript
external fun alert(message: String)
external val console: Console

external interface Console {
    fun log(message: String)
}

fun demo() {
    console.log("From Kotlin")
    alert("Kotlin says hi!")
}

// Access DOM
external val document: Document

external interface Document {
    fun getElementById(id: String): Element?
    fun createElement(tag: String): Element
}

external interface Element {
    var innerHTML: String
    fun appendChild(child: Element)
}

fun manipulateDOM() {
    val div = document.createElement("div")
    div.innerHTML = "Created by Kotlin!"
    document.getElementById("app")?.appendChild(div)
}
```

## Compose for Web

Kotlin's Jetpack Compose is being adapted for web via Kotlin/Wasm:

```kotlin
import androidx.compose.runtime.*
import org.jetbrains.compose.web.dom.*

@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Div {
        H1 { Text("Counter: $count") }

        Button(attrs = {
            onClick { count++ }
        }) {
            Text("Increment")
        }

        Button(attrs = {
            onClick { count = 0 }
        }) {
            Text("Reset")
        }
    }
}
```

## Coroutines Support

Kotlin's coroutines work in WASM:

```kotlin
import kotlinx.coroutines.*

suspend fun fetchData(url: String): String {
    delay(1000)  // Simulate async operation
    return "Data from $url"
}

fun main() {
    GlobalScope.launch {
        val data = fetchData("https://api.example.com")
        console.log(data)
    }
}
```

## When to Use Kotlin/Wasm

**Consider if**:
- Kotlin/JVM developer
- Want to share code with Android
- Multiplatform projects (Android/iOS/Web)

**Not yet ready for**:
- Production applications (still Alpha)
- Performance-critical code
- Small binary requirements

## Resources

- Official docs: https://kotlinlang.org/docs/wasm-overview.html
- Compose for Web: https://compose-web.ui.pages.jetbrains.team/
- kotlin-wrappers: https://github.com/JetBrains/kotlin-wrappers

Kotlin/Wasm is evolving rapidly. As it matures, expect improved performance, smaller binaries, and broader ecosystem support.
