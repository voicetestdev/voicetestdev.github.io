---
title: "Five Node Types and Global Interrupts: How Voicetest Models Real Agent Graphs"
description: "Logic, extract, end, transfer, and conversation nodes — plus global interrupts with go-back semantics. The voicetest IR now models the full structure of platform-specific agent graphs without losing fidelity."
date: 2026-05-08
---
{% raw %}
A voice agent graph is more than a list of conversational states with prompts. Real agents extract variables from what the caller said and branch on them. They evaluate deterministic equations to route by account type or balance. They end the call cleanly or transfer to a human. And they need ways to handle interrupts — "speak to a manager," "I want to start over," "this is an emergency" — from anywhere in the flow without an explicit edge from every node.

Earlier versions of voicetest's AgentGraph IR modeled all of this as one undifferentiated node type with prompts and transitions. That was enough for round-tripping simple flows but lost structure on import: a Retell logic node and a Retell conversation node looked identical in the IR, the engine drove them both the same way, and exports back to Retell rebuilt the structure heuristically.

The IR now distinguishes five node types and a global-node flag. This post covers what each one represents, how the engine handles it, and what it means for round-trip fidelity across platforms.

![A help-desk agent graph rendered in voicetest's web UI. Green: entry conversation node. Hexagon: extract node with the variable it pulls out. Diamond: logic split branching on that variable. Rectangles in the lower half: conversation, end, and transfer flows. Purple-bordered node: a global interrupt reachable from any conversation node, with a go-back edge that returns the caller to where they were.](/assets/blog/agent-graph-node-types.png)

### The node type enum

```python
class NodeType(StrEnum):
    CONVERSATION = "conversation"
    LOGIC = "logic"
    EXTRACT = "extract"
    END = "end"
    TRANSFER = "transfer"
```

Every `AgentNode` carries a `node_type`. The engine routes by type:

| Type           | Speaks? | Behavior                                                                                          |
| -------------- | ------- | ------------------------------------------------------------------------------------------------- |
| `conversation` | yes     | Generates a response from `state_prompt`, evaluates LLM-prompted transitions, fires always-edges  |
| `extract`      | no      | LLM extracts variables from the conversation so far, then branches on equation transitions        |
| `logic`        | no      | Pure deterministic branch — evaluates equation transitions on already-extracted variables         |
| `end`          | depends | If it has a prompt, agent speaks then call ends; if not, call ends immediately                    |
| `transfer`     | depends | Same as end, but the call status reflects a transfer rather than a hangup                         |

The engine's main loop walks past silent nodes (extract, logic, prompt-less end/transfer) without producing user-visible turns and stops at the next conversation node to actually generate a response. That distinction matters: a turn in the transcript should correspond to something the caller hears, not to internal routing.

### Conversation nodes

Conversation nodes are the default. They have a `state_prompt`, a list of LLM-prompted transitions, and optionally tools. The engine generates a response, then evaluates transitions.

```json
{
  "id": "ask_for_dob",
  "node_type": "conversation",
  "state_prompt": "Ask for the caller's date of birth to verify identity.",
  "transitions": [
    {
      "target_node_id": "verify",
      "condition": {"type": "llm_prompt", "value": "Caller provided a date of birth"}
    }
  ]
}
```

When `node_type` is omitted on import (older configs predate the field), `model_post_init` infers it from structure: equation-only transitions plus extract variables → `EXTRACT`; equation-only transitions without extract variables → `LOGIC`. Old graphs keep working without re-export.

### Extract nodes

Extract nodes are the bridge between LLM-driven conversation and deterministic routing. They run an LLM call against the conversation history to extract structured variables, then branch on those variables using equation transitions.

```json
{
  "id": "classify_intent",
  "node_type": "extract",
  "variables_to_extract": [
    {
      "name": "intent",
      "type": "string",
      "choices": ["billing", "technical", "cancel", "other"]
    }
  ],
  "transitions": [
    {
      "target_node_id": "billing_flow",
      "condition": {
        "type": "equation",
        "equations": [{"left": "intent", "operator": "==", "right": "billing"}]
      }
    },
    {
      "target_node_id": "tech_flow",
      "condition": {
        "type": "equation",
        "equations": [{"left": "intent", "operator": "==", "right": "technical"}]
      }
    }
  ]
}
```

The engine runs extraction silently — no message to the caller — then evaluates the equation transitions and continues. If extraction succeeds, the next conversation node speaks. If no equation matches, the node falls through to an always-edge fallback.

### Logic nodes

Logic nodes are pure branches. No LLM call, no extraction, just equation evaluation against variables that earlier extract nodes already populated:

```json
{
  "id": "branch_on_balance",
  "node_type": "logic",
  "transitions": [
    {
      "target_node_id": "collections_flow",
      "condition": {
        "type": "equation",
        "equations": [{"left": "balance", "operator": "<", "right": "0"}]
      }
    },
    {
      "target_node_id": "standard_flow",
      "condition": {"type": "always", "value": ""}
    }
  ]
}
```

The split between extract and logic mirrors what platforms like Retell distinguish: one class of node that costs an LLM call (extraction) and one class that is essentially free (logic). Round-tripping through the IR preserves this — exports rebuild the right node type rather than collapsing both into a generic conversation node.

### End and transfer nodes

End and transfer nodes terminate the call. They behave the same way structurally; the difference is what the platform reports as the disconnect reason. Both can carry a prompt — "Thanks for calling, goodbye" — in which case the agent speaks one final turn before the engine sets `end_call_invoked`. If the prompt is empty, the call ends immediately with no final turn.

```json
{
  "id": "wrap_up",
  "node_type": "end",
  "state_prompt": "Thank the caller and end the call."
}
```

```json
{
  "id": "transfer_to_human",
  "node_type": "transfer",
  "state_prompt": ""
}
```

### Global nodes: interrupts from anywhere

The five types above describe nodes by their *internal* behavior. Global nodes describe a property of how a node is *reached*. Any node can be marked global by adding a `global_node_setting`:

```json
{
  "id": "speak_to_manager",
  "node_type": "conversation",
  "state_prompt": "The caller has asked for a manager. Acknowledge, take their name and a brief reason, and tell them a manager will follow up within an hour.",
  "global_node_setting": {
    "condition": "Caller asks for a manager, supervisor, or to escalate",
    "go_back_conditions": [
      {
        "id": "back_to_origin",
        "condition": {
          "type": "llm_prompt",
          "value": "Caller is ready to continue with the original request"
        }
      }
    ]
  }
}
```

A global node is reachable from every conversation node without an explicit edge. The engine appends global entry conditions to every conversation node's transition options when it formats them for the LLM:

```python
available_transitions = self._module.format_transitions(
    self._current_node, originator_id=self._current_originator
)
```

The LLM picks from local transitions and global transitions in a single call, so ordering is unambiguous and there's no first-match-wins ambiguity. Zero global nodes means identical behavior to before — the feature is fully backward-compatible.

### Originator stack and go-back

Global nodes are not just one-way trapdoors. They support returning to the node that originated the interrupt:

```python
# Go-back: source is global and target matches the originator
if (
    source_node.global_node_setting
    and self._originator_stack
    and self._originator_stack[-1] == target
):
    self._originator_stack.pop()
elif target_node and target_node.global_node_setting:
    # Entering a global node: push current node as originator
    self._originator_stack.append(self._current_node)
elif source_node.global_node_setting and self._originator_stack:
    # Leaving a global node forward via regular edge: pop originator
    self._originator_stack.pop()
```

The originator is a *stack*, not a single value. That matters because globals can stack: caller asks for a manager (push A → speak_to_manager), then says "actually, this is an emergency" while in the manager flow (push speak_to_manager → emergency). When the emergency resolves, the engine pops back to speak_to_manager; when that resolves, it pops back to A.

Three transition events touch the stack:

1. **Enter global** → push current node as originator.
2. **Go-back to originator** → pop the stack; conversation resumes at the originating node with the full transcript context intact.
3. **Forward exit from a global** (via a regular non-go-back edge) → pop the stack; the original "where was I?" is now stale.

### What this buys you

For agent authors:

- **Cleaner graphs.** No more O(N) busywork adding "transfer to manager" edges from every conversation node. Add one global node, set the condition, done.
- **Resumable interrupts.** "Speak to a manager" and similar flows can return the caller to whatever they were trying to do, with full context. The conversation feels like a real interaction, not a state-machine reset.
- **Stackable interrupts.** Emergency paths nested inside escalation paths nested inside main flows all return correctly.

For voicetest internals:

- **Lossless round-trip with Retell CF.** Retell's `global_node_setting` (condition + go-back conditions) imports and exports with full fidelity.
- **Type-safe routing.** The engine's `is_logic_node()` / `is_extract_node()` / `is_end_node()` / `is_transfer_node()` checks replace structural heuristics. Exporters can target the right platform construct deterministically.
- **Backward compatibility.** Graphs imported under the old IR still load — `model_post_init` infers `node_type` from structure when the field is absent.

### Cross-platform implications

The five node types map cleanly onto what most platforms already model — Retell's distinction between conversation, extract, logic, end, and transfer nodes is mirrored in the IR; VAPI's simpler model maps multiple IR types onto the same VAPI primitive on export, with a note about the lossy step. Global nodes round-trip faithfully on platforms that support them and degrade gracefully (rendered as a normal node with explicit edges from all conversation nodes, where feasible) on platforms that don't.

This is the point of having an IR at all: model the union of platform features, export the intersection, annotate the difference. The richer the IR, the less fidelity is lost when you import from one platform, edit the graph, and push to another.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
{% endraw %}
