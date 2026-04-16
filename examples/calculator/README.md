# Calculator — Anvil Example

Try the full anvil workflow on this tiny project.

## Setup

```bash
cd examples/calculator
git init && git add . && git commit -m "init"
```

## Run

```bash
# Preview the plan
anvil-build --dry-run

# Build it — your coder writes the code, your auditor reviews the result
anvil-build
```

The plan has 2 tasks. Your coder writes `calculator.py` and `test_calculator.py`. Once both are written, pytest runs and the auditor looks at the outcome. If the tests pass, the auditor signs off. If not, the auditor fixes the failing code in place.
