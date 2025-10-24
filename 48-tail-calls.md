# Chapter 48: Tail Call Optimization

## The Tail Calls Proposal

WebAssembly's Tail Calls proposal enables tail call optimization (TCO), allowing functions to call other functions (or themselves) in tail position without growing the call stack.

**Status**: Standardized (Phase 4)
**Browser Support**: Chrome 112+, Firefox 121+

## What are Tail Calls?

A **tail call** is a function call that is the last operation before returning:

```javascript
// Tail call
function factorial(n, acc = 1) {
    if (n <= 1) return acc;
    return factorial(n - 1, n * acc);  // ← Tail call
}

// NOT a tail call
function factorial_bad(n) {
    if (n <= 1) return 1;
    return n * factorial_bad(n - 1);  // ← Not tail (multiply after)
}
```

## Why Tail Calls Matter

### Stack Overflow Prevention

**Without TCO**:
```javascript
// Stack overflow after ~10,000 calls
function count(n) {
    if (n === 0) return 0;
    return count(n - 1);  // Each call adds stack frame
}

count(100000);  // Stack overflow!
```

**With TCO**:
```javascript
// Constant stack space, infinite recursion possible
function count(n) {
    if (n === 0) return 0;
    return count(n - 1);  // Tail call optimized
}

count(100000000);  // Works!
```

### Functional Programming

Enables functional patterns without stack limits:

```scheme
;; Scheme-style programming
(define (map f lst acc)
  (if (null? lst)
      (reverse acc)
      (map f (cdr lst) (cons (f (car lst)) acc))))  ;; Tail call
```

## WAT Syntax

### return_call

```wat
(func $factorial (param $n i32) (param $acc i32) (result i32)
  (if (i32.le_s (local.get $n) (i32.const 1))
    (then
      (return (local.get $acc))
    )
  )

  ;; Tail call - reuses current stack frame
  (return_call $factorial
    (i32.sub (local.get $n) (i32.const 1))
    (i32.mul (local.get $n) (local.get $acc))
  )
)
```

### return_call_indirect

```wat
(table funcref (elem $add $multiply))

(func $add (param i32 i32) (result i32)
  (i32.add (local.get 0) (local.get 1))
)

(func $multiply (param i32 i32) (result i32)
  (i32.mul (local.get 0) (local.get 1))
)

(func $apply (param $op i32) (param $a i32) (param $b i32) (result i32)
  ;; Indirect tail call
  (return_call_indirect (type $binary-op)
    (local.get $a)
    (local.get $b)
    (local.get $op)
  )
)
```

## Classic Examples

### Factorial (Tail Recursive)

```wat
;; Tail-recursive factorial
(func $factorial (param $n i32) (result i32)
  (call $factorial-helper (local.get $n) (i32.const 1))
)

(func $factorial-helper (param $n i32) (param $acc i32) (result i32)
  (if (i32.le_s (local.get $n) (i32.const 1))
    (then
      (return (local.get $acc))
    )
  )

  (return_call $factorial-helper
    (i32.sub (local.get $n) (i32.const 1))
    (i32.mul (local.get $n) (local.get $acc))
  )
)
```

### Fibonacci (Tail Recursive)

```wat
(func $fibonacci (param $n i32) (result i32)
  (call $fib-helper (local.get $n) (i32.const 0) (i32.const 1))
)

(func $fib-helper (param $n i32) (param $a i32) (param $b i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then
      (return (local.get $a))
    )
  )

  (return_call $fib-helper
    (i32.sub (local.get $n) (i32.const 1))
    (local.get $b)
    (i32.add (local.get $a) (local.get $b))
  )
)
```

### Sum of List

```wat
(func $sum-list (param $head i32) (result i32)
  (call $sum-helper (local.get $head) (i32.const 0))
)

(func $sum-helper (param $node i32) (param $acc i32) (result i32)
  ;; If null, return accumulator
  (if (i32.eqz (local.get $node))
    (then
      (return (local.get $acc))
    )
  )

  (local $value i32)
  (local $next i32)

  ;; Read node value and next pointer
  (local.set $value (i32.load (local.get $node)))
  (local.set $next (i32.load offset=4 (local.get $node)))

  ;; Tail call with updated accumulator
  (return_call $sum-helper
    (local.get $next)
    (i32.add (local.get $acc) (local.get $value))
  )
)
```

## Mutual Recursion

Tail calls enable mutual recursion:

```wat
;; Check if number is even
(func $is-even (param $n i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then
      (return (i32.const 1))  ;; 0 is even
    )
  )

  (return_call $is-odd
    (i32.sub (local.get $n) (i32.const 1))
  )
)

;; Check if number is odd
(func $is-odd (param $n i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then
      (return (i32.const 0))  ;; 0 is not odd
    )
  )

  (return_call $is-even
    (i32.sub (local.get $n) (i32.const 1))
  )
)
```

