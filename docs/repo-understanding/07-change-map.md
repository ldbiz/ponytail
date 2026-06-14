# Change Map

A practical guide for where to look when making common changes to the ponytail codebase.

## If I Need to Change the Core Decision Ladder or Rules

**Change**: Add a new rung, modify existing steps, or reword the ladder logic

**Likely files to look at first**:
- `AGENTS.md` — Primary source of truth; every rung is documented here
- `skills/ponytail/SKILL.md` — Same content but with mode-specific filtering; read `AGENTS.md` first to understand the delta
- `.cursor/rules/ponytail.mdc` and other platform files — Should match `AGENTS.md`; validate with `scripts/check-rule-copies.js` after updating

**After editing AGENTS.md**:
1. Manually update `skills/ponytail/SKILL.md` (frontmatter + same content)
2. Run `node scripts/check-rule-copies.js` to find all platform files that need sync
3. Update each platform file to match
4. Run `node --test tests/*.test.js` to ensure instruction generation still works
5. Check README examples (`examples/` directory) to see if any need updating

**Caveats**:
- Changing the ladder fundamentally changes how all agents behave
- Test assumptions about instruction content may need updating
- Update examples/ to reflect the new rules so documentation stays accurate

---

## If I Need to Change Mode Levels or Intensity Filtering

**Change**: Modify lite/full/ultra behavior, add new level, or change filtering logic

**Likely files to look at first**:
- `hooks/ponytail-config.js` — Define new level in VALID_MODES and RUNTIME_MODES
- `skills/ponytail/SKILL.md` — Add mode-specific rows/examples keyed by the new level name
- `hooks/ponytail-instructions.js` `filterSkillBodyForMode()` — Adjust filtering logic if needed (currently filters by table row labels and example block labels)

**After editing**:
1. Update `AGENTS.md` if ladder itself changed; otherwise just SKILL.md
2. Run instructions through the filter: test that each level produces the expected subset of rules
3. Update `tests/hooks.test.js` to validate new mode if adding one
4. Test via: `node --test tests/hooks.test.js`

**Caveats**:
- SKILL.md filtering relies on Markdown table row labels and example section prefixes; changing those labels breaks filtering
- New modes must be added to both VALID_MODES and RUNTIME_MODES
- If a new mode is independent (like 'review'), add it to INDEPENDENT_MODES in ponytail-instructions.js

---

## If I Need to Change Configuration Resolution or Storage

**Change**: Add new config file location, change env var name, modify mode persistence logic

**Likely files to look at first**:
- `hooks/ponytail-config.js` — Controls config dir, file path, and resolution hierarchy
- `hooks/ponytail-runtime.js` — Manages flag file paths for each agent
- Platform-specific hooks if agent-specific (Claude Code vs Codex vs OpenCode)

**After editing**:
1. Update config path logic in `getConfigDir()` or `getConfigPath()`
2. Update flag file path in `ponytail-runtime.js` if agent-specific storage changed
3. Update `ponytail-activate.js` to read new config location
4. Update `tests/hooks.test.js` to test new path resolution
5. Update `05-config-and-env.md` with new defaults or paths

**To test**:
```bash
node --test tests/hooks.test.js
```

**Caveats**:
- Config resolution order is: env var → config file → default. Changing this breaks backwards compatibility
- Flag file paths are agent-specific (Claude Code vs Codex vs OpenCode); don't mix them up
- Windows paths are handled in `ponytail-config.js`; test on Windows if making path changes

---

## If I Need to Add Support for a New Agent Platform

**Change**: Integrate with a new AI agent harness (IDE, CLI tool, etc.)

