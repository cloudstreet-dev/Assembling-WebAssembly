# Chapter 39: Microservices with WebAssembly

## WASM for Microservices

WebAssembly offers unique advantages for microservice architectures:

- **Fast startup**: Sub-millisecond initialization
- **Small footprint**: MBs vs. GBs for containers
- **Strong isolation**: Sandboxed execution
- **Polyglot**: Multiple languages in one system
- **Portability**: Deploy anywhere

## HTTP Microservices

### Basic HTTP Service (Rust + wasmtime-wasi-http)

**Cargo.toml**:
```toml
[package]
name = "user-service"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**src/lib.rs**:
```rust
use serde::{Deserialize, Serialize};
use std::io::{self, Read};

#[derive(Deserialize)]
struct CreateUserRequest {
    username: String,
    email: String,
}

#[derive(Serialize)]
struct User {
    id: u32,
    username: String,
    email: String,
}

#[derive(Serialize)]
struct ApiResponse<T> {
    success: bool,
    data: Option<T>,
    error: Option<String>,
}

fn main() -> io::Result<()> {
    // Read request from stdin
    let mut request_body = String::new();
    io::stdin().read_to_string(&mut request_body)?;

    // Route based on environment variable
    let route = std::env::var("HTTP_PATH").unwrap_or_default();
    let method = std::env::var("HTTP_METHOD").unwrap_or_default();

    let response = match (method.as_str(), route.as_str()) {
        ("POST", "/users") => create_user(&request_body),
        ("GET", path) if path.starts_with("/users/") => {
            let id = path.trim_start_matches("/users/").parse().unwrap_or(0);
            get_user(id)
        }
        _ => {
            ApiResponse {
                success: false,
                data: None::<User>,
                error: Some("Not Found".to_string()),
            }
        }
    };

    println!("{}", serde_json::to_string(&response).unwrap());
    Ok(())
}

fn create_user(body: &str) -> ApiResponse<User> {
    match serde_json::from_str::<CreateUserRequest>(body) {
        Ok(req) => {
            let user = User {
                id: 1,  // In real app: generate ID
                username: req.username,
                email: req.email,
            };

            ApiResponse {
                success: true,
                data: Some(user),
                error: None,
            }
        }
        Err(e) => ApiResponse {
            success: false,
            data: None,
            error: Some(format!("Invalid request: {}", e)),
        },
    }
}

fn get_user(id: u32) -> ApiResponse<User> {
    // In real app: fetch from database
    let user = User {
        id,
        username: "alice".to_string(),
        email: "alice@example.com".to_string(),
    };

    ApiResponse {
        success: true,
        data: Some(user),
        error: None,
    }
}
```

### Service Gateway

**gateway.rs**:
```rust
use wasmtime::*;
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder};
use std::collections::HashMap;

struct ServiceGateway {
    engine: Engine,
    services: HashMap<String, Module>,
}

impl ServiceGateway {
    fn new() -> Result<Self> {
        let engine = Engine::default();
        let mut services = HashMap::new();

        // Load microservices
        services.insert(
            "users".to_string(),
            Module::from_file(&engine, "user-service.wasm")?,
        );

        services.insert(
            "orders".to_string(),
            Module::from_file(&engine, "order-service.wasm")?,
        );

        Ok(Self { engine, services })
    }

    fn route_request(&self, service: &str, path: &str, method: &str, body: &str)
        -> Result<String>
    {
        let module = self.services.get(service)
            .ok_or_else(|| anyhow!("Service not found"))?;

        // Create WASI context with request info
        let wasi = WasiCtxBuilder::new()
            .stdin(Box::new(std::io::Cursor::new(body.to_string())))
            .env("HTTP_PATH", path)?
            .env("HTTP_METHOD", method)?
            .build();

        let mut store = Store::new(&self.engine, wasi);

        let mut linker = Linker::new(&self.engine);
        wasmtime_wasi::add_to_linker(&mut linker, |s| s)?;

        let instance = linker.instantiate(&mut store, &module)?;

        // Capture stdout
        let mut stdout_buf = Vec::new();

        // Run service
        let start = instance.get_typed_func::<(), ()>(&mut store, "_start")?;
        start.call(&mut store, ())?;

        // Read response from stdout
        // (In real implementation, capture WASI stdout)

        Ok("Service response".to_string())
    }
}

