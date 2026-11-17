# GuideManifold (Adaptive Scoring) — ConvoSpec Compliance Assessment

| Field | Value |
|---|---|
| Instrument | GuideManifold demo — adaptive conversational scoring of GAD-7 / PHQ-9 |
| Source | [`adaptive.md`](../../adaptive.md) |
| Evidence base | [`adaptive-interaction-inventory.md`](./adaptive-interaction-inventory.md) (P01–P12) |
| Baseline | `spec.md` **v0.6.0** |
| Date | 2026-11-16 |

---

## Verdict

**Sufficient.** All twelve patterns are **Covered**. The instrument is a free-text conversation loop with a silent per-turn pipeline (score → estimate → select → generate) and dual termination. Every step maps to existing node types without a workaround.

**Covered 12 · Partial 0 · Gap 0.**

---

## 1. Scope and method

**Assessed**: one session — free-text exchange, silent scoring into 16 items, uncertainty-aware estimate, active turn selection over 22 candidates, LLM turn generation, transparency display, turn-budget and confidence-threshold termination — as described in `adaptive.md`.

**Excluded by the source**: training data and simulation, model architecture (CNP), demo mode, scorer backend choice, and UI layout.

**Method**: each inventory pattern mapped to node types (§4), edges and expressions (§5), composition (§6), execution semantics (§7), and validation rules (§8) of `spec.md` v0.6.0. The §11 open issues are treated as unavailable.

---

## 2. Part A — Required interactions and nodes

| Need | Node types / constructs |
|---|---|
| Free-text user exchange | `ask` (`text`) or `llm` (`conversational`) (§4.2, §4.3) |
| Silent per-turn scoring | `classify` (per-item labels) or `analyze` (batch scoring) (§4.3) |
| Estimate with uncertainty | `analyze` (CNP model) writing distributions to state (§4.3) |
| Active turn selection | `set` computing the winning candidate from scored state; or `views` expression (§4.5, §3.8) |
| LLM turn generation | `llm` (`conversational`) with a prompt conditioned on selected turn type (§4.3) |
| Transparency display | `say` rendering current estimate from state (§4.2) |
| Candidate registry | static list in config, or `graph_registry` for subgraph dispatch (§6.1) |
| Probe penalty + back-to-back doubling | `set` arithmetic on `state.last_turn_type` and `state.probe_penalty` (§4.5, §5.5) |
| Non-probe conversational moves | `llm` prompt variants or `say` templates (§4.2, §4.3) |
| Turn-budget termination | `loop.max_iterations` (§4.4) |
| Confidence-threshold termination | `loop.exit_when` on estimate confidence (§4.4) |
| Session state accumulation | `set` with `append` to a `(turn, scores)` list (§4.5, §5.5) |

---

## 3. Part B — Sufficiency assessment

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Free-text user exchange | Covered | `llm` (`conversational`) — one exchange per turn (P8, §4.3). |
| P02 Silent per-turn scoring | Covered | `classify` per item, or `analyze` for the batch. Silent (`await: none`), writes to state (§4.3). |
| P03 Estimate update with uncertainty | Covered | `analyze` (CNP backend) writing distributions to state. Silent (§4.3). |
| P04 Active turn selection | Covered | `set` computes expected-information-gain scores over the 22 candidates, subtracts probe penalty, writes the winner. The CNP's reply prediction is an `analyze` node (§4.3, §4.5). |
| P05 LLM turn generation | Covered | `llm` (`conversational`) with a prompt template conditioned on the selected turn type (§4.3). |
| P06 Transparency display | Covered | `say` with a message template reading from estimate state (§4.2). |
| P07 Candidate turn registry | Covered | Static list in config, or `graph_registry` if each candidate is a subgraph (§6.1, §6.2). |
| P08 Probe penalty + doubling | Covered | `set` arithmetic: `state.last_was_probe ? 2*lambda : lambda` (§4.5, §5.5). |
| P09 Non-probe conversational moves | Covered | Prompt variants in the `llm` node, or `say` templates (§4.2, §4.3). |
| P10 Turn-budget termination | Covered | `loop.max_iterations` (§4.4). |
| P11 Confidence-threshold termination | Covered | `loop.exit_when` on `views.confidence >= threshold` (§4.4, §3.8). |
| P12 Session state accumulation | Covered | `set` with `append(state.turn_log, ...)` inside the loop (§4.5, §5.5, §6.5.3). |

**Covered: 12 · Partial: 0 · Gap: 0.**

---

## 4. Worked encoding — the conversation loop

Compact sketch of the load-bearing pattern: the per-turn pipeline inside a bounded loop.