**Likely files to look at first**:
- **If hook-based**: Copy `hooks/ponytail-activate.js` and `ponytail-mode-tracker.js` as templates
- **If plugin/extension-based**: Use `pi-extension/index.js` or `.opencode/plugins/ponytail.mjs` as template
- **If static rules**: Create `.new-agent/rules/ponytail.*` (format TBD based on agent's rule file format)

**New adapter checklist**:
- [ ] Implement mode resolution (reuse `ponytail-config.js`)
- [ ] Implement instruction generation (reuse `ponytail-instructions.js`)
- [ ] Implement mode persistence (reuse `ponytail-runtime.js` or adapt it)
- [ ] Implement command/mode-switch detection (e.g., parse `/ponytail lite` from user input)
- [ ] Test: create `tests/your-agent.test.js` validating mode switching and instruction injection
- [ ] Document: update README install section, `docs/agent-portability.md`
- [ ] Sync: add new platform files to `scripts/check-rule-copies.js` validator
- [ ] Update `package.json` or manifest files if needed for the agent's installer

**Related files**:
- `docs/agent-portability.md` — Maps adapters to agent platforms
- `README.md` — Install section lists supported platforms
- `scripts/check-rule-copies.js` — Validates rule file sync across platforms

---

## If I Need to Fix a Bug in Hook Execution or Mode Tracking

**Change**: Fix incorrect mode being applied, flag file not written, or instructions not injected

**Likely files to look at first**:
- `hooks/ponytail-activate.js` — SessionStart logic; mode resolution, flag writing, instruction output
- `hooks/ponytail-mode-tracker.js` — UserPromptSubmit logic; command parsing, mode switching
- `hooks/ponytail-runtime.js` — Flag file path resolution; platform-specific paths may be wrong
- `tests/hooks.test.js` — Existing tests that reproduce the bug

**Debug approach**:
1. Write a minimal test reproducing the bug
2. Run with explicit env vars and temporary directories
3. Check flag file contents: `fs.readFileSync(flagPath, 'utf8')`
4. Check hook output: `result.stdout`, `result.stderr`
5. Check exit code: `result.status` (0 = success)

**After fixing**:
1. Verify test passes: `node --test tests/hooks.test.js`
2. Check that no other tests break: `node --test tests/*.test.js`
3. Test manually in your agent (Claude Code, Codex, etc.) to confirm real-world behavior

**Caveats**:
- Hooks run as child processes; environment setup is critical
- Windows and Linux/macOS have different path handling
- Tests may need to be split into `hooks.test.js` (POSIX) and `hooks-windows.test.js` (Windows)

---

## If I Need to Update Benchmarking or Performance Results

**Change**: Add new benchmark task, update test prompt, or reproduce results

**Likely files to look at first**:
- `benchmarks/promptfooconfig.yaml` — Test cases, providers (models), result metrics
- `benchmarks/prompts.json` — Prompt text, expected outputs
- `benchmarks/arms/` — Arm definitions (no-skill, caveman, ponytail)
- `benchmarks/results/` — Previous result reports

**To run benchmarks**:
```bash
cd benchmarks
npx promptfoo eval -c promptfooconfig.yaml
```

**Before running**:
1. Copy `.env.example` to `.env` and fill in `ANTHROPIC_API_KEY`
2. Ensure `skills/ponytail/SKILL.md` reflects latest rules (benchmarks use this file)

**After running**:
1. Results are generated in `benchmarks/` (view with `promptfoo view`)
2. Export to markdown: `promptfoo export -o results/YYYY-MM-DD-description.md`
3. Commit result report to document performance claims

**Caveats**:
- Benchmarking requires API calls to Anthropic; costs real money
- Results are stochastic (vary by model, prompts); averages are usually reported
- New benchmark arms or prompts need corresponding additions to config

---

## If I Need to Ensure Rule Files Stay in Sync Across Platforms

**Change**: Validate that `.cursor/`, `.windsurf/`, `.clinerules/`, etc. match `AGENTS.md`

**Likely files to look at first**:
- `scripts/check-rule-copies.js` — The validator
- `AGENTS.md` — Primary source
- All platform-specific rule files (listed in the script)

**To run validator**:
```bash
node scripts/check-rule-copies.js
```

**Exit codes**:
- 0: All files match primary
- Non-zero: Mismatch found; script prints which files are out of sync

**After making changes to AGENTS.md**:
1. Manually update platform files or use validator to find discrepancies
2. Run: `node scripts/check-rule-copies.js`
3. If still failing, check file formats (`.mdc` vs `.md` vs `.txt`; may have platform-specific markup)

**Caveats**:
- Validator compares normalized content; whitespace/comment changes may not be caught
- Some platform files may have platform-specific annotations; check manually if validator says "synced" but content looks different

---

## If I Need to Make Changes Related to Specific Agent Platforms

### Claude Code

**Files**: `hooks/ponytail-activate.js`, `hooks/ponytail-mode-tracker.js`, `hooks/hooks.json`, `~/.claude/settings.json` (user config)

**Specific issues**:
- Statusline badge: requires user to add to `~/.claude/settings.json`; nudge happens on SessionStart
- Hook output: plain text (not JSON)
- Flag file: `~/.claude/.ponytail-active`

### Codex

**Files**: Same hooks as Claude Code, but output format is JSON

**Specific issues**:
- Plugin data location: `${PLUGIN_DATA}` env var (set by Codex, not user)
- Hook output: JSON with `systemMessage` and `hookSpecificOutput` fields
- Flag file: `${PLUGIN_DATA}/.ponytail-active`

### OpenCode

**Files**: `.opencode/plugins/ponytail.mjs`, `opencode.json`

**Specific issues**:
- Plugin API: `experimental.chat.system.transform` handler (every turn)
- Command handling: `/ponytail <mode>` parsed by command handler
- Flag file: `~/.config/opencode/.ponytail-active`

### Pi

**Files**: `pi-extension/index.js`, `package.json` (entry point)

**Specific issues**:
- ES module (not CommonJS)
- Reuses CommonJS core modules via `createRequire`
- Exports functions, not a default handler

### Static rule files

**Files**: `.cursor/rules/`, `.windsurf/rules/`, `.clinerules/`, `.kiro/steering/`, etc.

**Specific issues**:
- No hooks; rules are read on agent startup
- No mode switching; content is static
- Format varies by agent (Markdown, MDC, etc.)
