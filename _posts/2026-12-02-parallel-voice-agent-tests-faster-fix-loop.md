---
title: "Cut Your Voice Agent's Time to Fix: Run Tests in Parallel"
description: "When a voice agent breaks, most of the time to recovery is spent waiting for the test suite to confirm a fix. Voicetest 0.56 runs tests in parallel, so each fix-and-verify loop takes about as long as the slowest test instead of all of them added together."
date: 2026-12-02
image: /assets/blog/2026-12-02/cover.png
---
{% raw %}
When a voice agent breaks in production, the fix is often a one-line prompt edit. Confirming it is what takes the time. You edit the prompt, rerun the suite to make sure the fix works and nothing else broke, read the results, and usually go around again. Your time to recovery is mostly the number of loops multiplied by how long each suite run takes.

Simulated conversations are slow tests. Each one is a multi-turn exchange in which every turn waits for the agent model and the simulated caller, followed by a judge pass for each metric. A single test can take a minute or more. Until now, voicetest ran a suite's tests one after another, so a run took as long as all its tests added together. Thirty one-minute tests meant half an hour per loop.

Voicetest 0.56 runs tests in parallel. This post covers why that works, what it does to suite time, how to choose a concurrency level, and what limits it.

### Where the time goes

A simulated test spends almost all of its time waiting for a model provider to respond. Your machine does very little: it builds prompts, parses responses, and writes results. Running tests one at a time leaves both your machine and your provider allowance mostly unused.

Tests in a run don't depend on each other. Each one starts a fresh conversation against the same agent graph. So nothing stops them running side by side. With 8 running at once, a run takes about as long as the total divided by 8. With as many slots as tests, it takes about as long as the slowest test.

### Starting a parallel run

Set `concurrency` in the run options when you start a run through the API. It accepts 1 to 16 and defaults to 1:

```bash
curl -X POST http://localhost:8000/api/agents/<agent-id>/runs \
  -H 'Content-Type: application/json' \
  -d '{"options": {"concurrency": 8}}'
```

The response includes the `concurrency` actually used, which is never more than the number of tests in the run. Replays of [production calls](https://voicetest.dev/blog/replay-production-call-transcripts-voice-agent-regression/) take the same option on `POST /api/runs/{id}/replay`.

Results show up in the web UI as each test finishes, in their place in the test list, and the Runs tab now updates live. If a coding agent or a CI job starts runs through the API, you can watch them fill in without reloading.

Concurrency is set per run and only through the API for now. It isn't a global setting, because the right value depends on which provider and model the run uses.

### What it does to suite time

We timed the 16-test demo suite (two copies of voicetest's 8 bundled demo tests) against Claude Opus through the Claude Code CLI, with the cache off, at each concurrency level. Speedup is measured against running the tests one at a time:

| Concurrency | Speedup | Suite time saved |
|---|---|---|
| 1 | baseline | — |
| 2 | 1.7× | 41% |
| 4 | 3.8× | 74% |
| 8 | 5.1× | 80% |
| 16 | 7.8× | 87% |

Every level finished all 16 tests without errors. Up to 4, speedup tracks concurrency almost one for one: 4 tests at a time made the suite nearly four times faster. After that, each doubling bought less. 8 reached 5.1×, and 16, the most this suite can use, reached 7.8×, cutting suite time by 87% but landing at about half the ideal 16×. Each level was run once, so expect some noise between neighbouring levels. The shape is the point: the provider started falling behind before voicetest ran out of tests to start.

Speedup stops growing for two reasons. Once concurrency reaches the number of tests, the run can't finish faster than its slowest test. And as concurrency rises, the provider starts throttling, and retried calls stretch each test out.

You can reproduce these numbers against your own provider:

```bash
uv run python scripts/bench_concurrency.py --model openai/gpt-4o-mini --repeat 2 --out results.csv
```

It runs the full suite once at each level with the cache off, so it costs one uncached suite run per level.

### Choosing a concurrency level

The limit is your provider, not your machine. Providers rate-limit requests and tokens per minute, and those limits usually apply to your whole organization or project, not to each API key. Adding more keys from the same account doesn't raise them.

A rough budget: a test uses some number of tokens over some number of minutes. Running N tests at once needs about N times that token rate. Compare that with your tokens-per-minute limit, and leave headroom for anything else sharing the account, such as production traffic.

In practice:

1. Start at 4.
2. Double it while wall time keeps dropping and you aren't seeing rate-limit retries.
3. Stop when wall time flattens or retries appear. Throttled calls back off and retry, so going past the limit doesn't fail the run. It just stops getting faster.

If the provider reports your quota as exhausted, voicetest doesn't wait hours for it to reset. Tests still queued are cancelled, and the tests already running record the error themselves, so the run ends with a clear reason rather than a stuck queue.

### When your provider is the bottleneck

If speedup flattens well before concurrency reaches the number of tests, the limit is your LLM provider, not voicetest. Past that point, more concurrency only adds throttling and retries. Extensions to manage this are coming: they'll distribute a parallel run across different providers, so one provider's rate limits no longer cap the whole run.

### What doesn't change

Results are the same as a sequential run. Each test is its own conversation, so running them side by side doesn't change what they see or how they're judged.

Cached reruns were already fast. If nothing changed, voicetest replays cached completions and a suite finishes in seconds. Parallel runs matter for the uncached case, which is exactly the fix loop: editing a prompt changes the requests for every turn that uses it, so those calls go to the provider again.

The way voicetest calls models also changed in 0.56. Model calls used to block a thread each while waiting on the provider, so the machine's thread pool capped how many could run at once. They're now asynchronous, so a run can wait on as many calls as its concurrency allows. This also means completions cached by earlier versions are fetched again once after upgrading.

### Local models

Parallel runs help most with hosted providers, which serve many requests at once. A [local model](https://voicetest.dev/blog/offline-voice-agent-testing-local-models/) runs on your own hardware, so parallel tests mostly queue for the same GPU. Raising concurrency there gains little and can slow each test down. For local runs, keep concurrency low and expect suite time to depend on your hardware.

### The loop, shorter

Most of the time to fix a broken agent is spent verifying fixes, and each verification is a full suite run. Running the suite in parallel turns a coffee break into something closer to a compile, which makes it practical to rerun everything after every change instead of saving it for the end. Pick a concurrency level your provider can sustain, and let the suite run as wide as that allows.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
{% endraw %}
