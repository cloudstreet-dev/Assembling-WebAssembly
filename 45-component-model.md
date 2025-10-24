# Chapter 45: The Component Model

## What is the Component Model?

The WebAssembly Component Model is a proposed standard for building composable, language-agnostic WASM modules with:

- **Rich interfaces**: Beyond just numbers (strings, records, variants, etc.)
- **Language interoperability**: Different languages working together seamlessly
- **Versioning**: Interface evolution without breaking changes
- **Virtualization**: Components can wrap and compose other components
- **Standards**: Common interfaces (WASI, HTTP, etc.)

**Status**: Proposal, actively developed by the Bytecode Alliance

## Key Concepts

### Components vs Modules

**Core Module** (current WASM):
```wat
(module
  (func (export "add") (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )
)
```
- Only numeric types (i32, i64, f32, f64)
- Linear memory for complex data
- Manual marshalling

**Component** (Component Model):
```wit
// WIT (WebAssembly Interface Types)
interface math {
  add: func(a: s32, b: s32) -> s32
  divide: func(a: float64, b: float64) -> result<float64, string>
}
```
- Rich types (strings, records, variants, results)
- Automatic marshalling
- Typed interfaces

## WIT (WebAssembly Interface Types)

### Basic Types

```wit
// Primitives
bool-val: bool
byte-val: u8
number: s32
big-number: s64
float: float32
double: float64

// Strings
name: string

// Lists
numbers: list<s32>
names: list<string>

// Tuples
point: tuple<float32, float32>

// Options
maybe-name: option<string>

// Results (for errors)
division: result<float64, string>
```

### Records (Structs)

```wit
record person {
  name: string,
  age: u32,
  email: option<string>
}

record point {
  x: float32,
  y: float32
}

record rectangle {
  top-left: point,
  bottom-right: point
}
```

### Variants (Enums/Tagged Unions)

```wit
variant color {
  rgb(u8, u8, u8),
  named(string)
}

variant result-type {
  ok(s32),
  err(string)
}

variant message {
  text(string),
  image(list<u8>),
  video(string, u64)  // URL and duration
}
```

### Interfaces

```wit
interface calculator {
  add: func(a: s32, b: s32) -> s32
  subtract: func(a: s32, b: s32) -> s32
  multiply: func(a: s32, b: s32) -> s32
  divide: func(a: s32, b: s32) -> result<s32, string>
}

interface user-service {
  record user {
    id: u32,
    username: string,
    email: string
  }

  create-user: func(username: string, email: string) -> result<user, string>
  get-user: func(id: u32) -> option<user>
  list-users: func() -> list<user>
}
```

### Worlds

Worlds define component boundaries (imports and exports):

```wit
world calculator-component {
  export calculator

  import logging: interface {
    log: func(message: string)
  }
}

world web-service {
  export http-handler: interface {
    handle-request: func(req: request) -> response
  }

  import database: interface {
    query: func(sql: string) -> result<list<row>, string>
  }
}
```

## Building Components

### Rust with wit-bindgen

**wit/world.wit**:
```wit
package example:calculator

interface operations {
  add: func(a: s32, b: s32) -> s32
  divide: func(a: s32, b: s32) -> result<s32, string>
}

world calculator {
  export operations
}
```

**Cargo.toml**:
```toml
[package]
name = "calculator"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "0.13"

[package.metadata.component]
package = "example:calculator"
```

**src/lib.rs**:
```rust
wit_bindgen::generate!({
    world: "calculator",
    path: "../wit"
});

struct Calculator;

impl Guest for Calculator {
    fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    fn divide(a: i32, b: i32) -> Result<i32, String> {
        if b == 0 {
            Err("Division by zero".to_string())
        } else {
            Ok(a / b)
        }
    }
}

export!(Calculator);
```

**Build**:
```bash
cargo build --target wasm32-wasi --release

# Convert core module to component
wasm-tools component new \
    target/wasm32-wasi/release/calculator.wasm \
    -o calculator.component.wasm
```

### Complex Example: User Service

**wit/user-service.wit**:
```wit
package example:users

interface types {
  record user {
    id: u32,
    username: string,
    email: string,
    created-at: u64
  }

  variant user-error {
    not-found,
    already-exists,
    invalid-email,
    database-error(string)
  }
}

interface service {
  use types.{user, user-error}

  create-user: func(username: string, email: string)
    -> result<user, user-error>

  get-user: func(id: u32) -> result<user, user-error>

  update-email: func(id: u32, new-email: string)
    -> result<_, user-error>

  list-users: func(offset: u32, limit: u32) -> list<user>

  delete-user: func(id: u32) -> result<_, user-error>
}

world user-service {
  export service
}
```

