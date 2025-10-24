# Chapter 28: Python and WebAssembly

## Python in the Browser

Python doesn't traditionally compile to WASM, but several projects enable running Python in browsers:

- **Pyodide**: CPython compiled to WASM
- **MicroPython**: Minimal Python for embedded systems
- **Brython**: Python to JavaScript transpiler (not WASM)

We'll focus on Pyodide, the most complete solution.

## Pyodide

Pyodide is CPython 3.11 compiled to WebAssembly with:
- Full Python standard library
- NumPy, Pandas, Matplotlib, scikit-learn
- Package installation via micropip
- JavaScript ↔ Python interop

**Use cases**: Data science notebooks, scientific computing, Python education

## Basic Usage

**HTML**:
```html
<!DOCTYPE html>
<html>
<head>
    <script src="https://cdn.jsdelivr.net/pyodide/v0.24.1/full/pyodide.js"></script>
</head>
<body>
    <script type="text/javascript">
        async function main() {
            let pyodide = await loadPyodide();

            // Run Python code
            pyodide.runPython(`
                print("Hello from Python!")
                x = 2 + 2
                print(f"2 + 2 = {x}")
            `);

            // Get result
            let result = pyodide.runPython("2 ** 10");
            console.log("2^10 =", result);  // 1024
        }
        main();
    </script>
</body>
</html>
```

## JavaScript Interop

### Python Calling JavaScript

```python
import js

# Access DOM
document = js.document
div = document.createElement("div")
div.innerHTML = "Created by Python!"
document.body.appendChild(div)

# Call JavaScript functions
js.alert("Hello from Python!")
js.console.log("Python says hi")

# Math operations
result = js.Math.sqrt(16)
print(f"sqrt(16) = {result}")
```

### JavaScript Calling Python

```javascript
let pyodide = await loadPyodide();

// Define Python function
pyodide.runPython(`
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
`);

// Call from JavaScript
let fib = pyodide.globals.get('fibonacci');
console.log(fib(10));  // 55
```

## Installing Packages

```javascript
let pyodide = await loadPyodide();

// Install packages
await pyodide.loadPackage("numpy");
await pyodide.loadPackage("matplotlib");

pyodide.runPython(`
import numpy as np

arr = np.array([1, 2, 3, 4, 5])
print(f"Mean: {arr.mean()}")
print(f"Std: {arr.std()}")
`);
```

## Data Analysis Example

```javascript
let pyodide = await loadPyodide();
await pyodide.loadPackage(["numpy", "pandas"]);

pyodide.runPython(`
import pandas as pd
import numpy as np

# Create dataset
data = {
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [25, 30, 35],
    'score': [85, 90, 95]
}

df = pd.DataFrame(data)
print(df)
print(f"\\nAverage age: {df['age'].mean()}")
print(f"Average score: {df['score'].mean()}")
`);
```

## Limitations

1. **Size**: Pyodide is ~10-30MB depending on packages
2. **Startup**: Initial load is slow
3. **Performance**: Slower than native Python or compiled languages
4. **No native extensions**: Can't use C extensions that weren't pre-compiled
5. **No multithreading**: Python's GIL + WASM limitations

## Use Cases

**Good for**:
- Data science demonstrations
- Interactive tutorials
- Jupyter-like notebooks in browser
- Scientific visualization

**Not suitable for**:
- Performance-critical applications
- Real-time processing
- Mobile devices (size constraints)

## JupyterLite

JupyterLite is Jupyter running entirely in the browser via Pyodide:

```bash
pip install jupyterlite
jupyter lite build
jupyter lite serve
```

Creates a full Jupyter environment with no server!

## MicroPython

For smaller footprint, MicroPython compiles to WASM:

- Much smaller (~170KB)
- Subset of Python
- No standard library packages
- Faster startup

**Use when**: Size matters more than compatibility

## Future: Python in WASI

Efforts exist to compile CPython for WASI, enabling Python CLI tools as WASM. Still experimental.

Python's WASM story is about bringing existing Python code to browsers, not compiling Python to efficient WASM. For performance, use Rust, C++, or other compiled languages. For Python ecosystem access, Pyodide is impressive but comes with size/speed tradeoffs.
