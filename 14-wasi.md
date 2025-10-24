# Chapter 14: WASI - WebAssembly System Interface

## What is WASI?

WASI (WebAssembly System Interface) is a modular system interface for WebAssembly. It's essentially a **capability-based POSIX** for WASM, enabling portable, secure access to operating system resources.

Without WASI, WASM has no standard way to:
- Read or write files
- Access environment variables
- Get the current time
- Generate random numbers
- Accept command-line arguments
- Print to stdout/stderr

WASI solves this while maintaining WASM's security guarantees.

## Why WASI Matters

### Portability

Code compiled to WASM+WASI runs unchanged on:
- Linux
- macOS
- Windows
- BSD systems
- Embedded systems
- Cloud platforms

Write once, run anywhere—for real.

### Security

WASI is capability-based. A WASM module can only access what it's explicitly granted:

```bash
# Grant access only to current directory
wasmtime --dir=. program.wasm

# Grant access to specific directory
wasmtime --dir=/tmp program.wasm

# No filesystem access
wasmtime program.wasm
```

No ambient authority. No accidental leaks.

### Standard Interface

Before WASI, every runtime had custom imports for system access. WASI standardizes this, so code works across runtimes:

- Wasmtime
- Wasmer
- WasmEdge
- WAMR
- Node.js (via wasi polyfill)
- Deno

## WASI Architecture

WASI is modular, with multiple proposals:

### wasi_snapshot_preview1

The current stable version, covering:
- **File I/O**: Open, read, write, close files
- **Filesystem**: Create, delete, stat, rename
- **Sockets**: (Proposal stage, not yet in snapshot_preview1)
- **Environment**: Access environment variables, command-line args
- **Clocks**: Get current time, sleep
- **Random**: Generate cryptographically secure random numbers
- **Process**: Exit codes

### Future Modules

WASI is evolving with specialized modules:
- **wasi-sockets**: Network access
- **wasi-http**: HTTP client/server
- **wasi-crypto**: Cryptographic operations
- **wasi-ml**: Machine learning inference
- **wasi-nn**: Neural network inference

Each module is optional and can be granted independently.

## WASI Functions

### File I/O

**Opening files**:

```rust
use std::fs::File;
use std::io::Read;

fn main() {
    let mut file = File::open("input.txt").expect("Failed to open file");
    let mut contents = String::new();
    file.read_to_string(&mut contents).expect("Failed to read");
    println!("{}", contents);
}
```

Compiles to WASI:

```bash
rustc --target wasm32-wasi main.rs -o program.wasm
wasmtime --dir=. program.wasm
```

Under the hood, this calls:
- `path_open` - Open file with flags
- `fd_read` - Read bytes from file descriptor
- `fd_close` - Close file descriptor

**Writing files**:

```rust
use std::fs::File;
use std::io::Write;

fn main() {
    let mut file = File::create("output.txt").expect("Failed to create file");
    file.write_all(b"Hello, WASI!").expect("Failed to write");
}
```

Uses:
- `path_open` with create flags
- `fd_write` - Write bytes to file descriptor

### Stdio

Standard input, output, and error are pre-opened file descriptors:

```rust
use std::io::{stdin, stdout, stderr};
use std::io::Write;

fn main() {
    println!("This goes to stdout");
    eprintln!("This goes to stderr");

    let mut input = String::new();
    stdin().read_line(&mut input).expect("Failed to read stdin");
    println!("You entered: {}", input);
}
```

### Environment Variables

```rust
use std::env;

fn main() {
    for (key, value) in env::vars() {
        println!("{}: {}", key, value);
    }

    match env::var("HOME") {
        Ok(val) => println!("HOME = {}", val),
        Err(_) => println!("HOME not set"),
    }
}
```

Grant access:

```bash
wasmtime --env HOME=/home/user program.wasm
```

Or inherit from host:

```bash
wasmtime --inherit-env program.wasm
```

### Command-Line Arguments

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("Program: {}", args[0]);
    for (i, arg) in args.iter().skip(1).enumerate() {
        println!("Arg {}: {}", i, arg);
    }
}
```

Run:

```bash
wasmtime program.wasm arg1 arg2 arg3
```

### Time and Clocks

```rust
use std::time::{SystemTime, UNIX_EPOCH};

fn main() {
    let now = SystemTime::now();
    let duration = now.duration_since(UNIX_EPOCH).expect("Time went backwards");
    println!("Seconds since epoch: {}", duration.as_secs());
}
```

Uses `clock_time_get` WASI function.

### Random Numbers

```rust
use rand::Rng;

fn main() {
    let mut rng = rand::thread_rng();
    let n: u32 = rng.gen();
    println!("Random number: {}", n);
}
```

Uses `random_get` WASI function for cryptographically secure randomness.

### Exit Codes

```rust
use std::process;

