# Chapter 37: CLI Tools with WebAssembly

## Why WASM for CLI Tools?

WebAssembly enables writing portable command-line tools that:
- Run on any platform (Windows, macOS, Linux) without recompilation
- Execute in sandboxed environments with controlled system access
- Provide fast startup and small binary sizes
- Leverage existing language ecosystems

## Building CLI Tools with WASI

### Rust CLI Tool

**Cargo.toml**:
```toml
[package]
name = "wasm-grep"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.0", features = ["derive"] }
regex = "1.7"

[[bin]]
name = "wasm-grep"
path = "src/main.rs"
```

**src/main.rs**:
```rust
use clap::Parser;
use regex::Regex;
use std::fs;
use std::io::{self, BufRead};

#[derive(Parser)]
#[command(name = "wasm-grep")]
#[command(about = "A simple grep implementation in WASM")]
struct Args {
    /// Pattern to search for
    pattern: String,

    /// Files to search (stdin if not provided)
    files: Vec<String>,

    /// Case insensitive search
    #[arg(short, long)]
    ignore_case: bool,

    /// Show line numbers
    #[arg(short = 'n', long)]
    line_number: bool,
}

fn main() -> io::Result<()> {
    let args = Args::parse();

    let pattern = if args.ignore_case {
        format!("(?i){}", args.pattern)
    } else {
        args.pattern.clone()
    };

    let re = Regex::new(&pattern)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidInput, e))?;

    if args.files.is_empty() {
        // Read from stdin
        search_stdin(&re, args.line_number)?;
    } else {
        // Search files
        for file_path in &args.files {
            search_file(&re, file_path, args.line_number)?;
        }
    }

    Ok(())
}

fn search_stdin(re: &Regex, show_line_numbers: bool) -> io::Result<()> {
    let stdin = io::stdin();
    for (line_num, line) in stdin.lock().lines().enumerate() {
        let line = line?;
        if re.is_match(&line) {
            print_match(&line, line_num + 1, show_line_numbers);
        }
    }
    Ok(())
}

fn search_file(re: &Regex, path: &str, show_line_numbers: bool) -> io::Result<()> {
    let content = fs::read_to_string(path)?;

    for (line_num, line) in content.lines().enumerate() {
        if re.is_match(line) {
            if show_line_numbers {
                println!("{}:{}:{}", path, line_num + 1, line);
            } else {
                println!("{}:{}", path, line);
            }
        }
    }

    Ok(())
}

fn print_match(line: &str, line_num: usize, show_line_numbers: bool) {
    if show_line_numbers {
        println!("{}:{}", line_num, line);
    } else {
        println!("{}", line);
    }
}
```

**Build**:
```bash
# Build for WASI
cargo build --target wasm32-wasi --release

# The output is at:
# target/wasm32-wasi/release/wasm-grep.wasm
```

**Run**:
```bash
# Using wasmtime
wasmtime run target/wasm32-wasi/release/wasm-grep.wasm -- "pattern" file.txt

# With flags
wasmtime run wasm-grep.wasm -- -i -n "error" logfile.txt

# From stdin
cat file.txt | wasmtime run wasm-grep.wasm -- "pattern"

# Map directories
wasmtime run wasm-grep.wasm --dir=. -- "pattern" ./src/main.rs
```

### File Processing Tool

**JSON processor example**:

```rust
use clap::Parser;
use serde_json::{Value, from_str, to_string_pretty};
use std::fs;
use std::io::{self, Read};

#[derive(Parser)]
struct Args {
    /// Input file (stdin if not provided)
    #[arg(short, long)]
    input: Option<String>,

    /// Output file (stdout if not provided)
    #[arg(short, long)]
    output: Option<String>,

    /// JQ-like filter
    #[arg(short, long)]
    filter: Option<String>,

    /// Pretty print
    #[arg(short, long)]
    pretty: bool,
}

fn main() -> io::Result<()> {
    let args = Args::parse();

    // Read input
    let input_data = if let Some(path) = &args.input {
        fs::read_to_string(path)?
    } else {
        let mut buffer = String::new();
        io::stdin().read_to_string(&mut buffer)?;
        buffer
    };

    // Parse JSON
    let mut json: Value = from_str(&input_data)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;

    // Apply filter if provided
    if let Some(filter) = &args.filter {
        json = apply_filter(&json, filter)?;
    }

    // Serialize
    let output_data = if args.pretty {
        to_string_pretty(&json)
    } else {
        serde_json::to_string(&json)
    }.map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;

    // Write output
    if let Some(path) = &args.output {
        fs::write(path, output_data)?;
    } else {
        println!("{}", output_data);
    }

    Ok(())
}

fn apply_filter(json: &Value, filter: &str) -> io::Result<Value> {
    // Simple implementation: extract field by path
    let parts: Vec<&str> = filter.split('.').collect();

    let mut current = json;
    for part in parts {
        current = &current[part];
    }

    Ok(current.clone())
}
```