```yaml
graph:
  id: guidemanifold
  version: 0.1.0
  entry: start
  state_schema:
    turn_log:       { type: list<object>, scope: session, mutable: true }
    estimates:      { type: object,     scope: session, mutable: true }
    confidence:     { type: number,     scope: session, mutable: true }
    selected_turn:  { type: string,     scope: session, mutable: true }
    last_was_probe: { type: boolean,    scope: session, mutable: true }
  views:
    ready:
      type: boolean
      expr: "state.confidence >= 0.9 or len(state.turn_log) >= 10"
  nodes:
    - id: start
      type: start
      config:
        init:
          turn_log: []
          estimates: {}
          confidence: 0
          selected_turn: ""
          last_was_probe: false
    - id: loop
      type: loop
      config:
        body_entry: chat
        exit_when: "views.ready"
        max_iterations: 10
    - id: chat
      type: llm
      config:
        model: chatbot
        mode: conversational
        prompt_template: "{{ state.selected_turn_prompt }}"
        slot: user_reply
    - id: score
      type: classify
      config:
        backend: llm
        model: scorer
        inputs: [state.user_reply]
        labels: [0, 1, 2, 3, "not_mentioned"]
        slot: scores
    - id: estimate
      type: analyze
      config:
        backend: cnp
        inputs: [state.turn_log, state.scores]
        slot: estimates
    - id: select_turn
      type: set
      config:
        assignments:
          - slot: selected_turn
            expression: "find(candidates, item.eig > max(map(candidates, j.eig)) - epsilon).id"
          - slot: last_was_probe
            expression: "state.selected_turn matches 'probe'"
    - id: show_estimate
      type: say
      config:
        message: "Current estimate: {{ state.estimates }} (confidence: {{ state.confidence }})"
    - id: end
      type: end
      config:
        outcome: completed
  edges:
    - { from: start,      to: loop,    type: next }
    - { from: loop,       port: body,  to: chat,      type: next }
    - { from: chat,       to: score,   type: next }
    - { from: score,      to: estimate, type: next }
    - { from: estimate,   to: select_turn, type: next }
    - { from: select_turn, to: show_estimate, type: next }
    - { from: show_estimate, to: loop,   type: next }
    - { from: loop,       port: exit,  to: end,       type: next }
```

The silent pipeline (score → estimate → select → display) runs between turns. Each node is `await: none` so they do not create turn boundaries. The loop body is the `llm` exchange; the rest is silent work before the next exchange.

**The CNP.** The estimate model is an `analyze` node because it is a silent inference step. Its output — a probability distribution over 16 items plus totals — is an `object` written to state. The confidence value is derived from that object and written separately.

**Turn selection.** The 22 candidates and their expected-information-gain scores are a `set` expression. The CNP's reply prediction (used to compute EIG) is an `analyze` node that runs before `set`. The probe penalty and back-to-back doubling are arithmetic on `state.last_was_probe`.

---

## 5. Observations

No gaps. Three notes on the encoding:

### The silent pipeline (P02–P04)
Score → estimate → select is a sequence of `await: none` nodes. This is the same shape as Ask Kai's derived facts: model calls that write to state between turns, not blocking the conversation. `analyze` is designed for exactly this (§4.3).

### Active turn selection (P04)
This is the most instrument-specific piece. The CNP predicts replies and computes expected information gain for each candidate. In ConvoSpec this is an `analyze` node (the prediction) followed by a `set` node (the selection). The candidate set is static (22 entries) and fits in config; if the candidates were dynamic, a `graph_registry` would dispatch.

### Transparency (P06)
Showing estimates after every reply is a `say` with a message template reading from estimate state. It is not a new node — it is one of the simplest `say` uses.

---

## 6. Verification

- **Spec citations** re-checked against `spec.md` v0.6.0: `analyze` (§4.3, silent, writes to state, no turn boundary); `classify` (§4.3, `llm` backend, closed labels); `llm` conversational (§4.3, one exchange per turn); `set` (§4.5, pure, atomic); `loop` with `max_iterations` and `exit_when` (§4.4); `say` (§4.2); `views` (§3.8); `graph_registry` (§6.1, §6.2). **No invented node types or fields.**
- **Source claims** re-checked against `adaptive.md`: 16 GAD-7/PHQ-9 items, 22 candidates (16 probes + 6 moves), probe penalty λ with back-to-back doubling, turn budget, confidence threshold, per-turn transparency, silent scoring pipeline.
- **Consistency**: no pattern is both Covered and a Gap.

---

## 7. Assumptions and limitations

- Evaluated against `spec.md` **v0.6.0 as written**; §11 open issues are unavailable.
- The source is a project README describing a research demo, not a clinical instrument. Terminology ("CNP", "scorer noise", "expected information gain") is model-level; the conversation properties are thin.
- The CNP is an opaque model; ConvoSpec treats it as an `analyze` backend. The spec does not need to know the model's internals, only that it is silent, writes to state, and is auditable (§4.3).
- The YAML sketch is illustrative, not a complete encoding.
