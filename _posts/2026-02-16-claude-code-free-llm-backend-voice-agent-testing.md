---
title: "Using Claude Code as a Free LLM Backend for Voice Agent Testing"
description: "Route voicetest LLM calls through your Claude Pro or Max subscription instead of paying per-token API costs. No API keys, no proxy — just the claude CLI."
date: 2026-02-16
---

Running a voice agent test suite means making a lot of LLM calls. Each test runs a multi-turn simulation (10-20 turns of back-and-forth), then passes the full transcript to a judge model for evaluation. A suite of 20 tests can easily hit 200+ LLM calls. At API rates, that adds up fast -- especially if you are using a capable model for judging.

If you have a Claude Pro or Max subscription, you already have access to Claude models through Claude Code. Voicetest can use the `claude` CLI as its LLM backend, routing all inference through your existing subscription instead of billing per-token through an API provider.

### How it works

Voicetest has a built-in Claude Code provider. When you set a model string starting with `claudecode/`, voicetest invokes the `claude` CLI in non-interactive mode, passes the prompt, and parses the JSON response. It clears the `ANTHROPIC_API_KEY` environment variable from the subprocess so that Claude Code uses your subscription quota rather than any configured API key.

No proxy server. No API key management. Just the `claude` binary on your PATH.

### Step 1: Install Claude Code

Follow the instructions at [claude.ai/claude-code](https://claude.ai/claude-code). After installation, verify it works:

```bash
claude --version
```

Make sure you are logged in to your Claude account.

### Step 2: Install voicetest

```bash
uv tool install voicetest
```

### Step 3: Configure settings.toml

Create `.voicetest/settings.toml` in your project directory:

```toml
[models]
agent = "claudecode/sonnet"
simulator = "claudecode/haiku"
judge = "claudecode/sonnet"

[run]
max_turns = 20
verbose = false
```

The model strings follow the pattern `claudecode/<variant>`. The supported variants are:

- `claudecode/haiku` -- Fast, cheap on quota. Good for simulation.
- `claudecode/sonnet` -- Balanced. Good for judging and agent simulation.
- `claudecode/opus` -- Most capable. Use when judging accuracy matters most.

### Step 4: Run tests

```bash
voicetest run \
  --agent agents/my-agent.json \
  --tests agents/my-tests.json \
  --all
```

No API keys needed. Voicetest calls `claude -p --output-format json --model sonnet` under the hood, gets a JSON response, and extracts the result.

### Model mixing

The three model roles in voicetest serve different purposes, and you can mix models to optimize for speed and accuracy:

**Simulator** (`simulator`): Plays the user persona. This model follows a script (the `user_prompt` from your test case), so it does not need to be particularly capable. Haiku is a good fit -- it is fast and consumes less of your quota.

**Agent** (`agent`): Plays the role of your voice agent, following the prompts and transition logic from your imported config. Sonnet handles this well.

**Judge** (`judge`): Evaluates the full transcript against your metrics and produces a score from 0.0 to 1.0 with written reasoning. This is where accuracy matters most. Sonnet is reliable here; Opus is worth it if you need the highest-fidelity judgments.

A practical configuration:

```toml
[models]
agent = "claudecode/sonnet"
simulator = "claudecode/haiku"
judge = "claudecode/sonnet"
```

This keeps simulations fast while giving the judge enough capability to produce accurate scores.

### Cost comparison

With API billing (e.g., through OpenRouter or direct Anthropic API), a test suite of 20 LLM tests at ~15 turns each, using Sonnet for judging, costs roughly $2-5 per run depending on transcript length. Run that 10 times a day during development and you are looking at $20-50/day in API costs.

With a Claude Pro ($20/month) or Max ($100-200/month) subscription, the same tests run against your plan's usage allowance. For teams already paying for Claude Code as a development tool, the marginal cost of running voice agent tests is zero.

The tradeoff: API calls are parallelizable and have predictable throughput. Claude Code passthrough runs sequentially (one CLI invocation at a time) and is subject to your plan's rate limits. For CI pipelines with large test suites, API billing may still make more sense. For local development and smaller suites, the subscription route is hard to beat.

### When to use which

| Scenario | Recommended backend |
|---|---|
| Local development, iterating on prompts | `claudecode/*` |
| Small CI suite (< 10 tests) | `claudecode/*` |
| Large CI suite, parallel runs | API provider (OpenRouter, Anthropic) |
| Team with shared API budget | API provider |
| Solo developer with Max subscription | `claudecode/*` |

### Getting started

```bash
uv tool install voicetest
voicetest demo
```

The `demo` command loads a sample healthcare receptionist agent with test cases so you can try it without any setup.

Voicetest is open source under Apache 2.0. [GitHub](https://github.com/voicetestdev/voicetest). [Docs](https://voicetest.dev/api/).