**Build and run**:
```bash
cargo build --target wasm32-wasi --release

# Pretty print JSON
echo '{"name":"Alice","age":30}' | wasmtime run json-tool.wasm -- --pretty

# Extract field
wasmtime run json-tool.wasm -- -i data.json -f "user.name"

# Process and save
wasmtime run json-tool.wasm --dir=. -- -i input.json -o output.json --pretty
```

## Go CLI Tools

### Simple CLI with TinyGo

```go
package main

import (
    "flag"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Define flags
    upper := flag.Bool("upper", false, "Convert to uppercase")
    lower := flag.Bool("lower", false, "Convert to lowercase")

    flag.Parse()

    // Read input
    args := flag.Args()
    var input string

    if len(args) > 0 {
        input = strings.Join(args, " ")
    } else {
        // Read from stdin
        data := make([]byte, 1024)
        n, _ := os.Stdin.Read(data)
        input = string(data[:n])
    }

    // Process
    var output string
    if *upper {
        output = strings.ToUpper(input)
    } else if *lower {
        output = strings.ToLower(input)
    } else {
        output = input
    }

    fmt.Println(output)
}
```

**Build**:
```bash
tinygo build -o text-tool.wasm -target=wasi main.go
```

**Run**:
```bash
wasmtime run text-tool.wasm -- --upper "hello world"
# Output: HELLO WORLD

echo "HELLO" | wasmtime run text-tool.wasm -- --lower
# Output: hello
```

### File Watcher

```go
package main

import (
    "flag"
    "fmt"
    "os"
    "time"
)

func main() {
    path := flag.String("path", ".", "Path to watch")
    flag.Parse()

    fmt.Printf("Watching: %s\n", *path)

    var lastModTime time.Time

    for {
        info, err := os.Stat(*path)
        if err != nil {
            fmt.Fprintf(os.Stderr, "Error: %v\n", err)
            os.Exit(1)
        }

        modTime := info.ModTime()

        if !lastModTime.IsZero() && modTime.After(lastModTime) {
            fmt.Printf("File changed at %v\n", modTime)
        }

        lastModTime = modTime
        time.Sleep(1 * time.Second)
    }
}
```

**Run**:
```bash
tinygo build -o watcher.wasm -target=wasi main.go

wasmtime run watcher.wasm --dir=. -- --path=./file.txt
```

## C/C++ CLI Tools

### Text Processing

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>

void process_line(char* line, int to_upper) {
    for (int i = 0; line[i]; i++) {
        line[i] = to_upper ? toupper(line[i]) : tolower(line[i]);
    }
}

int main(int argc, char* argv[]) {
    int to_upper = 0;

    // Parse arguments
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "--upper") == 0) {
            to_upper = 1;
        } else if (strcmp(argv[i], "--lower") == 0) {
            to_upper = 0;
        }
    }

    // Process stdin
    char line[1024];
    while (fgets(line, sizeof(line), stdin)) {
        process_line(line, to_upper);
        printf("%s", line);
    }

    return 0;
}
```

**Compile**:
```bash
# With wasi-sdk
/opt/wasi-sdk/bin/clang -o text.wasm text.c

# Run
echo "hello" | wasmtime run text.wasm -- --upper
```

## AssemblyScript CLI

While AssemblyScript is primarily for browser, you can create simple CLI tools:

```typescript
// cli.ts
import "wasi";
import { Console } from "as-wasi/assembly";

// Parse simple args
let args: string[] = [];
for (let i = 0; i < env.args.length; i++) {
    args.push(env.args[i]);
}

if (args.length < 2) {
    Console.error("Usage: program <message>");
    env.exit(1);
}

const message = args[1];
Console.log("You said: " + message);
```

**Compile**:
```bash
asc cli.ts -o cli.wasm --runtime stub --exportStart _start
```

## Distributing CLI Tools

### Standalone Executables

Create native executables from WASM:

**Using wasmer**:
```bash
# Create standalone executable
wasmer create-exe tool.wasm -o tool

