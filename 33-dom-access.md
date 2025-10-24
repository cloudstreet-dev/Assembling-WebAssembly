# Chapter 33: DOM Access from WebAssembly

## Why Direct DOM Access?

While JavaScript can manipulate the DOM on behalf of WASM, direct DOM access from WASM can improve performance for UI-heavy applications by reducing boundary crossings.

## Approaches to DOM Access

### 1. JavaScript Imports (Standard)

Import JS functions that manipulate DOM:

```javascript
const imports = {
    dom: {
        createElement: (tagPtr) => {
            const tag = readString(memory, tagPtr);
            const element = document.createElement(tag);
            return registerElement(element);  // Return handle
        },

        setInnerHTML: (elementHandle, htmlPtr) => {
            const element = getElement(elementHandle);
            const html = readString(memory, htmlPtr);
            element.innerHTML = html;
        },

        appendChild: (parentHandle, childHandle) => {
            const parent = getElement(parentHandle);
            const child = getElement(childHandle);
            parent.appendChild(child);
        },

        getElementById: (idPtr) => {
            const id = readString(memory, idPtr);
            const element = document.getElementById(id);
            return registerElement(element);
        }
    }
};
```

### 2. web-sys (Rust)

Rust's `web-sys` crate provides comprehensive DOM bindings:

```rust
use web_sys::{Document, Element, HtmlElement, window};
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn create_ui() -> Result<(), JsValue> {
    let window = window().unwrap();
    let document = window.document().unwrap();

    // Create elements
    let container = document.create_element("div")?;
    container.set_class_name("container");

    let title = document.create_element("h1")?;
    title.set_inner_html("WebAssembly DOM Demo");

    let button = document.create_element("button")?;
    button.set_inner_html("Click Me");

    // Build tree
    container.append_child(&title)?;
    container.append_child(&button)?;

    // Add to document
    let body = document.body().unwrap();
    body.append_child(&container)?;

    Ok(())
}
```

### 3. Custom Bindings (C/Emscripten)

Emscripten provides `emscripten/html5.h`:

```c
#include <emscripten/html5.h>
#include <string.h>

EM_JS(void, create_element_js, (const char* tag), {
    const tagName = UTF8ToString(tag);
    const element = document.createElement(tagName);
    document.body.appendChild(element);
});

void create_ui() {
    create_element_js("div");
}
```

## Building a UI from WASM

### Complete Example (Rust + web-sys)

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, Element, HtmlInputElement, HtmlButtonElement, window};

struct App {
    document: Document,
    container: Element,
    input: HtmlInputElement,
    output: Element,
}

impl App {
    fn new() -> Result<Self, JsValue> {
        let window = window().unwrap();
        let document = window.document().unwrap();

        let container = document.create_element("div")?;
        container.set_class_name("app");

        let input: HtmlInputElement = document
            .create_element("input")?
            .dyn_into()?;
        input.set_placeholder("Enter text...");

        let button: HtmlButtonElement = document
            .create_element("button")?
            .dyn_into()?;
        button.set_inner_text("Process");

        let output = document.create_element("div")?;
        output.set_class_name("output");

        container.append_child(&input)?;
        container.append_child(&button)?;
        container.append_child(&output)?;

        document.body().unwrap().append_child(&container)?;

        Ok(App {
            document,
            container,
            input,
            output,
        })
    }

    fn process(&self) {
        let text = self.input.value();
        let processed = text.to_uppercase();
        self.output.set_inner_html(&processed);
    }
}

#[wasm_bindgen(start)]
pub fn main() {
    let app = App::new().unwrap();

    // Set up event handler
    let button = app.container
        .query_selector("button")
        .unwrap()
        .unwrap()
        .dyn_into::<HtmlButtonElement>()
        .unwrap();

    let callback = Closure::wrap(Box::new(move || {
        app.process();
    }) as Box<dyn Fn()>);

    button.set_onclick(Some(callback.as_ref().unchecked_ref()));
    callback.forget();
}
```

## Event Handling

### Mouse Events

```rust
use web_sys::{MouseEvent, HtmlElement};

#[wasm_bindgen]
pub fn setup_click_handler() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let button = document
        .get_element_by_id("myButton")
        .unwrap()
        .dyn_into::<HtmlElement>()?;

    let closure = Closure::wrap(Box::new(move |event: MouseEvent| {
        web_sys::console::log_1(&format!(
            "Clicked at ({}, {})",
            event.client_x(),
            event.client_y()
        ).into());
    }) as Box<dyn Fn(MouseEvent)>);

    button.set_onclick(Some(closure.as_ref().unchecked_ref()));
    closure.forget();  // Keep alive

    Ok(())
}
```

### Keyboard Events

```rust
use web_sys::KeyboardEvent;

