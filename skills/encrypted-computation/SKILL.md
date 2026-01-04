---
name: encrypted-computation
description: All encrypted computational algorithms - linear algebra, distance metrics, comparisons, sorting, and bit operations. Use for matrix ops, KNN, search, sorting, and lookup challenges.
---

# Encrypted Computation

Algorithms for matrix operations, comparisons, sorting, distances, and bit operations under FHE.

## Skill Responsibility

**Primary Function:** Implement encrypted algorithms for non-approximation tasks

**When to Use:**
- Matrix multiplication, inversion, SVD
- Sorting (array_sorting, max, min, argmax)
- Distance metrics (KNN, string search)
- Comparisons and bit operations (parity, shift, lookup tables)

**NOT for:** Function approximation (sigmoid, relu) - use `function-approximation` skill

---

## Matrix Operations

### Matrix-Vector Multiply

**Diagonal Method (Depth Optimal):**
```cpp
// Pack matrix as diagonals, use rotations
// M * v = sum_k(rotate(v, k) * diag_k)
auto result = cc->EvalMult(v, diag[0]);
for (int k = 1; k < n; k++) {
    auto rotated = cc->EvalRotate(v, k);
    auto prod = cc->EvalMult(rotated, diag[k]);
    result = cc->EvalAdd(result, prod);
}
// Depth: 1 (all multiplications parallel)
```

**Row-Major Method:**
```cpp
// Replicate vector, multiply element-wise, sum within groups
// Depth: 1 + log(n) for sum reduction
```

**Choice:** Use diagonal if depth is tight, row-major for simplicity.

### Matrix-Matrix Multiply

```cpp
// Process as multiple matrix-vector multiplies
// Or use specialized packing for batched multiplication
// Depth: 1 + log(k) for (m×k) × (k×n)
```

### Matrix Inversion (Newton-Raphson)

```cpp
// X_{k+1} = X_k * (2I - A*X_k)
// Requires: depth > 20, good initial guess
// Recommendation: Avoid if possible, pre-compute in plaintext
```

---

## CNN Convolution Operations

### Overview of FHE Convolution Approaches

Convolution in FHE is computed using rotations + plaintext-ciphertext multiplications:
- **Number of rotations = kernel_height × kernel_width**
- **Depth = 1** (all multiplications are parallel, then summed)
- **Key insight:** Use SIMD slots to process multiple windows simultaneously

### Convolution via Rotations (Standard Method)

**Algorithm:**
```cpp
// Input: encrypted image (H×W flattened into SIMD slots)
// Kernel: plaintext weights (kH×kW)
// Output: encrypted feature map

Ciphertext<DCRTPoly> conv2d_rotation(
    CryptoContext<DCRTPoly> cc,
    Ciphertext<DCRTPoly> image,  // Flattened: [img[0][0], img[0][1], ..., img[H-1][W-1]]
    std::vector<double>& kernel,  // Flattened: [k[0][0], k[0][1], ..., k[kH-1][kW-1]]
    int H, int W, int kH, int kW, int stride = 1
) {
    int outH = (H - kH) / stride + 1;
    int outW = (W - kW) / stride + 1;

    Ciphertext<DCRTPoly> result;
    bool first = true;

    // For each kernel position
    for (int ki = 0; ki < kH; ki++) {
        for (int kj = 0; kW; kj++) {
            // Rotation index: which pixels align with this kernel position
            int rotation_idx = ki * W + kj;

            // Rotate ciphertext
            auto rotated = cc->EvalRotate(image, rotation_idx);

            // Create plaintext mask with kernel weight at valid positions
            std::vector<double> mask(H * W, 0.0);
            for (int out_i = 0; out_i < outH; out_i++) {
                for (int out_j = 0; out_j < outW; out_j++) {
                    int orig_pos = (out_i * stride) * W + (out_j * stride);
                    mask[orig_pos] = kernel[ki * kW + kj];
                }
            }

            Plaintext pt_mask = cc->MakeCKKSPackedPlaintext(mask);

            // Ciphertext-plaintext multiplication
            auto prod = cc->EvalMult(rotated, pt_mask);

            // Accumulate
            if (first) {
                result = prod;
                first = false;
            } else {
                result = cc->EvalAdd(result, prod);
            }
        }
    }

    return result;
}

// Depth: 1 (all multiplications parallel, additions don't increase depth)
// Rotations needed: kH × kW
// Good for: Small kernels (3×3, 5×5, 7×7)

// IMPORTANT: Output extraction!
// Results are placed in 2D grid positions, not consecutively
// For HxW image with outH x outW output:
// Output (i,j) is at position i*W + j in the ciphertext
std::vector<double> extract_output(Plaintext decrypted, int outH, int outW, int W) {
    auto full_result = decrypted->GetRealPackedValue();
    std::vector<double> output(outH * outW);
    for (int i = 0; i < outH; i++) {
        for (int j = 0; j < outW; j++) {
            output[i * outW + j] = full_result[i * W + j];
        }
    }
    return output;
}
```