# Now distributable as native binary
./tool --help
```

**Cross-platform executables**:
```bash
# Linux
wasmer create-exe tool.wasm -o tool-linux --target x86_64-linux

# macOS
wasmer create-exe tool.wasm -o tool-macos --target x86_64-darwin

# Windows
wasmer create-exe tool.wasm -o tool.exe --target x86_64-windows
```

### WAPM (WebAssembly Package Manager)

Publish tools to WAPM:

**wapm.toml**:
```toml
[package]
name = "username/tool-name"
version = "0.1.0"
description = "A useful CLI tool"
license = "MIT"
readme = "README.md"

[[module]]
name = "tool"
source = "target/wasm32-wasi/release/tool.wasm"
abi = "wasi"

[[command]]
name = "tool"
module = "tool"
```

**Publish**:
```bash
wapm publish
```

**Install and use**:
```bash
wapm install username/tool-name
wapm run tool -- --help
```

## Advanced CLI Patterns

### Interactive Prompts

```rust
use std::io::{self, Write};

fn prompt(message: &str) -> io::Result<String> {
    print!("{}", message);
    io::stdout().flush()?;

    let mut input = String::new();
    io::stdin().read_line(&mut input)?;

    Ok(input.trim().to_string())
}

fn main() -> io::Result<()> {
    let name = prompt("What's your name? ")?;
    let age = prompt("What's your age? ")?;

    println!("Hello, {}! You are {} years old.", name, age);

    Ok(())
}
```

### Progress Bars

```rust
use std::io::{self, Write};
use std::thread;
use std::time::Duration;

fn show_progress(current: usize, total: usize) {
    let percent = (current * 100) / total;
    let bar_width = 50;
    let filled = (current * bar_width) / total;

    print!("\r[");
    for i in 0..bar_width {
        if i < filled {
            print!("=");
        } else {
            print!(" ");
        }
    }
    print!("] {}%", percent);
    io::stdout().flush().unwrap();
}

fn main() {
    let total = 100;

    for i in 0..=total {
        show_progress(i, total);
        thread::sleep(Duration::from_millis(50));
    }

    println!("\nDone!");
}
```

### Colored Output

```rust
fn main() {
    // ANSI color codes
    const RED: &str = "\x1b[31m";
    const GREEN: &str = "\x1b[32m";
    const YELLOW: &str = "\x1b[33m";
    const BLUE: &str = "\x1b[34m";
    const RESET: &str = "\x1b[0m";

    println!("{}Error:{} Something went wrong", RED, RESET);
    println!("{}Success:{} Operation complete", GREEN, RESET);
    println!("{}Warning:{} Check configuration", YELLOW, RESET);
    println!("{}Info:{} Processing data", BLUE, RESET);
}
```

### Configuration Files

```rust
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Serialize, Deserialize, Default)]
struct Config {
    verbose: bool,
    output_dir: String,
    max_threads: usize,
}

fn load_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    if let Ok(contents) = fs::read_to_string(path) {
        Ok(toml::from_str(&contents)?)
    } else {
        Ok(Config::default())
    }
}

fn save_config(path: &str, config: &Config) -> Result<(), Box<dyn std::error::Error>> {
    let contents = toml::to_string(config)?;
    fs::write(path, contents)?;
    Ok(())
}

fn main() {
    let config = load_config("config.toml").unwrap_or_default();

    println!("Verbose: {}", config.verbose);
    println!("Output: {}", config.output_dir);
}
```

### Logging

```rust
use std::fs::OpenOptions;
use std::io::Write;
use std::time::SystemTime;

struct Logger {
    file: std::fs::File,
}

impl Logger {
    fn new(path: &str) -> io::Result<Self> {
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(path)?;

        Ok(Logger { file })
    }

    fn log(&mut self, level: &str, message: &str) -> io::Result<()> {
        let timestamp = SystemTime::now()
            .duration_since(SystemTime::UNIX_EPOCH)
            .unwrap()
            .as_secs();

        writeln!(self.file, "[{}] {}: {}", timestamp, level, message)?;
        self.file.flush()
    }

    fn info(&mut self, message: &str) -> io::Result<()> {
        self.log("INFO", message)
    }

