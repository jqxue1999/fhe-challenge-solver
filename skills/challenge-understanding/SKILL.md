---
name: challenge-understanding
description: Parse any FHE challenge specification to extract scheme, operation, constraints, and scoring. Use when analyzing new FHE challenges, extracting requirements, or determining challenge types.
---

# Challenge Understanding

Detect challenge category, parse requirements, and route to correct workflow.

## Skill Responsibility

**Primary Function:** First step for ANY challenge - detect type and determine workflow

**Inputs:**
- Challenge directory structure
- `challenge.md` file
- Template files in `templates/`
- Test files in `tests/`
- Data files in `data/` (if present)

**Outputs:**
- Challenge category (black-box / white-box-openfhe / white-box-ml)
- Parsed requirements and constraints
- CLI arguments for the solution
- List of algorithm skills needed
- Validation workflow to use

---

## Step 1: Detect Challenge Category

### Category Detection Algorithm

```python
def detect_challenge_category(challenge_dir):
    """
    Detect challenge category based on directory structure.
    """
    has_dockerfile = exists(f"{challenge_dir}/Dockerfile")
    has_verify_sh = exists(f"{challenge_dir}/verify.sh")
    has_testcase_dirs = any(d.startswith("testcase") for d in listdir(f"{challenge_dir}/tests/"))
    has_test_case_json = exists(f"{challenge_dir}/tests/test_case.json")
    has_data_folder = exists(f"{challenge_dir}/data/")

    # Black-box: Has Dockerfile + testcase directories with pre-encrypted files
    if has_dockerfile and has_testcase_dirs:
        return "black_box"

    # White-box ML: Has verify.sh + test_case.json + data folder
    if has_verify_sh and has_test_case_json and has_data_folder:
        return "white_box_ml"

    # White-box OpenFHE: Has verify.sh + test_case.json (no data folder)
    if has_verify_sh and has_test_case_json:
        return "white_box_openfhe"

    return "unknown"
```

### Category Characteristics

| Category | Dockerfile | verify.sh | tests/ | data/ | Templates |
|----------|-----------|-----------|--------|-------|-----------|
| black_box | ✓ | ✗ | testcase*/ dirs | ✗ | openfhe/ AND openfhe-python/ |
| white_box_openfhe | ✗ | ✓ | test_case.json | ✗ | openfhe/ AND openfhe-python/ |
| white_box_ml | ✗ | ✓ | test_case.json | ✓ | openfhe/ OR openfhe-python/ (check which exists) |

### Path Pattern Detection

User provides absolute path like `/home/yifei/data/cipherbench/aideml/fhe_challenge/black_box/challenge_sign`

| Category | Path Contains | Example |
|----------|--------------|---------|
| black_box | `/black_box/challenge_` | `.../fhe_challenge/black_box/challenge_sign` |
| white_box_openfhe | `/white_box/openfhe/challenge_` | `.../fhe_challenge/white_box/openfhe/challenge_max` |
| white_box_ml | `/white_box/ml_inference/challenge_` | `.../fhe_challenge/white_box/ml_inference/challenge_house_prediction` |
| non_openfhe (skip) | `/white_box/non-OpenFHE/` | `.../fhe_challenge/white_box/non-OpenFHE/challenge_swift` |

```python
def detect_category_from_path(challenge_path):
    """Detect category from absolute path."""
    if "/black_box/challenge_" in challenge_path:
        return "black_box"
    elif "/white_box/openfhe/challenge_" in challenge_path:
        return "white_box_openfhe"
    elif "/white_box/ml_inference/challenge_" in challenge_path:
        return "white_box_ml"
    elif "/white_box/non-OpenFHE/" in challenge_path:
        return "non_openfhe"  # Skip
    return "unknown"
```

---

## Step 2: Parse Challenge Requirements

### Read challenge.md

Extract key information:

```python
def parse_challenge_md(challenge_md_path):
    """
    Extract requirements from challenge.md
    """
    content = read_file(challenge_md_path)

    return {
        "challenge_id": extract_pattern(r"challenge_id:\s*(\w+)", content),
        "scheme": extract_scheme(content),  # CKKS, BFV, BGV
        "task_description": extract_section("Introduction", content),
        "input_format": extract_section("Input", content),
        "output_format": extract_section("Output", content),
        "constraints": extract_constraints(content),
        "accuracy_threshold": extract_accuracy(content),
        "evaluation_criteria": extract_section("Evaluation", content),
    }
```

### Scheme Detection