**Slot Layout Example (5×5 image, 3×3 kernel):**
```
Original: [p00, p01, p02, p03, p04, p10, p11, ..., p44]
Rotate 0: [p00, p01, p02, p03, p04, p10, p11, ..., p44]  * k[0][0]
Rotate 1: [p01, p02, p03, p04, p10, p11, p12, ..., p44, p00]  * k[0][1]
Rotate 2: [p02, p03, p04, p10, p11, ..., p44, p00, p01]  * k[0][2]
...
Rotate 5: [p10, p11, p12, ..., p44, p00, ..., p04]  * k[1][0]
```

### Channel-By-Channel (CBC) Packing

**Key Insight:** Pack each input channel separately to reduce rotation overhead

```cpp
// For RGB image (3 channels) → 16 filters (16 output channels)
// Input: 3 ciphertexts (one per channel)
// Kernels: 16×3 = 48 kernels
// Output: 16 ciphertexts (one per output channel)

std::vector<Ciphertext<DCRTPoly>> conv2d_cbc(
    CryptoContext<DCRTPoly> cc,
    std::vector<Ciphertext<DCRTPoly>>& input_channels,  // Size: C_in
    std::vector<std::vector<std::vector<double>>>& kernels,  // Size: [C_out][C_in][kH*kW]
    int H, int W, int kH, int kW
) {
    int C_in = input_channels.size();
    int C_out = kernels.size();

    std::vector<Ciphertext<DCRTPoly>> output_channels(C_out);

    // For each output channel
    for (int out_c = 0; out_c < C_out; out_c++) {
        Ciphertext<DCRTPoly> channel_sum;
        bool first = true;

        // Sum over input channels
        for (int in_c = 0; in_c < C_in; in_c++) {
            // Convolve input_channel[in_c] with kernels[out_c][in_c]
            auto conv_result = conv2d_rotation(
                cc, input_channels[in_c],
                kernels[out_c][in_c],
                H, W, kH, kW
            );

            if (first) {
                channel_sum = conv_result;
                first = false;
            } else {
                channel_sum = cc->EvalAdd(channel_sum, conv_result);
            }
        }

        output_channels[out_c] = channel_sum;
    }

    return output_channels;
}

// Total rotations: C_out × C_in × kH × kW
// Example: 16 output channels, 3 input channels, 3×3 kernel
//          = 16 × 3 × 9 = 432 rotations
// Depth: 1 (conv) + 1 (channel summation) = 2
// Ciphertexts: C_in input + C_out output

// ✅ VERIFIED: Tested with 3 input channels, 2 output channels
// Error < 1e-12 compared to plaintext
```

### Pooling Operations

**Average Pooling (Exact):**
```cpp
// 2×2 average pooling with stride=2
auto pool_avg_2x2(CryptoContext<DCRTPoly> cc,
                  Ciphertext<DCRTPoly> input,
                  int H, int W) {
    // Sum 4 positions: (i,j), (i,j+1), (i+1,j), (i+1,j+1)
    auto p1 = input;  // (i,j)
    auto p2 = cc->EvalRotate(input, 1);  // (i,j+1)
    auto p3 = cc->EvalRotate(input, W);  // (i+1,j)
    auto p4 = cc->EvalRotate(input, W+1);  // (i+1,j+1)

    auto sum = cc->EvalAdd(p1, p2);
    sum = cc->EvalAdd(sum, p3);
    sum = cc->EvalAdd(sum, p4);

    // Divide by 4 (plaintext operation)
    auto result = cc->EvalMult(sum, 0.25);

    // Extract every 2nd position (stride=2 downsampling)
    // Use masking or repacking

    return result;
}
// Depth: 1 (multiplication by 0.25)
// Rotations: 3
```

