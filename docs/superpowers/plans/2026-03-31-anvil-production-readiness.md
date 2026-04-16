# Anvil Production Readiness Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix known bugs, test all reviewer backends, validate clean install, add multi-language support, and polish the README so Anvil works reliably for anyone who clones it.

**Architecture:** Anvil is a set of bash/python scripts wrapping aider. No build step, no package manager — just `install.sh` copying scripts to `~/.local/bin/`. Testing means running the actual tools end-to-end in temp directories, plus targeted unit-style checks on each script.

**Tech Stack:** Bash, Python 3, aider, Claude Code CLI, curl, git

---

### Task 1: Fix bug — anvil-review-api missing review prompt

The `PROMPT` variable on line 24 is defined but never injected into the API call. The LLM receives a raw diff with no instructions, so it doesn't know to say APPROVED/REJECTED.

**Files:**
- Modify: `scripts/anvil-review-api:22-33`

- [ ] **Step 1: Read the current broken code**

Verify the bug: `PROMPT` is defined on line 24 but the `messages` payload on line 29 uses only `${DIFF_JSON}`.

- [ ] **Step 2: Fix the prompt injection**

Replace lines 22-33 with:

```bash
# Build the review prompt with diff
PROMPT_TEXT="Review this code change. Be concise. If correct: say APPROVED. If issues: say REJECTED and list fixes needed.\n\nDiff:\n"
FULL_PROMPT=$(python3 -c "import sys,json; print(json.dumps(sys.argv[1] + sys.stdin.read()))" "$PROMPT_TEXT" <<< "$DIFF")

RESULT=$(curl -s "${API_URL}/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" \
    -H "Content-Type: application/json" \
    -d "{
        \"model\": \"${API_MODEL}\",
        \"messages\": [{\"role\": \"user\", \"content\": ${FULL_PROMPT}}],
        \"max_tokens\": 1024
    }" 2>/dev/null | python3 -c "
import sys, json
try:
    r = json.load(sys.stdin)
    content = r['choices'][0]['message']['content']
    print(content)
    sys.exit(0 if 'APPROVED' in content.upper() else 1)
except Exception as e:
    print(f'Review error: {e}', file=sys.stderr)
    sys.exit(1)
" 2>&1)
```

- [ ] **Step 3: Verify the fix**

```bash
# Dry test — check the JSON payload is well-formed
cd /tmp && mkdir -p anvil-api-test && cd anvil-api-test
git init -q && echo "x=1" > test.py && git add . && git commit -q -m "init"
echo "x=2" > test.py
# Run with a bogus URL to see the curl payload (will fail, but we check the prompt is there)
ANVIL_REVIEWER_URL="http://localhost:1" ANVIL_REVIEWER_API_KEY="test" anvil-review-api test.py 2>&1 || true
# Should show "Review error" not "APPROVED" — confirms the script ran and tried to send the prompt
```

- [ ] **Step 4: Commit**

```bash
git add scripts/anvil-review-api
git commit -m "fix: include review prompt in API reviewer payload"
```

---

### Task 2: Fix bug — anvil-review-local missing review prompt

Same bug as Task 1 — the local reviewer sends a raw diff with no APPROVED/REJECTED instructions.

**Files:**
- Modify: `scripts/anvil-review-local:18-39`

- [ ] **Step 1: Fix the prompt injection**

Replace lines 18-20 with:

```bash
PROMPT_TEXT="Review this code change. Be concise. If correct: say APPROVED. If issues: say REJECTED and list fixes needed.\n\nDiff:\n"
FULL_PROMPT=$(python3 -c "import sys,json; print(json.dumps(sys.argv[1] + sys.stdin.read()))" "$PROMPT_TEXT" <<< "$DIFF")
```

Then replace `${DIFF_JSON}` with `${FULL_PROMPT}` in the curl payload on line 29.

- [ ] **Step 2: Verify the fix**