```python
def extract_scheme(content):
    content_lower = content.lower()
    if "ckks" in content_lower:
        return "CKKS"
    elif "bfv" in content_lower:
        return "BFV"
    elif "bgv" in content_lower:
        return "BGV"
    elif "tfhe" in content_lower:
        return "TFHE"
    return "CKKS"  # Default for most FHERMA challenges
```

### Constraint Extraction

```python
def extract_constraints(content):
    return {
        "mult_depth": extract_number(r"mult(?:iplication)?[_\s]?depth[:\s]+(\d+)", content),
        "ring_dimension": extract_number(r"ring[_\s]?dimension[:\s]+(\d+)", content),
        "batch_size": extract_number(r"batch[_\s]?size[:\s]+(\d+)", content),
        "scale_mod_size": extract_number(r"scale[_\s]?mod[_\s]?size[:\s]+(\d+)", content),
        "input_range": extract_range(content),  # e.g., [0, 255] or [-25, 25]
        "accuracy_threshold": extract_float(r"accuracy[:\s]+(\d+\.?\d*)%?", content),
    }
```

---

## Step 3: Parse Test Format

### Black-Box: Testcase Directories

```python
def parse_black_box_tests(tests_dir):
    """
    Parse pre-encrypted test case structure.
    """
    testcases = []
    for tc_dir in sorted(glob(f"{tests_dir}/testcase*")):
        tc = {
            "id": basename(tc_dir),
            "files": {
                "cc": find_file(tc_dir, ["cc.txt", "cc.json"]),
                "input": find_file(tc_dir, ["input.txt", "input.bin"]),
                "key_pub": find_file(tc_dir, ["key_pub.txt", "key_public.txt"]),
                "key_mult": find_file(tc_dir, ["key_mult.txt", "key_relin.txt"]),
                "key_rot": find_file(tc_dir, ["key_rot.txt", "key_auto.txt"]),
                "key_secret": find_file(tc_dir, ["key_secret.txt"]),  # For validation
                "expected_output": find_file(tc_dir, ["expected_output.txt"]),
                "plaintext_input": find_file(tc_dir, ["plaintext_input.txt"]),
            }
        }
        testcases.append(tc)
    return testcases
```

### White-Box: test_case.json

```python
def parse_white_box_tests(tests_dir):
    """
    Parse test_case.json format.
    """
    with open(f"{tests_dir}/test_case.json") as f:
        test_data = json.load(f)

    # test_case.json format:
    # [{
    #     "scheme": "CKKS",
    #     "significant_slots_number": 1,
    #     "runs": [{
    #         "input": [{"name": "array", "value": [...]}],
    #         "output": [...]
    #     }]
    # }]

    return {
        "scheme": test_data[0].get("scheme", "CKKS"),
        "significant_slots": test_data[0].get("significant_slots_number", 1),
        "runs": test_data[0].get("runs", []),
        "input_names": [inp["name"] for inp in test_data[0]["runs"][0]["input"]],
        "expected_outputs": [run["output"] for run in test_data[0]["runs"]],
    }
```

---

## Step 4: Extract CLI Arguments

### From Template main.cpp

```python
def extract_cli_args_cpp(main_cpp_path):
    """
    Parse CLI arguments from main.cpp template.
    """
    content = read_file(main_cpp_path)

    # Pattern: else if (arg == "--sample")
    args = re.findall(r'arg\s*==\s*"(--\w+)"', content)

    # Common mappings
    cli_map = {
        "--input": "Single encrypted input",
        "--sample": "Encrypted sample/feature vector",
        "--array": "Encrypted array",
        "--matrix_a": "First encrypted matrix",
        "--matrix_b": "Second encrypted matrix",
        "--cc": "CryptoContext path",
        "--output": "Output ciphertext path",
        "--key_pub": "Public key",
        "--key_public": "Public key (alternate)",
        "--key_mult": "Multiplication key",
        "--key_rot": "Rotation key",
    }

    return {arg: cli_map.get(arg, "Unknown") for arg in args}
```

### From Template app.py

```python
def extract_cli_args_python(app_py_path):
    """
    Parse CLI arguments from app.py template.
    """
    content = read_file(app_py_path)

    # Pattern: parser.add_argument('--sample')
    args = re.findall(r"add_argument\(['\"](-{1,2}\w+)['\"]", content)

    return args
```

---

## Step 5: Determine Required Skills

### Task Detection

