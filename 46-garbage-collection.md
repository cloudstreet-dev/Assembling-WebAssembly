# Chapter 46: Garbage Collection

## The GC Proposal

WebAssembly's Garbage Collection (GC) proposal adds native support for managed memory, enabling languages like Java, Kotlin, Dart, and others to compile more efficiently to WASM without shipping their own GC runtime.

**Status**: Proposal (Phase 4 - standardization)
**Goal**: First-class GC types in WASM

## Why GC in WASM?

### Current State (Linear Memory)

Languages with GC currently must:
1. Compile their own GC to WASM
2. Manage objects in linear memory
3. Ship large runtime (~MB+)
4. Pay performance costs

**Example**: Java/Kotlin via linear memory
- Ship entire JVM GC (~2MB+ overhead)
- All objects in linear memory
- No host GC integration
- Slower than native

### With GC Proposal

Languages can:
1. Use host's GC (browser's or runtime's)
2. Smaller binaries (no GC runtime)
3. Better performance
4. Interop with JavaScript objects

## New Reference Types

### Basic Types

```wat
;; Reference types
(type $object (struct))              ;; Generic object
(type $array (array i32))            ;; Array of i32
(type $string (array i16))           ;; UTF-16 string

;; Nullable vs non-nullable
anyref          ;; Nullable any reference
(ref $object)   ;; Non-null reference to $object
(ref null $object)  ;; Nullable reference
```

### Struct Types

```wat
(type $point (struct
  (field $x f64)
  (field $y f64)
))

(type $person (struct
  (field $name (ref $string))
  (field $age i32)
  (field $address (ref null $address))  ;; Nullable
))

(type $address (struct
  (field $street (ref $string))
  (field $city (ref $string))
  (field $zip i32)
))
```

### Array Types

```wat
;; Fixed element type
(type $int-array (array i32))
(type $float-array (array f64))

;; Reference arrays
(type $object-array (array (ref $object)))
(type $string-array (array (ref $string)))

;; Mutable vs immutable
(type $mutable-array (array (mut i32)))
(type $immutable-array (array i32))
```

## Creating and Accessing GC Objects

### Structs

```wat
(type $point (struct
  (field $x (mut f64))
  (field $y (mut f64))
))

(func $create-point (param $x f64) (param $y f64) (result (ref $point))
  ;; Create new struct
  (struct.new $point
    (local.get $x)
    (local.get $y)
  )
)

(func $get-x (param $p (ref $point)) (result f64)
  ;; Get field
  (struct.get $point $x (local.get $p))
)

(func $set-x (param $p (ref $point)) (param $new-x f64)
  ;; Set mutable field
  (struct.set $point $x (local.get $p) (local.get $new-x))
)

(func $distance (param $p1 (ref $point)) (param $p2 (ref $point)) (result f64)
  (local $dx f64)
  (local $dy f64)

  ;; Calculate dx = p2.x - p1.x
  (local.set $dx
    (f64.sub
      (struct.get $point $x (local.get $p2))
      (struct.get $point $x (local.get $p1))
    )
  )

  ;; Calculate dy = p2.y - p1.y
  (local.set $dy
    (f64.sub
      (struct.get $point $y (local.get $p2))
      (struct.get $point $y (local.get $p1))
    )
  )

  ;; Return sqrt(dx*dx + dy*dy)
  (f64.sqrt
    (f64.add
      (f64.mul (local.get $dx) (local.get $dx))
      (f64.mul (local.get $dy) (local.get $dy))
    )
  )
)
```

### Arrays

```wat
(type $int-array (array (mut i32)))

(func $create-array (param $size i32) (result (ref $int-array))
  ;; Create array filled with zeros
  (array.new_default $int-array (local.get $size))
)

(func $create-array-with-value (param $size i32) (param $value i32)
      (result (ref $int-array))
  ;; Create array filled with value
  (array.new $int-array (local.get $value) (local.get $size))
)

(func $get-element (param $arr (ref $int-array)) (param $index i32) (result i32)
  ;; Get array element
  (array.get $int-array (local.get $arr) (local.get $index))
)

(func $set-element (param $arr (ref $int-array)) (param $index i32) (param $value i32)
  ;; Set array element
  (array.set $int-array (local.get $arr) (local.get $index) (local.get $value))
)

(func $array-length (param $arr (ref $int-array)) (result i32)
  ;; Get array length
  (array.len (local.get $arr))
)

(func $sum-array (param $arr (ref $int-array)) (result i32)
  (local $i i32)
  (local $len i32)
  (local $sum i32)

  (local.set $len (array.len (local.get $arr)))
  (local.set $i (i32.const 0))
  (local.set $sum (i32.const 0))

  (block $break
    (loop $continue
      ;; Break if i >= len
      (br_if $break (i32.ge_u (local.get $i) (local.get $len)))

      ;; sum += arr[i]
      (local.set $sum
        (i32.add
          (local.get $sum)
          (array.get $int-array (local.get $arr) (local.get $i))
        )
      )

      ;; i++
      (local.set $i (i32.add (local.get $i) (i32.const 1)))

      (br $continue)
    )
  )

  (local.get $sum)
)
```

