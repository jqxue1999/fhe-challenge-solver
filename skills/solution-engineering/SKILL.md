---
name: solution-engineering
description: End-to-end solution creation - template adaptation, validation workflows (Docker/verify.sh), and artifact generation. Handles all engineering from template to validated solution.
---

# Solution Engineering

Template adaptation, validation workflows, and artifact generation.

## Skill Responsibility

**Primary Function:** Transform algorithm implementation into working, validated solution

**Inputs:**
- Challenge analysis from `challenge-understanding`
- Algorithm code from `function-approximation`, `encrypted-computation`, or `ml-pipeline`
- OpenFHE patterns from `openfhe-mastery`

**Outputs:**
- Adapted template code
- Successful build
- Validation results
- Solution artifacts

---

## Template Discovery

### By Challenge Category

```python
def find_template(challenge_dir, category):
    """Find the appropriate template for the challenge."""
    if category == "black_box":
        return f"{challenge_dir}/templates/openfhe/"

    elif category == "white_box_ml":
        # Check for Python first
        if exists(f"{challenge_dir}/templates/openfhe-python/"):
            return f"{challenge_dir}/templates/openfhe-python/"
        return f"{challenge_dir}/templates/openfhe/"

    elif category == "white_box_openfhe":
        return f"{challenge_dir}/templates/openfhe/"

    return None
```

### Template Files

**Black-Box (C++):**
```
templates/openfhe/
├── yourSolution.cpp    # IMPLEMENT: eval() function
├── yourSolution.h      # Header file
├── main.cpp            # CLI wrapper (don't modify)
└── CMakeLists.txt      # Build configuration
```

**White-Box OpenFHE (C++):**
```
templates/openfhe/
├── yourSolution.cpp    # IMPLEMENT: eval() function
├── yourSolution.h
├── main.cpp
├── CMakeLists.txt
└── config.json         # Crypto parameters
```

**White-Box ML (Python):**
```
templates/openfhe-python/
├── app.py             # IMPLEMENT: solve() function
└── config.json
```

---

## Template Adaptation

### Black-Box Pattern (yourSolution.cpp)

```cpp
#include "yourSolution.h"
#include "openfhe.h"

using namespace lbcrypto;

Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> ct) {
    // Enable required features
    cc->Enable(PKESchemeFeature::PKE);
    cc->Enable(PKESchemeFeature::KEYSWITCH);
    cc->Enable(PKESchemeFeature::LEVELEDSHE);
    cc->Enable(PKESchemeFeature::ADVANCEDSHE);

    // YOUR IMPLEMENTATION HERE
    // Example: Sigmoid approximation
    auto result = cc->EvalLogistic(ct, -8.0, 8.0, 7);

    return result;
}
```

### White-Box Pattern (yourSolution.cpp)

```cpp
#include "yourSolution.h"
#include "openfhe.h"

using namespace lbcrypto;

Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> ct_array) {
    // Features should already be enabled by main.cpp
    // But safe to enable again
    cc->Enable(PKESchemeFeature::ADVANCEDSHE);

    // YOUR IMPLEMENTATION HERE
    // Example: Find max in array
    // ...

    return result;
}
```

### Python Pattern (app.py solve function)

```python
from openfhe import *

def solve(cc, ct_sample, key_pub, key_mult, key_rot):
    """
    FHE inference function.

    Args:
        cc: CryptoContext
        ct_sample: Encrypted input ciphertext
        key_pub: Public key
        key_mult: Multiplication evaluation key
        key_rot: Rotation evaluation key

    Returns:
        Ciphertext with result
    """
    # Enable features
    cc.Enable(PKESchemeFeature.PKE)
    cc.Enable(PKESchemeFeature.KEYSWITCH)
    cc.Enable(PKESchemeFeature.LEVELEDSHE)
    cc.Enable(PKESchemeFeature.ADVANCEDSHE)

    # YOUR IMPLEMENTATION HERE
    # Example: Linear regression
    weights = [0.1, 0.2, ...]  # Trained weights
    bias = 0.5

    pt_weights = cc.MakeCKKSPackedPlaintext(weights)
    result = cc.EvalMult(ct_sample, pt_weights)
    result = cc.EvalSum(result, len(weights))

    pt_bias = cc.MakeCKKSPackedPlaintext([bias])
    result = cc.EvalAdd(result, pt_bias)

    return result
```

---

## Validation Workflows

### Black-Box Validation

```bash
# Build and run with Docker
cd challenge_directory

# Build Docker image
docker build -t challenge_name .

# Run for each testcase
for tc in tests/testcase*; do
    docker run --rm -v $(pwd)/$tc:/data challenge_name
done

# Check output
ls tests/testcase1/output.txt
```

### White-Box Validation

```bash
# Run verify.sh (uses fherma-validator)
cd challenge_directory
./verify.sh

# Or build-only mode for iteration
./verify.sh --build-only
```

### Understanding verify.sh

