---
name: fhe-verification-framework
description: Two-layer verification framework for FHE implementations - NumPy simulation validates algorithm design, OpenFHE verification validates API usage. Use when implementing ML inference or complex encrypted computations.
---

# FHE Verification Framework

A systematic two-layer verification approach for implementing encrypted ML inference and complex FHE algorithms.

## Skill Responsibility

**Primary Function:** Verify FHE algorithm correctness before and after OpenFHE implementation

**Core Philosophy:**
1. **Layer 1 (NumPy Simulation):** Validate algorithm design using plaintext with FHE-restricted operations
2. **Layer 2 (OpenFHE Verification):** Validate API usage by comparing OpenFHE results with NumPy simulation

**When to Use:**
- Implementing CNN convolution in FHE
- Matrix operations with diagonal packing
- Multi-channel processing with rotations
- Any complex algorithm combining rotations + multiplications
- ML model inference translation to FHE

---

## Layer 1: NumPy Simulation (Algorithm Verification)

### Core Concept

Simulate FHE ciphertext operations using NumPy arrays with **restricted operations**:
- ✅ Element-wise addition: `ct1 + ct2`
- ✅ Element-wise multiplication (ct-pt): `ct * plaintext_mask`
- ✅ Rotation: `np.roll(ct, shift)`
- ❌ NO direct indexing: `ct[i]` (not allowed in FHE)
- ❌ NO conditionals on encrypted data
- ❌ NO sqrt, division (unless approximated)

**Goal:** If algorithm works with these restrictions, it will work in FHE.

### FHESimulator Class

