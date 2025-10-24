# Chapter 40: WebAssembly in Embedded Systems

## Why WASM for Embedded?

WebAssembly offers unique advantages for embedded systems:

- **Small footprint**: Minimal memory requirements (KB range)
- **Sandboxed execution**: Safe execution of untrusted code
- **Deterministic**: Predictable performance
- **Portable**: Same code across different MCUs
- **Fast**: Near-native performance
- **Language flexibility**: Use high-level languages safely

## WASM Runtimes for Embedded

### WAMR (WebAssembly Micro Runtime)

**Status**: Production-ready, designed for embedded
**Footprint**: ~100KB for interpreter, ~500KB with JIT
**Platforms**: ARM, RISC-V, x86, Xtensa (ESP32)

**Features**:
- Interpreter mode (smallest)
- AOT compilation (fastest)
- JIT compilation (balance)
- Minimal memory overhead

**Basic Integration** (C):

```c
#include "wasm_export.h"
#include <stdio.h>

int main() {
    // Initialize runtime
    RuntimeInitArgs init_args;
    memset(&init_args, 0, sizeof(init_args));

    init_args.mem_alloc_type = Alloc_With_Pool;
    init_args.mem_alloc_option.pool.heap_buf = malloc(8 * 1024 * 1024);
    init_args.mem_alloc_option.pool.heap_size = 8 * 1024 * 1024;

    wasm_runtime_full_init(&init_args);

    // Load WASM bytecode
    FILE* file = fopen("app.wasm", "rb");
    fseek(file, 0, SEEK_END);
    size_t file_size = ftell(file);
    fseek(file, 0, SEEK_SET);

    uint8_t* buffer = malloc(file_size);
    fread(buffer, 1, file_size, file);
    fclose(file);

    // Load module
    char error_buf[128];
    wasm_module_t module = wasm_runtime_load(
        buffer, file_size,
        error_buf, sizeof(error_buf)
    );

    if (!module) {
        printf("Load failed: %s\n", error_buf);
        return 1;
    }

    // Instantiate module
    wasm_module_inst_t module_inst = wasm_runtime_instantiate(
        module,
        16 * 1024,  // Stack size: 16KB
        16 * 1024,  // Heap size: 16KB
        error_buf, sizeof(error_buf)
    );

    // Find and call function
    wasm_function_inst_t func = wasm_runtime_lookup_function(
        module_inst, "process_sensor", NULL
    );

    if (func) {
        uint32_t argv[2] = { 42, 100 };  // Arguments
        if (!wasm_runtime_call_wasm(module_inst, NULL, func, 2, argv)) {
            printf("Execution failed\n");
        } else {
            printf("Result: %u\n", argv[0]);  // Result in first arg
        }
    }

    // Cleanup
    wasm_runtime_deinstantiate(module_inst);
    wasm_runtime_unload(module);
    wasm_runtime_destroy();
    free(buffer);

    return 0;
}
```

**Providing host functions**:

```c
// GPIO control from WASM
static int32_t gpio_write(wasm_exec_env_t exec_env, int32_t pin, int32_t value) {
    // Platform-specific GPIO write
    // For example, on embedded Linux:
    // gpio_set_value(pin, value);

    printf("GPIO: Set pin %d to %d\n", pin, value);
    return 0;  // Success
}

static int32_t read_sensor(wasm_exec_env_t exec_env) {
    // Read actual sensor
    // For example, ADC reading:
    // return adc_read_channel(0);

    return 42;  // Mock sensor value
}

// Register native functions
static NativeSymbol native_symbols[] = {
    {"gpio_write", gpio_write, "(ii)i", NULL},
    {"read_sensor", read_sensor, "()i", NULL}
};

int register_natives() {
    return wasm_runtime_register_natives(
        "env",
        native_symbols,
        sizeof(native_symbols) / sizeof(NativeSymbol)
    );
}
```

### wasm3

