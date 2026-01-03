# Solve FHE Challenge

Autonomously solve an FHE challenge end-to-end with iterative improvement until success.

## Usage

```bash
# Navigate to challenge directory first
cd /home/yifei/data/cipherbench/aideml/fhe_challenge/black_box/challenge_sign
/fhe-challenge-autopilot:fhe-solve

# Or provide path as argument
/fhe-challenge-autopilot:fhe-solve /path/to/challenge_directory
```

## What It Does

### Phase 1: Challenge Analysis

1. **Detect category** from path pattern:
   - `*/black_box/challenge_*` → Black-box (Docker validation)
   - `*/white_box/openfhe/challenge_*` → White-box OpenFHE (verify.sh)
   - `*/white_box/ml_inference/challenge_*` → White-box ML (training + verify.sh)
   - `*/white_box/non-OpenFHE/*` → Skip (not supported)

2. **Parse challenge.md** for requirements:
   - Task description (sigmoid, max, sorting, regression, etc.)
   - Constraints (depth budget, accuracy threshold, input range)
   - Scheme (CKKS, BFV, BGV)

3. **Determine skills needed**:
   - `function-approximation` for sigmoid, relu, gelu, softmax, sign, exp, log
   - `encrypted-computation` for matrix ops, sorting, comparison, KNN, bit ops
   - `ml-pipeline` for challenges with training data

### Phase 2: Implementation

1. **Adapt template** from `templates/openfhe/` or `templates/openfhe-python/`
   - Edit `yourSolution.cpp` (C++) or `app.py` (Python)
   - Never generate from scratch - always modify existing template

2. **For ML challenges** (when `data/` folder exists):
   - Load and analyze training data
   - Train model to R² ≥ 0.85 (regression) or Accuracy ≥ 0.85 (classification)
   - Extract weights for FHE inference
   - Implement FHE inference with trained weights

3. **Apply correct CryptoContext pattern**:
   - Black-box: Load from ciphertext using `GetCryptoContext()`
   - White-box: Load from `--cc` CLI argument

### Phase 3: Validation

**Black-box challenges:**
```bash
cd <challenge_dir>
docker build -t challenge .
docker run --rm -v $(pwd)/tests/testcase1:/data challenge
```

**White-box challenges:**
```bash
cd <challenge_dir>
./verify.sh
```

### Phase 4: Iteration

If validation fails:
1. Parse error messages
2. Apply fixes (depth reduction, feature enabling, parameter adjustment)
3. Re-validate
4. Repeat until success

## Outputs

All solutions generate:
- `artifacts/metrics.json` - Benchmark metrics
- `artifacts/run.log` - Execution log
- `templates/openfhe/yourSolution.cpp` or `templates/openfhe-python/app.py` - Modified solution

## Metrics Schema

`artifacts/metrics.json`:
```json
{
  "timestamp": "2026-01-02T12:00:00Z",
  "challenge": "challenge_sign",
  "category": "black_box",
  "build": {
    "success": true,
    "time_seconds": 45.2
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
    "method": "EvalChebyshevSeries",
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

## Success Criteria

A solution is successful when:
1. Build completes without errors
2. Validation passes (Docker output or verify.sh)
3. Accuracy threshold met (if applicable)
4. All testcases processed

## Example Workflow

```bash
# Black-box challenge
cd /home/yifei/data/cipherbench/aideml/fhe_challenge/black_box/challenge_sign
/fhe-challenge-autopilot:fhe-solve

# White-box OpenFHE challenge
cd /home/yifei/data/cipherbench/aideml/fhe_challenge/white_box/openfhe/challenge_max
/fhe-challenge-autopilot:fhe-solve

# White-box ML challenge
cd /home/yifei/data/cipherbench/aideml/fhe_challenge/white_box/ml_inference/challenge_house_prediction
/fhe-challenge-autopilot:fhe-solve
```
