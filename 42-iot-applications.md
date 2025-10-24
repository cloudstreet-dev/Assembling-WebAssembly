# Chapter 42: IoT Applications with WebAssembly

## WASM for IoT

WebAssembly addresses key IoT challenges:

- **Security**: Sandboxed execution prevents device compromise
- **Portability**: Same code across different IoT hardware
- **Updates**: Safe over-the-air firmware updates
- **Extensibility**: Plugin systems for device capabilities
- **Multi-tenancy**: Run untrusted code from users/third-parties
- **Resource efficiency**: Small footprint for constrained devices

## Smart Home Devices

### Temperature Controller

**WASM module** (Rust):

```rust
#![no_std]

extern "C" {
    fn read_temperature() -> f32;
    fn read_humidity() -> f32;
    fn set_hvac_mode(mode: u32);  // 0=off, 1=heat, 2=cool, 3=fan
    fn set_fan_speed(speed: u32);  // 0-100
    fn log(msg_ptr: *const u8, msg_len: usize);
}

#[repr(C)]
pub struct ThermostatConfig {
    target_temp: f32,
    hysteresis: f32,
    mode: u32,  // 0=auto, 1=heat-only, 2=cool-only
}

static mut CONFIG: ThermostatConfig = ThermostatConfig {
    target_temp: 22.0,
    hysteresis: 0.5,
    mode: 0,
};

#[no_mangle]
pub extern "C" fn init(config_ptr: *const ThermostatConfig) {
    unsafe {
        CONFIG = *config_ptr;
    }
}

#[no_mangle]
pub extern "C" fn control_loop() {
    unsafe {
        let current_temp = read_temperature();
        let current_humidity = read_humidity();

        let target = CONFIG.target_temp;
        let hysteresis = CONFIG.hysteresis;

        // Determine HVAC action
        if current_temp < target - hysteresis {
            // Too cold
            if CONFIG.mode != 2 {  // Not cooling-only
                set_hvac_mode(1);  // Heat
                set_fan_speed(calculate_fan_speed(target - current_temp));
            }
        } else if current_temp > target + hysteresis {
            // Too hot
            if CONFIG.mode != 1 {  // Not heating-only
                set_hvac_mode(2);  // Cool
                set_fan_speed(calculate_fan_speed(current_temp - target));
            }
        } else {
            // In target range
            set_hvac_mode(0);  // Off
        }

        // Handle high humidity
        if current_humidity > 60.0 {
            set_fan_speed(50);  // Run fan to reduce humidity
        }
    }
}

fn calculate_fan_speed(temp_diff: f32) -> u32 {
    // Map temperature difference to fan speed
    let speed = (temp_diff * 20.0).min(100.0).max(30.0);
    speed as u32
}

#[no_mangle]
pub extern "C" fn set_target_temperature(temp: f32) {
    unsafe {
        CONFIG.target_temp = temp;
    }
}

#[no_mangle]
pub extern "C" fn get_status(output_ptr: *mut u8, output_len: usize) -> usize {
    unsafe {
        let current_temp = read_temperature();
        let status = format!(
            "{{\"temp\":{:.1},\"target\":{:.1}}}",
            current_temp, CONFIG.target_temp
        );

        let bytes = status.as_bytes();
        let copy_len = bytes.len().min(output_len);

        core::ptr::copy_nonoverlapping(
            bytes.as_ptr(),
            output_ptr,
            copy_len
        );

        copy_len
    }
}
```

**Device firmware** (C):

