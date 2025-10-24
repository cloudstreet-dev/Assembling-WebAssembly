# Chapter 43: SIMD in WebAssembly

## What is SIMD?

SIMD (Single Instruction, Multiple Data) allows processing multiple data elements with a single instruction. WebAssembly SIMD provides 128-bit vector operations, enabling significant performance improvements for data-parallel workloads.

**Performance gains**: 2-4x for many algorithms (image processing, audio, physics, etc.)

## The v128 Type

WASM SIMD introduces the `v128` type representing a 128-bit vector that can be interpreted as:

- **16 × i8**: 16 8-bit integers
- **8 × i16**: 8 16-bit integers
- **4 × i32**: 4 32-bit integers
- **2 × i64**: 2 64-bit integers
- **4 × f32**: 4 32-bit floats
- **2 × f64**: 2 64-bit floats

## Feature Detection

```javascript
const simdSupported = typeof WebAssembly.validate === 'function' &&
    WebAssembly.validate(new Uint8Array([
        0x00, 0x61, 0x73, 0x6d,  // magic
        0x01, 0x00, 0x00, 0x00,  // version
        0x01, 0x05, 0x01,        // type section
        0x60, 0x00, 0x01, 0x7b   // v128 result type
    ]));

console.log('SIMD supported:', simdSupported);
```

## Rust SIMD

### Basic SIMD Operations

```rust
#![feature(wasm_simd)]

use core::arch::wasm32::*;

#[no_mangle]
pub unsafe extern "C" fn add_arrays_simd(
    a: *const f32,
    b: *const f32,
    result: *mut f32,
    len: usize,
) {
    let chunks = len / 4;

    for i in 0..chunks {
        let offset = i * 4;

        // Load 4 floats from each array
        let va = v128_load(a.add(offset) as *const v128);
        let vb = v128_load(b.add(offset) as *const v128);

        // Add vectors
        let vr = f32x4_add(va, vb);

        // Store result
        v128_store(result.add(offset) as *mut v128, vr);
    }

    // Handle remaining elements (scalar)
    for i in (chunks * 4)..len {
        *result.add(i) = *a.add(i) + *b.add(i);
    }
}
```

### SIMD vs Scalar Comparison

```rust
// Scalar version
#[no_mangle]
pub unsafe extern "C" fn add_arrays_scalar(
    a: *const f32,
    b: *const f32,
    result: *mut f32,
    len: usize,
) {
    for i in 0..len {
        *result.add(i) = *a.add(i) + *b.add(i);
    }
}

// SIMD version (4x faster for large arrays)
#[no_mangle]
pub unsafe extern "C" fn add_arrays_simd(
    a: *const f32,
    b: *const f32,
    result: *mut f32,
    len: usize,
) {
    let chunks = len / 4;

    for i in 0..chunks {
        let offset = i * 4;

        let va = v128_load(a.add(offset) as *const v128);
        let vb = v128_load(b.add(offset) as *const v128);
        let vr = f32x4_add(va, vb);

        v128_store(result.add(offset) as *mut v128, vr);
    }

    // Scalar remainder
    for i in (chunks * 4)..len {
        *result.add(i) = *a.add(i) + *b.add(i);
    }
}
```

## Common SIMD Patterns

### Dot Product

```rust
use core::arch::wasm32::*;

#[no_mangle]
pub unsafe extern "C" fn dot_product(a: *const f32, b: *const f32, len: usize) -> f32 {
    let chunks = len / 4;
    let mut sum_vec = f32x4_splat(0.0);

    for i in 0..chunks {
        let offset = i * 4;

        let va = v128_load(a.add(offset) as *const v128);
        let vb = v128_load(b.add(offset) as *const v128);

        // Multiply and accumulate
        let prod = f32x4_mul(va, vb);
        sum_vec = f32x4_add(sum_vec, prod);
    }

    // Extract and sum components
    let mut result = f32x4_extract_lane::<0>(sum_vec)
                   + f32x4_extract_lane::<1>(sum_vec)
                   + f32x4_extract_lane::<2>(sum_vec)
                   + f32x4_extract_lane::<3>(sum_vec);

    // Add scalar remainder
    for i in (chunks * 4)..len {
        result += *a.add(i) * *b.add(i);
    }

    result
}
```

### Min/Max Reduction

