---
name: ml-pipeline
description: Full ML workflow - training, optimization, weight extraction, and FHE inference. Use for challenges with training data (data/ folder) requiring model development before encrypted inference.
---

# ML Pipeline

End-to-end machine learning: training, weight extraction, and FHE inference implementation.

## Skill Responsibility

**Primary Function:** Handle ML challenges from training data to FHE inference

**When to Use:**
- Challenge has `data/` folder with training data
- Challenge type is `white_box_ml`
- Challenges: house_prediction, cifar10, etc.

**Workflow:**
1. Load and analyze training data
2. Train model to high accuracy
3. Extract weights for FHE
4. Implement FHE inference

---

## Accuracy Thresholds

Before implementing FHE inference, ensure trained model meets:

| Task Type | Metric | Threshold |
|-----------|--------|-----------|
| Regression | R² | ≥ 0.85 |
| Classification | Accuracy | ≥ 0.85 |
| Multi-class | Top-1 Accuracy | ≥ 0.85 |

---

## Phase 1: Data Analysis

### Load Training Data

```python
import pandas as pd
import json

# Load data info
with open("data/data_info.json") as f:
    info = json.load(f)

# Load training data
X_train = pd.read_csv("data/X_train.csv")
y_train = pd.read_csv("data/y_train.csv")

print(f"Features: {X_train.shape[1]}")
print(f"Samples: {X_train.shape[0]}")
print(f"Target range: [{y_train.min()}, {y_train.max()}]")
```

### Determine Task Type

```python
def determine_task_type(y):
    unique_values = y.nunique()
    if unique_values <= 10:
        return "classification", unique_values
    else:
        return "regression", None

task_type, num_classes = determine_task_type(y_train)
```

### Feature Analysis

```python
# Check for missing values
print(f"Missing values: {X_train.isnull().sum().sum()}")

# Feature statistics
print(X_train.describe())

# Correlation with target (for regression)
if task_type == "regression":
    correlations = X_train.corrwith(y_train.iloc[:, 0]).abs().sort_values(ascending=False)
    print("Top correlated features:", correlations.head(10))
```

---

## Phase 2: Model Training

### For Regression (e.g., house_prediction)

```python
from sklearn.linear_model import LinearRegression, Ridge
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import cross_val_score
from sklearn.metrics import r2_score
import numpy as np

# Preprocessing
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)

# Try multiple models
models = {
    "LinearRegression": LinearRegression(),
    "Ridge": Ridge(alpha=1.0),
}

best_model = None
best_score = -np.inf

for name, model in models.items():
    scores = cross_val_score(model, X_scaled, y_train.values.ravel(), cv=5, scoring='r2')
    mean_score = scores.mean()
    print(f"{name}: R² = {mean_score:.4f} (+/- {scores.std()*2:.4f})")

    if mean_score > best_score:
        best_score = mean_score
        best_model = model

# Train final model
best_model.fit(X_scaled, y_train.values.ravel())
print(f"\nBest model R²: {best_score:.4f}")

# Verify threshold
assert best_score >= 0.85, f"Model R² {best_score:.4f} below threshold 0.85"
```

### For Classification (e.g., cifar10)

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
from sklearn.neural_network import MLPClassifier
from sklearn.metrics import accuracy_score

# Try multiple models
models = {
    "LogisticRegression": LogisticRegression(max_iter=1000),
    "MLP": MLPClassifier(hidden_layer_sizes=(64, 32), max_iter=500),
}

best_model = None
best_score = 0

for name, model in models.items():
    scores = cross_val_score(model, X_scaled, y_train.values.ravel(), cv=5)
    mean_score = scores.mean()
    print(f"{name}: Accuracy = {mean_score:.4f}")

    if mean_score > best_score:
        best_score = mean_score
        best_model = model

# Train final model
best_model.fit(X_scaled, y_train.values.ravel())
print(f"\nBest model accuracy: {best_score:.4f}")

# Verify threshold
assert best_score >= 0.85, f"Model accuracy {best_score:.4f} below threshold 0.85"
```

### Hyperparameter Tuning

```python
from sklearn.model_selection import GridSearchCV

# Example for Ridge regression
param_grid = {'alpha': [0.01, 0.1, 1.0, 10.0, 100.0]}
grid_search = GridSearchCV(Ridge(), param_grid, cv=5, scoring='r2')
grid_search.fit(X_scaled, y_train.values.ravel())

