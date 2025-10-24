# Chapter 11: Security Model

## Security by Design

WebAssembly was designed with security as a primary goal. Unlike native code or JavaScript, WASM runs in a hardened sandbox that provides:

- **Memory isolation** - Can't access memory it doesn't own
- **Type safety** - No type confusion attacks
- **Control flow integrity** - No arbitrary jumps or code injection
- **Capability-based security** - Explicit grants, not ambient authority
- **No undefined behavior** - Everything is well-defined

These guarantees are **enforced by the specification**. A conforming WASM engine must provide these protections.

## The Sandbox

### What the Sandbox Prevents

WASM code **cannot**:

- Access memory outside its linear memory
- Call functions not explicitly imported
- Access the file system directly
- Open network sockets directly
- Access the DOM directly (in browsers)
- Execute arbitrary machine code
- Forge function pointers
- Bypass type checking
- Escape the sandbox

Everything a WASM module does must go through explicit imports or its own linear memory.

### What the Sandbox Allows

WASM code **can**:

- Read and write its own linear memory
- Call imported functions (with host permission)
- Call its own functions
- Manipulate tables and globals
- Grow memory (within limits)
- Perform computation

All capabilities must be explicitly granted by the host.

## Memory Isolation

### Linear Memory is Sandboxed

Each WASM instance has its own linear memory. Instance A cannot access instance B's memory unless they explicitly share it via imports/exports.

```wasm
;; Instance A
(memory 1)
```

```wasm
;; Instance B
(memory 1)
```

These are **separate** memory spaces. No cross-instance access is possible.

### Bounds Checking

Every memory access is bounds-checked:

```wasm
i32.const 1000000  ;; Address beyond current memory
i32.load           ;; TRAP: out of bounds
```

There's no way to:
- Read beyond memory bounds
- Write beyond memory bounds
- Overflow a buffer to corrupt other memory

Bounds checks are mandatory. Engines may optimize them (e.g., using guard pages), but the safety guarantee remains.

### No Pointer Forgery

In WASM, pointers are just i32 indices into linear memory. You can compute any address, but you can't:

- Access memory outside the linear memory
- Access other instances' memory
- Access the host's memory

```wasm
i32.const -1       ;; Wraps to 0xFFFFFFFF
i32.load           ;; TRAP: out of bounds (assuming memory is smaller)
```

Even if you compute an arbitrary integer, bounds checking prevents escaping the sandbox.

## Type Safety

### Static Type Checking

WASM is statically typed. Every operation is type-checked at validation time:

```wasm
i32.const 5
f64.const 2.5
i32.add           ;; Error: i32.add expects [i32, i32], got [i32, f64]
```

Invalid modules are rejected before they run.

### No Type Confusion

You can't trick the type system:

```wasm
;; Can't treat a number as a function reference
i32.const 42
call_indirect     ;; Error: expects funcref, got i32
```

Function references (funcref) and data (i32/i64/f32/f64) are distinct. You can't forge a function pointer by manipulating integers.

### No Buffer Overflow Exploits

In C, buffer overflows can overwrite return addresses:

```c
char buffer[10];
strcpy(buffer, very_long_string);  // Overflow!
```

In WASM:

```wasm
;; Writing beyond bounds traps
i32.const 10      ;; Address beyond buffer
i32.const 42
i32.store         ;; TRAP: bounds check
```

