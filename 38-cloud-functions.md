# Chapter 38: Cloud Functions and Serverless

## Why WASM for Serverless?

WebAssembly's characteristics make it ideal for serverless/edge computing:

- **Fast cold starts**: 1-10ms vs. 100-1000ms for containers
- **Small binary sizes**: KBs to low MBs
- **Strong isolation**: Sandboxed execution
- **Portability**: Write once, deploy anywhere
- **Language flexibility**: Use any WASM-compatible language

## Cloudflare Workers

**Overview**: Serverless platform running on Cloudflare's edge network (300+ locations worldwide). Workers execute JavaScript and WebAssembly at the edge.

### Rust Worker with wasm-bindgen

**Cargo.toml**:
```toml
[package]
name = "worker"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**src/lib.rs**:
```rust
use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Request {
    name: String,
}

#[derive(Serialize)]
struct Response {
    message: String,
    timestamp: f64,
}

#[wasm_bindgen]
pub fn handle_request(input: &str) -> String {
    let req: Request = serde_json::from_str(input).unwrap_or(Request {
        name: "World".to_string(),
    });

    let response = Response {
        message: format!("Hello, {}!", req.name),
        timestamp: js_sys::Date::now(),
    };

    serde_json::to_string(&response).unwrap()
}
```

**JavaScript glue** (index.js):
```javascript
import { handle_request } from './worker.wasm';

export default {
    async fetch(request) {
        const { pathname } = new URL(request.url);

        if (pathname === '/hello') {
            const body = await request.json();
            const result = handle_request(JSON.stringify(body));

            return new Response(result, {
                headers: { 'Content-Type': 'application/json' }
            });
        }

        return new Response('Not Found', { status: 404 });
    }
};
```

**wrangler.toml**:
```toml
name = "wasm-worker"
main = "index.js"
compatibility_date = "2024-01-01"

[build]
command = "cargo build --target wasm32-unknown-unknown --release && cp target/wasm32-unknown-unknown/release/worker.wasm ."
```

**Build and deploy**:
```bash
# Build
wrangler build

# Test locally
wrangler dev

# Deploy
wrangler publish
```

### Complete API Example

```rust
use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    username: String,
    email: String,
}

#[derive(Serialize)]
struct User {
    id: u32,
    username: String,
    email: String,
    created_at: f64,
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
}

#[wasm_bindgen]
pub fn create_user(input: &str) -> String {
    let user_data: Result<CreateUser, _> = serde_json::from_str(input);

    match user_data {
        Ok(data) => {
            // Validate
            if data.username.is_empty() {
                return error_response("Username required");
            }

            if !data.email.contains('@') {
                return error_response("Invalid email");
            }

            // Create user
            let user = User {
                id: generate_id(),
                username: data.username,
                email: data.email,
                created_at: js_sys::Date::now(),
            };

            serde_json::to_string(&user).unwrap()
        }
        Err(_) => error_response("Invalid JSON"),
    }
}

fn error_response(message: &str) -> String {
    let err = ErrorResponse {
        error: message.to_string(),
    };
    serde_json::to_string(&err).unwrap()
}

fn generate_id() -> u32 {
    (js_sys::Math::random() * 1_000_000.0) as u32
}
```

**Worker JavaScript**:
```javascript
import { create_user } from './api.wasm';

export default {
    async fetch(request) {
        const { pathname } = new URL(request.url);

        if (pathname === '/users' && request.method === 'POST') {
            const body = await request.text();
            const result = create_user(body);

            return new Response(result, {
                headers: {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                }
            });
        }

        return new Response('Not Found', { status: 404 });
    }
};
```

## Fastly Compute@Edge

**Overview**: Edge computing platform using Wasmtime, supports multiple languages via WASM.

### Rust Service

**Cargo.toml**:
```toml
[package]
name = "fastly-service"
version = "0.1.0"
edition = "2021"