**Status**: Production-ready
**Footprint**: ~64KB interpreter
**Speed**: Slower than WAMR, but very compact
**Platforms**: Everything (even 8-bit MCUs!)

**Arduino Example**:

```cpp
#include <Arduino.h>
#include "wasm3.h"
#include "m3_env.h"

// Include compiled WASM as byte array
#include "app_wasm.h"

#define WASM_STACK_SIZE 2048
#define NATIVE_STACK_SIZE 2048

IM3Environment env;
IM3Runtime runtime;
IM3Module module;

// Host function: digitalWrite
m3ApiRawFunction(m3_arduino_digitalWrite) {
    m3ApiGetArg(int32_t, pin);
    m3ApiGetArg(int32_t, value);

    digitalWrite(pin, value);
    m3ApiSuccess();
}

// Host function: analogRead
m3ApiRawFunction(m3_arduino_analogRead) {
    m3ApiGetArg(int32_t, pin);

    int32_t value = analogRead(pin);

    m3ApiReturn(value);
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    Serial.println("Initializing WASM...");

    // Create environment
    env = m3_NewEnvironment();

    // Create runtime
    runtime = m3_NewRuntime(env, WASM_STACK_SIZE, NULL);

    // Parse module
    M3Result result = m3_ParseModule(env, &module, app_wasm, app_wasm_len);
    if (result) {
        Serial.printf("Parse failed: %s\n", result);
        return;
    }

    // Load module
    result = m3_LoadModule(runtime, module);
    if (result) {
        Serial.printf("Load failed: %s\n", result);
        return;
    }

    // Link Arduino functions
    m3_LinkRawFunction(module, "env", "digitalWrite", "v(ii)", &m3_arduino_digitalWrite);
    m3_LinkRawFunction(module, "env", "analogRead", "i(i)", &m3_arduino_analogRead);

    Serial.println("WASM initialized");
}

void loop() {
    // Find and call WASM function
    IM3Function func;
    M3Result result = m3_FindFunction(&func, runtime, "loop");

    if (result) {
        Serial.printf("Function not found: %s\n", result);
        delay(1000);
        return;
    }

    // Call function
    result = m3_CallV(func);
    if (result) {
        Serial.printf("Execution failed: %s\n", result);
    }

    delay(100);
}
```

**WASM app** (Rust):

```rust
#[no_mangle]
pub extern "C" fn digitalWrite(pin: i32, value: i32);

#[no_mangle]
pub extern "C" fn analogRead(pin: i32) -> i32;

const LED_PIN: i32 = 13;
const SENSOR_PIN: i32 = A0;

#[no_mangle]
pub extern "C" fn loop() {
    unsafe {
        let sensor_value = analogRead(SENSOR_PIN);

        // Blink LED based on sensor
        if sensor_value > 512 {
            digitalWrite(LED_PIN, 1);
        } else {
            digitalWrite(LED_PIN, 0);
        }
    }
}
```

**Build for embedded**:

```bash
rustc --target wasm32-unknown-unknown --crate-type cdylib -O app.rs -o app.wasm

# Convert to C header
xxd -i app.wasm > app_wasm.h
```

## ESP32 Integration

### WAMR on ESP32

**platformio.ini**:
```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = espidf

lib_deps =
    https://github.com/bytecodealliance/wasm-micro-runtime.git

build_flags =
    -DWASM_ENABLE_INTERP=1
    -DWASM_ENABLE_FAST_INTERP=1
    -DWASM_ENABLE_LIBC_WASI=0
```