```python
def detect_required_skills(challenge_info):
    """
    Determine which algorithm skills are needed.
    """
    skills = []
    description = challenge_info["task_description"].lower()

    # Function approximation indicators
    nonlinear_keywords = [
        "sigmoid", "relu", "gelu", "softmax", "tanh", "sign",
        "exp", "log", "sqrt", "inverse", "activation",
        "approximate", "polynomial", "chebyshev"
    ]
    if any(kw in description for kw in nonlinear_keywords):
        skills.append("function-approximation")

    # Encrypted computation indicators
    computation_keywords = [
        "matrix", "vector", "multiplication", "sort", "sorting",
        "max", "min", "argmax", "comparison", "distance", "knn",
        "lookup", "table", "parity", "shift", "xor", "bit"
    ]
    if any(kw in description for kw in computation_keywords):
        skills.append("encrypted-computation")

    # ML pipeline indicators
    if challenge_info.get("has_training_data"):
        skills.append("ml-pipeline")

    # Always need these
    skills.extend(["openfhe-mastery", "solution-engineering"])

    return list(set(skills))
```

---

## Step 6: Parse ML Training Data (If Present)

### data_info.json

```python
def parse_ml_data(data_dir):
    """
    Parse ML training data information.
    """
    info_path = f"{data_dir}/data_info.json"
    if not exists(info_path):
        return None

    with open(info_path) as f:
        info = json.load(f)

    return {
        "challenge": info.get("challenge"),
        "num_features": info.get("num_features"),
        "num_samples": info.get("num_samples"),
        "feature_columns": info.get("X_train", {}).get("columns", []),
        "feature_range": info.get("feature_range"),
        "X_train_path": f"{data_dir}/{info['X_train']['file']}",
        "y_train_path": f"{data_dir}/{info['y_train']['file']}",
        "task_type": infer_task_type(info),  # "regression" or "classification"
    }

def infer_task_type(info):
    """
    Infer if it's regression or classification.
    """
    num_classes = info.get("num_classes", 0)
    if num_classes <= 10:
        return "classification"
    return "regression"  # Many unique values = regression
```

---

## Output Format

After parsing, produce structured analysis:

```json
{
  "category": "white_box_ml",
  "challenge_id": "676035a7890eef39561cf7c9",
  "scheme": "CKKS",

  "task_summary": "Build FHE regression model for house price prediction",

  "constraints": {
    "mult_depth": 29,
    "ring_dimension": 131072,
    "batch_size": 65536,
    "accuracy_threshold": 0.85
  },

  "cli_arguments": {
    "--sample": "Encrypted feature vector",
    "--cc": "CryptoContext path",
    "--output": "Output ciphertext path",
    "--key_pub": "Public key",
    "--key_mult": "Multiplication key",
    "--key_rot": "Rotation key"
  },

  "test_info": {
    "format": "test_case.json",
    "num_runs": 1,
    "input_names": ["sample"],
    "significant_slots": 1
  },

  "ml_data": {
    "num_features": 13,
    "num_samples": 16346,
    "task_type": "regression",
    "X_train_path": "data/X_train.csv",
    "y_train_path": "data/y_train.csv"
  },

  "required_skills": [
    "ml-pipeline",
    "openfhe-mastery",
    "solution-engineering"
  ],

  "validation_workflow": {
    "command": "./verify.sh",
    "validator": "fherma-validator"
  },

  "template_path": "templates/openfhe-python/"
}
```

---

## Challenge-Specific Patterns

### Black-Box Activation Functions (relu, sigmoid, sign)

```json
{
  "category": "black_box",
  "required_skills": ["function-approximation", "openfhe-mastery", "solution-engineering"],
  "key_pattern": "Load CryptoContext FROM ciphertext",
  "validation_workflow": {
    "command": "docker build -t challenge . && docker run --rm -v $(pwd)/tests/testcase1:/data challenge"
  }
}
```

### White-Box Matrix Operations

```json
{
  "category": "white_box_openfhe",
  "required_skills": ["encrypted-computation", "openfhe-mastery", "solution-engineering"],
  "key_pattern": "Load CryptoContext from --cc argument",
  "validation_workflow": {
    "command": "./verify.sh"
  }
}
```

### White-Box ML Inference

```json
{
  "category": "white_box_ml",
  "required_skills": ["ml-pipeline", "openfhe-mastery", "solution-engineering"],
  "key_pattern": "Train model first, then implement FHE inference",
  "validation_workflow": {
    "command": "./verify.sh"
  }
}
```

---

## Integration with Other Skills

After challenge understanding is complete:

1. **Route to `ml-pipeline`** if `has_training_data` is true
2. **Route to `function-approximation`** if nonlinear functions detected
3. **Route to `encrypted-computation`** if matrix/comparison/sorting detected
4. **Always use `openfhe-mastery`** for API patterns
5. **Always use `solution-engineering`** for template adaptation and validation