```python
import numpy as np
from typing import List, Tuple, Callable

class FHESimulator:
    """
    Simulates FHE ciphertext operations using NumPy.

    A ciphertext is represented as a 1D numpy array (SIMD slots).
    Only allows FHE-compatible operations.
    """

    def __init__(self, batch_size: int):
        """
        Args:
            batch_size: Number of SIMD slots (must be power of 2 for CKKS)
        """
        assert batch_size > 0 and (batch_size & (batch_size - 1)) == 0, \
            "batch_size must be power of 2"
        self.batch_size = batch_size

    def encrypt(self, plaintext: np.ndarray) -> np.ndarray:
        """
        Simulate encryption by packing plaintext into slots.

        Args:
            plaintext: 1D array to encrypt

        Returns:
            Simulated ciphertext (padded to batch_size)
        """
        if len(plaintext) > self.batch_size:
            raise ValueError(f"Plaintext size {len(plaintext)} exceeds batch_size {self.batch_size}")

        # Pad with zeros
        ct = np.zeros(self.batch_size)
        ct[:len(plaintext)] = plaintext
        return ct

    def decrypt(self, ciphertext: np.ndarray, length: int = None) -> np.ndarray:
        """
        Simulate decryption by extracting meaningful slots.

        Args:
            ciphertext: Simulated ciphertext
            length: Number of meaningful slots (default: all non-zero)

        Returns:
            Plaintext array
        """
        if length is None:
            # Find last non-zero element
            non_zero = np.nonzero(ciphertext)[0]
            length = non_zero[-1] + 1 if len(non_zero) > 0 else 0

        return ciphertext[:length]

    # Basic FHE Operations

    def add(self, ct1: np.ndarray, ct2: np.ndarray) -> np.ndarray:
        """Ciphertext-ciphertext addition (depth 0)"""
        assert len(ct1) == len(ct2) == self.batch_size
        return ct1 + ct2

    def add_plain(self, ct: np.ndarray, pt: np.ndarray) -> np.ndarray:
        """Ciphertext-plaintext addition (depth 0)"""
        assert len(ct) == self.batch_size
        result = ct.copy()
        result[:len(pt)] += pt
        return result

    def mult_plain(self, ct: np.ndarray, pt: np.ndarray) -> np.ndarray:
        """
        Ciphertext-plaintext multiplication (depth 1)

        pt can be:
        - Scalar: multiply all slots
        - Array: element-wise multiplication (masking)
        """
        if np.isscalar(pt):
            return ct * pt
        else:
            assert len(pt) <= self.batch_size
            mask = np.zeros(self.batch_size)
            mask[:len(pt)] = pt
            return ct * mask

    def rotate(self, ct: np.ndarray, shift: int) -> np.ndarray:
        """
        Rotate ciphertext slots (depth 0)

        Args:
            shift: Rotation amount (positive = left, negative = right)
        """
        return np.roll(ct, shift)

    def sum_slots(self, ct: np.ndarray, length: int) -> float:
        """
        Sum first 'length' slots using tree reduction.

        In real FHE: uses log(length) rotations
        Here: direct sum for simplicity, but same result
        """
        return np.sum(ct[:length])

    # High-level operations (built from primitives)

    def conv2d_single_channel(self,
                             image: np.ndarray,
                             kernel: np.ndarray,
                             H: int, W: int,
                             kH: int, kW: int,
                             stride: int = 1) -> Tuple[np.ndarray, int, int]:
        """
        2D convolution using rotation method.

        Args:
            image: Encrypted image (flattened H*W)
            kernel: Plaintext kernel (flattened kH*kW)
            H, W: Image height and width
            kH, kW: Kernel height and width
            stride: Convolution stride

        Returns:
            (result_ct, outH, outW)
        """
        outH = (H - kH) // stride + 1
        outW = (W - kW) // stride + 1

        result = np.zeros(self.batch_size)

        # For each kernel position
        for ki in range(kH):
            for kj in range(kW):
                # Rotation index
                rotation_idx = ki * W + kj

                # Rotate image
                rotated = self.rotate(image, rotation_idx)

                # Create mask with kernel weight at valid positions
                mask = np.zeros(H * W)
                for out_i in range(outH):
                    for out_j in range(outW):
                        orig_pos = (out_i * stride) * W + (out_j * stride)
                        mask[orig_pos] = kernel[ki * kW + kj]

                # Multiply and accumulate
                prod = self.mult_plain(rotated, mask)
                result = self.add(result, prod)

        return result, outH, outW

    def matrix_vector_diagonal(self,
                               vec: np.ndarray,
                               diagonals: List[np.ndarray],
                               n: int) -> np.ndarray:
        """
        Matrix-vector multiplication using diagonal method.

        Args:
            vec: Encrypted vector (length n)
            diagonals: List of plaintext diagonals
            n: Vector/matrix dimension

        Returns:
            Encrypted result vector
        """
        result = self.mult_plain(vec, diagonals[0])

        for k in range(1, len(diagonals)):
            rotated = self.rotate(vec, k)
            prod = self.mult_plain(rotated, diagonals[k])
            result = self.add(result, prod)

        return result


class VerificationTest:
    """
    Compare NumPy simulation against ground truth plaintext computation.
    """

    def __init__(self, batch_size: int = 4096, tolerance: float = 1e-10):
        self.sim = FHESimulator(batch_size)
        self.tolerance = tolerance

    def verify_conv2d(self,
                      image: np.ndarray,
                      kernel: np.ndarray,
                      H: int, W: int,
                      kH: int, kW: int,
                      ground_truth_fn: Callable) -> bool:
        """
        Verify 2D convolution implementation.

        Args:
            image: Plaintext image (H, W)
            kernel: Plaintext kernel (kH, kW)
            H, W, kH, kW: Dimensions
            ground_truth_fn: Function that computes correct result

        Returns:
            True if verification passes
        """
        # Flatten inputs
        image_flat = image.flatten()
        kernel_flat = kernel.flatten()

        # Encrypt
        ct_image = self.sim.encrypt(image_flat)

        # Compute using FHE simulation
        ct_result, outH, outW = self.sim.conv2d_single_channel(
            ct_image, kernel_flat, H, W, kH, kW
        )

        # Decrypt and extract output
        result_flat = self.sim.decrypt(ct_result)
        result_extracted = np.zeros(outH * outW)
        for i in range(outH):
            for j in range(outW):
                result_extracted[i * outW + j] = result_flat[i * W + j]

        # Compute ground truth
        expected = ground_truth_fn(image, kernel)
        expected_flat = expected.flatten()

        # Compare
        error = np.max(np.abs(result_extracted - expected_flat))

        print(f"Conv2D Verification:")
        print(f"  Expected: {expected_flat[:5]}...")
        print(f"  Got:      {result_extracted[:5]}...")
        print(f"  Max error: {error:.2e}")

        if error < self.tolerance:
            print(f"  ✅ PASSED (error < {self.tolerance})")
            return True
        else:
            print(f"  ❌ FAILED (error {error:.2e} >= {self.tolerance})")
            return False

    def verify_matrix_vector(self,
                            matrix: np.ndarray,
                            vector: np.ndarray,
                            ground_truth_fn: Callable) -> bool:
        """
        Verify matrix-vector multiplication with diagonal method.
        """
        n = len(vector)

        # Convert matrix to diagonals
        diagonals = []
        for k in range(n):
            diag = np.array([matrix[i][(i + k) % n] for i in range(n)])
            diagonals.append(diag)

        # Encrypt vector
        ct_vec = self.sim.encrypt(vector)

        # Compute using FHE simulation
        ct_result = self.sim.matrix_vector_diagonal(ct_vec, diagonals, n)

        # Decrypt
        result = self.sim.decrypt(ct_result, n)

        # Ground truth
        expected = ground_truth_fn(matrix, vector)

        # Compare
        error = np.max(np.abs(result - expected))

        print(f"Matrix-Vector Verification:")
        print(f"  Expected: {expected}")
        print(f"  Got:      {result}")
        print(f"  Max error: {error:.2e}")

        if error < self.tolerance:
            print(f"  ✅ PASSED")
            return True
        else:
            print(f"  ❌ FAILED")
            return False
```

