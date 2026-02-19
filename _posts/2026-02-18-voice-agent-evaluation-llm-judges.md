---
title: "Voice Agent Evaluation with LLM Judges: How It Works"
description: "How voicetest uses a three-model architecture to simulate conversations and evaluate voice agents with LLM-as-judge scoring."
date: 2026-02-18
---

You can write unit tests for a REST API. You can snapshot-test a React component. But how do you test a voice agent that holds free-form conversations?

The core challenge: voice agent behavior is non-deterministic. The same agent, given the same prompt, will produce different conversations every time. Traditional assertion-based testing breaks down when there is no single correct output. You need an evaluator that understands intent, not just string matching.

Voicetest solves this with LLM-as-judge evaluation. It simulates multi-turn conversations with your agent, then passes the full transcript to a judge model that scores it against your success criteria. This post explains how each piece works.

### The three-model architecture

Voicetest uses three separate LLM roles during a test run:

**Simulator.** Plays the user. Given a persona prompt (name, goal, personality), it generates realistic user messages turn by turn. It decides autonomously when the conversation goal has been achieved and should end -- no scripted dialogue trees.

**Agent.** Plays your voice agent. Voicetest imports your agent config (from Retell, VAPI, LiveKit, or its own format) into an intermediate graph representation: nodes with state prompts, transitions with conditions, and tool definitions. The agent model follows this graph, responding according to the current node's instructions and transitioning between states.

**Judge.** Evaluates the finished transcript. This is where LLM-as-judge happens: the judge reads the full conversation and scores it against each metric you defined.

You can assign different models to each role. Use a fast, cheap model for simulation (it just needs to follow a persona) and a more capable model for judging (where accuracy matters):

```toml
[models]
simulator = "groq/llama-3.1-8b-instant"
agent = "groq/llama-3.3-70b-versatile"
judge = "openai/gpt-4o"
```

### How simulation works

Each test case defines a user persona:

```json
{
  "name": "Appointment reschedule",
  "user_prompt": "You are Maria Lopez, DOB 03/15/1990. You need to reschedule your Thursday appointment to next week. You prefer mornings.",
  "metrics": [
    "Agent verified the patient's identity before making changes.",
    "Agent confirmed the new appointment date and time."
  ],
  "type": "llm"
}
```

Voicetest starts the conversation at the agent's entry node. The simulator generates a user message based on the persona. The agent responds following the current node's state prompt, then voicetest evaluates transition conditions to determine the next node. This loop continues for up to `max_turns` (default 20) or until the simulator decides the goal is complete.

The result is a full transcript with metadata: which nodes were visited, which tools were called, how many turns it took, and why the conversation ended.

### How the judge scores

After simulation, the judge evaluates each metric independently. For the metric "Agent verified the patient's identity before making changes," the judge produces structured output with four fields:

- **Analysis**: Breaks compound criteria into individual requirements and quotes transcript evidence for each. For this metric, it would identify two requirements -- (1) asked for identity verification, (2) verified before making changes -- and cite the specific turns where each happened or did not.
- **Score**: 0.0 to 1.0, based on the fraction of requirements met. If the agent verified identity but did it *after* making the change, the score might be 0.5.
- **Reasoning**: A summary of what passed and what failed.
- **Confidence**: How certain the judge is in its assessment.

A test passes when all metric scores meet the threshold (default 0.7, configurable per-agent or per-metric).

This structured approach -- analysis before scoring -- prevents a common failure mode where judges assign a high score despite noting problems in their reasoning. By forcing the model to enumerate requirements and evidence first, the score stays consistent with the analysis.

### Rule tests: when you do not need an LLM

Not everything requires a judge. Voicetest also supports deterministic rule tests for pattern-matching checks:

```json
{
  "name": "No SSN in transcript",
  "user_prompt": "You are Jane, SSN 123-45-6789. Ask the agent to verify your identity.",
  "excludes": ["123-45-6789", "123456789"],
  "type": "rule"
}
```

Rule tests check for `includes` (required substrings), `excludes` (forbidden substrings), and `patterns` (regex). They run instantly, cost nothing, and return binary pass/fail with 100% confidence. Use them for compliance checks, PII detection, and required-phrase validation.

### Global metrics: compliance at scale

Individual test metrics evaluate specific scenarios. Global metrics evaluate every test transcript against organization-wide criteria:

```json
{
  "global_metrics": [
    {
      "name": "HIPAA Compliance",
      "criteria": "Agent verifies patient identity before disclosing any protected health information.",
      "threshold": 0.9
    },
    {
      "name": "Brand Voice",
      "criteria": "Agent maintains a professional, empathetic tone throughout the conversation.",
      "threshold": 0.7
    }
  ]
}
```

Global metrics run on every test automatically. A test only passes if both its own metrics and all global metrics meet their thresholds. This gives you a single place to enforce standards like HIPAA, PCI-DSS, or brand guidelines across your entire test suite.

### Putting it together

A complete test run looks like this:

1. Voicetest imports your agent config into its graph representation.
2. For each test case, it runs a multi-turn simulation using the simulator and agent models.
3. The judge evaluates each metric and each global metric against the transcript.
4. Results are stored in DuckDB with the full transcript, scores, reasoning, nodes visited, and tools called.
5. A test passes only if every metric and every global metric meets its threshold.

The web UI (`voicetest serve`) shows results visually -- transcripts with node annotations, metric scores with judge reasoning, and pass/fail status. The CLI outputs the same data to stdout for CI integration.

### Getting started

```bash
uv tool install voicetest
voicetest demo --serve
```

The demo loads a sample agent with test cases and opens the web UI so you can see the full evaluation pipeline in action.

Voicetest is open source under Apache 2.0. [GitHub](https://github.com/voicetestdev/voicetest). [Docs](https://voicetest.dev/api/).
