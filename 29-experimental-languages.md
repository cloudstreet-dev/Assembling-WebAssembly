# Chapter 29: Experimental Languages

## Emerging WASM-First Languages

Several new languages are designed specifically for WebAssembly, taking advantage of its capabilities while avoiding legacy constraints.

## Grain

**Website**: https://grain-lang.org/

Grain is a functional programming language built for WebAssembly from the ground up.

### Features

- Functional-first with ML-style syntax
- Static typing with type inference
- Immutable by default
- Algebraic data types
- Pattern matching
- Built-in WebAssembly support

### Example

```grain
// Basic functions
let add = (a, b) => a + b

let rec factorial = (n) => {
  if (n <= 1) {
    1
  } else {
    n * factorial(n - 1)
  }
}

// Pattern matching
enum Option<a> {
  Some(a),
  None
}

let getOrDefault = (opt, default) => {
  match (opt) {
    Some(value) => value,
    None => default
  }
}

// Records
record Point {
  x: Number,
  y: Number
}

let distance = (p: Point) => {
  sqrt(p.x * p.x + p.y * p.y)
}

// Lists and higher-order functions
let numbers = [1, 2, 3, 4, 5]
let doubled = List.map(x => x * 2, numbers)
let sum = List.reduce((acc, x) => acc + x, 0, numbers)
```

### Compilation

```bash
grain compile program.gr
wasmtime program.wasm
```

### Status

- **Maturity**: Early development
- **Ecosystem**: Growing but small
- **Production**: Not recommended yet

## Moonbit

**Website**: https://www.moonbitlang.com/

Moonbit is a Rust-like language optimized for WebAssembly, developed by the creators of ReScript.

### Features

- Rust-inspired syntax
- WASM-first design
- Efficient compilation
- Interop with JavaScript
- Modern type system

### Example

```moonbit
// Basic syntax
fn add(a : Int, b : Int) -> Int {
  a + b
}

// Structs and methods
struct Point {
  x : Double
  y : Double
}

impl Point {
  fn distance(self : Point) -> Double {
    (self.x * self.x + self.y * self.y).sqrt()
  }

  fn add(self : Point, other : Point) -> Point {
    Point { x: self.x + other.x, y: self.y + other.y }
  }
}

// Enums and pattern matching
enum Result<T, E> {
  Ok(T)
  Err(E)
}

fn divide(a : Int, b : Int) -> Result<Int, String> {
  if b == 0 {
    Err("Division by zero")
  } else {
    Ok(a / b)
  }
}

fn handle_result(r : Result<Int, String>) -> Int {
  match r {
    Ok(value) => value
    Err(msg) => {
      println(msg)
      0
    }
  }
}

// Generics
fn map<T, U>(list : List<T>, f : (T) -> U) -> List<U> {
  match list {
    Nil => Nil
    Cons(head, tail) => Cons(f(head), map(tail, f))
  }
}
```

### Build System

```bash
moonbit build
moonbit run
```

### Status

- **Maturity**: Beta/early release
- **Performance**: Excellent (optimized for WASM)
- **Ecosystem**: Nascent
- **Tooling**: Modern and fast

## Virgil

**Website**: https://github.com/titzer/virgil

Virgil is a fast, lightweight language that compiles to WASM and native code.

### Features

- Simple, C-like syntax
- Compile-time execution
- Zero-cost abstractions
- Tiny runtime
- Multiple backends (WASM, native)

### Example

```virgil
// Basic types and functions
def add(a: int, b: int) -> int {
    return a + b;
}

// Classes
class Point {
    var x: int;
    var y: int;

    new(x: int, y: int) {
        this.x = x;
        this.y = y;
    }

    def distance() -> int {
        return x * x + y * y; // Simplified for int
    }
}

// Variants (algebraic data types)
type Option<T> {
    case None;
    case Some(val: T);
}

def getOrElse<T>(opt: Option<T>, default: T) -> T {
    match (opt) {
        None => return default;
        Some(v) => return v;
    }
}

// Arrays and loops
def sum(arr: Array<int>) -> int {
    var total = 0;
    for (i < arr.length) {
        total = total + arr[i];
    }
    return total;
}
```