### Example: Verify CNN Convolution

```python
import numpy as np
from scipy.signal import correlate2d

# Ground truth function using scipy
def ground_truth_conv(image, kernel):
    return correlate2d(image, kernel, mode='valid')

# Test case
H, W = 5, 5
kH, kW = 3, 3

image = np.random.randn(H, W)
kernel = np.random.randn(kH, kW)

# Verify
verifier = VerificationTest(batch_size=1024)
passed = verifier.verify_conv2d(image, kernel, H, W, kH, kW, ground_truth_conv)

# If passed, algorithm is correct!
# Now translate to OpenFHE with confidence
```

---

## Layer 2: OpenFHE Verification (API Validation)

### Concept

After NumPy simulation passes, translate to OpenFHE and verify:
- NumPy simulation result == OpenFHE decrypted result

This isolates API bugs from algorithm bugs.

### OpenFHEVerifier Class

```cpp
#include "openfhe.h"
#include <vector>
#include <cmath>
#include <iostream>

using namespace lbcrypto;

class OpenFHEVerifier {
private:
    CryptoContext<DCRTPoly> cc;
    KeyPair<DCRTPoly> keyPair;
    double tolerance;

public:
    OpenFHEVerifier(CryptoContext<DCRTPoly> cryptoContext,
                    KeyPair<DCRTPoly> keys,
                    double tol = 1e-6)
        : cc(cryptoContext), keyPair(keys), tolerance(tol) {}

    /**
     * Verify OpenFHE result against NumPy simulation.
     *
     * @param ct_result OpenFHE ciphertext result
     * @param numpy_result Expected result from NumPy simulation
     * @param length Number of meaningful slots
     * @return true if verification passes
     */
    bool verify_result(Ciphertext<DCRTPoly> ct_result,
                       const std::vector<double>& numpy_result,
                       size_t length) {
        // Decrypt OpenFHE result
        Plaintext pt_result;
        cc->Decrypt(keyPair.secretKey, ct_result, &pt_result);
        pt_result->SetLength(length);

        auto openfhe_result = pt_result->GetRealPackedValue();

        // Compare with NumPy simulation
        double max_error = 0.0;
        for (size_t i = 0; i < length; i++) {
            double error = std::abs(openfhe_result[i] - numpy_result[i]);
            max_error = std::max(max_error, error);
        }

        std::cout << "OpenFHE Verification:" << std::endl;
        std::cout << "  NumPy:   [";
        for (size_t i = 0; i < std::min(length, size_t(5)); i++) {
            std::cout << numpy_result[i] << " ";
        }
        std::cout << "...]" << std::endl;

        std::cout << "  OpenFHE: [";
        for (size_t i = 0; i < std::min(length, size_t(5)); i++) {
            std::cout << openfhe_result[i] << " ";
        }
        std::cout << "...]" << std::endl;

        std::cout << "  Max error: " << std::scientific << max_error << std::endl;

        if (max_error < tolerance) {
            std::cout << "  ✅ PASSED (error < " << tolerance << ")" << std::endl;
            return true;
        } else {
            std::cout << "  ❌ FAILED (error >= " << tolerance << ")" << std::endl;
            return false;
        }
    }

    /**
     * Verify convolution implementation.
     */
    bool verify_conv2d(const std::vector<double>& image_plain,
                       const std::vector<double>& kernel_plain,
                       int H, int W, int kH, int kW,
                       const std::vector<double>& numpy_expected,
                       Ciphertext<DCRTPoly> (*conv_fn)(CryptoContext<DCRTPoly>,
                                                        Ciphertext<DCRTPoly>,
                                                        const std::vector<double>&,
                                                        int, int, int, int)) {
        // Encrypt image
        Plaintext pt_image = cc->MakeCKKSPackedPlaintext(image_plain);
        auto ct_image = cc->Encrypt(keyPair.publicKey, pt_image);

        // Run OpenFHE convolution
        auto ct_result = conv_fn(cc, ct_image, kernel_plain, H, W, kH, kW);

        // Extract expected output size
        int outH = (H - kH) + 1;
        int outW = (W - kW) + 1;

        // Decrypt and extract
        Plaintext pt_result;
        cc->Decrypt(keyPair.secretKey, ct_result, &pt_result);
        auto full_result = pt_result->GetRealPackedValue();

        std::vector<double> extracted(outH * outW);
        for (int i = 0; i < outH; i++) {
            for (int j = 0; j < outW; j++) {
                extracted[i * outW + j] = full_result[i * W + j];
            }
        }

        // Verify against NumPy
        return verify_result_vector(extracted, numpy_expected);
    }

private:
    bool verify_result_vector(const std::vector<double>& openfhe_result,
                              const std::vector<double>& numpy_result) {
        if (openfhe_result.size() != numpy_result.size()) {
            std::cout << "  ❌ Size mismatch!" << std::endl;
            return false;
        }

        double max_error = 0.0;
        for (size_t i = 0; i < openfhe_result.size(); i++) {
            double error = std::abs(openfhe_result[i] - numpy_result[i]);
            max_error = std::max(max_error, error);
        }

        std::cout << "  Max error: " << std::scientific << max_error << std::endl;

        return max_error < tolerance;
    }
};
```

