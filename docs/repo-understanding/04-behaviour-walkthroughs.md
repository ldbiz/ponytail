# Behaviour Walkthroughs

## 1. User Installs Ponytail and Starts a Session

**Trigger**: User follows install instructions for their agent platform (e.g., `/plugin marketplace add DietrichGebert/ponytail` in Claude Code)

**Files involved**:
- Platform-specific install manifest (e.g., `.claude-plugin/plugin.json`, `.agents/plugins/marketplace.json`)
- `hooks/ponytail-activate.js` (activation logic)
- `hooks/ponytail-config.js` (mode resolution)
- `skills/ponytail/SKILL.md` (ruleset source)

**Flow**:

1. Agent harness copies ponytail files to its config directory
2. User starts a new session or existing session triggers SessionStart event
3. `ponytail-activate.js` hook runs:
   - Resolves default mode (env var → config file → 'full')
   - Writes mode to platform-specific flag file
   - Reads `SKILL.md`
   - Generates instructions via `ponytail-instructions.js`
   - Outputs instructions in platform-specific format
4. Agent's LLM context now includes ponytail rules
5. User sees a status message: "PONYTAIL:FULL" (statusline badge, if configured) or inline notification

**Inputs**:
- `PONYTAIL_DEFAULT_MODE` environment variable (optional)
- `~/.config/ponytail/config.json` (optional)

**Outputs**:
- Mode flag file written (e.g., `~/.claude/.ponytail-active`)
- Rules injected into agent context
- User notified of active mode

**External calls**: None (fully local)

**Tests**: `tests/hooks.test.js` — validates SessionStart hook execution, mode resolution, flag writing, and instruction injection

---

## 2. User Changes Intensity Level Mid-Session

**Trigger**: User types `/ponytail ultra` (or `/ponytail lite`, `/ponytail full`, `/ponytail off`)

**Files involved**:
- `hooks/ponytail-mode-tracker.js` (detects command)
- `hooks/ponytail-config.js` (validates mode)
- `hooks/ponytail-runtime.js` (persists to flag file)

**Flow**:

1. User submits a message containing `/ponytail ultra`
2. UserPromptSubmit hook fires (`ponytail-mode-tracker.js`)
3. Hook parses user input to detect ponytail command
4. If match found: extract requested mode, validate it against VALID_MODES
5. If valid: write mode to flag file
6. On next user message: SessionStart equivalent fires, reads new flag, injects new instructions
7. Mode change is now active for subsequent turns

**Inputs**:
- User command: `/ponytail <level>` where `<level>` ∈ {lite, full, ultra, off}

**Outputs**:
- Mode flag file updated
- On next turn: new instructions injected matching new level

**External calls**: None

**Side effects**: All subsequent agent responses use the new intensity level until changed again

**Tests**: `tests/hooks.test.js` — validates command parsing, mode validation, and persistence across turns

---

## 3. Agent Injects Rules into Every Turn (OpenCode-Specific)

**Trigger**: User invokes OpenCode (no installation step; uses plugin from this repo or installed version)

**Files involved**:
- `.opencode/plugins/ponytail.mjs` (plugin implementation)
- `hooks/ponytail-config.js` (mode resolution)
- `hooks/ponytail-instructions.js` (instruction generation)

**Flow**:

1. OpenCode starts a chat session
2. Plugin's `experimental.chat.system.transform` handler fires on every turn
3. Handler reads current mode from flag file (`~/.config/opencode/.ponytail-active`)
4. Mode is resolved using shared `ponytail-config.js` logic
5. Instructions are generated using shared `ponytail-instructions.js`
6. Instructions are appended to the system prompt
7. LLM responds with rules active

**Inputs**:
- `~/.config/opencode/.ponytail-active` (flag file, if exists)
- `PONYTAIL_DEFAULT_MODE` environment variable (if set)

**Outputs**:
- System prompt augmented with ponytail rules (every turn)

**External calls**: None

**Side effects**: Rules are always active; no separate session setup needed

**Tests**: `tests/opencode-plugin.test.js` — validates plugin loads, transform handler runs, instructions are appended

---

## 4. User Requests Rule-Sync Validation (Development Workflow)

**Trigger**: Developer runs `node scripts/check-rule-copies.js` before committing

**Files involved**:
- `scripts/check-rule-copies.js` (validator)
- `AGENTS.md` (primary ruleset)
- `.cursor/rules/ponytail.mdc` and other platform-specific rule files

**Flow**:

1. Script reads `AGENTS.md` (primary source of truth)
2. Reads each platform-specific rule file (`.cursor/`, `.windsurf/`, `.clinerules/`, etc.)
3. Compares content: normalizes whitespace/formatting, extracts core ruleset
4. If any platform file differs from primary: report discrepancy and exit with error
5. Developer updates out-of-sync files or fixes the issue
6. Script re-runs and passes

**Inputs**:
- Rule files in multiple locations

**Outputs**:
- Exit 0 if all files match
- Exit non-zero with error message if files are out of sync

**External calls**: None (local file comparison)

**Side effects**: Prevents inconsistent rules across platforms from being committed

**Tests**: Implicitly tested via CI workflows; explicit test coverage not present

---

## 5. Benchmarking Suite Compares Ponytail vs No-Skill Baseline

**Trigger**: Developer runs `npx promptfoo eval -c benchmarks/promptfooconfig.yaml` in benchmarks directory

**Files involved**:
- `benchmarks/promptfooconfig.yaml` (promptfoo configuration)
- `benchmarks/prompts.json` (test cases: email validator, debounce, CSV sum, etc.)
- `benchmarks/arms/` (baseline, caveman skill, ponytail skill definitions)
- `skills/ponytail/SKILL.md` (source of ponytail arm)

**Flow**:

1. promptfoo reads configuration and test cases
2. For each test case and each arm (baseline, caveman, ponytail):
   - LLM is prompted with the test task and the arm's rules (or no rules for baseline)
   - LLM generates code
   - Outputs are evaluated: code size, cost, latency
3. Results aggregated and compared across arms and models (Haiku, Sonnet, Opus)
4. Report generated showing ponytail's improvements

**Inputs**:
- `ANTHROPIC_API_KEY` environment variable (for Claude API access)
- Test prompts (standard tasks like "write email validator")
- Arm definitions (SKILL.md for ponytail)

**Outputs**:
- Markdown report with metrics: lines of code, cost (in $), latency (in sec)
- Comparison showing ponytail's advantages

**External calls**: Anthropic API (LLM requests for each test × model)

**Side effects**: Generates benchmark results; no system state changes

**Tests**: No automated tests; results are manual review

---

## Summary: Behaviours Tested

| Behaviour | Coverage |
|---|---|
| Session startup and mode resolution | `tests/hooks.test.js` |
| Mode switching and persistence | `tests/hooks.test.js`, `tests/hooks-windows.test.js` |
| Instruction generation and filtering | `tests/hooks.test.js` (validation of instruction text) |
| Config file loading (JSON) | `tests/hooks.test.js` |
| Platform-specific hook output formatting | `tests/hooks.test.js` (JSON for Codex, plain text for Claude) |
| OpenCode plugin system integration | `tests/opencode-plugin.test.js` |
| Gemini extension integration | `tests/gemini-extension.test.js` |
| Rule file synchronization | Manual (no automated test; caught by visual review) |
| Benchmarking workflow | Manual (results reviewed, metrics recorded) |
