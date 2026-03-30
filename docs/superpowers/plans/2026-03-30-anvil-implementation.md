# Anvil Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build anvil — a set of scripts that wrap aider with a local LLM as coder and Claude Code CLI as reviewer, plus an installer for open-source distribution.

**Architecture:** Shell scripts in `scripts/`, config templates in `templates/`, one Python script for the automated build loop. `install.sh` copies scripts to `~/.local/bin/` and writes config files. No Python packaging — just files.

**Tech Stack:** Bash, Python 3.12+, aider (dependency), Claude Code CLI (optional dependency)

---

### File Structure

```
anvil-cli/                       # GitHub repo root
├── install.sh                   # interactive installer
├── scripts/
│   ├── anvil                    # launcher (bash)
│   ├── anvil-review             # review dispatcher (bash)
│   ├── anvil-review-api         # API reviewer fallback (bash)
│   ├── anvil-review-local       # local LLM reviewer fallback (bash)
│   ├── anvil-plan               # planning script (bash)
│   ├── anvil-plan-answers       # plan Q&A followup (bash)
│   └── anvil-build              # automated build loop (python)
├── templates/
│   ├── aider.conf.yml           # aider config template
│   └── anvil.env                # reviewer config template
├── README.md
└── LICENSE                      # Apache-2.0
```

Installed on user's machine:
```
~/.local/bin/anvil
~/.local/bin/anvil-review
~/.local/bin/anvil-review-api
~/.local/bin/anvil-review-local
~/.local/bin/anvil-plan
~/.local/bin/anvil-plan-answers
~/.local/bin/anvil-build
~/.aider.conf.yml
~/.anvil.env
```

---

### Task 1: Restructure repo for anvil-cli

**Files:**
- Remove: `forge/` (old Python package), `pyproject.toml`, `forge.egg-info/`
- Create: `scripts/`, `templates/`, `LICENSE`

- [ ] **Step 1: Archive old forge code**

```bash
cd /home/workspace/workbench/forge
mkdir -p .archive
mv forge/ .archive/forge-old
mv pyproject.toml .archive/
rm -rf forge.egg-info
```

- [ ] **Step 2: Create new directory structure**

```bash
mkdir -p scripts templates
```

- [ ] **Step 3: Create LICENSE file**

```bash
cat > LICENSE << 'EOF'
                                 Apache License
                           Version 2.0, January 2004
                        http://www.apache.org/licenses/

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
EOF
```

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: restructure repo for anvil-cli"
```

---

### Task 2: Write `anvil` launcher script

**Files:**
- Create: `scripts/anvil`

- [ ] **Step 1: Write the launcher**

```bash
cat > scripts/anvil << 'SCRIPT'
#!/usr/bin/env bash
# anvil — launch aider with local LLM + Claude Code review
# https://github.com/steamvibe/anvil-cli

set -euo pipefail

# Load config if exists
ANVIL_ENV="${HOME}/.anvil.env"
[ -f "$ANVIL_ENV" ] && source "$ANVIL_ENV"

# Prevent OpenBLAS thread explosion on machines running llama.cpp
export OPENBLAS_NUM_THREADS="${OPENBLAS_NUM_THREADS:-1}"

exec aider "$@"
SCRIPT
chmod +x scripts/anvil
```

The actual model and endpoint config lives in `~/.aider.conf.yml` — the launcher just sets env vars and passes through all args to aider.

- [ ] **Step 2: Verify it runs**

```bash
cd /home/steamvibe/forge-test/clean
OPENAI_API_BASE=http://127.0.0.1:8081/v1 OPENAI_API_KEY=x /home/workspace/workbench/forge/scripts/anvil --model openai/Qwen3-Coder-Next --no-analytics --no-show-model-warnings --no-show-release-notes --message "say hello" --yes
```

Expected: Aider connects to Hercules and responds.

- [ ] **Step 3: Commit**

```bash
git add scripts/anvil
git commit -m "feat: add anvil launcher script"
```

---

### Task 3: Write `anvil-review` — Claude Code reviewer

**Files:**
- Create: `scripts/anvil-review`

- [ ] **Step 1: Write the review dispatcher**

```bash
cat > scripts/anvil-review << 'SCRIPT'
#!/usr/bin/env bash
# anvil-review — aider --lint-cmd hook that sends diffs to a reviewer
# Called by aider after every file edit. Exit 0 = approved, non-zero = rejected.

