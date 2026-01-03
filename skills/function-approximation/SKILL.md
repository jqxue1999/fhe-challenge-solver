---
name: function-approximation
description: Approximate any function for homomorphic evaluation including polynomials, rational functions, lookup tables, and piecewise methods. Use when approximating non-linear functions, designing circuits, or optimizing accuracy under depth constraints.
---

# Function Approximation

Polynomial and rational approximation for nonlinear functions under FHE with web search for SOTA methods.

## Skill Responsibility

**Primary Function:** Approximate nonlinear functions for encrypted evaluation

**When to Use:**
- Challenge requires sigmoid, relu, gelu, softmax, sign, tanh
- Challenge requires exp, log, sqrt, inverse
- Description mentions "approximate", "polynomial", "activation"

**DO NOT Use When:**
- Only linear operations (matrix multiplication, rotations)
- Only aggregations (sum, mean)
- No nonlinear function explicitly mentioned

---

## Approximation Methods

### Method 1: Chebyshev Polynomials (Preferred for CKKS)

**Best for:** Smooth functions, wide input ranges, uniform error

```cpp
// OpenFHE Chebyshev evaluation
std::vector<double> coefficients = {c0, c1, c2, ...};  // Chebyshev coefficients
double lowerBound = -8.0;
double upperBound = 8.0;
auto result = cc->EvalChebyshevSeries(inputCt, coefficients, lowerBound, upperBound);
```

**Computing Chebyshev Coefficients:**
```python
import numpy as np
from numpy.polynomial import chebyshev as C

def chebyshev_coefficients(func, lower, upper, degree):
    """Compute Chebyshev coefficients for function approximation."""
    # Sample points (Chebyshev nodes)
    n = degree + 1
    k = np.arange(n)
    nodes = np.cos((2*k + 1) * np.pi / (2*n))  # Nodes in [-1, 1]

    # Map to [lower, upper]
    x = 0.5 * (upper - lower) * nodes + 0.5 * (upper + lower)

    # Function values
    y = func(x)

    # Compute coefficients
    coeffs = C.chebfit(nodes, y, degree)
    return coeffs.tolist()

# Example: Sigmoid
sigmoid = lambda x: 1 / (1 + np.exp(-x))
coeffs = chebyshev_coefficients(sigmoid, -8, 8, degree=7)
```

### Method 2: OpenFHE Built-in Functions

```cpp
// Sigmoid/Logistic (built-in)
auto result = cc->EvalLogistic(inputCt, lowerBound, upperBound, degree);

// General polynomial: p(x) = a0 + a1*x + a2*x^2 + ...
std::vector<double> coeffs = {a0, a1, a2, ...};
auto result = cc->EvalPoly(inputCt, coeffs);

// Chebyshev with custom function
auto result = cc->EvalChebyshevFunction(
    [](double x) { return 1.0 / (1.0 + std::exp(-x)); },  // Function
    inputCt, lowerBound, upperBound, degree
);
```

### Method 3: Paterson-Stockmeyer (Depth Optimization)

**Use when:** Depth budget is tight, polynomial degree is high

| Evaluation Method | Depth for Degree d |
|-------------------|-------------------|
| Horner (standard) | d - 1 |
| Paterson-Stockmeyer | ~2√d |

**Example:** Degree 16 polynomial
- Horner: depth 15
- Paterson-Stockmeyer: depth ~8

OpenFHE often uses Paterson-Stockmeyer internally for `EvalPoly`.

### Method 4: Rational Approximation (Padé)

**Best for:** Functions with steep gradients, better accuracy than polynomials

**Form:** P(x) / Q(x) where P, Q are polynomials

**Depth cost:** Higher than polynomial (requires division approximation)

---

## Function-Specific Approximations

### Sigmoid: σ(x) = 1 / (1 + exp(-x))

**Characteristics:** Range (0, 1), saturates for |x| > 6

**Strategy by depth budget:**

| Depth | Degree | Range | Expected Accuracy |
|-------|--------|-------|-------------------|
| 3-4 | 3-5 | [-4, 4] | 70-85% |
| 5-6 | 5-7 | [-6, 6] | 85-90% |
| 7-10 | 7-13 | [-8, 8] | 90-95% |

```cpp
// OpenFHE sigmoid
auto result = cc->EvalLogistic(inputCt, -8.0, 8.0, 7);
```

**Chebyshev coefficients (degree 7, range [-8, 8]):**
```python
# Approximate values - compute precisely for your range
coeffs = [0.5, 0.197, 0.0, -0.004, 0.0, 0.00026, 0.0, -0.000008]
```

### ReLU: max(0, x)

**Characteristics:** Non-smooth at x=0, unbounded

**Approximation strategies:**

1. **Smooth approximation:**
   ```python
   # SoftPlus: log(1 + exp(x))
   # Scaled sigmoid: x * sigmoid(x)
   ```

2. **Polynomial (for bounded input):**
   ```python
   # For x in [-a, a], approximate with odd polynomial
   # ReLU(x) ≈ 0.5*x + polynomial_approx_of_sign(x) * 0.5*x
   ```

### GELU: x * Φ(x) where Φ is standard normal CDF

**Approximation:**
```python
# GELU ≈ 0.5 * x * (1 + tanh(sqrt(2/π) * (x + 0.044715 * x³)))
# Or polynomial approximation
```

### Sign: sign(x)

**Characteristics:** Discontinuous at x=0

**Approximation strategies:**

1. **High-degree odd polynomial:**
   ```python
   # sign(x) ≈ a1*x + a3*x³ + a5*x⁵ + ...
   # Needs high degree for sharp transition
   ```

2. **Iterated approximation:**
   ```python
   # Start with rough approximation, refine iteratively
   # f(x) → f(f(x)) → f(f(f(x))) ...
   ```

