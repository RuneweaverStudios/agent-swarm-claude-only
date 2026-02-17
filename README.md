# Agent Swarm (Claude Only) | OpenClaw Skill

> IMPORTANT: ANTHROPIC API KEY IS REQUIRED
>
> Agent Swarm (Claude Only) only supports `anthropic/...` models and requires a configured Anthropic API key in OpenClaw.
> Without Anthropic API key, subagent delegation and `sessions_spawn` routing will fail.

**LLM routing and subagent delegation using Claude models only.** Routes each task to the right Claude model (Haiku, Sonnet, Opus), spawns subagents, and reduces API costs by using cheaper models for simple tasks. **Parallel tasks:** one message can spawn multiple subagents at once (e.g. "fix the bug and write a poem" → code + creative in parallel).

**v1.0.0 — Claude-only routing for Anthropic API key users.** Efficiently routes tasks to Claude Haiku (fast), Claude Sonnet (code/quality), and Claude Opus (complex). **Source:** Based on agent-swarm skill, optimized for Claude models.

Agent Swarm (Claude Only) | OpenClaw Skill routes your OpenClaw tasks to the best Claude model for the job and delegates work to subagents. You save API costs (orchestrator stays on Claude Haiku; only the task runs on the matched model) and get better results—Claude Sonnet for code, Claude Opus for complex tasks.

## Why Agent Swarm (Claude Only)

Perfect for users who have their Anthropic API key configured as their default language model in OpenClaw. Instead of using OpenRouter with multiple providers, this skill efficiently routes tasks within the Claude model family:

- **Claude Haiku** for fast, simple tasks (monitoring, checks, summaries)
- **Claude Sonnet** for code generation and quality work
- **Claude Opus** for the most complex, architectural tasks

With Claude-only routing, you get efficient cost management while staying within the Anthropic ecosystem.

## Requirements (critical)

**Platform Configuration Required:**
- **Anthropic API key**: Must be configured in OpenClaw platform settings (not provided by this skill)
- **OPENCLAW_HOME** (optional): Environment variable pointing to OpenClaw workspace root. If not set, defaults to `~/.openclaw`
- **openclaw.json access**: The router reads `tools.exec.host` and `tools.exec.node` from `openclaw.json` (located at `$OPENCLAW_HOME/openclaw.json` or `~/.openclaw/openclaw.json`). Only these two fields are accessed; no gateway secrets or API keys are read.

**Model Requirements:**
- **Anthropic API key is mandatory** — All model delegation uses Anthropic (`anthropic/...` prefix). Configure OpenClaw with an Anthropic API key.
- If Anthropic API key is not configured in OpenClaw, delegation will fail

## Default behavior

**Session default / orchestrator:** Claude Haiku (`anthropic/claude-3-haiku`) — fast, cheap, reliable at tool-calling.

The router delegates tasks to tier-specific sub-agents (Sonnet for code, Opus for complex, etc.) via `sessions_spawn`. Simple tasks (check, status, list) down-route to Claude Haiku.

---

## Orchestrator flow (task delegation)

The **main agent (Claude Haiku)** does not do user tasks itself. For every user **task** (code, research, write, build, etc.):

1. Run Agent Swarm router: `python scripts/router.py spawn --json "<user message>"` and parse the JSON.
2. Call **sessions_spawn** with the `task` and `model` from the router output (use the exact `model` value).
3. Forward the sub-agent's result to the user.

**Example:**
```
router: {"task":"write a poem","model":"anthropic/claude-3.5-sonnet","sessionTarget":"isolated"}
→ sessions_spawn(task="write a poem", model="anthropic/claude-3.5-sonnet", sessionTarget="isolated")
→ Forward Claude Sonnet's poem to user. Say "Using: Claude Sonnet 3.5".
```

**Exception:** Meta-questions ("what model are you?") you answer yourself.

### Parallel tasks

For one message with multiple tasks, use **`spawn --json --multi "<message>"`**. The router splits on *and*, *then*, *;*, and *also*, classifies each part, and returns `{"parallel": true, "spawns": [{task, model, sessionTarget}, ...], "count": N}`. The orchestrator can then call `sessions_spawn` for each entry and run them in parallel; use subagent-tracker to see progress.

**Example:** `spawn --json --multi "fix the login bug and write a short poem"` → two spawns (e.g. Claude Sonnet for code, Claude Sonnet for poem).

---

## Quick start

```bash
# Install the skill
cd ~/.openclaw/workspace/skills
git clone <repository-url> agent-swarm-claude-only

python scripts/router.py default
python scripts/router.py classify "your task description"
```

---

## Features

- **Orchestrator** — Claude Haiku delegates to tier-specific sub-agents via `sessions_spawn`
- **Parallel tasks** — `spawn --json --multi "task A and task B"` splits on *and* / *then* / *;* and returns an array of spawn params; orchestrator can spawn all and run them in parallel
- 7 tiers: FAST, REASONING, CREATIVE, RESEARCH, CODE, QUALITY, VISION
- All models via Anthropic API (single API key)
- Config-driven: `config.json` for models and routing rules
- **Security-focused** — Input validation, config patch whitelisting

---

## Default agents (edit in config.json)

All defaults are in **`config.json`**. Edit these two places to change which model runs for each tier:

| What to edit | Key in config.json | Purpose |
|--------------|--------------------|---------|
| Session default / orchestrator | `default_model` | Model for new sessions and the main agent |
| Per-tier primary | `routing_rules.<TIER>.primary` | Model used when a task matches that tier (FAST, CODE, CREATIVE, etc.) |

