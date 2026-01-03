---
name: openfhe-mastery
description: Complete OpenFHE library expertise covering CKKS, BFV, BGV, and boolean schemes with serialization, key management, and advanced operations. Use when implementing any FHE solution in OpenFHE.
---

# OpenFHE Mastery

OpenFHE C++ and Python API patterns with web search for documentation and examples.

## Skill Responsibility

**Primary Function:** Provide correct OpenFHE API usage for all challenge types

**Key Capabilities:**
- Serialization patterns (black-box vs white-box)
- Key management and feature enabling
- CKKS operations (polynomial eval, rotations, bootstrapping)
- CMake configuration
- Common pitfalls and error recovery
- Web search for API documentation

---

## Critical Pattern: CryptoContext Loading

### Black-Box Challenges (Load from Ciphertext)

**This is the #1 most common issue: CryptoContext mismatch errors!**

```cpp
// CORRECT: Load ciphertext FIRST, extract CryptoContext from it
#include "openfhe.h"
using namespace lbcrypto;

int main() {
    // 1. Load ciphertext first - it brings its CryptoContext
    Ciphertext<DCRTPoly> inputCt;
    Serial::DeserializeFromFile("/data/input.txt", inputCt, SerType::BINARY);

    // 2. Get CryptoContext from ciphertext (SAME instance!)
    CryptoContext<DCRTPoly> cc = inputCt->GetCryptoContext();

    // 3. Enable required features
    cc->Enable(PKESchemeFeature::PKE);
    cc->Enable(PKESchemeFeature::KEYSWITCH);
    cc->Enable(PKESchemeFeature::LEVELEDSHE);
    cc->Enable(PKESchemeFeature::ADVANCEDSHE);

    // 4. Load keys into the SAME context
    std::ifstream multKeyFile("/data/key_mult.txt", std::ios::binary);
    cc->DeserializeEvalMultKey(multKeyFile, SerType::BINARY);
    multKeyFile.close();

    std::ifstream rotKeyFile("/data/key_rot.txt", std::ios::binary);
    cc->DeserializeEvalAutomorphismKey(rotKeyFile, SerType::BINARY);
    rotKeyFile.close();

    // 5. Perform computation - context GUARANTEED to match
    auto result = eval(cc, inputCt);

    // 6. Save output
    Serial::SerializeToFile("/data/output.txt", result, SerType::BINARY);

    return 0;
}
```

**Why this works:** The ciphertext contains a reference to its original CryptoContext. Using `GetCryptoContext()` returns that exact instance, so keys and operations are guaranteed compatible.

**DON'T DO THIS (Common Mistake):**
```cpp
// WRONG: Loads a separate CryptoContext
CryptoContext<DCRTPoly> cc;
Serial::DeserializeFromFile("cc.json", cc, SerType::JSON);  // NEW instance

Ciphertext<DCRTPoly> input;
Serial::DeserializeFromFile("input.txt", input, SerType::BINARY);  // HAS ITS OWN context

auto result = cc->EvalLogistic(input, ...);  // FATAL ERROR: context mismatch!
```

### White-Box Challenges (Load from File)

```cpp
// CORRECT: Load CryptoContext from --cc argument
#include "openfhe.h"
using namespace lbcrypto;

int main(int argc, char* argv[]) {
    std::string ccLocation, inputLocation, outputLocation;
    std::string keyPubLocation, keyMultLocation, keyRotLocation;

    // Parse CLI arguments
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i];
        if (arg == "--cc") ccLocation = argv[++i];
        else if (arg == "--array") inputLocation = argv[++i];
        else if (arg == "--output") outputLocation = argv[++i];
        else if (arg == "--key_pub") keyPubLocation = argv[++i];
        else if (arg == "--key_mult") keyMultLocation = argv[++i];
        else if (arg == "--key_rot") keyRotLocation = argv[++i];
    }

    // 1. Load CryptoContext from file (fherma-validator provides it)
    CryptoContext<DCRTPoly> cc;
    Serial::DeserializeFromFile(ccLocation, cc, SerType::BINARY);

    // 2. Enable features
    cc->Enable(PKESchemeFeature::PKE);
    cc->Enable(PKESchemeFeature::KEYSWITCH);
    cc->Enable(PKESchemeFeature::LEVELEDSHE);
    cc->Enable(PKESchemeFeature::ADVANCEDSHE);

    // 3. Load keys
    std::ifstream multKeyFile(keyMultLocation, std::ios::binary);
    cc->DeserializeEvalMultKey(multKeyFile, SerType::BINARY);
    multKeyFile.close();

    std::ifstream rotKeyFile(keyRotLocation, std::ios::binary);
    cc->DeserializeEvalAutomorphismKey(rotKeyFile, SerType::BINARY);
    rotKeyFile.close();

    // 4. Load encrypted input
    Ciphertext<DCRTPoly> inputCt;
    Serial::DeserializeFromFile(inputLocation, inputCt, SerType::BINARY);

    // 5. Perform computation
    auto result = eval(cc, inputCt);

    // 6. Save output
    Serial::SerializeToFile(outputLocation, result, SerType::BINARY);

    return 0;
}
```