## Subtyping and Casting

### Type Hierarchy

```wat
(type $shape (struct
  (field $x f64)
  (field $y f64)
))

(type $circle (sub $shape (struct
  (field $x f64)
  (field $y f64)
  (field $radius f64)
)))

(type $rectangle (sub $shape (struct
  (field $x f64)
  (field $y f64)
  (field $width f64)
  (field $height f64)
)))

;; Circle and Rectangle are subtypes of Shape
```

### Type Tests and Casts

```wat
(func $area (param $shape (ref $shape)) (result f64)
  (local $circle (ref null $circle))
  (local $rect (ref null $rectangle))

  ;; Test if circle
  (if (result f64) (ref.test (ref $circle) (local.get $shape))
    (then
      ;; Cast to circle
      (local.set $circle (ref.cast (ref $circle) (local.get $shape)))

      ;; Calculate circle area: π * r²
      (f64.mul
        (f64.const 3.14159)
        (f64.mul
          (struct.get $circle $radius (local.get $circle))
          (struct.get $circle $radius (local.get $circle))
        )
      )
    )
    (else
      ;; Test if rectangle
      (if (result f64) (ref.test (ref $rectangle) (local.get $shape))
        (then
          (local.set $rect (ref.cast (ref $rectangle) (local.get $shape)))

          ;; Calculate rectangle area: width * height
          (f64.mul
            (struct.get $rectangle $width (local.get $rect))
            (struct.get $rectangle $height (local.get $rect))
          )
        )
        (else
          ;; Unknown shape
          (f64.const 0)
        )
      )
    )
  )
)
```

## Null Handling

```wat
(type $person (struct
  (field $name (ref $string))
  (field $age i32)
))

(func $get-age (param $p (ref null $person)) (result i32)
  ;; Check for null
  (if (result i32) (ref.is_null (local.get $p))
    (then
      (i32.const -1)  ;; Return -1 for null
    )
    (else
      ;; Safe to access
      (struct.get $person $age (local.get $p))
    )
  )
)

(func $safe-access (param $p (ref null $person)) (result i32)
  ;; Convert nullable to non-nullable (traps if null)
  (struct.get $person $age
    (ref.as_non_null (local.get $p))
  )
)
```

## Language Integration

### Java/Kotlin Example

**Kotlin code**:
```kotlin
data class Point(val x: Double, val y: Double) {
    fun distance(other: Point): Double {
        val dx = other.x - x
        val dy = other.y - y
        return sqrt(dx * dx + dy * dy)
    }
}

fun main() {
    val p1 = Point(0.0, 0.0)
    val p2 = Point(3.0, 4.0)
    println(p1.distance(p2))  // 5.0
}
```

**Compiles to** (simplified):
```wat
(type $Point (struct
  (field $x f64)
  (field $y f64)
))

(func $Point.distance (param $this (ref $Point)) (param $other (ref $Point))
      (result f64)
  (local $dx f64)
  (local $dy f64)

  (local.set $dx
    (f64.sub
      (struct.get $Point $x (local.get $other))
      (struct.get $Point $x (local.get $this))
    )
  )

  (local.set $dy
    (f64.sub
      (struct.get $Point $y (local.get $other))
      (struct.get $Point $y (local.get $this))
    )
  )

  (call $sqrt
    (f64.add
      (f64.mul (local.get $dx) (local.get $dx))
      (f64.mul (local.get $dy) (local.get $dy))
    )
  )
)
```

### Dart Example

**Dart code**:
```dart
class User {
  String name;
  int age;

  User(this.name, this.age);

  String describe() => "$name is $age years old";
}

void main() {
  var user = User("Alice", 30);
  print(user.describe());
}
```

**With GC proposal**:
- `User` → WASM struct
- `name` → GC-managed string
- Methods → WASM functions
- No need for Dart VM in WASM

## JavaScript Interop

### Accessing JS Objects from WASM

```javascript
// JavaScript
const obj = {
    x: 10,
    y: 20,
    getName: function() { return "Point"; }
};

// Pass to WASM
wasmInstance.exports.processObject(obj);
```

```wat
;; WASM with externref (JS object reference)
(func (export "processObject") (param $obj externref)
  ;; Call back to JS to access properties
  (call $js_get_x (local.get $obj))
  ;; ...
)
```

### Creating JS-Compatible Objects

```wat
;; Create object that JS can use
(type $js-point (struct
  (field $x (export "x") f64)
  (field $y (export "y") f64)
))

(func (export "createPoint") (param $x f64) (param $y f64)
      (result (ref $js-point))
  (struct.new $js-point (local.get $x) (local.get $y))
)
```

