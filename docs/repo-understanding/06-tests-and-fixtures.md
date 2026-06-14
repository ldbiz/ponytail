# Tests and Fixtures

## Test Framework

**Framework**: Node.js native test runner (`node --test`)

**Why**: No external test dependencies; lightweight, built-in, no setup overhead

**Run all tests**:
```bash
node --test tests/*.test.js
```

**Run specific test file**:
```bash
node tests/hooks.test.js
```

## Test Files

| File | Purpose | Coverage |
|---|---|---|
| `tests/hooks.test.js` | Hook activation and mode persistence (Linux/macOS) | SessionStart/UserPromptSubmit lifecycle, config resolution, mode flags, instruction injection |
| `tests/hooks-windows.test.js` | Windows-specific hook behavior | Path handling, USERPROFILE vs HOME, Windows config directories |
| `tests/opencode-plugin.test.js` | OpenCode plugin integration | Plugin load, system.transform handler, instruction appending |
| `tests/gemini-extension.test.js` | Gemini CLI extension | Extension loading, instruction generation |

## Key Test Patterns

### 1. Temporary Environment Setup

Tests create isolated temporary directories to avoid polluting user config:

```javascript
const temp = fs.mkdtempSync(path.join(os.tmpdir(), 'ponytail-hooks-'));
const home = path.join(temp, 'home');
const pluginData = path.join(temp, 'plugin-data');
fs.mkdirSync(home, { recursive: true });
```

**Why**: Each test run is independent; no shared state; clean environment for assertions

### 2. Hook Execution via Process Spawn

Hooks are run as separate Node.js processes to simulate agent behavior:

```javascript
function run(script, env, input = '') {
  return spawnSync(process.execPath, [path.join(root, 'hooks', script)], {
    env: { ...process.env, ...env },
    input,
    encoding: 'utf8',
  });
}
```

**Why**: Validates real-world hook execution; catches environment/path issues

### 3. Fixture Setup: Environment Variables

Tests set environment variables to simulate different scenarios:

```javascript
const codexEnv = {
  HOME: home,
  USERPROFILE: home,
  PLUGIN_DATA: pluginData,
  PONYTAIL_DEFAULT_MODE: 'ultra',
};
```

**Scenarios**:
- Codex mode: `PLUGIN_DATA` set
- Claude Code mode: `PLUGIN_DATA` unset
- Mode override: `PONYTAIL_DEFAULT_MODE` set to specific level

### 4. State Assertions

Tests verify expected state files are written and readable:

```javascript
result = run('ponytail-activate.js', codexEnv);
assert.equal(result.status, 0, result.stderr);
assert.equal(fs.readFileSync(codexState, 'utf8'), 'ultra');
```

**Assertions**:
- Exit code is 0
- Flag file exists and contains expected mode
- Output is valid JSON (Codex) or plain text (Claude Code)

### 5. Mode Switching Across Turns

Tests verify persistence across multiple hook invocations:

```javascript
// Turn 1: activate with default mode
result = run('ponytail-activate.js', codexEnv);
assert.equal(fs.readFileSync(codexState, 'utf8'), 'ultra');

// Turn 2: user changes mode
result = run('ponytail-mode-tracker.js', codexEnv, JSON.stringify({ prompt: '@ponytail lite' }));
assert.equal(fs.readFileSync(codexState, 'utf8'), 'lite');

// Turn 3: verify mode persisted
result = run('ponytail-activate.js', codexEnv);
assert.equal(fs.readFileSync(codexState, 'utf8'), 'lite');
```

**Why**: Simulates multi-turn user sessions; validates mode is sticky

## Fixture Data

### Standard Test Config Files

Tests do NOT use fixtures on disk; they use environment variables and temporary directories. This keeps tests isolated and reproducible.

However, tests verify that when config files are present, they are read correctly:

**Test**: Creating a config file and verifying it's loaded
```javascript
const configPath = path.join(home, '.config', 'ponytail', 'config.json');
fs.mkdirSync(path.dirname(configPath), { recursive: true });
fs.writeFileSync(configPath, JSON.stringify({ defaultMode: 'ultra' }), 'utf8');

const mode = getDefaultMode(); // Should return 'ultra'
```

## Coverage

### Behaviours Tested

| Behaviour | Test File | Status |
|---|---|---|
| SessionStart hook runs without error | `hooks.test.js` | ✅ |
| Mode resolution from PONYTAIL_DEFAULT_MODE env var | `hooks.test.js` | ✅ |
| Mode resolution from config file | `hooks.test.js` | ✅ |
| Mode flag file written on activation | `hooks.test.js` | ✅ |
| Instructions filtered by intensity level | `hooks.test.js` | ✅ |
| Codex-specific JSON output format | `hooks.test.js` | ✅ |
| Claude Code plain text output format | `hooks.test.js` | ✅ |
| Mode switching via UserPromptSubmit hook | `hooks.test.js` | ✅ |
| Mode change persists across turns | `hooks.test.js` | ✅ |
| Windows path handling (USERPROFILE) | `hooks-windows.test.js` | ✅ |
| OpenCode plugin system integration | `opencode-plugin.test.js` | ✅ |
| Gemini extension integration | `gemini-extension.test.js` | ✅ |

### Gaps

| Gap | Why It Matters | Workaround |
|---|---|---|
| No integration test across all agent platforms | Manual testing before each release | README install instructions guide users; community reports issues |
| No test for rule file synchronization | Catches out-of-sync `.cursor/`, `.windsurf/`, etc. | Manual review + `check-rule-copies.js` pre-commit hook |
| No test for Gemini CLI statusline generation | Statusline badge is pure shell script | Manual testing on macOS/Linux |
| No test for Pi extension in Pi harness | Requires Pi runtime environment | Manual testing in Pi agent |
| No benchmark test in CI | Benchmarking requires LLM API calls | Manual run before each release; results stored in `benchmarks/results/` |

## Running Tests

### Quick Test

```bash
cd /path/to/ponytail
node --test tests/*.test.js
```

### Watch Mode

Not built-in to Node.js test runner; use external tool or re-run manually:

```bash
nodemon --exec "node --test tests/*.test.js"
```

### Specific Test

```bash
node tests/hooks.test.js
```

### Debug

Print intermediate output:

```bash
node --test tests/hooks.test.js --verbose
```

## Test Code Quality

**No external dependencies**: Tests use only Node.js built-ins (assert, fs, path, os, child_process)

**Minimal setup**: Each test runs in isolation; no shared state between test files

**Self-contained**: Temporary directories and environment variables are created and cleaned up within each test

**Real-world validation**: Hooks are spawned as actual processes, not mocked; catches real path/environment issues

**Fast**: All tests complete in < 1 second (no I/O wait, no network calls)