```rust
#[no_mangle]
pub unsafe extern "C" fn find_min(data: *const f32, len: usize) -> f32 {
    if len == 0 {
        return f32::INFINITY;
    }

    let chunks = len / 4;

    // Initialize with first value repeated
    let mut min_vec = f32x4_splat(*data);

    for i in 0..chunks {
        let v = v128_load(data.add(i * 4) as *const v128);
        min_vec = f32x4_min(min_vec, v);
    }

    // Extract minimum
    let mut result = f32x4_extract_lane::<0>(min_vec)
        .min(f32x4_extract_lane::<1>(min_vec))
        .min(f32x4_extract_lane::<2>(min_vec))
        .min(f32x4_extract_lane::<3>(min_vec));

    // Check scalar remainder
    for i in (chunks * 4)..len {
        result = result.min(*data.add(i));
    }

    result
}
```

### Data Transformation

```rust
// Convert RGBA to grayscale
#[no_mangle]
pub unsafe extern "C" fn rgba_to_gray(
    rgba: *const u8,
    gray: *mut u8,
    pixel_count: usize,
) {
    // Weights: R=0.299, G=0.587, B=0.114 (scaled to u8)
    let weights_r = u8x16_splat(77);   // 0.299 * 256
    let weights_g = u8x16_splat(150);  // 0.587 * 256
    let weights_b = u8x16_splat(29);   // 0.114 * 256

    let chunks = pixel_count / 4;  // Process 4 pixels (16 bytes) at a time

    for i in 0..chunks {
        // Load 4 RGBA pixels (16 bytes)
        let pixels = v128_load(rgba.add(i * 16) as *const v128);

        // Shuffle to separate R, G, B channels
        // This is simplified; actual implementation needs careful shuffling

        // weighted_sum = R*0.299 + G*0.587 + B*0.114
        // Store 4 grayscale values

        // For simplicity, showing concept:
        for j in 0..4 {
            let offset = (i * 4 + j) * 4;
            let r = *rgba.add(offset) as u16;
            let g = *rgba.add(offset + 1) as u16;
            let b = *rgba.add(offset + 2) as u16;

            let gray_val = ((r * 77 + g * 150 + b * 29) >> 8) as u8;
            *gray.add(i * 4 + j) = gray_val;
        }
    }

    // Scalar remainder
    for i in (chunks * 4)..pixel_count {
        let offset = i * 4;
        let r = *rgba.add(offset) as u16;
        let g = *rgba.add(offset + 1) as u16;
        let b = *rgba.add(offset + 2) as u16;

        *gray.add(i) = ((r * 77 + g * 150 + b * 29) >> 8) as u8;
    }
}
```

## Image Processing

### Blur Filter (Box Blur)

```rust
#[no_mangle]
pub unsafe extern "C" fn box_blur(
    src: *const u8,
    dst: *mut u8,
    width: usize,
    height: usize,
) {
    let kernel_size = 3;
    let half = kernel_size / 2;

    for y in half..(height - half) {
        for x in half..(width - half) {
            let mut sum = u32x4_splat(0);

            // Sum 3x3 neighborhood
            for ky in 0..kernel_size {
                for kx in 0..kernel_size {
                    let py = y + ky - half;
                    let px = x + kx - half;
                    let idx = (py * width + px) * 4;

                    let pixel = v128_load(src.add(idx) as *const v128);

                    // Extend u8 to u32 and accumulate
                    // (simplified - actual implementation needs proper widening)
                }
            }

            // Divide by kernel area (9)
            // sum = u32x4_div(sum, 9);

            // Store result
            // (simplified)
        }
    }
}
```

### Image Scaling (Nearest Neighbor)

```rust
#[no_mangle]
pub unsafe extern "C" fn scale_image(
    src: *const u8,
    dst: *mut u8,
    src_width: usize,
    src_height: usize,
    dst_width: usize,
    dst_height: usize,
) {
    let x_ratio = (src_width << 16) / dst_width;
    let y_ratio = (src_height << 16) / dst_height;

    for y in 0..dst_height {
        let src_y = (y * y_ratio) >> 16;

        for x_chunk in 0..(dst_width / 4) {
            // Process 4 output pixels
            for i in 0..4 {
                let x = x_chunk * 4 + i;
                let src_x = (x * x_ratio) >> 16;

                let src_idx = (src_y * src_width + src_x) * 4;
                let dst_idx = (y * dst_width + x) * 4;

                // Copy 4 bytes (RGBA)
                *dst.add(dst_idx) = *src.add(src_idx);
                *dst.add(dst_idx + 1) = *src.add(src_idx + 1);
                *dst.add(dst_idx + 2) = *src.add(src_idx + 2);
                *dst.add(dst_idx + 3) = *src.add(src_idx + 3);
            }
        }
    }
}
```