```c
#include "wamr_runtime.h"
#include "sensors.h"
#include "hvac.h"
#include <stdio.h>

static wasm_module_inst_t module_inst = NULL;

// Host functions
static float host_read_temperature(wasm_exec_env_t exec_env) {
    return temperature_sensor_read();
}

static float host_read_humidity(wasm_exec_env_t exec_env) {
    return humidity_sensor_read();
}

static void host_set_hvac_mode(wasm_exec_env_t exec_env, uint32_t mode) {
    hvac_set_mode(mode);
}

static void host_set_fan_speed(wasm_exec_env_t exec_env, uint32_t speed) {
    fan_set_speed(speed);
}

static void host_log(wasm_exec_env_t exec_env, const char* msg, uint32_t len) {
    printf("[WASM] %.*s\n", len, msg);
}

static NativeSymbol native_symbols[] = {
    {"read_temperature", host_read_temperature, "()F", NULL},
    {"read_humidity", host_read_humidity, "()F", NULL},
    {"set_hvac_mode", host_set_hvac_mode, "(i)v", NULL},
    {"set_fan_speed", host_set_fan_speed, "(i)v", NULL},
    {"log", host_log, "(*~)v", NULL},
};

void thermostat_init() {
    // Initialize WASM runtime
    RuntimeInitArgs init_args;
    memset(&init_args, 0, sizeof(init_args));

    static uint8_t heap[32 * 1024];
    init_args.mem_alloc_type = Alloc_With_Pool;
    init_args.mem_alloc_option.pool.heap_buf = heap;
    init_args.mem_alloc_option.pool.heap_size = sizeof(heap);

    wasm_runtime_full_init(&init_args);

    // Register host functions
    wasm_runtime_register_natives(
        "env", native_symbols,
        sizeof(native_symbols) / sizeof(NativeSymbol)
    );

    // Load thermostat WASM module from flash
    extern const uint8_t thermostat_wasm[];
    extern const uint32_t thermostat_wasm_size;

    char error_buf[128];
    wasm_module_t module = wasm_runtime_load(
        thermostat_wasm, thermostat_wasm_size,
        error_buf, sizeof(error_buf)
    );

    module_inst = wasm_runtime_instantiate(
        module, 4096, 4096, error_buf, sizeof(error_buf)
    );

    // Initialize with config
    typedef struct {
        float target_temp;
        float hysteresis;
        uint32_t mode;
    } ThermostatConfig;

    ThermostatConfig config = {
        .target_temp = 22.0,
        .hysteresis = 0.5,
        .mode = 0
    };

    wasm_function_inst_t init_func = wasm_runtime_lookup_function(
        module_inst, "init", NULL
    );

    uint32_t argv[1] = { (uint32_t)&config };
    wasm_runtime_call_wasm(module_inst, NULL, init_func, 1, argv);
}

void thermostat_update() {
    // Call control loop every cycle
    wasm_function_inst_t loop_func = wasm_runtime_lookup_function(
        module_inst, "control_loop", NULL
    );

    wasm_runtime_call_wasm(module_inst, NULL, loop_func, 0, NULL);
}

void thermostat_set_target(float temp) {
    wasm_function_inst_t set_func = wasm_runtime_lookup_function(
        module_inst, "set_target_temperature", NULL
    );

    union {
        float f;
        uint32_t i;
    } temp_bits;
    temp_bits.f = temp;

    uint32_t argv[1] = { temp_bits.i };
    wasm_runtime_call_wasm(module_inst, NULL, set_func, 1, argv);
}
```

### Security Camera System

**Motion detection WASM**:

```rust
#![no_std]

extern "C" {
    fn read_camera_frame(buffer_ptr: *mut u8, width: u32, height: u32) -> u32;
    fn trigger_alert(confidence: u32);
}

const WIDTH: usize = 320;
const HEIGHT: usize = 240;

static mut PREV_FRAME: [u8; WIDTH * HEIGHT] = [0; WIDTH * HEIGHT];
static mut CURRENT_FRAME: [u8; WIDTH * HEIGHT] = [0; WIDTH * HEIGHT];

#[no_mangle]
pub extern "C" fn detect_motion() -> u32 {
    unsafe {
        // Read new frame (grayscale)
        read_camera_frame(
            CURRENT_FRAME.as_mut_ptr(),
            WIDTH as u32,
            HEIGHT as u32
        );

        // Compare with previous frame
        let mut diff_pixels = 0;
        let threshold = 30;  // Pixel difference threshold

        for i in 0..(WIDTH * HEIGHT) {
            let diff = (CURRENT_FRAME[i] as i32 - PREV_FRAME[i] as i32).abs();

            if diff > threshold {
                diff_pixels += 1;
            }
        }

        // Calculate percentage of changed pixels
        let total_pixels = (WIDTH * HEIGHT) as u32;
        let change_percent = (diff_pixels * 100) / total_pixels;

        // Trigger alert if significant motion
        if change_percent > 5 {  // 5% of pixels changed
            trigger_alert(change_percent);
        }

        // Store current frame as previous
        core::ptr::copy_nonoverlapping(
            CURRENT_FRAME.as_ptr(),
            PREV_FRAME.as_mut_ptr(),
            WIDTH * HEIGHT
        );

        change_percent
    }
}

#[no_mangle]
pub extern "C" fn analyze_region(x: u32, y: u32, w: u32, h: u32) -> u32 {
    // Analyze specific region for motion
    unsafe {
        let mut diff_pixels = 0;

        for row in y..(y + h) {
            for col in x..(x + w) {
                let idx = (row as usize * WIDTH) + col as usize;

                if idx < WIDTH * HEIGHT {
                    let diff = (CURRENT_FRAME[idx] as i32 - PREV_FRAME[idx] as i32).abs();

                    if diff > 30 {
                        diff_pixels += 1;
                    }
                }
            }
        }

        let total = w * h;
        (diff_pixels * 100) / total
    }
}
```

