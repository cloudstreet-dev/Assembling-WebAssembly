# Chapter 16: Testing

## Why Test WASM?

WebAssembly modules should be tested as rigorously as any other code:

- **Bugs hide in compilation**: Issues can emerge during compilation to WASM
- **Cross-platform behavior**: WASM runs in many environments (browsers, Node, wasmtime, wasmer)
- **Interface contracts**: Imports and exports must work correctly
- **Performance regressions**: WASM optimizations can change behavior subtly
- **Security**: Verify sandboxing works as expected

Good testing practices ensure your WASM code works reliably everywhere.

## Testing Strategies

### Unit Testing

Test individual functions in isolation.

### Integration Testing

Test how WASM modules interact with the host environment.

### End-to-End Testing

Test complete workflows, including UI in browsers.

### Performance Testing

Benchmark and regression-test performance.

### Compatibility Testing

Test across different WASM runtimes and browsers.

## Testing Rust WASM

### wasm-bindgen-test

The standard tool for testing Rust WASM in browsers:

**Setup**:

```toml
[dev-dependencies]
wasm-bindgen-test = "0.3"
```

**Writing tests**:

```rust
use wasm_bindgen_test::*;

#[wasm_bindgen_test]
fn test_add() {
    assert_eq!(2 + 2, 4);
}

#[wasm_bindgen_test]
fn test_string_manipulation() {
    let s = String::from("hello");
    assert_eq!(s.to_uppercase(), "HELLO");
}
```

**Running tests**:

```bash
# In Node.js
wasm-pack test --node

# In browser (requires geckodriver or chromedriver)
wasm-pack test --headless --firefox
wasm-pack test --headless --chrome

# In both
wasm-pack test --node --headless --firefox
```

**Test output**:

```
running 2 tests
test test_add ... ok
test test_string_manipulation ... ok

test result: ok. 2 passed; 0 failed; 0 ignored
```

### Testing with wasm-bindgen Features

**Testing DOM interaction**:

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_test::*;
use web_sys::Document;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_dom() {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();

    let div = document.create_element("div").unwrap();
    div.set_inner_html("Hello, test!");

    assert_eq!(div.inner_html(), "Hello, test!");
}
```

**Async tests**:

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_test::*;
use wasm_bindgen_futures::JsFuture;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_fetch() {
    let window = web_sys::window().unwrap();
    let response = JsFuture::from(window.fetch_with_str("/api/data")).await.unwrap();

    assert!(response.is_instance_of::<web_sys::Response>());
}
```

### Native Rust Tests (for Non-Web Code)

Not all Rust code needs to run in WASM. Test native parts normally:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_pure_function() {
        // This runs natively, not in WASM
        assert_eq!(add(2, 3), 5);
    }
}
```

Run:

```bash
cargo test  # Native tests
```

### Conditional Compilation for Tests

```rust
#[cfg(target_arch = "wasm32")]
use wasm_bindgen::prelude::*;

