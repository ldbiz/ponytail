# Diagrams

## 1. Runtime Activation Flow

```mermaid
graph TD
    A["Agent starts session"]
    B["SessionStart event fires"]
    C["Hook: ponytail-activate.js"]
    D["Resolve mode:<br/>env var → config file → full"]
    E["Write mode flag:<br/>~/.claude/.ponytail-active"]
    F["Read SKILL.md"]
    G["Filter by intensity"]
    H["Generate instructions"]
    I["Output to agent:<br/>Claude: text<br/>Codex: JSON<br/>OpenCode: append to system"]
    J["Agent LLM receives rules<br/>Active for session"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    D --> F
    F --> G
    G --> H
    H --> I
    I --> J
    
    K["User input: /ponytail ultra"]
    L["UserPromptSubmit hook"]
    M["Parse command<br/>Validate mode"]
    N["Write new mode to flag"]
    O["Next turn: read new flag<br/>re-inject with new intensity"]
    
    K --> L
    L --> M
    M --> N
    N --> O
```

## 2. Module Interaction Diagram

```mermaid
graph TD
    A["AGENTS.md<br/>(primary ruleset)"]
    B["skills/ponytail/SKILL.md<br/>(mode-filtered version)"]
    
    C["ponytail-config.js<br/>(mode resolution)"]
    D["ponytail-instructions.js<br/>(filter & generate)"]
    E["ponytail-runtime.js<br/>(state & output)"]
    
    F["ponytail-activate.js<br/>(SessionStart)"]
    G["ponytail-mode-tracker.js<br/>(UserPromptSubmit)"]
    
    H["pi-extension/index.js<br/>(Pi integration)"]
    I[".opencode/plugins/ponytail.mjs<br/>(OpenCode integration)"]
    
    J["Agent harness<br/>(Claude Code, Codex, OpenCode, etc.)"]
    
    A -->|copied to| B
    B -->|read by| D
    A -->|source for static| F
    
    C -->|resolves default| F
    C -->|resolves default| H
    C -->|resolves default| I
    
    D -->|generates filtered| F
    D -->|generates filtered| H
    D -->|generates filtered| I
    
    E -->|manages state| F
    E -->|manages state| G
    
    F -->|injects context| J
    G -->|updates mode| J
    H -->|provides functions| J
    I -->|injects every turn| J
```

## 3. Configuration Resolution Hierarchy

```mermaid
graph TD
    A["Determine effective mode for session"]
    B["PONYTAIL_DEFAULT_MODE<br/>env variable set?"]
    C{Yes}
    D{No}
    E["Config file exists?<br/>~/.config/ponytail/config.json"]
    F{Yes}
    G{No}
    H["Use env var<br/>mode"]
    I["Read config file<br/>defaultMode field"]
    J["defaultMode<br/>present & valid?"]
    K{Yes}
    L{No}
    M["Use config<br/>mode"]
    N["Use hardcoded<br/>default: full"]
    O["Use resolved<br/>mode"]
    
    A --> B
    B --> C
    B --> D
    C --> H
    D --> E
    E --> F
    E --> G
    F --> I
    G --> N
    I --> J
    J --> K
    J --> L
    K --> M
    L --> N
    
    H --> O
    M --> O
    N --> O
```

## 4. Mode Persistence Across Turns

```mermaid
sequenceDiagram
    actor User
    participant Agent
    participant Hook
    participant FS as Flag File
    
    User->>Agent: Start session
    Agent->>Hook: SessionStart event
    Hook->>FS: Read flag (or use default)
    FS-->>Hook: Return mode (or default)
    Hook->>Agent: Inject instructions for mode
    Agent-->>User: Show status: PONYTAIL:FULL
    
    User->>Agent: /ponytail ultra
    Agent->>Hook: UserPromptSubmit event
    Hook->>Hook: Parse: ultra
    Hook->>FS: Write "ultra" to flag
    FS-->>Hook: OK
    
    User->>Agent: Next message
    Agent->>Hook: SessionStart (new turn)
    Hook->>FS: Read flag
    FS-->>Hook: "ultra"
    Hook->>Agent: Inject instructions for ultra
    Agent-->>User: Show status: PONYTAIL:ULTRA
```

## 5. Mode Filtering in SKILL.md

```mermaid
graph LR
    A["SKILL.md with all modes<br/>table rows & examples<br/>for lite/full/ultra"]
    B["filterSkillBodyForMode<br/>input: mode=full"]
    C["Split by lines<br/>identify mode-keyed items"]
    D["Table rows:<br/>filter by **mode** label"]
    E["Example blocks:<br/>filter by mode: label"]
    F["Other lines:<br/>keep verbatim"]
    G["Filtered output:<br/>only full intensity content"]
    
    A --> B
    B --> C
    C --> D
    C --> E
    C --> F
    D --> G
    E --> G
    F --> G
```

## 6. Installation to Active Flow (User Perspective)

```mermaid
graph TD
    A["User: Install ponytail<br/>/plugin install<br/>or equivalent"]
    B["Agent downloads plugin<br/>copies to config dir"]
    C["User: Start agent session"]
    D["Agent auto-loads<br/>ponytail hooks/plugins"]
    E["Ponytail SessionStart fires"]
    F["Ponytail injected<br/>Agent shows active status"]
    G["User: Type code request<br/>with ponytail rules active"]
    H["Agent responses<br/>follow lazy ladder"]
    I["User: /ponytail ultra<br/>switch mode mid-session"]
    J["Next turn:<br/>ultra intensity active"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I --> J
```

## 7. Platform Adapter Architecture

```mermaid
graph TD
    A["Shared Core<br/>ponytail-config.js<br/>ponytail-instructions.js<br/>ponytail-runtime.js<br/>SKILL.md"]
    
    B["Claude Code Adapter<br/>hooks/ponytail-activate.js<br/>hooks/ponytail-mode-tracker.js<br/>hooks.json"]
    C["Codex Adapter<br/>same hooks,<br/>JSON output format"]
    D["Pi Adapter<br/>pi-extension/index.js<br/>ES module"]
    E["OpenCode Adapter<br/>.opencode/plugins/ponytail.mjs<br/>plugin system"]
    F["Static Rules<br/>.cursor/rules/*.mdc<br/>.windsurf/rules/*<br/>.clinerules/*"]
    
    G["Claude Code<br/>Agent"]
    H["Codex<br/>Agent"]
    I["Pi<br/>Agent"]
    J["OpenCode<br/>Agent"]
    K["Cursor/Windsurf<br/>etc."]
    
    A --> B
    A --> C
    A --> D
    A --> E
    A --> F
    
    B --> G
    C --> H
    D --> I
    E --> J
    F --> K
```

## Notes on Diagram Accuracy

- All diagrams represent real code flow and module relationships found in the repository
- Sequence diagram (4) shows typical multi-turn session; specific timings depend on agent harness
- Module interaction (2) shows data flow; not all dependencies are shown for clarity
- Configuration hierarchy (3) is implemented in `ponytail-config.js` `getDefaultMode()`
- Mode filtering (5) is implemented in `ponytail-instructions.js` `filterSkillBodyForMode()`