## Industrial IoT

### Sensor Network Node

```rust
#![no_std]

extern "C" {
    fn read_sensor(sensor_id: u32) -> f32;
    fn send_telemetry(data_ptr: *const u8, data_len: usize);
    fn get_timestamp() -> u64;
}

#[repr(C)]
struct SensorReading {
    timestamp: u64,
    sensor_id: u32,
    value: f32,
    status: u8,  // 0=ok, 1=warn, 2=error
}

#[no_mangle]
pub extern "C" fn collect_and_send() {
    unsafe {
        const NUM_SENSORS: u32 = 4;
        let mut readings = [SensorReading {
            timestamp: 0,
            sensor_id: 0,
            value: 0.0,
            status: 0,
        }; 4];

        let timestamp = get_timestamp();

        for i in 0..NUM_SENSORS {
            let value = read_sensor(i);

            // Determine status based on thresholds
            let status = match i {
                0 => {  // Temperature sensor
                    if value < -10.0 || value > 60.0 {
                        2  // Error
                    } else if value < 0.0 || value > 50.0 {
                        1  // Warning
                    } else {
                        0  // OK
                    }
                }
                1 => {  // Pressure sensor
                    if value < 0.8 || value > 1.2 {
                        2
                    } else if value < 0.9 || value > 1.1 {
                        1
                    } else {
                        0
                    }
                }
                _ => 0,
            };

            readings[i as usize] = SensorReading {
                timestamp,
                sensor_id: i,
                value,
                status,
            };
        }

        // Send telemetry
        let data_ptr = readings.as_ptr() as *const u8;
        let data_len = core::mem::size_of::<SensorReading>() * NUM_SENSORS as usize;

        send_telemetry(data_ptr, data_len);
    }
}
```

### Predictive Maintenance

```rust
#![no_std]

extern "C" {
    fn read_vibration_sensor() -> f32;
    fn read_temperature() -> f32;
    fn schedule_maintenance(urgency: u32);
}

const HISTORY_SIZE: usize = 100;

static mut VIBRATION_HISTORY: [f32; HISTORY_SIZE] = [0.0; HISTORY_SIZE];
static mut TEMP_HISTORY: [f32; HISTORY_SIZE] = [0.0; HISTORY_SIZE];
static mut HISTORY_INDEX: usize = 0;

#[no_mangle]
pub extern "C" fn analyze_equipment_health() -> u32 {
    unsafe {
        // Read current values
        let vibration = read_vibration_sensor();
        let temperature = read_temperature();

        // Store in history
        VIBRATION_HISTORY[HISTORY_INDEX] = vibration;
        TEMP_HISTORY[HISTORY_INDEX] = temperature;
        HISTORY_INDEX = (HISTORY_INDEX + 1) % HISTORY_SIZE;

        // Calculate statistics
        let avg_vibration = calculate_average(&VIBRATION_HISTORY);
        let avg_temp = calculate_average(&TEMP_HISTORY);

        let vibration_trend = calculate_trend(&VIBRATION_HISTORY);
        let temp_trend = calculate_trend(&TEMP_HISTORY);

        // Health score (0-100)
        let mut health_score = 100;

        // Penalize high vibration
        if avg_vibration > 5.0 {
            health_score -= 20;
        }

        // Penalize high temperature
        if avg_temp > 80.0 {
            health_score -= 20;
        }

        // Penalize increasing trends
        if vibration_trend > 0.1 {
            health_score -= 15;
        }

        if temp_trend > 0.5 {
            health_score -= 15;
        }

        // Schedule maintenance if needed
        if health_score < 60 {
            let urgency = if health_score < 40 {
                2  // Urgent
            } else {
                1  // Soon
            };

            schedule_maintenance(urgency);
        }

        health_score as u32
    }
}

fn calculate_average(data: &[f32]) -> f32 {
    let sum: f32 = data.iter().sum();
    sum / data.len() as f32
}

fn calculate_trend(data: &[f32]) -> f32 {
    // Simple linear trend
    let n = data.len();
    let mid = n / 2;

    let first_half: f32 = data[..mid].iter().sum::<f32>() / mid as f32;
    let second_half: f32 = data[mid..].iter().sum::<f32>() / (n - mid) as f32;

    second_half - first_half
}
```