## State Machines

Tail calls enable elegant state machine implementations:

```wat
(func $state-init (param $input i32) (result i32)
  (if (i32.eq (local.get $input) (i32.const 65))  ;; 'A'
    (then
      (return_call $state-a (local.get $input))
    )
    (else
      (return_call $state-error)
    )
  )
)

(func $state-a (param $input i32) (result i32)
  ;; Read next input
  (local.set $input (call $read-input))

  (if (i32.eq (local.get $input) (i32.const 66))  ;; 'B'
    (then
      (return_call $state-b (local.get $input))
    )
    (else
      (return_call $state-error)
    )
  )
)

(func $state-b (param $input i32) (result i32)
  (local.set $input (call $read-input))

  (if (i32.eq (local.get $input) (i32.const 67))  ;; 'C'
    (then
      (return_call $state-accept)
    )
    (else
      (return_call $state-error)
    )
  )
)

(func $state-accept (result i32)
  (i32.const 1)  ;; Success
)

(func $state-error (result i32)
  (i32.const 0)  ;; Error
)
```

## Trampolines

Tail calls eliminate the need for trampolines:

**Without tail calls** (JavaScript):
```javascript
function trampoline(fn) {
    let result = fn();
    while (typeof result === 'function') {
        result = result();
    }
    return result;
}

function factorial(n, acc = 1) {
    if (n <= 1) return acc;
    return () => factorial(n - 1, n * acc);  // Return thunk
}

const result = trampoline(() => factorial(10000));
```

**With tail calls** (WAT):
```wat
;; Direct tail recursion, no trampoline needed
(func $factorial (param $n i32) (param $acc i32) (result i32)
  (if (i32.le_s (local.get $n) (i32.const 1))
    (then (return (local.get $acc)))
  )

  (return_call $factorial
    (i32.sub (local.get $n) (i32.const 1))
    (i32.mul (local.get $n) (local.get $acc))
  )
)
```

## Continuation-Passing Style (CPS)

Tail calls enable CPS transformations:

```wat
;; Regular style
(func $sum-list (param $list i32) (result i32)
  (if (i32.eqz (local.get $list))
    (then (return (i32.const 0)))
  )

  (i32.add
    (call $car (local.get $list))
    (call $sum-list (call $cdr (local.get $list)))
  )
)

;; CPS style with tail calls
(func $sum-list-cps (param $list i32) (param $cont i32) (result i32)
  (if (i32.eqz (local.get $list))
    (then
      (return_call_indirect (type $continuation)
        (i32.const 0)
        (local.get $cont)
      )
    )
  )

  (return_call $sum-list-cps
    (call $cdr (local.get $list))
    (call $make-continuation
      (call $car (local.get $list))
      (local.get $cont)
    )
  )
)
```

## Compiler Usage

### Rust

```rust
// Rust doesn't guarantee TCO, but you can enable for WASM

fn factorial(n: u32, acc: u32) -> u32 {
    if n <= 1 {
        acc
    } else {
        factorial(n - 1, n * acc)  // May be tail-call optimized
    }
}
```

**Check generated WAT**:
```bash
cargo build --target wasm32-unknown-unknown --release
wasm2wat target/wasm32-unknown-unknown/release/app.wasm | grep return_call
```

### Scheme to WASM

Scheme naturally uses tail calls:

```scheme
(define (factorial n acc)
  (if (<= n 1)
      acc
      (factorial (- n 1) (* n acc))))

;; Compiles to return_call in WASM
```

### OCaml

OCaml guarantees tail call optimization:

```ocaml
let rec factorial n acc =
  if n <= 1 then acc
  else factorial (n - 1) (n * acc)

(* Always tail-call optimized *)
```

## Performance Characteristics

### Stack Usage

**Without TCO**:
- Stack depth = recursion depth
- O(n) space complexity
- Stack overflow risk

**With TCO**:
- Stack depth = constant (1 frame)
- O(1) space complexity
- No stack overflow

### Time Complexity

Tail calls have minimal overhead:

```wat
;; Regular call
(call $func (local.get $arg))
;; ~5-10 cycles

;; Tail call
(return_call $func (local.get $arg))
;; ~5-10 cycles (same!)
```

### Comparison

```javascript
// Benchmark: Sum 1 to 1,000,000

// Without TCO (loop)
function sumLoop(n) {
    let sum = 0;
    for (let i = 1; i <= n; i++) {
        sum += i;
    }
    return sum;
}

// With TCO (recursion)
function sumTailRec(n, acc = 0) {
    if (n === 0) return acc;
    return sumTailRec(n - 1, acc + n);
}

// Results:
// Loop: ~1ms
// Tail recursion with TCO: ~1.2ms
// Tail recursion without TCO: Stack overflow!
```