fn main() {
    println!("Exiting with code 42");
    process::exit(42);
}
```

```bash
wasmtime program.wasm
echo $?  # 42
```

## Directory and File Permissions

WASI uses pre-opened directories (preopens) for security:

```bash
# Read-only access to /data
wasmtime --dir=/data::ro program.wasm

# Read-write access to /tmp
wasmtime --dir=/tmp::rw program.wasm

# Map guest path to host path
wasmtime --mapdir=/app:/home/user/myapp program.wasm
```

Inside WASM, code sees the guest paths (/data, /tmp, /app), but they map to host paths with controlled permissions.

### Path Capabilities

When you open /data/file.txt, WASI checks:
1. Was /data granted as a preopen?
2. Does the permission allow reading?
3. Is the path within /data (no escaping via ..)?

If any check fails, the operation fails.

## WASI in Different Languages

### Rust

Best support via `wasm32-wasi` target:

```bash
rustup target add wasm32-wasi
cargo build --target wasm32-wasi
```

Standard library mostly works:
- File I/O: ✓
- Environment variables: ✓
- Threading: ✗ (no thread support yet in WASI)
- Networking: ✗ (wasi-sockets not yet standardized)

### C/C++

Via wasi-sdk:

```bash
# Install wasi-sdk
wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-20/wasi-sdk-20.0-linux.tar.gz
tar xf wasi-sdk-20.0-linux.tar.gz

# Compile
/path/to/wasi-sdk/bin/clang --target=wasm32-wasi -o program.wasm program.c
```

Example:

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    printf("Hello from WASI!\n");

    FILE *f = fopen("output.txt", "w");
    if (f) {
        fprintf(f, "Written from C!\n");
        fclose(f);
    }

    return 0;
}
```

### Go (TinyGo)

TinyGo supports WASI:

```bash
tinygo build -target=wasi -o program.wasm main.go
wasmtime program.wasm
```

Example:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("Hello from Go+WASI!")

    data := []byte("Written from Go!\n")
    os.WriteFile("output.txt", data, 0644)
}
```

Note: Standard Go (gc) doesn't support WASI target yet. Use TinyGo.

### AssemblyScript

Limited WASI support via bindings:

```typescript
import "wasi"
import { Console } from "as-wasi"

Console.log("Hello from AssemblyScript!")
```

### C# (.NET)

Experimental WASI support:

```bash
dotnet add package Wasi.Sdk --prerelease
dotnet build -t:WasiBuild
```

## WASI API Reference

### File Descriptors

- `fd_read(fd, iovs, iovs_len) -> (errno, nread)`
- `fd_write(fd, iovs, iovs_len) -> (errno, nwritten)`
- `fd_close(fd) -> errno`
- `fd_seek(fd, offset, whence) -> (errno, newoffset)`
- `fd_tell(fd) -> (errno, offset)`
- `fd_fdstat_get(fd) -> (errno, stat)`

### Paths and Filesystem

- `path_open(dirfd, dirflags, path, oflags, fs_rights_base, fs_rights_inheriting, fdflags) -> (errno, fd)`
- `path_create_directory(dirfd, path) -> errno`
- `path_remove_directory(dirfd, path) -> errno`
- `path_unlink_file(dirfd, path) -> errno`
- `path_rename(old_dirfd, old_path, new_dirfd, new_path) -> errno`
- `path_filestat_get(dirfd, path) -> (errno, filestat)`

### Environment

- `environ_get(environ, environ_buf) -> errno`
- `environ_sizes_get() -> (errno, environ_count, environ_buf_size)`
- `args_get(argv, argv_buf) -> errno`
- `args_sizes_get() -> (errno, argc, argv_buf_size)`

### Clock

- `clock_time_get(clock_id, precision) -> (errno, timestamp)`

Clock IDs:
- `REALTIME`: Wall clock time
- `MONOTONIC`: Monotonically increasing time
- `PROCESS_CPUTIME_ID`: Process CPU time
- `THREAD_CPUTIME_ID`: Thread CPU time

### Random

- `random_get(buf, buf_len) -> errno`

Fills buffer with cryptographically secure random bytes.

### Process

- `proc_exit(exit_code) -> !` (never returns)

## WASI Permissions Model

### Rights System

WASI uses a rights system for fine-grained control:

**File descriptor rights**:
- `FD_READ`: Can read from fd
- `FD_WRITE`: Can write to fd
- `FD_SEEK`: Can seek in fd
- `FD_TELL`: Can get offset
- `FD_FDSTAT_SET_FLAGS`: Can change fd flags
- `FD_SYNC`: Can sync fd
- `FD_ADVISE`: Can provide advice
- `FD_ALLOCATE`: Can allocate space

**Path rights**:
- `PATH_CREATE_DIRECTORY`: Can create directories
- `PATH_CREATE_FILE`: Can create files
- `PATH_LINK_SOURCE`: Can create hard links (source)
- `PATH_LINK_TARGET`: Can create hard links (target)
- `PATH_OPEN`: Can open paths
- `PATH_READLINK`: Can read symlinks
- `PATH_RENAME_SOURCE`: Can rename (source)
- `PATH_RENAME_TARGET`: Can rename (target)
- `PATH_FILESTAT_GET`: Can stat files
- `PATH_FILESTAT_SET_SIZE`: Can truncate
- `PATH_FILESTAT_SET_TIMES`: Can set times
- `PATH_SYMLINK`: Can create symlinks
- `PATH_REMOVE_DIRECTORY`: Can remove directories
- `PATH_UNLINK_FILE`: Can unlink files

When opening a file, you request specific rights. The runtime grants only what's allowed by the preopen.

### Capability Inheritance

Child file descriptors inherit rights from their parent:

```
Directory /data (rights: READ, OPEN, FILESTAT_GET)
  ├─ Open /data/file.txt (inherits: READ, OPEN, FILESTAT_GET)
  └─ Open /data/subdir (inherits: READ, OPEN, FILESTAT_GET)
