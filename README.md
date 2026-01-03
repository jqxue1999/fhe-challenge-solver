# FHE Challenge Solver Plugin

A Claude Code plugin for autonomously solving Fully Homomorphic Encryption (FHE) challenges from the FHERMA benchmark suite.

## Features

- **Autonomous Challenge Solving**: End-to-end solution with iterative refinement
- **Specialized Skills**: Domain expertise for FHE operations, function approximation, encrypted computation, and ML inference
- **Multi-Category Support**: Black-box, white-box OpenFHE, and ML inference challenges
- **OpenFHE Mastery**: Comprehensive C++ and Python API patterns

## Quick Start for New Users

1. **Clone the plugin** to your preferred location:
   ```bash
   cd ~
   git clone https://github.com/jqxue1999/fhe-challenge-solver.git
   ```

2. **Navigate to any FHE challenge** on your machine:
   ```bash
   cd /path/to/your/fhe_challenge/black_box/challenge_sign
   ```

3. **Start Claude Code with the plugin** and run the solver:
   ```bash
   claude --plugin-dir ~/fhe-challenge-solver
   # Then use the command (note: just /fhe-solve without namespace):
   /fhe-solve
   ```

That's it! The plugin will autonomously solve the challenge and generate results in `artifacts/`.

## Installation

### Option 1: Clone and Use with --plugin-dir (Recommended)

Clone the repository to any location on your machine:

```bash
# Clone to your preferred location
cd ~  # or ~/plugins, or anywhere you like
git clone https://github.com/jqxue1999/fhe-challenge-solver.git

# Navigate to your FHE challenge directory
cd /path/to/your/fhe_challenges

# Start Claude Code with the plugin
claude --plugin-dir ~/fhe-challenge-solver

# Your command is now available (use /fhe-solve without namespace)
/fhe-solve ./black_box/challenge_sign
```

**Tip:** Add an alias to your shell config for easier access:
```bash
# Add to ~/.bashrc or ~/.zshrc
alias claude-fhe='claude --plugin-dir ~/fhe-challenge-solver'

# Then use:
claude-fhe
/fhe-solve
```

### Option 2: Download Without Git

```bash
# Download the plugin
mkdir -p ~/claude-plugins
cd ~/claude-plugins
curl -L https://github.com/jqxue1999/fhe-challenge-solver/archive/refs/heads/main.zip -o fhe-challenge-solver.zip
unzip fhe-challenge-solver.zip
mv fhe-challenge-solver-main fhe-challenge-solver

# Use from your project
cd /path/to/your/fhe/challenges
claude --plugin-dir ~/claude-plugins/fhe-challenge-solver
```

### Future: Install via Marketplace

To enable `/plugin install` command, this plugin would need to be added to a Claude Code marketplace. For now, use `--plugin-dir` as shown above. See the [marketplace documentation](https://code.claude.com/docs/en/discover-plugins) for details on creating a marketplace.

## Usage

The plugin works independently from your project files. You can use it in any directory.

### Solve an FHE Challenge

```bash
# Option 1: Navigate to the challenge directory first
cd /path/to/your/fhe_challenges/black_box/challenge_sign
/fhe-solve

# Option 2: Provide the challenge path as an argument from anywhere
cd /path/to/your/workspace
/fhe-solve /path/to/fhe_challenges/black_box/challenge_sign

# Option 3: Use relative paths
cd /path/to/your/fhe_challenges
/fhe-solve ./black_box/challenge_sign
```

### Where Files Are Modified

The plugin modifies files **inside the challenge directory**, not in the plugin directory:
- Edits: `<challenge_dir>/templates/openfhe/yourSolution.cpp` (or `app.py`)
- Creates: `<challenge_dir>/artifacts/metrics.json`
- Creates: `<challenge_dir>/artifacts/run.log`

Your plugin installation and your FHE challenges are completely separate!

**Directory Structure Example:**
```
User's Machine:
├── ~/claude-plugins/fhe-challenge-solver/    ← Plugin installation (read-only)
│   ├── .claude-plugin/
│   ├── agents/
│   ├── skills/
│   └── commands/
│
└── ~/projects/my-fhe-research/                ← User's work directory
    └── fhe_challenges/
        └── black_box/
            └── challenge_sign/                ← Challenge directory (modified)
                ├── templates/
                │   └── openfhe/
                │       └── yourSolution.cpp   ← Plugin edits this
                ├── artifacts/                 ← Plugin creates this
                │   ├── metrics.json
                │   └── run.log
                └── tests/
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
