---
name: hugging-face-toad
description: Use when users want to create new agents, configure agent behavior, build sub-agents as tools, or set up few-shot prompting patterns. Combine with hugging-face-tool-builder and huggingface-cli for powerful Hugging Face integrations.
---

# Agent Builder

Create AgentCards—markdown files with YAML frontmatter—that define agents for this environment.

**Good agents are:**
- **Goal-focused** — Direct themselves toward outcomes, not just execute instructions
- **Simple** — Minimal complexity; leverage tools over elaborate prompting
- **Flexible** — Adapt to unexpected circumstances while remaining efficient
- **Context-efficient** — Return concise, actionable responses; don't bloat parent context with intermediate work

AgentCards support preloaded User/Assistant conversations for few-shot prompting or as starting points for future interactions.

**Use cases include:**
- Creating "experts" with access to URLs or specific filesystem areas
- Automating tasks and workflows
- Using few-shot prompting to transform content (documents, prose, code)

## Installation & Loading

| Location | Behavior |
|----------|----------|
| `.fast-agent/agent-cards/` | Loaded on startup, accessible to users |
| `.fast-agent/tool-cards/` | Loaded as tools for the default agent |

**⚠️ CRITICAL:** In Toad, restart the application after adding/modifying agents. Use `ctrl+o` to view available agents.

**⚠️ IMPORTANT:** Paths are relative to the cards folder. Don't place other `.md` files in these directories—use subdirectories for companion files.

Hot-load agents as tools: `/card <filename.md> --tool`

## Choosing the Right Approach

| Approach | Consumer | Best For |
|----------|----------|----------|
| **Agent** | User | Direct interaction, exploration, multi-turn conversation |
| **Agent-as-Tool** | Agent | Encapsulated multi-step reasoning where only results matter |
| **Shell** | Agent | Leveraging existing CLI tools, piping, simple commands |
| **Python Function** | Agent | Data transformation, structured I/O, complex processing, API calls |

**Decision flow:**
1. Does the user need to interact directly? → **Agent**
2. Does it require LLM reasoning across steps? → **Agent-as-Tool**
3. Does an existing CLI tool do this? → **Shell**
4. Otherwise → **Python Function**

## AgentCard Format

### Frontmatter

```yaml
---
name: agent_name        # defaults to filename (preferred)
description: string     # used when exposed as a tool
agents: [list]          # child agents available as tools
---
```

#### Configuration Reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | filename | AgentCard identifier |
| `description` | string | — | Tool description when used as sub-agent |
| `default` | boolean | false | Mark as default when multiple agents loaded |
| `model` | string | config | Model selector—use default unless specified |
| `use_history` | boolean | true | Retain conversation history; `false` for stateless agents |
| `shell` | boolean | false | Enable `execute` tool for shell commands |
| `cwd` | string | — | Working directory for shell (requires `shell: true`) |
| `skills` | list/string | auto | Skills directory or `[]` to disable |
| `agents` | list | [] | Child agents as tools |
| `messages` | string/list | — | Path(s) to history files (relative to agent-cards) |
| `function_tools` | list | [] | Python functions: `["module.py:function_name"]` |
| `request_params` | dict | {} | Model params (e.g., `{temperature: 0.7}`)—use defaults unless requested |
| `tool_only` | boolean | false | Hide from agent list; available only as a tool for other agents |

### Body (System Prompt)

The markdown body after frontmatter defines the system prompt. If omitted, the default is used.

### Placeholders

| Placeholder | Description | On Error |
|-------------|-------------|----------|
| `{{currentDate}}` | Current date | — |
| `{{hostPlatform}}` | Platform info | — |
| `{{pythonVer}}` | Python version | — |
| `{{workspaceRoot}}` | Working directory | Empty |
| `{{env}}` | Environment description | Empty |
| `{{serverInstructions}}` | MCP server instructions (XML) | Empty |
| `{{agentSkills}}` | Available skills (XML) | Empty |
| `{{file:path}}` | File content (relative path) | Error |
| `{{file_silent:path}}` | File content (relative path) | Empty |
| `{{url:https://...}}` | URL content | Empty |