// Usage
fn main() -> Result<()> {
    let gateway = ServiceGateway::new()?;

    let response = gateway.route_request(
        "users",
        "/users",
        "POST",
        r#"{"username": "alice", "email": "alice@example.com"}"#,
    )?;

    println!("Response: {}", response);

    Ok(())
}
```

## Service-to-Service Communication

### gRPC-like Pattern

**Protocol definition** (shared types):

```rust
// shared/src/lib.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct GetUserRequest {
    pub user_id: u32,
}

#[derive(Serialize, Deserialize)]
pub struct GetUserResponse {
    pub id: u32,
    pub username: String,
    pub email: String,
}

#[derive(Serialize, Deserialize)]
pub struct CreateOrderRequest {
    pub user_id: u32,
    pub items: Vec<OrderItem>,
}

#[derive(Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: u32,
    pub quantity: u32,
}

#[derive(Serialize, Deserialize)]
pub struct CreateOrderResponse {
    pub order_id: u32,
    pub total: f64,
}
```

**Order service calling User service**:

```rust
use shared::*;

// Order service needs user info
fn create_order(req: CreateOrderRequest) -> Result<CreateOrderResponse> {
    // Call user service to validate user
    let user_req = GetUserRequest {
        user_id: req.user_id,
    };

    let user = call_service("users", "get_user", &user_req)?;

    // Process order
    let total = calculate_total(&req.items);

    Ok(CreateOrderResponse {
        order_id: generate_order_id(),
        total,
    })
}

fn call_service<T, R>(service: &str, method: &str, request: &T)
    -> Result<R>
where
    T: Serialize,
    R: for<'de> Deserialize<'de>,
{
    // Serialize request
    let request_json = serde_json::to_string(request)?;

    // Call service (via host function)
    let response_json = invoke_service_host(service, method, &request_json)?;

    // Deserialize response
    let response = serde_json::from_str(&response_json)?;

    Ok(response)
}

// Host function (provided by gateway)
#[link(wasm_import_module = "service")]
extern "C" {
    fn invoke(
        service_ptr: *const u8,
        service_len: usize,
        method_ptr: *const u8,
        method_len: usize,
        request_ptr: *const u8,
        request_len: usize,
    ) -> i32;  // Returns response pointer
}

fn invoke_service_host(service: &str, method: &str, request: &str)
    -> Result<String>
{
    unsafe {
        let response_ptr = invoke(
            service.as_ptr(),
            service.len(),
            method.as_ptr(),
            method.len(),
            request.as_ptr(),
            request.len(),
        );

        // Read response from memory
        // (simplified - real impl would read from returned pointer)
        Ok("{}".to_string())
    }
}
```

**Gateway implementation**:

```rust
struct ServiceRegistry {
    services: HashMap<String, Box<dyn Service>>,
}

trait Service {
    fn call(&self, method: &str, request: &str) -> Result<String>;
}

impl ServiceRegistry {
    fn invoke(&self, service: &str, method: &str, request: &str)
        -> Result<String>
    {
        let svc = self.services.get(service)
            .ok_or_else(|| anyhow!("Service not found"))?;

        svc.call(method, request)
    }
}

// Provide as host function to WASM services
fn create_invoke_func(registry: Arc<Mutex<ServiceRegistry>>) -> Func {
    Func::wrap(&mut store, move |
        service_ptr: i32,
        service_len: i32,
        method_ptr: i32,
        method_len: i32,
        request_ptr: i32,
        request_len: i32,
    | -> i32 {
        // Read service name, method, request from WASM memory
        // Call registry.invoke()
        // Write response to WASM memory
        // Return pointer
        0
    })
}
```

## Message Queue Pattern

### Event-Driven Microservices

**Event types**:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    UserCreated {
        user_id: u32,
        username: String,
        email: String,
    },
    OrderPlaced {
        order_id: u32,
        user_id: u32,
        total: f64,
    },
    PaymentProcessed {
        order_id: u32,
        success: bool,
    },
}
```