```javascript
// JavaScript
const point = wasmInstance.exports.createPoint(3.0, 4.0);
console.log(point.x, point.y);  // Access fields directly
```

## Performance Implications

### Benefits

1. **Smaller binaries**: No GC runtime needed
2. **Faster startup**: No GC initialization
3. **Better memory**: Host GC is optimized
4. **Interop**: Direct access to JS objects

### Considerations

1. **GC pauses**: Depends on host GC
2. **Memory pressure**: Shared with host
3. **Different semantics**: Host GC != language GC
4. **Debugging**: GC types harder to inspect

## Migration Strategy

### Before (Linear Memory)

```rust
// Rust simulating GC with linear memory
struct Object {
    data: Vec<u8>,
}

static mut OBJECTS: Vec<Object> = Vec::new();

#[no_mangle]
pub extern "C" fn create_object(size: usize) -> usize {
    unsafe {
        let obj = Object {
            data: vec![0; size],
        };
        OBJECTS.push(obj);
        OBJECTS.len() - 1  // Return handle
    }
}
```

### After (GC Types)

```wat
(type $object (array (mut i8)))

(func $create-object (param $size i32) (result (ref $object))
  ;; Direct GC allocation
  (array.new_default $object (local.get $size))
)
```

## Current Status

### Supported

- **Browsers**: Chrome 119+, Firefox (experimental)
- **Runtimes**: wasmtime (experimental), wasmer (in progress)
- **Languages**: Experimental support in Kotlin, Dart

### Not Yet

- **Production-ready**: Still experimental
- **Full language support**: Most languages not updated
- **Tooling**: Limited debugging/profiling support

## Example: Linked List

```wat
(type $node (struct
  (field $value i32)
  (field $next (ref null $node))
))

(func $create-list (param $values (ref $int-array)) (result (ref null $node))
  (local $i i32)
  (local $len i32)
  (local $head (ref null $node))
  (local $current (ref null $node))
  (local $new-node (ref $node))

  (local.set $len (array.len (local.get $values)))

  (if (i32.eqz (local.get $len))
    (then
      (return (ref.null $node))
    )
  )

  (local.set $i (i32.const 0))

  (loop $build
    ;; Create new node
    (local.set $new-node
      (struct.new $node
        (array.get $int-array (local.get $values) (local.get $i))
        (ref.null $node)
      )
    )

    (if (i32.eqz (local.get $i))
      (then
        ;; First node
        (local.set $head (local.get $new-node))
        (local.set $current (local.get $new-node))
      )
      (else
        ;; Link to previous
        (struct.set $node $next
          (local.get $current)
          (local.get $new-node)
        )
        (local.set $current (local.get $new-node))
      )
    )

    ;; Next iteration
    (local.set $i (i32.add (local.get $i) (i32.const 1)))
    (br_if $build (i32.lt_u (local.get $i) (local.get $len)))
  )

  (local.get $head)
)

(func $sum-list (param $head (ref null $node)) (result i32)
  (local $sum i32)
  (local $current (ref null $node))

  (local.set $sum (i32.const 0))
  (local.set $current (local.get $head))

  (block $break
    (loop $continue
      ;; Break if null
      (br_if $break (ref.is_null (local.get $current)))

      ;; Add value
      (local.set $sum
        (i32.add
          (local.get $sum)
          (struct.get $node $value (ref.as_non_null (local.get $current)))
        )
      )

      ;; Move to next
      (local.set $current
        (struct.get $node $next (ref.as_non_null (local.get $current)))
      )

      (br $continue)
    )
  )

  (local.get $sum)
)
```

## Best Practices

1. **Start experimenting**: Try GC proposal in non-production
2. **Benchmark**: Compare with linear memory approach
3. **Use for new code**: Easier than migrating
4. **Consider trade-offs**: GC pauses vs manual memory
5. **Test interop**: Ensure JS integration works
6. **Monitor support**: Track browser/runtime updates
7. **Plan migration**: Strategy for existing codebases
8. **Profile memory**: Understand GC behavior

## Future Outlook

The GC proposal will:

- Enable efficient compilation of managed languages
- Reduce binary sizes significantly
- Improve JavaScript interop
- Unlock new use cases (e.g., full JVM in browser)
- Standardize memory management across WASM

Languages actively working on GC support:
- **Kotlin/Wasm**: Official support coming
- **Dart**: Experimental implementation
- **Java**: GraalVM exploring
- **OCaml**: Considering
- **Scheme**: Multiple implementations planned

## Next Steps

WebAssembly's GC proposal represents a major evolution, enabling managed languages to compile efficiently without shipping their own GC runtime. While still experimental, it's rapidly maturing and will unlock new possibilities for WASM.

Next, we'll explore exception handling in WebAssembly, another proposal that enables languages with exceptions to compile more naturally to WASM.