---

## Serialization Types

| Object | SerType | Notes |
|--------|---------|-------|
| CryptoContext | BINARY or JSON | White-box uses BINARY from --cc |
| Ciphertext | BINARY | Always BINARY |
| Public Key | BINARY | Always BINARY |
| Mult/Relin Key | BINARY | Load via stream |
| Rotation Key | BINARY | Load via stream |

**Note:** FHERMA challenges often use `.txt` extension but BINARY format internally.

---

## Python API (openfhe-python)

### Basic Pattern for ML Inference

```python
from openfhe import *

def solve(cc, ct_sample, key_pub, key_mult, key_rot):
    """
    FHE inference function.

    Args:
        cc: CryptoContext
        ct_sample: Encrypted input ciphertext
        key_pub: Public key (may be None)
        key_mult: Multiplication evaluation key
        key_rot: Rotation evaluation key

    Returns:
        Ciphertext with result
    """
    # Enable features (may already be enabled)
    cc.Enable(PKESchemeFeature.PKE)
    cc.Enable(PKESchemeFeature.KEYSWITCH)
    cc.Enable(PKESchemeFeature.LEVELEDSHE)
    cc.Enable(PKESchemeFeature.ADVANCEDSHE)

    # Example: Linear model inference
    # weights and bias should be loaded from trained model
    weights = [0.1, 0.2, 0.3, ...]
    bias = 0.5

    # Create plaintext from weights
    pt_weights = cc.MakeCKKSPackedPlaintext(weights)

    # Dot product: ct_sample * weights
    result = cc.EvalMult(ct_sample, pt_weights)

    # Sum slots to get single value
    result = cc.EvalSum(result, len(weights))

    # Add bias
    pt_bias = cc.MakeCKKSPackedPlaintext([bias])
    result = cc.EvalAdd(result, pt_bias)

    return result
```

### Deserialization in Python

```python
# Load CryptoContext
cc, success = DeserializeCryptoContext(cc_path, SerType.BINARY)

# Load keys
cc.DeserializeEvalMultKey(key_mult_path, SerType.BINARY)
cc.DeserializeEvalAutomorphismKey(key_rot_path, SerType.BINARY)

# Load ciphertext
ct, success = DeserializeCiphertext(input_path, SerType.BINARY)
```

---

## CKKS Operations Reference

### Polynomial Evaluation

```cpp
// Chebyshev approximation (preferred for smooth functions)
std::vector<double> coefficients = {0.5, 0.25, -0.041, 0.0078, ...};
double lowerBound = -8.0;
double upperBound = 8.0;
auto result = cc->EvalChebyshevSeries(inputCt, coefficients, lowerBound, upperBound);

// Built-in logistic/sigmoid
auto result = cc->EvalLogistic(inputCt, -8.0, 8.0, 7);  // degree 7

// General polynomial: p(x) = a0 + a1*x + a2*x^2 + ...
std::vector<double> coeffs = {a0, a1, a2, ...};
auto result = cc->EvalPoly(inputCt, coeffs);
```

### Rotation Operations

```cpp
// Rotate slots (requires rotation keys)
auto rotated = cc->EvalRotate(ct, 1);    // Rotate left by 1
auto rotated = cc->EvalRotate(ct, -1);   // Rotate right by 1
auto rotated = cc->EvalRotate(ct, k);    // Rotate by k positions

// Sum all slots (tree reduction using rotations)
auto sum = cc->EvalSum(ct, batchSize);
```

### Plaintext Operations

```cpp
// Create plaintext from vector
std::vector<double> values = {1.0, 2.0, 3.0};
Plaintext pt = cc->MakeCKKSPackedPlaintext(values);

// Ciphertext-plaintext operations
auto result = cc->EvalAdd(ct, pt);     // ct + pt
auto result = cc->EvalMult(ct, pt);    // ct * pt
auto result = cc->EvalMult(ct, 2.5);   // ct * scalar
```

### Ciphertext-Ciphertext Operations

```cpp
auto sum = cc->EvalAdd(ct1, ct2);      // Addition (free)
auto diff = cc->EvalSub(ct1, ct2);     // Subtraction (free)
auto prod = cc->EvalMult(ct1, ct2);    // Multiplication (consumes 1 level)
auto neg = cc->EvalNegate(ct);         // Negation (free)
auto sq = cc->EvalSquare(ct);          // Square (consumes 1 level)
```

---

## CMake Configuration

### Correct CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(fhe_solution)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find OpenFHE - use mixed-case variable names!
find_package(OpenFHE REQUIRED)

add_executable(solution
    main.cpp
    yourSolution.cpp
)

