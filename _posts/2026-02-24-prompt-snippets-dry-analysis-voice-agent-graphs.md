---
title: "Prompt Snippets and Auto-DRY Analysis for Voice Agent Graphs"
description: "How voicetest detects duplicated prompt text across agent nodes and extracts reusable snippets to keep your configs maintainable and your tests consistent."
date: 2026-02-24
---
{% raw %}
Voice agent configs accumulate duplicated text fast. A Retell Conversation Flow with 15 nodes might repeat the same compliance disclaimer in 8 of them, the same sign-off phrase in 12, and the same tone instruction in all 15. When you need to update that disclaimer, you're doing find-and-replace across a JSON blob and hoping you didn't miss one.

Voicetest 0.23 adds prompt snippets and automatic DRY analysis to fix this. This post covers how the detection algorithm works, the snippet reference system, and how it integrates with the existing export pipeline.

### The problem in concrete terms

Here's a simplified agent graph with three nodes. Notice the repeated text:

```json
{
  "nodes": {
    "greeting": {
      "state_prompt": "Welcome the caller. Always be professional and empathetic in your responses. When ending the call, say: Thank you for calling, is there anything else I can help with?"
    },
    "billing": {
      "state_prompt": "Help with billing inquiries. Always be professional and empathetic in your responses. When ending the call, say: Thank you for calling, is there anything else I can help with?"
    },
    "transfer": {
      "state_prompt": "Transfer to a human agent. Always be professional and empathetic in your responses. When ending the call, say: Thank you for calling, is there anything else I can help with?"
    }
  }
}
```

Two sentences are duplicated across all three nodes. In a real agent with 15-20 nodes, this kind of duplication is the norm. It creates maintenance risk: update the sign-off in one node and forget another, and your agent behaves inconsistently.

### How the DRY analyzer works

The `voicetest.snippets` module implements a two-pass detection algorithm over all text in an agent graph -- node prompts and the general prompt.

**Pass 1: Exact matches.** `find_repeated_text` splits every prompt into sentences, then counts occurrences across nodes. Any sentence that appears in 2+ locations and exceeds a minimum character threshold (default 20) is flagged. The result includes the matched text and which node IDs contain it.

```python
from voicetest.snippets import find_repeated_text

results = find_repeated_text(graph, min_length=20)
for match in results:
    print(f"'{match.text}' found in nodes: {match.locations}")
```

**Pass 2: Fuzzy matches.** `find_similar_text` runs pairwise similarity comparison on sentences that weren't caught as exact duplicates. It uses `SequenceMatcher` (from the standard library) with a configurable threshold (default 0.8). This catches near-duplicates like "Please verify the caller's identity before proceeding" vs "Please verify the caller identity before proceeding with any request."

```python
from voicetest.snippets import find_similar_text

results = find_similar_text(graph, threshold=0.8, min_length=30)
for match in results:
    print(f"Similarity {match.similarity:.0%}: {match.texts}")
```

The `suggest_snippets` function runs both passes and returns a combined result:

```python
from voicetest.snippets import suggest_snippets

suggestions = suggest_snippets(graph, min_length=20)
print(f"Exact duplicates: {len(suggestions.exact)}")
print(f"Fuzzy matches: {len(suggestions.fuzzy)}")
```

### The snippet reference system

Snippets use `{%name%}` syntax (percent-delimited braces) to distinguish them from dynamic variables (`{{name}}`). They're defined at the agent level and expanded before variable substitution:

```json
{
  "snippets": {
    "tone": "Always be professional and empathetic in your responses.",
    "sign_off": "Thank you for calling, is there anything else I can help with?"
  },
  "nodes": {
    "greeting": {
      "state_prompt": "Welcome the caller. {%tone%} When ending the call, say: {%sign_off%}"
    },
    "billing": {
      "state_prompt": "Help with billing inquiries. {%tone%} When ending the call, say: {%sign_off%}"
    }
  }
}
```

### Expansion ordering

During a test run, the `ConversationEngine` expands snippets first, then substitutes dynamic variables:

```python
# In ConversationEngine.process_turn():
general_instructions = expand_snippets(self._module.instructions, self.graph.snippets)
state_instructions = expand_snippets(state_module.instructions, self.graph.snippets)
general_instructions = substitute_variables(general_instructions, self._dynamic_variables)
state_instructions = substitute_variables(state_instructions, self._dynamic_variables)
```

This ordering matters. Snippets are static text blocks resolved at expansion time. Variables are runtime values (caller name, account ID, etc.) that differ per conversation. Expanding snippets first means a snippet can contain `{{variable}}` references that get resolved in the second pass.

### Export modes

When an agent uses snippets, the export pipeline offers two modes:

- **Raw (`.vt.json`)**: Preserves `{%snippet%}` references and the snippets dictionary. This is the voicetest-native format for version control and sharing.
- **Expanded**: Resolves all snippet references to plain text. Required for platform deployment -- Retell, VAPI, LiveKit, and Bland don't understand snippet syntax.

The `expand_graph_snippets` function produces a deep copy with all references resolved:

```python
from voicetest.templating import expand_graph_snippets

expanded = expand_graph_snippets(graph)
# expanded.snippets == {}
# expanded.nodes["greeting"].state_prompt contains the full text
# original graph is unchanged
```

Platform-specific exporters (Retell, VAPI, Bland, Telnyx, LiveKit) always receive expanded graphs. The voicetest IR exporter preserves references.

### REST API

The snippet system is fully exposed via REST:

```bash
# List snippets
GET /api/agents/{id}/snippets

# Create/update a snippet
PUT /api/agents/{id}/snippets/tone
{"text": "Always be professional and empathetic."}

# Delete a snippet
DELETE /api/agents/{id}/snippets/tone

# Run DRY analysis
POST /api/agents/{id}/analyze-dry
# Returns: {"exact": [...], "fuzzy": [...]}

# Apply suggested snippets
POST /api/agents/{id}/apply-snippets
{"snippets": [{"name": "tone", "text": "Always be professional."}]}
```

### Web UI

In the agent view, the Snippets section shows all defined snippets with inline editing. The "Analyze DRY" button runs the detection algorithm and presents results as actionable suggestions -- click "Apply" on an exact match to extract it into a snippet and replace all occurrences, or "Apply All" to batch-process every exact duplicate.

### Why this matters for testing

Duplicated prompts aren't just a maintenance problem -- they're a testing problem. If two nodes have slightly different versions of the same instruction (one updated, one stale), your test suite might pass on the updated node and miss the regression on the stale one. Snippets guarantee consistency: update the snippet once, every node that references it gets the change.

Combined with [voicetest's LLM-as-judge evaluation](/blog/voice-agent-evaluation-llm-judges/), snippets make your test results more reliable. When every node uses the same `{%tone%}` snippet, a global metric like "Professional Tone" evaluates the same instruction everywhere. No more false passes from nodes running outdated prompt text.

### Getting started

```bash
uv tool install voicetest
voicetest demo --serve
```

Voicetest is open source under Apache 2.0. [GitHub](https://github.com/voicetestdev/voicetest). [Docs](https://voicetest.dev/api/).
{% endraw %}
