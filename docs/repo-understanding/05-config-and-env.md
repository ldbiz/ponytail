# Configuration and Environment

## Environment Variables

| Variable | Purpose | Default | Example | Required? |
|---|---|---|---|---|
| `PONYTAIL_DEFAULT_MODE` | Default intensity level on session startup | `full` | `ultra` | No |
| `XDG_CONFIG_HOME` | Linux/macOS config directory (used if set) | `~/.config` | `/custom/config` | No |
| `APPDATA` | Windows app data directory | `~/AppData/Roaming` | `C:\Users\User\AppData` | No |
| `PLUGIN_DATA` | Codex plugin data directory | (not set) | `/path/to/plugin/data` | Internal only |
| `ANTHROPIC_API_KEY` | Claude API key for benchmarking | (not set) | `sk-ant-...` | Only for benchmarks |

## Config Files

### User Configuration: `~/.config/ponytail/config.json`

**Purpose**: User-level default intensity mode, persisted across sessions and agents

**Platform locations**:
- Linux/macOS: `$XDG_CONFIG_HOME/ponytail/config.json` (default `~/.config/ponytail/config.json`)
- Windows: `%APPDATA%\ponytail\config.json` (default `~/AppData/Roaming/ponytail/config.json`)

**Format**: JSON

**Example**:
```json
{
  "defaultMode": "ultra"
}
```

**Fields**:
- `defaultMode`: String, one of {off, lite, full, ultra, review}. If invalid or absent, defaults to 'full'.

**Resolution order** (highest to lowest priority):
1. `PONYTAIL_DEFAULT_MODE` environment variable
2. `config.json` `defaultMode` field
3. Hardcoded `DEFAULT_MODE = 'full'`

**Lifetime**: Persistent until user modifies or deletes the file

### Hooks Configuration: `hooks/hooks.json`

**Purpose**: Registers lifecycle hooks with the Claude Code agent

**Format**: JSON, following Claude Code hook schema

**Example**:
```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup|resume|clear|compact",
      "hooks": [{
        "type": "command",
        "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-activate.js\"",
        "timeout": 5,
        "statusMessage": "Loading ponytail mode..."
      }]
    }],
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-mode-tracker.js\"",
        "timeout": 5,
        "statusMessage": "Tracking ponytail mode..."
      }]
    }]
  }
}
```

**Fields**:
- `SessionStart`: Fires when user starts a session (startup, resume, clear, compact)
- `UserPromptSubmit`: Fires when user submits a message
- Each hook has `type` (command), `command` (shell command or Node script path), `timeout` (seconds)

**Lifetime**: Persistent while Claude Code plugin is installed

### OpenCode Configuration: `opencode.json`

**Purpose**: Registers the OpenCode plugin

**Example**:
```json
{
  "plugin": ["./.opencode/plugins/ponytail.mjs"]
}
```

**Activation**: When OpenCode runs from the ponytail repo or from an installed copy, the plugin is auto-loaded and active every turn

### Command Definitions: `commands/*.toml`

**Purpose**: Registers user-facing commands (e.g., `/ponytail`) for agents that support structured commands

**Example**: `commands/ponytail.toml`
```toml
description = "Switch ponytail intensity level (lite/full/ultra/off)"
prompt = "Switch to ponytail {{args}} mode. If no level specified, use full. ..."
```

**Similar files**: `commands/ponytail-review.toml`, `commands/ponytail-audit.toml`

**Activation**: Loaded by Codex when ponytail plugin is active

## Mode State Files

| Agent | Path | Format | Example | Lifetime |
|---|---|---|---|---|
| Claude Code | `~/.claude/.ponytail-active` | Plain text (mode name) | `ultra` | Until changed or session ends |
| Codex | `${PLUGIN_DATA}/.ponytail-active` | Plain text | `lite` | Until changed or plugin unloaded |
| OpenCode | `~/.config/opencode/.ponytail-active` | Plain text | `full` | Until changed or agent restart |

**Purpose**: Persists current mode so the next turn/session uses the same level

**Writing**: Happens when mode is set (on SessionStart or via `/ponytail <level>` command)

**Reading**: Hook/plugin reads on activation to determine which intensity level to use

## Mode Defaults

| Level | When Used | Intensity | Use Case |
|---|---|---|---|
| **full** | Default if nothing else is set | 🟡 Balanced | General development; YAGNI + stdlib first |
| **lite** | When user types `/ponytail lite` | 🟢 Gentle | Offers lazy alternatives but ships what's asked |
| **ultra** | When user types `/ponytail ultra` | 🔴 Aggressive | Ship one-liners, challenge requirements |
| **off** | When user types `/ponytail off` | ⚪ Disabled | Disables ponytail; agent behaves normally |

## Benchmarking Configuration

**File**: `benchmarks/promptfooconfig.yaml`

**Purpose**: Defines test cases, arms (skill variants), and models for performance comparison

**Key sections**:
- `providers`: LLM models to test (Haiku, Sonnet, Opus, etc.)
- `testCases`: Standard tasks (email validator, debounce, CSV sum, countdown timer, rate limiter)
- `prompts`: Test prompts and expected outputs
- `results`: Metrics to capture (cost, latency, token counts)

**Requires**: `ANTHROPIC_API_KEY` environment variable set

**Run**: `npx promptfoo eval -c benchmarks/promptfooconfig.yaml`

**Output**: `benchmarks/results/` with markdown reports showing lines of code, cost ($), and latency (sec) by arm and model

## Environment File: `.env.example`

**Purpose**: Template for benchmarking environment variables

**Contents**:
```
ANTHROPIC_API_KEY=sk-ant-...
```

**Activation**: Copy to `.env` (gitignored); promptfoo auto-loads `ANTHROPIC_API_KEY`

**Notes**: No environment files are checked in; `.env` is user-specific and never committed

## Config Resolution Order (High-Level)

```
Determine effective mode for this session:

  Environment Variable
    PONYTAIL_DEFAULT_MODE=ultra
        ↓ (if set, use it)
  Config File
    ~/.config/ponytail/config.json { "defaultMode": "ultra" }
        ↓ (if file exists and defaultMode is set, use it)
  Hardcoded Default
    'full'
        ↓ (always falls back to this)
```

Implementation: `hooks/ponytail-config.js` `getDefaultMode()`

## Secrets and Credentials

**Expected**: None in static configuration or repo

**Benchmarking only**: `ANTHROPIC_API_KEY` is set by user in `.env` (user-local, not committed)

## Platform-Specific Considerations

### Windows

- Config paths use `%APPDATA%` instead of `~/.config`
- Hook scripts have `commandWindows` variant (PowerShell syntax)
- Flag files are written to `${PLUGIN_DATA}` for Codex (handles paths correctly)

### Linux/macOS

- Config paths use `$XDG_CONFIG_HOME` (default `~/.config`)
- Home directory is `~` (resolved by Node.js `os.homedir()`)
- No platform-specific differences in logic

## Running Tests

**Test files**: `tests/*.test.js`

**Framework**: Node.js native test runner (no external dependencies)

**Run all tests**:
```bash
node --test tests/*.test.js
```

**Run specific test**:
```bash
node tests/hooks.test.js
```

**Environment for tests**:
- Creates temporary directories (`os.tmpdir()`)
- Tests both Codex (`PLUGIN_DATA` set) and Claude Code (`PLUGIN_DATA` unset) modes
- Validates config file loading, mode switching, flag persistence

**Exit codes**:
- 0: All tests pass
- Non-zero: Test failure; check stderr for details