Example: to make CODE use a different model, edit `routing_rules.CODE.primary`. Fallbacks are in `routing_rules.<TIER>.fallback`. The router loads this file from the skill root (parent of `scripts/`).

---

## Models

| Tier | Model | Cost/M (in/out) |
|------|-------|-----------------|
| **Default / orchestrator** | Claude Haiku | $0.25 / $1.25 |
| FAST | Claude Haiku | $0.25 / $1.25 |
| REASONING | Claude Sonnet 3.5 | $3.00 / $15.00 |
| CREATIVE | Claude Sonnet 3.5 | $3.00 / $15.00 |
| RESEARCH | Claude Haiku | $0.25 / $1.25 |
| CODE | Claude Sonnet 3.5 | $3.00 / $15.00 |
| QUALITY | Claude Sonnet 4 | $3.00 / $15.00 |
| VISION | Claude Sonnet 3.5 | $3.00 / $15.00 |

**Fallbacks:** FAST → Claude Haiku; QUALITY → Claude Sonnet 4, Claude Opus; CODE → Claude Sonnet 3.5, Claude Sonnet 4.

---

## CLI usage

```bash
python scripts/router.py default                          # Show default model
python scripts/router.py classify "fix lint errors"        # Classify → tier + model
python scripts/router.py score "build a React auth system" # Detailed scoring
python scripts/router.py cost "design a landing page"      # Cost estimate
python scripts/router.py spawn "research best LLMs"        # Spawn params (human)
python scripts/router.py spawn --json "research best LLMs" # JSON for sessions_spawn
python scripts/router.py spawn --json --multi "fix bug and write poem" # Parallel tasks → array of spawns
python scripts/router.py models                            # List all models
```

---

## In-code usage

```python
from scripts.router import ClaudeRouter

router = ClaudeRouter()

default = router.get_default_model()
tier = router.classify_task("check server status")        # → "FAST"
result = router.recommend_model("build auth system")       # → {tier, model, fallback, reasoning}
spawn = router.spawn_agent("fix this bug", label="bugfix") # → {params: {task, model, sessionTarget}}
cost = router.estimate_cost("design landing page")         # → {tier, model, cost, currency}
```

---

## Tier detection

| Tier | Example keywords |
|------|------------------|
| **FAST** | check, get, list, show, status, monitor, fetch, simple |
| **REASONING** | prove, logic, analyze, derive, math, step by step |
| **CREATIVE** | creative, write, story, design, UI, UX, frontend, website |
| **RESEARCH** | research, find, search, lookup, web, information |
| **CODE** | code, function, debug, fix, implement, refactor, test, React, JWT |
| **QUALITY** | complex, architecture, design, system, comprehensive |
| **VISION** | image, picture, photo, screenshot, visual |

- **Vision** has priority: if vision keywords are present, task is VISION regardless of other keywords.
- **Agentic** tasks (multi-step) are bumped to at least CODE.

---

## Security

### Input Validation

The router validates and sanitizes all inputs to prevent injection attacks:

- **Task strings**: Validated for length (max 10KB), null bytes, and suspicious patterns
- **Config patches**: Only allows modifications to `tools.exec.host` and `tools.exec.node` (whitelist approach)
- **Labels**: Validated for length and null bytes

### Safe Execution (Critical for Orchestrators)

**When calling `router.py` from orchestrator code, always use `subprocess` with a list of arguments, never shell string interpolation:**

```python
# ✅ SAFE: Use subprocess with list arguments
import subprocess
result = subprocess.run(
    ["python3", "/path/to/router.py", "spawn", "--json", user_message],
    capture_output=True,
    text=True
)

# ❌ UNSAFE: Shell string interpolation (vulnerable to injection)
import os
os.system(f'python3 router.py spawn --json "{user_message}"')  # DON'T DO THIS
```

The router uses Python's `argparse`, which safely handles arguments when passed as a list. Shell string interpolation is vulnerable to command injection if the user message contains shell metacharacters (`;`, `|`, `&`, `$()`, etc.).

### Config Patch Safety

The `recommended_config_patch` only modifies safe fields:
- `tools.exec.host` (must be 'sandbox' or 'node')
- `tools.exec.node` (only when host is 'node')

All config patches are validated before being returned. The orchestrator should validate patches again before applying them to `openclaw.json`.

### Prompt Injection Mitigation

Task strings are passed to `sessions_spawn` and then to sub-agents. While the router validates input format, prompt injection protection is primarily the responsibility of:
1. The orchestrator (validating task strings)
2. The sub-agent LLM (resisting prompt injection)
3. The OpenClaw platform (sanitizing `sessions_spawn` inputs)

### File Access Scope

The router reads `openclaw.json` **only** to inspect `tools.exec.host` and `tools.exec.node` configuration. This is necessary to determine the execution environment for spawned sub-agents.

**Important:**
- The router **does not** read gateway secrets, API keys, or any other sensitive configuration
- Only `tools.exec.host` and `tools.exec.node` are accessed
- No data is written to `openclaw.json` except via validated config patches (whitelisted to `tools.exec.*` only)
- The router does not persist, upload, or transmit any tokens or credentials
- The phrase "saves tokens" in documentation refers to **API cost savings** (using cheaper models for simple tasks), not token storage or collection

---

## Configuration

- **`config.json`** — Model list and `routing_rules` per tier; `default_model` (e.g. `anthropic/claude-3-haiku`) for session default and orchestrator.
- Router loads `config.json` from the parent of `scripts/` (skill root).

---

## License / author

Austin. Part of the OpenClaw skills ecosystem.