**src/lib.rs**:
```rust
use std::collections::HashMap;
use std::sync::Mutex;

wit_bindgen::generate!({
    world: "user-service",
    path: "../wit"
});

use exports::example::users::types::{User, UserError};

struct UserService {
    users: Mutex<HashMap<u32, User>>,
    next_id: Mutex<u32>,
}

static SERVICE: UserService = UserService {
    users: Mutex::new(HashMap::new()),
    next_id: Mutex::new(1),
};

impl Guest for UserService {
    fn create_user(username: String, email: String) -> Result<User, UserError> {
        if !email.contains('@') {
            return Err(UserError::InvalidEmail);
        }

        let mut users = SERVICE.users.lock().unwrap();

        // Check if username exists
        if users.values().any(|u| u.username == username) {
            return Err(UserError::AlreadyExists);
        }

        let mut next_id = SERVICE.next_id.lock().unwrap();
        let id = *next_id;
        *next_id += 1;

        let user = User {
            id,
            username,
            email,
            created_at: current_timestamp(),
        };

        users.insert(id, user.clone());

        Ok(user)
    }

    fn get_user(id: u32) -> Result<User, UserError> {
        let users = SERVICE.users.lock().unwrap();

        users.get(&id)
            .cloned()
            .ok_or(UserError::NotFound)
    }

    fn update_email(id: u32, new_email: String) -> Result<(), UserError> {
        if !new_email.contains('@') {
            return Err(UserError::InvalidEmail);
        }

        let mut users = SERVICE.users.lock().unwrap();

        let user = users.get_mut(&id)
            .ok_or(UserError::NotFound)?;

        user.email = new_email;

        Ok(())
    }

    fn list_users(offset: u32, limit: u32) -> Vec<User> {
        let users = SERVICE.users.lock().unwrap();

        users.values()
            .skip(offset as usize)
            .take(limit as usize)
            .cloned()
            .collect()
    }

    fn delete_user(id: u32) -> Result<(), UserError> {
        let mut users = SERVICE.users.lock().unwrap();

        users.remove(&id)
            .map(|_| ())
            .ok_or(UserError::NotFound)
    }
}

fn current_timestamp() -> u64 {
    // In real implementation, get actual timestamp
    0
}

export!(UserService);
```

## Component Composition

### Importing Components

**consumer.wit**:
```wit
package example:consumer

interface consumer {
  use example:users/types.{user}

  process-user: func(user-id: u32) -> string
}

world consumer-world {
  import example:users/service
  export consumer
}
```

**src/lib.rs**:
```rust
wit_bindgen::generate!({
    world: "consumer-world",
    path: "../wit"
});

use example::users::service;
use example::users::types::User;

struct Consumer;

impl Guest for Consumer {
    fn process_user(user_id: u32) -> String {
        // Call imported user service
        match service::get_user(user_id) {
            Ok(user) => {
                format!("User: {} ({})", user.username, user.email)
            }
            Err(e) => {
                format!("Error: {:?}", e)
            }
        }
    }
}

export!(Consumer);
```

### Component Linking

```bash
# Build both components
cargo build --target wasm32-wasi --release

# Convert to components
wasm-tools component new user-service.wasm -o user.component.wasm
wasm-tools component new consumer.wasm -o consumer.component.wasm

# Compose components
wasm-tools compose consumer.component.wasm \
    --definitions user.component.wasm \
    -o composed.component.wasm
```

## Runtime Support

### wasmtime

```rust
use wasmtime::component::*;
use wasmtime::{Config, Engine, Store};

fn main() -> Result<()> {
    // Create engine with component support
    let mut config = Config::new();
    config.wasm_component_model(true);

    let engine = Engine::new(&config)?;
    let mut store = Store::new(&engine, ());

    // Load component
    let component = Component::from_file(&engine, "calculator.component.wasm")?;

    // Create linker
    let linker = Linker::new(&engine);

    // Instantiate
    let instance = linker.instantiate(&mut store, &component)?;

    // Get and call function
    let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
    let result = add.call(&mut store, (5, 3))?;

    println!("5 + 3 = {}", result);

    Ok(())
}
```

### JavaScript (jco)

```bash
# Transpile component to JavaScript
npx jco transpile calculator.component.wasm -o calculator

# Use in JavaScript
import { add, divide } from './calculator/calculator.js';

console.log(add(5, 3));  // 8

const result = divide(10, 2);
if (result.tag === 'ok') {
    console.log('Result:', result.val);
} else {
    console.error('Error:', result.val);
}
```

## WASI Preview 2

The Component Model enables WASI Preview 2 with rich interfaces:

