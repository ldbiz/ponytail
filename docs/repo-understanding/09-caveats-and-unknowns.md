# Caveats and Unknowns

## Uncertainty and Open Questions

### Mode Switching Timing

**Issue**: The problem statement says mode changes take effect "on the next turn," but the exact mechanism is agent-specific.

**What we know**:
- Flag file is written immediately when user types `/ponytail ultra`
- Hook reads flag on next SessionStart or plugin invocation
- Timing depends on agent harness (e.g., OpenCode's `experimental.chat.system.transform` fires every turn; Claude Code's hook fires on SessionStart)

**Unknown**: Precise turn count before new mode takes effect in each agent. For some agents, it may be immediate; for others, delayed until next session.

**Impact**: Users may be confused if they change mode and the next response still uses the old level. Behavior likely varies by agent.

---

### Cross-Platform Path Handling

**Issue**: Config paths differ significantly across Windows, Linux, and macOS.

**What we know**:
- Linux/macOS: `$XDG_CONFIG_HOME/ponytail/config.json` (default `~/.config/ponytail/config.json`)
- Windows: `%APPDATA%\ponytail\config.json`
- Code uses `os.homedir()` and platform checks to resolve paths
- Tests include Windows-specific tests (`hooks-windows.test.js`)

**Unknown**:
- How often Windows users encounter path resolution bugs in real-world usage
- Whether XDG_CONFIG_HOME is respected correctly on all systems
- Corner cases with non-ASCII characters in paths

**Impact**: Cross-platform compatibility could regress if path handling is changed without careful testing on all platforms.

---

### Stateline Badge Configuration (Claude Code Only)

**Issue**: Statusline badge showing active mode (`[PONYTAIL:ULTRA]`) requires manual user setup.

**What we know**:
- `ponytail-activate.js` detects missing statusline config and nudges user
- User must add a snippet to `~/.claude/settings.json`
- Badge shows mode via shell script (`ponytail-statusline.sh` or `.ps1`)

**Unknown**:
- Whether all Claude Code users see the nudge message
- Whether users follow the setup instructions
- Whether the statusline script works on all systems (bash on Linux/macOS, PowerShell on Windows)

**Impact**: Users may not realize ponytail is active because the badge is not configured. Adoption may be affected.

---

### Instruction Injection Completeness

**Issue**: Each agent platform has a different API for injecting context. Does every platform actually execute the rules?

**What we know**:
- Claude Code: plain text to stdout (becomes system message)
- Codex: JSON with systemMessage field
- OpenCode: appended to system prompt every turn
- Pi: functions exported; caller embeds in context
- Static rules: files loaded by editor on startup

**Unknown**:
- Do all agents actually read the injected instructions?
- Are there agents where the injection mechanism is broken or ignored?
- For static rule files, do all agents load them on startup?

**Impact**: A user could install ponytail and not have it apply any rules if injection fails silently.

---

### Rule Filter Accuracy for New Modes

**Issue**: Mode filtering in `filterSkillBodyForMode()` relies on Markdown structure.

**What we know**:
- Filters table rows by looking for `| **<mode>** |` pattern
- Filters example blocks by looking for `- <mode>:` pattern
- Other lines are kept verbatim

**Unknown**:
- Are there edge cases where the regex fails?
- What happens if a table row or example title includes "lite/full/ultra" incidentally?
- If SKILL.md formatting changes, could filtering break silently?

**Impact**: New modes added to the ladder might not filter correctly, resulting in wrong rules being injected. Risk is low because tests validate generated instructions, but edge cases could slip through.

---

### Config File Format Stability

**Issue**: Config file format is JSON with one field `{ "defaultMode": "full" }`. Is this stable or expected to evolve?

**What we know**:
- Current format: JSON with single `defaultMode` field
- No validation or schema enforcement
- Invalid JSON causes silent fallback to 'full'

**Unknown**:
- Will new config options be added in future?
- Should a schema or version field be added for future-proofing?
- How should old/invalid configs be handled?