## Audio Processing

### Mixing Audio Channels

```rust
#[no_mangle]
pub unsafe extern "C" fn mix_audio(
    channel1: *const f32,
    channel2: *const f32,
    output: *mut f32,
    len: usize,
    volume1: f32,
    volume2: f32,
) {
    let vol1_vec = f32x4_splat(volume1);
    let vol2_vec = f32x4_splat(volume2);

    let chunks = len / 4;

    for i in 0..chunks {
        let offset = i * 4;

        let ch1 = v128_load(channel1.add(offset) as *const v128);
        let ch2 = v128_load(channel2.add(offset) as *const v128);

        // Apply volume
        let ch1_scaled = f32x4_mul(ch1, vol1_vec);
        let ch2_scaled = f32x4_mul(ch2, vol2_vec);

        // Mix (add)
        let mixed = f32x4_add(ch1_scaled, ch2_scaled);

        v128_store(output.add(offset) as *mut v128, mixed);
    }

    // Scalar remainder
    for i in (chunks * 4)..len {
        *output.add(i) = *channel1.add(i) * volume1 + *channel2.add(i) * volume2;
    }
}
```

### Apply Audio Gain

```rust
#[no_mangle]
pub unsafe extern "C" fn apply_gain(
    samples: *mut f32,
    len: usize,
    gain: f32,
) {
    let gain_vec = f32x4_splat(gain);
    let chunks = len / 4;

    for i in 0..chunks {
        let offset = i * 4;

        let v = v128_load(samples.add(offset) as *const v128);
        let scaled = f32x4_mul(v, gain_vec);

        v128_store(samples.add(offset) as *mut v128, scaled);
    }

    // Scalar remainder
    for i in (chunks * 4)..len {
        *samples.add(i) *= gain;
    }
}
```

## Physics and Mathematics

### Vector Operations

```rust
// 3D vector addition with SIMD
#[repr(C, align(16))]
struct Vec3 {
    x: f32,
    y: f32,
    z: f32,
    _pad: f32,  // Padding for alignment
}

#[no_mangle]
pub unsafe extern "C" fn vec3_add_batch(
    a: *const Vec3,
    b: *const Vec3,
    result: *mut Vec3,
    count: usize,
) {
    for i in 0..count {
        let va = v128_load(a.add(i) as *const v128);
        let vb = v128_load(b.add(i) as *const v128);
        let vr = f32x4_add(va, vb);

        v128_store(result.add(i) as *mut v128, vr);
    }
}

// Matrix-vector multiplication (4x4 matrix, 4D vector)
#[no_mangle]
pub unsafe extern "C" fn mat4_mul_vec4(
    mat: *const f32,  // 16 floats (row-major)
    vec: *const f32,  // 4 floats
    result: *mut f32,
) {
    let v = v128_load(vec as *const v128);

    for row in 0..4 {
        let row_vec = v128_load(mat.add(row * 4) as *const v128);

        // Multiply and sum
        let prod = f32x4_mul(row_vec, v);
        let sum = f32x4_extract_lane::<0>(prod)
                + f32x4_extract_lane::<1>(prod)
                + f32x4_extract_lane::<2>(prod)
                + f32x4_extract_lane::<3>(prod);

        *result.add(row) = sum;
    }
}
```

## C/C++ SIMD

### With Clang/wasm-simd128

```c
#include <wasm_simd128.h>

void add_arrays_simd(const float* a, const float* b, float* result, size_t len) {
    size_t chunks = len / 4;

    for (size_t i = 0; i < chunks; i++) {
        v128_t va = wasm_v128_load(&a[i * 4]);
        v128_t vb = wasm_v128_load(&b[i * 4]);
        v128_t vr = wasm_f32x4_add(va, vb);

        wasm_v128_store(&result[i * 4], vr);
    }

    // Scalar remainder
    for (size_t i = chunks * 4; i < len; i++) {
        result[i] = a[i] + b[i];
    }
}

// RGB to grayscale
void rgb_to_gray(const uint8_t* rgb, uint8_t* gray, size_t pixel_count) {
    v128_t weights = wasm_u16x8_make(77, 150, 29, 0, 77, 150, 29, 0);

    size_t chunks = pixel_count / 8;  // Process 8 pixels at a time

    for (size_t i = 0; i < chunks * 8; i++) {
        size_t rgb_idx = i * 3;

        uint8_t r = rgb[rgb_idx];
        uint8_t g = rgb[rgb_idx + 1];
        uint8_t b = rgb[rgb_idx + 2];

        uint16_t gray_val = (r * 77 + g * 150 + b * 29) >> 8;
        gray[i] = (uint8_t)gray_val;
    }

    // Scalar remainder
    for (size_t i = chunks * 8; i < pixel_count; i++) {
        size_t rgb_idx = i * 3;
        gray[i] = (uint8_t)((rgb[rgb_idx] * 77 +
                             rgb[rgb_idx + 1] * 150 +
                             rgb[rgb_idx + 2] * 29) >> 8);
    }
}
```