#[wasm_bindgen]
pub fn setup_keyboard() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();

    let closure = Closure::wrap(Box::new(move |event: KeyboardEvent| {
        let key = event.key();
        web_sys::console::log_1(&format!("Key pressed: {}", key).into());

        if key == "Enter" {
            // Handle Enter key
        }
    }) as Box<dyn Fn(KeyboardEvent)>);

    document.set_onkeydown(Some(closure.as_ref().unchecked_ref()));
    closure.forget();

    Ok(())
}
```

### Form Events

```rust
use web_sys::{HtmlFormElement, FormData, Event};

#[wasm_bindgen]
pub fn setup_form() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let form = document
        .get_element_by_id("myForm")
        .unwrap()
        .dyn_into::<HtmlFormElement>()?;

    let closure = Closure::wrap(Box::new(move |event: Event| {
        event.prevent_default();

        let form = event
            .target()
            .unwrap()
            .dyn_into::<HtmlFormElement>()
            .unwrap();

        let form_data = FormData::new_with_form(&form).unwrap();

        // Access form fields
        for entry in form_data.entries().into_iter() {
            let entry = entry.unwrap();
            let key = entry.get(0).as_string().unwrap();
            let value = entry.get(1).as_string().unwrap();
            web_sys::console::log_1(&format!("{}: {}", key, value).into());
        }
    }) as Box<dyn Fn(Event)>);

    form.add_event_listener_with_callback(
        "submit",
        closure.as_ref().unchecked_ref()
    )?;
    closure.forget();

    Ok(())
}
```

## Canvas Manipulation

### 2D Canvas

```rust
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

#[wasm_bindgen]
pub fn draw_on_canvas() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let canvas = document
        .get_element_by_id("canvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()?;

    let context = canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()?;

    // Clear canvas
    context.clear_rect(0.0, 0.0, canvas.width().into(), canvas.height().into());

    // Draw shapes
    context.set_fill_style(&"red".into());
    context.fill_rect(10.0, 10.0, 100.0, 100.0);

    context.set_stroke_style(&"blue".into());
    context.set_line_width(5.0);
    context.stroke_rect(150.0, 10.0, 100.0, 100.0);

    // Draw circle
    context.begin_path();
    context.arc(200.0, 200.0, 50.0, 0.0, std::f64::consts::PI * 2.0)?;
    context.set_fill_style(&"green".into());
    context.fill();

    // Draw text
    context.set_font("24px Arial");
    context.set_fill_style(&"black".into());
    context.fill_text("Hello from WASM!", 10.0, 300.0)?;

    Ok(())
}
```

### Image Processing

```rust
use web_sys::{ImageData, CanvasRenderingContext2d};
use wasm_bindgen::Clamped;

#[wasm_bindgen]
pub fn apply_grayscale(context: &CanvasRenderingContext2d) -> Result<(), JsValue> {
    let width = context.canvas().unwrap().width();
    let height = context.canvas().unwrap().height();

    let image_data = context.get_image_data(
        0.0, 0.0, width as f64, height as f64
    )?;

    let mut data = image_data.data().to_vec();

    // Apply grayscale
    for i in (0..data.len()).step_by(4) {
        let r = data[i] as f32;
        let g = data[i + 1] as f32;
        let b = data[i + 2] as f32;

        let gray = (0.299 * r + 0.587 * g + 0.114 * b) as u8;

        data[i] = gray;
        data[i + 1] = gray;
        data[i + 2] = gray;
        // Alpha unchanged
    }

    let new_image_data = ImageData::new_with_u8_clamped_array_and_sh(
        Clamped(&data),
        width,
        height,
    )?;

    context.put_image_data(&new_image_data, 0.0, 0.0)?;

    Ok(())
}
```

## Working with CSS

### Adding Styles

```rust
#[wasm_bindgen]
pub fn apply_styles() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let element = document.get_element_by_id("myDiv").unwrap();

    // Direct style manipulation
    let style = element.dyn_ref::<HtmlElement>().unwrap().style();
    style.set_property("background-color", "lightblue")?;
    style.set_property("padding", "20px")?;
    style.set_property("border-radius", "10px")?;

    // Add CSS class
    element.set_class_name("styled-element");

    Ok(())
}
```

### CSS Animations

```rust
#[wasm_bindgen]
pub fn animate_element() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let element = document.get_element_by_id("animated").unwrap();

    let style = element.dyn_ref::<HtmlElement>().unwrap().style();
    style.set_property("transition", "all 0.5s ease")?;
    style.set_property("transform", "translateX(200px)")?;

    Ok(())
}
```

## Local Storage

```rust
use web_sys::Storage;