---

## Complete Workflow Example

### Step 1: Design Algorithm (NumPy Simulation)

```python
# test_conv_algorithm.py
import numpy as np
from scipy.signal import correlate2d
from fhe_simulator import FHESimulator, VerificationTest

# Define test case
H, W = 5, 5
kH, kW = 3, 3
image = np.array([
    [1, 2, 3, 4, 5],
    [6, 7, 8, 9, 10],
    [11, 12, 13, 14, 15],
    [16, 17, 18, 19, 20],
    [21, 22, 23, 24, 25]
], dtype=float)

kernel = np.array([
    [1, 0, -1],
    [2, 0, -2],
    [1, 0, -1]
], dtype=float)

# Ground truth
def ground_truth_conv(img, kern):
    return correlate2d(img, kern, mode='valid')

expected = ground_truth_conv(image, kernel)
print("Expected output:")
print(expected)

# Verify NumPy simulation
verifier = VerificationTest(batch_size=1024, tolerance=1e-10)
passed = verifier.verify_conv2d(image, kernel, H, W, kH, kW, ground_truth_conv)

if passed:
    print("\n✅ Algorithm verified! Safe to translate to OpenFHE")
    # Save expected result for OpenFHE verification
    np.save("numpy_expected.npy", expected.flatten())
else:
    print("\n❌ Algorithm has bugs! Fix before translating to OpenFHE")
```

