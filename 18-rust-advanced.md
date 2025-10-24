# Chapter 18: Rust Advanced - wasm-bindgen and wasm-pack

## wasm-bindgen: The Game Changer

wasm-bindgen transforms Rust WASM development by automatically generating JavaScript bindings. Instead of manual memory management and C-style FFI, you get:

- Automatic type conversions
- Rich JavaScript interop
- TypeScript definitions
- DOM access
- JavaScript classes in Rust

**Installation**:

```bash
cargo install wasm-bindgen-cli
```

## Basic wasm-bindgen

### Setup

**Cargo.toml**:

```toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

### Simple Example

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

**Build**:

```bash
cargo build --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/my_wasm_project.wasm \
    --out-dir pkg \
    --target web
```

**Generated files**:
- `pkg/my_wasm_project_bg.wasm` - The WASM module
- `pkg/my_wasm_project.js` - JavaScript glue code
- `pkg/my_wasm_project.d.ts` - TypeScript definitions

**Use in JavaScript**:

```javascript
import init, { greet, add } from './pkg/my_wasm_project.js';

async function run() {
    await init();  // Load WASM
    console.log(greet("World"));  // "Hello, World!"
    console.log(add(5, 3));       // 8
}

run();
```

## Type Conversions

wasm-bindgen handles type conversions automatically:

### Primitives

```rust
#[wasm_bindgen]
pub fn types_demo(
    a: i32,
    b: u32,
    c: f64,
    d: bool,
) -> (i32, u32, f64, bool) {
    (a, b, c, d)
}
```

JavaScript sees normal numbers and booleans.

### Strings

```rust
#[wasm_bindgen]
pub fn string_operations(input: &str) -> String {
    input.to_uppercase()
}

#[wasm_bindgen]
pub fn owns_string(input: String) -> usize {
    input.len()  // Consumes the string
}
```

Strings are automatically encoded/decoded as UTF-8.

### Vectors and Slices

```rust
#[wasm_bindgen]
pub fn sum_vec(numbers: Vec<i32>) -> i32 {
    numbers.iter().sum()
}

#[wasm_bindgen]
pub fn process_slice(data: &[u8]) -> Vec<u8> {
    data.iter().map(|x| x * 2).collect()
}
```

JavaScript arrays are converted to Rust vectors.

### Option and Result

```rust
#[wasm_bindgen]
pub fn find_index(arr: Vec<i32>, target: i32) -> Option<usize> {
    arr.iter().position(|&x| x == target)
}

#[wasm_bindgen]
pub fn safe_divide(a: f64, b: f64) -> Result<f64, JsValue> {
    if b == 0.0 {
        Err(JsValue::from_str("Division by zero"))
    } else {
        Ok(a / b)
    }
}
```

JavaScript:

```javascript
const index = find_index([1, 2, 3, 4], 3);
console.log(index);  // 2 or undefined

try {
    const result = safe_divide(10, 0);
} catch (e) {
    console.error(e);  // "Division by zero"
}
```

## JavaScript Interop

### Calling JavaScript from Rust

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    // Import JavaScript functions
    fn alert(s: &str);

    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    #[wasm_bindgen(js_namespace = Math)]
    fn random() -> f64;
}

#[wasm_bindgen]
pub fn demo_js_calls() {
    log("Hello from Rust!");
    let r = random();
    alert(&format!("Random: {}", r));
}
```

### Accessing JavaScript Objects

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    type MyJsClass;

    #[wasm_bindgen(constructor)]
    fn new() -> MyJsClass;

    #[wasm_bindgen(method)]
    fn doSomething(this: &MyJsClass, value: i32) -> i32;

    #[wasm_bindgen(method, getter)]
    fn value(this: &MyJsClass) -> i32;

    #[wasm_bindgen(method, setter)]
    fn set_value(this: &MyJsClass, value: i32);
}

#[wasm_bindgen]
pub fn use_js_class() -> i32 {
    let obj = MyJsClass::new();
    obj.set_value(42);
    obj.doSomething(10)
}
```

## DOM Access with web-sys

web-sys provides bindings for all Web APIs:

**Cargo.toml**:

```toml
[dependencies]
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = ["Document", "Element", "Window"] }
```

### Basic DOM Manipulation

```rust
use wasm_bindgen::prelude::*;
use web_sys::{Document, Element, Window};

