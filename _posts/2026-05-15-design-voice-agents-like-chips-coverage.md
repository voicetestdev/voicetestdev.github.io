---
title: "Designing Voice Agents Like Chips: Coverage Closure for Agent FSMs"
description: "Voice agent graphs are finite state machines. The chip industry has forty years of methodology for verifying FSMs too large to enumerate — structural coverage, functional covergroups, coverage closure. This post maps that methodology onto voice agent testing and shows what voicetest already supports, what's missing, and why the framing matters."
date: 2026-05-15
---
{% raw %}
A voice agent and a SoC differ entirely at the substrate level — one is a graph of prompts driving an LLM, the other is millions of gates etched into silicon. One level of abstraction up, the structures align: both are finite state machines whose interesting behavior lives in transitions and interactions between states, both have an authoring language that is compiled by something downstream (a synthesis toolchain in one case, the inference loop in the other), and both fail in long-tail conditions outside the test set.

The chip industry has spent forty years developing methodology for verifying designs whose state spaces cannot be enumerated. The agent industry has spent roughly three. The fraction of that methodology that transfers is the subject of this post.

### Verification in EDA

A chip designer writes RTL — a register-transfer-level description of registers, combinational logic, and state machines. A synthesis toolchain compiles RTL down to a gate-level netlist, then physical design places and routes the netlist on the die. Functional bugs that escape into silicon cost a respin: months of time and millions of dollars. Verification investment is sized accordingly. The deliverable is a regression suite of thousands of testbenches that runs continuously and a quantitative measure of what those testbenches have exercised: *coverage*.

Coverage decomposes into two categories. **Code coverage** is structural: line coverage (every RTL line executed), branch coverage (every branch taken), toggle coverage (every signal driven 0 and 1), and FSM coverage (every state and transition visited). **Functional coverage** is intent-based: explicit declarations called *covergroups* enumerate the scenarios the design must be verified against — concurrent writes to the same address, FIFO full while interrupt pending, specific variable combinations across modules. Covergroups are first-class artifacts checked into the design repository alongside the RTL, and the regression report shows which bins have been hit.

The last 10% of coverage — *coverage closure* — is the dominant cost. Targeted tests for specific un-hit bins take weeks to author, depend on accumulated institutional knowledge of how the design class has historically broken, and frequently require modifying constraints or exclusions in adjacent tests. Closure is what design houses tap their cross-generation history for.

This framework — structural coverage, functional covergroups, closure as the long-pole task, regression suites that compound across product generations — is the foundation of modern chip verification. None of it is in the standard agent-testing playbook.

### Mapping EDA verification onto agent graphs

Voicetest's [graph IR](/blog/2026-05-08-node-types-global-interrupts-voicetest-graph-ir) exposes the FSM structure of a voice agent directly. Five node types (`conversation`, `logic`, `extract`, `end`, `transfer`), typed transitions (LLM-prompted vs equation-evaluated), global interrupts with go-back semantics. Structurally this is an FSM specification — not an analogy, not a metaphor.

| Chip design                                          | Voice agent                                                |
| ---------------------------------------------------- | ---------------------------------------------------------- |
| RTL source                                           | AgentGraph (nodes, transitions, prompts, snippets, tools)  |
| Synthesizer + place-and-route                        | LLM at inference time                                      |
| Testbench (stimulus, monitor, scoreboard)            | voicetest runner + UserSimulator + LLM judges              |
| Simulation cycle                                     | Conversation turn                                          |
| Constrained random stimulus                          | UserSimulator parameterized by persona / objectives        |
| Regression suite                                     | Test cases + imported production calls                     |
| Equivalence check across DUT implementations         | Multi-platform replay (Retell vs VAPI vs LiveKit)          |
| Coverage report                                      | *(missing)*                                                |
| Covergroups                                          | *(missing)*                                                |
| Closure workflow                                     | *(missing)*                                                |

The first six rows are operations voicetest already supports. The last three rows describe a methodology gap. The chip-design vocabulary for closing that gap is mature; the agent equivalent is unbuilt.

### Structural coverage for agent FSMs

Each structural-coverage metric in EDA has a direct agent analogue. All of them are computable today from data the runner already records (the conversation trace plus the node trajectory through the graph):

- **Node coverage** — fraction of `AgentNode`s visited across the regression suite. Direct analogue of FSM state coverage.
- **Transition coverage** — fraction of transitions taken, with each guard condition exercised. Direct analogue of FSM transition coverage. Typically the highest-value structural metric, because agent bugs concentrate on edges rather than within states.
- **Variable extraction coverage** — every named extract variable produced at least once, with multiple distinct values per variable where the schema admits them.
- **Tool invocation coverage** — every tool called, with representative argument shapes.
- **Snippet coverage** — every prompt snippet rendered in at least one traversal. [Snippets](/blog/2026-02-24-prompt-snippets-dry-analysis-voice-agent-graphs) shared across nodes create coverage holes: a regression introduced via one usage may not be caught by a test that only renders the snippet through a different node.
- **Global-interrupt coverage** — every global interrupt triggered, with go-back resumption verified from at least one nontrivial source node.

