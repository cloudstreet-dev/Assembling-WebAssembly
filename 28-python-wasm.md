# Chapter 28: Python and WebAssembly

## Python in the Browser

Python traditionally relies on an interpreter and C extensions, making it challenging to compile to WebAssembly. Unlike compiled languages like Rust or Go, Python brings its entire runtime, standard library, and package ecosystem. Several approaches exist:

- **Pyodide**: Full CPython 3.11+ compiled to WASM with scientific stack
- **PyScript**: Framework wrapping Pyodide for Python in HTML
- **MicroPython**: Minimal Python for embedded systems, WASM-capable
- **RustPython**: Python implementation in Rust (experimental WASM support)

We'll focus on **Pyodide**, the most complete and practical solution for scientific computing and data science in the browser.

## Why Python in WebAssembly?

### Use Cases

**Data Science Education**:
- Interactive tutorials without installation
- Live coding environments
- Jupyter-like notebooks in browser

**Scientific Visualization**:
- Real-time data analysis
- Interactive plots and charts
- Bringing Python visualization to web apps

**Legacy Code**:
- Reusing existing Python libraries
- Gradual migration to web
- Prototyping before rewriting in Rust/JS

**Domain-Specific Applications**:
- Bioinformatics analysis tools
- Financial modeling calculators
- Machine learning demos

## Pyodide: CPython in WebAssembly

Pyodide is CPython 3.11 compiled to WebAssembly with:

- **Full Python standard library**
- **NumPy, Pandas, scikit-learn, SciPy, Matplotlib**
- **Package installation via micropip**
- **Bidirectional JavaScript ↔ Python interop**
- **NumPy array sharing with JS (zero-copy)**

**Size**: ~10-30MB depending on packages loaded
**Startup**: 1-3 seconds on modern hardware
**Performance**: 3-5x slower than native Python

## Getting Started

### Basic Setup

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Pyodide Demo</title>
    <script src="https://cdn.jsdelivr.net/pyodide/v0.24.1/full/pyodide.js"></script>
</head>
<body>
    <h1>Python in the Browser</h1>
    <div id="output"></div>

    <script type="text/javascript">
        async function main() {
            // Load Pyodide
            let pyodide = await loadPyodide({
                indexURL: "https://cdn.jsdelivr.net/pyodide/v0.24.1/full/"
            });

            console.log("Pyodide loaded successfully");

            // Run Python code
            pyodide.runPython(`
                import sys
                print(f"Python {sys.version}")
                print(f"Running in WebAssembly!")

                # Simple calculation
                x = 2 + 2
                print(f"2 + 2 = {x}")
            `);

            // Get result
            let result = pyodide.runPython("2 ** 10");
            console.log("2^10 =", result);  // 1024

            document.getElementById("output").textContent =
                `Result: ${result}`;
        }

        main();
    </script>
</body>
</html>
```

### Interactive REPL

```javascript
async function createREPL() {
    const pyodide = await loadPyodide();

    const input = document.getElementById('code-input');
    const output = document.getElementById('code-output');
    const runButton = document.getElementById('run-button');

    runButton.addEventListener('click', () => {
        const code = input.value;
        try {
            // Capture stdout
            pyodide.runPython(`
                import sys
                from io import StringIO
                sys.stdout = StringIO()
            `);

            // Run user code
            const result = pyodide.runPython(code);

            // Get stdout
            const stdout = pyodide.runPython('sys.stdout.getvalue()');

            output.textContent = stdout;
            if (result !== undefined) {
                output.textContent += `\n=> ${result}`;
            }
        } catch (err) {
            output.textContent = `Error: ${err}`;
        }
    });
}

createREPL();
```

## JavaScript Interop

### Python Calling JavaScript

```python
import js

# Access DOM
document = js.document
div = document.createElement("div")
div.innerHTML = "Created by <strong>Python</strong>!"
document.body.appendChild(div)

# Call JavaScript functions
js.alert("Hello from Python!")
js.console.log("Python says hi")
js.console.warn("This is a warning")

# Use JavaScript Math
result = js.Math.sqrt(16)
print(f"sqrt(16) = {result}")  # 4.0

random_val = js.Math.random()
print(f"Random: {random_val}")

# Access window object
js.window.location.href  # Current URL
js.window.innerWidth     # Browser width

# LocalStorage
js.localStorage.setItem("myKey", "myValue")
value = js.localStorage.getItem("myKey")
print(f"Stored value: {value}")