[dependencies]
fastly = "0.9"
serde_json = "1.0"
```

**src/main.rs**:
```rust
use fastly::http::{Method, StatusCode};
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(mut req: Request) -> Result<Response, Error> {
    match (req.get_method(), req.get_path()) {
        (&Method::GET, "/") => {
            Ok(Response::from_status(StatusCode::OK)
                .with_body_text_plain("Hello from Fastly!"))
        }

        (&Method::GET, "/json") => {
            let data = serde_json::json!({
                "message": "Hello, World!",
                "edge_location": std::env::var("FASTLY_POP").unwrap_or_default()
            });

            Ok(Response::from_status(StatusCode::OK)
                .with_body_json(&data)?)
        }

        (&Method::POST, "/echo") => {
            let body = req.take_body_str();
            Ok(Response::from_status(StatusCode::OK)
                .with_body_text_plain(&body))
        }

        _ => {
            Ok(Response::from_status(StatusCode::NOT_FOUND)
                .with_body_text_plain("Not Found"))
        }
    }
}
```

**Backend requests**:
```rust
use fastly::http::Method;
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(req: Request) -> Result<Response, Error> {
    match req.get_path() {
        "/proxy" => {
            // Forward to backend
            let backend_req = Request::get("https://api.example.com/data")
                .with_header("User-Agent", "Fastly-Edge");

            let mut backend_resp = backend_req.send("api_backend")?;

            // Modify response
            backend_resp.set_header("X-Served-By", "Fastly-Edge");

            Ok(backend_resp)
        }

        "/cached" => {
            // With caching
            let mut backend_resp = Request::get("https://api.example.com/data")
                .send("api_backend")?;

            backend_resp.set_header("Cache-Control", "public, max-age=300");
            Ok(backend_resp)
        }

        _ => Ok(Response::from_status(StatusCode::NOT_FOUND)),
    }
}
```

**Build and deploy**:
```bash
fastly compute build
fastly compute deploy
```

## Fermyon Spin

**Overview**: Serverless platform built specifically for WebAssembly, supports HTTP triggers, Redis, etc.

### Rust HTTP Handler

**spin.toml**:
```toml
spin_manifest_version = "1"
name = "spin-app"
version = "0.1.0"

[[component]]
id = "hello"
source = "target/wasm32-wasi/release/hello.wasm"
[component.trigger]
route = "/..."
[component.build]
command = "cargo build --target wasm32-wasi --release"
```

**src/lib.rs**:
```rust
use spin_sdk::{
    http::{Request, Response},
    http_component,
};

#[http_component]
fn handle_request(req: Request) -> Result<Response> {
    match req.uri().path() {
        "/" => Ok(Response::builder()
            .status(200)
            .header("Content-Type", "text/plain")
            .body(Some("Hello from Spin!".into()))?),

        "/json" => {
            let json = r#"{"message": "Hello, World!"}"#;
            Ok(Response::builder()
                .status(200)
                .header("Content-Type", "application/json")
                .body(Some(json.into()))?)
        }

        _ => Ok(Response::builder()
            .status(404)
            .body(Some("Not Found".into()))?),
    }
}
```

### With State (Redis)

```rust
use spin_sdk::{
    http::{Request, Response},
    http_component,
    redis,
};

#[http_component]
fn handle_request(req: Request) -> Result<Response> {
    let path = req.uri().path();

    if path.starts_with("/counter") {
        // Increment counter in Redis
        let count = redis::incr("counter")?;

        let body = format!("Counter: {}", count);

        return Ok(Response::builder()
            .status(200)
            .body(Some(body.into()))?);
    }

    Ok(Response::builder()
        .status(404)
        .body(Some("Not Found".into()))?)
}
```

**Run locally**:
```bash
spin build
spin up
```

**Deploy to Fermyon Cloud**:
```bash
spin deploy
```

## Wasmer Edge

**Overview**: Edge computing platform by Wasmer, supports multiple languages.

```rust
use wasmer_sdk::*;

#[wasmer_sdk::handler]
async fn handle(req: Request) -> Result<Response, Error> {
    let path = req.path();

    match path {
        "/" => {
            Ok(Response::new()
                .with_status(200)
                .with_body("Hello from Wasmer Edge!"))
        }

        "/headers" => {
            let user_agent = req
                .headers()
                .get("User-Agent")
                .unwrap_or("Unknown");

            Ok(Response::new()
                .with_status(200)
                .with_body(format!("User-Agent: {}", user_agent)))
        }

        _ => {
            Ok(Response::new()
                .with_status(404)
                .with_body("Not Found"))
        }
    }
}
```

## AWS Lambda with WASM

While not native, you can run WASM in Lambda using custom runtimes.

### Custom Runtime Approach

**bootstrap** (Lambda runtime):
```bash
#!/bin/sh
set -euo pipefail

# Get Lambda environment
LAMBDA_TASK_ROOT="${LAMBDA_TASK_ROOT:-/var/task}"

# Run wasmtime
exec wasmtime run \
    --dir=. \
    "$LAMBDA_TASK_ROOT/function.wasm"
```

**Rust Lambda Function**:
```rust
use serde::{Deserialize, Serialize};
use std::io::{self, Read};

#[derive(Deserialize)]
struct LambdaEvent {
    body: String,
}

#[derive(Serialize)]
struct LambdaResponse {
    statusCode: u16,
    body: String,
}