    fn error(&mut self, message: &str) -> io::Result<()> {
        self.log("ERROR", message)
    }
}

fn main() -> io::Result<()> {
    let mut logger = Logger::new("app.log")?;

    logger.info("Application started")?;
    logger.error("Something failed")?;

    Ok(())
}
```

## Complete Example: File Converter

```rust
use clap::{Parser, ValueEnum};
use std::fs;
use std::io::{self, Read, Write};

#[derive(Clone, ValueEnum)]
enum Format {
    Json,
    Yaml,
    Toml,
}

#[derive(Parser)]
#[command(name = "converter")]
#[command(about = "Convert between config file formats")]
struct Args {
    /// Input file
    input: String,

    /// Output file
    #[arg(short, long)]
    output: Option<String>,

    /// Input format
    #[arg(short = 'f', long, value_enum)]
    from: Format,

    /// Output format
    #[arg(short = 't', long, value_enum)]
    to: Format,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();

    // Read input
    let input_data = fs::read_to_string(&args.input)?;

    // Parse based on input format
    let value: serde_json::Value = match args.from {
        Format::Json => serde_json::from_str(&input_data)?,
        Format::Yaml => serde_yaml::from_str(&input_data)?,
        Format::Toml => toml::from_str(&input_data)?,
    };

    // Serialize to output format
    let output_data = match args.to {
        Format::Json => serde_json::to_string_pretty(&value)?,
        Format::Yaml => serde_yaml::to_string(&value)?,
        Format::Toml => toml::to_string(&value)?,
    };

    // Write output
    if let Some(output_path) = args.output {
        fs::write(output_path, output_data)?;
    } else {
        println!("{}", output_data);
    }

    Ok(())
}
```

**Build and use**:
```bash
cargo build --target wasm32-wasi --release

# Convert JSON to YAML
wasmtime run converter.wasm --dir=. -- config.json -f json -t yaml -o config.yaml

# Convert TOML to JSON (output to stdout)
wasmtime run converter.wasm --dir=. -- Cargo.toml -f toml -t json
```

## Performance Optimization

### Reduce Binary Size

```bash
# Rust
cargo build --target wasm32-wasi --release
wasm-opt -Oz -o optimized.wasm target/wasm32-wasi/release/tool.wasm

# TinyGo
tinygo build -o tool.wasm -opt=z -target=wasi main.go
```

### Fast Startup

Use AOT compilation:

```bash
# Pre-compile
wasmtime compile tool.wasm -o tool.cwasm

# Run pre-compiled (faster startup)
wasmtime run tool.cwasm -- args
```

### Minimize I/O

Batch operations when possible:

```rust
// ✗ Slow: many small writes
for line in lines {
    writeln!(file, "{}", line)?;
}

// ✓ Fast: buffer writes
let mut output = String::new();
for line in lines {
    output.push_str(line);
    output.push('\n');
}
fs::write(path, output)?;
```

## Testing CLI Tools

### Integration Tests

```rust
#[cfg(test)]
mod tests {
    use std::process::Command;

    #[test]
    fn test_basic_functionality() {
        let output = Command::new("wasmtime")
            .args(&["run", "tool.wasm", "--", "--help"])
            .output()
            .expect("Failed to run tool");

        assert!(output.status.success());
        assert!(String::from_utf8_lossy(&output.stdout).contains("Usage"));
    }

    #[test]
    fn test_with_input() {
        let output = Command::new("wasmtime")
            .args(&["run", "tool.wasm", "--", "test", "input"])
            .output()
            .expect("Failed to run tool");

        assert!(output.status.success());
        // Verify output
    }
}
```

## Best Practices

1. **Always provide `--help`**: Document your CLI clearly
2. **Handle errors gracefully**: Provide meaningful error messages
3. **Use exit codes**: 0 for success, non-zero for errors
4. **Support stdin/stdout**: Enable piping
5. **Respect WASI permissions**: Request only needed directory access
6. **Optimize binary size**: Use release builds and wasm-opt
7. **Version your tools**: Include `--version` flag
8. **Test across platforms**: Verify on different OSes

## Next Steps

WebAssembly enables building portable, secure CLI tools that run anywhere with WASI support. With Rust, Go, C, and other languages, you can create powerful command-line utilities that leverage WASM's sandboxing and portability.

Next, we'll explore using WebAssembly in cloud functions and serverless environments, where WASM's fast cold starts and isolation provide significant advantages.