**main.c**:
```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "wasm_export.h"

#define LED_GPIO 2

// Host function for LED control
static int32_t led_set(wasm_exec_env_t exec_env, int32_t state) {
    gpio_set_level(LED_GPIO, state);
    return 0;
}

static NativeSymbol native_symbols[] = {
    {"led_set", led_set, "(i)i", NULL}
};

void wasm_task(void *arg) {
    // Initialize GPIO
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    // Initialize WASM runtime
    RuntimeInitArgs init_args;
    memset(&init_args, 0, sizeof(init_args));

    static uint8_t heap[64 * 1024];  // 64KB heap
    init_args.mem_alloc_type = Alloc_With_Pool;
    init_args.mem_alloc_option.pool.heap_buf = heap;
    init_args.mem_alloc_option.pool.heap_size = sizeof(heap);

    wasm_runtime_full_init(&init_args);

    // Register native functions
    wasm_runtime_register_natives("env", native_symbols,
        sizeof(native_symbols) / sizeof(NativeSymbol));

    // Load WASM module (from flash)
    extern const uint8_t app_wasm_start[] asm("_binary_app_wasm_start");
    extern const uint8_t app_wasm_end[] asm("_binary_app_wasm_end");
    size_t app_wasm_size = app_wasm_end - app_wasm_start;

    char error_buf[128];
    wasm_module_t module = wasm_runtime_load(
        (uint8_t*)app_wasm_start, app_wasm_size,
        error_buf, sizeof(error_buf)
    );

    wasm_module_inst_t module_inst = wasm_runtime_instantiate(
        module, 8 * 1024, 8 * 1024, error_buf, sizeof(error_buf)
    );

    // Find function
    wasm_function_inst_t blink_func = wasm_runtime_lookup_function(
        module_inst, "blink", NULL
    );

    // Call repeatedly
    while (1) {
        wasm_runtime_call_wasm(module_inst, NULL, blink_func, 0, NULL);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}

void app_main(void) {
    xTaskCreate(wasm_task, "wasm_task", 8192, NULL, 5, NULL);
}
```

**WASM app**:

```rust
extern "C" {
    fn led_set(state: i32) -> i32;
}

static mut BLINK_STATE: bool = false;

#[no_mangle]
pub extern "C" fn blink() {
    unsafe {
        BLINK_STATE = !BLINK_STATE;
        led_set(if BLINK_STATE { 1 } else { 0 });
    }
}
```

## Real-Time Constraints

### Deterministic Execution

WASM is deterministic, but runtime overhead matters:

```c
// Measure execution time
uint32_t start = micros();

wasm_runtime_call_wasm(module_inst, NULL, func, argc, argv);

uint32_t duration = micros() - start;

if (duration > MAX_ALLOWED_US) {
    printf("WARNING: Execution took %u us\n", duration);
}
```

### AOT Compilation

Pre-compile for guaranteed performance:

```bash
# Compile WASM to native code
wamrc --target=thumbv7m -o app.aot app.wasm

# Or for specific CPU
wamrc --target=armv7 --cpu=cortex-m4 -o app.aot app.wasm
```

**Load AOT module**:

```c
// Load AOT instead of WASM
wasm_module_t module = wasm_runtime_load_from_sections(
    section_list,  // Pre-compiled sections
    false,  // Not bytecode
    error_buf, sizeof(error_buf)
);
```

## Sensor Processing Example

**Temperature monitoring system**:

```rust
#![no_std]

extern "C" {
    fn read_temperature_sensor() -> i32;  // Returns temp * 100
    fn set_heater(state: i32);
    fn set_cooler(state: i32);
    fn log_message(msg_ptr: *const u8, msg_len: usize);
}

const TARGET_TEMP: i32 = 2200;  // 22.00°C
const HYSTERESIS: i32 = 50;      // 0.50°C

static mut LAST_STATE: State = State::Idle;

#[derive(Clone, Copy)]
enum State {
    Idle,
    Heating,
    Cooling,
}

#[no_mangle]
pub extern "C" fn control_loop() {
    unsafe {
        let current_temp = read_temperature_sensor();

        let new_state = match LAST_STATE {
            State::Idle => {
                if current_temp < TARGET_TEMP - HYSTERESIS {
                    State::Heating
                } else if current_temp > TARGET_TEMP + HYSTERESIS {
                    State::Cooling
                } else {
                    State::Idle
                }
            }
            State::Heating => {
                if current_temp >= TARGET_TEMP {
                    State::Idle
                } else {
                    State::Heating
                }
            }
            State::Cooling => {
                if current_temp <= TARGET_TEMP {
                    State::Idle
                } else {
                    State::Cooling
                }
            }
        };

        // Update outputs if state changed
        if !matches!((LAST_STATE, new_state), (State::Heating, State::Heating)) {
            set_heater(if matches!(new_state, State::Heating) { 1 } else { 0 });
        }

        if !matches!((LAST_STATE, new_state), (State::Cooling, State::Cooling)) {
            set_cooler(if matches!(new_state, State::Cooling) { 1 } else { 0 });
        }

        LAST_STATE = new_state;
    }
}

#[no_mangle]
pub extern "C" fn get_status() -> i32 {
    unsafe {
        match LAST_STATE {
            State::Idle => 0,
            State::Heating => 1,
            State::Cooling => 2,
        }
    }
}
```

## Memory Management

### Fixed Memory Pools

```c
// Static memory allocation for embedded
static uint8_t wasm_heap[32 * 1024];   // 32KB for WASM
static uint8_t wasm_stack[4 * 1024];    // 4KB stack

RuntimeInitArgs init_args;
memset(&init_args, 0, sizeof(init_args));

init_args.mem_alloc_type = Alloc_With_Pool;
init_args.mem_alloc_option.pool.heap_buf = wasm_heap;
init_args.mem_alloc_option.pool.heap_size = sizeof(wasm_heap);

wasm_runtime_full_init(&init_args);

// Instantiate with fixed sizes
wasm_module_inst_t module_inst = wasm_runtime_instantiate(
    module,
    sizeof(wasm_stack),  // Stack
    0,                    // No additional heap
    error_buf, sizeof(error_buf)
);
```

### Monitoring Memory Usage

```c
void print_memory_usage(wasm_module_inst_t module_inst) {
    uint32_t total, used, free;

    bool success = wasm_runtime_get_mem_alloc_info(
        module_inst, &total, &used, &free
    );

    if (success) {
        printf("Memory - Total: %u, Used: %u, Free: %u\n",
               total, used, free);
    }
}
```

## Multi-Module Systems

**Plugin architecture**:

```c
typedef struct {
    char name[32];
    wasm_module_t module;
    wasm_module_inst_t instance;
} Plugin;

#define MAX_PLUGINS 8
static Plugin plugins[MAX_PLUGINS];
static int plugin_count = 0;

int load_plugin(const char* name, const uint8_t* wasm_data, size_t size) {
    if (plugin_count >= MAX_PLUGINS) {
        return -1;
    }

    Plugin* p = &plugins[plugin_count];
    strncpy(p->name, name, sizeof(p->name) - 1);

    char error_buf[128];
    p->module = wasm_runtime_load(wasm_data, size, error_buf, sizeof(error_buf));

    if (!p->module) {
        printf("Failed to load plugin %s: %s\n", name, error_buf);
        return -1;
    }

    p->instance = wasm_runtime_instantiate(
        p->module, 4096, 4096, error_buf, sizeof(error_buf)
    );

    if (!p->instance) {
        wasm_runtime_unload(p->module);
        return -1;
    }

    plugin_count++;
    return plugin_count - 1;
}

void call_plugin(int plugin_id, const char* func_name) {
    if (plugin_id < 0 || plugin_id >= plugin_count) {
        return;
    }

    Plugin* p = &plugins[plugin_id];

    wasm_function_inst_t func = wasm_runtime_lookup_function(
        p->instance, func_name, NULL
    );

    if (func) {
        wasm_runtime_call_wasm(p->instance, NULL, func, 0, NULL);
    }
}
```

## Security Considerations

### Resource Limits