fn main() -> io::Result<()> {
    // Read event from stdin
    let mut event_json = String::new();
    io::stdin().read_to_string(&mut event_json)?;

    let event: LambdaEvent = serde_json::from_str(&event_json)
        .unwrap_or(LambdaEvent {
            body: "{}".to_string(),
        });

    // Process
    let response = LambdaResponse {
        statusCode: 200,
        body: format!("Received: {}", event.body),
    };

    // Write response to stdout
    println!("{}", serde_json::to_string(&response).unwrap());

    Ok(())
}
```

**Build and package**:
```bash
cargo build --target wasm32-wasi --release

mkdir -p layer/bin
cp target/wasm32-wasi/release/function.wasm layer/
cp bootstrap layer/
chmod +x layer/bootstrap

cd layer
zip -r ../lambda.zip .
```

## Common Patterns

### Request Routing

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Route {
    method: String,
    path: String,
}

pub fn route(method: &str, path: &str) -> String {
    match (method, path) {
        ("GET", "/") => home(),
        ("GET", "/about") => about(),
        ("POST", "/api/users") => create_user(),
        ("GET", path) if path.starts_with("/api/users/") => get_user(path),
        _ => not_found(),
    }
}

fn home() -> String {
    "Home Page".to_string()
}

fn about() -> String {
    "About Page".to_string()
}

fn create_user() -> String {
    r#"{"status": "created"}"#.to_string()
}

fn get_user(path: &str) -> String {
    let id = path.trim_start_matches("/api/users/");
    format!(r#"{{"id": "{}"}}"#, id)
}

fn not_found() -> String {
    "404 Not Found".to_string()
}
```

### JSON Processing

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Input {
    numbers: Vec<i32>,
}

#[derive(Serialize)]
struct Output {
    sum: i32,
    average: f64,
    max: i32,
    min: i32,
}

#[wasm_bindgen]
pub fn process_numbers(input: &str) -> String {
    let data: Input = match serde_json::from_str(input) {
        Ok(d) => d,
        Err(_) => return r#"{"error": "Invalid JSON"}"#.to_string(),
    };

    if data.numbers.is_empty() {
        return r#"{"error": "Empty array"}"#.to_string();
    }

    let sum: i32 = data.numbers.iter().sum();
    let average = sum as f64 / data.numbers.len() as f64;
    let max = *data.numbers.iter().max().unwrap();
    let min = *data.numbers.iter().min().unwrap();

    let output = Output {
        sum,
        average,
        max,
        min,
    };

    serde_json::to_string(&output).unwrap()
}
```

### Caching

```rust
use std::collections::HashMap;
use std::sync::Mutex;

// Simple in-memory cache
static CACHE: Mutex<Option<HashMap<String, String>>> = Mutex::new(None);

pub fn get_cached(key: &str) -> Option<String> {
    let mut cache = CACHE.lock().unwrap();

    if cache.is_none() {
        *cache = Some(HashMap::new());
    }

    cache.as_ref().unwrap().get(key).cloned()
}

pub fn set_cached(key: String, value: String) {
    let mut cache = CACHE.lock().unwrap();

    if cache.is_none() {
        *cache = Some(HashMap::new());
    }

    cache.as_mut().unwrap().insert(key, value);
}

#[wasm_bindgen]
pub fn handle_with_cache(key: &str) -> String {
    // Check cache
    if let Some(cached) = get_cached(key) {
        return cached;
    }

    // Compute result
    let result = expensive_computation(key);

    // Cache result
    set_cached(key.to_string(), result.clone());

    result
}

fn expensive_computation(input: &str) -> String {
    // Simulate expensive operation
    format!("Computed: {}", input)
}
```

### Rate Limiting

```rust
use std::collections::HashMap;
use std::sync::Mutex;

struct RateLimiter {
    requests: HashMap<String, Vec<f64>>,
    limit: usize,
    window_ms: f64,
}

impl RateLimiter {
    fn new(limit: usize, window_ms: f64) -> Self {
        Self {
            requests: HashMap::new(),
            limit,
            window_ms,
        }
    }

    fn check(&mut self, ip: &str, now: f64) -> bool {
        let cutoff = now - self.window_ms;

        let timestamps = self
            .requests
            .entry(ip.to_string())
            .or_insert_with(Vec::new);

        // Remove old requests
        timestamps.retain(|&t| t > cutoff);

        if timestamps.len() >= self.limit {
            return false;  // Rate limit exceeded
        }

        timestamps.push(now);
        true
    }
}

static LIMITER: Mutex<Option<RateLimiter>> = Mutex::new(None);

