# Concepts

## The Decision Ladder

**Meaning**: A 6-step evaluation process that agents use before writing any code. Stop at the first rung that holds; if multiple rungs apply, take the highest.

**Where it appears**: `AGENTS.md` (steps 1–6 under "Before writing any code"), `skills/ponytail/SKILL.md` (expanded with examples for each intensity level)

**The rungs**:
1. Does this need to exist at all? (YAGNI — skip speculative work)
2. Does the standard library do it? (use built-in APIs)
3. Does a native platform feature cover it? (`<input type="date">` over a picker library)
4. Does an already-installed dependency solve it? (no new dependencies)
5. Can this be one line? (implement it as a one-liner)
6. Only then: write the minimum code that works

**Related files**: `hooks/ponytail-instructions.js` (injects filtered ladder), `tests/hooks.test.js` (validates ladder is present in injected context)

---

## Intensity Level

**Meaning**: A mode that controls how strictly the ladder is applied.

- **lite**: Build what's asked, but name the lazier alternative
- **full** (default): Ladder enforced; stdlib and native first; shortest diff
- **ultra**: YAGNI extremist; deletion before addition; challenge requirements in parallel with shipping one-liners
- **off**: Ponytail inactive; normal agent behavior

**Where it appears**: `ponytail-config.js` (VALID_MODES = ['off', 'lite', 'full', 'ultra', 'review']), `SKILL.md` (table rows and examples keyed by intensity), `ponytail-instructions.js` (filterSkillBodyForMode)

**Related files**: `hooks/ponytail-activate.js` (applies default level on session start), `hooks/ponytail-mode-tracker.js` (detects `/ponytail <level>` commands and persists)

---

## Mode Persistence

**Meaning**: The storage and retrieval of a user's chosen intensity level, so it remains active across multiple agent turns.

**How it works**:
1. User invokes `/ponytail ultra` (or uses agent-specific syntax)
2. Hook/plugin detects the command and calls `writeMode(mode)`
3. Mode is written to a flag file specific to the agent harness
4. On next turn, hook/plugin reads the flag and applies that level

**Where it appears**: `ponytail-config.js` (resolves effective mode), `ponytail-runtime.js` (manages flag file paths), `ponytail-activate.js` (writes flag on session start)

**Flag file locations**:
- Claude Code: `~/.claude/.ponytail-active`
- Codex: `${PLUGIN_DATA}/.ponytail-active`
- OpenCode: `${XDG_CONFIG_HOME:-~/.config}/opencode/.ponytail-active`

**Related files**: `tests/hooks.test.js` (validates persistence across turns)

---

## Instruction Injection

**Meaning**: The process of adding ponytail rules to an agent's system prompt (or equivalent) so the agent applies them.

**Mechanisms by platform**:
- **Claude Code**: Hook fires on SessionStart; writes JSON object with `systemMessage` field
- **Codex**: Hook writes JSON with `systemMessage` and `hookSpecificOutput` fields
- **OpenCode**: Plugin function `experimental.chat.system.transform` appends ponytail instructions to `output.system` every turn
- **Pi**: Extension calls `getPonytailInstructions()` and embeds result in context
- **Rule files**: Platform configs (Cursor, Windsurf, Kiro) include AGENTS.md verbatim; loaded directly by editor

**Where it appears**: `ponytail-instructions.js` (builds instructions from SKILL.md), `ponytail-activate.js` (Claude Code injection), `.opencode/plugins/ponytail.mjs` (OpenCode injection), `pi-extension/index.js` (Pi integration)

**Related files**: `SKILL.md` (source of truth for injected content)

---

## Platform Adapter

**Meaning**: A platform-specific bridge that connects ponytail's core logic to an agent harness.

**Examples**:
- Claude Code: hook-based (`ponytail-activate.js`, `ponytail-mode-tracker.js`)
- Codex: hook-based with JSON output formatting
- Pi: ES module extension exporting resolver functions
- OpenCode: ES module plugin with command and transform handlers
- Cursor/Windsurf/Kiro: static rule files copied from AGENTS.md

**Why it matters**: Each agent harness has a different API. Adapters normalize the ponytail logic (same decision ladder, same mode resolution) to work everywhere.

**Where it appears**: Platform-specific directories (`.cursor/`, `.windsurf/`, `.clinerules/`, `pi-extension/`, `.opencode/plugins/`, `gemini-extension.json`)

**Related files**: `scripts/check-rule-copies.js` (validates adapters stay in sync), `docs/agent-portability.md` (maps files to agent platforms)

---

## SKILL.md

**Meaning**: The canonical source file for ponytail rules, structured with Markdown frontmatter and mode-filtered content.

**Structure**:
- Frontmatter: name, description, license
- Mode-specific content: table rows and example sections keyed by intensity level
- Non-mode-specific content: general rules that apply to all levels

**Filtering**: `ponytail-instructions.js` reads SKILL.md and removes sections tagged with intensity levels other than the active one.

**Where it appears**: `skills/ponytail/SKILL.md` (primary), `skills/ponytail-review/SKILL.md`, `skills/ponytail-audit/SKILL.md`, `skills/ponytail-help/SKILL.md`

**Related files**: `hooks/ponytail-instructions.js` (filterSkillBodyForMode), `AGENTS.md` (same content, different format for agents that read static rules)

---

## ponytail: Comment

**Meaning**: A marker comment placed in generated code to indicate an intentional simplification with a known ceiling or upgrade path.

**Format**: `ponytail: <description>. <ceiling/limitation>. <upgrade path>.`

**Example**: `// ponytail: global lock. Per-account locks if throughput matters.`

**Why it matters**: Signals that a shortcut was deliberate and documented, so maintainers know what was skipped and when to fix it.

**Where it appears**: Generated code examples in SKILL.md, benchmarks/results/ (production task walkthroughs)

**Related files**: `AGENTS.md` (rule about marking simplifications)

---

## Test

**Meaning**: Validation scripts that verify hooks, mode resolution, and instruction generation work as expected.

**Framework**: Node.js native test runner (no external test framework)

**Where it appears**: `tests/hooks.test.js`, `tests/hooks-windows.test.js`, `tests/gemini-extension.test.js`, `tests/opencode-plugin.test.js`

**Coverage**: Config file loading, mode switching, state persistence, instruction generation, platform-specific output formats

**Related files**: `hooks/ponytail-*.js` (code under test)