# Fetch API
from pyodide.ffi import to_js

async def fetch_data(url):
    response = await js.fetch(url)
    data = await response.text()
    return data

# Event listeners
def on_click(event):
    js.console.log(f"Clicked at ({event.clientX}, {event.clientY})")

button = document.getElementById("myButton")
button.addEventListener("click", on_click)
```

### JavaScript Calling Python

```javascript
let pyodide = await loadPyodide();

// Define Python functions
pyodide.runPython(`
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

class Calculator:
    def __init__(self):
        self.history = []

    def add(self, a, b):
        result = a + b
        self.history.append(f"{a} + {b} = {result}")
        return result

    def multiply(self, a, b):
        result = a * b
        self.history.append(f"{a} * {b} = {result}")
        return result

    def get_history(self):
        return self.history
`);

// Call functions from JavaScript
let fib = pyodide.globals.get('fibonacci');
console.log('fib(10) =', fib(10));  // 55

let isPrime = pyodide.globals.get('is_prime');
console.log('is_prime(17) =', isPrime(17));  // true

// Use Python classes
let Calculator = pyodide.globals.get('Calculator');
let calc = Calculator();
console.log('2 + 3 =', calc.add(2, 3));
console.log('4 * 5 =', calc.multiply(4, 5));
console.log('History:', calc.get_history().toJs());

// Cleanup
fib.destroy();
isPrime.destroy();
calc.destroy();
Calculator.destroy();
```

## Installing Packages

### Using micropip

```javascript
let pyodide = await loadPyodide();

// Install packages
await pyodide.loadPackage("numpy");
await pyodide.loadPackage("pandas");
await pyodide.loadPackage("matplotlib");
await pyodide.loadPackage("scikit-learn");

pyodide.runPython(`
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

print("Packages loaded successfully!")
print(f"NumPy version: {np.__version__}")
print(f"Pandas version: {pd.__version__}")
`);
```

### Using micropip (Python-side)

```python
import micropip

# Install from PyPI
await micropip.install('requests')
await micropip.install('beautifulsoup4')

# Install specific version
await micropip.install('numpy==1.24.0')

# Install from wheel URL
await micropip.install(
    'https://example.com/my_package-1.0-py3-none-any.whl'
)

# List installed packages
packages = micropip.list()
print(packages)
```

## NumPy: Scientific Computing

### Array Operations

```python
import numpy as np

# Create arrays
arr = np.array([1, 2, 3, 4, 5])
print(f"Array: {arr}")
print(f"Mean: {arr.mean()}")
print(f"Std: {arr.std()}")
print(f"Sum: {arr.sum()}")

# 2D arrays
matrix = np.array([
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
])
print(f"Matrix:\n{matrix}")
print(f"Shape: {matrix.shape}")
print(f"Transpose:\n{matrix.T}")

# Mathematical operations
squared = arr ** 2
print(f"Squared: {squared}")

# Linear algebra
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])

dot_product = np.dot(a, b)
print(f"Dot product:\n{dot_product}")

# Random numbers
random_data = np.random.randn(1000)
print(f"Random mean: {random_data.mean():.3f}")
print(f"Random std: {random_data.std():.3f}")

# Statistics
data = np.array([23, 45, 67, 12, 89, 34, 56, 78])
print(f"Median: {np.median(data)}")
print(f"Percentiles: {np.percentile(data, [25, 50, 75])}")
```

### Zero-Copy Array Sharing

```javascript
let pyodide = await loadPyodide();
await pyodide.loadPackage("numpy");

// Create NumPy array in Python
pyodide.runPython(`
import numpy as np
py_array = np.array([1, 2, 3, 4, 5], dtype=np.float64)
`);

// Get reference to Python array (zero-copy!)
let pyArray = pyodide.globals.get('py_array');
let jsArray = pyArray.toJs();

console.log('Python array:', jsArray);

// Modify in JavaScript
jsArray[0] = 100;

// Changes visible in Python!
pyodide.runPython('print(py_array[0])');  // 100.0

// Cleanup
pyArray.destroy();
```

## Pandas: Data Analysis

### DataFrames

```python
import pandas as pd
import numpy as np

# Create DataFrame
data = {
    'name': ['Alice', 'Bob', 'Charlie', 'David', 'Eve'],
    'age': [25, 30, 35, 28, 32],
    'salary': [50000, 60000, 75000, 55000, 70000],
    'department': ['Engineering', 'Sales', 'Engineering', 'HR', 'Sales']
}