**filesystem.wit**:
```wit
interface filesystem {
  record file-info {
    size: u64,
    modified: u64,
    is-directory: bool
  }

  variant error {
    not-found,
    permission-denied,
    io-error(string)
  }

  read-file: func(path: string) -> result<list<u8>, error>
  write-file: func(path: string, contents: list<u8>) -> result<_, error>
  list-directory: func(path: string) -> result<list<string>, error>
  get-file-info: func(path: string) -> result<file-info, error>
}
```

**http.wit**:
```wit
interface http {
  record request {
    method: string,
    uri: string,
    headers: list<tuple<string, string>>,
    body: option<list<u8>>
  }

  record response {
    status: u16,
    headers: list<tuple<string, string>>,
    body: option<list<u8>>
  }

  handle: func(req: request) -> response
}
```

## Virtual Adapters

Components can provide adapters for different interfaces:

**adapter.wit**:
```wit
world adapter {
  // Import old interface
  import legacy-api: interface {
    get-data: func(id: u32) -> list<u8>
  }

  // Export new interface
  export modern-api: interface {
    record data {
      id: u32,
      content: string,
      metadata: list<tuple<string, string>>
    }

    get-data: func(id: u32) -> result<data, string>
  }
}
```

**Implementation**:
```rust
wit_bindgen::generate!({
    world: "adapter",
    path: "../wit"
});

struct Adapter;

impl Guest for Adapter {
    fn get_data(id: u32) -> Result<Data, String> {
        // Call legacy API
        let raw_data = legacy_api::get_data(id);

        // Parse and transform
        let content = String::from_utf8(raw_data)
            .map_err(|e| format!("Invalid UTF-8: {}", e))?;

        Ok(Data {
            id,
            content,
            metadata: vec![
                ("source".to_string(), "legacy".to_string()),
            ],
        })
    }
}
```

## Resource Types

Resources represent stateful objects:

**resources.wit**:
```wit
interface database {
  resource connection {
    constructor(url: string)
    query: func(sql: string) -> result<list<row>, string>
    close: func()
  }

  record row {
    columns: list<column-value>
  }

  variant column-value {
    null,
    integer(s64),
    text(string),
    blob(list<u8>)
  }
}
```

**Rust implementation**:
```rust
wit_bindgen::generate!({
    world: "database-world",
    path: "../wit"
});

struct Connection {
    url: String,
    // Internal connection state
}

impl exports::database::Guest for Connection {
    fn new(url: String) -> Self {
        Connection {
            url,
            // Initialize connection
        }
    }

    fn query(&self, sql: String) -> Result<Vec<Row>, String> {
        // Execute query
        Ok(vec![])
    }

    fn close(self) {
        // Clean up connection
    }
}
```

## Versioning and Evolution

### Adding Optional Fields

```wit
// Version 1
record user-v1 {
  id: u32,
  name: string
}

// Version 2 (backward compatible)
record user-v2 {
  id: u32,
  name: string,
  email: option<string>  // Optional field
}
```

### Variant Extension

```wit
// Version 1
variant error-v1 {
  not-found,
  permission-denied
}

// Version 2 (backward compatible)
variant error-v2 {
  not-found,
  permission-denied,
  timeout,           // New variant
  rate-limited       // New variant
}
```

## Debugging Components

```bash
# Inspect component
wasm-tools component wit calculator.component.wasm

# Validate component
wasm-tools validate calculator.component.wasm --features component-model

# Print detailed info
wasm-tools print calculator.component.wasm
```

## Best Practices

1. **Start with WIT**: Design interfaces before implementation
2. **Use records**: Group related data
3. **Return results**: Use `result<T, E>` for errors
4. **Version carefully**: Plan for evolution
5. **Document interfaces**: Clear comments in WIT files
6. **Test composition**: Verify component interactions
7. **Minimize dependencies**: Fewer imports = more reusable
8. **Use resources wisely**: For stateful objects
9. **Consider performance**: Marshalling has overhead
10. **Follow conventions**: Consistent naming (kebab-case)

## Current Limitations

- **Tooling maturity**: Still evolving
- **Runtime support**: Limited (wasmtime, jco)
- **Language support**: Best for Rust, improving for others
- **Performance**: Some overhead vs core modules
- **Ecosystem**: Fewer libraries/components available

## Future Outlook

The Component Model will enable:

- **Universal packages**: Language-agnostic modules
- **Plugin ecosystems**: Standardized extension points
- **Microservices**: Type-safe service composition
- **Edge computing**: Composable edge functions
- **AI/ML**: Modular model serving
- **Gaming**: Component-based game engines

## Next Steps

The Component Model represents the future of WebAssembly, enabling rich, composable, language-agnostic modules. While still under development, it's already usable for building modular applications with strong type safety and seamless interoperability.

Next, we'll explore garbage collection in WebAssembly, a proposal that will enable languages like Java, Python, and JavaScript to compile more efficiently to WASM.
