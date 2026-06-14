# Repository Summary: Ponytail

## Purpose

Ponytail is a plugin/ruleset system that injects a "lazy senior developer" mode into AI coding agents. It enforces a minimalist coding philosophy by requiring agents to evaluate code through a decision ladder before writing any implementation. The goal is to dramatically reduce code bloat, complexity, and cost while improving speed and reliability.

## Technology Stack

- **Language**: JavaScript (CommonJS for core logic, ES modules for OpenCode plugin)
- **Package manager**: npm
- **Test framework**: Node.js native test runner
- **Target platforms**: Claude Code, Codex, Pi, OpenCode, Gemini CLI, Cline, Windsurf, Cursor, Kiro, GitHub Copilot CLI
- **Configuration**: JSON (user config), TOML (command definitions), YAML (workflows)
- **Benchmarking**: promptfoo

## Main Runtime Model

Ponytail is a non-executable plugin system: it has no independent runtime. Instead, it deploys:

1. **Rule files** (`AGENTS.md`, `.cursor/rules/*.mdc`, etc.) that agents read as context
2. **Hooks** that fire on agent lifecycle events (session start, user input) to inject/update ruleset context
3. **Skills** (SKILL.md files) that define behaviors like `/ponytail`, `/ponytail-review`, `/ponytail-audit`
4. **Extensions** that register commands and integrate into agent harnesses (Claude Code plugin, OpenCode plugin, Pi extension)

There is no process to run, no server to start. Installation copies files to agent config directories; usage is a command like `/ponytail lite` or `@ponytail` (agent-dependent syntax).

## Main External Services / Dependencies

- **Benchmarking**: promptfoo (evaluates agent behavior)
- **LLM providers**: Claude (Anthropic), Gemini (Google) — used only during benchmarks, not at runtime
- **Distribution**: GitHub (releases)

No external runtime services required. Configuration is fully local.

## Main Entry Points

- **Agent installations**: Platform-specific, e.g., `/plugin install` in Claude Code
- **User commands**: `/ponytail <mode>`, `/ponytail-review`, `/ponytail-audit`, `/ponytail-help` (syntax varies by agent platform)
- **Lifecycle hooks**: Session start, user input (managed by agent harness)

## Architecture Summary

Ponytail centralizes its decision logic in a single ruleset (the "ladder": 6 steps for evaluating code before writing it) and distributes it as:

1. **Shared core modules** (`hooks/ponytail-*.js`): Mode resolution, instruction generation, state management
2. **Platform-specific adapters**:
   - Claude Code: hook-based system injecting rules on SessionStart
   - Codex: JSON hook configuration
   - Pi: ES module extension
   - OpenCode: ES module plugin with command handlers
   - Gemini CLI: rule file + SKILL.md
   - Others (Cursor, Windsurf, Cline, Kiro): copy-paste rule files

3. **Skill definitions** (`skills/ponytail/*.md`): The actual ruleset, mode-filtered for intensity levels (lite/full/ultra)
4. **Tests** (`.test.js` files): Validate hook behavior, mode switching, and state persistence

All agents read the same Markdown source of truth for rules, ensuring consistency across platforms. Mode persistence is agent-specific (flag files in config directories) but the resolution logic is shared (`ponytail-config.js`).

## Read These First

To understand how Ponytail works, start with:

1. **`README.md`** — High-level pitch, install instructions, example results
2. **`AGENTS.md`** — The core ruleset (primary source for all agent configurations)
3. **`skills/ponytail/SKILL.md`** — Detailed ruleset with intensity levels; read this for understanding what "lazy" means
4. **`hooks/ponytail-config.js`** — Mode resolution: how default mode is selected (env var → config file → 'full')
5. **`hooks/ponytail-activate.js`** — Session lifecycle: shows how rules are injected on session start
6. **`tests/hooks.test.js`** — Functional tests demonstrating expected behavior (mode persistence, config loading)