Even if the bounds check somehow failed (it can't in a correct implementation), there are no return addresses in memory—they're in the VM's control structures, inaccessible to WASM.

## Control Flow Integrity

### Structured Control Flow

WASM doesn't have `goto` or arbitrary jumps. All control flow is structured:

```wasm
(block $label
  ;; Can only branch to $label
  br $label
)
```

You can't:
- Jump into the middle of a block
- Jump to an arbitrary address
- Overwrite return addresses

### Return Address Protection

Return addresses aren't stored in linear memory. They're managed by the VM. There's no way to manipulate them from WASM code.

### No Code Injection

You can't write executable code to memory and execute it. Linear memory is data-only. Code comes from the module's code section, which is immutable.

```wasm
;; Can't do this:
i32.const 0
;; Write machine code bytes to memory
i32.const 0x90909090
i32.store
;; Try to execute it (impossible in WASM)
```

Code and data are strictly separated.

## Capability-Based Security

### Principle of Least Privilege

WASM follows capability-based security: a module can only do what it's explicitly granted permission to do.

Want to log to console?

```wasm
(import "console" "log" (func $log (param i32)))
```

The host must provide this import. Without it, the module has no console access.

Want to access the file system?

```wasm
(import "wasi_snapshot_preview1" "fd_read" (func $fd_read ...))
```

The host provides (or denies) this capability.

### No Ambient Authority

Unlike traditional programs, WASM modules don't have ambient authority. They can't:

- Access environment variables (unless imported)
- Read files (unless file handles are provided)
- Make network requests (unless socket capabilities are granted)
- Access system APIs directly

Every capability is explicit and controllable.

### Capability Attenuation

The host can provide restricted capabilities:

```javascript
// Full console access
const fullImports = {
  console: {
    log: console.log,
    error: console.error,
    warn: console.warn
  }
};

// Restricted: only logging
const restrictedImports = {
  console: {
    log: console.log,
    error: () => {},  // No-op
    warn: () => {}    // No-op
  }
};

// No console access
const noImports = {};
```

The module works within whatever it's given.

## Validation and Guarantees

### Validation is Mandatory

Before a WASM module can execute, it must be validated:

1. **Type checking** - All operations have correct types
2. **Stack balancing** - Functions leave the stack in the expected state
3. **Control flow** - Blocks are well-nested, branches target valid labels
4. **Memory/table bounds** - Indices are valid at declaration time
5. **Constant expressions** - Initializers are valid constant expressions

Invalid modules are **rejected**. They never execute.

### No Undefined Behavior

Unlike C/C++, WASM has no undefined behavior. Everything is specified:

- **Integer overflow** - Wraps (defined behavior)
- **Division by zero** - Traps (well-defined error)
- **Out-of-bounds access** - Traps
- **Uninitialized variables** - Locals are zero-initialized
- **Invalid function calls** - Type-checked, trap if signature doesn't match

If something would be undefined in C, WASM either defines it or makes it trap.

## Attack Surface and Mitigations

### Side-Channel Attacks

WASM doesn't prevent all side-channel attacks:

**Timing attacks** - Code execution time can leak information:

```wasm
;; Time taken reveals password length
(func $check_password (param $input_ptr i32) (param $len i32) (result i32)
  ;; Compare byte-by-byte, return early on mismatch
  ;; Timing reveals first incorrect byte position
)
```

**Mitigation**: Use constant-time algorithms when needed.

**Spectre/Meltdown** - Speculative execution can leak data.

**Mitigation**: Engines may disable shared memory or use site isolation.

### Resource Exhaustion

WASM code can consume resources:

**Infinite loops:**

```wasm
(loop $forever
  br $forever
)
```

**Mitigation**: Host can timeout execution or provide a watchdog.

**Memory exhaustion:**

```wasm
(loop $alloc
  i32.const 1
  memory.grow
  br $alloc
)
```

**Mitigation**: Set memory limits, monitor growth.

### Integer Overflow

Integer overflow wraps in WASM (like C's unsigned):

```wasm
i32.const 0x7FFFFFFF  ;; INT_MAX
i32.const 1
i32.add               ;; 0x80000000 (wraps to negative in signed interpretation)
```

**Mitigation**: Use wider types, check for overflow manually if needed.

### Denial of Service

Malicious modules can cause DoS:

```wasm
;; Allocate maximum memory
i32.const 1000000
memory.grow

;; Infinite loop
(loop br 0)
```

**Mitigation**: Resource limits, timeouts, monitoring.

## Trust Boundaries

### Trusted: The WASM Engine

The engine (browser, wasmtime, wasmer) is trusted to:

- Correctly implement the specification
- Enforce sandboxing
- Validate modules
- Perform bounds checking

If the engine has bugs, security breaks. This is why engines are heavily audited.

### Untrusted: WASM Modules

WASM modules are untrusted. They're assumed to be potentially malicious. The sandbox must contain them.

### Trusted: Host Imports

Imports are provided by the host and are trusted. If the host provides a file access import, that's a security decision by the host.

## WASI Security Model

WASI (WebAssembly System Interface) extends capability-based security to system calls:

**File system access:**

```bash
wasmtime --dir=/tmp mymodule.wasm
```

The module can only access /tmp, not the entire filesystem.

**Network access:**

Some WASI implementations allow granular network permissions:

```bash
wasmtime --allow-network=https://api.example.com mymodule.wasm
```

WASI enables sandboxed system programs with fine-grained capabilities.

## Browser-Specific Security

### Same-Origin Policy

In browsers, WASM modules respect the same-origin policy. They can't:

- Load modules from other origins without CORS
- Access other origins' data

### Content Security Policy

WASM loading can be controlled via CSP:

```http
Content-Security-Policy: script-src 'self' 'wasm-unsafe-eval'
```

`'wasm-unsafe-eval'` is required for `WebAssembly.compile()` in some CSP modes.

### Cross-Origin Isolation

For some features (SharedArrayBuffer, high-resolution timers), the page must be cross-origin isolated:

```http
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

This prevents certain side-channel attacks.

## Security Best Practices

### For Module Authors

1. **Validate inputs** - Don't trust data from imports
2. **Use bounds-checked access** - Even though WASM enforces it, be explicit
3. **Avoid timing leaks** - Use constant-time algorithms for secrets
4. **Limit resource usage** - Don't allocate unbounded memory
5. **Handle errors gracefully** - Don't assume imports always succeed

### For Host Developers

1. **Minimize imports** - Provide only necessary capabilities
2. **Validate module sources** - Only load modules from trusted origins
3. **Set resource limits** - Limit memory, execution time
4. **Monitor resource usage** - Watch for DoS attempts
5. **Audit imported modules** - Review third-party WASM
6. **Use isolation** - Run untrusted modules in separate processes if possible

### For WASM Consumers

1. **Load from trusted sources** - Verify module integrity
2. **Use subresource integrity** - In browsers, use SRI hashes
3. **Audit dependencies** - Review what modules do
4. **Update engines regularly** - Security patches matter
5. **Enable security features** - Use CSP, COEP, COOP appropriately

## Real-World Security Considerations

### Running Untrusted Code

WASM is designed for this. Examples:

- **Code playgrounds** - Run user-submitted code safely
- **Plugin systems** - Third-party plugins can't escape
- **Serverless functions** - Isolate customer code

The sandbox makes this feasible.

### Cryptography

WASM is suitable for crypto:

- No undefined behavior
- Type safety prevents many bugs
- Constant-time operations are possible

But watch for:

- Timing side-channels
- Speculative execution leaks
- Insufficient randomness (need host import for random)

### Smart Contracts

Blockchain platforms use WASM for smart contracts:

- **Determinism** - WASM execution is deterministic
- **Isolation** - Contracts can't interfere with each other
- **Gas metering** - Track execution cost

WASM's security model is critical here.

## Limitations

WASM's security model doesn't protect against:

- **Logical bugs** - A module can have business logic flaws
- **Data exfiltration via imports** - If you import a "sendToServer" function, the module can use it
- **Social engineering** - Malicious modules can be designed to trick users
- **Host vulnerabilities** - Bugs in the host environment

WASM provides a sandbox, not a magic security solution.

## Future Enhancements

Proposals that improve security:

**Memory64** - 64-bit memory addressing with larger address space

**Type imports/exports** - Better interfaces between modules

**GC proposal** - Managed memory for high-level languages, reduces memory corruption risk

**Component model** - Better encapsulation and composition

## Auditing and Verification

Tools for analyzing WASM modules:

**wasm-validate** - Check module validity

```bash
wasm-validate module.wasm
```

**wasm-objdump** - Inspect module contents

```bash
wasm-objdump -x module.wasm
```

**Decompilers** - Convert WASM to readable format (wasm2wat)

**Static analysis** - Custom tools to check for specific patterns

## Incident Response

If a malicious module is detected:

1. **Terminate execution** - Kill the instance
2. **Revoke capabilities** - Remove host imports
3. **Analyze the module** - Decompile, inspect bytecode
4. **Update filters** - Block known-malicious modules
5. **Patch vulnerabilities** - If an engine bug was exploited

## Conclusion

WebAssembly's security model is comprehensive:

- **Sandboxing** prevents escape
- **Type safety** prevents confusion attacks
- **Memory isolation** prevents cross-instance attacks
- **Capability-based security** enables principle of least privilege
- **Validation** ensures only safe modules run

Combined, these make WASM one of the safest execution environments for untrusted code. Understanding the security guarantees—and their limits—is essential for building secure systems with WebAssembly.

## Next Steps

We've completed Part 2, covering runtime and execution. In Part 3, we'll explore the toolchain and ecosystem—compilers, runtimes, WASI, debugging, and testing. Understanding the tools is key to productively developing with WebAssembly.
