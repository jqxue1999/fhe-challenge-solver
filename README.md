# FHE Challenge Solver Plugin

A Claude Code plugin for autonomously solving Fully Homomorphic Encryption (FHE) challenges from the FHERMA benchmark suite.

## Features

- **Autonomous Challenge Solving**: End-to-end solution with iterative refinement
- **Specialized Skills**: Domain expertise for FHE operations, function approximation, encrypted computation, and ML inference
- **Multi-Category Support**: Black-box, white-box OpenFHE, and ML inference challenges
- **OpenFHE Mastery**: Comprehensive C++ and Python API patterns

## Installation

### Local Testing

```bash
# Test the plugin from this directory
claude --plugin-dir /home/jiaq/Research/Code/CC
```

### From GitHub (after publishing)

```bash
# Add from GitHub repository
/plugin marketplace add yourusername/fhe-challenge-solver

# Install the plugin
/plugin install fhe-challenge-solver
```

## Usage

### Solve an FHE Challenge

```bash
# Navigate to challenge directory
cd /path/to/fhe_challenge/black_box/challenge_sign

# Run the autonomous solver
/fhe-challenge-solver:fhe-solve

# Or provide path as argument
/fhe-challenge-solver:fhe-solve /path/to/challenge_directory
```

## Plugin Components

### Commands

- **/fhe-solve** - Autonomously solve an FHE challenge end-to-end

### Agents

- **fhe-challenge-autopilot** - Fully autonomous solver with iterative refinement (model: opus)

### Skills

- **challenge-understanding** - Parse FHE challenge specifications and detect categories
- **function-approximation** - Polynomial/Chebyshev approximation for nonlinear functions
- **encrypted-computation** - Matrix operations, sorting, comparisons, and distance metrics
- **ml-pipeline** - End-to-end ML training and FHE inference implementation
- **openfhe-mastery** - Complete OpenFHE library expertise (CKKS, BFV, BGV)
- **solution-engineering** - Template adaptation, validation workflows, and artifact generation

## Challenge Categories

The plugin handles three types of FHE challenges:

1. **Black-box** (`/black_box/challenge_*`)
   - Docker-based validation
   - CryptoContext loaded from ciphertext

2. **White-box OpenFHE** (`/white_box/openfhe/challenge_*`)
   - verify.sh validation
   - CryptoContext from --cc argument

3. **White-box ML** (`/white_box/ml_inference/challenge_*`)
   - Model training required (R² or Accuracy ≥ 0.85)
   - FHE inference implementation

## Example Workflow

```bash
# The agent automatically:
# 1. Detects challenge category (black-box/white-box/ML)
# 2. Parses requirements from challenge.md
# 3. Selects appropriate algorithm (Chebyshev, matrix ops, etc.)
# 4. Implements solution in template (C++ or Python)
# 5. Validates with docker/verify.sh
# 6. Iterates until success or 10 attempts
# 7. Generates metrics.json with benchmarks
```

## Requirements

- Docker (for black-box challenges)
- OpenFHE library (for local development)
- Python packages: numpy, scikit-learn, pandas (for ML challenges)

## Output Artifacts

All solutions generate:
- `artifacts/metrics.json` - Benchmark metrics (timing, accuracy, depth)
- `artifacts/run.log` - Execution log
- Modified template files with working implementation

## License

MIT

## Contributing

This plugin is designed for the FHERMA benchmark suite. Contributions welcome for:
- Additional approximation methods
- New challenge categories
- Performance optimizations
- Extended ML model support
