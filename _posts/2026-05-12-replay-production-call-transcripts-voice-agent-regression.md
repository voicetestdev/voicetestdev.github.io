---
title: "Replay Production Call Transcripts Against Your Voice Agent's Current Graph"
description: "Import Retell call transcripts as voicetest runs, replay them against the agent's current configuration, and surface behavioral drift before it ships to production."
date: 2026-05-12
---

The hardest regressions to catch in voice agents are the ones that pass every synthetic test and only break on the actual conversations real users have. You ship a prompt change, the LLM judges still score everything green, and a week later support tickets pile up because the agent now confirms the wrong appointment type or skips the identity check on a path nobody wrote a test for.

Synthetic tests are bounded by the imagination of whoever wrote them. Production traffic is not. Voicetest 0.41 closes that gap with two new operations: import real call transcripts as runs, then replay them against the agent's current graph to see whether the live agent still produces the same outcomes.

This post covers the import format, the replay mechanics, and a workflow for using your own production calls as a regression suite.

### Two operations, one data model

| Operation        | What it does                                                                                          | CLI                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Import calls** | Parse a platform transcript dump and persist as a Run with `status="imported"` Results                | `voicetest import-call --agent <id> --transcript file.json` |
| **Replay**       | Drive a fresh conversation against the agent's current graph using a source Run's user turns as a script | `voicetest replay <run-id>`                                 |

Both produce ordinary `Run` records. Imported runs and replay runs render in the same UI as simulated runs, share the same storage, and can be compared turn-for-turn.

### Importing Retell calls

Voicetest accepts the Retell call object in the shape returned by `GET /v2/get-call/{call_id}` and in the post-call webhook envelope. Single object, array of objects, or an array of webhook payloads — all four shapes parse the same way:

```json
{
  "call_id": "call_abc123",
  "transcript_object": [
    {"role": "agent", "content": "Hi, how can I help?"},
    {"role": "user", "content": "I need to cancel my order."},
    {"role": "agent", "content": "Sure — can I get your order number?"},
    {"role": "user", "content": "It's 4471."}
  ],
  "duration_ms": 60000,
  "start_timestamp": 1700000000000,
  "end_timestamp": 1700000060000,
  "disconnection_reason": "user_hangup"
}
```

The adapter maps Retell's `role: "agent"` to voicetest's `role: "assistant"` and ignores word-level timing. One CLI call ingests an entire dump:

```bash
voicetest import-call --agent receptionist --transcript yesterdays-calls.json
# Imported 247 conversation(s) into run abc123de
```

Each call becomes a `Result` inside a single `Run`. Results have `status="imported"`, no `test_case_id`, and no `call_id` linkage to a synthetic test — they are passive captures of what actually happened.

Other platforms (VAPI, LiveKit, Telnyx, Bland) are not supported in v1, but the `--format` flag is parameterized so adapters can be added without breaking changes.

### Replay against the current graph

Replay reuses the existing conversation engine. The only difference is the user side: instead of an LLM-driven `UserSimulator`, replay runs use `ScriptedUserSimulator`, which yields the recorded user turns from the source transcript in order:

```python
class ScriptedUserSimulator:
    def __init__(self, source_transcript: list[Message]):
        self._user_turns = [m.content for m in source_transcript if m.role == "user"]
        self._index = 0

    async def generate(self, transcript, on_token=None, on_error=None):
        if self._index >= len(self._user_turns):
            return None
        message = self._user_turns[self._index]
        self._index += 1
        return SimulatorResponse(message=message)
```

The runner drives the conversation against the live agent the same way it would for a normal test run. Source agent turns are discarded — only the user side is scripted. The live agent's responses replace the recorded ones, so what you get back is "how my agent responds today to the user-side of yesterday's call."

```bash
voicetest replay abc123de
```

The new run lands next to the source in the runs UI. Open the source run on the left, the replay run on the right, scroll the transcripts side by side.

### What replay catches that synthetic tests miss

Three kinds of regression slip past synthetic suites and surface clearly under replay:

1. **Phrasings the suite never simulated.** Real users say "yeah I dunno can you just send it whenever" — `UserSimulator` produces grammatical, goal-directed turns. If a prompt change makes the agent fragile to filler or hedging, replay catches it; the simulator probably won't.
2. **Long-tail intents.** A receptionist agent's test suite has 20 cases. Production has 200 distinct user goals. Replay against a week of calls exercises whatever distribution actually exists.
3. **Drift between platforms.** If you maintain the agent on two platforms (e.g. Retell + VAPI for failover), replay the same Retell transcripts against both. Divergence shows up immediately.

### Best-effort divergence handling

Replay does not try to recover gracefully when the live agent diverges from the recorded conversation. If today's agent asks a different question than yesterday's, the next recorded user turn may not fit:

> **Recorded:** Agent: "What's your account number?" → User: "4471."
> **Live:** Agent: "Can you spell your last name?" → User: "4471." *(scripted turn doesn't match the new question)*

The replay continues anyway. The conversation as a whole still produces a transcript you can judge — the regression signal is "this conversation is now incoherent" and that is exactly the thing you want to know about.

A future version may add LLM-based divergence handling (re-prompting the simulator when the script doesn't fit). v1 keeps the contract simple: deterministic, reproducible, fast.

### A regression suite from production traffic in 60 seconds

Putting it together — assuming you already have a Retell agent and you can pull a day's worth of call dumps:

```bash
# 1. Import yesterday's production calls as a Run
voicetest import-call --agent receptionist --transcript prod-calls-2026-05-06.json
# Imported 312 conversation(s) into run prod-2026-05-06

# 2. Make whatever prompt change you want to test
$EDITOR agents/receptionist.json

# 3. Push the new graph to your agent's storage
voicetest import --file agents/receptionist.json --id receptionist

# 4. Replay yesterday's calls against the new graph
voicetest replay prod-2026-05-06
# Replayed 312 conversation(s) into run replay-abc123

# 5. Open both runs in the UI and scan for divergence
voicetest serve
```

312 real conversations, replayed against the new graph, with no test cases written. The synthetic suite still has its place — it is the regression net for the cases you've already thought about. Replay is the net for the cases you haven't.

### Limitations to know about

- **Retell only in v1.** Adapters for VAPI, LiveKit, Bland, Telnyx are tracked but not yet shipped.
- **No PII redaction at import time.** If your transcripts contain sensitive data, redact before ingest. Voicetest stores Results in the same DuckDB file as the rest of your runs.
- **No diff view yet.** Source and replay are separate runs in the UI; comparing them is manual scrolling for now. A built-in turn-aligned diff is on the roadmap.
- **No automatic test-case generation from imports.** Imported transcripts are runs, not test cases. Clustering them into a deduplicated test suite is on the proposals list.

### REST and UI surfaces

Both operations are also available via REST and the web UI:

```bash
# Import
curl -X POST http://localhost:8000/api/agents/receptionist/import-call \
  -F "transcript=@prod-calls.json" \
  -F "format=retell"

# Replay
curl -X POST http://localhost:8000/api/runs/prod-2026-05-06/replay
```

The agent page has an "Import Calls…" button; the run detail page has a "Replay" button. Same data model, same storage — pick whichever surface fits.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