```

Can't escalate privileges—if parent doesn't have WRITE, children can't get WRITE.

## WASI in Practice

### Building a CLI Tool

```rust
use std::env;
use std::fs;
use std::io::{self, BufRead, BufReader};

fn main() -> io::Result<()> {
    let args: Vec<String> = env::args().collect();

    if args.len() < 2 {
        eprintln!("Usage: {} <filename>", args[0]);
        std::process::exit(1);
    }

    let file = fs::File::open(&args[1])?;
    let reader = BufReader::new(file);

    let mut line_count = 0;
    let mut word_count = 0;

    for line in reader.lines() {
        line_count += 1;
        word_count += line?.split_whitespace().count();
    }

    println!("Lines: {}", line_count);
    println!("Words: {}", word_count);

    Ok(())
}
```

Build and run:

```bash
cargo build --target wasm32-wasi --release
wasmtime --dir=. target/wasm32-wasi/release/wordcount.wasm input.txt
```

### Creating a Filter

```rust
use std::io::{self, BufRead, Write};

fn main() -> io::Result<()> {
    let stdin = io::stdin();
    let mut stdout = io::stdout();

    for line in stdin.lock().lines() {
        let line = line?;
        let upper = line.to_uppercase();
        writeln!(stdout, "{}", upper)?;
    }

    Ok(())
}
```

Use as a Unix filter:

```bash
echo "hello world" | wasmtime uppercase.wasm
# HELLO WORLD
```

### File Processing

```rust
use std::fs;
use std::io::{self, Read, Write};
use std::path::Path;

fn process_file(input: &Path, output: &Path) -> io::Result<()> {
    let mut data = String::new();
    fs::File::open(input)?.read_to_string(&mut data)?;

    // Process data (example: reverse lines)
    let processed: String = data
        .lines()
        .rev()
        .collect::<Vec<_>>()
        .join("\n");

    fs::File::create(output)?.write_all(processed.as_bytes())?;

    Ok(())
}

fn main() {
    let args: Vec<_> = std::env::args().collect();
    if args.len() != 3 {
        eprintln!("Usage: {} <input> <output>", args[0]);
        std::process::exit(1);
    }

    process_file(Path::new(&args[1]), Path::new(&args[2]))
        .expect("Failed to process file");
}
```

Run:

```bash
wasmtime --dir=. processor.wasm input.txt output.txt
```

## WASI Limitations (as of snapshot_preview1)

### No Threading

WASI doesn't yet support threads. Workarounds:
- Use async/await for concurrency
- Use multiple WASM instances
- Wait for wasi-threads proposal

### No Networking

Sockets aren't in snapshot_preview1. The wasi-sockets and wasi-http proposals will fix this.

Current workaround: Import host functions for network access.

### No Process Spawning

Can't spawn child processes. Workaround: Import host function to spawn processes.

### Limited Filesystem Operations

Some operations aren't supported:
- File permissions (chmod)
- Ownership (chown)
- Some extended attributes

## WASI Evolution

### Component Model

The next generation of WASI uses the Component Model for:
- Better composition
- High-level types (strings, lists, records)
- Language-neutral interfaces

### Preview 2

WASI Preview 2 introduces:
- `wasi:io` - Async I/O streams
- `wasi:filesystem` - Improved filesystem API
- `wasi:sockets` - Network sockets
- `wasi:http` - HTTP client and server

These use the Component Model and are designed for async, modern applications.

## Best Practices

1. **Minimal capabilities**: Only request necessary directory access
2. **Validate paths**: Check user-provided paths before use
3. **Handle errors**: WASI functions can fail; check return values
4. **Use standard libraries**: Rust's `std`, C's `libc` handle WASI details
5. **Test with different runtimes**: Ensure portability across wasmtime, wasmer, etc.
6. **Consider fallbacks**: Not all runtimes support all WASI features

## Next Steps

WASI unlocks system programming for WebAssembly. With file I/O, environment access, and standard interfaces, you can build CLI tools, server applications, and embedded programs in WASM. Next, we'll explore debugging—how to troubleshoot WASM modules when things go wrong.