#[wasm_bindgen]
pub fn use_local_storage() -> Result<(), JsValue> {
    let window = window().unwrap();
    let storage = window.local_storage()?.unwrap();

    // Set item
    storage.set_item("username", "Alice")?;
    storage.set_item("score", "42")?;

    // Get item
    if let Some(username) = storage.get_item("username")? {
        web_sys::console::log_1(&format!("User: {}", username).into());
    }

    // Remove item
    storage.remove_item("score")?;

    // Clear all
    // storage.clear()?;

    Ok(())
}
```

## Fetch API

```rust
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_data(url: String) -> Result<String, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");

    let request = Request::new_with_str_and_init(&url, &opts)?;

    let window = window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;

    let text = JsFuture::from(resp.text()?).await?;

    Ok(text.as_string().unwrap())
}
```

## Timers

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[wasm_bindgen]
pub fn setup_timer() -> Result<(), JsValue> {
    let window = window().unwrap();

    // setTimeout
    let callback = Closure::wrap(Box::new(|| {
        web_sys::console::log_1(&"Timer fired!".into());
    }) as Box<dyn Fn()>);

    window.set_timeout_with_callback_and_timeout_and_arguments_0(
        callback.as_ref().unchecked_ref(),
        1000  // 1 second
    )?;

    callback.forget();

    // setInterval
    let counter = Rc::new(RefCell::new(0));
    let counter_clone = counter.clone();

    let interval_callback = Closure::wrap(Box::new(move || {
        let mut count = counter_clone.borrow_mut();
        *count += 1;
        web_sys::console::log_1(&format!("Interval: {}", *count).into());
    }) as Box<dyn Fn()>);

    window.set_interval_with_callback_and_timeout_and_arguments_0(
        interval_callback.as_ref().unchecked_ref(),
        500  // Every 500ms
    )?;

    interval_callback.forget();

    Ok(())
}
```

## Performance Considerations

### Batch DOM Operations

```rust
// ✗ Bad: individual operations cause reflows
for i in 0..100 {
    let element = document.create_element("div")?;
    element.set_inner_html(&format!("Item {}", i));
    container.append_child(&element)?;
}

// ✓ Good: build fragment, append once
let fragment = document.create_document_fragment();
for i in 0..100 {
    let element = document.create_element("div")?;
    element.set_inner_html(&format!("Item {}", i));
    fragment.append_child(&element)?;
}
container.append_child(&fragment)?;  // Single reflow
```

### Use RequestAnimationFrame

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

#[wasm_bindgen]
pub fn animate() {
    let f = Rc::new(RefCell::new(None));
    let g = f.clone();

    *g.borrow_mut() = Some(Closure::wrap(Box::new(move || {
        // Animation logic here

        // Request next frame
        request_animation_frame(f.borrow().as_ref().unwrap());
    }) as Box<dyn FnMut()>));

    request_animation_frame(g.borrow().as_ref().unwrap());
}

fn request_animation_frame(f: &Closure<dyn FnMut()>) {
    window()
        .unwrap()
        .request_animation_frame(f.as_ref().unchecked_ref())
        .unwrap();
}
```

## Debugging DOM Operations

```rust
#[wasm_bindgen]
pub fn debug_dom() {
    let document = window().unwrap().document().unwrap();

    // Log element info
    if let Some(element) = document.get_element_by_id("myElement") {
        web_sys::console::log_1(&format!(
            "Tag: {}, Class: {}",
            element.tag_name(),
            element.class_name()
        ).into());
    }

    // Query selector debugging
    let elements = document.query_selector_all("div.myClass").unwrap();
    web_sys::console::log_1(&format!("Found {} elements", elements.length()).into());
}
```

## Best Practices

1. **Minimize DOM operations**: Batch when possible
2. **Use document fragments**: Reduce reflows
3. **Cache element references**: Don't query repeatedly
4. **Use event delegation**: One handler for many elements
5. **Forget closures carefully**: Prevent memory leaks
6. **Profile with DevTools**: Find bottlenecks
7. **Test across browsers**: WASM/DOM support varies slightly

## Next Steps

Direct DOM access from WebAssembly enables building rich, interactive web applications. With libraries like web-sys (Rust) or custom bindings, you can manipulate the DOM efficiently while keeping compute-heavy logic in WASM. Next, we'll explore using Web Workers to run WASM in background threads.
