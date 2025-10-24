# Contributing to Assembling WebAssembly

Thank you for your interest in contributing to "Assembling WebAssembly"! This document provides guidelines for contributing to this open educational resource.

## How to Contribute

We welcome contributions of all kinds:

- **Corrections**: Fix typos, code errors, or technical inaccuracies
- **Clarifications**: Improve explanations or add missing context
- **Examples**: Add new code examples or improve existing ones
- **Updates**: Keep content current with evolving WebAssembly specifications
- **Translations**: Help translate chapters to other languages
- **Suggestions**: Propose new topics or chapter improvements

## Getting Started

### Prerequisites

To build and test the code examples, you'll need:

- **Rust** (1.70+): For Rust examples and wasm-pack
  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  rustup target add wasm32-unknown-unknown
  ```

- **wasm-pack**: For building Rust→WASM projects
  ```bash
  cargo install wasm-pack
  ```

- **Node.js** (18+): For JavaScript tooling
  ```bash
  # Install from https://nodejs.org/
  ```

- **Go/TinyGo** (optional): For Go examples
  ```bash
  # TinyGo: https://tinygo.org/getting-started/install/
  ```

- **Emscripten** (optional): For C/C++ examples
  ```bash
  git clone https://github.com/emscripten-core/emsdk.git
  cd emsdk
  ./emsdk install latest
  ./emsdk activate latest
  source ./emsdk_env.sh
  ```

- **wasmtime** (optional): For testing standalone WASM
  ```bash
  curl https://wasmtime.dev/install.sh -sSf | bash
  ```

### Repository Structure

```
Assembling-WebAssembly/
├── README.md                    # Book overview and table of contents
├── CONTRIBUTING.md              # This file
├── glossary.md                  # Term definitions
├── 01-introduction.md           # Chapter 1
├── 02-wat-basics.md            # Chapter 2
├── ...                          # Chapters 3-52
├── examples/                    # Standalone code examples
│   ├── 01-hello-world/
│   ├── 02-memory-access/
│   └── ...
└── exercises/                   # Practice exercises
    ├── solutions/
    └── tests/
```

## Contribution Guidelines

### Making Changes

1. **Fork the repository**
   ```bash
   # Click "Fork" on GitHub
   git clone https://github.com/YOUR-USERNAME/Assembling-WebAssembly.git
   cd Assembling-WebAssembly
   ```

2. **Create a branch**
   ```bash
   git checkout -b fix/chapter-05-typo
   # or
   git checkout -b feat/add-python-example
   ```

3. **Make your changes**
   - Follow the style guide below
   - Test all code examples
   - Update related documentation

4. **Commit with clear messages**
   ```bash
   git add .
   git commit -m "Fix typo in Chapter 5: Memory section

   - Correct explanation of memory.grow behavior
   - Add note about maximum memory limits"
   ```

5. **Push and create a pull request**
   ```bash
   git push origin fix/chapter-05-typo
   # Create PR on GitHub
   ```

### Commit Message Format

Use clear, descriptive commit messages:

```
<type>: <short summary>

<optional longer description>
```

**Types**:
- `fix`: Bug fixes, typos, errors
- `feat`: New features, examples, or content
- `docs`: Documentation improvements
- `refactor`: Code reorganization without behavior changes
- `test`: Adding or updating tests
- `chore`: Tooling, dependencies, or repo maintenance

**Examples**:
```
fix: Correct Rust ownership example in Chapter 17

The previous example had a borrow checker error.
Updated to use clone() for simplicity.

feat: Add Python/Pyodide example to Chapter 26

- Install instructions for Pyodide
- Example of NumPy integration
- Performance comparison with JavaScript

docs: Clarify memory alignment in Chapter 3

Added diagram showing byte alignment for i32.store
```

## Style Guide

### Markdown Formatting

#### Chapter Structure

Each chapter should follow this structure:

```markdown
# Chapter N: Title

## Section 1

Introduction paragraph explaining the concept.

### Subsection 1.1

Detailed explanation with examples.

## Section 2

...

## Best Practices

Practical guidelines and recommendations.

## Common Pitfalls

Common mistakes and how to avoid them.