#[wasm_bindgen]
pub fn create_element() -> Result<(), JsValue> {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();

    let div = document.create_element("div")?;
    div.set_inner_html("Hello from Rust!");
    div.set_attribute("id", "rust-content")?;

    let body = document.body().unwrap();
    body.append_child(&div)?;

    Ok(())
}
```

### Event Handling

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, HtmlElement};

#[wasm_bindgen]
pub fn add_click_handler() -> Result<(), JsValue> {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();

    let button = document
        .get_element_by_id("myButton")
        .unwrap()
        .dyn_into::<HtmlElement>()?;

    let closure = Closure::wrap(Box::new(move |_event: web_sys::MouseEvent| {
        web_sys::console::log_1(&"Button clicked!".into());
    }) as Box<dyn Fn(_)>);

    button.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;

    closure.forget();  // Keep closure alive

    Ok(())
}
```

### Canvas Drawing

```rust
use wasm_bindgen::prelude::*;
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

#[wasm_bindgen]
pub fn draw_on_canvas() -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document
        .get_element_by_id("myCanvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()?;

    let context = canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()?;

    context.begin_path();
    context.arc(75.0, 75.0, 50.0, 0.0, std::f64::consts::PI * 2.0)?;
    context.set_fill_style(&"#FF0000".into());
    context.fill();

    Ok(())
}
```

## Rust Classes in JavaScript

Export Rust structs as JavaScript classes:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Counter {
    value: i32,
}

#[wasm_bindgen]
impl Counter {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Counter {
        Counter { value: 0 }
    }

    pub fn increment(&mut self) {
        self.value += 1;
    }

    pub fn decrement(&mut self) {
        self.value -= 1;
    }

    pub fn value(&self) -> i32 {
        self.value
    }

    pub fn reset(&mut self) {
        self.value = 0;
    }
}
```

JavaScript:

```javascript
import { Counter } from './pkg/my_wasm_project.js';

const counter = new Counter();
counter.increment();
counter.increment();
console.log(counter.value());  // 2
counter.reset();
console.log(counter.value());  // 0
```

## wasm-pack: The Complete Workflow

wasm-pack streamlines Rust WASM development:

**Installation**:

```bash
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
```

### Project Setup

```bash
wasm-pack new my-project
cd my-project
```

**Structure**:
```
my-project/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── web.rs
```

### Building

```bash
# For web (ES modules)
wasm-pack build --target web

# For Node.js
wasm-pack build --target nodejs

# For bundlers (webpack, rollup)
wasm-pack build --target bundler

# For no-modules
wasm-pack build --target no-modules
```

### Publishing to npm

```bash
wasm-pack build --target bundler
wasm-pack publish
```

**Generates package.json**:

```json
{
  "name": "my-wasm-project",
  "version": "0.1.0",
  "files": [
    "my_wasm_project_bg.wasm",
    "my_wasm_project.js",
    "my_wasm_project.d.ts"
  ],
  "module": "my_wasm_project.js",
  "types": "my_wasm_project.d.ts"
}
```

## Advanced Patterns

### Async/Await

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_data(url: String) -> Result<String, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");

    let request = Request::new_with_str_and_init(&url, &opts)?;
    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;

    let resp: Response = resp_value.dyn_into()?;
    let text = JsFuture::from(resp.text()?).await?;

    Ok(text.as_string().unwrap())
}
```

JavaScript:

```javascript
const data = await fetch_data('https://api.example.com/data');
console.log(data);
```

### Closures

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn create_callback() -> Result<JsValue, JsValue> {
    let closure = Closure::wrap(Box::new(move || {
        web_sys::console::log_1(&"Callback invoked!".into());
    }) as Box<dyn Fn()>);

    let js_value = closure.as_ref().clone();
    closure.forget();  // Keep closure alive

    Ok(js_value)
}
```

### Shared Memory Between Rust and JavaScript

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct SharedBuffer {
    data: Vec<u8>,
}

#[wasm_bindgen]
impl SharedBuffer {
    #[wasm_bindgen(constructor)]
    pub fn new(size: usize) -> SharedBuffer {
        SharedBuffer {
            data: vec![0; size],
        }
    }

    pub fn get_buffer(&mut self) -> *mut u8 {
        self.data.as_mut_ptr()
    }

    pub fn get(&self, index: usize) -> u8 {
        self.data[index]
    }

    pub fn set(&mut self, index: usize, value: u8) {
        self.data[index] = value;
    }

    pub fn len(&self) -> usize {
        self.data.len()
    }
}
```

## Performance Optimization

### Avoid Unnecessary Copies

```rust
// Bad: copies string
#[wasm_bindgen]
pub fn process_string_bad(s: String) -> String {
    s.to_uppercase()
}

// Good: borrows string
#[wasm_bindgen]
pub fn process_string_good(s: &str) -> String {
    s.to_uppercase()
}
```

### Use js-sys for Zero-Copy Arrays

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Uint8Array};

