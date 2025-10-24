# Chapter 47: Exception Handling

## The Exception Handling Proposal

WebAssembly's Exception Handling proposal adds first-class support for exceptions, enabling languages with exception semantics (C++, Java, Python, etc.) to compile more naturally and efficiently.

**Status**: Standardized (Phase 4)
**Browser Support**: Chrome 95+, Firefox 100+, Safari 15.2+

## Why Exception Handling?

### Before: Manual Error Handling

Languages had to use workarounds:

**C++ exceptions** → **Return codes**:
```cpp
// Native C++
try {
    riskyOperation();
} catch (const Exception& e) {
    handleError(e);
}

// Compiled to WASM (before proposal)
int error_code = riskyOperation();
if (error_code != 0) {
    handleError(error_code);
}
```

Problems:
- Changes semantics
- Performance overhead
- Complex transformation
- Loses type information

### After: Native Exceptions

```wat
(tag $error (param i32))  ;; Exception tag

(func $risky
  (try (result i32)
    (do
      ;; Try block
      (call $riskyOperation)
    )
    (catch $error
      ;; Catch block
      (call $handleError)
      (i32.const 0)
    )
  )
)
```

## Exception Tags

Tags define exception types:

```wat
;; Simple tag (no payload)
(tag $simple-error)

;; Tag with i32 payload
(tag $code-error (param i32))

;; Tag with multiple values
(tag $detailed-error (param i32 i32))

;; Tag with reference
(tag $object-error (param externref))
```

## Throwing Exceptions

```wat
(tag $error (param i32))

(func $divide (param $a i32) (param $b i32) (result i32)
  ;; Check for division by zero
  (if (i32.eqz (local.get $b))
    (then
      ;; Throw exception with error code
      (throw $error (i32.const 1))  ;; Error code 1 = div by zero
    )
  )

  (i32.div_s (local.get $a) (local.get $b))
)

(func $sqrt (param $x f64) (result f64)
  ;; Check for negative input
  (if (f64.lt (local.get $x) (f64.const 0))
    (then
      (throw $error (i32.const 2))  ;; Error code 2 = negative input
    )
  )

  ;; Calculate sqrt (simplified)
  (f64.sqrt (local.get $x))
)
```

## Catching Exceptions

### Single Catch

```wat
(tag $error (param i32))

(func $safe-divide (param $a i32) (param $b i32) (result i32)
  (try (result i32)
    (do
      (call $divide (local.get $a) (local.get $b))
    )
    (catch $error
      (drop)  ;; Drop error code
      (i32.const -1)  ;; Return -1 on error
    )
  )
)
```

### Multiple Catch Clauses

```wat
(tag $div-by-zero)
(tag $overflow)
(tag $underflow)

(func $compute (param $x i32) (result i32)
  (try (result i32)
    (do
      (call $complex-calculation (local.get $x))
    )
    (catch $div-by-zero
      (i32.const 0)  ;; Return 0
    )
    (catch $overflow
      (i32.const 2147483647)  ;; Return max i32
    )
    (catch $underflow
      (i32.const -2147483648)  ;; Return min i32
    )
  )
)
```

### Catch All

```wat
(func $robust-operation (result i32)
  (try (result i32)
    (do
      (call $might-throw-anything)
    )
    (catch_all
      (i32.const -1)  ;; Return -1 for any exception
    )
  )
)
```

### Re-throwing

```wat
(tag $error (param i32))

(func $handle-some-errors (result i32)
  (try (result i32)
    (do
      (call $risky-operation)
    )
    (catch $error
      (local.set $code)  ;; Get error code

      ;; Handle specific errors
      (if (i32.eq (local.get $code) (i32.const 1))
        (then
          ;; Handle error code 1
          (return (i32.const 0))
        )
      )

      ;; Re-throw unhandled errors
      (throw $error (local.get $code))
    )
  )
)
```

## Finally Blocks

Use `delegate` for cleanup:

```wat
(func $with-cleanup
  (try
    (do
      (call $acquire-resource)

      (try
        (do
          (call $use-resource)
        )
        (delegate 0)  ;; Propagate exceptions to outer try
      )

      (call $release-resource)
    )
    (catch_all
      (call $release-resource)  ;; Cleanup on exception
      (rethrow 0)  ;; Re-throw
    )
  )
)
```

## C++ Integration

### Compiling C++ Exceptions

**C++ code**:
```cpp
#include <stdexcept>
#include <iostream>

int divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("Division by zero");
    }
    return a / b;
}

int safe_divide(int a, int b) {
    try {
        return divide(a, b);
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return -1;
    }
}
```

