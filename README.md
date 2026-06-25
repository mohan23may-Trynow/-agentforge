# AgentForge

A starter framework for **local-AI agents**. Drop files into folders and the
framework auto-registers them — agents, MCP servers, and custom tools — then
runs any agent on **any local AI tool**, exports it to **Claude Code**, or lists
everything in a browsable UI. Build agents by editing small files instead of
wiring everything from scratch.

```
agents/*.agent.yaml   ->  agents (provider, model, prompt, tool + MCP scopes)
mcp/*.json            ->  MCP servers (same config shape as Claude Desktop)
tools/*.py            ->  custom @tool functions (auto-registered)
```

## Works with any local AI tool

The framework isn't tied to Ollama. Almost every local runner exposes an
OpenAI-compatible `/v1/chat/completions` endpoint, so one backend covers them
all — you just name a provider:

| Provider    | Default endpoint              | Tool calling |
|-------------|-------------------------------|:------------:|
| `ollama`    | `http://localhost:11434/v1`   | ✓ |
| `lmstudio`  | `http://localhost:1234/v1`    | ✓ |
| `llamacpp`  | `http://localhost:8080/v1`    | ✓ |
| `jan`       | `http://localhost:1337/v1`    | ✓ |
| `localai`   | `http://localhost:8080/v1`    | ✓ |
| `vllm`      | `http://localhost:8000/v1`    | ✓ |
| `gpt4all`   | `http://localhost:4891/v1`    | prompt-only |
| `openclaw`  | your runner's `/v1` endpoint  | ✓ |
| `custom`    | set `base_url` yourself       | ✓ |

Run `forge providers` to see how to start each one. Ports are documented
defaults — override `base_url` in an agent file if yours differs.

## Quickstart

```bash
pip install -e .            # installs the `forge` command (only dep: pyyaml)

forge detect               # scan your machine for running local AI tools
forge init                 # auto-detect one, pick its model, scaffold a wired agent
                           #   + write SETUP-<provider>.md telling you how to finish setup

forge list                  # show discovered agents, tools, MCP servers
forge providers             # local-AI tools and how to start each
forge doctor                # validate (catch missing tool/MCP references)

forge run bug-triage "login button returns 500 on submit"
forge new my-agent          # scaffold a blank agent file
forge export                # write a Claude Code project (.claude/agents, CLAUDE.md, .mcp.json)
forge serve                 # browse + run agents at http://127.0.0.1:8765
```

`forge init` is the "complete my local AI" path: it probes localhost for any
supported runner, asks the one it finds what models it has, fills in the
endpoint + model for you, writes a ready-to-run agent, and generates a
provider-specific `SETUP-*.md` with the exact steps to bring the model fully
online. If nothing's running yet, it still scaffolds everything and tells you
how to start your tool.

`forge run` needs a local model up. Everything else (detect, init, listing,
validating, scaffolding, exporting, the UI) works with no model.

## Project layout

```
your-project/
├─ agents/                 your agents (one YAML each)
│  ├─ inbox-triage.agent.yaml
│  ├─ bug-triage.agent.yaml
│  └─ _template.agent.yaml  (copied by `forge new`)
├─ mcp/                    MCP server configs (*.json)
│  ├─ demo.json
│  ├─ filesystem.json
│  └─ servers/demo_server.py   a tiny stdio MCP server for testing
├─ tools/                  custom tools (*.py with @tool)
│  └─ example_tools.py
├─ ui/index.html           the agent-browser UI
└─ src/agentforge/         the framework
```

## Add an agent

Create `agents/<name>.agent.yaml`:

```yaml
name: inbox-triage
description: Sorts incoming tasks into priority buckets.
provider: ollama          # or lmstudio | llamacpp | jan | vllm | openclaw | custom
model: ""                 # blank uses the provider default
mcp: [demo]               # MCP servers it may use
tools: [read_file, triage_bucket, write_note]
system: |
  You are an inbox-triage agent...
```

## Add a tool

Any `@tool` function in `tools/*.py` is auto-discovered; the JSON schema is
inferred from the signature:

```python
from agentforge import tool

@tool
def read_dir(path: str = ".") -> str:
    """List files at a path."""
    ...
```

## Add an MCP server

Drop a standard config in `mcp/`:

```json
{ "mcpServers": { "filesystem": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"] } } }
```

The framework spawns it over stdio, lists its tools, and exposes them to agents
that list the server under `mcp:`. Tools are namespaced `server__tool`.

## Add a provider

Edit `src/agentforge/providers.py` and add an entry to `PROVIDERS`. If your
runner speaks the OpenAI API, that's all — no backend code needed.

## How it fits together

```
Registry.discover()  ── scans agents/ mcp/ tools/
        │
        ├─ make_backend(agent)         provider -> OpenAI-compat (or Ollama native)
        │     └─ run loop: model <-> tools (custom + MCP) until final answer
        │
        └─ ClaudeCodeBackend.export_all()   same agents -> .claude/agents + CLAUDE.md + .mcp.json
```

## Tests

```bash
python tests/test_smoke.py
```

Covers discovery, tool-schema inference, a live MCP stdio roundtrip against the
bundled demo server, and the Claude Code export.

## License

MIT.