```bash
#!/bin/bash
# Actual verify.sh workflow

# 1. Copy template files to app_build/
APP_DIR="$SCRIPT_DIR/app_build"
rm -rf "$APP_DIR"
mkdir -p "$APP_DIR"
cp "$SCRIPT_DIR/templates/openfhe/CMakeLists.txt" "$APP_DIR/"
cp "$SCRIPT_DIR/templates/openfhe/main.cpp" "$APP_DIR/"
cp "$SCRIPT_DIR/templates/openfhe/yourSolution.h" "$APP_DIR/"
cp "$SCRIPT_DIR/templates/openfhe/yourSolution.cpp" "$APP_DIR/"
cp "$SCRIPT_DIR/templates/openfhe/config.json" "$APP_DIR/"

# 2. Run fherma-validator (builds and tests inside container)
docker run -t \
    -v "$SCRIPT_DIR:/fherma" \
    yashalabinc/fherma-validator \
    --project-folder=/fherma/app_build \
    --testcase=/fherma/tests/test_case.json
```

**Important:** Edit files in `templates/openfhe/yourSolution.cpp` (C++) or `templates/openfhe-python/app.py` (Python), then run `./verify.sh`. The script copies templates to `app_build/` before validation.

**Note:** For Python templates, verify.sh copies `app.py` and `config.json` instead of C++ files.

---

## Error Recovery

### Common Errors and Fixes

| Error | Symptom | Fix |
|-------|---------|-----|
| CryptoContext mismatch | "different crypto context" | Load ciphertext first for black-box |
| ADVANCEDSHE not enabled | "operation not enabled" | Add `cc->Enable(ADVANCEDSHE)` |
| Depth exceeded | "level is negative" | Reduce polynomial degree |
| CMake OpenFHE not found | "Could not find OpenFHE" | Use `OpenFHE_*` not `OPENFHE_*` |
| Rotation key missing | "key not found for index" | Check config.json rotation indices |

### Recovery Pattern

```python
def recover_from_error(error_log, solution_code):
    """Apply automatic fixes for common errors."""

    if "crypto context" in error_log.lower():
        return fix_crypto_context_loading(solution_code)

    if "operation has not been enabled" in error_log:
        return add_feature_enabling(solution_code)

    if "level is negative" in error_log:
        return reduce_polynomial_degree(solution_code)

    if "Could not find OpenFHE" in error_log:
        return fix_cmake_variables(solution_code)

    return None  # Unknown error
```

---

## config.json Parameters

```json
{
  "indexes_for_rotation_key": [1, 2, 4, 8, 16, 32],
  "mult_depth": 29,
  "ring_dimension": 131072,
  "scale_mod_size": 59,
  "first_mod_size": 60,
  "batch_size": 65536,
  "enable_bootstrapping": false,
  "levels_available_after_bootstrap": 10,
  "level_budget": [4, 4]
}
```

**Key Parameters:**
- `mult_depth`: Maximum multiplications before noise overflow
- `ring_dimension`: Affects security and slot count
- `indexes_for_rotation_key`: Available rotation indices
- `enable_bootstrapping`: If true, can refresh ciphertext

---

## Artifact Generation

### Output Structure

```
artifacts/
├── output_tc1.bin        # Output for testcase 1
├── output_tc2.bin        # Output for testcase 2 (if exists)
├── metrics.json          # Benchmark metrics
├── run.log              # Execution log
└── best/                # Preserved best solution
```

### Metrics Collection

```json
{
  "timestamp": "2026-01-02T12:00:00Z",
  "challenge": "challenge_sigmoid",
  "category": "black_box",

  "build": {
    "success": true,
    "time_seconds": 45.2,
    "errors": []
  },

  "execution": {
    "success": true,
    "time_seconds": 3.8,
    "testcases_passed": 2,
    "testcases_total": 2
  },

  "solution": {
    "language": "C++",
    "scheme": "CKKS",
    "method": "EvalLogistic",
    "degree": 7,
    "range": [-8, 8]
  },

  "validation": {
    "depth_used": 7,
    "depth_budget": 7,
    "accuracy": 0.92,
    "threshold": 0.80,
    "passed": true
  }
}
```

---

## Iteration Strategy

### Full Workflow

```python
def engineer_solution(challenge_dir, category, implementation):
    """Main solution engineering workflow."""

    # 1. Find and read template
    template_dir = find_template(challenge_dir, category)
    template_code = read_template(template_dir)

    # 2. Inject implementation
    solution_code = inject_implementation(template_code, implementation)

    # 3. Build
    if category == "black_box":
        build_result = docker_build(challenge_dir)
    else:
        build_result = cmake_build(challenge_dir)

    # 4. Handle build errors
    if not build_result.success:
        fixed_code = recover_from_error(build_result.error_log, solution_code)
        if fixed_code:
            write_solution(template_dir, fixed_code)
            build_result = rebuild()

    # 5. Validate
    if category == "black_box":
        validation = run_docker_validation(challenge_dir)
    else:
        validation = run_verify_sh(challenge_dir)

    # 6. Collect metrics
    metrics = collect_metrics(build_result, validation)

    # 7. Generate artifacts
    generate_artifacts(challenge_dir, metrics)

    return validation.passed
```

### Iteration Loop

```python
max_iterations = 5
for iteration in range(max_iterations):
    success = engineer_solution(challenge_dir, category, implementation)

    if success:
        print(f"Solution passed on iteration {iteration + 1}")
        break

    # Adjust implementation based on errors
    implementation = adjust_implementation(implementation, validation.errors)
```

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `challenge-understanding` | Provides category, constraints, CLI arguments |
| `openfhe-mastery` | Provides API patterns, error fixes |
| `function-approximation` | Provides polynomial implementation code |
| `encrypted-computation` | Provides algorithm implementation code |
| `ml-pipeline` | Provides trained weights and inference code |