**Compile with exceptions**:
```bash
em++ -fwasm-exceptions -O3 exceptions.cpp -o exceptions.js
```

**Generated WASM** (simplified):
```wat
(tag $cpp_exception (param externref))

(func $divide (param $a i32) (param $b i32) (result i32)
  (if (i32.eqz (local.get $b))
    (then
      ;; Create exception object
      (call $create_runtime_error)
      (throw $cpp_exception)
    )
  )

  (i32.div_s (local.get $a) (local.get $b))
)

(func $safe_divide (param $a i32) (param $b i32) (result i32)
  (try (result i32)
    (do
      (call $divide (local.get $a) (local.get $b))
    )
    (catch $cpp_exception
      ;; Handle exception
      (call $log_error)
      (drop)
      (i32.const -1)
    )
  )
)
```

### Exception Object Access

```cpp
class CustomException {
public:
    int code;
    std::string message;

    CustomException(int c, const std::string& m)
        : code(c), message(m) {}

    int getCode() const { return code; }
    std::string getMessage() const { return message; }
};

void process() {
    try {
        throw CustomException(42, "Something went wrong");
    } catch (const CustomException& e) {
        std::cout << "Code: " << e.getCode() << std::endl;
        std::cout << "Message: " << e.getMessage() << std::endl;
    }
}
```

## Rust with panic=abort

Rust doesn't use exceptions by default (uses `panic!`), but you can enable unwinding:

**Cargo.toml**:
```toml
[profile.dev]
panic = "unwind"

[profile.release]
panic = "unwind"
```

**Build**:
```bash
cargo build --target wasm32-unknown-unknown
```

**Note**: Most Rust WASM uses `panic=abort` for smaller binaries.

## JavaScript Interop

### Catching JS Exceptions in WASM

```wat
(import "js" "throwError" (func $js-throw))

(tag $js-exception (param externref))

(func $call-js
  (try
    (do
      (call $js-throw)
    )
    (catch $js-exception
      ;; Exception object on stack (externref)
      (call $log-exception)
      (drop)
    )
  )
)
```

**JavaScript**:
```javascript
const imports = {
    js: {
        throwError: () => {
            throw new Error("JavaScript error");
        }
    }
};

const { instance } = await WebAssembly.instantiate(wasmModule, imports);

try {
    instance.exports.call_js();
} catch (e) {
    // WASM exception propagated to JS
    console.error("Caught in JS:", e);
}
```

### Throwing WASM Exceptions to JS

```wat
(tag $wasm-error (param i32))
(func (export "risky") (param $x i32)
  (if (i32.lt_s (local.get $x) (i32.const 0))
    (then
      (throw $wasm-error (local.get $x))
    )
  )
)
```

```javascript
try {
    instance.exports.risky(-5);
} catch (e) {
    if (e instanceof WebAssembly.Exception) {
        console.log("WASM exception caught");
        console.log("Payload:", e.getArg(0));
    }
}
```

## Performance Implications

### Zero-Cost Exceptions

With native exception handling:

```cpp
// No overhead in happy path
int result = divide(10, 2);  // No exception checks

// Only pay when exception thrown
try {
    result = divide(10, 0);  // Throws
} catch (...) {
    // Handle
}
```

### vs Manual Error Codes

```cpp
// Manual approach: always check return codes
int error;
int result = divide_with_error(10, 2, &error);
if (error != 0) {
    // Handle error
}

// Every call has checking overhead!
```

## Exception Handling Patterns

### RAII (Resource Acquisition Is Initialization)

```cpp
class FileHandle {
    FILE* file;
public:
    FileHandle(const char* path) {
        file = fopen(path, "r");
        if (!file) {
            throw std::runtime_error("Failed to open file");
        }
    }

    ~FileHandle() {
        if (file) {
            fclose(file);
        }
    }

    void read() {
        if (!file) {
            throw std::runtime_error("File not open");
        }
        // Read from file
    }
};

void process_file() {
    FileHandle file("data.txt");  // Opens file
    file.read();
    // File automatically closed when leaving scope
    // Even if exception thrown!
}
```

### Custom Exception Hierarchy

```cpp
class BaseException : public std::exception {
protected:
    std::string msg;
public:
    BaseException(const std::string& m) : msg(m) {}
    const char* what() const noexcept override {
        return msg.c_str();
    }
};

class NetworkException : public BaseException {
public:
    NetworkException(const std::string& m)
        : BaseException("Network: " + m) {}
};

class DatabaseException : public BaseException {
public:
    DatabaseException(const std::string& m)
        : BaseException("Database: " + m) {}
};

void handle_errors() {
    try {
        risky_network_operation();
    } catch (const NetworkException& e) {
        // Handle network errors
    } catch (const DatabaseException& e) {
        // Handle database errors
    } catch (const BaseException& e) {
        // Handle other errors
    }
}
```