df = pd.DataFrame(data)
print(df)

# Basic statistics
print(f"\nAverage age: {df['age'].mean():.1f}")
print(f"Average salary: ${df['salary'].mean():,.0f}")
print(f"Max salary: ${df['salary'].max():,.0f}")

# Filtering
engineers = df[df['department'] == 'Engineering']
print(f"\nEngineers:\n{engineers}")

high_earners = df[df['salary'] > 60000]
print(f"\nHigh earners:\n{high_earners}")

# Group by
dept_stats = df.groupby('department').agg({
    'age': 'mean',
    'salary': ['mean', 'min', 'max']
})
print(f"\nDepartment statistics:\n{dept_stats}")

# Sorting
sorted_df = df.sort_values('salary', ascending=False)
print(f"\nSorted by salary:\n{sorted_df}")
```

### Data Processing Example

```javascript
await pyodide.loadPackage(["pandas", "numpy"]);

pyodide.runPython(`
import pandas as pd
import numpy as np
from io import StringIO

# Simulate CSV data
csv_data = """date,temperature,humidity,pressure
2024-01-01,15.2,65,1013.2
2024-01-02,16.1,62,1014.5
2024-01-03,14.8,68,1012.8
2024-01-04,17.3,60,1015.1
2024-01-05,15.9,63,1013.9"""

# Load data
df = pd.read_csv(StringIO(csv_data))
df['date'] = pd.to_datetime(df['date'])

# Analysis
print("Dataset shape:", df.shape)
print("\nSummary statistics:")
print(df.describe())

print("\nAverage temperature:", df['temperature'].mean())
print("Temperature range:", df['temperature'].max() - df['temperature'].min())

# Add computed columns
df['temp_fahrenheit'] = df['temperature'] * 9/5 + 32
df['comfort_index'] = df['temperature'] * 0.7 + df['humidity'] * 0.3

print("\nProcessed data:")
print(df)
`);
```

## Matplotlib: Visualization

### Creating Plots

```python
import matplotlib.pyplot as plt
import numpy as np

# Generate data
x = np.linspace(0, 10, 100)
y1 = np.sin(x)
y2 = np.cos(x)

# Create figure
plt.figure(figsize=(10, 6))

# Plot multiple lines
plt.plot(x, y1, label='sin(x)', linewidth=2)
plt.plot(x, y2, label='cos(x)', linewidth=2)

plt.xlabel('x')
plt.ylabel('y')
plt.title('Trigonometric Functions')
plt.legend()
plt.grid(True, alpha=0.3)

# Save to buffer and display
import io
import base64

buf = io.BytesIO()
plt.savefig(buf, format='png', dpi=100, bbox_inches='tight')
buf.seek(0)

# Encode image for display
img_base64 = base64.b64encode(buf.read()).decode('utf-8')
img_html = f'<img src="data:image/png;base64,{img_base64}">'

# Display in browser
import js
js.document.getElementById('plot-output').innerHTML = img_html
```

### Interactive Dashboard

```javascript
await pyodide.loadPackage(["numpy", "matplotlib"]);

async function createPlot(plotType) {
    const code = `
import matplotlib.pyplot as plt
import numpy as np
import io
import base64

# Generate data
np.random.seed(42)
data = np.random.randn(1000)

# Create plot
plt.figure(figsize=(8, 6))

if "${plotType}" == "histogram":
    plt.hist(data, bins=30, edgecolor='black', alpha=0.7)
    plt.title('Histogram of Random Data')
    plt.xlabel('Value')
    plt.ylabel('Frequency')
elif "${plotType}" == "scatter":
    x = np.random.randn(100)
    y = 2 * x + np.random.randn(100) * 0.5
    plt.scatter(x, y, alpha=0.6)
    plt.title('Scatter Plot')
    plt.xlabel('X')
    plt.ylabel('Y')
elif "${plotType}" == "line":
    x = np.linspace(0, 4*np.pi, 100)
    plt.plot(x, np.sin(x), label='sin(x)')
    plt.plot(x, np.cos(x), label='cos(x)')
    plt.title('Line Plot')
    plt.legend()
    plt.grid(True, alpha=0.3)

plt.tight_layout()

# Convert to base64
buf = io.BytesIO()
plt.savefig(buf, format='png', dpi=100)
buf.seek(0)
img_b64 = base64.b64encode(buf.read()).decode('utf-8')
plt.close()

img_b64
    `;

    const imgBase64 = pyodide.runPython(code);
    document.getElementById('plot-display').innerHTML =
        `<img src="data:image/png;base64,${imgBase64}">`;
}