#[wasm_bindgen]
pub fn check_rate_limit(ip: &str) -> bool {
    let mut limiter_opt = LIMITER.lock().unwrap();

    if limiter_opt.is_none() {
        *limiter_opt = Some(RateLimiter::new(100, 60_000.0));  // 100 req/min
    }

    let limiter = limiter_opt.as_mut().unwrap();
    let now = js_sys::Date::now();

    limiter.check(ip, now)
}
```

## Performance Optimization

### Minimize Binary Size

```bash
# Rust
cargo build --target wasm32-unknown-unknown --release
wasm-opt -Oz -o optimized.wasm target/wasm32-unknown-unknown/release/app.wasm
wasm-strip optimized.wasm

# Typical results:
# Before: 500KB
# After wasm-opt: 150KB
# After gzip: 50KB
```

### Lazy Initialization

```rust
use std::sync::Once;

static INIT: Once = Once::new();
static mut INITIALIZED: bool = false;

fn initialize() {
    INIT.call_once(|| {
        // Expensive one-time setup
        unsafe {
            INITIALIZED = true;
        }
    });
}

#[wasm_bindgen]
pub fn handle_request(input: &str) -> String {
    initialize();  // Only runs once

    // Process request
    process(input)
}
```

### Avoid Allocations

```rust
// ✗ Slow: allocates on every call
#[wasm_bindgen]
pub fn process(input: &str) -> String {
    let mut result = String::new();
    // ...
    result
}

// ✓ Fast: reuse buffer
use std::cell::RefCell;

thread_local! {
    static BUFFER: RefCell<String> = RefCell::new(String::with_capacity(1024));
}

#[wasm_bindgen]
pub fn process(input: &str) -> String {
    BUFFER.with(|buf| {
        let mut buffer = buf.borrow_mut();
        buffer.clear();
        // Use buffer
        buffer.clone()  // Only allocate on return
    })
}
```

## Monitoring and Logging

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    #[wasm_bindgen(js_namespace = console)]
    fn error(s: &str);
}

macro_rules! log {
    ($($t:tt)*) => {
        log(&format!($($t)*))
    };
}

macro_rules! error {
    ($($t:tt)*) => {
        error(&format!($($t)*))
    };
}

#[wasm_bindgen]
pub fn handle_request(input: &str) -> String {
    log!("Request received: {} bytes", input.len());

    match process(input) {
        Ok(result) => {
            log!("Request processed successfully");
            result
        }
        Err(e) => {
            error!("Error processing request: {}", e);
            format!(r#"{{"error": "{}"}}"#, e)
        }
    }
}
```

## Error Handling

```rust
use serde::Serialize;

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    code: u16,
}

impl ErrorResponse {
    fn new(code: u16, message: &str) -> Self {
        Self {
            error: message.to_string(),
            code,
        }
    }

    fn to_json(&self) -> String {
        serde_json::to_string(self).unwrap()
    }
}

#[wasm_bindgen]
pub fn safe_handle(input: &str) -> String {
    match handle_internal(input) {
        Ok(result) => result,
        Err(e) => {
            let error = match e.kind() {
                ErrorKind::InvalidInput => {
                    ErrorResponse::new(400, "Invalid input")
                }
                ErrorKind::NotFound => {
                    ErrorResponse::new(404, "Not found")
                }
                _ => {
                    ErrorResponse::new(500, "Internal error")
                }
            };

            error.to_json()
        }
    }
}

fn handle_internal(input: &str) -> io::Result<String> {
    // Process with Result type
    Ok("Success".to_string())
}
```

## Testing Serverless Functions

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_handle_request() {
        let input = r#"{"name": "Alice"}"#;
        let result = handle_request(input);
        assert!(result.contains("Alice"));
    }

    #[test]
    fn test_invalid_json() {
        let input = "invalid json";
        let result = handle_request(input);
        assert!(result.contains("error"));
    }

    #[test]
    fn test_rate_limiting() {
        assert!(check_rate_limit("127.0.0.1"));
        // Simulate many requests
        for _ in 0..100 {
            check_rate_limit("127.0.0.1");
        }
        assert!(!check_rate_limit("127.0.0.1"));
    }
}
```

## Best Practices

1. **Minimize dependencies**: Smaller binaries = faster cold starts
2. **Optimize for size**: Use wasm-opt and strip symbols
3. **Cache expensive operations**: Reuse computations across invocations
4. **Handle errors gracefully**: Return appropriate HTTP status codes
5. **Log strategically**: Essential information only
6. **Test thoroughly**: Unit and integration tests
7. **Monitor performance**: Track cold starts and execution time
8. **Set appropriate timeouts**: Fail fast on slow operations

## Next Steps

WebAssembly is transforming serverless computing with its fast cold starts, small size, and strong isolation. Platforms like Cloudflare Workers, Fastly Compute@Edge, and Fermyon Spin make deploying WASM functions simple and efficient.

Next, we'll explore building microservices with WebAssembly, creating distributed systems that leverage WASM's portability and performance.