**Event producer** (Order Service):

```rust
fn place_order(req: CreateOrderRequest) -> Result<()> {
    // Create order
    let order_id = create_order_in_db(&req)?;

    // Publish event
    let event = Event::OrderPlaced {
        order_id,
        user_id: req.user_id,
        total: calculate_total(&req.items),
    };

    publish_event(&event)?;

    Ok(())
}

fn publish_event(event: &Event) -> Result<()> {
    let event_json = serde_json::to_string(event)?;

    // Call host function to publish
    unsafe {
        publish_event_host(event_json.as_ptr(), event_json.len());
    }

    Ok(())
}

#[link(wasm_import_module = "events")]
extern "C" {
    fn publish_event_host(ptr: *const u8, len: usize);
}
```

**Event consumer** (Email Service):

```rust
fn handle_event(event_json: &str) -> Result<()> {
    let event: Event = serde_json::from_str(event_json)?;

    match event {
        Event::UserCreated { email, username, .. } => {
            send_welcome_email(&email, &username)?;
        }
        Event::OrderPlaced { user_id, order_id, .. } => {
            send_order_confirmation(user_id, order_id)?;
        }
        _ => {}
    }

    Ok(())
}

fn send_welcome_email(email: &str, username: &str) -> Result<()> {
    // Call host function to send email
    println!("Sending welcome email to {}", email);
    Ok(())
}
```

**Event broker** (Gateway):

```rust
use std::sync::mpsc;

struct EventBroker {
    subscribers: HashMap<String, Vec<mpsc::Sender<String>>>,
}

impl EventBroker {
    fn subscribe(&mut self, event_type: &str, subscriber: mpsc::Sender<String>) {
        self.subscribers
            .entry(event_type.to_string())
            .or_insert_with(Vec::new)
            .push(subscriber);
    }

    fn publish(&self, event_type: &str, event_data: String) {
        if let Some(subs) = self.subscribers.get(event_type) {
            for sub in subs {
                let _ = sub.send(event_data.clone());
            }
        }
    }
}

// Service handler with event subscription
struct EventDrivenService {
    module: Module,
    engine: Engine,
    event_rx: mpsc::Receiver<String>,
}

impl EventDrivenService {
    fn run(&self) -> Result<()> {
        loop {
            // Wait for events
            if let Ok(event_data) = self.event_rx.recv() {
                // Process event in WASM
                self.handle_event(&event_data)?;
            }
        }
    }

    fn handle_event(&self, event_data: &str) -> Result<()> {
        let wasi = WasiCtxBuilder::new()
            .stdin(Box::new(std::io::Cursor::new(event_data.to_string())))
            .build();

        let mut store = Store::new(&self.engine, wasi);

        let mut linker = Linker::new(&self.engine);
        wasmtime_wasi::add_to_linker(&mut linker, |s| s)?;

        let instance = linker.instantiate(&mut store, &self.module)?;

        let handle_func = instance
            .get_typed_func::<(), ()>(&mut store, "handle_event")?;

        handle_func.call(&mut store, ())?;

        Ok(())
    }
}
```

## Container Orchestration

### Docker with WASM

**Dockerfile**:
```dockerfile
FROM scratch

# Copy WASM module
COPY target/wasm32-wasi/release/service.wasm /service.wasm

# Runtime (wasmtime or wasmer)
COPY --from=wasmtime/wasmtime:latest /usr/local/bin/wasmtime /wasmtime

# Run service
CMD ["/wasmtime", "run", "--dir=/data", "/service.wasm"]
```

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  user-service:
    image: user-service:wasm
    ports:
      - "8001:8000"
    environment:
      - SERVICE_NAME=users
      - DB_HOST=postgres
    volumes:
      - ./data:/data

  order-service:
    image: order-service:wasm
    ports:
      - "8002:8000"
    environment:
      - SERVICE_NAME=orders
      - USER_SERVICE_URL=http://user-service:8000
    depends_on:
      - user-service

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Kubernetes with WASM

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      runtimeClassName: wasmtime  # Using containerd-wasm-shim
      containers:
      - name: user-service
        image: ghcr.io/myorg/user-service:wasm
        ports:
        - containerPort: 8000
        env:
        - name: SERVICE_NAME
          value: "users"
        resources:
          limits:
            memory: "64Mi"  # Much smaller than traditional containers
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
```

### Spin (WASM-native orchestration)

**spin.toml**:
```toml
spin_manifest_version = "1"
name = "microservices-app"
version = "0.1.0"