// Add button handlers
document.getElementById('hist-btn').onclick = () => createPlot('histogram');
document.getElementById('scatter-btn').onclick = () => createPlot('scatter');
document.getElementById('line-btn').onclick = () => createPlot('line');
```

## Machine Learning Demo

### scikit-learn

```python
from sklearn import datasets
from sklearn.model_selection import train_test_split
from sklearn.tree import DecisionTreeClassifier
from sklearn.metrics import accuracy_score, classification_report

# Load dataset
iris = datasets.load_iris()
X, y = iris.data, iris.target

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)

# Train model
clf = DecisionTreeClassifier(max_depth=3, random_state=42)
clf.fit(X_train, y_train)

# Predictions
y_pred = clf.predict(X_test)

# Evaluate
accuracy = accuracy_score(y_test, y_pred)
print(f"Accuracy: {accuracy:.2%}")

print("\nClassification Report:")
print(classification_report(y_test, y_pred,
                           target_names=iris.target_names))

# Predict new samples
new_sample = [[5.1, 3.5, 1.4, 0.2]]
prediction = clf.predict(new_sample)
print(f"\nPrediction for {new_sample[0]}: {iris.target_names[prediction[0]]}")
```

### Complete ML Application

```javascript
await pyodide.loadPackage(["numpy", "scikit-learn"]);

const mlApp = `
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, confusion_matrix

class MLModel:
    def __init__(self):
        self.model = None
        self.is_trained = False

    def generate_data(self, n_samples=1000, n_features=20):
        """Generate synthetic classification data"""
        X, y = make_classification(
            n_samples=n_samples,
            n_features=n_features,
            n_informative=15,
            n_redundant=5,
            random_state=42
        )
        return X, y

    def train(self, X, y):
        """Train the model"""
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42
        )

        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.model.fit(X_train, y_train)

        # Evaluate
        train_score = self.model.score(X_train, y_train)
        test_score = self.model.score(X_test, y_test)

        self.is_trained = True

        return {
            'train_accuracy': float(train_score),
            'test_accuracy': float(test_score),
            'n_samples': len(X),
            'n_features': X.shape[1]
        }

    def predict(self, X):
        """Make predictions"""
        if not self.is_trained:
            raise ValueError("Model not trained yet")
        return self.model.predict(X).tolist()

    def feature_importance(self):
        """Get feature importances"""
        if not self.is_trained:
            raise ValueError("Model not trained yet")
        return self.model.feature_importances_.tolist()

# Create global instance
ml_model = MLModel()
`;

pyodide.runPython(mlApp);

// Train model
async function trainModel() {
    const result = pyodide.runPython(`
X, y = ml_model.generate_data(n_samples=1000, n_features=20)
ml_model.train(X, y)
    `);

    console.log('Training complete:', result.toJs());
    return result.toJs();
}

// Make predictions
function predict(features) {
    pyodide.globals.set('test_features', features);
    const predictions = pyodide.runPython(`
import numpy as np
X_test = np.array(test_features)
ml_model.predict(X_test)
    `);
    return predictions.toJs();
}

// Use the model
const trainingResults = await trainModel();
console.log('Accuracy:', trainingResults.test_accuracy);
```

## Performance Benchmarks

### Computation Speed

```javascript
await pyodide.loadPackage("numpy");

// Benchmark: Matrix multiplication
const benchmarkCode = `
import numpy as np
import time

def benchmark_matmul(size):
    A = np.random.randn(size, size)
    B = np.random.randn(size, size)

    start = time.time()
    C = np.dot(A, B)
    elapsed = (time.time() - start) * 1000  # ms

    return elapsed

# Run benchmarks
results = {}
for size in [100, 200, 500]:
    elapsed = benchmark_matmul(size)
    results[size] = elapsed
    print(f"Matrix {size}x{size}: {elapsed:.2f}ms")

results
`;

const results = pyodide.runPython(benchmarkCode).toJs();

// Compare with JavaScript (using ml-matrix library)
const jsResults = {
    100: 15,
    200: 45,
    500: 280
};

