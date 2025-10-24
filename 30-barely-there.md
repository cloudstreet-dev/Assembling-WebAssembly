# Chapter 30: Languages with Minimal WASM Support

## The Long Tail of WASM Support

Many languages have experimental, incomplete, or community-driven WebAssembly support. These implementations range from "technically works" to "proof of concept only."

This chapter surveys languages where WASM support exists but isn't production-ready or mainstream.

## Ruby (ruby.wasm)

**Status**: Experimental
**Project**: https://github.com/ruby/ruby.wasm

CRuby (the standard Ruby implementation) compiled to WASM via Emscripten.

### How to Use

```html
<script src="https://cdn.jsdelivr.net/npm/ruby-head-wasm-wasi@latest/dist/browser.script.iife.js"></script>
<script>
  const { DefaultRubyVM } = window["ruby-wasm-wasi"];

  const main = async () => {
    const response = await fetch("https://cdn.jsdelivr.net/npm/ruby-head-wasm-wasi@latest/dist/ruby.wasm");
    const buffer = await response.arrayBuffer();
    const module = await WebAssembly.compile(buffer);
    const { vm } = await DefaultRubyVM(module);

    vm.eval(`
      puts "Hello from Ruby!"

      def factorial(n)
        return 1 if n <= 1
        n * factorial(n - 1)
      end

      puts "Factorial of 5: #{factorial(5)}"
    `);
  };

  main();
</script>
```

### Limitations