pub fn platform_specific_function() -> String {
    #[cfg(target_arch = "wasm32")]
    {
        "Running in WASM".to_string()
    }

    #[cfg(not(target_arch = "wasm32"))]
    {
        "Running natively".to_string()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_platform() {
        let result = platform_specific_function();

        #[cfg(target_arch = "wasm32")]
        assert_eq!(result, "Running in WASM");

        #[cfg(not(target_arch = "wasm32"))]
        assert_eq!(result, "Running natively");
    }
}
```

## Testing C/C++ WASM

### Emscripten with Node.js

**Write tests** in C++:

```cpp
// test.cpp
#include <assert.h>
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main() {
    assert(add(2, 3) == 5);
    assert(add(-1, 1) == 0);
    printf("All tests passed!\n");
    return 0;
}
```

**Compile and run**:

```bash
emcc test.cpp -o test.js
node test.js
# All tests passed!
```

### Using a Test Framework

**Google Test with WASM**:

```cpp
// test.cpp
#include <gtest/gtest.h>

int add(int a, int b) {
    return a + b;
}

TEST(MathTest, Addition) {
    EXPECT_EQ(add(2, 3), 5);
    EXPECT_EQ(add(-1, 1), 0);
}

int main(int argc, char **argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

**Compile**:

```bash
emcc test.cpp -I/path/to/googletest/include \
     -L/path/to/googletest/lib -lgtest -lgtest_main \
     -o test.js

node test.js
# [==========] Running 1 test from 1 test suite.
# [----------] Global test environment set-up.
# [----------] 1 test from MathTest
# [ RUN      ] MathTest.Addition
# [       OK ] MathTest.Addition (0 ms)
# [----------] 1 test from MathTest (0 ms total)
# ...
```

### WASI Tests

For WASI targets:

```c
// test.c
#include <assert.h>
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main() {
    assert(add(2, 3) == 5);
    printf("Tests passed\n");
    return 0;
}
```

```bash
clang --target=wasm32-wasi -o test.wasm test.c
wasmtime test.wasm
# Tests passed
```

## Testing AssemblyScript

**Built-in test runner**:

```typescript
// module.spec.ts
import { add } from "./module";

describe("Math operations", () => {
  it("should add two numbers", () => {
    expect(add(2, 3)).toBe(5);
  });

  it("should handle negative numbers", () => {
    expect(add(-1, 1)).toBe(0);
  });
});
```

**Run tests**:

```bash
npm test
# or
npx asp --test
```

## Testing Go WASM

### TinyGo

**Write tests**:

```go
// math_test.go
package main

import "testing"

func TestAdd(t *testing.T) {
    result := add(2, 3)
    if result != 5 {
        t.Errorf("Expected 5, got %d", result)
    }
}

func TestAddNegative(t *testing.T) {
    result := add(-1, 1)
    if result != 0 {
        t.Errorf("Expected 0, got %d", result)
    }
}
```

**Run natively** (fast iteration):

```bash
go test ./...
```

**Run in WASM** (Node.js):

```bash
GOOS=js GOARCH=wasm go test
# or with TinyGo:
tinygo test -target=wasm
```

## Integration Testing with JavaScript

Test WASM modules from JavaScript:

### Jest

**Install**:

```bash
npm install --save-dev jest
```

**Test file** (test/module.test.js):

```javascript
const fs = require('fs');
const path = require('path');

describe('WASM Module', () => {
  let instance;

  beforeAll(async () => {
    const wasmPath = path.join(__dirname, '../build/module.wasm');
    const wasmBuffer = fs.readFileSync(wasmPath);
    const wasmModule = await WebAssembly.compile(wasmBuffer);
    instance = await WebAssembly.instantiate(wasmModule);
  });

  test('add function', () => {
    expect(instance.exports.add(2, 3)).toBe(5);
  });

  test('multiply function', () => {
    expect(instance.exports.multiply(4, 5)).toBe(20);
  });

  test('exported memory', () => {
    const memory = instance.exports.memory;
    expect(memory).toBeInstanceOf(WebAssembly.Memory);

    const view = new Uint8Array(memory.buffer);
    expect(view.length).toBeGreaterThan(0);
  });
});
```

**Run**:

```bash
npx jest
```

### Mocha

```javascript
const assert = require('assert');
const fs = require('fs');

describe('WASM Module', function() {
  let instance;

  before(async function() {
    const wasmBuffer = fs.readFileSync('./build/module.wasm');
    const wasmModule = await WebAssembly.compile(wasmBuffer);
    instance = await WebAssembly.instantiate(wasmModule);
  });

  it('should add numbers', function() {
    assert.strictEqual(instance.exports.add(2, 3), 5);
  });

  it('should handle large numbers', function() {
    assert.strictEqual(instance.exports.add(1000000, 2000000), 3000000);
  });
});
```

**Run**:

```bash
npx mocha
```

## Browser Testing

### Puppeteer

Automate browser testing:

**Install**:

```bash
npm install --save-dev puppeteer
```

**Test script**:

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();

  // Load page that uses WASM
  await page.goto('http://localhost:8080');

  // Wait for WASM to load
  await page.waitForFunction(() => window.wasmReady === true);

  // Test WASM exports
  const result = await page.evaluate(() => {
    return window.wasmInstance.exports.add(2, 3);
  });

  console.assert(result === 5, 'add(2, 3) should equal 5');

  await browser.close();
  console.log('All tests passed!');
})();
```

### Playwright

Cross-browser testing:

```javascript
const { test, expect } = require('@playwright/test');

