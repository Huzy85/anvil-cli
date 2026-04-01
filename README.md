# anvil

> v0.1.0 — First public release

**Build software with AI without burning through your subscription.**

Anvil uses a cheaper AI model to write your code while a smarter model (like Claude) checks every change before it goes in. You describe what you want, Anvil breaks it into steps, the coding model writes the code, and Claude catches any mistakes. You get the quality of a top-tier AI review without paying for every single line it writes.

The coding model can be anything: a free local model running on your own machine, a cheap API like Deepseek or OpenRouter, or even Claude itself if you prefer. Claude handles the planning and review — the part where quality actually matters. Everything else goes to whichever model costs you least.

It works on top of [aider](https://github.com/paul-gauthier/aider), an open-source coding tool. If you've been vibe coding with Claude or ChatGPT and watching your credits disappear, Anvil keeps your paid model for the work that matters and lets a cheaper model handle the bulk writing.

![Anvil install demo](docs/anvil-install.gif)

---

## What is this, in plain English?

You know how when you use Claude or ChatGPT to write code, it burns through your credits fast? That's because every line of code it writes costs money.

Anvil fixes that. It uses a cheaper AI model to do the actual writing, then only calls Claude to check the work — like having a junior developer write the code and a senior developer review it. You get good quality output without paying for every keystroke.

The "cheaper model" can be a free AI running on your own machine (no internet needed), a low-cost API like Deepseek or OpenRouter, or Claude Haiku. You pick. Claude just does the reviews.

Here's what happens when you use it:

1. You describe what you want to build in plain English
2. Claude turns that into a step-by-step plan
3. The cheaper coding model writes the code, one step at a time
4. Claude checks each piece before it goes in — if something's wrong, it asks the coding model to fix it

The result: a 4-component project that would normally use ~8,000 Claude tokens uses ~5,800 instead. About 30% cheaper, with the same quality checks.

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
┌─────────────────────────────────────┐
│  Aider (terminal)                   │
│                                     │
│  Chat ──► Coding model              │
│  Plan ──► Claude Code CLI           │
│  Code ──► Coding model + Claude     │
└─────────────────────────────────────┘
```

Three phases:

1. **Chat** — brainstorm with your coding model via aider
2. **Plan** — `/run anvil-plan` sends the conversation to Claude for a structured task plan
3. **Build** — `/run anvil-build` feeds tasks to aider one by one; Claude reviews each edit automatically

## Requirements

- [aider](https://aider.chat) — `pip install aider-chat`
- git
- A coding model: local (Ollama, llama.cpp, LM Studio) or API (Deepseek, OpenRouter, Anthropic, etc.)
- Claude Code CLI — `npm install -g @anthropic-ai/claude-code`

## What happens during review

When aider edits a file, `anvil-review` runs automatically:

1. **Review 1-2:** Claude reads the diff and the actual files. If the code has real bugs (crashes, wrong logic, security issues), it says REJECTED with exact fixes. Style issues are ignored.
2. **Review 3 (escalation):** If the coding model can't fix the issues after 2 rejections, Claude takes over and fixes the code directly using file editing tools.
3. **Timeout/rate-limit:** If Claude is unavailable, the build stops and tells you. You decide whether to continue, retry, or switch to a different reviewer.
4. **Max turns:** Claude has up to 6 turns per review (configurable via `ANVIL_REVIEW_MAX_TURNS` in `~/.anvil.env`). If it hits the limit without concluding, the review is flagged as incomplete rather than silently approved.

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
| `anvil-review` | Claude Code reviewer (aider lint hook) |
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

## Token efficiency

Anvil uses significantly fewer tokens than an equivalent direct Claude Code session because each review sends a small, focused context rather than accumulating the full conversation history.

Benchmark (LogFlow — 4 components, 4 languages, all tests passing):

| Approach | Tokens |
|----------|--------|
| Anvil (9 incremental reviews) | ~5,800 |
| Direct Claude Code session (estimated) | ~8,300 |
| Saving | ~2,500 (~30%) |

The gap grows with project size. For a 4-file project it is roughly 30%. For larger projects the context bloat compounds harder, so the saving increases.

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
| Chatting and brainstorming | Your coding model | Depends on your model choice |
| Writing code | Your coding model | Depends on your model choice |
| Fixing rejected code | Your coding model | Depends on your model choice |
| Planning (once per project) | Claude / Gemini / Codex | Yes |
| Reviewing each edit | Claude / Gemini / Codex | Yes |

If you use a free local model or a cheap API for coding, the bulk of the work costs you nothing or very little. You only pay full price for planning and review.

## What this is not

- Not a fork of aider — just scripts and config on top of it
- Not an Anthropic API wrapper — uses Claude Code CLI with your existing subscription
- Not a custom terminal UI — aider is the interface

## License

Apache-2.0