**Notes:**
- File paths must be relative
- `{{file_silent:AGENTS.md}}` is commonly used for project-specific docs
- Unresolved placeholders become empty strings

### Default System Prompt

```markdown
You are a helpful AI Agent.

{{serverInstructions}}
{{agentSkills}}
{{file_silent:AGENTS.md}}
{{env}}

The current date is {{currentDate}}.
```

### Loading External Prompts

Use `instruction: <path|url>` to load from a file or URL.

**TIP:** Use raw file URLs:
- **GitHub:** `https://raw.githubusercontent.com/.../SKILL.md`
- **Gist:** `https://gist.githubusercontent.com/.../raw/.../file.py`
- **HuggingFace:** `https://huggingface.co/.../raw/main/README.md`
- **Docs:** `https://modelcontextprotocol.io/docs/learn/architecture.md`

Centralizing prompts in URLs simplifies managing behavior across multiple agents.

## Preloading Message History

History files use this format:

- **Simple mode:** If no delimiter on first non-empty line, entire file = one user message
- **Delimited mode:** Use exact markers on their own line:
  - `---USER`
  - `---ASSISTANT`
  - `---RESOURCE` (next line = relative path to embed)

**Example:**
```
---USER
Summarize the attached config.

---RESOURCE
configs/app.yaml

---ASSISTANT
Here's a brief summary...
```

**Usage:**
```yaml
---
name: my_agent
messages: ./messages/history.md
---
```

## Agents as Tools

Child agents handle multi-turn tasks (e.g., search and summarization) within their own context, returning only results to the parent.

```yaml
agents:
  - search_agent
  - tool_agent
```

Child agents can use faster/cheaper models and have their own tools or shell access.

### Tool-only Agents

For agents intended exclusively as tools, use `tool_only: true`:

```yaml
---
name: code_formatter
tool_only: true
description: Formats code according to project style guidelines.
---

## Design Considerations

### Model Selection

Agents use the harness default model unless `model: <model_id>` is specified. For speed-critical tasks (search, data processing), `gpt-oss` is a cost-effective choice.

### Shell Access

Enable with `shell: true`. Adds an `execute` tool for bash commands. In Toad, shell access is present by default.

### Python Functions

Reference functions as `module.py:function_name`:

```yaml
---
name: my_agent
function_tools:
  - tools.py:add
  - hf_api_tool.py:hf_api_request
---
```

Paths resolve relative to the AgentCard directory.

**How it works:**
1. Loader parses the `module.py:function_name` spec
2. Dynamically imports the module via `importlib.util`
3. Wraps the function as a FastMCP tool
4. Auto-generates JSON schema from type hints and docstrings

| Requirement | Detail |
|-------------|--------|
| Type hints | Required for schema generation |
| Return type | Must be JSON-serializable |
| Execution | Same Python process (no sandbox) |
| Imports | Standard Python—any installed package works |

**Example:**
```python
import os
from urllib.request import Request, urlopen

def hf_api_request(endpoint: str, method: str = "GET") -> dict:
    token = os.getenv("HF_TOKEN")
    # ... implementation
```

Python functions are often faster and more robust for data processing.

### History Management

| Agent Type | History Behavior |
|------------|------------------|
| Standard agents | Retained (conversational) |
| Agents as tools | Not retained between invocations |

Disable with `use_history: false` for:
- Unbiased question-answering (no conversation skew)
- Content transformation via few-shot prompting (consistent pattern application)

### History Management

| Agent Type | History Behavior |
|------------|------------------|
| Standard agents | Retained (conversational) |
| Agents as tools | Not retained between invocations |

Disable with `use_history: false` for:
- Unbiased question-answering (no conversation skew)
- Content transformation via few-shot prompting (consistent pattern application)### Skills

Skills are SKILL.md files that extend agent capabilities. Descriptions auto-load into the system prompt via `{{agentSkills}}`.

| Configuration | Effect |
|---------------|--------|
| Default | Auto-discover from `.fast-agent/skills/` |
| `skills: ./custom-skills/` | Use specific directory |
| `skills: []` | Disable skills |

Disable skills for narrowly-focused agents. Excluding `{{agentSkills}}` from the prompt also works but generates a warning.
```

