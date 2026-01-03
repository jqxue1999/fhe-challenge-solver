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
| `ml-pipeline` | Uses matrix operations for layer computations |
| `solution-engineering` | Injects algorithm code into templates |