**Compile**:
```bash
clang --target=wasm32 -msimd128 -O3 -o simd.wasm simd.c
```

## JavaScript Interface

```javascript
const { instance } = await WebAssembly.instantiateStreaming(
    fetch('simd.wasm'),
    {}
);

// Allocate arrays in WASM memory
const len = 1024;
const bytesPerFloat = 4;

const aPtr = instance.exports.alloc(len * bytesPerFloat);
const bPtr = instance.exports.alloc(len * bytesPerFloat);
const resultPtr = instance.exports.alloc(len * bytesPerFloat);

// Create views
const memory = new Float32Array(instance.exports.memory.buffer);

const a = memory.subarray(aPtr / 4, aPtr / 4 + len);
const b = memory.subarray(bPtr / 4, bPtr / 4 + len);
const result = memory.subarray(resultPtr / 4, resultPtr / 4 + len);

// Fill input arrays
for (let i = 0; i < len; i++) {
    a[i] = i;
    b[i] = i * 2;
}

// Call SIMD function
instance.exports.add_arrays_simd(aPtr, bPtr, resultPtr, len);

console.log('Result:', Array.from(result.slice(0, 10)));

// Compare with scalar
const resultScalarPtr = instance.exports.alloc(len * bytesPerFloat);
instance.exports.add_arrays_scalar(aPtr, bPtr, resultScalarPtr, len);

// Verify results match
const resultScalar = memory.subarray(resultScalarPtr / 4, resultScalarPtr / 4 + len);
const match = result.every((v, i) => v === resultScalar[i]);
console.log('SIMD matches scalar:', match);

// Cleanup
instance.exports.free(aPtr);
instance.exports.free(bPtr);
instance.exports.free(resultPtr);
instance.exports.free(resultScalarPtr);
```

## Performance Benchmarking

```javascript
function benchmark(name, fn, iterations = 1000) {
    const start = performance.now();

    for (let i = 0; i < iterations; i++) {
        fn();
    }

    const end = performance.now();
    const avg = (end - start) / iterations;

    console.log(`${name}: ${avg.toFixed(3)}ms per iteration`);
}

// Benchmark scalar vs SIMD
const len = 100000;

benchmark('Scalar Add', () => {
    instance.exports.add_arrays_scalar(aPtr, bPtr, resultPtr, len);
});

benchmark('SIMD Add', () => {
    instance.exports.add_arrays_simd(aPtr, bPtr, resultPtr, len);
});

// Typical results:
// Scalar Add: 0.123ms per iteration
// SIMD Add: 0.031ms per iteration
// ~4x speedup
```

## Best Practices

1. **Check alignment**: SIMD loads/stores prefer 16-byte alignment
2. **Handle remainders**: Always process non-SIMD-sized tail
3. **Benchmark**: Measure actual performance gains
4. **Use when appropriate**: Best for data-parallel workloads
5. **Compile with -O3**: Optimizations crucial for SIMD
6. **Test portability**: Ensure fallback for non-SIMD platforms
7. **Profile first**: Identify hotspots before optimizing
8. **Avoid excessive shuffling**: Minimize data rearrangement

## Common Use Cases

### Image Processing
- Color space conversions
- Filters (blur, sharpen, edge detection)
- Alpha blending
- Scaling and rotation

### Audio
- Mixing and mastering
- Effects (reverb, echo, equalization)
- Format conversion
- Compression/decompression

### Physics
- Particle systems
- Collision detection
- Vector/matrix math
- Numerical integration

### Data Processing
- Statistics (mean, variance, correlation)
- Signal processing (FFT, filtering)
- Cryptography (hashing, encryption)
- Compression

## Next Steps

SIMD in WebAssembly enables significant performance improvements for data-parallel workloads. By leveraging v128 vectors, you can process 4 floats or integers simultaneously, achieving 2-4x speedups for appropriate algorithms.

Next, we'll explore threading in WebAssembly, enabling true parallel execution across multiple cores with shared memory and atomics.