```c
// Limit execution time with fuel metering
wasm_runtime_set_max_exec_time(module_inst, 100);  // 100ms max

// Limit memory growth
// Set in module instantiation (fixed heap size)

// Limit stack depth
// Set in runtime initialization (stack size)
```

### Capability Control

```c
// Only expose safe functions
static NativeSymbol safe_functions[] = {
    {"read_sensor", read_sensor_impl, "()i", NULL},
    {"log", log_impl, "(ii)v", NULL},
    // Don't expose: flash_write, network_send, etc.
};

wasm_runtime_register_natives("env", safe_functions,
    sizeof(safe_functions) / sizeof(NativeSymbol));
```

## Over-the-Air Updates

**Update WASM modules safely**:

```c
int update_application(const uint8_t* new_wasm, size_t size) {
    // Validate new module first
    char error_buf[128];
    wasm_module_t test_module = wasm_runtime_load(
        new_wasm, size, error_buf, sizeof(error_buf)
    );

    if (!test_module) {
        printf("Invalid module: %s\n", error_buf);
        return -1;
    }

    // Test instantiation
    wasm_module_inst_t test_inst = wasm_runtime_instantiate(
        test_module, 4096, 4096, error_buf, sizeof(error_buf)
    );

    if (!test_inst) {
        wasm_runtime_unload(test_module);
        return -1;
    }

    // Verify required functions exist
    wasm_function_inst_t main_func = wasm_runtime_lookup_function(
        test_inst, "main", NULL
    );

    if (!main_func) {
        wasm_runtime_deinstantiate(test_inst);
        wasm_runtime_unload(test_module);
        return -1;
    }

    // Module is valid, write to flash
    wasm_runtime_deinstantiate(test_inst);
    wasm_runtime_unload(test_module);

    // Write to persistent storage
    // flash_write(WASM_APP_ADDRESS, new_wasm, size);

    printf("Application updated successfully\n");
    return 0;
}
```

## Performance Optimization

### Minimize Boundary Crossings

```rust
// ✗ Slow: Many host calls
for i in 0..100 {
    unsafe { write_pixel(i, calculate_value(i)); }
}

// ✓ Fast: Batch processing
let mut buffer = [0u8; 100];
for i in 0..100 {
    buffer[i] = calculate_value(i);
}
unsafe { write_pixels(buffer.as_ptr(), buffer.len()); }
```

### Use SIMD Where Available

```rust
#[cfg(target_feature = "simd128")]
use core::arch::wasm32::*;

#[no_mangle]
pub extern "C" fn process_samples(samples: *mut i16, count: usize) {
    #[cfg(target_feature = "simd128")]
    {
        // SIMD processing
        let count_simd = count / 8;
        for i in 0..count_simd {
            unsafe {
                let ptr = samples.add(i * 8) as *mut v128;
                let data = v128_load(ptr);
                let scaled = i16x8_mul(data, i16x8_splat(2));
                v128_store(ptr, scaled);
            }
        }
    }

    #[cfg(not(target_feature = "simd128"))]
    {
        // Scalar fallback
        unsafe {
            for i in 0..count {
                *samples.add(i) *= 2;
            }
        }
    }
}
```

## Best Practices

1. **Start with interpreter**: Test functionality before AOT
2. **Measure everything**: Profile execution time and memory
3. **Use static allocation**: Avoid runtime heap growth
4. **Minimize host calls**: Batch operations
5. **Test thoroughly**: Embedded bugs are hard to fix
6. **Plan for updates**: OTA update mechanisms
7. **Limit resource usage**: Set strict limits
8. **Monitor in production**: Track performance metrics

## Next Steps

WebAssembly enables safe, portable code execution even in resource-constrained embedded systems. With runtimes like WAMR and wasm3, you can run WASM on microcontrollers, enabling plugin systems, over-the-air updates, and secure multi-tenant execution.

Next, we'll explore WebAssembly in edge computing environments, where WASM's fast cold starts and small size enable scalable, distributed computation at the network edge.
