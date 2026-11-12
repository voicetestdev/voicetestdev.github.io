---
title: "Your Agent Said It Right. Did the Caller Hear It? Live Call Testing in Voicetest"
description: "Text simulations test what a voice agent means to say. Live calls test what the caller actually hears. Voicetest now runs test cases as real audio calls over LiveKit, records both texts for every turn, and judges the heard version."
date: 2026-11-11
image: /assets/blog/2026-11-11/cover.png
---
A voice agent can pass every text simulation and still fail on a real call. The LLM produces the right sentence, and then the speech pipeline mangles it: a phone number read as a quantity, a drug name the TTS voice can't pronounce, an appointment time that STT on the other end turns into a different time. Text simulations never see any of this, because they stop at what the model *meant* to say.

Voicetest now runs test cases as live audio calls. The same simulated caller that drives text runs joins a real WebRTC call, speaks through TTS, and listens through STT. Nobody has to be on the line. Every turn ends up with two texts: what was intended, and what was heard.

This post covers how live calls work, what they catch, and how to run one.

### Intended vs. heard

In a cascade voice pipeline (STT → LLM → TTS), the LLM's output is not what reaches the caller. It is an input to TTS, and the caller hears whatever TTS produced. A defect at that boundary looks like this:

| | Text |
|---|---|
| **Intended** (`content`) | If you need to reschedule, call us at 415-555-0123. |
| **Heard** (`heard`) | If you need to reschedule, call us at four hundred fifteen, five hundred fifty-five, one hundred twenty-three. |

Both lines say the same thing to an LLM judge reading text. Only one of them is usable by a person writing a number down.

Voicetest records this by adding an **observer** to each call. The observer subscribes to the audio each side publishes and transcribes it, so every turn in the transcript carries:

- **`content`**: the intended text, which is the agent's LLM output or the simulated caller's line.
- **`heard`**: what the observer transcribed from the audio the other side actually received.

![How a live call is recorded. The simulated caller and your agent talk over real audio in a LiveKit room. An observer listens in and transcribes what each side actually received. Each turn of the result stores the intended text and the heard text, and the judges score test metrics on the intended text and audio metrics on the heard text.](/assets/blog/2026-11-11/live-call-observer.svg)

The runs view marks up each turn with a word-level diff of the two, so a mangled number or a dropped "not" stands out without re-reading the whole call.

### Judged on what was heard

A finished live call is saved as a normal result, marked `source_kind="live"`, next to simulated and imported results in the same run history. It is judged three ways:

1. The test case's metrics, against the intended transcript.
2. The same metrics again, against the **heard** text. These are stored as audio metrics.
3. The agent's global metrics.

The second pass is the point. A metric like *"Agent gave the caller the callback number"* passes on the intended text above, and may well pass on the heard text too, since a number was technically given. Write the metric the way the caller experiences it, *"Agent read the callback number digit by digit,"* and the two passes disagree:

![An example live call in voicetest's runs view. The agent's last turn shows the intended phone number struck through in red and the heard version, "four hundred fifteen, five hundred fifty-five, one hundred twenty-three", highlighted in green. Under Metrics, "Agent read the callback number digit by digit" passes against the intended text. Under Audio Evaluation, the same metric fails at 0% because the caller heard a quantity rather than digits.](/assets/blog/2026-11-11/live-call-heard-diff.png)

The text metric passes, because the LLM wrote the digits. The audio metric fails, because the caller never heard them. That's the TTS regression a text-only suite lets through.

### How this differs from audio evaluation

Voicetest already had [audio evaluation](https://voicetest.dev/docs/features/#audio-evaluation). It takes a finished *text* simulation and round-trips the agent's lines through TTS and STT afterwards. That's cheap, and it catches the same class of pronunciation defect.

Live calls run the whole conversation over audio instead. The agent responds to what it *heard* the caller say, not to clean text, so STT errors on the caller's side change the course of the call the way they would in production. If the agent mishears "Tuesday" as "Thursday" in turn 3, the rest of the conversation follows from that mistake, and the transcript shows where it happened.

Use audio evaluation as a fast check across a whole suite. Use live calls for the test cases where hearing matters: numbers, names, dates, confirmations, and anything a caller has to act on.

### Running one

Live calls need the voice services: a LiveKit server, a faster-whisper STT server, and a Kokoro TTS server. `voicetest up` starts all of them in Docker along with the backend:

```bash
uv tool install voicetest
voicetest up
```

Then run a test case as a call. In the Web UI, each test in the Tests tab has a **Call** button. It starts the call and switches to the Runs tab, where the transcript streams in turn by turn and the result is saved when the call ends.

From the CLI, point at an agent in the database and a test by name:

```bash
voicetest call \
  --agent-id <agent-id> \
  --tests tests.json \
  --test "Refill with callback number"
```

Over REST, it's `POST /api/agents/{id}/calls/start` with a `test_id`. Leave out `test_id`, or use the **Talk** button on the agent page, and the call waits for a human caller to join from the browser instead. Those calls are saved as runs too, judged against the agent's global metrics, which is useful for checking a prompt change by ear before writing a test for it.

`voicetest down` stops the services.

### No API keys required

The whole stack can run on your machine. The voice services are already local, and the agent, the simulated caller, and the judge can use a local model through llama.cpp's `llama-server`:

```bash
brew install llama.cpp
llama-server -hf Qwen/Qwen3-8B-GGUF:Q4_K_M --port 8080 --reasoning-budget 0
```

```toml
# .voicetest/settings.toml
[models]
agent = "llamafile/qwen3-8b"
simulator = "llamafile/qwen3-8b"
judge = "llamafile/qwen3-8b"
```

This is also how voicetest tests itself. Its CI starts the stack in Docker and runs real calls end to end against a local model, with no external keys. Nothing in the call path needs a cloud account.

### What's next

Observation works on the published audio, regardless of how each side produced it. Today that's the cascade pipeline. A native speech-to-speech agent, with no separate text step, would be observed the same way.

Timing isn't measured yet. Latency to first audio, dead air, and barge-in handling are the next metrics, and they need the same observer that's already on the call. If you run voice agents in production and have a timing failure you want caught, open an issue: [github.com/voicetestdev/voicetest/issues](https://github.com/voicetestdev/voicetest/issues).

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
