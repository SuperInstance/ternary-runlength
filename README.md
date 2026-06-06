# ternary-runlength

Run-length encoding (RLE) for ternary sequences where values belong to the set **{-1, 0, +1}**. Supports basic RLE, adaptive mode selection, 2D encoding, and differential encoding.

## Overview

Run-length encoding is one of the simplest and most effective compression techniques for data with repeated values. This crate provides a comprehensive RLE implementation tailored for ternary data — sequences where every element is -1, 0, or +1. This constraint is common in sentiment analysis, ternary logic circuits, quantized neural networks, and balanced ternary computing systems.

Beyond basic RLE, this crate offers:
- **Adaptive encoding** that automatically chooses the best strategy
- **2D encoding** for ternary grids/matrices
- **Differential encoding** for smooth/slowly-changing sequences

## Features

### Basic RLE
- `encode()` — Compress a ternary sequence into runs of (value, count) pairs
- `decode()` — Decompress runs back to the original ternary sequence
- Compression ratio calculation built into results

### Adaptive RLE
- `adaptive_encode()` — Automatically selects between standard and differential RLE based on which gives better compression for the input data
- Intelligently detects ramps and smooth transitions where differential encoding excels

### 2D RLE
- `encode_2d()` — Encode 2D ternary grids by scanning in both row-major and column-major order, choosing the better result
- `decode_2d()` — Reconstruct grids from encoded runs given dimensions

### Differential RLE
- `compute_differences()` — Convert absolute values to consecutive differences
- `reconstruct_from_differences()` — Recover original data from differences
- Often achieves better compression for sequences with gradual transitions

## Usage

### Basic Encoding/Decoding

```rust
use ternary_runlength::{encode, decode};

let data = vec![-1, -1, -1, 0, 0, 1, 1, 1, 1];
let result = encode(&data);

println!("{} runs (from {} elements)", result.runs.len(), result.original_len);
println!("Compression ratio: {:.2}", result.compression_ratio());

let decoded = decode(&result.runs);
assert_eq!(data, decoded);
```

### Adaptive Encoding

```rust
use ternary_runlength::{adaptive_encode, RleMode};

let data = vec![-1, 0, 1, -1, 0, 1]; // Ramping pattern
let (result, mode) = adaptive_encode(&data);

match mode {
    RleMode::Standard => println!("Used standard RLE"),
    RleMode::Differential => println!("Used differential RLE (better for this data)"),
}
```

### 2D Encoding

```rust
use ternary_runlength::{encode_2d, decode_2d};

let grid = vec![
    vec![-1, -1, 0],
    vec![0,  0,  1],
    vec![1,  1,  1],
];

let result = encode_2d(&grid);
let decoded = decode_2d(&result.runs, 3, 3);
```

### Differential Encoding

```rust
use ternary_runlength::{compute_differences, reconstruct_from_differences, encode, decode};

let data = vec![-1, 0, 1, 0, -1]; // Slowly changing
let diffs = compute_differences(&data);
// diffs = [-1, 1, 1, -1, -1]

let encoded = encode(&diffs);
let decoded_diffs = decode(&encoded.runs);
let reconstructed = reconstruct_from_differences(&decoded_diffs);
assert_eq!(data, reconstructed);
```

## Compression Behavior

| Input Pattern | Standard RLE | Differential RLE |
|:-:|:-:|:-:|
| All same value | Excellent (1 run) | Good |
| Alternating values | Poor (expansion) | Varies |
| Gradual ramps | Moderate | Good |
| Random | Poor (expansion) | Poor (expansion) |

Compression ratio = (2 × number_of_runs) / original_length. Values below 1.0 indicate compression; above 1.0 indicates expansion.

## When to Use

- **Good fit**: Data with long runs of identical values, repetitive patterns, slowly-varying sequences
- **Poor fit**: Highly random data, rapidly alternating values (consider ternary-arithmetic instead)

## License

MIT
