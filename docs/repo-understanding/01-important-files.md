# Important Files and Directories

| File / Directory | Role | Why It Matters |
|---|---|---|
| **AGENTS.md** | Core ruleset | The primary rule source read by all agents; defines the 6-rung decision ladder and all behavior |
| **skills/ponytail/SKILL.md** | Skill definition + mode filters | The same content as AGENTS.md but with mode-specific sections (lite/full/ultra); used for instruction injection |
| **hooks/ponytail-config.js** | Mode resolution | Resolves effective mode from env var, config file, or default; used by all platforms |
| **hooks/ponytail-instructions.js** | Instruction builder | Reads SKILL.md, filters it by intensity level, and generates the context to inject into agents |
| **hooks/ponytail-activate.js** | Session lifecycle hook | Runs on agent SessionStart; writes mode flag, injects ruleset, detects missing statusline config |
| **hooks/ponytail-runtime.js** | Hook output / state management | Formats hook output (JSON for Codex, plain text for Claude Code); manages mode flag files |
| **pi-extension/index.js** | Pi harness integration | Implements Pi extension API; exports mode resolution and instruction builder functions |
| **.opencode/plugins/ponytail.mjs** | OpenCode plugin | ES module plugin; injects ruleset into system prompt every turn, persists mode via command handler |
| **scripts/check-rule-copies.js** | Rule sync validator | Ensures all platform-specific rule files (`.cursor/`, `.clinerules/`, etc.) stay in sync with AGENTS.md |
| **tests/hooks.test.js** | Hook behavior tests | Validates mode switching, state persistence, instruction generation, config file loading |
| **commands/ponytail.toml** | Codex command definition | Registers `/ponytail` command for Codex; similar TOML files exist for review and audit skills |
| **.cursor/rules/ponytail.mdc** | Cursor rule file | Platform-specific rule file copied from AGENTS.md; loaded by Cursor editor |
| **opencode.json** | OpenCode config | Registers the OpenCode plugin; when running OpenCode, add plugin entry to auto-load ponytail |
| **package.json** | Project metadata | Minimal npm package; registers Pi extension entry point; no meaningful dependencies |
| **.env.example** | Benchmark config | Placeholder for benchmarking (promptfoo) configuration; requires ANTHROPIC_API_KEY for running benchmarks |
| **benchmarks/** | Performance suite | promptfoo configuration and results; demonstrates code size / cost / latency improvements |

## Key Directories

| Directory | Purpose |
|---|---|
| **hooks/** | Node.js scripts that run on agent lifecycle events (SessionStart, UserPromptSubmit) |
| **skills/** | Skill definitions (SKILL.md files) for `/ponytail`, `/ponytail-review`, `/ponytail-audit`, `/ponytail-help` |
| **tests/** | Unit and integration tests for hooks, mode resolution, and config loading |
| **.cursor/, .windsurf/, .clinerules/, .kiro/**, etc. | Platform-specific rule/config files; sync'd from AGENTS.md via check-rule-copies.js |
| **commands/** | TOML command definitions for agents that support structured commands |
| **.github/**, **.agents/**, **.claude-plugin/**, **pi-extension/**, **.opencode/**, **gemini-extension.json** | Agent harness integration files (install manifests, marketplace listings, etc.) |
| **docs/** | Documentation (agent portability, repo understanding — this pack) |
| **examples/** | Sample code snippets demonstrating ponytail in action |
| **benchmarks/** | Performance evaluation: promptfoo config, result reports, test prompts |
