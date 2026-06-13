# ternary-runlength

Run-length encoding for **ternary sequences** (values in {-1, 0, +1}). Provides standard RLE, differential RLE with adaptive selection, and 2D grid compression with row/column-major scan optimization.

## Why It Matters

Ternary weight matrices in neural networks (e.g., BitNet, TernaryBERT) are enormous but highly compressible. Consecutive weights frequently repeat or form simple arithmetic progressions, making RLE a natural fit. This crate provides:

1. **Standard RLE** — best for flat regions (consecutive identical values)
2. **Differential RLE** — best for ramps (consecutive +1 increments)
3. **Adaptive selection** — automatically picks the better scheme per sequence
4. **2D encoding** — scans row-major and column-major, picks the better

## How It Works

### Standard RLE

Consecutive identical values are grouped into `(value, count)` pairs:

```
Input:  [-1, -1, -1, 0, 0, +1, +1, +1, +1]
Output: [(-1, 3), (0, 2), (+1, 4)]
Encoded size: 2 × 3 = 6 values (vs. 9 original)
```

**Compression ratio:**

```
ρ = 2 · R / N
```

where *R* is the number of runs and *N* is the original length. `ρ < 1` means compression; `ρ > 1` means expansion (worst case: alternating pattern → 2N).

**Complexity:** O(N) encode, O(N) decode, O(1) ratio.

### Differential RLE

Instead of encoding absolute values, encode the first element plus the differences between consecutive elements, then apply RLE to the difference sequence:

```
Input:     [-1, 0, +1, 0, -1]
Diffs:     [-1, +1, +1, -1, -1]
RLE(diffs): [(-1, 1), (+1, 2), (-1, 2)]
```

Differential encoding excels when values follow arithmetic progressions (constant differences), which are common in structured ternary weight matrices.

**Reconstruction:**

```
x₀ = d₀
xₙ = xₙ₋₁ + dₙ
```

This is a cumulative sum (prefix sum), O(N).

### Adaptive Encoding

The `adaptive_encode` function computes both standard and differential RLE, then selects whichever produces fewer runs:

```
if runs(standard) ≤ runs(differential):
    use Standard
else:
    use Differential
```

This guarantees compression at least as good as either method alone.

**Complexity:** O(N) — two passes for encoding, one comparison.

### 2D Grid Encoding

For a grid of size *R × C*, the encoder scans both row-major and column-major order:

```
row_data[i·C + j] = grid[i][j]
col_data[j·R + i] = grid[i][j]
```

It then applies standard RLE to both and selects the orientation with fewer runs. This captures both horizontal and vertical spatial correlation.

**Complexity:** O(R·C) — two full scans plus two RLE passes.

### Compression Ratio Bound

For a ternary sequence of length *N* with *k* runs (standard RLE):

- Best case: *k = 1* → `ρ = 2/N` (near-zero)
- Worst case: *k = N* (alternating) → `ρ = 2` (doubles size)
- Break-even: *k = N/2* → `ρ = 1` (no compression)

## Quick Start

```rust
use ternary_runlength::{encode, decode, adaptive_encode, RleMode, Run};

let data = vec![-1, -1, -1, 0, 0, 1, 1, 1, 1];
let result = encode(&data);
assert_eq!(result.runs.len(), 3);
assert_eq!(result.compression_ratio(), 6.0 / 9.0);

let decoded = decode(&result.runs);
assert_eq!(decoded, data);

// Adaptive: picks the better of standard vs differential
let (adaptive_result, mode) = adaptive_encode(&data);
```

## API

| Function | Signature | Description |
|----------|-----------|-------------|
| `encode(data)` | `→ RleResult` | Standard RLE |
| `decode(runs)` | `→ Vec<i32>` | Decode runs to sequence |
| `adaptive_encode(data)` | `→ (RleResult, RleMode)` | Auto-select best mode |
| `compute_differences(data)` | `→ Vec<i32>` | Delta-encode |
| `reconstruct_from_differences(diffs)` | `→ Vec<i32>` | Delta-decode |
| `encode_2d(grid)` | `→ RleResult` | Best of row/col-major |
| `decode_2d(runs, rows, cols)` | `→ Vec<Vec<i32>>` | Reconstruct grid |

### Types

```rust
pub struct Run { pub value: i32, pub count: usize }
pub struct RleResult { pub runs: Vec<Run>, pub original_len: usize, pub encoded_len: usize }
pub enum RleMode { Standard, Differential }
```

## Architecture Notes

The **γ + η = C** invariant manifests in adaptive encoding: *generation* (γ) is the raw data stream, *entropy* (η) is the compressibility (measured by run count), and *conservation* (C) is the selection invariant — the adaptive encoder always achieves `runs ≤ min(runs_standard, runs_differential)`. This conservation guarantee means the encoder never does worse than either method alone.

## References

- **RLE fundamentals:** Salomon, D. *Data Compression: The Complete Reference* (2007), §1.4
- **Differential encoding for neural weights:** Qin, H. et al. "BinaryBERT" (2021) §3.2
- **2D spatial correlation:** Lempel, A. & Ziv, J. "On the Complexity of Finite Sequences" (1976)
- **Ternary weight compression:** Alemdar, H. et al. "Ternary Weight Networks" (2017)

## License

MIT