#[wasm_bindgen]
pub fn process_typed_array(input: &Uint8Array) -> Uint8Array {
    let len = input.length() as usize;
    let mut data = vec![0u8; len];
    input.copy_to(&mut data);

    // Process data
    for byte in &mut data {
        *byte = byte.wrapping_mul(2);
    }

    // Return as typed array
    unsafe { Uint8Array::view(&data) }
}
```

### Batch Operations

```rust
// Bad: many calls from JS
#[wasm_bindgen]
pub fn process_one(x: i32) -> i32 {
    x * 2
}

// Good: one call, batch processing
#[wasm_bindgen]
pub fn process_many(arr: Vec<i32>) -> Vec<i32> {
    arr.into_iter().map(|x| x * 2).collect()
}
```

## Testing with wasm-bindgen-test

```rust
use wasm_bindgen_test::*;

#[wasm_bindgen_test]
fn test_add() {
    assert_eq!(add(2, 3), 5);
}

#[wasm_bindgen_test]
async fn test_async_function() {
    let result = fetch_data("test.json").await;
    assert!(result.is_ok());
}
```

Run tests:

```bash
wasm-pack test --headless --firefox
wasm-pack test --node
```

## Real-World Example: Image Processing

```rust
use wasm_bindgen::prelude::*;
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement, ImageData};

#[wasm_bindgen]
pub struct ImageProcessor {
    width: u32,
    height: u32,
    data: Vec<u8>,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> ImageProcessor {
        let size = (width * height * 4) as usize;
        ImageProcessor {
            width,
            height,
            data: vec![0; size],
        }
    }

    pub fn load_from_canvas(&mut self, canvas_id: &str) -> Result<(), JsValue> {
        let document = web_sys::window().unwrap().document().unwrap();
        let canvas = document
            .get_element_by_id(canvas_id)
            .unwrap()
            .dyn_into::<HtmlCanvasElement>()?;

        let context = canvas
            .get_context("2d")?
            .unwrap()
            .dyn_into::<CanvasRenderingContext2d>()?;

        let image_data = context.get_image_data(0.0, 0.0, self.width as f64, self.height as f64)?;
        self.data = image_data.data().to_vec();

        Ok(())
    }

    pub fn grayscale(&mut self) {
        for i in (0..self.data.len()).step_by(4) {
            let r = self.data[i] as f32;
            let g = self.data[i + 1] as f32;
            let b = self.data[i + 2] as f32;

            let gray = (0.299 * r + 0.587 * g + 0.114 * b) as u8;

            self.data[i] = gray;
            self.data[i + 1] = gray;
            self.data[i + 2] = gray;
        }
    }

    pub fn invert(&mut self) {
        for i in (0..self.data.len()).step_by(4) {
            self.data[i] = 255 - self.data[i];
            self.data[i + 1] = 255 - self.data[i + 1];
            self.data[i + 2] = 255 - self.data[i + 2];
        }
    }

    pub fn render_to_canvas(&self, canvas_id: &str) -> Result<(), JsValue> {
        let document = web_sys::window().unwrap().document().unwrap();
        let canvas = document
            .get_element_by_id(canvas_id)
            .unwrap()
            .dyn_into::<HtmlCanvasElement>()?;

        let context = canvas
            .get_context("2d")?
            .unwrap()
            .dyn_into::<CanvasRenderingContext2d>()?;

        let image_data = ImageData::new_with_u8_clamped_array_and_sh(
            wasm_bindgen::Clamped(&self.data),
            self.width,
            self.height,
        )?;

        context.put_image_data(&image_data, 0.0, 0.0)?;

        Ok(())
    }
}
```

JavaScript usage:

```javascript
import { ImageProcessor } from './pkg/my_wasm_project.js';

const processor = new ImageProcessor(800, 600);
processor.load_from_canvas('sourceCanvas');
processor.grayscale();
processor.render_to_canvas('outputCanvas');
```

## Best Practices

1. **Use wasm-pack**: Don't manually run wasm-bindgen
2. **Leverage TypeScript**: Generated .d.ts files catch errors
3. **Minimize JS↔WASM calls**: Batch operations
4. **Use typed arrays**: Zero-copy for large data
5. **Handle errors**: Use Result<T, JsValue>
6. **Profile**: Use browser DevTools to find bottlenecks
7. **Test in target environment**: wasm-bindgen-test in browsers
8. **Version carefully**: wasm-bindgen CLI and library must match

## Next Steps

wasm-bindgen and wasm-pack transform Rust WASM development from low-level FFI to high-level, type-safe JavaScript interop. With DOM access via web-sys, you can build complete web applications entirely in Rust. Next, we'll explore Go's approach to WebAssembly—both the official compiler and TinyGo.