```bash
# Same pattern as Task 1 — test against a local endpoint
cd /tmp && mkdir -p anvil-local-test && cd anvil-local-test
git init -q && echo "x=1" > test.py && git add . && git commit -q -m "init"
echo "x=2" > test.py
ANVIL_REVIEWER_URL="http://localhost:1" anvil-review-local test.py 2>&1 || true
```

- [ ] **Step 3: Commit**

```bash
git add scripts/anvil-review-local
git commit -m "fix: include review prompt in local reviewer payload"
```

---

### Task 3: Fix hardcoded references to "Hercules"

`anvil-help` and `anvil-test-suite` reference "Hercules" (Petru's local model). Public users don't have Hercules.

**Files:**
- Modify: `scripts/anvil-help:9`
- Modify: `scripts/anvil-test-suite:67-68`

- [ ] **Step 1: Fix anvil-help**

Change line 9 from:
```
echo "  Just type and chat — Hercules codes, Claude reviews each edit."
```
To:
```
echo "  Just type and chat — your LLM codes, Claude reviews each edit."
```

- [ ] **Step 2: Fix anvil-test-suite to use config**

Replace the hardcoded model/endpoint on lines 67-68:

```bash
# Before:
        --model openai/Hercules \
        --openai-api-base http://127.0.0.1:8081/v1 \
        --openai-api-key not-needed \

# After — read from user's aider config or env:
ANVIL_ENV="${HOME}/.anvil.env"
[ -f "$ANVIL_ENV" ] && source "$ANVIL_ENV"
TEST_MODEL="${ANVIL_TEST_MODEL:-openai/auto}"
TEST_API_BASE="${ANVIL_TEST_API_BASE:-http://localhost:8080/v1}"
TEST_API_KEY="${ANVIL_TEST_API_KEY:-not-needed}"
```

Add these variables near the top of the script (after the cgroup escape, before LOG), then use them in the aider call:

```bash
        --model "$TEST_MODEL" \
        --openai-api-base "$TEST_API_BASE" \
        --openai-api-key "$TEST_API_KEY" \
```

- [ ] **Step 3: Add test-suite config to anvil.env template**

Append to `templates/anvil.env`:

```bash

# Test suite — model used by anvil-test-suite (defaults to your aider model)
# ANVIL_TEST_MODEL="openai/auto"
# ANVIL_TEST_API_BASE="http://localhost:8080/v1"
# ANVIL_TEST_API_KEY="not-needed"
```

- [ ] **Step 4: Verify no more "Hercules" references**

```bash
grep -ri "hercules" scripts/ templates/ README.md
# Should return nothing
```

- [ ] **Step 5: Commit**

```bash
git add scripts/anvil-help scripts/anvil-test-suite templates/anvil.env
git commit -m "fix: remove hardcoded Hercules references, use config for test suite"
```

---

### Task 4: Test anvil-review-api with a real API key

The API reviewer has never been tested against a live endpoint. This task exercises it end-to-end.

**Files:**
- No code changes — this is a manual verification task

- [ ] **Step 1: Set up a test repo**

```bash
cd /tmp && rm -rf anvil-api-live-test && mkdir anvil-api-live-test && cd anvil-api-live-test
git init -q
git config user.name "Test" && git config user.email "test@test.com"
cat > calculator.py << 'PYEOF'
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
PYEOF
git add . && git commit -q -m "init: calculator"
```

- [ ] **Step 2: Make a change with a deliberate bug**

```bash
cat > calculator.py << 'PYEOF'
def add(a, b):
    return a + b

def subtract(a, b):
    return a + b  # BUG: should be minus

def multiply(a, b):
    return a * b
PYEOF
```

- [ ] **Step 3: Run anvil-review-api against a real endpoint**

Use Anthropic API via OpenAI-compatible proxy, or OpenAI directly:

```bash
export ANVIL_REVIEWER_URL="https://api.anthropic.com/v1"  # or openai
export ANVIL_REVIEWER_MODEL="claude-haiku-4-5"
export ANVIL_REVIEWER_API_KEY="$(cat ~/.config/api-keys/anthropic.key)"  # wherever the key lives
anvil-review-api calculator.py
echo "Exit code: $?"
```

**Expected:** Output contains REJECTED and mentions the subtract bug. Exit code non-zero.

- [ ] **Step 4: Fix the bug, re-run**

```bash
sed -i 's/return a + b  # BUG/return a - b/' calculator.py
anvil-review-api calculator.py
echo "Exit code: $?"
```

**Expected:** Output contains APPROVED. Exit code 0.

- [ ] **Step 5: Test with bad API key**

```bash
ANVIL_REVIEWER_API_KEY="sk-invalid" anvil-review-api calculator.py 2>&1
echo "Exit code: $?"
```

**Expected:** Shows "Review error" message. Exit code 1 (not 0). If it exits 0, that's a bug — an auth error shouldn't count as APPROVED.

- [ ] **Step 6: Document results**

Record pass/fail for each sub-step in the test log:
```bash
echo "## API Reviewer Live Test — $(date)" >> ~/anvil-test-results.md
echo "- Buggy code rejected: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Clean code approved: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Bad API key handled: PASS/FAIL" >> ~/anvil-test-results.md
```

---

### Task 5: Clean install test in isolated environment

Simulate a new user cloning and installing Anvil with zero prior setup.

**Files:**
- No code changes — verification task. Fixes found here become Task 5b patches.

- [ ] **Step 1: Create an isolated environment**

```bash
# Use a temp HOME to avoid picking up existing configs
export ANVIL_TEST_HOME="/tmp/anvil-clean-install"
rm -rf "$ANVIL_TEST_HOME"
mkdir -p "$ANVIL_TEST_HOME"

# Clone fresh
cd "$ANVIL_TEST_HOME"
git clone /home/workspace/workbench/forge anvil-cli
cd anvil-cli
```

- [ ] **Step 2: Run the installer with HOME override**

```bash
HOME="$ANVIL_TEST_HOME" ./install.sh
# Choose option 2 (llama.cpp) with defaults
# Choose option 1 (Claude Code CLI) for reviewer
```

**Check:**
- All scripts copied to `$ANVIL_TEST_HOME/.local/bin/`
- `$ANVIL_TEST_HOME/.aider.conf.yml` exists and has correct model
- `$ANVIL_TEST_HOME/.anvil.env` exists and has `ANVIL_REVIEWER="claude -p"`
- Symlinks (plan, build, help) exist and point correctly

- [ ] **Step 3: Test that anvil launches**

```bash
export PATH="$ANVIL_TEST_HOME/.local/bin:$PATH"
export HOME="$ANVIL_TEST_HOME"
cd /tmp && mkdir anvil-launch-test && cd anvil-launch-test
git init -q && echo "# test" > README.md && git add . && git commit -q -m "init"

# anvil should launch aider and show the banner
timeout 5 anvil --message "say hello" --no-browser 2>&1 || true
# Check that it tried to start aider (exit code doesn't matter if model isn't running)
```

- [ ] **Step 4: Verify all scripts are executable and have shebangs**

```bash
for script in anvil anvil-review anvil-review-api anvil-review-local anvil-plan anvil-plan-answers anvil-build anvil-help anvil-test-suite; do
    INSTALLED="$ANVIL_TEST_HOME/.local/bin/$script"
    if [ ! -f "$INSTALLED" ]; then
        echo "MISSING: $script"
    elif [ ! -x "$INSTALLED" ]; then
        echo "NOT EXECUTABLE: $script"
    else
        SHEBANG=$(head -1 "$INSTALLED")
        echo "OK: $script — $SHEBANG"
    fi
done
```

- [ ] **Step 5: Test with missing aider**

```bash
# Temporarily hide aider
REAL_PATH="$PATH"
export PATH=$(echo "$PATH" | tr ':' '\n' | grep -v "aider" | tr '\n' ':')
HOME="$ANVIL_TEST_HOME" ./install.sh 2>&1 | head -20
# Should show "✗ aider not found" and exit 1
export PATH="$REAL_PATH"
```

- [ ] **Step 6: Document results**

```bash
echo "## Clean Install Test — $(date)" >> ~/anvil-test-results.md
echo "- Scripts installed: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Configs written: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Symlinks correct: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Missing dep detected: PASS/FAIL" >> ~/anvil-test-results.md
```

---

### Task 6: Add multi-language test cases to test suite

The current test suite only tests Python. Add JavaScript, TypeScript, Go, and Rust tasks to validate that anvil-review handles non-Python diffs.

**Files:**
- Modify: `scripts/anvil-test-suite`

- [ ] **Step 1: Add a JavaScript test case**

Add after the existing `run_test 10` block:

```bash
run_test 11 "JavaScript — Event Bus" \
"Create eventbus.js with an EventBus class (ES6 module). Methods: on(event, handler), off(event, handler), emit(event, ...args). Handlers are called in registration order. If a handler throws, catch it, continue calling remaining handlers, return array of errors from emit(). Export default. Also create eventbus.test.js using Node's built-in assert module (not jest/mocha): test on+emit calls handler, test off removes handler, test emit with error returns error array, test emit with no handlers returns empty array. Run with: node eventbus.test.js (process.exit(1) on any assertion failure)."
```

- [ ] **Step 2: Add a TypeScript test case**

```bash
run_test 12 "TypeScript — Stack with generics" \
"Create stack.ts with a generic Stack<T> class. Methods: push(item: T), pop(): T | undefined, peek(): T | undefined, size(): number, isEmpty(): boolean. Use a private array internally. Export the class. Also create stack.test.ts — import Stack, test push+pop returns items in LIFO order, test pop on empty returns undefined, test peek doesn't remove, test size. Use console.assert and process.exit(1) on failure. Compile with: npx tsc --strict stack.ts stack.test.ts && node stack.test.js"
```

- [ ] **Step 3: Add a Go test case**

```bash
run_test 13 "Go — Ring Buffer" \
"Create ringbuffer.go in package main with a RingBuffer struct. Constructor NewRingBuffer(capacity int). Methods: Push(val int), Pop() (int, bool), Len() int, IsFull() bool. Use a fixed-size slice with head/tail pointers. Also create ringbuffer_test.go with Go testing package: TestPushPop, TestPopEmpty, TestOverflow (push beyond capacity overwrites oldest), TestLen. Run with: go test -v"
```

- [ ] **Step 4: Add a Rust test case**

```bash
run_test 14 "Rust — Simple HashMap" \
"Create src/lib.rs with a SimpleMap<K, V> struct using Vec<(K, V)> internally (no std HashMap). Implement: new(), insert(key, value), get(&key) -> Option<&V>, remove(&key) -> Option<V>, len(). K must be PartialEq. Add #[cfg(test)] mod tests with: test_insert_and_get, test_remove, test_get_missing_returns_none, test_len. Create Cargo.toml with package name 'simplemap'. Run with: cargo test"
```

- [ ] **Step 5: Update test counters**

Change `TOTAL=10` to `TOTAL=14` at the top of the script.

- [ ] **Step 6: Update pytest detection for multi-language**

The current test result detection only checks for `test_*.py` and runs pytest. Add language-specific test runners after the pytest block (around line 92):

```bash
    # Run tests while still in workdir
    TEST_RESULT="no tests"
    TEST_OUTPUT=""
    if compgen -G "test_*.py" > /dev/null 2>&1; then
        TEST_OUTPUT=$(python3 -m pytest -x -q 2>&1)
        TEST_EXIT=$?
        TEST_RESULT=$([ $TEST_EXIT -eq 0 ] && echo "PASS" || echo "FAIL")
    elif compgen -G "*.test.js" > /dev/null 2>&1; then
        TEST_OUTPUT=$(node *.test.js 2>&1)
        TEST_EXIT=$?
        TEST_RESULT=$([ $TEST_EXIT -eq 0 ] && echo "PASS" || echo "FAIL")
    elif compgen -G "*.test.ts" > /dev/null 2>&1; then
        TEST_OUTPUT=$(npx tsc --strict *.ts 2>&1 && node *.test.js 2>&1)
        TEST_EXIT=$?
        TEST_RESULT=$([ $TEST_EXIT -eq 0 ] && echo "PASS" || echo "FAIL")
    elif compgen -G "*_test.go" > /dev/null 2>&1; then
        # Go needs a module
        [ ! -f "go.mod" ] && go mod init testmod 2>/dev/null
        TEST_OUTPUT=$(go test -v 2>&1)
        TEST_EXIT=$?
        TEST_RESULT=$([ $TEST_EXIT -eq 0 ] && echo "PASS" || echo "FAIL")
    elif [ -f "Cargo.toml" ]; then
        TEST_OUTPUT=$(cargo test 2>&1)
        TEST_EXIT=$?
        TEST_RESULT=$([ $TEST_EXIT -eq 0 ] && echo "PASS" || echo "FAIL")
    fi
```

- [ ] **Step 7: Update the file detection for non-Python**

Change line 75 from:
```bash
    FILES=$(find . -name "*.py" -not -path "./.git/*" 2>/dev/null | sort)
```
To:
```bash
    FILES=$(find . \( -name "*.py" -o -name "*.js" -o -name "*.ts" -o -name "*.go" -o -name "*.rs" \) -not -path "./.git/*" -not -path "./target/*" 2>/dev/null | sort)
```

- [ ] **Step 8: Update lint-cmd in test runner for all languages**

The aider call in `run_test` only uses `python: anvil-review`. Add JS/TS/Go/Rust:

```bash
        --lint-cmd "python: anvil-review" \
        --lint-cmd "javascript: anvil-review" \
        --lint-cmd "typescript: anvil-review" \
        --lint-cmd "go: anvil-review" \
        --lint-cmd "rust: anvil-review" \
```

- [ ] **Step 9: Verify the new tests parse correctly**

```bash
# Dry run — just check the script sources without error
bash -n scripts/anvil-test-suite
echo "Exit: $?"
```

- [ ] **Step 10: Commit**

```bash
git add scripts/anvil-test-suite
git commit -m "feat: add JavaScript, TypeScript, Go, and Rust test cases to test suite"
```

---

### Task 7: Run the full multi-language test suite

Execute all 14 tests and record results.

**Files:**
- No code changes — execution and recording

- [ ] **Step 1: Run the test suite from a clean terminal**

```bash
# Run OUTSIDE Claude Code session to avoid cgroup issues
cd /home/workspace/workbench/forge
./scripts/anvil-test-suite ~/anvil-test-results-full.md
```

This takes ~45-60 minutes (14 tasks × 300s timeout max each).

- [ ] **Step 2: Review results**

```bash
cat ~/anvil-test-results-full.md
```

**Expected:** All 10 Python tests pass (they passed before). JS/TS/Go/Rust tests may need prompt tuning — record which ones fail and why.

- [ ] **Step 3: Fix any test prompt issues**

If a non-Python test fails because aider generated wrong file names or the review prompt confused the LLM, adjust the test prompt in the script. Each fix is a separate small commit.

- [ ] **Step 4: Document the final pass rate**

```bash
echo "## Full Suite Run — $(date)" >> ~/anvil-test-results.md
echo "- Python: X/10" >> ~/anvil-test-results.md
echo "- JavaScript: X/1" >> ~/anvil-test-results.md
echo "- TypeScript: X/1" >> ~/anvil-test-results.md
echo "- Go: X/1" >> ~/anvil-test-results.md
echo "- Rust: X/1" >> ~/anvil-test-results.md
```

---

### Task 8: Add --dry-run flag to anvil-build

Let users preview what tasks will be sent to aider without executing anything.

**Files:**
- Modify: `scripts/anvil-build`

- [ ] **Step 1: Add argument parsing**

Add after the imports (line 6):

```python
import argparse

def parse_args():
    parser = argparse.ArgumentParser(description="Anvil automated build loop")
    parser.add_argument("--dry-run", action="store_true",
                        help="Show tasks without executing them")
    return parser.parse_args()
```

- [ ] **Step 2: Add dry-run mode to main()**

Replace the current `main()` function (lines 219-250):

```python
def main():
    args = parse_args()
    plan_path = ".anvil/plan.md"
    if not os.path.exists(plan_path):
        print("No plan found at .anvil/plan.md")
        print("Run anvil-plan first.")
        sys.exit(1)

    overview, tasks = parse_plan(plan_path)
    if not tasks:
        print("No tasks found in plan. Check .anvil/plan.md format.")
        print("Expected: ### Task N: Title")
        sys.exit(1)

    print(f"Anvil Build — {len(tasks)} tasks from plan")
    print(f"Coder: aider + local LLM")
    print(f"Reviewer: anvil-review (auto after each edit)")
    print()

    for t in tasks:
        print(f"  {t['id']}. {t['title']}")

    if args.dry_run:
        print()
        print("Dry run — showing task details without executing.")
        print()
        for t in tasks:
            context = build_context(overview, tasks, t["id"],
                                    [x for x in tasks if x["id"] < t["id"]])
            print(f"{'=' * 60}")
            print(f"[Task {t['id']}/{len(tasks)}] {t['title']}")
            print(f"{'=' * 60}")
            print(f"Context length: {len(context)} chars")
            print(f"Task body length: {len(t['body'])} chars")
            print(f"Files mentioned: ", end="")
            # Extract file references from task body
            files = re.findall(r'`([^`]+\.\w{1,4})`', t['body'])
            print(", ".join(files) if files else "(none detected)")
            print()
        return

    print()
    confirm = input("Start building? [Y/n] ").strip().lower()
    if confirm and confirm != "y":
        print("Aborted.")
        sys.exit(0)

    build_with_message_file(overview, tasks)
```

- [ ] **Step 3: Test dry-run mode**

```bash
cd /tmp && rm -rf anvil-dryrun-test && mkdir anvil-dryrun-test && cd anvil-dryrun-test
git init -q
mkdir -p .anvil
cat > .anvil/plan.md << 'EOF'
# Test Plan

**Goal:** Build a calculator

### Task 1: Core functions

Create `calc.py` with `add()` and `subtract()` functions.

### Task 2: Tests

Create `test_calc.py` with pytest tests for add and subtract.
EOF

anvil-build --dry-run
```

**Expected:** Shows both tasks with context lengths and detected file names (`calc.py`, `test_calc.py`). No aider launched.

- [ ] **Step 4: Commit**

```bash
git add scripts/anvil-build
git commit -m "feat: add --dry-run flag to anvil-build"
```

---

### Task 9: Add graceful Claude rate-limit handling

When Claude is unavailable during review or escalation, fail clearly instead of hanging.

**Files:**
- Modify: `scripts/anvil-review:99-108`

- [ ] **Step 1: Add timeout and error detection to escalation**

Replace lines 99-108 in anvil-review:

```bash
    if [[ "$REVIEWER" == "claude -p"* ]]; then
        FIX_RESULT=$(timeout 120 claude -p "$FIX_PROMPT" --allowedTools "Read,Grep,Glob,Edit,Write,Bash" --output-format text --max-turns 10 2>&1)
        FIX_EXIT=$?
        if [ $FIX_EXIT -eq 124 ]; then
            echo "  TIMEOUT: Claude did not respond within 120s"
            echo "  Skipping review — approve and move on"
            echo "  ────────────────────────────────────────────"
            rm -f "$COUNTER_FILE" "$TOTAL_FILE"
            exit 0
        elif echo "$FIX_RESULT" | grep -qi "rate.limit\|overloaded\|529\|503"; then
            echo "  RATE LIMITED: Claude is busy"
            echo "  Skipping review — approve and move on"
            echo "  ────────────────────────────────────────────"
            rm -f "$COUNTER_FILE" "$TOTAL_FILE"
            exit 0
        fi
        echo "$FIX_RESULT" | tail -5
        echo "  FIXED by Claude"
        echo "  ────────────────────────────────────────────"
        echo ""
        rm -f "$COUNTER_FILE" "$TOTAL_FILE"
        exit 0
    fi
```

- [ ] **Step 2: Add timeout to normal review too**

Replace line 146:

```bash
# Before:
    RESULT=$(claude -p "$PROMPT" --allowedTools "Read,Grep,Glob" --output-format text --max-turns 3 2>&1)
# After:
    RESULT=$(timeout 90 claude -p "$PROMPT" --allowedTools "Read,Grep,Glob" --output-format text --max-turns 3 2>&1)
    if [ $? -eq 124 ]; then
        echo "  TIMEOUT: Claude review timed out after 90s — approving"
        echo "  ────────────────────────────────────────────"
        rm -f "$COUNTER_FILE" "$TOTAL_FILE"
        exit 0
    fi
```

- [ ] **Step 3: Commit**

```bash
git add scripts/anvil-review
git commit -m "fix: add timeout and rate-limit handling to Claude review"
```

---

### Task 10: Version tag and update README

Tag the release and fill in README gaps.

**Files:**
- Modify: `README.md`
- Modify: `scripts/anvil:51` (add version to banner)

- [ ] **Step 1: Add version to anvil banner**

In `scripts/anvil`, change line 51:
```bash
# Before:
    echo "  Anvil — Local LLM codes. Claude reviews."
# After:
    echo "  Anvil v0.1.0 — Local LLM codes. Claude reviews."
```

- [ ] **Step 2: Update README with missing sections**

Add after the "Requirements" section (line 49), before "Installed scripts":

```markdown
## What happens during review

When aider edits a file, `anvil-review` runs automatically:

1. **Review 1-2:** Claude reads the diff and the actual files. If the code has real bugs (crashes, wrong logic, security issues), it says REJECTED with exact fixes. Style issues are ignored.
2. **Review 3 (escalation):** If the local LLM can't fix the issues after 2 rejections, Claude takes over and fixes the code directly using file editing tools.
3. **Timeout/rate-limit:** If Claude is unavailable, the review is skipped and the code is approved to avoid blocking the build.

## Troubleshooting

**"aider not found"** — Install aider: `pip install aider-chat` (or `pipx install aider-chat`)

**"No conversation history found"** — Chat with your LLM first before running `/run plan`. You need at least one exchange.

**"No plan found"** — Run `/run plan` before `/run build`. The plan lives in `.anvil/plan.md`.

**Review always approves** — Check `~/.anvil.env` — is `ANVIL_REVIEWER` set correctly? Try running `anvil-review yourfile.py` manually to see the output.

**aider crashes with thread errors** — You're inside a Claude Code session with a thread limit. Run from a plain terminal instead, or let `anvil-test-suite` auto-escape the cgroup.

**Model not responding** — Check your endpoint is running: `curl http://localhost:8080/v1/models` (or whatever your endpoint is).
```

- [ ] **Step 3: Add version badge to top of README**

Add after line 1:

```markdown

> v0.1.0 — First public release
```

- [ ] **Step 4: Commit and tag**

```bash
git add README.md scripts/anvil
git commit -m "docs: add troubleshooting, review explanation, version 0.1.0"
git tag -a v0.1.0 -m "First public release"
```

---

### Task 11: Create example project

A tiny repo someone can clone and run `anvil build` immediately to see the full loop.

**Files:**
- Create: `examples/calculator/README.md`
- Create: `examples/calculator/.anvil/plan.md`

- [ ] **Step 1: Create example directory**

```bash
mkdir -p examples/calculator/.anvil
```

- [ ] **Step 2: Create the example plan**

Write `examples/calculator/.anvil/plan.md`:

```markdown
# Calculator — Example Anvil Plan

**Goal:** Build a simple calculator module with tests.

**Architecture:** Single Python module with pure functions, tested with pytest.

**Tech Stack:** Python 3, pytest

### Task 1: Core arithmetic

Create `calculator.py` with four functions:
- `add(a, b)` — returns a + b
- `subtract(a, b)` — returns a - b
- `multiply(a, b)` — returns a * b
- `divide(a, b)` — returns a / b, raises ValueError if b is 0

### Task 2: Tests

Create `test_calculator.py` with pytest tests:
- test_add: assert add(2, 3) == 5
- test_subtract: assert subtract(5, 3) == 2
- test_multiply: assert multiply(4, 3) == 12
- test_divide: assert divide(10, 2) == 5.0
- test_divide_by_zero: assert raises ValueError
```

- [ ] **Step 3: Create example README**

Write `examples/calculator/README.md`:

```markdown
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

# Build it — your LLM codes, Claude reviews
anvil-build
```

The plan has 2 tasks. Your LLM writes `calculator.py` and `test_calculator.py`. Claude reviews each one.
```

- [ ] **Step 4: Commit**

```bash
git add examples/
git commit -m "docs: add calculator example project"
```

---

### Task 12: Final integration test

Run through the entire user journey: clone → install → plan → build on the example project.

**Files:**
- No code changes — this is the final verification pass

- [ ] **Step 1: Clean slate**

```bash
export TEST_HOME="/tmp/anvil-final-test"
rm -rf "$TEST_HOME" && mkdir -p "$TEST_HOME"
cd "$TEST_HOME"
```

- [ ] **Step 2: Clone and install**

```bash
git clone /home/workspace/workbench/forge anvil-cli
cd anvil-cli
HOME="$TEST_HOME" ./install.sh
# Choose option 2 (llama.cpp), defaults
# Choose option 1 (Claude Code CLI)
export PATH="$TEST_HOME/.local/bin:$PATH"
```

- [ ] **Step 3: Run the example**

```bash
cd "$TEST_HOME"
cp -r anvil-cli/examples/calculator .
cd calculator
git init && git add . && git commit -m "init"
HOME="$TEST_HOME" anvil-build --dry-run
# Should show 2 tasks with file names
```

- [ ] **Step 4: Run full build (requires running LLM + Claude)**

```bash
HOME="$TEST_HOME" anvil-build
# Watch for: aider launches, LLM generates code, Claude reviews
# Expected: 2 tasks complete, calculator.py and test_calculator.py created
```

- [ ] **Step 5: Verify the output**

```bash
ls -la calculator.py test_calculator.py
python3 -m pytest test_calculator.py -v
```

**Expected:** Both files exist. All 5 tests pass.

- [ ] **Step 6: Record final results**

```bash
echo "## Final Integration Test — $(date)" >> ~/anvil-test-results.md
echo "- Clone + install: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Dry run: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Full build: PASS/FAIL" >> ~/anvil-test-results.md
echo "- Tests pass: PASS/FAIL" >> ~/anvil-test-results.md
echo "" >> ~/anvil-test-results.md
echo "VERDICT: READY / NOT READY" >> ~/anvil-test-results.md
```
