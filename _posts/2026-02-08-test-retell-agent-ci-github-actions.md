---
title: "How to Test a Retell Agent in CI with GitHub Actions"
description: "Set up automated voice agent testing for your Retell Conversation Flow. Import your agent, write test cases, and run them on every push with GitHub Actions."
date: 2026-02-08
---

Manual testing of voice agents does not scale. You click through a few conversations in the Retell dashboard, confirm the agent sounds right, and ship it. Then someone updates a prompt, a transition breaks, and you find out from a customer complaint. The feedback loop is days, not minutes.

Voicetest fixes this. It imports your Retell Conversation Flow, simulates multi-turn conversations using an LLM, and evaluates the results with an LLM judge that produces scores and reasoning. You can run it locally, but the real value comes from running it in CI on every push.

This post walks through the full setup: from installing voicetest to a working GitHub Actions workflow that tests your Retell agent automatically.

### Step 1: Install voicetest

Voicetest is a Python CLI tool published on PyPI. The recommended way to install it is as a uv tool:

```bash
uv tool install voicetest
```

Verify it works:

```bash
voicetest --version
```

### Step 2: Export your Retell agent

In the Retell dashboard, open your Conversation Flow and export it as JSON. Save it to your repo:

```
agents/
  receptionist.json
```

The exported JSON contains your nodes, edges, prompts, transition conditions, and tool definitions. Voicetest auto-detects the Retell format by looking for `start_node_id` and `nodes` in the JSON.

If you prefer to pull the config programmatically (useful for keeping tests in sync with the live agent), voicetest can also fetch directly from the Retell API:

```bash
export RETELL_API_KEY=your_key_here
```

### Step 3: Write test cases

Create a test file with one or more test cases. Each test defines a simulated user persona, what the user will do, and metrics for the LLM judge to evaluate:

```json
[
  {
    "name": "Billing inquiry",
    "user_prompt": "Say you are Jane Smith with account 12345. You're confused about a charge on your bill and want help understanding it.",
    "metrics": [
      "Agent greeted the customer and addressed the billing concern.",
      "Agent was helpful and professional throughout."
    ],
    "type": "llm"
  },
  {
    "name": "No PII in transcript",
    "user_prompt": "You are Jane with SSN 123-45-6789. Verify your identity.",
    "includes": ["verified", "identity"],
    "excludes": ["123-45-6789", "123456789"],
    "type": "rule"
  }
]
```

There are two test types. LLM tests (`"type": "llm"`) run a full multi-turn simulation and then pass the transcript to an LLM judge, which scores each metric from 0.0 to 1.0 with written reasoning. Rule tests (`"type": "rule"`) use deterministic pattern matching -- checking that the transcript includes required strings, excludes forbidden ones, or matches regex patterns. Rule tests are fast and free, good for compliance checks like PII leakage.

Save this as `agents/receptionist-tests.json`.

### Step 4: Configure your LLM backend

Voicetest uses LiteLLM model strings, so any provider works. Create a `.voicetest/settings.toml` in your project root:

```toml
[models]
agent = "groq/llama-3.3-70b-versatile"
simulator = "groq/llama-3.1-8b-instant"
judge = "groq/llama-3.3-70b-versatile"

[run]
max_turns = 20
verbose = false
```

The `simulator` model plays the user. It should be fast and cheap since it just follows the persona script. The `judge` model evaluates the transcript and should be accurate. The `agent` model plays the role of your voice agent, following the prompts and transitions from your Retell config.

### Step 5: Run locally

Before setting up CI, verify everything works:

```bash
export GROQ_API_KEY=your_key_here

voicetest run \
  --agent agents/receptionist.json \
  --tests agents/receptionist-tests.json \
  --all
```

You will see each test run, the simulated conversation, and the judge's scores. Fix any test definitions that do not match your agent's behavior, then commit everything:

```bash
git add agents/ .voicetest/settings.toml
git commit -m "Add voicetest config and test cases"
```

### Step 6: Set up GitHub Actions

Add your API key as a repository secret. Go to Settings > Secrets and variables > Actions, and add `GROQ_API_KEY`.

Then create `.github/workflows/voicetest.yml`:

```yaml
name: Voice Agent Tests

on:
  push:
    paths:
      - "agents/**"
  pull_request:
    paths:
      - "agents/**"
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        run: uv python install 3.12

      - name: Install voicetest
        run: uv tool install voicetest

      - name: Run voice agent tests
        env:
          GROQ_API_KEY: {% raw %}${{ secrets.GROQ_API_KEY }}{% endraw %}
        run: |
          voicetest run \
            --agent agents/receptionist.json \
            --tests agents/receptionist-tests.json \
            --all
```

The workflow triggers on any change to files in `agents/`, which means prompt edits, new test cases, or config changes all trigger a test run. The `workflow_dispatch` trigger lets you run tests manually from the GitHub UI.

### What's next

Once you have CI working, there are a few things worth exploring:

**Global compliance metrics.** Voicetest supports HIPAA and PCI-DSS compliance checks that run across the entire transcript, not just per-test. These catch issues like agents accidentally reading back credit card numbers or disclosing PHI.

**Format conversion.** If you ever want to move from Retell to VAPI or LiveKit, voicetest can convert your agent config between platforms via its AgentGraph intermediate representation:

```bash
voicetest export --agent agents/receptionist.json --format vapi-assistant
```

**The web UI.** For a visual interface during development, run `voicetest serve` and open `http://localhost:8000`. You get a dashboard with test results, transcripts, and scores.

Voicetest is open source under Apache 2.0. [GitHub](https://github.com/voicetestdev/voicetest). [Docs](https://voicetest.dev/api/).