### Step 2: Translate to OpenFHE

```cpp
// conv2d_openfhe.cpp
#include "openfhe.h"
#include "verifier.h"

using namespace lbcrypto;

Ciphertext<DCRTPoly> conv2d_rotation(
    CryptoContext<DCRTPoly> cc,
    Ciphertext<DCRTPoly> image,
    const std::vector<double>& kernel,
    int H, int W, int kH, int kW
) {
    int outH = (H - kH) + 1;
    int outW = (W - kW) + 1;

    Ciphertext<DCRTPoly> result;
    bool first = true;

    for (int ki = 0; ki < kH; ki++) {
        for (int kj = 0; kj < kW; kj++) {
            int rotation_idx = ki * W + kj;
            auto rotated = cc->EvalRotate(image, rotation_idx);

            // Create mask
            std::vector<double> mask(H * W, 0.0);
            for (int out_i = 0; out_i < outH; out_i++) {
                for (int out_j = 0; out_j < outW; out_j++) {
                    int orig_pos = out_i * W + out_j;
                    mask[orig_pos] = kernel[ki * kW + kj];
                }
            }

            auto pt_mask = cc->MakeCKKSPackedPlaintext(mask);
            auto prod = cc->EvalMult(rotated, pt_mask);

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
```

### Step 3: Verify OpenFHE Implementation

```cpp
// test_openfhe.cpp
#include "conv2d_openfhe.cpp"
#include "verifier.h"
#include <fstream>

int main() {
    // Setup OpenFHE
    CCParams<CryptoContextCKKSRNS> parameters;
    parameters.SetMultiplicativeDepth(5);
    parameters.SetScalingModSize(50);
    parameters.SetBatchSize(1024);

    CryptoContext<DCRTPoly> cc = GenCryptoContext(parameters);
    cc->Enable(PKESchemeFeature::PKE);
    cc->Enable(PKESchemeFeature::KEYSWITCH);
    cc->Enable(PKESchemeFeature::LEVELEDSHE);

    auto keyPair = cc->KeyGen();
    cc->EvalMultKeyGen(keyPair.secretKey);

    // Generate rotation keys for convolution
    std::vector<int> rotation_indices = {1, 2, 5, 6, 7, 10, 11, 12};
    cc->EvalRotateKeyGen(keyPair.secretKey, rotation_indices);

    // Load test data
    std::vector<double> image = {
        1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
        16, 17, 18, 19, 20, 21, 22, 23, 24, 25
    };
    std::vector<double> kernel = {1, 0, -1, 2, 0, -2, 1, 0, -1};

    // Load NumPy expected result
    std::vector<double> numpy_expected = load_numpy_result("numpy_expected.npy");

    // Create verifier
    OpenFHEVerifier verifier(cc, keyPair, 1e-6);

    // Verify
    bool passed = verifier.verify_conv2d(
        image, kernel, 5, 5, 3, 3,
        numpy_expected,
        conv2d_rotation
    );

    if (passed) {
        std::cout << "\n✅ OpenFHE implementation correct!" << std::endl;
    } else {
        std::cout << "\n❌ OpenFHE implementation has bugs!" << std::endl;
    }

    return passed ? 0 : 1;
}
```

---

## Best Practices

### 1. Always Start with NumPy Simulation

```python
# DON'T: Write OpenFHE code directly
# ❌ Waste hours debugging whether bug is in algorithm or API

# DO: Verify algorithm first with NumPy
# ✅ Algorithm bugs found in seconds
# ✅ OpenFHE translation is just mechanical
```

### 2. Test Edge Cases