print(f"Best params: {grid_search.best_params_}")
print(f"Best R²: {grid_search.best_score_:.4f}")
```

---

## Phase 3: Weight Extraction

### Linear Models

```python
def extract_linear_weights(model, scaler):
    """Extract weights and bias for FHE inference."""
    # Get raw weights
    weights = model.coef_.flatten()
    bias = model.intercept_

    # Combine with scaler: y = w^T * ((x - mean) / std) + b
    # = (w / std)^T * x + (b - w^T * mean / std)
    scaled_weights = weights / scaler.scale_
    adjusted_bias = bias - np.dot(weights, scaler.mean_ / scaler.scale_)

    return {
        "weights": scaled_weights.tolist(),
        "bias": float(adjusted_bias),
        "num_features": len(weights)
    }

model_params = extract_linear_weights(best_model, scaler)
print(f"Extracted {model_params['num_features']} weights")
```

### Neural Networks (MLP)

```python
def extract_mlp_weights(model):
    """Extract weights from sklearn MLP."""
    layers = []
    for i, (W, b) in enumerate(zip(model.coefs_, model.intercepts_)):
        layers.append({
            "W": W.tolist(),
            "b": b.tolist(),
            "input_size": W.shape[0],
            "output_size": W.shape[1]
        })
    return {"layers": layers, "activation": "relu"}

mlp_params = extract_mlp_weights(best_model)
```

### Save for FHE Implementation

```python
import json

# Save model parameters
with open("model_params.json", "w") as f:
    json.dump(model_params, f, indent=2)
```

---

## Phase 4: FHE Inference Implementation

### Linear Regression (Python)

```python
from openfhe import *

def solve(cc, ct_sample, key_pub, key_mult, key_rot):
    """FHE inference for linear regression."""
    # Load trained weights
    weights = [0.1, 0.2, ...]  # From model_params.json
    bias = 0.5

    # Enable features
    cc.Enable(PKESchemeFeature.PKE)
    cc.Enable(PKESchemeFeature.KEYSWITCH)
    cc.Enable(PKESchemeFeature.LEVELEDSHE)
    cc.Enable(PKESchemeFeature.ADVANCEDSHE)

    # Create plaintext weights
    pt_weights = cc.MakeCKKSPackedPlaintext(weights)

    # Dot product: encrypted_sample * weights
    result = cc.EvalMult(ct_sample, pt_weights)

    # Sum all slots (inner product)
    result = cc.EvalSum(result, len(weights))

    # Add bias
    pt_bias = cc.MakeCKKSPackedPlaintext([bias])
    result = cc.EvalAdd(result, pt_bias)

    return result
```

### Linear Regression (C++)

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> ct_sample) {
    // Trained weights (extracted from model)
    std::vector<double> weights = {0.1, 0.2, ...};
    double bias = 0.5;

    // Create plaintext
    auto pt_weights = cc->MakeCKKSPackedPlaintext(weights);

    // Dot product
    auto result = cc->EvalMult(ct_sample, pt_weights);

    // Sum slots
    result = cc->EvalSum(result, weights.size());

    // Add bias
    auto pt_bias = cc->MakeCKKSPackedPlaintext({bias});
    result = cc->EvalAdd(result, pt_bias);

    return result;
}
```

### Classification with Argmax

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> ct_sample) {
    // For multi-class: compute logits for each class
    // Then return argmax (index of max logit)

    // Logits = W * x + b
    auto logits = linear_layer(cc, ct_sample, weights, biases);

    // Argmax uses comparison tree
    // Output: ciphertext where slot i has max value if class i is predicted
    return logits;  // Or implement argmax if needed
}
```

### CNN for Image Classification

```cpp
Ciphertext<DCRTPoly> eval(CryptoContext<DCRTPoly> cc,
                          Ciphertext<DCRTPoly> ct_image) {
    // Conv layer 1
    auto conv1_out = conv_layer(cc, ct_image, conv1_filters, conv1_bias);
    auto relu1_out = approximate_relu(cc, conv1_out);

    // Pooling
    auto pool1_out = avg_pool(cc, relu1_out, pool_size);

    // Flatten
    auto flat = pool1_out;  // Reorganize slots if needed

    // FC layer
    auto fc1_out = linear_layer(cc, flat, fc1_weights, fc1_bias);
    auto relu2_out = approximate_relu(cc, fc1_out);

    // Output layer
    auto logits = linear_layer(cc, relu2_out, fc2_weights, fc2_bias);

    return logits;
}
```

---

## Web Search for SOTA ML-FHE

### When to Search

- Need FHE-friendly architectures for image classification
- Looking for depth-efficient neural network designs
- Want state-of-the-art accuracy on encrypted data
- Need quantization techniques for FHE

### Search Queries

```
# FHE-friendly neural networks
site:arxiv.org FHE neural network inference CKKS
site:scholar.google.com "homomorphic encryption" CNN architecture