[[component]]
id = "user-service"
source = "users/target/wasm32-wasi/release/user_service.wasm"
[component.trigger]
route = "/users/..."
[component.build]
command = "cargo build --target wasm32-wasi --release"
workdir = "users"

[[component]]
id = "order-service"
source = "orders/target/wasm32-wasi/release/order_service.wasm"
[component.trigger]
route = "/orders/..."
[component.build]
command = "cargo build --target wasm32-wasi --release"
workdir = "orders"

[[component]]
id = "gateway"
source = "gateway/target/wasm32-wasi/release/gateway.wasm"
[component.trigger]
route = "/..."
[component.build]
command = "cargo build --target wasm32-wasi --release"
workdir = "gateway"
```

## Database Integration

### PostgreSQL Connection

```rust
// Note: Database drivers in WASM are limited
// Typically call host functions for DB access

#[link(wasm_import_module = "database")]
extern "C" {
    fn query(
        sql_ptr: *const u8,
        sql_len: usize,
        params_ptr: *const u8,
        params_len: usize,
    ) -> i32;
}

fn db_query(sql: &str, params: &str) -> Result<String> {
    unsafe {
        let result_ptr = query(
            sql.as_ptr(),
            sql.len(),
            params.as_ptr(),
            params.len(),
        );

        // Read result from returned pointer
        // (simplified)
        Ok("[]".to_string())
    }
}

// Usage
fn get_user(id: u32) -> Result<User> {
    let sql = "SELECT * FROM users WHERE id = $1";
    let params = serde_json::to_string(&[id])?;

    let result_json = db_query(sql, &params)?;
    let users: Vec<User> = serde_json::from_str(&result_json)?;

    users.into_iter().next()
        .ok_or_else(|| anyhow!("User not found"))
}
```

**Host-side database provider**:

```rust
use postgres::{Client, NoTls};

fn create_db_query_func(db_url: String) -> Func {
    Func::wrap(&mut store, move |
        mut caller: Caller<WasiCtx>,
        sql_ptr: i32,
        sql_len: i32,
        params_ptr: i32,
        params_len: i32,
    | -> i32 {
        // Read SQL and params from WASM memory
        let memory = caller.get_export("memory").unwrap().into_memory().unwrap();
        let data = memory.data(&caller);

        let sql = std::str::from_utf8(
            &data[sql_ptr as usize..(sql_ptr + sql_len) as usize]
        ).unwrap();

        let params_json = std::str::from_utf8(
            &data[params_ptr as usize..(params_ptr + params_len) as usize]
        ).unwrap();

        // Execute query
        let mut client = Client::connect(&db_url, NoTls).unwrap();
        let rows = client.query(sql, &[]).unwrap();

        // Serialize result
        let result_json = serde_json::to_string(&rows).unwrap();

        // Write result to WASM memory and return pointer
        // (simplified)
        0
    })
}
```

## Service Discovery

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct ServiceInfo {
    name: String,
    url: String,
    health: bool,
}

struct ServiceDiscovery {
    services: Arc<Mutex<HashMap<String, Vec<ServiceInfo>>>>,
}

impl ServiceDiscovery {
    fn new() -> Self {
        Self {
            services: Arc::new(Mutex::new(HashMap::new())),
        }
    }

    fn register(&self, service: ServiceInfo) {
        let mut services = self.services.lock().unwrap();
        services
            .entry(service.name.clone())
            .or_insert_with(Vec::new)
            .push(service);
    }

    fn discover(&self, service_name: &str) -> Option<ServiceInfo> {
        let services = self.services.lock().unwrap();

        services.get(service_name)
            .and_then(|instances| {
                // Simple round-robin
                instances.iter()
                    .find(|s| s.health)
                    .cloned()
            })
    }

    fn health_check(&self, service_name: &str, instance_url: &str, healthy: bool) {
        let mut services = self.services.lock().unwrap();

        if let Some(instances) = services.get_mut(service_name) {
            for instance in instances {
                if instance.url == instance_url {
                    instance.health = healthy;
                }
            }
        }
    }
}

// Provide to WASM services
fn create_discover_func(registry: Arc<ServiceDiscovery>) -> Func {
    Func::wrap(&mut store, move |service_name_ptr: i32, service_name_len: i32| -> i32 {
        // Read service name from WASM memory
        // Call registry.discover()
        // Write result to WASM memory
        // Return pointer
        0
    })
}
```

