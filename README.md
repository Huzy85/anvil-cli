# anvil

> v0.1.0 — First public release

**A two-model pipeline: cheap coder writes, smart auditor checks.**

Anvil is an opinionated wrapper around [aider](https://github.com/paul-gauthier/aider). A cheaper AI model writes the code. A smarter model (Claude by default) plans the work and reviews the result. You pick which model plays each role.

The coder can be a free local model (Ollama, llama.cpp), a cheap cloud API (Deepseek, Kimi, OpenRouter), or Claude itself. The auditor is usually Claude Code CLI using your existing subscription.

**Your savings scale with your coder.** A coder that writes clean, test-passing code means the auditor only needs a quick confirmation, and you pay pennies instead of dollars. A weaker coder still gets the job done — the auditor steps in and fixes what needs fixing — you just save less. Either way, the work ships.

Pick the coder that fits your budget and the job gets done.

![Anvil install demo](docs/anvil-install.gif)

---

## What is this, in plain English?

Using Claude or ChatGPT to write code costs real money because every line it writes is a paid token. Anvil lets a cheaper model do the writing and uses Claude for the parts where quality actually matters: the plan and the final check.

Think of it as a junior developer writing code and a senior developer reviewing it. The junior is cheap or free. The senior is paid, but only reads the final result.

Here is what happens when you use it:

1. You chat with the cheaper model about what you want to build.
2. The cheaper model (or Claude, your choice) turns the conversation into a numbered task list.
3. The cheaper model writes each file.
4. Tests run. If they pass, the auditor takes a quick look and signs off (cheap). If they fail, the auditor reads the code and fixes the bugs (more expensive).

The cheaper model can be a free local AI, a low-cost API like Deepseek, or Claude Haiku. Pick whichever matches your budget and quality bar.

---

## Never used a terminal before?

The terminal (also called the command line) is a text-based way to run programs on your computer. It looks intimidating but you only need to know three things: how to open it, how to type a command, and how to press Enter.

**On a Mac:** Press `Cmd + Space`, type `Terminal`, press Enter.

**On Linux:** Press `Ctrl + Alt + T`, or search for "Terminal" in your apps.

**On Windows:** Install [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) first (it takes about 10 minutes and Microsoft's instructions are clear), then open "Ubuntu" from your Start menu. That gives you a Linux terminal.

Once the terminal is open, you type commands and press Enter to run them. That's it. The install instructions below are just a list of commands to copy and paste, one at a time.

If a command asks for your password, type it (you won't see the characters appear — that's normal) and press Enter.

---

## What you need before installing

- A Mac, Linux, or Windows (via WSL) computer
- **[aider](https://aider.chat)** — the coding tool Anvil runs on top of. Install it with: `pip install aider-chat`
- **[Claude Code](https://claude.ai/code)** — the Claude command-line tool. Install it with: `npm install -g @anthropic-ai/claude-code` (requires a Claude subscription)
- **A coding model** — the AI that writes the code. Pick one:
  - **Free, runs on your machine:** [Ollama](https://ollama.com) — download and install like any app, no account needed
  - **Cheap API:** [Deepseek](https://platform.deepseek.com) or [OpenRouter](https://openrouter.ai) — sign up, add a few dollars of credit, get an API key
  - **No setup:** Use Claude for everything — costs more but nothing extra to install
- **Python 3** and `pip` — most computers already have these. Check by typing `python3 --version` in the terminal

Not sure if something is installed? Run `./install.sh` after cloning — it checks everything and tells you what's missing before doing anything.

---

## Install

Open a terminal and run these commands one at a time:

```bash
# 1. Install aider (the coding tool)
pip install aider-chat

# 2. Install Claude Code (the reviewer) — skip if you already have it
npm install -g @anthropic-ai/claude-code

# 3. Get Anvil
git clone https://github.com/Huzy85/anvil
cd anvil
./install.sh
```

The installer asks two questions: which coding model to use and which reviewer to use. Answer them and you're done.

Then go to any project folder and run:

```bash
anvil
```

---

## How it works (technical)

```
You
 │
 ▼
┌─────────────────────────────────────────┐
│  Phase 1 — Plan                         │
│    anvil-plan                           │
│    Coder (or auditor) writes task list  │
│                                         │
│  Phase 2 — Build                        │
│    anvil-build                          │
│    Coder writes each file via aider     │
│                                         │
│  Phase 3 — Review (single pass)         │
│    Tests run automatically              │
│    Pass: auditor stamps APPROVED (cheap)│
│    Fail: auditor reads + fixes (costly) │
└─────────────────────────────────────────┘
```

Three phases:

1. **Plan** — `anvil-plan` asks the coder for a numbered task list. Falls back to the auditor if the coder stalls.
2. **Build** — `anvil-build` feeds tasks to aider one at a time. The coder writes full file contents. No per-edit auditor call.
3. **Review** — one auditor call at the end. Pytest runs first. On green, the auditor returns a cheap APPROVED. On red, the auditor gets the failing output plus the source files and is expected to fix the bugs in place.

## Requirements

- [aider](https://aider.chat) — `pip install aider-chat`
- git
- A coding model: local (Ollama, llama.cpp, LM Studio) or API (Deepseek, OpenRouter, Anthropic, etc.)
- Claude Code CLI — `npm install -g @anthropic-ai/claude-code`

## What happens during review

After the coder has written every file, `anvil-build` runs `pytest -q` once and passes the result to the auditor:

1. **Tests pass** — the auditor gets only the test output and is asked to reply APPROVED. No file upload, 3 turns max. This is the cheap path (a few cents).
2. **Tests fail** — the auditor gets the failing output plus every source file and up to 15 turns to fix the bugs directly. This is the expensive path (roughly the cost of a direct session).
3. **Max turns hit** — the run ends with a warning. Nothing is silently approved.
4. **Auditor unavailable** — the build stops and tells you. You decide whether to retry or switch auditors.

This design keeps the auditor idle when the coder does its job, which is where the savings come from. When the coder needs help, the auditor is there to finish the work. Either way, the final code passes its tests.

## Troubleshooting

**"aider not found"** — Install aider: `pip install aider-chat` (or `pipx install aider-chat`)

**"No conversation history found"** — Chat with your coding model first before running `/run plan`. You need at least one exchange.

**"No plan found"** — Run `/run plan` before `/run build`. The plan lives in `.anvil/plan.md`.

**Review always approves** — Check `~/.anvil.env` — is `ANVIL_REVIEWER` set correctly? Try running `anvil-review yourfile.py` manually to see the output.

**aider crashes with thread errors** — You're inside a Claude Code session with a thread limit. Run from a plain terminal instead, or let `anvil-test-suite` auto-escape the cgroup.

**Model not responding** — Check your endpoint is running: `curl http://localhost:8080/v1/models` (or whatever your endpoint is).

## Installed scripts

All go to `~/.local/bin/`:

| Script | Purpose |
|--------|---------|
| `anvil` | Launch aider with your coding model + review config |
| `anvil-review` | Standalone reviewer, usable for ad-hoc file checks |
| `anvil-review-api` | API reviewer alternative (OpenAI-compatible) |
| `anvil-review-local` | Local model reviewer alternative |
| `anvil-plan` | Send aider conversation to Claude for planning |
| `anvil-plan-answers` | Follow-up for Claude's planning questions |
| `anvil-build` | Automated build loop — feeds tasks to aider |
| `anvil-test-suite` | Benchmark your coding model across 10 tasks |
| `anvil-test-suite-resume` | Resume a test suite run from the last completed task |

## Benchmarking

Run the test suite to measure how well your coding model performs:

```bash
anvil-test-suite
# Results written to ~/anvil-test-results.md
```

Runs 10 tasks (thread-safe queue, LRU cache, retry decorator, event emitter, CSV parser, rate limiter, state machine, JSON validator, promise chain, debounced callback). Each task is reviewed by Claude. Results show pass/fail, review count, and rejection details.

**Note:** Do not run `anvil-test-suite` from inside a Claude Code session. The cgroup that Claude Code runs in has a thread limit that interferes with aider. Run it from a plain terminal instead. The script will warn you if it detects this and auto-escape, but a fresh terminal is cleaner.

If a run is interrupted, use `anvil-test-suite-resume` to continue from the last completed task.

## How much does it save?

The answer depends entirely on your coder. Here is a real benchmark: a URL shortener spec (8 files, 29 tests), measured end-to-end with the cost shim tracking every paid call.

| Coder | Tests pass first try | Anvil cost | Direct Claude |
|-------|---------------------|------------|---------------|
| Hercules (Qwen3-Coder-Next, local, free) | No. 4 failing tests. | $0.49 | $0.44 |
| Same setup if Hercules had passed | Yes | ~$0.02 | $0.44 |

Two things to read here:

1. **The ceiling is high.** If your coder nails the spec, you pay cents for the APPROVED stamp instead of a full session. For this spec that would have been roughly a 95% saving.
2. **The floor is close to direct.** If the coder misses, the auditor does the fix-up work and you land around parity. You still get working code and you still paid less for the coder phase (free in this case).

The knob to turn is the coder's first-try accuracy. A smarter cloud coder like Deepseek or Kimi pushes more runs into the cheap lane. A free local model works for routine code (CRUD, boilerplate, standard library wrappers) where first-try accuracy is naturally high.

Anvil also scales better than raw Claude on very small tasks — the two-model overhead shows up as a fixed cost, so trivial jobs (single function, toy scripts) are not where this tool shines. It earns its keep on real specs.

## Configuration

Two config files, both written by the installer:

- `~/.aider.conf.yml` — model, endpoint, API key, lint settings
- `~/.anvil.env` — reviewer command, retry limits, diff size cap

Edit `~/.anvil.env` to switch reviewers without reinstalling:

```bash
# Claude Code CLI (recommended — full support)
ANVIL_REVIEWER="claude -p"

# Gemini CLI (experimental — see limitations below)
ANVIL_REVIEWER="gemini -p"

# OpenAI Codex CLI (experimental — see limitations below)
ANVIL_REVIEWER="codex exec"

# Or use an API
ANVIL_REVIEWER="anvil-review-api"
ANVIL_REVIEWER_URL="https://api.openai.com/v1"
ANVIL_REVIEWER_MODEL="gpt-4o"
ANVIL_REVIEWER_API_KEY="sk-..."

# Or use a local model as reviewer
ANVIL_REVIEWER="anvil-review-local"
ANVIL_REVIEWER_URL="http://localhost:11434/v1"
ANVIL_REVIEWER_MODEL="auto"
```

## Reviewer comparison

Claude Code CLI is the only fully tested reviewer. Gemini and Codex work for basic review but have not been tested end-to-end with Anvil.

| Feature | Claude Code CLI | Gemini CLI | Codex CLI | API / Local model |
|---------|----------------|------------|-----------|-----------------|
| Code review | Yes | Yes | Yes | Yes |
| Auto-fix escalation | Yes | No | Untested | No |
| Tool use in review | Yes | No (headless) | Untested | No |
| Tested with Anvil | Yes | No | No | Yes (API only) |
| Install | `npm i -g @anthropic-ai/claude-code` | `npm i -g @google/gemini-cli` | `npm i -g @openai/codex` | N/A |

**Limitations of non-Claude reviewers:**
- No auto-fix escalation. After max rejections, the build stops and asks you to fix the code manually.
- Gemini CLI blocks all file tools in headless mode, so it can only review diffs, not read source files for context.
- Codex CLI escalation support has not been verified. It may work but we haven't tested it.

## Model flexibility

The coding model is whatever aider supports. The installer handles config for:

- Ollama
- llama.cpp / LM Studio / vLLM (OpenAI-compatible)
- Deepseek API
- OpenRouter (any model)
- Anthropic API (Haiku/Sonnet)
- Any custom OpenAI-compatible endpoint

Override on the fly:

```bash
anvil --model openai/different-model
```

## Where your tokens go

| Activity | Who does it | Paid? |
|----------|------------|-------|
| Chatting and brainstorming | Coder | Coder rate |
| Planning (once per project) | Coder first, auditor as fallback | Coder rate, or auditor if coder stalls |
| Writing every file | Coder | Coder rate |
| Final review — tests pass | Auditor | Small auditor call (cents) |
| Final review — tests fail | Auditor | Full auditor call (dollars) |

Pick a free local model and the bulk of the pipeline is free. Pick a cheap API and it is pennies. The auditor is the only place you pay full rate, and only on the fix-up path.

## What this is not

- Not a fork of aider — just scripts and config on top of it
- Not an Anthropic API wrapper — uses Claude Code CLI with your existing subscription
- Not a custom terminal UI — aider is the interface

## License

Apache-2.0