console.log('Python/NumPy:', results);
console.log('JavaScript:', jsResults);
console.log('Slowdown:', {
    100: results[100] / jsResults[100],
    200: results[200] / jsResults[200],
    500: results[500] / jsResults[500]
});
```

**Typical Results**:
```
Matrix Operations (1000x1000):
- NumPy in Pyodide: 180ms
- Native NumPy: 45ms
- JavaScript (typed arrays): 850ms
- Pyodide slowdown vs native: 4x
- Pyodide speedup vs JS: 4.7x
```

### Startup Time

```javascript
console.time('Pyodide load');
const pyodide = await loadPyodide();
console.timeEnd('Pyodide load');
// Typically: 1-3 seconds

console.time('NumPy load');
await pyodide.loadPackage('numpy');
console.timeEnd('NumPy load');
// Typically: 0.5-1.5 seconds

console.time('Pandas load');
await pyodide.loadPackage('pandas');
console.timeEnd('Pandas load');
// Typically: 1-2 seconds
```

### Memory Usage

```javascript
// Pyodide memory footprint
await pyodide.loadPackage(['numpy', 'pandas', 'matplotlib']);

const memory = performance.memory;
console.log('Heap size:', (memory.usedJSHeapSize / 1024 / 1024).toFixed(2), 'MB');
console.log('Heap limit:', (memory.jsHeapSizeLimit / 1024 / 1024).toFixed(2), 'MB');

// Typical memory usage:
// Pyodide alone: ~20MB
// + NumPy: ~30MB
// + Pandas: ~45MB
// + Matplotlib: ~55MB
```

## JupyterLite: Jupyter in the Browser

JupyterLite provides a full Jupyter environment powered by Pyodide:

### Installation and Setup

```bash
pip install jupyterlite-core
jupyter lite init
jupyter lite build
jupyter lite serve
```

### Custom Configuration

```json
// jupyter-lite.json
{
  "jupyter-lite-schema-version": 0,
  "jupyter-config-data": {
    "appName": "My Data Science Lab",
    "appVersion": "0.1.0",
    "disabledExtensions": [],
    "settingsOverrides": {
      "@jupyterlab/apputils-extension:themes": {
        "theme": "JupyterLab Dark"
      }
    }
  }
}
```

### Deploying to GitHub Pages

```yaml
# .github/workflows/deploy.yml
name: Deploy JupyterLite

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install jupyterlite-core
          pip install jupyterlab-miami-nights  # Optional theme

      - name: Build site
        run: |
          jupyter lite build --output-dir dist

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

## PyScript: Python in HTML

PyScript makes Python integration even easier:

```html
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="https://pyscript.net/latest/pyscript.css" />
    <script defer src="https://pyscript.net/latest/pyscript.js"></script>
</head>
<body>
    <py-config>
        packages = ["numpy", "matplotlib"]
    </py-config>

    <h1>PyScript Demo</h1>

    <py-script>
import numpy as np
import matplotlib.pyplot as plt

# Generate data
x = np.linspace(0, 2*np.pi, 100)
y = np.sin(x)

# Create plot
fig, ax = plt.subplots()
ax.plot(x, y)
ax.set_title('Sine Wave')
ax.set_xlabel('x')
ax.set_ylabel('sin(x)')
ax.grid(True, alpha=0.3)

display(fig, target="plot-output")
    </py-script>

    <div id="plot-output"></div>
</body>
</html>
```

## WASI Python (Experimental)

Efforts exist to run CPython via WASI for server-side applications:

### Building Python for WASI

```bash
# Clone CPython
git clone https://github.com/python/cpython
cd cpython

# Configure for WASI
./configure --host=wasm32-wasi --build=$(./config.guess) \
    --with-build-python=$(which python3)

# Build
make

# Run with wasmtime
wasmtime run --dir=. python.wasm -- -c "print('Hello from WASI Python')"
```

### Limitations

- No threading support yet
- Limited C extension support
- Experimental, not production-ready
- Performance slower than native Python

## Best Practices

### 1. Lazy Load Packages

```javascript
// ✗ Bad: Load everything upfront
await pyodide.loadPackage(['numpy', 'pandas', 'matplotlib', 'scikit-learn']);
// Takes 5-10 seconds!

// ✓ Good: Load on demand
async function runAnalysis() {
    if (!window.numpyLoaded) {
        await pyodide.loadPackage('numpy');
        window.numpyLoaded = true;
    }
    // Run analysis
}
```

