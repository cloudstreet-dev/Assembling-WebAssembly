# Chapter 10: Tables

## What Are Tables?

Tables are WebAssembly's mechanism for storing references—either function references (funcref) or external references (externref). They're one of WASM's most powerful features, enabling:

- **Indirect function calls** - Call functions dynamically
- **Dynamic linking** - Add/change functions at runtime
- **Callbacks** - Store function pointers for later invocation
- **Virtual dispatch** - Implement vtables for object-oriented patterns
- **Plugin systems** - Load code dynamically

## Why Tables Instead of Memory?

You might ask: why not store function pointers in linear memory as integers?

**Security.** If function pointers were integers in memory, you could:
- Forge function pointers by writing arbitrary integers
- Call arbitrary code by manipulating memory
- Break the sandbox

Tables prevent this. Function references are **opaque**—you can't forge them, inspect them as integers, or manipulate them arbitrarily. The only way to get a funcref is through explicit WASM instructions.

This makes indirect calls safe.

## Table Types

Tables have two element types:

### funcref - Function References

```wasm
(table 10 funcref)  ;; 10 slots for function references
```

Used for:
- Indirect function calls
- Virtual dispatch
- Callback storage

### externref - External References

```wasm
(table 50 externref)  ;; 50 slots for external references
```

Used for:
- Storing JavaScript objects (in browsers)
- Storing host objects (in other environments)
- Interfacing with garbage-collected host systems

An externref is completely opaque to WASM. You can store it, pass it around, return it to the host, but you can't inspect its contents or manipulate it from WASM.

## Declaring Tables

### Basic Declaration

```wasm
(table 10 funcref)           ;; Min 10 slots, no max
(table 5 20 externref)       ;; Min 5, max 20 slots
(table $dispatch 100 funcref) ;; Named table
```

Like memory, tables have minimum and optional maximum sizes.

### Importing Tables

```wasm
(import "env" "table" (table 50 funcref))
```

Multiple modules can share a table, enabling dynamic linking.

### Exporting Tables

```wasm
(table 10 funcref)
(export "table" (table 0))
```

The host can access and manipulate the exported table.

## Initializing Tables

### Element Segments

Element segments populate tables with function references:

```wasm
(module
  (table 10 funcref)

  (func $f0 (result i32) i32.const 0)
  (func $f1 (result i32) i32.const 1)
  (func $f2 (result i32) i32.const 2)

  ;; Initialize table[0..2]
  (elem (i32.const 0) $f0 $f1 $f2)
)
```

This runs at instantiation, before any code executes.

### Active vs. Passive Elements

**Active segments** (default):

```wasm
(elem (i32.const 5) $func1 $func2)  ;; Initializes table[5..6]
```

Runs automatically during instantiation.

**Passive segments:**

```wasm
(elem $funcs funcref (ref.func $f1) (ref.func $f2))

(func $init
  i32.const 10     ;; Destination index in table
  i32.const 0      ;; Source offset in segment
  i32.const 2      ;; Count
  table.init $funcs  ;; Copy segment to table
  elem.drop $funcs   ;; Drop segment (free memory)
)
```

Passive segments allow lazy initialization.

## Table Instructions

### table.get

Get a reference from the table:

```wasm
i32.const 5     ;; Index
table.get       ;; Get table[5], push funcref or externref
```

Returns the reference at the given index. Traps if index is out of bounds.

### table.set

Set a reference in the table:

```wasm
i32.const 7     ;; Index
ref.func $foo   ;; Function reference
table.set       ;; Set table[7] = $foo
```

Traps if index is out of bounds.

### table.size

Get current table size:

```wasm
table.size      ;; Push size (i32)
```

### table.grow

Grow the table:

```wasm
ref.null func   ;; Initial value for new slots (null funcref)
i32.const 5     ;; Number of slots to add
table.grow      ;; Returns previous size, or -1 on failure
```

Like memory.grow, table.grow can fail if it would exceed the maximum or if the runtime can't allocate.

### table.fill

Fill a range with a value:

```wasm
i32.const 0     ;; Start index
ref.null func   ;; Value (null funcref)
i32.const 10    ;; Count
table.fill      ;; Fill table[0..9] with null
```

### table.copy

Copy elements within or between tables:

```wasm
i32.const 10    ;; Destination index
i32.const 0     ;; Source index
i32.const 5     ;; Count
table.copy      ;; Copy table[0..4] to table[10..14]
```

### table.init

Copy from a passive element segment:

```wasm
(elem $funcs funcref (ref.func $f1) (ref.func $f2))

(func $init
  i32.const 0      ;; Destination in table
  i32.const 0      ;; Source offset in segment
  i32.const 2      ;; Count
  table.init $funcs
  elem.drop $funcs
)
```

## Indirect Function Calls

The primary use of tables: calling functions indirectly.

### call_indirect

```wasm
(type $callback (func (param i32) (result i32)))
(table 10 funcref)

(func $double (param i32) (result i32)
  local.get 0
  i32.const 2
  i32.mul
)

(func $triple (param i32) (result i32)
  local.get 0
  i32.const 3
  i32.mul
)

(elem (i32.const 0) $double $triple)

(func $apply (param $func_index i32) (param $value i32) (result i32)
  local.get $value
  local.get $func_index
  call_indirect (type $callback)  ;; Call table[$func_index]($value)
)
```

**How call_indirect works:**

1. Pop the function index from the stack
2. Look up table[index]
3. Check that the function has the expected type signature
4. Pop parameters from the stack
5. Call the function
6. Push result

If the index is out of bounds or the signature doesn't match, it traps.

### Type Checking

call_indirect requires a type annotation:

```wasm
call_indirect (type $callback)
```

At runtime, the actual function's signature is checked against $callback. If they don't match, it traps.

This prevents calling a function with the wrong parameters or expecting the wrong return type.

## Patterns and Use Cases

### Function Pointer Arrays

Store callbacks:

```wasm
(type $handler (func (param i32)))
(table $handlers 100 funcref)

(func $on_click (param i32) ...)
(func $on_keypress (param i32) ...)

(func $register_handlers
  i32.const 0
  ref.func $on_click
  table.set

  i32.const 1
  ref.func $on_keypress
  table.set
)

(func $dispatch_event (param $event_type i32) (param $data i32)
  local.get $data
  local.get $event_type
  call_indirect (type $handler)
)
```

### Virtual Dispatch (OOP)

Implement virtual method tables:

```wasm
;; Vtable layout:
;; [0] = Dog.speak
;; [1] = Dog.run
;; [2] = Cat.speak
;; [3] = Cat.run

(type $method (func (param i32)))

(func $dog_speak (param $this i32) ...)
(func $dog_run (param $this i32) ...)
(func $cat_speak (param $this i32) ...)
(func $cat_run (param $this i32) ...)

(table $vtable 10 funcref)
(elem (i32.const 0) $dog_speak $dog_run $cat_speak $cat_run)

;; Object layout in memory:
;; [0..3] = vtable index (0 for Dog, 2 for Cat)
;; [4..7] = data fields

(func $call_method (param $obj_ptr i32) (param $method_offset i32)
  ;; Load vtable base
  local.get $obj_ptr
  i32.load  ;; Get vtable index

  ;; Add method offset
  local.get $method_offset
  i32.add

  ;; Call method
  local.get $obj_ptr
  swap  ;; Implementation detail: get function index on top
  call_indirect (type $method)
)
```

### Dynamic Code Loading

Load functions into table at runtime:

```wasm
(table $plugins 50 funcref)

(func (export "register_plugin") (param $index i32) (param $func_ref funcref)
  local.get $index
  local.get $func_ref
  table.set
)

(func (export "call_plugin") (param $index i32) (param $arg i32) (result i32)
  local.get $arg
  local.get $index
  call_indirect (type $plugin_func)
)
```

From JavaScript:

```javascript
const plugin = instance.exports.some_function;
instance.exports.register_plugin(0, plugin);
```

### Switch/Case Dispatch

Optimize switch statements:

```wasm
(type $case_handler (func))
(table $cases 10 funcref)

(func $case0 ...)
(func $case1 ...)
(func $case2 ...)

(elem (i32.const 0) $case0 $case1 $case2)

(func $switch (param $value i32)
  local.get $value
  call_indirect (type $case_handler)  ;; Jump directly to case handler
)
```

This is more efficient than a series of if/else.

## externref Tables

externref tables store host objects:

```wasm
(table $objects 100 externref)

(func (export "store_object") (param $index i32) (param $obj externref)
  local.get $index
  local.get $obj
  table.set
)

(func (export "get_object") (param $index i32) (result externref)
  local.get $index
  table.get
)
```

From JavaScript:

```javascript
const obj = { name: "test", value: 42 };
instance.exports.store_object(0, obj);

const retrieved = instance.exports.get_object(0);
console.log(retrieved.name);  // "test"
```

WASM can't inspect the object, but it can store and return it to JavaScript.

### Use Cases for externref

- **DOM elements** - Store references to DOM nodes
- **JavaScript objects** - Pass complex objects without serialization
- **Callbacks** - Store JavaScript functions (though funcref is often better)
- **Resources** - Store handles to files, sockets, etc.

## Multi-Table Support

Currently, most engines support one table per module. The multi-table proposal allows multiple tables:

```wasm
(table $funcs 50 funcref)
(table $objects 100 externref)

table.get $funcs
table.set $objects
```

This enables:
- Separate tables for functions and objects
- Multiple function tables for different purposes
- Better organization

## Performance Considerations

### Direct vs. Indirect Calls

**Direct calls** are faster:

```wasm
call $function  ;; Direct: ~1-2 cycles
```

**Indirect calls** are slower but still fast:

```wasm
call_indirect (type $sig)  ;; Indirect: ~5-10 cycles
```

The overhead includes:
- Table lookup
- Type check
- Indirect jump (harder to predict)

Still, indirect calls are much faster than calling through imports.

### Table Size

Large tables aren't a problem—memory overhead is minimal (few bytes per slot). Accessing any element is O(1).

### Type Check Optimization

Many engines optimize the type check:
- If the type is known at compile time, the check may be eliminated
- Some engines use inline caching

## Bounds Checking

All table accesses are bounds-checked:

```wasm
(table 10 funcref)

i32.const 20  ;; Out of bounds
table.get     ;; TRAP
```

There's no way to access memory beyond the table's current size.

## Debugging Tables

### Inspecting Tables in JavaScript

```javascript
const table = instance.exports.table;
console.log('Table size:', table.length);
console.log('Table[0]:', table.get(0));

table.set(5, instance.exports.some_function);
```

### Browser DevTools

- Set breakpoints before call_indirect
- Inspect table contents
- Check function signatures

### Disassembly

```bash
wasm-objdump -x module.wasm | grep -A10 "Table\|Element"
```

Shows table definitions and element segments.

## Best Practices

1. **Use tables for polymorphism** - Better than if/else chains
2. **Validate indices** - Check bounds before call_indirect if possible
3. **Group related functions** - Organize tables logically
4. **Initialize eagerly** - Use active segments when possible
5. **Document table layout** - Comment what each slot contains
6. **Prefer funcref** - Use externref only when necessary
7. **Consider type safety** - Ensure signatures match at instantiation

## Common Pitfalls

**Forgetting type annotations:**

```wasm
call_indirect  ;; Error: missing type
```

**Index confusion:**

```wasm
;; Pushing index after parameters
local.get $param
i32.const 5
call_indirect (type $sig)  ;; Wrong order!

;; Correct:
i32.const 5
local.get $param
call_indirect (type $sig)  ;; Index is popped last (top of stack)
```

**Uninitialized slots:**

```wasm
(table 10 funcref)  ;; All slots are null

i32.const 5
call_indirect  ;; TRAP: slot is null
```

Initialize all slots you'll use, or check for null.

## Next Steps

We've covered tables—how they work, how to use them, and patterns for indirect dispatch. Next, we'll examine WebAssembly's security model, understanding how sandboxing, capability-based security, and memory isolation keep WASM code safe.