test('WASM module works', async ({ page }) => {
  await page.goto('http://localhost:8080');

  // Wait for WASM
  await page.waitForFunction(() => window.wasmReady);

  // Call WASM function
  const result = await page.evaluate(() => {
    return window.wasmInstance.exports.add(10, 20);
  });

  expect(result).toBe(30);
});

test('WASM handles errors', async ({ page }) => {
  await page.goto('http://localhost:8080');
  await page.waitForFunction(() => window.wasmReady);

  // Test error handling
  const errorThrown = await page.evaluate(() => {
    try {
      window.wasmInstance.exports.divide(10, 0);
      return false;
    } catch (e) {
      return true;
    }
  });

  expect(errorThrown).toBe(true);
});
```

**Run across browsers**:

```bash
npx playwright test --project=chromium
npx playwright test --project=firefox
npx playwright test --project=webkit
```

## Testing WASI Applications

### Command-Line Testing

**Test script** (test.sh):

```bash
#!/bin/bash

# Build WASM
cargo build --target wasm32-wasi --release

# Test with different inputs
echo "Testing with positive input..."
echo "42" | wasmtime target/wasm32-wasi/release/program.wasm
if [ $? -ne 0 ]; then
    echo "Test failed!"
    exit 1
fi

echo "Testing with negative input..."
echo "-10" | wasmtime target/wasm32-wasi/release/program.wasm
if [ $? -eq 0 ]; then
    echo "Should have failed but didn't!"
    exit 1
fi

echo "All tests passed!"
```

### Testing File I/O

```rust
#[cfg(test)]
mod tests {
    use std::fs;
    use std::io::Write;

    #[test]
    fn test_file_processing() {
        // Create test input
        let mut file = fs::File::create("test_input.txt").unwrap();
        file.write_all(b"test data").unwrap();
        drop(file);

        // Run function
        super::process_file("test_input.txt", "test_output.txt").unwrap();

        // Verify output
        let output = fs::read_to_string("test_output.txt").unwrap();
        assert_eq!(output, "EXPECTED OUTPUT");

        // Cleanup
        fs::remove_file("test_input.txt").unwrap();
        fs::remove_file("test_output.txt").unwrap();
    }
}
```

Run with directory access:

```bash
cargo test
# or for WASI:
cargo build --target wasm32-wasi --release
wasmtime --dir=. target/wasm32-wasi/release/deps/myprogram-*.wasm
```

## Performance Testing

### Benchmarking in Rust

**Criterion.rs** (works with WASM via wasm-bindgen-test):

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

**Run**:

```bash
cargo bench
```

### JavaScript Performance Testing

```javascript
const { performance } = require('perf_hooks');

async function benchmarkWASM() {
  const fs = require('fs');
  const wasmBuffer = fs.readFileSync('./module.wasm');
  const { instance } = await WebAssembly.instantiate(wasmBuffer);

  const iterations = 1000000;

  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    instance.exports.add(i, i + 1);
  }
  const end = performance.now();

  const duration = end - start;
  const opsPerSecond = iterations / (duration / 1000);

  console.log(`Duration: ${duration.toFixed(2)}ms`);
  console.log(`Ops/sec: ${opsPerSecond.toFixed(0)}`);
}