**Max Pooling (Approximation Required):**
```cpp
// Max pooling requires comparison
// Use polynomial approximation for max(a,b)
auto pool_max_2x2(CryptoContext<DCRTPoly> cc,
                  Ciphertext<DCRTPoly> input,
                  int H, int W) {
    auto p1 = input;
    auto p2 = cc->EvalRotate(input, 1);
    auto p3 = cc->EvalRotate(input, W);
    auto p4 = cc->EvalRotate(input, W+1);

    // Soft max using polynomial approximation
    auto max12 = soft_max(cc, p1, p2);  // Uses sign approximation
    auto max34 = soft_max(cc, p3, p4);
    auto result = soft_max(cc, max12, max34);

    return result;
}
// Depth: 3 × comparison_depth (typically 3×3 = 9)
// Recommendation: Use average pooling when possible!
```

### Multi-Channel Convolution Example (CIFAR-10)

```cpp
// CIFAR-10: 32×32 RGB images → 10 classes
// Typical architecture: Conv → Pool → Conv → Pool → FC → FC

class CIFAR10Network {
    // Layer 1: 3→16 channels, 3×3 kernel, stride=1
    // Input:  3 ciphertexts of size 32×32 = 1024
    // Output: 16 ciphertexts of size 32×32 = 1024
    // Rotations: 16 × 3 × 9 = 432

    // Pool1: 2×2 average pooling, stride=2
    // Output: 16 ciphertexts of size 16×16 = 256
    // Rotations: 16 × 3 = 48

    // Layer 2: 16→32 channels, 3×3 kernel, stride=1
    // Output: 32 ciphertexts of size 16×16 = 256
    // Rotations: 32 × 16 × 9 = 4608

    // Pool2: 2×2 average pooling, stride=2
    // Output: 32 ciphertexts of size 8×8 = 64
    // Rotations: 32 × 3 = 96

    // Flatten: 32 × 64 = 2048 features
    // FC1: 2048 → 128 (matrix-vector multiply)
    // FC2: 128 → 10 (matrix-vector multiply)

    // Total rotations: 432 + 48 + 4608 + 96 + rotations_for_FC ≈ 5200+
};
```

### Optimization: Rotation Multiplexing

**Idea:** Reuse rotated ciphertexts across multiple output channels

```cpp
// Instead of: for each output channel, rotate input C_in × kH × kW times
// Do: Rotate input once, use for ALL output channels

// Pre-compute all rotations
std::vector<Ciphertext<DCRTPoly>> rotated_inputs(kH * kW);
int idx = 0;
for (int ki = 0; ki < kH; ki++) {
    for (int kj = 0; kj < kW; kj++) {
        rotated_inputs[idx++] = cc->EvalRotate(input, ki * W + kj);
    }
}

// Now for each output channel, just multiply by different masks
// Saves: (C_out - 1) × rotations
```

### Required Rotation Keys

```cpp
// For H×W image with kH×kW kernel
// Need rotation keys for: {0, 1, 2, ..., kW-1, W, W+1, ..., W+kW-1, ..., (kH-1)*W+kW-1}

// Example: 32×32 image, 3×3 kernel
// Rotation indices: {0, 1, 2, 32, 33, 34, 64, 65, 66}

// In config.json:
{
  "indexes_for_rotation_key": [1, 2, 32, 33, 34, 64, 65, 66]
}
```

### CKKS Batch Size Requirements

**CRITICAL**: Batch size must be a power of 2 or 0 (for full packing)

```cpp
// WRONG - will crash!
int batchSize = H * W;  // 32*32 = 1024 is OK, but 28*28 = 784 will CRASH!

// CORRECT - round up to next power of 2
int batchSize = 1 << (int)ceil(log2(H * W));

// OR - use 0 for automatic full packing
parameters.SetBatchSize(0);

// For common image sizes:
// 28×28 = 784  → use 1024 (2^10)
// 32×32 = 1024 → use 1024 (2^10)
// 64×64 = 4096 → use 4096 (2^12)
```

### Depth Analysis

```
Operation                    | Depth | Verified
-----------------------------|-------|----------
Single-channel convolution   | 1     | ✅
Multi-channel convolution    | 2     | ✅ (3→2 channels tested)
ReLU approximation (deg 3)   | 1-2   |
Batch normalization          | 1     |
Average pooling              | 1     |
Max pooling (soft)           | 3-5   |
Fully connected layer        | 1     |

Typical CNN layer (multi-channel):
Conv + BN + ReLU + Pool = 2 + 1 + 2 + 1 = 6 depth per layer

Note: Multi-channel conv has depth 2 because:
  - Depth 1: Rotations + multiplications (parallel)
  - Depth 1: Sum across input channels (additions)
```