### 2. Minimize Python ↔ JS Boundary Crossings

```python
# ✗ Bad: Frequent JS calls
for i in range(1000):
    js.console.log(f"Processing {i}")  # 1000 JS calls!

# ✓ Good: Batch operations
messages = [f"Processing {i}" for i in range(1000)]
js.console.log("\n".join(messages))  # 1 JS call
```

### 3. Use Web Workers for Heavy Computation

```javascript
// worker.js
importScripts('https://cdn.jsdelivr.net/pyodide/v0.24.1/full/pyodide.js');

let pyodide;

self.onmessage = async (event) => {
    if (!pyodide) {
        pyodide = await loadPyodide();
        await pyodide.loadPackage('numpy');
    }

    const { code } = event.data;
    try {
        const result = pyodide.runPython(code);
        self.postMessage({ success: true, result });
    } catch (error) {
        self.postMessage({ success: false, error: error.message });
    }
};
```

### 4. Cache Compiled Modules

```javascript
// Store compiled modules in IndexedDB
async function getCachedPyodide() {
    const cache = await caches.open('pyodide-v1');
    const cachedResponse = await cache.match('pyodide.asm.js');

    if (cachedResponse) {
        // Use cached version
        return loadPyodide({ indexURL: cache });
    }

    // Download and cache
    const pyodide = await loadPyodide();
    // Cache for next time
    return pyodide;
}
```

## Limitations and Trade-offs

### Size

- **Pyodide core**: ~6MB (gzipped)
- **NumPy**: ~3MB
- **Pandas**: ~8MB
- **Matplotlib**: ~5MB
- **scikit-learn**: ~9MB

Total for full data science stack: ~30MB

### Performance

- **3-5x slower** than native Python
- **GC pauses** can cause jank in UI
- **No native threads** (Python GIL + WASM limitations)
- **Cold start**: 1-3 seconds initial load

### Compatibility

- **Most pure Python** packages work
- **Some C extensions** work (if compiled to WASM)
- **System-level operations** limited
- **File I/O** requires special handling
- **No pip**: Use micropip instead

## When to Use Python/WASM

**✓ Good use cases**:
- Educational platforms and tutorials
- Data science demonstrations
- Interactive notebooks
- Prototyping and experimentation
- Bringing existing Python code to web without rewriting

**✗ Not suitable for**:
- Performance-critical applications (use Rust/C++)
- Real-time processing
- Mobile devices (size/performance constraints)
- Production web applications (unless specifically for data science)
- Applications requiring small bundle sizes

## Comparison: Pyodide vs Alternatives

### vs Native JavaScript

**Pyodide advantages**:
- NumPy/SciPy ecosystem
- Existing Python libraries
- Familiar syntax for Python developers

**JavaScript advantages**:
- 10x smaller
- 3-5x faster
- No load time
- Better browser integration

### vs Compiled Languages (Rust, C++)

**Pyodide advantages**:
- Rapid prototyping
- Dynamic typing
- Interactive REPL

**Compiled advantages**:
- 10-100x faster execution
- 100x smaller binaries
- Better performance guarantees

## Resources

- **Pyodide docs**: https://pyodide.org/en/stable/
- **PyScript**: https://pyscript.net/
- **JupyterLite**: https://jupyterlite.readthedocs.io/
- **Examples**: https://github.com/pyodide/pyodide/tree/main/examples
- **Discord**: Pyodide community Discord

## Conclusion

Python in WebAssembly via Pyodide is impressive but comes with significant trade-offs. It excels for:

1. **Education**: No installation required for learners
2. **Data Science demos**: Show off analyses in the browser
3. **Prototyping**: Test ideas before production rewrite
4. **Legacy code**: Gradual browser migration

For production applications requiring performance, consider:
- **Rust + wasm-bindgen** for computation
- **JavaScript + WebGL** for visualization
- **Backend API** for heavy processing

Pyodide shows what's possible—running a full Python environment in the browser—but for most production use cases, languages designed for compilation to WASM (Rust, C++, Go) will provide better performance and smaller binaries.

## Next Steps

We've now covered languages from compiled (Rust, C++, Go) to managed (Kotlin, C#) to dynamic (Python). In the next chapter, we'll explore experimental languages that are being designed specifically for WebAssembly as a first-class target, showing the future evolution of WASM-native development.