**Impact**: If config format changes, old user configs may become invalid or ignored silently. Migration path unclear.

---

### Test Coverage Gaps

**Known gaps**:

1. **No integration test for all platforms together** — Tests are per-agent-platform; no end-to-end test across all agents
2. **No test for static rule file generation** — `.cursor/`, `.windsurf/` etc. are assumed to stay in sync; validated by manual `check-rule-copies.js` script
3. **No test for benchmarking workflow** — Benchmarks require LLM API calls; skipped in CI
4. **No test for Gemini CLI statusline** — Statusline generation is a shell script; not tested
5. **No test for Pi extension in actual Pi agent** — Pi tests exist but don't run in actual Pi harness
6. **No load/stress testing** — Unknown how system performs with large ruleset or many users

**Impact**: Bugs may slip through in untested code paths. Release testing is manual.

---

### Platform Coverage Unknown

**Known platforms**:
- Claude Code ✅ (documented, tested)
- Codex ✅ (documented, tested)
- Pi ✅ (documented, tested)
- OpenCode ✅ (documented, tested)
- Gemini CLI ✅ (documented, extension exists)
- Cursor, Windsurf, Cline, Kiro ✅ (rule files exist)
- GitHub Copilot CLI ✅ (reads AGENTS.md)

**Unclear**:
- Do all these platforms actually work? Some are documented but untested.
- Are there other platforms that should be supported?
- Community feedback on platform support is needed.

**Impact**: Users may install ponytail on an unsupported platform and get no indication it's not working.

---

### Example Code Correctness

**Issue**: `examples/` directory shows "before/after" code samples using ponytail style.

**What we know**:
- Examples are documentation only; not tests
- They demonstrate the lazy decision ladder in action
- No validation that examples actually work

**Unknown**:
- Are the "after" examples syntactically correct?
- Do they actually run?
- Are they up-to-date with the latest rules?

**Impact**: Examples may mislead users about what ponytail produces. Visual review only.

---

### Benchmark Methodology

**Issue**: Benchmarks compare ponytail vs no-skill vs caveman skill.

**What we know**:
- Metrics: lines of code, cost ($), latency (sec)
- Models: Haiku, Sonnet, Opus
- Tasks: 5 everyday tasks (email validator, debounce, etc.)
- Methodology is documented in benchmarks/README.md

**Unknown**:
- Are 5 tasks enough to be representative?
- Do results generalize to production workloads?
- Why these specific tasks and not others?
- Are there tasks where ponytail underperforms?

**Impact**: Claims about 80-94% code reduction are based on a small sample. May not hold for all domains.

---

### Dependencies and Security

**What we know**:
- `package.json` has no runtime dependencies (only dev dependencies for benchmarking)
- Only depends on Node.js built-ins and promptfoo for benchmarks

**Unknown**:
- Has anyone performed a security audit?
- Are there any known CVEs in dependencies?
- How is promptfoo (benchmarking only) vetted?

**Impact**: Low risk because no production dependencies, but unknown unknowns remain.

---

### Performance at Scale

**Issue**: Unknown how system performs if mode is switched frequently or if SKILL.md becomes very large.

**What we know**:
- Config resolution is O(1) file I/O
- Instruction filtering is O(n) line-scan through SKILL.md
- Filtering happens on every hook fire

**Unknown**:
- If SKILL.md grows to 10k+ lines, does filtering become slow?
- If user switches mode 100 times per session, does performance degrade?
- Memory usage for large instruction strings?

**Impact**: Probably not a concern for current scale, but scaling is untested.

---

### Documentation Maintenance

**Issue**: This repo understanding pack is generated from code exploration. How will it stay in sync?

**What we know**:
- Documentation is version-controlled in `docs/repo-understanding/`
- No automation to keep it in sync with code

**Unknown**:
- Who is responsible for updating docs when code changes?
- Is there a code review checklist to update docs?
- How often does documentation become stale?

**Impact**: Documentation may diverge from reality over time. Manual discipline required.