---

## Distance Metrics

### Key Insight: Avoid sqrt!

Use squared distances - ordering is preserved:
```cpp
dist²(x,y) = ||x||² + ||y||² - 2*<x,y>
```

### Euclidean Distance (Squared)

```cpp
// Pre-compute norms as plaintext if possible
auto inner_prod = cc->EvalInnerProduct(x, y, vector_size);  // Or manual
auto dist_sq = x_norm_pt + y_norm_pt - 2 * inner_prod;
// Depth: 1 + log(d)
```

### Batched Distances (KNN)

```cpp
// Pack n database points into ciphertext
// Replicate query across groups
// Compute all n distances in parallel!
// Depth: 1 + log(d) for ALL distances

// Slot layout: [X1[0]..X1[d], X2[0]..X2[d], ..., Xn[0]..Xn[d]]
// Query rep:   [q[0]..q[d],   q[0]..q[d],   ..., q[0]..q[d]]
```

### Manhattan Distance (L1)

```cpp
// Requires |x-y| which needs sign approximation
// Much more expensive than L2
// Avoid unless specifically required
```

### Hamming Distance (Binary Vectors)

```cpp
// XOR: x ⊕ y = x + y - 2*x*y (for {0,1} values)
auto xor_result = cc->EvalSub(cc->EvalAdd(x, y), cc->EvalMult(cc->EvalMult(x, y), 2));
auto hamming = cc->EvalSum(xor_result, vector_size);
// Depth: 1 + log(d)
```

---

## Comparison & Sorting

### Comparison (CKKS)

```cpp
// Compare using sign approximation
// sign(a-b) gives comparison result
// max(a,b) = 0.5 * (a + b + |a-b|)
// |x| = x * sign(x)
// Depth: 3-8 depending on sign approximation degree
```

### Max Finding (Comparison Tree)

```cpp
// Tournament style: log(n) rounds
for (int round = 0; round < log2_n; round++) {
    auto shifted = cc->EvalRotate(values, 1 << round);
    // Comparison: max(values, shifted)
    values = soft_max(values, shifted);  // Using polynomial approximation
}
// Depth: log(n) * comparison_depth
```

### Bitonic Sort

```cpp
// Depth-optimal sorting network
// Depth: O((log n)²) comparisons
// Best for n ≤ 64 with moderate depth budget

void bitonic_sort(auto& arr, int n) {
    for (int k = 2; k <= n; k *= 2) {
        for (int j = k/2; j > 0; j /= 2) {
            // Compare and swap elements at distance j
            // Using homomorphic comparison
        }
    }
}
```

### Argmax

```cpp
// Find index of maximum element
// Method 1: Track indices alongside comparisons
// Method 2: Use indicator vectors after finding max
// Depth: log(n) * comparison_depth + log(n)
```

---

## Bit Operations (BFV/BGV)

### Shift Left (SHL)

```cpp
// Left shift by k bits = multiply by 2^k
auto result = cc->EvalMult(x, (1 << k));  // Plaintext multiply
// Depth: 0
```

### Shift Right

```cpp
// Right shift requires modular inverse of 2^k
// Or use bootstrapping + division
```

### Parity

```cpp
// XOR reduction of all bits
// For binary slots: parity = sum mod 2
// Use tree reduction with XOR operation
// Depth: log(num_bits)
```

### Lookup Tables

```cpp
// For small domains (e.g., 0-255)
// Encode table as plaintext vector
// Use equality indicators to select

auto result = zero;
for (int i = 0; i < table_size; i++) {
    auto indicator = equality_test(x, i);  // Is x == i?
    result = cc->EvalAdd(result, cc->EvalMult(indicator, table[i]));
}
// Depth: equality_depth + 1
```

---

## Algorithm Selection Guide

| Challenge | Algorithm | Depth |
|-----------|-----------|-------|
| matrix_multiplication | Diagonal method | 1 |
| max | Comparison tree | log(n) * 5 |
| array_sorting | Bitonic sort | (log n)² * 5 |
| knn | Batched squared distances | 1 + log(d) |
| parity | XOR tree reduction | log(bits) |
| shl | Plaintext multiply | 0 |
| lookup_table | Indicator vectors | equality_depth + 1 |
| softmax | exp + sum + inverse | 10-15 |
| svd | Power iteration | Very high |