## Over-the-Air Updates

### Secure WASM Module Updates

**Update manager** (device firmware):

```c
#include "wamr_runtime.h"
#include "crypto.h"
#include "flash.h"
#include <stdio.h>

#define MODULE_MAX_SIZE (128 * 1024)  // 128KB
#define SIGNATURE_SIZE 64

typedef struct {
    uint32_t version;
    uint32_t size;
    uint8_t signature[SIGNATURE_SIZE];
    uint8_t data[MODULE_MAX_SIZE];
} WasmUpdate;

int verify_module_signature(const uint8_t* data, uint32_t size,
                            const uint8_t* signature)
{
    // Verify signature using device's public key
    uint8_t hash[32];
    sha256(data, size, hash);

    return ed25519_verify(signature, hash, sizeof(hash), device_public_key);
}

int apply_wasm_update(const WasmUpdate* update) {
    printf("Applying WASM update, version %u, size %u\n",
           update->version, update->size);

    // 1. Verify signature
    if (!verify_module_signature(update->data, update->size,
                                 update->signature)) {
        printf("Invalid signature!\n");
        return -1;
    }

    // 2. Validate WASM module
    char error_buf[128];
    wasm_module_t test_module = wasm_runtime_load(
        update->data, update->size,
        error_buf, sizeof(error_buf)
    );

    if (!test_module) {
        printf("Invalid WASM module: %s\n", error_buf);
        return -1;
    }

    // 3. Test instantiation
    wasm_module_inst_t test_inst = wasm_runtime_instantiate(
        test_module, 4096, 4096, error_buf, sizeof(error_buf)
    );

    if (!test_inst) {
        wasm_runtime_unload(test_module);
        printf("Instantiation failed: %s\n", error_buf);
        return -1;
    }

    // 4. Verify required exports
    wasm_function_inst_t required_func = wasm_runtime_lookup_function(
        test_inst, "init", NULL
    );

    if (!required_func) {
        wasm_runtime_deinstantiate(test_inst);
        wasm_runtime_unload(test_module);
        printf("Missing required function\n");
        return -1;
    }

    // Module is valid, write to flash
    wasm_runtime_deinstantiate(test_inst);
    wasm_runtime_unload(test_module);

    // 5. Write to flash (backup old version first)
    flash_backup_module();
    flash_write_module(update->data, update->size);
    flash_write_version(update->version);

    printf("Update applied successfully\n");

    return 0;
}

// Rollback if new module fails
void rollback_module() {
    printf("Rolling back to previous version\n");

    flash_restore_backup();

    // Restart module
    // ...
}
```

### Incremental Updates

```c
#define CHUNK_SIZE 1024

typedef struct {
    uint32_t total_size;
    uint32_t chunk_index;
    uint32_t chunk_count;
    uint8_t chunk_data[CHUNK_SIZE];
} UpdateChunk;

static uint8_t* update_buffer = NULL;
static uint32_t received_bytes = 0;
static uint32_t total_size = 0;

int handle_update_chunk(const UpdateChunk* chunk) {
    // Initialize buffer on first chunk
    if (chunk->chunk_index == 0) {
        if (update_buffer) {
            free(update_buffer);
        }

        total_size = chunk->total_size;
        update_buffer = malloc(total_size);
        received_bytes = 0;
    }

    // Copy chunk data
    uint32_t offset = chunk->chunk_index * CHUNK_SIZE;
    uint32_t copy_size = (offset + CHUNK_SIZE > total_size) ?
                         (total_size - offset) : CHUNK_SIZE;

    memcpy(update_buffer + offset, chunk->chunk_data, copy_size);
    received_bytes += copy_size;

    printf("Received chunk %u/%u\n",
           chunk->chunk_index + 1, chunk->chunk_count);

    // Complete when all chunks received
    if (received_bytes == total_size) {
        printf("All chunks received, applying update\n");

        WasmUpdate update;
        update.size = total_size;
        memcpy(update.data, update_buffer, total_size);

        int result = apply_wasm_update(&update);

        free(update_buffer);
        update_buffer = NULL;

        return result;
    }

    return 0;  // More chunks expected
}
```

## Multi-Tenant IoT Gateway

**Plugin system for IoT gateway**:

```c
typedef struct {
    char device_id[32];
    wasm_module_t module;
    wasm_module_inst_t instance;
    uint32_t last_run_ms;
    uint32_t interval_ms;
} DevicePlugin;

#define MAX_DEVICES 32
static DevicePlugin devices[MAX_DEVICES];
static int device_count = 0;

int register_device_plugin(const char* device_id,
                           const uint8_t* wasm_data,
                           uint32_t wasm_size,
                           uint32_t interval_ms)
{
    if (device_count >= MAX_DEVICES) {
        return -1;
    }

    DevicePlugin* plugin = &devices[device_count];

    strncpy(plugin->device_id, device_id, sizeof(plugin->device_id) - 1);
    plugin->interval_ms = interval_ms;
    plugin->last_run_ms = 0;

    // Load module
    char error_buf[128];
    plugin->module = wasm_runtime_load(
        wasm_data, wasm_size,
        error_buf, sizeof(error_buf)
    );

    if (!plugin->module) {
        return -1;
    }

    // Instantiate with limited resources
    plugin->instance = wasm_runtime_instantiate(
        plugin->module,
        2048,  // 2KB stack
        2048,  // 2KB heap
        error_buf, sizeof(error_buf)
    );

    if (!plugin->instance) {
        wasm_runtime_unload(plugin->module);
        return -1;
    }

    device_count++;

    printf("Registered device plugin: %s\n", device_id);

    return device_count - 1;
}

void run_device_plugins() {
    uint32_t current_ms = get_system_time_ms();

    for (int i = 0; i < device_count; i++) {
        DevicePlugin* plugin = &devices[i];

        // Check if it's time to run
        if (current_ms - plugin->last_run_ms >= plugin->interval_ms) {
            // Call plugin's update function
            wasm_function_inst_t update_func = wasm_runtime_lookup_function(
                plugin->instance, "update", NULL
            );

            if (update_func) {
                wasm_runtime_call_wasm(
                    plugin->instance, NULL, update_func, 0, NULL
                );
            }

            plugin->last_run_ms = current_ms;
        }
    }
}
```

## Resource Management

### Memory Budgets

```c
// Per-device memory limits
#define DEVICE_STACK_SIZE 2048
#define DEVICE_HEAP_SIZE 4096

wasm_module_inst_t create_limited_instance(wasm_module_t module) {
    char error_buf[128];

    wasm_module_inst_t instance = wasm_runtime_instantiate(
        module,
        DEVICE_STACK_SIZE,
        DEVICE_HEAP_SIZE,
        error_buf, sizeof(error_buf)
    );

    if (!instance) {
        printf("Failed to instantiate: %s\n", error_buf);
        return NULL;
    }

    // Set execution time limit (100ms)
    wasm_runtime_set_max_exec_time(instance, 100);

    return instance;
}
```

### CPU Time Limits

```c
#include <time.h>

int run_with_timeout(wasm_module_inst_t instance,
                     const char* func_name,
                     uint32_t timeout_ms)
{
    wasm_function_inst_t func = wasm_runtime_lookup_function(
        instance, func_name, NULL
    );

    if (!func) {
        return -1;
    }

    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    bool success = wasm_runtime_call_wasm(instance, NULL, func, 0, NULL);

    clock_gettime(CLOCK_MONOTONIC, &end);

    uint64_t elapsed_ms =
        (end.tv_sec - start.tv_sec) * 1000 +
        (end.tv_nsec - start.tv_nsec) / 1000000;

    if (elapsed_ms > timeout_ms) {
        printf("Function exceeded timeout: %llu ms\n", elapsed_ms);
        return -1;
    }

    return success ? 0 : -1;
}
```

## Best Practices

1. **Validate modules**: Always verify signatures before loading
2. **Limit resources**: Set strict memory and CPU limits
3. **Test updates**: Validate before applying OTA updates
4. **Enable rollback**: Keep previous version for recovery
5. **Monitor health**: Track module execution metrics
6. **Sandbox strictly**: Minimal host function exposure
7. **Use AOT**: Pre-compile for consistent performance
8. **Plan for failures**: Graceful degradation
9. **Version carefully**: Semantic versioning for modules
10. **Log selectively**: Essential information only

## Next Steps

WebAssembly enables secure, updatable, and extensible IoT devices. From smart home thermostats to industrial sensor networks, WASM provides strong isolation, portability, and safe over-the-air updates—critical requirements for modern IoT deployments.

Next, in **Part 9**, we'll dive into advanced WebAssembly topics including SIMD, threads, garbage collection proposals, and cutting-edge features that are shaping WASM's future.
