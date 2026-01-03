# Usage Examples

Quick examples of using the FHE Challenge Solver plugin in different scenarios.

## Initial Setup (One Time)

```bash
# Clone the plugin to your home directory
cd ~
git clone https://github.com/jqxue1999/fhe-challenge-solver.git
```

## Example 1: Black-Box Challenge (Docker-based)

```bash
# You have challenges at: ~/research/fhe_challenges/
cd ~/research/fhe_challenges/black_box/challenge_sign

# Start Claude Code with the plugin
claude --plugin-dir ~/fhe-challenge-solver

# Run the solver (autonomous, no user input needed)
/fhe-solve

# Output will be in:
# - artifacts/metrics.json
# - artifacts/run.log
# - templates/openfhe/yourSolution.cpp (modified)
```

## Example 2: White-Box OpenFHE Challenge

```bash
cd ~/research/fhe_challenges/white_box/openfhe/challenge_max

claude --plugin-dir ~/fhe-challenge-solver

/fhe-solve
```

## Example 3: White-Box ML Challenge

```bash
cd ~/research/fhe_challenges/white_box/ml_inference/challenge_house_prediction

claude --plugin-dir ~/fhe-challenge-solver

# The plugin will:
# 1. Load training data from data/X_train.csv, y_train.csv
# 2. Train model until R² ≥ 0.85
# 3. Extract weights
# 4. Implement FHE inference in templates/openfhe-python/app.py
# 5. Run ./verify.sh to validate

/fhe-solve
```

## Example 4: Specify Challenge Path from Different Directory

```bash
# You're working in a different directory
cd ~/workspace

# Start Claude Code with plugin
claude --plugin-dir ~/fhe-challenge-solver

# Provide full path to challenge
/fhe-solve ~/research/fhe_challenges/black_box/challenge_sign

# Or relative path
/fhe-solve ../research/fhe_challenges/black_box/challenge_sign
```

## Example 5: Using Shell Alias (Recommended for Frequent Use)

```bash
# Add to ~/.bashrc or ~/.zshrc (one time)
echo "alias claude-fhe='claude --plugin-dir ~/fhe-challenge-solver'" >> ~/.bashrc
source ~/.bashrc

# Now you can simply type:
cd ~/research/fhe_challenges/black_box/challenge_sigmoid
claude-fhe
/fhe-solve
```

## Example 6: Batch Processing Multiple Challenges

```bash
# Create a script to process multiple challenges
cat > solve_all.sh << 'EOF'
#!/bin/bash
CHALLENGES=(
  "black_box/challenge_sign"
  "black_box/challenge_sigmoid"
  "white_box/openfhe/challenge_max"
)

for challenge in "${CHALLENGES[@]}"; do
  echo "Solving: $challenge"
  cd ~/research/fhe_challenges/$challenge
  claude --plugin-dir ~/fhe-challenge-solver -c "/fhe-solve"
done
EOF

chmod +x solve_all.sh
./solve_all.sh
```

## Example 7: Contributing/Testing Plugin Changes

```bash
# Fork and clone your own version
git clone https://github.com/YOUR_USERNAME/fhe-challenge-solver.git ~/my-fhe-solver

# Make changes to skills or agents
cd ~/my-fhe-solver
# Edit files...

# Test your changes
cd ~/research/fhe_challenges/black_box/challenge_sign
claude --plugin-dir ~/my-fhe-solver
/fhe-solve
```

## Understanding Output Artifacts

After running the solver, check these files:

```bash
cd <challenge_directory>

# Benchmark metrics (timing, accuracy, depth used)
cat artifacts/metrics.json

# Full execution log
less artifacts/run.log

# The implemented solution (C++)
cat templates/openfhe/yourSolution.cpp

# Or for Python ML challenges
cat templates/openfhe-python/app.py
```

## Troubleshooting

### Plugin not found
```bash
# Verify plugin location
ls -la ~/fhe-challenge-solver/.claude-plugin/plugin.json

# Should show the plugin manifest
```

### Command not available
```bash
# Make sure you started Claude Code with --plugin-dir
claude --plugin-dir ~/fhe-challenge-solver

# Check available commands (inside Claude Code session)
/help
```

### Docker permission errors (Black-box challenges)
```bash
# Add your user to docker group
sudo usermod -aG docker $USER
# Log out and log back in
```

### OpenFHE not found (White-box challenges)
```bash
# Verify OpenFHE installation
which openfhe

# Check CMake can find OpenFHE
cmake --find-package -DNAME=OpenFHE -DCOMPILER_ID=GNU -DLANGUAGE=CXX
```

## Advanced: Using Skills Directly

The plugin provides 6 skills that can be used independently:

```bash
# Inside Claude Code with plugin loaded
claude --plugin-dir ~/fhe-challenge-solver

# Ask Claude to use specific skills
> Can you use the function-approximation skill to find Chebyshev coefficients
> for sigmoid with degree 7 on range [-8, 8]?

> Use the ml-pipeline skill to train a regression model for this data
```

## Getting Updates

```bash
# Update your local plugin copy
cd ~/fhe-challenge-solver
git pull origin main

# Continue using with latest version
```