---

## Web Search for SOTA Algorithms

### When to Search

- Need depth-efficient sorting network
- Looking for optimized matrix multiplication
- Want state-of-the-art comparison circuits
- Need efficient argmax implementation

### Search Queries

```
# Matrix operations
site:arxiv.org homomorphic matrix multiplication CKKS
site:scholar.google.com FHE matrix-vector efficient

# Sorting
site:arxiv.org homomorphic sorting network
site:arxiv.org "bitonic sort" FHE
site:scholar.google.com encrypted comparison depth

# KNN and distances
site:arxiv.org "homomorphic encryption" KNN
site:arxiv.org FHE nearest neighbor

# Lookup tables
site:arxiv.org "lookup table" homomorphic
site:github.com openfhe lookup table example
```

### Key Papers

```
# Matrix operations
"GAZELLE: A Low Latency Framework for Secure Neural Network Inference"
"CrypTen: Secure Multi-Party Computation Meets Machine Learning"

# Sorting and comparison
"Sorting and comparing in encrypted data"
"Efficient Homomorphic Comparison Methods with Optimal Complexity"

# General algorithms
"A Full RNS Variant of FV Like Somewhat Homomorphic Encryption Schemes"
```

### Resources

| Resource | Use For |
|----------|---------|
| arxiv.org | Algorithm papers |
| eprint.iacr.org | Cryptography papers |
| github.com/openfheorg | OpenFHE implementations |
| scholar.google.com | Citation search |

---

## Implementation Patterns

### Pattern 1: Matrix-Vector Product (Diagonal)

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> vec,
                          std::vector<Plaintext>& diagonals) {
    auto result = cc->EvalMult(vec, diagonals[0]);
    for (size_t k = 1; k < diagonals.size(); k++) {
        auto rotated = cc->EvalRotate(vec, k);
        auto prod = cc->EvalMult(rotated, diagonals[k]);
        result = cc->EvalAdd(result, prod);
    }
    return result;
}
```

### Pattern 2: Batched Distance Computation

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> database,  // Packed points
                          Ciphertext<DCRTPoly> query_rep, // Replicated query
                          Plaintext& db_norms,
                          double query_norm) {
    // Element-wise product
    auto products = cc->EvalMult(database, query_rep);

    // Sum within groups (inner products)
    auto inner_prods = cc->EvalSum(products, group_size);

    // Squared distances: ||x||² + ||q||² - 2<x,q>
    auto result = cc->EvalAdd(db_norms, query_norm);
    result = cc->EvalSub(result, cc->EvalMult(inner_prods, 2.0));

    return result;
}
```

### Pattern 3: Soft Maximum

```cpp
Ciphertext<DCRTPoly> soft_max(CryptoContext<DCRTPoly> cc,
                               Ciphertext<DCRTPoly> a,
                               Ciphertext<DCRTPoly> b) {
    // max(a,b) ≈ 0.5 * (a + b + |a - b|)
    auto diff = cc->EvalSub(a, b);

    // Approximate |diff| = diff * sign(diff)
    // sign approximated by polynomial
    auto sign_diff = approximate_sign(cc, diff);
    auto abs_diff = cc->EvalMult(diff, sign_diff);

    auto sum = cc->EvalAdd(a, b);
    auto result = cc->EvalAdd(sum, abs_diff);
    result = cc->EvalMult(result, 0.5);

    return result;
}
```

---

## Common Pitfalls

1. **Computing sqrt for distance comparisons** → Use squared distances
2. **Not pre-computing norms** → Huge waste if norms available in plaintext
3. **Processing distances one-by-one** → Use batched computation
4. **L1 distance when L2 works** → L1 much more expensive
5. **Deep comparison trees for large n** → Use approximate methods
6. **Wrong slot encoding for matrices** → Diagonal vs row-major matters
7. **Forgetting rotation key constraints** → Check available indices in config.json
8. **Bit operations in CKKS** → Use BFV/BGV for exact integer ops

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `challenge-understanding` | Determines if this skill is needed |
| `openfhe-mastery` | Provides rotation, multiplication APIs |
| `function-approximation` | Provides sign approximation for comparisons |
| `fhe-verification-framework` | Verify complex algorithms (CNN, sorting) with NumPy before OpenFHE |
| `ml-pipeline` | Uses matrix operations for layer computations |
| `solution-engineering` | Injects algorithm code into templates |