Structural coverage does not establish correctness. Exercising a transition is not the same as verifying the response generated on the other side. It establishes the precondition for any other correctness claim: an un-visited transition cannot have been verified by any judge, simulator, or assertion in the suite.

### Functional coverage and covergroups

Structural coverage is necessary but insufficient. The substantive question is whether the suite exercises the *scenarios* that actually break agents under production conditions. In SystemVerilog the construct is `covergroup`: a declarative specification of bins to hit and crosses to capture interactions between variables. The agent analogue:

```yaml
covergroups:
  identity_robustness:
    description: "Caller hesitates or refuses to provide identifying info."
    bins:
      - refuses_then_provides_on_retry
      - provides_partial_then_corrects
      - provides_in_unexpected_format
      - asks_why_we_need_it_first

  global_interrupt_during_extract:
    description: "Caller invokes a global interrupt mid-extract, then returns."
    crosses:
      - source_node: [identity_extract, payment_extract, account_extract]
      - interrupt: [speak_to_human, start_over, emergency]
      - resumption: [completes_after, abandons_after]
```

The syntax is incidental. What matters is that scenario intent becomes a *declared, version-controlled artifact* — separate from the test bodies that hit it, queryable for zero-hit bins, gateable in CI on coverage regressions in addition to test-failure regressions. Covergroups carried across releases compound into a documented record of what the agent has been verified against — the same role they play across chip generations.

### Coverage closure

EDA's working assumption is that the first 90% of coverage is cheap and the last 10% consumes the remaining 90% of verification effort. The agent equivalent: synthetic happy-path suites are cheap; the long tail of production-distribution user phrasings is where reliability is established or lost. Replay-from-production ([voicetest 0.41](/blog/2026-05-12-replay-production-call-transcripts-voice-agent-regression)) supplies one half of the answer — production traffic as constrained-random stimulus over the input space. The other half is the targeted closure work: read the coverage report, identify un-hit bins, author a test that exercises each one, regenerate the report, iterate.

### Where the analogy breaks down

Chip verification includes formal methods: mathematical proof that property `P` holds across all input sequences. LLM agents admit no such proof. Inference is probabilistic, prompts are natural language, and statements of the form "state `S` produces response `R`" are not provable from the model. The formal-methods tier of the EDA verification stack does not transfer.

The simulation tier does. Most chip verification is not formal — it is constrained-random simulation against coverage targets, with formal applied selectively to specific properties on specific blocks. That tier assumes the design under test cannot be exhaustively enumerated and that probabilistic exploration with coverage closure is the operative discipline. Voice agents fit that regime precisely. The probabilistic nature of LLM inference arguably *increases* the importance of coverage metrics, because correctness cannot be proved at all — only sampled.

### Worked example: a help-desk agent

Consider a help-desk agent: greeting → identity-extract → logic-split on `account_type` → either billing-conversation or technical-conversation → end. One global interrupt, `speak_to_human`, reachable from any conversation node, with a go-back edge to the originating state.

An initial test suite covering a billing happy path, a technical happy path, and one `speak_to_human` invocation from greeting produces the following coverage profile:

<svg viewBox="0 0 720 440" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:24px auto;max-width:100%;background:#141414;border:1px solid #262626;border-radius:8px;" role="img" aria-label="Help-desk agent graph with covered transitions in solid cyan and uncovered transitions in dashed gray.">
  <defs>
    <marker id="arrow-cov" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#00B4D8"/></marker>
    <marker id="arrow-miss" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#555"/></marker>
  </defs>

  <text x="360" y="26" text-anchor="middle" fill="#fafafa" font-family="-apple-system,sans-serif" font-size="13" font-weight="600">Help-desk agent coverage after 3 happy-path tests</text>

  <!-- Main flow, all covered -->
  <g font-family="-apple-system,sans-serif" font-size="11" fill="#fafafa">
    <rect x="30"  y="80" width="100" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="80"  y="105" text-anchor="middle">greeting</text>

    <rect x="170" y="80" width="120" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="230" y="105" text-anchor="middle">identity_extract</text>

    <rect x="330" y="80" width="120" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="390" y="105" text-anchor="middle">account_split</text>

    <rect x="490" y="80" width="90" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="535" y="105" text-anchor="middle">billing</text>

    <rect x="620" y="80" width="80" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="660" y="105" text-anchor="middle">end_billing</text>
  </g>

  <!-- Covered main edges (cyan solid) -->
  <g stroke="#00B4D8" stroke-width="2" fill="none">
    <line x1="130" y1="100" x2="168" y2="100" marker-end="url(#arrow-cov)"/>
    <line x1="290" y1="100" x2="328" y2="100" marker-end="url(#arrow-cov)"/>
    <line x1="450" y1="100" x2="488" y2="100" marker-end="url(#arrow-cov)"/>
    <line x1="580" y1="100" x2="618" y2="100" marker-end="url(#arrow-cov)"/>
  </g>

  <!-- Technical conversation: covered. transfer_tech end node: uncovered. -->
  <rect x="490" y="180" width="90" height="40" rx="6" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
  <text x="535" y="205" text-anchor="middle" font-family="-apple-system,sans-serif" font-size="11" fill="#fafafa">technical</text>

  <rect x="620" y="180" width="80" height="40" rx="6" fill="#0a0a0a" stroke="#555" stroke-width="2" stroke-dasharray="4 4"/>
  <text x="660" y="205" text-anchor="middle" font-family="-apple-system,sans-serif" font-size="11" fill="#888">transfer_tech</text>

  <!-- account_split → technical: covered. technical → transfer_tech: uncovered. -->
  <path d="M 420 120 Q 450 150 490 195" stroke="#00B4D8" stroke-width="2" fill="none" marker-end="url(#arrow-cov)"/>
  <line x1="580" y1="200" x2="618" y2="200" stroke="#555" stroke-width="2" fill="none" stroke-dasharray="4 4" marker-end="url(#arrow-miss)"/>

  <!-- identity_extract self-loop (retry), uncovered -->
  <g stroke="#555" stroke-width="2" fill="none" stroke-dasharray="4 4">
    <path d="M 200 80 Q 200 50 230 50 Q 260 50 260 80" marker-end="url(#arrow-miss)"/>
  </g>
  <text x="230" y="42" text-anchor="middle" fill="#888" font-family="-apple-system,sans-serif" font-size="10" font-style="italic">retry</text>

  <!-- Global interrupt node -->
  <g font-family="-apple-system,sans-serif" font-size="11" fill="#fafafa">
    <rect x="280" y="330" width="160" height="44" rx="22" fill="#0a0a0a" stroke="#00B4D8" stroke-width="2"/>
    <text x="360" y="350" text-anchor="middle">speak_to_human</text>
    <text x="360" y="365" text-anchor="middle" fill="#888" font-size="10" font-style="italic">global interrupt</text>
  </g>

  <!-- Spokes into global interrupt -->
  <!-- From greeting: covered -->
  <path d="M 80 120 Q 80 330 280 352" stroke="#00B4D8" stroke-width="2" fill="none" marker-end="url(#arrow-cov)"/>
  <!-- From identity_extract: uncovered -->
  <path d="M 230 120 Q 230 270 290 330" stroke="#555" stroke-width="2" fill="none" stroke-dasharray="4 4" marker-end="url(#arrow-miss)"/>
  <!-- From billing: uncovered. Routed around the right of `technical` to avoid passing through it. -->
  <path d="M 580 100 Q 690 240 440 345" stroke="#555" stroke-width="2" fill="none" stroke-dasharray="4 4" marker-end="url(#arrow-miss)"/>
  <!-- From technical: uncovered -->
  <path d="M 490 210 Q 460 280 435 330" stroke="#555" stroke-width="2" fill="none" stroke-dasharray="4 4" marker-end="url(#arrow-miss)"/>

  <!-- Legend -->
  <g font-family="-apple-system,sans-serif" font-size="11" fill="#888">
    <line x1="30" y1="410" x2="60" y2="410" stroke="#00B4D8" stroke-width="2"/>
    <text x="68" y="414">exercised</text>
    <line x1="160" y1="410" x2="190" y2="410" stroke="#555" stroke-width="2" stroke-dasharray="4 4"/>
    <text x="198" y="414">uncovered</text>
    <text x="690" y="414" text-anchor="end">3 tests · 7/8 nodes · 6/12 transitions · 1/4 interrupts</text>
  </g>
</svg>

Structural coverage:

- Node coverage: 7 / 8 — the technical branch's transfer node never visited
- Transition coverage: 6 / 12 — every refusal/retry edge untouched
- Variable extraction coverage: `account_type` extracted in 2 / 4 documented values
- Global-interrupt coverage: 1 / 4 candidate origin nodes; go-back resumption never verified

Functional coverage against the covergroups declared above plus one ambiguity case:

- `identity_robustness`: 0 / 4 bins hit
- `global_interrupt_during_extract`: 0 / 9 crosses hit
- `ambiguous_account_type_in_extract`: 0 hits

That report is the artifact a coverage-driven workflow gates on before shipping a prompt change. Six targeted test cases close most of the structural gaps; the remaining bins drive closure work that compounds across releases.

### Voicetest's coverage roadmap

Voicetest today records the data required to compute structural coverage — node trajectories and transition fires are already part of the run-result schema — but does not surface coverage as a first-class artifact. Functional covergroups are not declarable. There is no `voicetest coverage` command.

The roadmap is to add structural coverage reporting first (accounting against data already collected), covergroup declaration and reporting second (schema and workflow design required), and closure tooling third (delta reports, gap explanations, CI gates on coverage regressions).

The framing matters more than the features. If agent teams adopt the question *"what is our transition coverage"* with the seriousness EDA teams apply to *"what is our FSM coverage"*, the tooling will follow. The bet EDA made in the 1990s — that quantitative coverage as a first-class artifact would dominate any specific new test type for reliability impact — paid off across forty years of progressively harder designs. The same bet is available to agent teams now, on a problem with the same structural shape.

Voicetest is open source under Apache 2.0. GitHub: [github.com/voicetestdev/voicetest](https://github.com/voicetestdev/voicetest)
{% endraw %}
