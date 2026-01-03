# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an FHE (Fully Homomorphic Encryption) challenge solving framework using Claude Code agents, skills, and commands to autonomously solve cryptographic challenges from the FHERMA benchmark suite.

## High-Level Architecture

### Three-Layer System

1. **Agents** (`/agents/`) - Autonomous workflows that orchestrate skills
   - `fhe-challenge-autopilot` - Main autonomous solver with iterative refinement

2. **Skills** (`/skills/`) - Specialized domain knowledge modules
   - `challenge-understanding` - Parse requirements, detect category, route workflows
   - `function-approximation` - Polynomial/Chebyshev approximation for nonlinear functions
   - `encrypted-computation` - Matrix ops, sorting, distance metrics, comparisons
   - `ml-pipeline` - End-to-end ML training and FHE inference
   - `openfhe-mastery` - OpenFHE C++/Python API patterns and serialization
   - `solution-engineering` - Template adaptation, build, validation, artifacts

3. **Commands** (`/commands/`) - User-facing slash commands
   - `/fhe-solve` - Entry point to solve an FHE challenge

### Challenge Categories

The system handles three challenge types (detected from path patterns):

- **Black-box** (`/black_box/challenge_*`) - Docker-based validation, CryptoContext from ciphertext
- **White-box OpenFHE** (`/white_box/openfhe/challenge_*`) - verify.sh validation, CryptoContext from --cc arg
- **White-box ML** (`/white_box/ml_inference/challenge_*`) - Requires training model first, then FHE inference

### Workflow Pattern

```
User → /fhe-solve → fhe-challenge-autopilot agent → Skills (challenge-understanding →
function-approximation/encrypted-computation/ml-pipeline → openfhe-mastery →
solution-engineering) → Iterative validation until success
```

## Common Commands

### Running Challenges

```bash
# Navigate to challenge directory first
cd /path/to/fhe_challenge/black_box/challenge_sign
# Then invoke the slash command (Claude Code will expand it)
/fhe-solve

# Or provide path as argument
/fhe-solve /path/to/challenge_directory
```

### Validation Commands

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

## Critical Implementation Patterns

### CryptoContext Loading (MOST COMMON ERROR)

**Black-box (load from ciphertext):**
```cpp
// CORRECT: Load ciphertext FIRST, extract CryptoContext from it
Ciphertext<DCRTPoly> inputCt;
Serial::DeserializeFromFile("/data/input.txt", inputCt, SerType::BINARY);
CryptoContext<DCRTPoly> cc = inputCt->GetCryptoContext();  // Same instance!

// WRONG: Loading separate context causes mismatch errors
CryptoContext<DCRTPoly> cc;
Serial::DeserializeFromFile("cc.json", cc, SerType::JSON);  // Different instance!
```

**White-box (load from --cc argument):**
```cpp
CryptoContext<DCRTPoly> cc;
Serial::DeserializeFromFile(ccLocation, cc, SerType::BINARY);
```

### Template File Locations

Never generate code from scratch - always modify existing templates:

- **C++ templates:** `templates/openfhe/yourSolution.cpp` (implement `eval()` function)
- **Python templates:** `templates/openfhe-python/app.py` (implement `solve()` function)
- **Build config:** `templates/openfhe/config.json` (depth budget, rotation indices)

### Depth Budget Management

All challenges have multiplication depth constraints. Check `config.json`:

```json
{"mult_depth": 29}  // Maximum depth available
```

Common depth costs:
- Linear operations (add, rotate): 0 depth
- Multiplication: 1 depth
- Polynomial degree d: ~d depth (or ~2√d with Paterson-Stockmeyer)
- Chebyshev degree 7: ~7 depth

### Function Approximation Strategy

For nonlinear functions (sigmoid, relu, sign, gelu):

1. Use `EvalLogistic` for sigmoid: `cc->EvalLogistic(ct, -8.0, 8.0, 7)`
2. Use `EvalChebyshevSeries` for custom approximations
3. Adjust degree vs range trade-off based on depth budget
4. Test accuracy on plaintext before encrypting

### ML Challenge Workflow

For challenges with `data/` folder:

1. Load training data from `X_train.csv`, `y_train.csv`
2. Train model until R² ≥ 0.85 (regression) or Accuracy ≥ 0.85 (classification)
3. Extract weights accounting for StandardScaler transformation
4. Implement FHE inference as dot product + bias for linear models

## Skill Usage Guidelines

- **Always start with `challenge-understanding`** to detect category and parse requirements
- Use `function-approximation` for sigmoid, relu, gelu, sign, tanh, exp, log, sqrt
- Use `encrypted-computation` for matrix ops, sorting, max, argmax, KNN, distances
- Use `ml-pipeline` when `data/` folder exists
- Always use `openfhe-mastery` for API patterns
- Always use `solution-engineering` for final template adaptation and validation

## Web Search Integration

Skills can search online for:
- Latest research papers: `site:arxiv.org CKKS sigmoid approximation`
- OpenFHE API docs: `site:openfhe.org EvalChebyshevSeries`
- Examples: `site:github.com/openfheorg matrix multiplication example`
- Error solutions: `site:github.com/openfheorg/openfhe-development/issues "crypto context mismatch"`

## Autonomy Model

The `fhe-challenge-autopilot` agent operates **fully autonomously**:
- Never asks for permission
- Iterates until success or 10 attempts
- Makes all implementation decisions
- Reports only final results

## Output Artifacts

All solutions generate:
- `artifacts/metrics.json` - Benchmark metrics (timing, accuracy, depth used)
- `artifacts/run.log` - Execution log
- Modified template files with working implementation

## Common Pitfalls

1. **CryptoContext mismatch** - Load ciphertext first for black-box, use `GetCryptoContext()`
2. **Missing ADVANCEDSHE** - Enable before polynomial operations: `cc->Enable(PKESchemeFeature::ADVANCEDSHE)`
3. **Depth exceeded** - Reduce polynomial degree or narrow input range
4. **Computing sqrt for distances** - Use squared distances (ordering preserved)
5. **Processing distances sequentially** - Use batched computation with SIMD slots
6. **Wrong CMake variables** - Use `OpenFHE_SHARED_LIBRARIES` not `OPENFHE_LIBRARIES`