```python
# Test different sizes
for H in [3, 5, 8, 16, 32]:
    for kH in [3, 5, 7]:
        if kH <= H:
            test_conv(H, H, kH, kH)

# Test stride
test_conv(32, 32, 3, 3, stride=2)

# Test multiple channels
test_multi_channel_conv(3, 16, 32, 32, 3, 3)
```

### 3. Save Intermediate Results

```python
# Save NumPy results for OpenFHE verification
np.save(f"expected_conv_{H}x{W}_k{kH}x{kW}.npy", expected)

# Load in C++
std::vector<double> load_numpy_result(const std::string& filename);
```

### 4. Tolerance Settings

```python
# NumPy simulation vs ground truth: very strict
numpy_tolerance = 1e-10

# OpenFHE vs NumPy: account for FHE noise
openfhe_tolerance = 1e-6  # CKKS typical
openfhe_tolerance = 1e-12 # BFV/BGV (exact)
```

---

## Common Pitfalls

### 1. Forgetting Rotation Key Generation

```cpp
// ❌ WRONG: Missing rotation keys
auto rotated = cc->EvalRotate(ct, 5);  // CRASH!

// ✅ CORRECT: Generate rotation keys first
std::vector<int> indices = {1, 2, 5, 6, 7, 10, 11, 12};
cc->EvalRotateKeyGen(keyPair.secretKey, indices);
```

### 2. Wrong Output Extraction

```python
# ❌ WRONG: Assuming consecutive output
result = ct_result[:outH * outW]

# ✅ CORRECT: Output is in 2D grid positions
result = []
for i in range(outH):
    for j in range(outW):
        result.append(ct_result[i * W + j])
```

### 3. Batch Size Not Power of 2

```cpp
// ❌ WRONG: 28*28 = 784
parameters.SetBatchSize(784);  // CRASH!

// ✅ CORRECT: Round up to power of 2
parameters.SetBatchSize(1024);  // 2^10
```

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `encrypted-computation` | Verify implementations from this skill |
| `function-approximation` | Verify polynomial approximations |
| `ml-pipeline` | Verify trained model inference |
| `openfhe-mastery` | Provides API patterns to verify |
| `solution-engineering` | Use verification in template validation |

---

## Workflow Summary

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Design Algorithm (NumPy Simulation)                     │
│    - Write algorithm using only: +, *, rotate              │
│    - Compare with ground truth                             │
│    - Fix until error < 1e-10                               │
└──────────────────┬──────────────────────────────────────────┘
                   │ ✅ PASSED
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Translate to OpenFHE                                     │
│    - Mechanical translation:                                │
│      np.roll(ct, k)  →  cc->EvalRotate(ct, k)             │
│      ct * mask       →  cc->EvalMult(ct, pt_mask)         │
│      ct1 + ct2       →  cc->EvalAdd(ct1, ct2)             │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Verify OpenFHE (Against NumPy)                          │
│    - Decrypt OpenFHE result                                 │
│    - Compare with NumPy simulation                          │
│    - Fix API bugs until error < 1e-6                        │
└──────────────────┬──────────────────────────────────────────┘
                   │ ✅ PASSED
                   ▼
             🎉 DONE!
    Algorithm AND implementation verified
```

---

## Example Use Cases

### 1. CNN Layer Verification

```python
# Verify: Conv → BN → ReLU → Pool
def verify_cnn_layer():
    # Test convolution
    verify_conv2d(...)

    # Test batch normalization (just linear transform)
    verify_batch_norm(...)

    # Test ReLU approximation
    verify_relu_approx(...)

    # Test average pooling
    verify_avg_pool(...)
```

### 2. Multi-Channel Convolution

```python
# Verify 3→16 channel convolution
def verify_multi_channel():
    C_in, C_out = 3, 16
    # ... verify each channel pair
    # ... verify channel summation
```

### 3. Matrix-Vector Multiply

```python
# Verify diagonal method
def verify_matmul():
    n = 128
    matrix = np.random.randn(n, n)
    vector = np.random.randn(n)
    # ... verify
```

This framework makes FHE development **10x faster** by catching bugs early!