# Specific challenges
site:arxiv.org encrypted image classification CIFAR
site:arxiv.org FHE linear regression

# Optimization techniques
site:arxiv.org FHE activation approximation
site:arxiv.org "depth budget" neural network FHE

# Frameworks
site:github.com encrypted machine learning inference
site:github.com FHE neural network
```

### Key Papers

```
# Foundational
"CryptoNets: Applying Neural Networks to Encrypted Data"
"GAZELLE: A Low Latency Framework for Secure Neural Network Inference"
"Faster CryptoNets: Leveraging Sparsity for Real-World Encrypted Inference"

# Recent advances
"PEGASUS: Bridging Polynomial and Non-polynomial Evaluations in Homomorphic Encryption"
"ResNet is All You Need? Modeling A(X) in HE-friendly Convolutional Neural Network"
```

---

## FHE-Friendly Model Design

### Architecture Guidelines

| Component | FHE-Friendly | Avoid |
|-----------|--------------|-------|
| Activation | Low-degree polynomial, square | ReLU (expensive), softmax |
| Pooling | Average pooling | Max pooling (expensive) |
| Normalization | Pre-computed batch norm | Layer norm (encrypted) |
| Layers | Shallow (2-3 layers) | Deep networks |

### Depth Budget Planning

```
Typical depth budget: 15-29

Linear layer: 1 depth
Polynomial activation (degree 3): 3 depth
Average pooling: log(pool_size) depth

Example CNN:
- Conv1 + ReLU3: 1 + 3 = 4
- AvgPool (2x2): 1
- Conv2 + ReLU3: 1 + 3 = 4
- AvgPool (2x2): 1
- FC + ReLU3: 1 + 3 = 4
- FC output: 1
Total: 16 depth
```

### Polynomial Activations

```python
# Train model with polynomial activations
# Square activation: f(x) = x²
class SquareActivation(nn.Module):
    def forward(self, x):
        return x ** 2

# Cubic approximation to ReLU
class CubicApproxReLU(nn.Module):
    def forward(self, x):
        return 0.5 * x + 0.125 * x**3  # Approximate ReLU
```

---

## Challenge-Specific Guidance

### house_prediction

**Model:** Linear regression
**Training:**
```python
from sklearn.linear_model import Ridge
model = Ridge(alpha=1.0)
model.fit(X_scaled, y_train)
```
**FHE:** Single linear layer (dot product + bias)
**Depth:** 1 + log(features)

### cifar10

**Model:** Small CNN or MLP
**Training:**
```python
# Use PyTorch for more control
model = nn.Sequential(
    nn.Conv2d(3, 16, 3, padding=1),
    SquareActivation(),  # FHE-friendly
    nn.AvgPool2d(2),
    nn.Flatten(),
    nn.Linear(16 * 16 * 16, 64),
    SquareActivation(),
    nn.Linear(64, 10)
)
```
**FHE:** Conv → Square → AvgPool → FC → Square → FC
**Depth:** ~12-15

---

## Common Pitfalls

1. **Training without FHE constraints** → Use FHE-friendly activations during training
2. **Forgetting scaler transformation** → Fuse normalization into weights
3. **Too deep networks** → Stay within depth budget
4. **Max pooling in FHE** → Use average pooling
5. **Full softmax** → Skip if only need argmax

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `challenge-understanding` | Detects ML challenge, parses data_info.json |
| `openfhe-mastery` | Python/C++ API for FHE inference |
| `function-approximation` | Activation function approximations |
| `encrypted-computation` | Matrix operations, argmax |
| `fhe-verification-framework` | Verify CNN or complex network layers with NumPy before OpenFHE |
| `solution-engineering` | Template adaptation, validation |