### Softmax: softmax(x_i) = exp(x_i) / Σ exp(x_j)

**Components needed:**
1. Exponential approximation for each slot
2. Sum across slots (rotation + addition)
3. Inverse approximation (1/sum)
4. Final multiplication

**Depth:** High (10-15+ levels typically)

### Max/Argmax

**Approximation via comparison:**
```python
# max(a, b) = 0.5 * (a + b + |a - b|)
# |x| ≈ x * sign(x)
# sign(x) approximated by polynomial
```

---

## Depth Budget Management

### Depth-Degree Trade-off Table

| Depth Budget | Max Safe Degree | Strategy |
|--------------|-----------------|----------|
| 3 | 3-4 | Aggressive clipping, low degree |
| 5 | 5-7 | Moderate clipping |
| 7 | 7-13 | Standard Chebyshev |
| 10+ | 15+ | Paterson-Stockmeyer |
| 15+ | 30+ | Multi-stage or bootstrapping |

### Input Range Clipping

```python
# Soft clipping (smooth, uses depth)
clipped = lower + (upper - lower) * sigmoid((x - lower) / scale)

# Hard clipping (discontinuous but cheap)
# Just document that inputs outside range may have errors
```

**Typical ranges:**
- Sigmoid: [-8, 8] or [-6, 6] for tighter depth
- ReLU: Depends on expected input distribution
- Softmax: Log-space computation helps with range

---

## Web Search for SOTA Methods

### When to Search

- Need optimal Chebyshev coefficients for specific function
- Looking for depth-efficient evaluation methods
- Want to find published approximation parameters
- Need to understand new activation functions

### Search Queries

```
# Optimal polynomial approximation
site:arxiv.org CKKS sigmoid polynomial approximation
site:scholar.google.com homomorphic sigmoid Chebyshev

# Specific functions
site:arxiv.org FHE GELU approximation
site:arxiv.org homomorphic ReLU approximation depth

# Depth optimization
site:arxiv.org Paterson-Stockmeyer FHE
site:scholar.google.com "multiplicative depth" polynomial evaluation

# OpenFHE specific
site:github.com/openfheorg EvalChebyshevSeries
site:openfhe.org polynomial evaluation
```

### Key Papers to Search

```
# Classic references
"Chimera: Combining Ring-LWE-based" (hybrid FHE)
"Bootstrapping for Approximate Homomorphic Encryption"
"PEGASUS: Bridging Polynomial and Non-polynomial Evaluations"

# Activation functions
"CryptoNets" (neural network inference)
"GAZELLE" (efficient garbled circuits + FHE)
"Faster CryptoNets" (depth optimization)
```

### Resources

| Resource | Use For |
|----------|---------|
| arxiv.org | Latest research papers |
| scholar.google.com | Academic papers with citations |
| github.com/openfheorg | OpenFHE examples and issues |
| eprint.iacr.org | Cryptography preprints |

---

## Accuracy Testing

### Test on Plaintext First

```python
import numpy as np

def test_approximation(func, approx_func, test_range, n_points=1000):
    """Test approximation accuracy before encryption."""
    x = np.linspace(test_range[0], test_range[1], n_points)
    y_true = func(x)
    y_approx = approx_func(x)

    max_error = np.max(np.abs(y_true - y_approx))
    mean_error = np.mean(np.abs(y_true - y_approx))
    accuracy = np.mean(np.abs(y_true - y_approx) < 0.1)  # Within 0.1

    print(f"Max error: {max_error:.6f}")
    print(f"Mean error: {mean_error:.6f}")
    print(f"Accuracy (within 0.1): {accuracy*100:.1f}%")

    return max_error, mean_error, accuracy
```

### FHERMA Accuracy Criteria

Most FHERMA challenges use:
- **Point-wise threshold:** Each output slot within tolerance
- **Significant slots:** Only first N slots checked
- **Typical threshold:** 80-90% accuracy required

---

## Implementation Patterns

### Pattern 1: Simple Polynomial (Most Common)

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc, Ciphertext<DCRTPoly> input) {
    // Chebyshev coefficients for sigmoid on [-8, 8], degree 7
    std::vector<double> coeffs = {0.5, 0.197, 0.0, -0.004, 0.0, 0.00026, 0.0, -0.000008};

    return cc->EvalChebyshevSeries(input, coeffs, -8.0, 8.0);
}
```

### Pattern 2: Built-in Logistic

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc, Ciphertext<DCRTPoly> input) {
    return cc->EvalLogistic(input, -8.0, 8.0, 7);
}
```

### Pattern 3: Custom Polynomial with Range Adjustment

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc, Ciphertext<DCRTPoly> input) {
    // Scale input from [-25, 25] to [-1, 1]
    auto scaled = cc->EvalMult(input, 1.0/25.0);

    // Evaluate polynomial on [-1, 1]
    std::vector<double> coeffs = {...};  // Computed for [-1, 1]
    auto result = cc->EvalPoly(scaled, coeffs);

    return result;
}
```

---

## Common Pitfalls

1. **Ignoring input range:** Approximation accurate on [-1, 1] may fail on [-25, 25]
2. **Not clipping:** Inputs outside approximation range cause large errors
3. **Wrong evaluation order:** Horner vs Paterson-Stockmeyer affects depth
4. **Forgetting rescale:** CKKS requires rescaling after multiplications
5. **Over-engineering:** Sometimes degree 3 polynomial is sufficient

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `challenge-understanding` | Determines if this skill is needed |
| `openfhe-mastery` | Provides `EvalChebyshevSeries`, `EvalLogistic` APIs |
| `ml-pipeline` | Uses activation approximations for neural network inference |
| `solution-engineering` | Injects approximation code into templates |