### Compilation

```bash
virgil compile -target=wasm program.v3
```

### Status

- **Maturity**: Stable but niche
- **Performance**: Excellent
- **Size**: Very small binaries
- **Use case**: Embedded systems, WASM experiments

## Forest

Forest is a typed functional language targeting WebAssembly.

### Features

- Pure functional
- Dependent types
- Type-level programming
- WASM optimization

### Example

```forest
// Pure functions
let add : Int -> Int -> Int
add = \a b -> a + b

// Higher-order functions
let map : (a -> b) -> List a -> List b
map f []     = []
map f (x:xs) = f x : map f xs

// Type-level programming
data Vec (n : Nat) (a : Type) where
  Nil  : Vec 0 a
  Cons : a -> Vec n a -> Vec (n + 1) a

-- Type-safe head (cannot fail)
head : Vec (n + 1) a -> a
head (Cons x xs) = x
```

### Status

- **Maturity**: Research/experimental
- **Focus**: Type safety and correctness
- **Use case**: Academic, research projects

## Motoko (for Internet Computer)

**Website**: https://internetcomputer.org/docs/motoko

Motoko is designed for the Internet Computer blockchain but compiles to WASM.

### Features

- Actor-based concurrency
- Async/await syntax
- Strong typing
- Blockchain integration

### Example

```motoko
actor Counter {
  var count : Nat = 0;

  public func increment() : async Nat {
    count += 1;
    count
  };

  public query func get() : async Nat {
    count
  };

  public func reset() : async () {
    count := 0;
  };
}

// Using actors
let counter = actor {
  public func test() : async () {
    let c : Counter = actor("counter-canister-id");
    await c.increment();
    let value = await c.get();
    Debug.print(debug_show(value));
  };
};
```

### Status

- **Maturity**: Production (for Internet Computer)
- **Ecosystem**: Internet Computer focused
- **Use case**: Blockchain smart contracts

## Comparison Matrix

| Language | Paradigm | Type System | Status | Best For |
|----------|----------|-------------|--------|----------|
| Grain | Functional | Static, inferred | Alpha | FP enthusiasts, experimentation |
| Moonbit | Multi-paradigm | Static, strong | Beta | General WASM development |
| Virgil | Imperative | Static | Stable | Embedded, small binaries |
| Forest | Functional | Dependent | Research | Type theory, research |
| Motoko | Actor-based | Static | Production | Internet Computer apps |

## Why Experimental Languages Matter

These languages explore:
- **WASM-native design**: Not constrained by legacy
- **New paradigms**: Actor models, dependent types
- **Optimization opportunities**: Designed for WASM from scratch
- **Future directions**: Influence mainstream languages

## Adoption Considerations

**Pros**:
- Cutting-edge features
- WASM-optimized
- Small communities (easier contributions)
- Learning opportunities

**Cons**:
- Immature tooling
- Breaking changes
- Small ecosystems
- Limited libraries
- Uncertain futures

## Getting Involved

1. **Try examples**: Most have online playgrounds
2. **Read documentation**: Understand the philosophy
3. **Join communities**: Discord, GitHub discussions
4. **Contribute**: Early projects welcome contributors
5. **Build projects**: Best way to learn

## Future Outlook

Some experimental languages will:
- Mature into production tools (like Rust once was)
- Influence mainstream languages
- Find niche applications
- Fade away

The experimentation is valuable regardless—it pushes WebAssembly forward and explores what's possible.

## Resources

- **Grain**: https://grain-lang.org/docs/guide
- **Moonbit**: https://www.moonbitlang.com/docs/
- **Virgil**: https://github.com/titzer/virgil/tree/master/doc
- **WASM Language Landscape**: https://github.com/appcypher/awesome-wasm-langs

Experimental languages represent the cutting edge of WebAssembly. While not ready for production, they're worth watching and experimenting with—they might just be the future of WASM development.