- Large binary size (~10MB+)
- Slow startup
- Limited ecosystem (many gems won't work)
- No C extension support
- Experimental status

### Use Cases

- Ruby education in browser
- Demo/playground environments
- Not recommended for production

## PHP (php-wasm)

**Status**: Community experimental
**Project**: https://github.com/seanmorris/php-wasm

PHP compiled to WebAssembly via Emscripten.

### Example

```html
<script src="php-wasm.js"></script>
<script>
  const php = await PhpWeb.load();

  php.run(`<?php
    echo "Hello from PHP!\\n";

    function fibonacci($n) {
      if ($n <= 1) return $n;
      return fibonacci($n - 1) + fibonacci($n - 2);
    }

    echo "Fib(10): " . fibonacci(10) . "\\n";
  ?>`);
</script>
```

### Limitations

- Very large binary
- Most PHP extensions don't work
- Filesystem operations limited
- Not recommended for actual use

### WordPress in WASM

The WordPress Playground uses php-wasm to run WordPress entirely in the browser—impressive but experimental.

## Perl (webperl)

**Status**: Experimental
**Project**: https://github.com/haukex/webperl

Perl 5 compiled to WebAssembly.

### Example

```javascript
// Load Perl
const perl = await WebPerl.load();

// Run Perl code
perl.eval(`
  print "Hello from Perl!\\n";

  sub factorial {
    my $n = shift;
    return 1 if $n <= 1;
    return $n * factorial($n - 1);
  }

  print "Factorial: ", factorial(5), "\\n";
`);
```

### Limitations

- Large size
- Limited CPAN module support
- Primarily educational/demo use

## Dart (experimental)

**Status**: Officially experimental
**Project**: Part of Dart SDK

Dart is exploring WASM as a compilation target alongside its current JavaScript compilation.

### Flutter and WASM

Flutter (Dart's UI framework) is working on WASM support for web:

```dart
// Regular Dart/Flutter code
import 'package:flutter/material.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Center(
          child: Text('Flutter on WASM (experimental)'),
        ),
      ),
    );
  }
}
```

**Compile** (when stable):
```bash
flutter build web --wasm
```

### Status

- In development
- Not ready for production
- Performance improvements expected
- Watch for official releases

## Haskell (Asterius)

**Status**: Research/experimental
**Project**: https://github.com/tweag/asterius

Haskell compiled to WebAssembly via custom compiler.

### Example

```haskell
-- Export functions
foreign export javascript "add" add :: Int -> Int -> Int

add :: Int -> Int -> Int
add x y = x + y

-- Higher-order functions
foreign export javascript "map" jsMap :: (Int -> Int) -> [Int] -> [Int]

jsMap :: (Int -> Int) -> [Int] -> [Int]
jsMap = map

-- Example usage
main :: IO ()
main = putStrLn "Haskell WASM module compiled"
```

### Limitations

- Incomplete GHC support
- Large runtime
- Immature tooling
- Research project status

### Alternative: GHCJS

GHCJS compiles Haskell to JavaScript (not WASM), but it's more mature for web development.

## OCaml (experimental)

**Status**: Community experimental
**Projects**: wasm_of_ocaml, OCaml multicore with WASM backend

OCaml has experimental WASM support through various projects.

### Example

```ocaml
(* Export function *)
let add x y = x + y

(* Lists and pattern matching *)
let rec sum = function
  | [] -> 0
  | x :: xs -> x + sum xs

(* Algebraic data types *)
type tree =
  | Leaf
  | Node of int * tree * tree

let rec depth = function
  | Leaf -> 0
  | Node (_, left, right) ->
      1 + max (depth left) (depth right)
```

### Status

- Multiple competing approaches
- None production-ready
- OCaml 5 (multicore) may improve WASM story

## Elixir/Erlang (Lumen)

**Status**: Archived/abandoned
**Project**: https://github.com/lumen/lumen (no longer maintained)

Lumen was an attempt to compile Erlang/Elixir to WASM. The project has been abandoned.

### Why It Failed

- Erlang VM (BEAM) is complex
- Actor model doesn't map well to WASM
- Resource constraints
- Small team

### Lessons Learned

Not all languages map well to WebAssembly. Languages with:
- Complex runtimes (VMs within VMs)
- Thread-based concurrency models
- Dynamic code loading

...face significant challenges.

## Julia (experimental)

**Status**: Early research
**Approach**: Compile via LLVM to WASM

Julia uses LLVM, which has a WASM backend, but full Julia-to-WASM is not yet viable.

### Challenges

- JIT compilation model
- Large runtime requirements
- Dynamic typing challenges
- Package ecosystem dependencies

### Partial Success

Some Julia code can be compiled to WASM for specific use cases, but it's not a general solution.

## Lisp Dialects

Various Lisp implementations have WASM experiments:

### Scheme (BiwaScheme)

```scheme
;; BiwaScheme can run in browser (transpiled to JS, not WASM)
(define (factorial n)
  (if (<= n 1)
      1
      (* n (factorial (- n 1)))))

(display (factorial 5))
```

### Common Lisp (JSCL)

JSCL compiles Common Lisp to JavaScript (not WASM), but WASM backends are being explored.

## Why Some Languages Struggle with WASM

### Architecture Mismatches

1. **Dynamic code generation**: Languages that generate code at runtime (Julia, some Lisps)
2. **Complex runtimes**: Large VMs (JVM, BEAM) don't fit well
3. **Threading models**: Languages built around OS threads
4. **C extensions**: Heavy reliance on native extensions

### Resource Constraints

1. **Binary size**: Runtimes can be huge (10-30MB+)
2. **Startup time**: Initialization overhead
3. **Memory usage**: Large heap requirements

### Ecosystem Issues

1. **Library dependencies**: Native dependencies don't work
2. **Tooling immaturity**: Build systems, debuggers
3. **Community size**: Small teams can't sustain effort

## When "Barely There" is Good Enough

Despite limitations, these implementations can be useful for:

- **Education**: Teaching programming in the browser
- **Demos**: Showing off language features
- **Playgrounds**: Interactive coding environments
- **Code golf**: Fun experiments
- **Legacy code**: Running old code without setup

## The Future

Some "barely there" implementations will:
- **Mature**: Gain official support, stabilize
- **Fade**: Abandoned due to lack of interest/resources
- **Merge**: Unified efforts replace competing projects
- **Inspire**: Influence design of newer languages

## What We've Learned

Successful WASM language support requires:

1. **Official backing**: Or strong community
2. **Suitable architecture**: Static compilation, manageable runtime
3. **Sustained effort**: Years of development
4. **Use cases**: Clear value proposition
5. **Ecosystem**: Libraries, tools, documentation

## Recommendations

**For experimentation**: Try these languages if you're curious

**For production**: Stick to:
- Rust
- C/C++
- Go (official or TinyGo)
- AssemblyScript
- C#/Blazor (for .NET teams)

**For contributions**: These experimental projects need help! If you're interested in language implementation, contributing to a "barely there" WASM implementation is valuable experience.

## Resources

- **Awesome WASM Languages**: https://github.com/appcypher/awesome-wasm-langs
- **WASM Language Support Matrix**: Track which features work in each language
- **Language-specific forums**: Most have discussions about WASM progress

The "barely there" category is fluid—today's experiment might be tomorrow's production toolchain. Keep watching this space!

## Part 5 Summary

We've surveyed the WebAssembly language landscape:

- **Part 4** covered production-ready languages (Rust, Go, C/C++, AssemblyScript, .NET)
- **Part 5** explored alternatives:
  - **Zig**: Modern systems programming
  - **Swift**: iOS/web code sharing
  - **Kotlin**: Android/web multiplatform
  - **Python**: Data science in browser (Pyodide)
  - **Experimental**: Grain, Moonbit, Virgil, etc.
  - **Barely there**: Ruby, PHP, Perl, Haskell, etc.

Choose your language based on:
- Maturity requirements
- Performance needs
- Binary size constraints
- Ecosystem requirements
- Team expertise
- Project timeline

Next, in **Part 6**, we'll dive into web integration—how to effectively use WASM in browsers with JavaScript, DOM access, and modern web APIs.