## Debugging Tail Calls

### Stack Traces

Tail calls don't add stack frames:

```
// Without tail calls
at factorial (wasm:0x123)
at factorial (wasm:0x123)
at factorial (wasm:0x123)
... (1000 more)
at main (wasm:0x456)

// With tail calls
at factorial (wasm:0x123)
at main (wasm:0x456)
```

This can make debugging harder (less context) but prevents stack overflow.

### Debugging Tips

1. **Add logging**: Print arguments before tail calls
2. **Use counters**: Track recursion depth manually
3. **Conditional tail calls**: Only optimize in release builds
4. **Trace manually**: Add trace parameter

```wat
;; With trace parameter
(func $factorial (param $n i32) (param $acc i32) (param $trace i32) (result i32)
  (if (local.get $trace)
    (then
      (call $log (local.get $n) (local.get $acc))
    )
  )

  (if (i32.le_s (local.get $n) (i32.const 1))
    (then (return (local.get $acc)))
  )

  (return_call $factorial
    (i32.sub (local.get $n) (i32.const 1))
    (i32.mul (local.get $n) (local.get $acc))
    (local.get $trace)
  )
)
```

## When NOT to Use Tail Calls

1. **Need stack traces**: For debugging
2. **Not in tail position**: Can't optimize
3. **Performance critical**: Loop might be faster
4. **Complex control flow**: Hard to convert to tail form

```wat
;; Can't use tail call (not in tail position)
(func $fibonacci-bad (param $n i32) (result i32)
  (if (i32.le_s (local.get $n) (i32.const 1))
    (then (return (i32.const 1)))
  )

  ;; Not tail calls - addition happens after recursion
  (i32.add
    (call $fibonacci-bad (i32.sub (local.get $n) (i32.const 1)))
    (call $fibonacci-bad (i32.sub (local.get $n) (i32.const 2)))
  )
)
```

## Converting to Tail Form

### Accumulator Pattern

**Non-tail**:
```wat
(func $sum (param $n i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then (return (i32.const 0)))
  )

  (i32.add
    (local.get $n)
    (call $sum (i32.sub (local.get $n) (i32.const 1)))
  )
)
```

**Tail form**:
```wat
(func $sum (param $n i32) (result i32)
  (call $sum-helper (local.get $n) (i32.const 0))
)

(func $sum-helper (param $n i32) (param $acc i32) (result i32)
  (if (i32.eqz (local.get $n))
    (then (return (local.get $acc)))
  )

  (return_call $sum-helper
    (i32.sub (local.get $n) (i32.const 1))
    (i32.add (local.get $acc) (local.get $n))
  )
)
```

### Continuation Passing

**Non-tail**:
```wat
(func $process-list (param $list i32) (result i32)
  ;; Process each element, but not tail recursive
)
```

**Tail form with continuations**:
```wat
(func $process-list (param $list i32) (param $cont i32) (result i32)
  ;; Use continuations to make tail recursive
)
```

## Best Practices

1. **Use for recursive algorithms**: Natural fit
2. **Accumulator pattern**: Convert non-tail to tail
3. **State machines**: Elegant with tail calls
4. **Document intent**: Comment tail calls clearly
5. **Test stack usage**: Verify no stack growth
6. **Profile**: Measure actual performance
7. **Consider alternatives**: Sometimes loops are better
8. **Plan for debugging**: Add trace functionality

## Browser Support

Check for tail call support:

```javascript
const supportsTailCalls = (() => {
    try {
        new WebAssembly.Module(new Uint8Array([
            0x00, 0x61, 0x73, 0x6d,  // magic
            0x01, 0x00, 0x00, 0x00,  // version
            0x01, 0x04, 0x01,        // type section
            0x60, 0x00, 0x00,        // function type
            0x03, 0x02, 0x01, 0x00,  // function section
            0x0a, 0x06, 0x01,        // code section
            0x04, 0x00,              // function body
            0x12, 0x00,              // return_call 0
            0x0b                     // end
        ]));
        return true;
    } catch {
        return false;
    }
})();

console.log('Tail calls supported:', supportsTailCalls);
```

## Next Steps

Tail call optimization in WebAssembly enables efficient recursion and functional programming patterns without stack growth. By using `return_call` and `return_call_indirect`, you can write elegant recursive algorithms that run in constant stack space.

This completes **Part 9: Advanced Topics**. We've covered SIMD, threading, the Component Model, garbage collection, exception handling, and tail calls—the cutting-edge features shaping WebAssembly's future.

Next, in **Part 10: Practice**, we'll put everything together with real-world projects and practical applications of WebAssembly.