## Load Balancing

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

struct LoadBalancer {
    instances: Vec<ServiceInstance>,
    current: AtomicUsize,
}

struct ServiceInstance {
    id: String,
    module: Module,
    healthy: bool,
}

impl LoadBalancer {
    fn get_instance(&self) -> Option<&ServiceInstance> {
        if self.instances.is_empty() {
            return None;
        }

        // Round-robin
        let index = self.current.fetch_add(1, Ordering::Relaxed) % self.instances.len();

        // Find next healthy instance
        for i in 0..self.instances.len() {
            let idx = (index + i) % self.instances.len();
            if self.instances[idx].healthy {
                return Some(&self.instances[idx]);
            }
        }

        None
    }

    fn handle_request(&self, request: &str) -> Result<String> {
        let instance = self.get_instance()
            .ok_or_else(|| anyhow!("No healthy instances"))?;

        // Execute request on instance
        execute_on_instance(instance, request)
    }
}
```

## Monitoring and Observability

```rust
use std::time::Instant;

struct ServiceMetrics {
    request_count: AtomicUsize,
    error_count: AtomicUsize,
    total_duration_ms: AtomicUsize,
}

impl ServiceMetrics {
    fn record_request(&self, duration_ms: u64, error: bool) {
        self.request_count.fetch_add(1, Ordering::Relaxed);

        if error {
            self.error_count.fetch_add(1, Ordering::Relaxed);
        }

        self.total_duration_ms.fetch_add(duration_ms as usize, Ordering::Relaxed);
    }

    fn get_stats(&self) -> Stats {
        let requests = self.request_count.load(Ordering::Relaxed);
        let errors = self.error_count.load(Ordering::Relaxed);
        let total_ms = self.total_duration_ms.load(Ordering::Relaxed);

        Stats {
            requests,
            errors,
            error_rate: if requests > 0 {
                (errors as f64 / requests as f64) * 100.0
            } else {
                0.0
            },
            avg_duration_ms: if requests > 0 {
                total_ms / requests
            } else {
                0
            },
        }
    }
}

// Instrumented service call
fn call_service_instrumented(
    service: &str,
    request: &str,
    metrics: &ServiceMetrics,
) -> Result<String> {
    let start = Instant::now();
    let result = call_service(service, request);
    let duration = start.elapsed().as_millis() as u64;

    metrics.record_request(duration, result.is_err());

    result
}
```

## Best Practices

1. **Keep services small**: Single responsibility principle
2. **Design for failure**: Retry logic, circuit breakers
3. **Version APIs**: Backward compatibility
4. **Use async patterns**: Non-blocking I/O where possible
5. **Monitor everything**: Metrics, logs, traces
6. **Automate deployment**: CI/CD pipelines
7. **Test integration**: Service-to-service tests
8. **Secure communication**: Authentication, encryption
9. **Optimize binaries**: Small, fast-starting services
10. **Document interfaces**: Clear API specifications

## Next Steps

WebAssembly enables building lightweight, fast-starting microservices with strong isolation guarantees. Whether using traditional orchestration (Kubernetes, Docker Compose) or WASM-native platforms (Spin), WASM microservices offer significant advantages in resource usage and cold start times.

Next, in **Part 8**, we'll explore WebAssembly in embedded systems and edge computing, where WASM's small footprint and security shine in resource-constrained environments.
