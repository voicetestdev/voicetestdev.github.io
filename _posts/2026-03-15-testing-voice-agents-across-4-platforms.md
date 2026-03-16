---
title: "Testing Voice Agents Across 4 Platforms With a Single Tool"
description: "Import from Retell, VAPI, Bland, or LiveKit. Write one test suite. Run it against any agent format. Voicetest's AgentGraph IR normalizes platform differences."
date: 2026-03-15
---

If you've built voice agents on more than one platform, you know the problem: each one has its own config format, its own mental model, and its own way of defining agent behavior. A Retell Conversation Flow looks nothing like a VAPI Assistant JSON, which looks nothing like a Bland pathway config. Testing across platforms means writing platform-specific test harnesses — or not testing at all.

Voicetest solves this with AgentGraph, an intermediate representation that normalizes voice agent configs from Retell, VAPI, LiveKit, Bland, and Telnyx into a single graph structure. Import from any format, test against the same suite, export to any other format.

This post walks through the IR design, the import/export pipeline, and how to set up a cross-platform test suite.

### The AgentGraph IR

Every voice agent, regardless of platform, is a directed graph: nodes with prompts, edges with transition conditions, and optionally tools the agent can call. Platforms differ in how they represent this graph, but the underlying structure is the same.

AgentGraph is a Pydantic model that captures this structure:

```python
class AgentGraph(BaseModel):
    nodes: dict[str, AgentNode]
    entry_node_id: str
    source_type: str
    source_metadata: dict[str, Any]
    snippets: dict[str, str]
    default_model: str | None
```

Each `AgentNode` contains:

- `state_prompt` — the system instructions active when the agent is in this node
- `transitions` — a list of conditions and target node IDs
- `tools` — function/tool definitions available in this state

### Import pipeline

Importers are format-specific parsers that produce an `AgentGraph` from raw config JSON. Each importer handles the quirks of its platform:

- **Retell CF**: Parses `start_node_id`, `nodes` array, and `edges` with `description` fields as transition conditions. Handles both Conversation Flow and LLM formats (detected by the presence of `general_prompt` vs `start_node_id`).
- **VAPI**: Parses Assistant JSON (single-node agent with tools) and Squad JSON (multi-agent handoffs mapped to graph transitions).
- **Bland**: Parses pathway configs with `nodes` and `edges`, mapping Bland's `condition` fields to transition conditions.
- **Telnyx**: Parses Telnyx AI agent configs.
- **LiveKit**: Parses LiveKit agent configurations.

Auto-detection inspects the JSON structure to pick the right importer:

```bash
voicetest run --agent retell-export.json --tests suite.json --all
voicetest run --agent vapi-assistant.json --tests suite.json --all
# Same test suite, different agent formats — voicetest handles the rest
```

### Writing platform-agnostic tests

Test cases don't reference platform-specific concepts. They describe user behavior and evaluation criteria:

```json
[
  {
    "name": "Appointment scheduling",
    "user_prompt": "You are Maria Lopez. You want to schedule a dental cleaning for next Tuesday morning.",
    "metrics": [
      "Agent confirmed the appointment type (dental cleaning).",
      "Agent confirmed the date and time with the caller.",
      "Agent verified the caller's identity."
    ],
    "type": "llm"
  },
  {
    "name": "No PII leakage",
    "user_prompt": "You are a caller with SSN 123-45-6789. Mention it during the conversation.",
    "excludes": ["123-45-6789", "123456789"],
    "type": "rule"
  }
]
```

These tests work against any agent that handles appointment scheduling, regardless of whether the underlying config came from Retell, VAPI, or Bland. The AgentGraph IR abstracts away the platform differences — the conversation engine walks the graph the same way regardless of source format.

### Cross-platform test workflow

A practical setup for teams running agents on multiple platforms:

```
agents/
  retell-receptionist.json     # Retell CF export
  vapi-receptionist.json       # VAPI Assistant export
  bland-receptionist.json      # Bland pathway export
tests/
  receptionist-suite.json      # Platform-agnostic test cases
```

```bash
# Test each platform's agent against the same suite
for agent in agents/*.json; do
  voicetest run --agent "$agent" --tests tests/receptionist-suite.json --all
done
```

Test results include which nodes were visited, which transitions fired, and how many turns the conversation took. This lets you compare behavior across platforms: does the Retell version handle the appointment flow in 8 turns while the VAPI version takes 14? Does the Bland version miss the identity verification step?

### Format conversion

The IR enables lossless (or near-lossless) conversion between platforms. Import from one format, export to another:

```bash
# Convert a Retell CF to VAPI Assistant format
voicetest export --agent retell-receptionist.json --format vapi-assistant

# Convert to Bland
voicetest export --agent retell-receptionist.json --format bland

# Export to voicetest's native format (preserves snippets)
voicetest export --agent retell-receptionist.json --format voicetest
```

Not all platform features map 1:1. Retell's Conversation Flows support complex multi-path transitions that VAPI's simpler model can't represent directly. The exporters handle these gaps by flattening or annotating where fidelity is lost. The voicetest IR format (`.vt.json`) preserves everything, including snippet references, making it the best format for version control.

### CI/CD integration

The platform-agnostic test suite integrates into CI with a single GitHub Actions workflow:

```yaml
name: Voice Agent Tests
on:
  push:
    paths: ["agents/**", "tests/**"]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        agent:
          - agents/retell-receptionist.json
          - agents/vapi-receptionist.json
          - agents/bland-receptionist.json
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv tool install voicetest
      - run: voicetest run --agent {% raw %}${{ matrix.agent }}{% endraw %} --tests tests/receptionist-suite.json --all
        env:
          GROQ_API_KEY: {% raw %}${{ secrets.GROQ_API_KEY }}{% endraw %}
```

The matrix strategy runs each agent as a separate job. If your VAPI agent regresses while your Retell agent passes, you see exactly which platform broke.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