# CRITICAL: Use OpenFHE_* not OPENFHE_*
target_include_directories(solution PRIVATE
    /usr/local/include
    ${OpenFHE_INCLUDE}
    ${OpenFHE_INCLUDE}/third-party/include
    ${OpenFHE_INCLUDE}/core
    ${OpenFHE_INCLUDE}/pke
)

# Match OpenFHE's build flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenFHE_CXX_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenFHE_EXE_LINKER_FLAGS}")

target_link_libraries(solution ${OpenFHE_SHARED_LIBRARIES})
```

### Common CMake Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Could not find OpenFHE` | Package not installed | Install OpenFHE or check path |
| `OPENFHE_LIBRARIES undefined` | Wrong variable name | Use `OpenFHE_SHARED_LIBRARIES` |
| `openfhe.h not found` | Missing include path | Add `/usr/local/include` |

---

## Common Errors and Fixes

### Error: CryptoContext Mismatch
```
Error: "The key was not generated with this crypto context"
```
**Fix:** Load ciphertext first, use `GetCryptoContext()` (see black-box pattern above)

### Error: ADVANCEDSHE Not Enabled
```
Error: "This operation has not been enabled"
```
**Fix:** Add feature enabling:
```cpp
cc->Enable(PKESchemeFeature::ADVANCEDSHE);
```

### Error: Depth/Level Exceeded
```
Error: "Level is negative" or "Exceeded depth budget"
```
**Fixes:**
1. Reduce polynomial degree
2. Use Paterson-Stockmeyer evaluation
3. Use bootstrapping (if enabled in config.json)

### Error: Scale Mismatch
```
Error: "Scales are not equal"
```
**Fix:** Usually handled automatically, but may need `cc->ModReduce()` in some cases

### Error: Rotation Key Missing
```
Error: "Rotation key not found for index X"
```
**Fix:** Check `config.json` has required rotation indices:
```json
{"indexes_for_rotation_key": [1, 2, 4, 8, 16, ...]}
```

---

## Feature Enabling Reference

Enable features AFTER getting CryptoContext:

```cpp
// For basic operations
cc->Enable(PKESchemeFeature::PKE);        // Encryption/decryption
cc->Enable(PKESchemeFeature::KEYSWITCH);  // Key switching
cc->Enable(PKESchemeFeature::LEVELEDSHE); // Add, mult, rotate

// For polynomial evaluation (EvalChebyshev, EvalLogistic, EvalPoly)
cc->Enable(PKESchemeFeature::ADVANCEDSHE);

// For bootstrapping
cc->Enable(PKESchemeFeature::FHE);
```

---

## Web Search for OpenFHE

### When to Search
- Unknown API function or parameters
- Version-specific behavior (v1.1.1 vs v1.1.4 vs v1.2.x)
- Error messages not covered above
- Advanced operations (bootstrapping, multi-party)

### Search Queries

```
# Official documentation
site:openfhe.org EvalChebyshevSeries
site:openfhe.org CKKS bootstrapping

# GitHub source code
site:github.com/openfheorg openfhe EvalRotate
site:github.com/openfheorg openfhe-development src/pke

# GitHub issues (for errors)
site:github.com/openfheorg/openfhe-development/issues "crypto context mismatch"

# Examples
site:github.com openfhe CKKS example
github openfhe matrix multiplication

# Specific version
site:github.com/openfheorg/openfhe-development/releases v1.1.4
```

### Key Resources

| Resource | URL | Use For |
|----------|-----|---------|
| OpenFHE GitHub | github.com/openfheorg/openfhe-development | Source, issues |
| OpenFHE Docs | openfhe.org/docs | Official docs |
| OpenFHE Python | github.com/openfheorg/openfhe-python | Python bindings |
| OpenFHE Examples | github.com/openfheorg/openfhe-development/tree/main/src/pke/examples | Code examples |

---

## Version Differences

### v1.1.1 → v1.1.4
- Improved CKKS bootstrapping
- Better serialization
- Most APIs unchanged

### v1.1.4 → v1.2.x
- Some API changes
- Check release notes

**Tip:** Search `site:github.com/openfheorg/openfhe-development/releases` for version info

---

## Headers Reference

```cpp
// Main header (includes most things)
#include "openfhe.h"

// Or specific headers
#include "openfhe/pke/openfhe.h"
#include "openfhe/pke/scheme/ckksrns/ckksrns-ser.h"
#include "openfhe/pke/key/key-ser.h"
#include "openfhe/pke/ciphertext-ser.h"
```

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `function-approximation` | Provides Chebyshev coefficients → use `EvalChebyshevSeries` |
| `encrypted-computation` | Provides algorithm → implement with OpenFHE operations |
| `ml-pipeline` | Provides trained weights → implement FHE inference |
| `solution-engineering` | Uses OpenFHE patterns for template adaptation |