## Exception Safety Levels

### No-throw Guarantee

```cpp
// Never throws exceptions
void safe_operation() noexcept {
    // Implementation that can't fail
}
```

### Basic Exception Safety

```cpp
// If exception thrown, no resources leaked
void basic_safety() {
    auto ptr = std::make_unique<Data>();
    process(ptr.get());  // May throw
    // ptr automatically cleaned up
}
```

### Strong Exception Safety

```cpp
// If exception thrown, state unchanged
void strong_safety(std::vector<int>& vec, int value) {
    auto copy = vec;  // Make copy
    copy.push_back(value);  // May throw
    vec = std::move(copy);  // No-throw swap
}
```

### No-leak Guarantee

```cpp
class Container {
    Data* data;
public:
    Container() : data(new Data()) {}

    ~Container() {
        delete data;  // Always cleaned up
    }

    void process() {
        data->risky_operation();  // May throw
        // data still cleaned up by destructor
    }
};
```

## Debugging Exceptions

### Stack Traces

```cpp
#include <iostream>
#include <exception>
#include <execinfo.h>

void print_stack_trace() {
    void* array[10];
    size_t size = backtrace(array, 10);
    char** strings = backtrace_symbols(array, size);

    std::cerr << "Stack trace:" << std::endl;
    for (size_t i = 0; i < size; i++) {
        std::cerr << strings[i] << std::endl;
    }

    free(strings);
}

void function_c() {
    throw std::runtime_error("Error in C");
}

void function_b() {
    function_c();
}

void function_a() {
    try {
        function_b();
    } catch (const std::exception& e) {
        std::cerr << "Exception: " << e.what() << std::endl;
        print_stack_trace();
    }
}
```

### Browser DevTools

```javascript
try {
    instance.exports.risky_function();
} catch (e) {
    console.error("Exception caught:");
    console.error("Message:", e.message);
    console.error("Stack:", e.stack);

    if (e instanceof WebAssembly.Exception) {
        console.error("WASM exception, tag:", e.is(errorTag));
    }
}
```

## Best Practices

1. **Use RAII**: Automatic resource management
2. **Catch specific**: Don't catch everything
3. **Re-throw when appropriate**: Don't hide errors
4. **Document throws**: What exceptions can be thrown
5. **Minimize try scope**: Only wrap risky code
6. **Clean up resources**: Use destructors/finally
7. **Test exception paths**: Often under-tested
8. **Consider no-except**: Mark no-throw functions
9. **Profile**: Measure actual performance impact
10. **Use strong guarantees**: When possible

## Common Pitfalls

### Catching Everything

```cpp
// ✗ Bad: Hides all errors
try {
    critical_operation();
} catch (...) {
    // What went wrong? We don't know!
}

// ✓ Good: Specific handling
try {
    critical_operation();
} catch (const DatabaseException& e) {
    log_database_error(e);
    throw;  // Re-throw if can't handle
} catch (const NetworkException& e) {
    log_network_error(e);
    retry();
}
```

### Resource Leaks

```cpp
// ✗ Bad: Leaks on exception
void bad() {
    Data* ptr = new Data();
    risky_operation();  // May throw
    delete ptr;  // Never reached if exception!
}

// ✓ Good: RAII
void good() {
    std::unique_ptr<Data> ptr(new Data());
    risky_operation();  // Cleanup automatic
}
```

### Throwing in Destructors

```cpp
// ✗ Bad: Can cause double exception
class Bad {
public:
    ~Bad() {
        throw std::runtime_error("Oops");  // BAD!
    }
};

// ✓ Good: No-throw destructor
class Good {
public:
    ~Good() noexcept {
        try {
            cleanup();
        } catch (...) {
            // Log but don't throw
        }
    }
};
```

## Current Limitations

- **Binary size**: Exception tables add overhead
- **Performance**: Some overhead vs panic=abort
- **Interop**: Limited cross-language exception passing
- **Debugging**: Stack traces less detailed than native

## Future Improvements

Planned enhancements:
- **Stack trace**: Better stack trace support
- **Type information**: Richer exception metadata
- **Cross-language**: Standardized exception types
- **Zero-cost**: Further optimization

## Next Steps

WebAssembly's exception handling enables languages with exception semantics to compile naturally and efficiently. With first-class support for try/catch/finally and zero-cost exceptions, C++, Java, and other languages can use their native error handling patterns.

Next, we'll explore tail calls in WebAssembly, an optimization that enables efficient recursion and functional programming patterns.