benchmarkWASM();
```

### Regression Testing

Track performance over time:

```javascript
const fs = require('fs');

async function runBenchmark() {
  // ... run benchmark ...

  const result = {
    date: new Date().toISOString(),
    commit: process.env.GIT_COMMIT,
    opsPerSecond: opsPerSecond
  };

  // Append to results file
  let results = [];
  if (fs.existsSync('benchmark_results.json')) {
    results = JSON.parse(fs.readFileSync('benchmark_results.json'));
  }
  results.push(result);
  fs.writeFileSync('benchmark_results.json', JSON.stringify(results, null, 2));

  // Check for regression
  if (results.length > 1) {
    const previous = results[results.length - 2];
    const change = ((result.opsPerSecond - previous.opsPerSecond) / previous.opsPerSecond) * 100;

    if (change < -10) {
      console.error(`Performance regression: ${change.toFixed(2)}%`);
      process.exit(1);
    }
  }
}
```

## Cross-Runtime Testing

Test the same WASM module in multiple runtimes:

**test_all_runtimes.sh**:

```bash
#!/bin/bash

echo "Building WASM..."
cargo build --target wasm32-wasi --release
WASM=target/wasm32-wasi/release/program.wasm

echo "Testing with wasmtime..."
wasmtime $WASM
if [ $? -ne 0 ]; then echo "wasmtime failed!"; exit 1; fi

echo "Testing with wasmer..."
wasmer run $WASM
if [ $? -ne 0 ]; then echo "wasmer failed!"; exit 1; fi

echo "Testing with wasm3..."
wasm3 $WASM
if [ $? -ne 0 ]; then echo "wasm3 failed!"; exit 1; fi

echo "All runtimes passed!"
```

## Fuzzing

Test with random inputs to find edge cases:

### cargo-fuzz

```bash
cargo install cargo-fuzz

# Initialize fuzzing
cargo fuzz init

# Create fuzz target
cargo fuzz add fuzz_target_1
```

**fuzz/fuzz_targets/fuzz_target_1.rs**:

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if data.len() >= 8 {
        let a = i32::from_le_bytes([data[0], data[1], data[2], data[3]]);
        let b = i32::from_le_bytes([data[4], data[5], data[6], data[7]]);

        // Test function with random inputs
        let _ = my_crate::add(a, b);
    }
});
```

**Run**:

```bash
cargo fuzz run fuzz_target_1
```

## Continuous Integration

### GitHub Actions

**.github/workflows/test.yml**:

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          target: wasm32-unknown-unknown

      - name: Install wasm-pack
        run: curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

      - name: Run tests
        run: wasm-pack test --node --headless --firefox --chrome

      - name: Build release
        run: wasm-pack build --release

      - name: Test WASI
        run: |
          rustup target add wasm32-wasi
          cargo build --target wasm32-wasi --release
          curl https://wasmtime.dev/install.sh -sSf | bash
          export PATH="$HOME/.wasmtime/bin:$PATH"
          wasmtime target/wasm32-wasi/release/program.wasm
```

## Testing Best Practices

1. **Test at multiple levels**: Unit, integration, end-to-end
2. **Test in target environments**: Browser, Node, wasmtime, etc.
3. **Automate tests**: CI/CD pipeline
4. **Test error paths**: Not just happy paths
5. **Use assertions liberally**: Catch bugs early
6. **Test with realistic data**: Not just toy examples
7. **Benchmark regularly**: Catch performance regressions
8. **Test cross-platform**: Different browsers, runtimes, OSes
9. **Keep tests fast**: Quick feedback loop
10. **Document test requirements**: Make it easy for contributors

## Next Steps

Robust testing ensures your WASM code works correctly across environments. With unit tests, integration tests, browser automation, and performance benchmarks, you can ship WASM confidently.

We've completed Part 3 on the toolchain and ecosystem. Next, in Part 4, we'll dive deep into specific languages—starting with Rust, the language with arguably the best WASM support today.
