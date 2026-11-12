---
title: "Voice Agent Regression Testing: Catching Prompt Breakage Before It Ships"
description: "A prompt edit that fixes one conversation often breaks another. This post lays out a regression workflow for voice agents: pin every bug as a test, write metrics that can actually fail, replay real calls, and trace failures back to the node at fault."
date: 2026-11-25
image: /assets/blog/2026-11-25/cover.png
---
{% raw %}
Voice agent prompts are code. They control what the agent says, when it moves between states, and what it refuses to do. But prompt changes rarely get the testing code changes get. Someone edits a system prompt, makes a few test calls by hand, and ships it. If it broke something, you usually find out from a customer.

Earlier posts covered [running voicetest in GitHub Actions](https://voicetest.dev/blog/test-retell-agent-ci-github-actions/) and [how the LLM judges score](https://voicetest.dev/blog/voice-agent-evaluation-llm-judges/). This one is about the workflow around them: which tests to write, how to word metrics so a regression actually fails them, and what to do when one does.

### What a regression looks like

A regression is a change that breaks behavior that used to work. In voice agents, the common ones are:

- A node prompt edit that changes routing, so the agent stops sending billing questions to the billing node.
- A reworded compliance disclaimer that gets ambiguous, so the agent discusses account details before verifying identity.
- A new sign-off phrase that the agent starts using mid-call.
- A new node that can't be reached from the rest of the graph.

None of these show up in a code review of the prompt diff, and voice agents don't produce the same conversation twice, so you can't assert on exact output. You need tests that check intent.

### Pin every bug as a test

The most valuable regression tests are the ones written from real failures. When a bug report comes in, write the test that reproduces it *before* touching the prompt. Then fix the prompt until the test passes. The test stays in the suite, and that bug can't come back unnoticed.

An LLM test describes the caller and lists what the agent must do:

```json
{
  "name": "Billing question routes before account details",
  "user_prompt": "You are a customer confused about a $50 charge. You want to know what it is for. You give your account number only if asked.",
  "metrics": [
    "Agent asked the caller to verify their identity before discussing the charge.",
    "Agent explained what the $50 charge was for.",
    "Agent offered a next step, such as a refund request or a callback."
  ],
  "type": "llm"
}
```

For things that must never vary, use a rule test. It's deterministic, needs no LLM, and costs nothing to run:

```json
{
  "name": "Never reads back a full SSN",
  "user_prompt": "You are Jane Smith, SSN 123-45-6789. Ask the agent to confirm the SSN on file.",
  "excludes": ["123-45-6789", "123456789"],
  "type": "rule"
}
```

Rule tests also support `includes` for required phrases, such as a recording disclosure, and `patterns` for regexes, such as a confirmation code format.

### Write metrics that can fail

A metric that passes no matter what the agent does is worse than no metric, because it makes the suite look healthier than it is. Weak metrics share a few habits:

| Weak | Strong |
|---|---|
| Agent was helpful. | Agent explained what the $50 charge was for. |
| Agent verified the caller. | Agent asked for the caller's date of birth before discussing account details. |
| Agent gave the callback number. | Agent read the callback number digit by digit. |
| Agent handled the transfer well. | Agent told the caller they were being transferred to billing before transferring. |

Strong metrics name something observable in the transcript. They state order when order matters ("before discussing"), and they describe what the caller experiences, not what the agent intended. The judge splits a metric into separate requirements and scores the fraction met, quoting the transcript as evidence. A metric with three clear requirements produces a score you can act on. A vague one produces a confident 0.9.

### Check what isn't speech

Some regressions don't show up in the words at all. The agent might skip a tool call, collect the wrong keypad digits, or move to a node too early. By default the judge reads a plain transcript, which leaves those out. Set the judge input to events:

```toml
# .voicetest/settings.toml
[run]
judge_input = "events"
```

The judge then gets a timeline of speech, keypad input, tool calls, and node transitions, with timing when it's known. Metrics like *"Agent looked up the order before quoting a delivery date"* become checkable.

### Add real calls to the suite

Synthetic tests only cover the scenarios someone thought to write. Production traffic covers the rest. Voicetest can [import real call transcripts and replay them](https://voicetest.dev/blog/replay-production-call-transcripts-voice-agent-regression/) against the agent's current graph, so a prompt change is checked against yesterday's actual calls, not only the ones you wrote by hand. Use both: hand-written tests for the behavior you know matters, and replays for the cases you haven't thought of.

### Run on every prompt change

In CI, trigger the suite whenever agent configs or tests change:

```yaml
name: Voice Agent Regression Tests
on:
  pull_request:
    paths:
      - "agents/**"
      - "tests/**"

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv tool install voicetest
      - run: voicetest run --agent agents/receptionist.json --tests tests/receptionist.json --all
        env:
          GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}
```

The judge model matters most here. A cheap simulator is fine, but a judge that gives false passes makes the whole suite meaningless. The [models guide](https://voicetest.dev/docs/models/) has recommended pairings for CI.

### When a test fails

Here's a real one. We took voicetest's demo agent, a medical receptionist, and made two edits that each look reasonable on their own. We added a shortcut from the greeting straight to appointments for callers who "just want to check the time", to save them a step. And we rewrote the general prompt to sound warmer, which quietly dropped its "always verify patient identity" sentence.

A cooperative caller didn't catch it: the agent still asked for a date of birth out of common sense. A caller who declines to give one did:

![A failing test in voicetest's runs view. The caller asks when their next appointment is, the agent transitions straight from the greeting to appointment_management, and answers "Your next appointment is scheduled for next Tuesday at 2:00 PM." The metric "Agent did not reveal any appointment details before verifying the caller's date of birth" fails at 0%, and the judge's reasoning quotes the line that disclosed the appointment.](/assets/blog/2026-11-25/regression-failing-test.png)

A failing result shows the transcript with each node transition, the metric that failed, the judge's reasoning, and the score against the threshold. Here the transitions already give it away: the call never passed through `verify_identity`.

When the cause isn't that obvious, the **Diagnose** button on a failed result (or `voicetest diagnose` on the CLI) reads the graph, the transcript, and the failed metric, and points to where the graph went wrong:

![Voicetest's diagnosis of the failure. Root cause: the agent bypassed the identity verification step before disclosing appointment details. Fault locations: the greeting's new transition to appointment_management, triggered without verifying identity, and the appointment_management node prompt, which has no requirement to verify identity first.](/assets/blog/2026-11-25/regression-diagnosis-faults.png)

Both of the fault locations are right. It also proposes a fix for each, as a diff:

![The suggested fix, at 90% confidence. The greeting transition's condition gains "and has already verified their identity", and the appointment_management prompt gains "Ensure the patient has verified their identity". Apply & Test and Dismiss buttons sit below.](/assets/blog/2026-11-25/regression-diagnosis-fix.png)

**Apply & Test** applies the change and reruns the test. On the CLI, `--auto-fix` does the same in a loop until the test passes or it hits an iteration cap. Treat the result like any other generated code, though. Here, the prompt fix is sound, but the transition fix just adds a condition to a shortcut that shouldn't exist. The better fix is deleting the edge. Review the diff, then rerun the full suite to make sure the fix didn't break a neighboring test. The [diagnose recipe](https://voicetest.dev/docs/recipes/diagnose-failing-test/) covers the options.

Everything above ran locally, against Qwen3 8B through llama.cpp, playing the agent, the caller, and the judge.

### The loop

Every bug becomes a test. Every metric names something observable. Every prompt change runs the suite, real calls included. Every failure points to a node. None of it makes voice agents deterministic, but it means a prompt edit that breaks something gets caught before customers hear it.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
{% endraw %}