## Next Steps

Brief conclusion and transition to next chapter.
```

#### Code Blocks

Always specify the language for syntax highlighting:

````markdown
```rust
fn main() {
    println!("Hello, WebAssembly!");
}
```

```javascript
const instance = await WebAssembly.instantiate(wasmBytes);
console.log(instance.exports.add(2, 3));
```

```wat
(module
  (func (export "add") (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )
)
```
````

#### Links

- Use relative links for internal references:
  ```markdown
  See [Chapter 3: Memory](03-memory.md) for details.
  ```

- Use absolute URLs for external resources:
  ```markdown
  Check the [WebAssembly specification](https://webassembly.github.io/spec/).
  ```

#### Emphasis

- Use **bold** for important terms on first use
- Use `code` for inline code, function names, and technical terms
- Use *italics* sparingly for emphasis

#### Lists

- Use `-` for unordered lists
- Use `1.` for ordered lists (auto-numbering)
- Maintain consistent indentation (2 spaces)

```markdown
Proper list formatting:

1. First item
   - Sub-item A
   - Sub-item B
2. Second item
3. Third item
```

### Code Style

#### WAT (WebAssembly Text Format)

```wat
;; Use comments to explain complex operations
(module
  ;; Import section
  (import "env" "memory" (memory 1))

  ;; Type definitions
  (type $binary-op (func (param i32 i32) (result i32)))

  ;; Function definitions with clear names
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add
  )

  ;; Exports
  (export "add" (func $add))
)
```

**Guidelines**:
- Use descriptive function and parameter names with `$`
- Add comments for non-obvious operations
- Group related declarations together
- Use consistent indentation (2 spaces)

#### Rust

```rust
//! Module-level documentation

use wasm_bindgen::prelude::*;

/// Brief description of function purpose.
///
/// # Arguments
/// * `x` - Description of parameter
///
/// # Returns
/// Description of return value
///
/// # Examples
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
#[wasm_bindgen]
pub fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

**Guidelines**:
- Follow [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- Use rustfmt for formatting: `cargo fmt`
- Use clippy for linting: `cargo clippy`
- Add documentation comments for public APIs
- Include examples in documentation

#### JavaScript

```javascript
/**
 * Load and instantiate a WebAssembly module.
 *
 * @param {string} url - Path to the WASM file
 * @param {object} imports - Import object
 * @returns {Promise<WebAssembly.Instance>}
 */
async function loadWasm(url, imports = {}) {
    const response = await fetch(url);
    const bytes = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(bytes, imports);
    return instance;
}
```

**Guidelines**:
- Use modern ES6+ syntax (async/await, arrow functions, const/let)
- Add JSDoc comments for functions
- Use descriptive variable names
- Handle errors appropriately

#### C/C++

```c
/**
 * Fibonacci function compiled to WebAssembly.
 *
 * @param n The input number
 * @return The nth Fibonacci number
 */
int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

**Guidelines**:
- Follow standard C/C++ formatting
- Use Emscripten attributes where appropriate: `EMSCRIPTEN_KEEPALIVE`
- Document memory management assumptions
- Note any platform-specific behavior

### Technical Writing

#### Clarity

- **Use simple language**: Avoid unnecessary jargon
- **Be specific**: "The function returns an i32" not "it gives back a number"
- **Active voice**: "The compiler generates code" not "code is generated"
- **Short sentences**: Break complex ideas into multiple sentences

#### Examples

Every concept should have:

1. **Explanation**: What it is and why it matters
2. **Syntax**: How to write it
3. **Example**: Working code demonstrating the concept
4. **Output**: What the code produces
5. **Use cases**: When to use this technique

#### Code Comments

```rust
// ✓ Good: Explains WHY
// Use saturating arithmetic to prevent overflow in audio processing
let sample = (left_sample + right_sample).saturating_div(2);

// ✗ Bad: States the obvious
// Divide by 2
let sample = (left_sample + right_sample) / 2;
```

#### Diagrams

When helpful, include ASCII diagrams:

```
Memory Layout:
┌─────────┬─────────┬─────────┬─────────┐
│  Byte 0 │  Byte 1 │  Byte 2 │  Byte 3 │
├─────────┴─────────┴─────────┴─────────┤
│           i32 value (4 bytes)          │
└────────────────────────────────────────┘
```

## Testing Code Examples

All code examples should be tested before submission.

### Rust Examples

```bash
# Create a test project
cargo new --lib example-test
cd example-test

# Add your code to src/lib.rs

# Build for wasm32
cargo build --target wasm32-unknown-unknown

# Or use wasm-pack
wasm-pack build --target web

# Run tests
cargo test
```

### WAT Examples

```bash
# Compile WAT to WASM
wat2wasm example.wat -o example.wasm

# Run with wasmtime
wasmtime example.wasm --invoke function_name arg1 arg2

# Or test in browser
python3 -m http.server
# Open http://localhost:8000
```

### JavaScript Integration

```bash
# Create test HTML
cat > test.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Test</title></head>
<body>
<script type="module">
async function test() {
    const response = await fetch('example.wasm');
    const bytes = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(bytes);

    console.log('Result:', instance.exports.test());
}
test();
</script>
</body>
</html>
EOF

# Serve and test
python3 -m http.server
```

## Reporting Issues

When reporting an issue, please include:

1. **Chapter and section**: Where the issue occurs
2. **Description**: What's wrong or unclear
3. **Expected vs. actual**: What you expected vs. what you see
4. **Environment**: Browser/runtime versions (if applicable)
5. **Steps to reproduce**: How to see the problem

**Example**:

```markdown
**Chapter**: 17 - Rust and WebAssembly
**Section**: "Using wasm-bindgen"

**Description**: The code example fails to compile with error:
"cannot find value `console` in module `web_sys`"

**Expected**: Code should compile and run
**Actual**: Compilation error

**Environment**:
- Rust 1.75.0
- wasm-bindgen 0.2.90

**Steps**:
1. Create new project: `cargo new example`
2. Add code from chapter to src/lib.rs
3. Run `wasm-pack build`
4. See error

**Possible fix**: Add `web-sys` with `console` feature to Cargo.toml
```

## Adding New Content

### New Chapters

If proposing a new chapter:

1. **Create an issue** describing the chapter
2. **Outline the content**: Main sections and topics
3. **Justify the addition**: Why this chapter is needed
4. **Estimate scope**: How long/detailed it will be
5. **Wait for feedback** before writing

### New Examples

When adding examples:

1. **Make them self-contained**: Include all necessary code
2. **Provide build instructions**: How to compile and run
3. **Add error handling**: Don't assume happy path
4. **Test thoroughly**: Verify on multiple browsers/runtimes
5. **Document assumptions**: Required versions, features, etc.

### Translations

We welcome translations! To translate a chapter:

1. **Create a language directory**: e.g., `translations/es/`
2. **Keep structure consistent**: Match original chapter structure
3. **Translate content, not code**: Code examples stay in English (with translated comments)
4. **Update links**: Point to translated versions where available
5. **Credit translators**: Add your name to the translation file

## Code Review Process

All contributions go through review:

1. **Automated checks**:
   - Markdown linting
   - Code formatting
   - Link validation
   - Spell checking

2. **Technical review**:
   - Code correctness
   - Technical accuracy
   - Best practices

3. **Editorial review**:
   - Clarity and readability
   - Consistency with existing content
   - Grammar and spelling

4. **Approval and merge**:
   - At least one maintainer approval required
   - CI must pass
   - Then merged to main branch

## Recognition

Contributors are recognized in:

- Commit history (preserved forever)
- Release notes for significant contributions
- Contributors section (coming soon)

## Questions?

If you have questions about contributing:

1. **Check existing issues**: Your question may be answered
2. **Open a discussion**: Use GitHub Discussions for questions
3. **Contact maintainers**: See README for contact information

## License

By contributing, you agree that your contributions will be licensed under the same license as the rest of the project (see LICENSE file).

---

**Thank you for helping improve "Assembling WebAssembly"!**

Your contributions help developers worldwide learn WebAssembly and build better applications. Every correction, clarification, and example makes this resource more valuable.

Happy contributing!