set -uo pipefail

ANVIL_ENV="${HOME}/.anvil.env"
[ -f "$ANVIL_ENV" ] && source "$ANVIL_ENV"

REVIEWER="${ANVIL_REVIEWER:-claude -p}"
MAX_DIFF_CHARS="${ANVIL_MAX_DIFF_CHARS:-40000}"

FILES="$@"
[ -z "$FILES" ] && exit 0

# Get the diff for changed files — staged or last commit
DIFF=$(git diff HEAD -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && DIFF=$(git diff HEAD~1 -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && exit 0

# Truncate huge diffs
DIFF_LEN=${#DIFF}
if [ "$DIFF_LEN" -gt "$MAX_DIFF_CHARS" ]; then
    DIFF="${DIFF:0:$MAX_DIFF_CHARS}

[diff truncated — ${DIFF_LEN} chars total, showing first ${MAX_DIFF_CHARS}]"
fi

PROMPT="Review this code change. Be concise. Focus on bugs, logic errors, and security issues. Ignore style.
If the code is correct and complete: say APPROVED on its own line.
If there are issues: say REJECTED on its own line, then list what needs fixing.

Diff:
${DIFF}"

if [[ "$REVIEWER" == "claude -p"* ]]; then
    RESULT=$(claude -p "$PROMPT" --allowedTools "Read,Grep,Glob" --output-format text --max-turns 1 2>&1)
elif [[ "$REVIEWER" == "anvil-review-api" ]]; then
    RESULT=$(anvil-review-api $FILES 2>&1)
elif [[ "$REVIEWER" == "anvil-review-local" ]]; then
    RESULT=$(anvil-review-local $FILES 2>&1)
else
    RESULT=$($REVIEWER $FILES 2>&1)
fi

echo "$RESULT"

echo "$RESULT" | grep -qi "APPROVED" && exit 0 || exit 1
SCRIPT
chmod +x scripts/anvil-review
```

- [ ] **Step 2: Test with a real diff**

Create a test change and run the review script against it:

```bash
cd /home/steamvibe/forge-test/clean
echo 'print("bug")' >> hello.py
git add hello.py
ANVIL_REVIEWER="claude -p" /home/workspace/workbench/forge/scripts/anvil-review hello.py
echo "Exit code: $?"
git checkout hello.py
```

Expected: Claude reviews the diff and prints APPROVED or REJECTED with exit code 0 or 1.

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add scripts/anvil-review
git commit -m "feat: add anvil-review — Claude Code lint hook for aider"
```

---

### Task 4: Write reviewer alternatives (API + local)

**Files:**
- Create: `scripts/anvil-review-api`
- Create: `scripts/anvil-review-local`

- [ ] **Step 1: Write API reviewer**

```bash
cat > scripts/anvil-review-api << 'SCRIPT'
#!/usr/bin/env bash
# anvil-review-api — review code via any OpenAI-compatible API
# Used when ANVIL_REVIEWER="anvil-review-api" in ~/.anvil.env

set -uo pipefail

ANVIL_ENV="${HOME}/.anvil.env"
[ -f "$ANVIL_ENV" ] && source "$ANVIL_ENV"

API_URL="${ANVIL_REVIEWER_URL:-https://api.openai.com/v1}"
API_MODEL="${ANVIL_REVIEWER_MODEL:-gpt-4o}"
API_KEY="${ANVIL_REVIEWER_API_KEY:-}"

FILES="$@"
[ -z "$FILES" ] && exit 0

DIFF=$(git diff HEAD -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && DIFF=$(git diff HEAD~1 -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && exit 0

# Escape diff for JSON
DIFF_JSON=$(python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))" <<< "$DIFF")

PROMPT="Review this code change. Be concise. If correct: say APPROVED. If issues: say REJECTED and list fixes needed.\n\nDiff:\n"

RESULT=$(curl -s "${API_URL}/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" \
    -H "Content-Type: application/json" \
    -d "{
        \"model\": \"${API_MODEL}\",
        \"messages\": [{\"role\": \"user\", \"content\": ${DIFF_JSON}}],
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

echo "$RESULT"
exit ${PIPESTATUS[0]:-1}
SCRIPT
chmod +x scripts/anvil-review-api
```

- [ ] **Step 2: Write local LLM reviewer**

```bash
cat > scripts/anvil-review-local << 'SCRIPT'
#!/usr/bin/env bash
# anvil-review-local — review code via a local LLM (no API key needed)
# Used when ANVIL_REVIEWER="anvil-review-local" in ~/.anvil.env

set -uo pipefail

ANVIL_ENV="${HOME}/.anvil.env"
[ -f "$ANVIL_ENV" ] && source "$ANVIL_ENV"

API_URL="${ANVIL_REVIEWER_URL:-http://localhost:11434/v1}"
API_MODEL="${ANVIL_REVIEWER_MODEL:-auto}"

FILES="$@"
[ -z "$FILES" ] && exit 0

DIFF=$(git diff HEAD -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && DIFF=$(git diff HEAD~1 -- $FILES 2>/dev/null)
[ -z "$DIFF" ] && exit 0

DIFF_JSON=$(python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))" <<< "$DIFF")

RESULT=$(curl -s "${API_URL}/chat/completions" \
    -H "Content-Type: application/json" \
    -d "{
        \"model\": \"${API_MODEL}\",
        \"messages\": [{\"role\": \"user\", \"content\": ${DIFF_JSON}}],
        \"max_tokens\": 1024,
        \"chat_template_kwargs\": {\"enable_thinking\": false}
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

echo "$RESULT"
exit ${PIPESTATUS[0]:-1}
SCRIPT
chmod +x scripts/anvil-review-local
```

- [ ] **Step 3: Test local reviewer against Hermes**

```bash
cd /home/steamvibe/forge-test/clean
echo 'print("test")' >> hello.py
git add hello.py
ANVIL_REVIEWER_URL="http://localhost:8082/v1" ANVIL_REVIEWER_MODEL="auto" /home/workspace/workbench/forge/scripts/anvil-review-local hello.py
echo "Exit: $?"
git checkout hello.py
```

- [ ] **Step 4: Commit**

```bash
cd /home/workspace/workbench/forge
git add scripts/anvil-review-api scripts/anvil-review-local
git commit -m "feat: add API and local LLM reviewer alternatives"
```

---

### Task 5: Write `anvil-plan` — Claude planning script

**Files:**
- Create: `scripts/anvil-plan`

- [ ] **Step 1: Write the planning script**

```bash
cat > scripts/anvil-plan << 'SCRIPT'
#!/usr/bin/env bash
# anvil-plan — extract conversation from aider, send to Claude for detailed planning
# Run from within aider: /run anvil-plan

set -uo pipefail

HISTORY=".aider.chat.history.md"
if [ ! -f "$HISTORY" ]; then
    echo "No conversation history found."
    echo "Chat with your LLM first, then run: /run anvil-plan"
    exit 1
fi

# Extract the conversation — user messages (####) and assistant responses
# Skip aider metadata lines (starting with >)
CONVERSATION=$(awk '
    /^#### / { sub(/^#### /, ""); print "USER: " $0; next }
    /^> / { next }
    /^# aider/ { next }
    { print }
' "$HISTORY" | tail -200)

if [ -z "$CONVERSATION" ]; then
    echo "No conversation content found in history."
    exit 1
fi

mkdir -p .anvil

echo "Sending conversation to Claude for detailed planning..."
echo ""

RESULT=$(claude -p "You are a senior software architect. Below is a conversation where a developer brainstormed a project idea with an LLM.

Your job: produce a detailed implementation plan with numbered tasks.

Each task must have:
- A clear title
- What files to create or modify
- Step-by-step instructions (specific enough for a coding agent)
- Acceptance criteria

If you need clarification before planning, put questions under a QUESTIONS: header at the top. Otherwise go straight to the task list.

Output the plan in markdown. Start each task with ### Task N: Title

---

Conversation:
${CONVERSATION}" --allowedTools "Read,Grep,Glob" --output-format text --max-turns 5 2>&1)

echo "$RESULT"

echo "$RESULT" > .anvil/plan.md
echo ""
echo "────────────────────────────────────"
echo "Plan saved to .anvil/plan.md"

if echo "$RESULT" | grep -qi "^QUESTIONS:"; then
    echo ""
    echo "Claude has questions. Answer them with:"
    echo "  /run anvil-plan-answers \"your answers here\""
else
    echo "Plan ready. Review it, then start building with:"
    echo "  /run anvil-build"
fi
SCRIPT
chmod +x scripts/anvil-plan
```

- [ ] **Step 2: Verify script is syntactically correct**

```bash
bash -n /home/workspace/workbench/forge/scripts/anvil-plan && echo "OK"
```

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add scripts/anvil-plan
git commit -m "feat: add anvil-plan — Claude Code planning script"
```

---

### Task 6: Write `anvil-plan-answers` — follow-up for Claude's questions

**Files:**
- Create: `scripts/anvil-plan-answers`

- [ ] **Step 1: Write the script**

```bash
cat > scripts/anvil-plan-answers << 'SCRIPT'
#!/usr/bin/env bash
# anvil-plan-answers — answer Claude's planning questions and get final plan
# Run from within aider: /run anvil-plan-answers "answer1. answer2."

set -uo pipefail

ANSWERS="$*"
if [ -z "$ANSWERS" ]; then
    echo "Usage: /run anvil-plan-answers \"your answers to Claude's questions\""
    exit 1
fi

PLAN_FILE=".anvil/plan.md"
if [ ! -f "$PLAN_FILE" ]; then
    echo "No plan found. Run anvil-plan first."
    exit 1
fi

PLAN=$(cat "$PLAN_FILE")

echo "Sending answers to Claude..."
echo ""

RESULT=$(claude -p "You previously produced this plan and had questions:

${PLAN}

The developer answered:
${ANSWERS}

Now produce the final implementation plan with numbered tasks. No more questions. Use the same format: ### Task N: Title for each task." --allowedTools "Read,Grep,Glob" --output-format text --max-turns 5 2>&1)

echo "$RESULT"

echo "$RESULT" > "$PLAN_FILE"
echo ""
echo "────────────────────────────────────"
echo "Updated plan saved to .anvil/plan.md"
echo "Review it, then start building with:"
echo "  /run anvil-build"
SCRIPT
chmod +x scripts/anvil-plan-answers
```

- [ ] **Step 2: Verify syntax**

```bash
bash -n /home/workspace/workbench/forge/scripts/anvil-plan-answers && echo "OK"
```

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add scripts/anvil-plan-answers
git commit -m "feat: add anvil-plan-answers — follow-up for Claude's planning questions"
```

---

### Task 7: Write `anvil-build` — automated build loop

**Files:**
- Create: `scripts/anvil-build`

This is the core automation — parses the plan, drives aider programmatically, feeds tasks one by one.

- [ ] **Step 1: Write the build script**

```bash
cat > scripts/anvil-build << 'SCRIPT'
#!/usr/bin/env python3
"""anvil-build — automated build loop.

Parses tasks from .anvil/plan.md, feeds each to aider via its Python API.
Aider's --lint-cmd triggers Claude review automatically after each edit.

Run from within aider: /run anvil-build
Or standalone: anvil-build
"""

import os
import re
import sys


def parse_tasks(plan_path):
    """Extract tasks from the plan markdown."""
    with open(plan_path) as f:
        content = f.read()

    tasks = []
    # Split on ### Task N: Title
    parts = re.split(r'### Task \d+:\s*', content)
    titles = re.findall(r'### Task \d+:\s*(.+)', content)

    for i, (title, body) in enumerate(zip(titles, parts[1:]), 1):
        tasks.append({
            "id": i,
            "title": title.strip(),
            "description": f"Task {i}: {title.strip()}\n\n{body.strip()}",
        })
    return tasks


def build_with_aider_api(tasks):
    """Drive aider programmatically via its Python API."""
    try:
        from aider.coders import Coder
        from aider.models import Model
        from aider.io import InputOutput
    except ImportError:
        print("Error: aider not installed. Run: pip install aider-chat")
        sys.exit(1)

    # Load config from env / anvil.env
    anvil_env = os.path.expanduser("~/.anvil.env")
    env = {}
    if os.path.exists(anvil_env):
        with open(anvil_env) as f:
            for line in f:
                line = line.strip()
                if line and not line.startswith("#") and "=" in line:
                    k, v = line.split("=", 1)
                    env[k.strip()] = v.strip().strip('"').strip("'")

    max_retries = int(env.get("ANVIL_MAX_RETRIES", "2"))

    io = InputOutput(yes=True)
    model_name = os.environ.get("AIDER_MODEL", "openai/Qwen3-Coder-Next")
    model = Model(model_name)

    # Discover files in the repo
    import glob
    fnames = []
    for ext in ["*.py", "*.js", "*.ts", "*.jsx", "*.tsx", "*.go", "*.rs"]:
        fnames.extend(glob.glob(f"**/{ext}", recursive=True))

    coder = Coder.create(
        main_model=model,
        io=io,
        fnames=fnames[:20],  # limit to avoid overwhelming context
        auto_lint=True,
        lint_cmds={"python": "anvil-review"},
    )

    total = len(tasks)
    accepted = 0
    failed = 0

    for task in tasks:
        print(f"\n{'='*60}")
        print(f"[Task {task['id']}/{total}] {task['title']}")
        print(f"{'='*60}\n")

        try:
            coder.run(task["description"])
            accepted += 1
            print(f"\n✓ Task {task['id']} complete")
        except Exception as e:
            print(f"\n✗ Task {task['id']} failed: {e}")
            failed += 1

    print(f"\n{'='*60}")
    print(f"Build complete: {accepted} accepted, {failed} failed out of {total}")
    print(f"{'='*60}")


def build_with_message_file(tasks):
    """Fallback: drive aider via --message-file for each task."""
    import subprocess
    import tempfile

    total = len(tasks)
    accepted = 0
    failed = 0

    for task in tasks:
        print(f"\n{'='*60}")
        print(f"[Task {task['id']}/{total}] {task['title']}")
        print(f"{'='*60}\n")

        with tempfile.NamedTemporaryFile(mode="w", suffix=".md", delete=False) as f:
            f.write(task["description"])
            msg_file = f.name

        try:
            result = subprocess.run(
                ["aider", "--message-file", msg_file, "--yes", "--no-analytics"],
                timeout=600,
            )
            if result.returncode == 0:
                accepted += 1
                print(f"\n✓ Task {task['id']} complete")
            else:
                failed += 1
                print(f"\n✗ Task {task['id']} exited with code {result.returncode}")
        except subprocess.TimeoutExpired:
            failed += 1
            print(f"\n✗ Task {task['id']} timed out after 600s")
        finally:
            os.unlink(msg_file)

    print(f"\n{'='*60}")
    print(f"Build complete: {accepted} accepted, {failed} failed out of {total}")
    print(f"{'='*60}")


def main():
    plan_path = ".anvil/plan.md"
    if not os.path.exists(plan_path):
        print("No plan found at .anvil/plan.md")
        print("Run anvil-plan first.")
        sys.exit(1)

    tasks = parse_tasks(plan_path)
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

    print()
    confirm = input("Start building? [Y/n] ").strip().lower()
    if confirm and confirm != "y":
        print("Aborted.")
        sys.exit(0)

    # Try Python API first, fall back to message-file
    try:
        from aider.coders import Coder
        build_with_aider_api(tasks)
    except ImportError:
        print("Aider Python API not available, using --message-file fallback")
        build_with_message_file(tasks)


if __name__ == "__main__":
    main()
SCRIPT
chmod +x scripts/anvil-build
```

- [ ] **Step 2: Verify syntax**

```bash
python3 -m py_compile /home/workspace/workbench/forge/scripts/anvil-build && echo "OK"
```

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add scripts/anvil-build
git commit -m "feat: add anvil-build — automated task-by-task build with aider"
```

---

### Task 8: Write config templates

**Files:**
- Create: `templates/aider.conf.yml`
- Create: `templates/anvil.env`

- [ ] **Step 1: Write aider config template**

```bash
cat > templates/aider.conf.yml << 'EOF'
# Anvil — aider configuration
# This file is written by install.sh. Edit to change defaults.
# See: https://aider.chat/docs/config.html

model: __MODEL__
openai-api-base: __API_BASE__
openai-api-key: __API_KEY__

auto-lint: true
lint-cmd: anvil-review

no-analytics: true
no-show-model-warnings: true
no-show-release-notes: true

map-tokens: 1024
EOF
```

- [ ] **Step 2: Write anvil.env template**

```bash
cat > templates/anvil.env << 'EOF'
# Anvil reviewer configuration
# Edit this file to change how code gets reviewed.

# Reviewer command — called after every aider edit
# Options:
#   "claude -p"          — Claude Code CLI (subscription, recommended)
#   "anvil-review-api"   — any OpenAI-compatible API
#   "anvil-review-local" — local LLM (free)
ANVIL_REVIEWER="__REVIEWER__"

# API reviewer settings (only used when ANVIL_REVIEWER="anvil-review-api")
# ANVIL_REVIEWER_URL="https://api.openai.com/v1"
# ANVIL_REVIEWER_MODEL="gpt-4o"
# ANVIL_REVIEWER_API_KEY="sk-..."

# Local reviewer settings (only used when ANVIL_REVIEWER="anvil-review-local")
# ANVIL_REVIEWER_URL="http://localhost:11434/v1"
# ANVIL_REVIEWER_MODEL="auto"

# Max diff size sent to reviewer (chars). Longer diffs are truncated.
ANVIL_MAX_DIFF_CHARS=40000

# Max retries before escalation (0 = no retries, just report failure)
ANVIL_MAX_RETRIES=2
EOF
```

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add templates/
git commit -m "feat: add config templates for aider and anvil"
```

---

### Task 9: Write `install.sh` — interactive installer

**Files:**
- Create: `install.sh`

- [ ] **Step 1: Write the installer**

```bash
cat > install.sh << 'INSTALLER'
#!/usr/bin/env bash
# Anvil installer — sets up scripts and config
# Usage: ./install.sh

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
BIN_DIR="${HOME}/.local/bin"
AIDER_CONF="${HOME}/.aider.conf.yml"
ANVIL_ENV="${HOME}/.anvil.env"

echo ""
echo "  Anvil Setup"
echo "  ─────────────────────────────────"
echo "  Local LLM codes. Claude reviews."
echo ""

# ── Check dependencies ──────────────────────────────
MISSING=0

if command -v aider &>/dev/null; then
    AIDER_VER=$(aider --version 2>&1 | head -1)
    echo "  ✓ aider ($AIDER_VER)"
else
    echo "  ✗ aider not found"
    echo "    Install: pip install aider-chat"
    MISSING=1
fi

if command -v git &>/dev/null; then
    echo "  ✓ git"
else
    echo "  ✗ git not found"
    MISSING=1
fi

HAS_CLAUDE=0
if command -v claude &>/dev/null; then
    CLAUDE_VER=$(claude --version 2>&1 | head -1)
    echo "  ✓ Claude Code ($CLAUDE_VER)"
    HAS_CLAUDE=1
else
    echo "  ⚠ Claude Code not found (optional — can use API reviewer instead)"
fi

echo ""

[ "$MISSING" -eq 1 ] && echo "Install missing dependencies first." && exit 1

# ── Coder model ─────────────────────────────────────
echo "Coder model (the LLM that writes code):"
echo ""
echo "  1. Local LLM — Ollama"
echo "  2. Local LLM — llama.cpp / LM Studio / vLLM (OpenAI-compatible)"
echo "  3. API — Deepseek"
echo "  4. API — OpenRouter"
echo "  5. API — Anthropic (Haiku/Sonnet)"
echo "  6. Custom endpoint"
echo ""
read -rp "Choice [2]: " CODER_CHOICE
CODER_CHOICE="${CODER_CHOICE:-2}"

case "$CODER_CHOICE" in
    1)
        read -rp "Ollama model name [codellama]: " MODEL_NAME
        MODEL_NAME="${MODEL_NAME:-codellama}"
        API_BASE="http://localhost:11434"
        API_KEY="not-needed"
        MODEL="ollama_chat/${MODEL_NAME}"
        ;;
    2)
        read -rp "Endpoint URL [http://localhost:8080/v1]: " API_BASE
        API_BASE="${API_BASE:-http://localhost:8080/v1}"
        read -rp "Model name [auto]: " MODEL_NAME
        MODEL_NAME="${MODEL_NAME:-auto}"
        API_KEY="not-needed"
        MODEL="openai/${MODEL_NAME}"
        ;;
    3)
        read -rp "Deepseek API key: " API_KEY
        API_BASE="https://api.deepseek.com/v1"
        MODEL="deepseek/deepseek-coder"
        ;;
    4)
        read -rp "OpenRouter API key: " API_KEY
        read -rp "Model [meta-llama/llama-3-70b]: " MODEL_NAME
        MODEL_NAME="${MODEL_NAME:-meta-llama/llama-3-70b}"
        API_BASE="https://openrouter.ai/api/v1"
        MODEL="openrouter/${MODEL_NAME}"
        ;;
    5)
        read -rp "Anthropic API key: " API_KEY
        read -rp "Model [claude-haiku-4-5]: " MODEL_NAME
        MODEL_NAME="${MODEL_NAME:-claude-haiku-4-5}"
        API_BASE=""
        MODEL="anthropic/${MODEL_NAME}"
        ;;
    6)
        read -rp "Endpoint URL: " API_BASE
        read -rp "Model name: " MODEL_NAME
        read -rp "API key (empty if none): " API_KEY
        API_KEY="${API_KEY:-not-needed}"
        MODEL="openai/${MODEL_NAME}"
        ;;
    *)
        echo "Invalid choice"; exit 1 ;;
esac

# ── Reviewer ────────────────────────────────────────
echo ""
echo "Reviewer (checks code after each edit):"
echo ""
if [ "$HAS_CLAUDE" -eq 1 ]; then
    echo "  1. Claude Code CLI (subscription — recommended)"
else
    echo "  1. Claude Code CLI (not installed — install first)"
fi
echo "  2. OpenAI-compatible API"
echo "  3. Local LLM"
echo "  4. None (skip review)"
echo ""
read -rp "Choice [1]: " REVIEWER_CHOICE
REVIEWER_CHOICE="${REVIEWER_CHOICE:-1}"

REVIEWER_CMD="claude -p"
REVIEWER_EXTRA=""

case "$REVIEWER_CHOICE" in
    1) REVIEWER_CMD="claude -p" ;;
    2)
        read -rp "Reviewer API URL [https://api.openai.com/v1]: " R_URL
        R_URL="${R_URL:-https://api.openai.com/v1}"
        read -rp "Reviewer model [gpt-4o]: " R_MODEL
        R_MODEL="${R_MODEL:-gpt-4o}"
        read -rp "Reviewer API key: " R_KEY
        REVIEWER_CMD="anvil-review-api"
        REVIEWER_EXTRA="ANVIL_REVIEWER_URL=\"${R_URL}\"
ANVIL_REVIEWER_MODEL=\"${R_MODEL}\"
ANVIL_REVIEWER_API_KEY=\"${R_KEY}\""
        ;;
    3)
        read -rp "Reviewer LLM URL [http://localhost:11434/v1]: " R_URL
        R_URL="${R_URL:-http://localhost:11434/v1}"
        read -rp "Reviewer model [auto]: " R_MODEL
        R_MODEL="${R_MODEL:-auto}"
        REVIEWER_CMD="anvil-review-local"
        REVIEWER_EXTRA="ANVIL_REVIEWER_URL=\"${R_URL}\"
ANVIL_REVIEWER_MODEL=\"${R_MODEL}\""
        ;;
    4) REVIEWER_CMD="true" ;;
    *) echo "Invalid choice"; exit 1 ;;
esac

# ── Install scripts ─────────────────────────────────
echo ""
echo "Installing scripts to ${BIN_DIR}/..."
mkdir -p "$BIN_DIR"

for script in anvil anvil-review anvil-review-api anvil-review-local anvil-plan anvil-plan-answers anvil-build; do
    if [ -f "${SCRIPT_DIR}/scripts/${script}" ]; then
        cp "${SCRIPT_DIR}/scripts/${script}" "${BIN_DIR}/${script}"
        chmod +x "${BIN_DIR}/${script}"
        echo "  ✓ ${script}"
    fi
done

# ── Write configs ───────────────────────────────────
echo ""

# Aider config
sed -e "s|__MODEL__|${MODEL}|" \
    -e "s|__API_BASE__|${API_BASE}|" \
    -e "s|__API_KEY__|${API_KEY}|" \
    "${SCRIPT_DIR}/templates/aider.conf.yml" > "$AIDER_CONF"
echo "  ✓ ${AIDER_CONF}"

# Anvil env
sed "s|__REVIEWER__|${REVIEWER_CMD}|" \
    "${SCRIPT_DIR}/templates/anvil.env" > "$ANVIL_ENV"

if [ -n "$REVIEWER_EXTRA" ]; then
    echo "" >> "$ANVIL_ENV"
    echo "$REVIEWER_EXTRA" >> "$ANVIL_ENV"
fi
echo "  ✓ ${ANVIL_ENV}"

# ── Verify PATH ─────────────────────────────────────
if [[ ":$PATH:" != *":${BIN_DIR}:"* ]]; then
    echo ""
    echo "  ⚠ ${BIN_DIR} is not in your PATH."
    echo "  Add to your shell profile:"
    echo "    export PATH=\"\${HOME}/.local/bin:\${PATH}\""
fi

echo ""
echo "  ─────────────────────────────────"
echo "  Done. Type 'anvil' in any git repo to start."
echo ""
INSTALLER
chmod +x install.sh
```

- [ ] **Step 2: Test installer runs (dry run)**

```bash
bash -n /home/workspace/workbench/forge/install.sh && echo "Syntax OK"
```

- [ ] **Step 3: Commit**

```bash
cd /home/workspace/workbench/forge
git add install.sh
git commit -m "feat: add interactive installer"
```

---

### Task 10: Write README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Create `README.md` with:
- One-line description: "Local LLM codes. Claude reviews. You ship."
- How it works (3-phase diagram)
- Quick start (4 commands)
- Configuration (model setup, reviewer options)
- Requirements (aider, git, optional Claude Code)
- How it saves tokens (comparison table)

The README should be concise — under 200 lines. Show the workflow, not the implementation.

- [ ] **Step 2: Commit**

```bash
cd /home/workspace/workbench/forge
git add README.md
git commit -m "docs: add README"
```

---

### Task 11: End-to-end test on M5

**Files:** None — this is a verification task.

- [ ] **Step 1: Run installer**

```bash
cd /home/workspace/workbench/forge
./install.sh
# Choose: llama.cpp, http://localhost:8081/v1, Qwen3-Coder-Next
# Choose: Claude Code CLI
```

- [ ] **Step 2: Test anvil launch**

```bash
cd /home/steamvibe/forge-test/clean
anvil
# Type: "say hello" — expect Hercules response
# Type: /exit
```

- [ ] **Step 3: Test anvil-review**

```bash
cd /home/steamvibe/forge-test/clean
echo 'print("test")' >> hello.py
git add hello.py && git commit -m "test change"
anvil-review hello.py
echo "Exit: $?"
git revert --no-edit HEAD
```

- [ ] **Step 4: Test full workflow**

```bash
cd /home/steamvibe/forge-test/clean
anvil
# Chat: "I want a CLI calculator that adds and subtracts two numbers"
# Chat back and forth with Hercules
# /run anvil-plan
# Review the plan
# /run anvil-build
# Watch Hercules code + Claude review
```

- [ ] **Step 5: Verify results**

Check that:
- Files were created
- Git log shows commits per task
- Claude review output visible in build log
- Calculator works when run

---

## Self-Review

**Spec coverage check:**
- ✓ Phase 1 (chat): Task 2 (anvil launcher)
- ✓ Phase 2 (plan): Tasks 5-6 (anvil-plan, anvil-plan-answers)
- ✓ Phase 3 (build+review): Tasks 3, 7 (anvil-review, anvil-build)
- ✓ Reviewer alternatives: Task 4 (API + local)
- ✓ Config: Task 8 (templates)
- ✓ Installer: Task 9
- ✓ Model flexibility: Handled by installer choices + aider config
- ✓ Token economics: Documented in spec, README covers it

**Placeholder scan:** No TBD/TODO. All scripts have full code. README is described but content specified.

**Type consistency:** Script names consistent throughout (anvil, anvil-review, anvil-plan, anvil-plan-answers, anvil-build). Config paths consistent (~/.anvil.env, ~/.aider.conf.yml, .anvil/plan.md).
