---
title: "Running Voice Agent Tests Fully Offline With Local Models"
description: "Models that fit on one consumer GPU now score where the frontier was a year ago. Route every LLM call voicetest makes, for the agent, the simulated caller, and the judge, through a local llama.cpp server: no API keys, no per-token cost, and no agent configs or transcripts leaving your machine."
date: 2026-11-18
image: /assets/blog/2026-11-18/cover.png
---
Running voice agent tests against cloud APIs means sending your agent configs and test transcripts to a third-party provider. For some teams, such as healthcare, finance, and government, that's not allowed. For everyone else, the per-token cost adds up during development: every test run is a multi-turn conversation plus a judge pass, and you rerun the suite each time you touch a prompt.

Voicetest can run fully offline, with every LLM call going to a model on your own machine. No API keys, no network calls to model providers, and nothing leaves your machine. This post covers why that's now practical, the setup with llama.cpp, how to pick a model for each role, and when to use local versus cloud models.

### Why this is practical now

Not long ago, a model that fit on one consumer GPU was a poor stand-in for the hosted models most voice agents run on. That has changed quickly.

![Line chart of the Epoch Capabilities Index for the best model released to date, from 2023 to November 2026. The best closed model rises from GPT-4 at 122 to Claude Fable 5.2 at 174. The best open-weight model of 40 billion parameters or fewer rises from Mistral 7B at 103 in September 2023 to Qwen 3.8 27B at 153 in August 2026, one point above where GPT-5 scored at its release a year earlier.](/assets/blog/2026-11-18/local-vs-frontier-eci.svg)

[Epoch AI's Capabilities Index](https://epoch.ai/benchmarks) combines dozens of benchmarks onto one scale. On it, the best open-weight model small enough for one consumer GPU went from 103 (Mistral 7B, September 2023) to 153 (Qwen 3.8 27B, August 2026). That's above GPT-5, which scored 150 when it was released in August 2025.

Local models aren't closing the gap. The frontier keeps moving, and the distance in points is about what it was in 2024. What they do is track the frontier roughly a year behind, and a year-old frontier model is plenty for playing a caller and iterating on prompts. Treat benchmark scores for small models with some care, though: Epoch notes they can be tuned toward benchmarks, so check results on your own test suite before trusting them.

### Setup

[llama.cpp](https://github.com/ggml-org/llama.cpp) is the open-source inference engine most local model tools are built on. It ships `llama-server`, a single binary that downloads a model from Hugging Face and serves it over an OpenAI-compatible API:

```bash
brew install llama.cpp
llama-server -hf Qwen/Qwen3-8B-GGUF:Q4_K_M --port 8080 --reasoning-budget 0
```

Qwen3 models think before they answer by default. `--reasoning-budget 0` turns that off. Thinking slows down every turn, and a long think can use up the response budget before any answer comes out, which shows up in voicetest as parse errors and retries.

Voicetest routes model calls through LiteLLM, and LiteLLM's `llamafile/` provider talks to an OpenAI-compatible server at `http://127.0.0.1:8080/v1`, which is where `llama-server` listens. Point all three roles at it:

```toml
# .voicetest/settings.toml
[models]
agent = "llamafile/qwen3-8b"
simulator = "llamafile/qwen3-8b"
judge = "llamafile/qwen3-8b"
```

The name after `llamafile/` is only a label; the server answers with whatever model it loaded. If the server runs somewhere else, set `LLAMAFILE_API_BASE`. Then run tests as usual:

```bash
voicetest run --agent agents/receptionist.json --tests tests/receptionist.json --all
```

### Picking a model for each role

The three roles don't need the same capability, and it helps to know which one is limiting you.

**Simulator.** The simulator plays the caller. It follows a persona prompt and moves toward the caller's goal. It's the easiest role, and an 8B model handles it well.

**Agent.** The agent plays your voice agent. It follows the graph's prompts, keeps track of where it is in the conversation, and chooses transitions. This is the hardest role. Bigger models follow complex graphs more consistently, and a small local model won't behave exactly like the model your production agent runs on. For graphs with real branching, use the strongest model that fits:

```bash
llama-server -hf unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M --port 8080 --reasoning-budget 0
```

**Judge.** The judge reads each finished transcript and scores it against every metric, explaining its reasoning. An inaccurate judge is worse than a slow one, because it gives you false passes or false alarms. Use the largest model you can run.

One `llama-server` serves one model, so the simplest setup gives every role the same one, sized for the hardest role you run locally.

To check that everything runs end to end, a tiny model is enough:

```bash
llama-server -hf Qwen/Qwen3-0.6B-GGUF:Q8_0 --port 8080 --reasoning-budget 0
voicetest smoke-test --model llamafile/qwen3-0.6b
```

That confirms the pipeline works. It says nothing about whether a prompt change is safe to ship: a model this small will fail plenty of metrics.

### Hardware

Memory is the limit. An 8B model at Q4 fits comfortably on a laptop. A 27B model at Q4 needs around 17 GB for its weights, which means a 24 GB GPU or an Apple Silicon Mac with 32 GB or more. Anything much larger needs a workstation.

If you can't run a strong judge locally, there's a practical middle ground: keep the agent and simulator local, and point only the judge at a cloud model:

```toml
[models]
agent = "llamafile/qwen3.8-27b"
simulator = "llamafile/qwen3.8-27b"
judge = "openai/gpt-4o"
```

Only judge calls leave your machine, and they contain the simulated transcript and your metric criteria. They don't include your agent's graph or prompts.

### The full voice stack, also local

`voicetest up` starts the services for live audio calls in Docker: a LiveKit server, faster-whisper for speech-to-text, and Kokoro for text-to-speech. With a local model configured, a [live call test](https://voicetest.dev/blog/live-call-testing-what-the-caller-heard/) runs entirely on your machine, through the real audio pipeline, with nothing sent to an outside service.

### What you give up

Local runs are slower than hosted inference. Every turn of every conversation waits on your hardware, and the judge pass adds a generation per metric. A suite that finishes in a few minutes against a hosted provider can take much longer locally. That's fine for a suite you run while you work on prompts, and less fine for a CI check on every pull request.

Results are also only as good as the model. A local judge that's much weaker than the model you'd normally use will pass things a stronger judge would fail. Check a sample of local results against a stronger judge before you rely on them.

### When to use each

| Scenario | Recommendation |
|---|---|
| Iterating on prompts, rerunning tests often | Local |
| Data can't leave your network | Local, or local with a self-hosted judge |
| CI on every pull request | Hosted provider, for speed |
| Final check before deploying | Strongest judge you have access to |
| Limited hardware | Hosted provider, or local agent and simulator with a hosted judge |

Models can be swapped per role in settings without touching your tests. Iterate locally while you work, then let CI check the same suite against a hosted model before merging.

*Chart data: Epoch AI, [Epoch Capabilities Index](https://epoch.ai/benchmarks), November 2026 snapshot, licensed CC-BY 4.0.*

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
