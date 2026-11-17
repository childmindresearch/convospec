# MoBI×MSM Structured Chat Trial — ConvoSpec Compliance Assessment

| Field | Value |
|---|---|
| Instrument | MoBI×MSM in-lab structured chat trial |
| Source | [`MoBIxLLM_task_summary_for_GK_20261116.md`](../../MoBIxLLM_task_summary_for_GK_20261116.md) (121 lines) |
| Evidence base | [`mobi-interaction-inventory.md`](./mobi-interaction-inventory.md) (P01–P24) |
| Baseline | `spec.md` **v0.4.0** |
| Date | 2026-11-16 |

---

## Verdict

**Sufficient with reservations.** ConvoSpec v0.4.0 encodes the trial's control flow — 14 ordered steps, a two-reply chat, pre/post repeated measures, repetition with isolation, and a mid-trial stop rule — without new node types. Nineteen of twenty-four patterns are **Covered**.

The five partials cluster in one place: **the model interface**. Prior evaluations (KSADS, DASH) treated the LLM as an incidental component of a structured interview; here the LLM *is* the manipulation. Stance is defined as a system-prompt change (L29–L35), both bot replies must see the conversation so far (L86), and one stop rule inspects generated text (L82). v0.4.0 models a model call as a single opaque `prompt_template` → `slot` exchange, which is one abstraction level too coarse for this instrument.

**Covered 19 · Partial 5 · Gap 0.** No partial prevents delivery; each has a workaround that is brittle, verbose, or distorts the interaction. Details in [§5](#5-gaps-and-observations).

*Re-scored against `spec.md` v0.5.0 after integration: all five partials become Covered — **Covered 24 · Partial 0 · Gap 0**, verdict **Sufficient** ([§8](#8-post-integration-re-evaluation-v050)).*

---

## 1. Scope and method

**Assessed**: one structured chat trial and the factors that vary across trials (L5–L9).

**Excluded by the source**: the rest of the session — calibration, reference tasks, self-directed chat, questionnaires — and the wording of stories and rating items (L9). Physiology, keystroke, and screen recording (L7) are instrumentation, not conversation.

**Method**: each pattern in the inventory was mapped to node types (§4), edges and expressions (§5), composition (§6), execution semantics (§7), and validation rules (§8) of `spec.md` v0.4.0. The §11 open issues are treated as *unavailable*.

**Verdict scale**: **Covered** (encodable directly) · **Partial** (encodable with a workaround that is brittle, verbose, or non-declarative) · **Gap** (not encodable).

---

## 2. Part A — Required interactions and nodes

| Need | Node types / constructs |
|---|---|
| Fixed 14-step trial | Sequential edges; `ask`, `llm`, `say` (§4.2–§4.3) |
| Three manipulated factors | `state` slots set per trial; `context` for graph-level parameters (§3.2, §3.6) |
| Repeated trials | `foreach` over an assignment list + `call` with `state_isolation: isolated` (§4.4, §6.2) |
| Story + two options, held for the trial | State slots; `ask.options_ref` for the forced choice (§4.2, VR-8) |
| Two bot replies conditioned on stance | `llm` (`conversational`) with `prompt_template` interpolation (§4.3) |
| Pre/post repeated items | Six distinct `ask` nodes; pairing via `views` (§3.8) |
| Crisis stop rule | `classify` + `branch` + `end` (§4.3, §4.4) |
| Trial records | `set` + `append` accumulator into `list<object>`; `views` for derived values (§4.5, §6.5.3) |
| Session close | `end`; `session.export` for analysis (§4.1, §3.7) |

---

## 3. Part B — Sufficiency assessment

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Session identity and design parameters | Covered | §3.1 session fields (`session_id`, `seed`, `context`); design parameters as `context`. |
| P02 Balanced randomized condition assignment | **Partial** | No permutation or without-replacement primitive, and expressions cannot read `session.seed` (§5.3, §5.5). `sample` draws once per node over edges (§4.4), so it cannot produce a balanced block. Workaround: compute the assignment outside and supply it as `context` (G1). |
| P03 Story sampling without replacement | **Partial** | Same root cause as P02. Per-type pools and per-participant subsets (L50–L51) require draws without replacement (G1). |
| P04 Option order counterbalancing | Covered | `sample` (`uniform`, `record_slot`) gives a seeded binary flip; or the order is read from the pre-supplied assignment. Order is fixed in state for the trial. |
| P05 Practice trial | Covered | `call` the trial subgraph with fixed parameters (`calibrated`, practice story); `input_map` (§4.4, §6.2). |
| P06 Story presentation | Covered | Story and options in state; interpolated into prompts. Persistence across the trial is rendering (non-goal, §1). |
| P07 Forced binary choice | Covered | `ask` (`single_select`) with `options_ref` bound to a list-typed view of the two options (§4.2, VR-8). |
| P08 Choice certainty | Covered | `ask` (`scale`) (§4.2). |
| P09 Absolute fit | Covered | `ask` (`scale`). |
| P10 Justification | Covered | `ask` (`text`, `multiline`). |
| P11 Stance-conditioned bot reply | **Partial** | `llm.prompt_template` is a single template; there is no system-message field. Stance is *defined* as a system-prompt instruction (L29–L35), so the manipulation cannot be expressed at the level it is defined. Workaround: place the base prompt in `context` and interpolate it into one template (G2). |
| P12 Participant reply 1 | Covered | `llm` (`conversational`) emits the bot message and writes the validated reply to `slot` (§4.3, P8). |
| P13 Second bot reply with history | **Partial** | `llm` is specified as exactly one exchange; no rule says whether the model receives the conversation so far (§4.3, §7.1). Workaround: accumulate a transcript in state and interpolate it (G3). |
| P14 Participant reply 2, terminal | Covered | Second `llm` (`conversational`); its `slot` receives the reply, and the next edge skips any further generation. |
| P15 Crisis stop rule | **Partial** | `llm` writes the *reply* to `slot`, not the generated message (§4.3), so `classify.inputs` cannot address the text that was sent. Workaround: split into `llm` (`silent`) → `classify` → `say` → `ask`, which reconstructs the exchange (G3). |
| P16 End-of-trial message | Covered | `say` (§4.2). |
| P17 Post binary choice | Covered | `ask` (`single_select`); a distinct slot from P07. |
| P18 Post choice certainty | Covered | `ask` (`scale`). |
| P19 Post absolute fit | Covered | `ask` (`scale`). |
| P20 Activation rating | Covered | `ask` (`scale`). |
| P21 Trial record assembly | Covered | `set` + `append` into `list<object>` (§4.5, §6.5.3); factor values from state; `views` for derived deltas (§3.8). |
| P22 Repetition over trials with isolation | Covered | `foreach` over the assignment list; `call` with `state_isolation: isolated` and `output_map` (§4.4, §6.2, VR-23). |
| P23 Early trial termination | **Partial** | A trial subgraph can end early at an `end` node, but §6.2 does not state that a subgraph `end` returns through the caller's `out`, nor how `outcome` reaches the caller. Workaround: write a `trial_status` slot before each `end` and map it out (G4). |
| P24 Session close and export | Covered | `end` (§4.1); `session.export` allowlist (§3.7, VR-21). |

**Covered: 19 · Partial: 5 · Gap: 0.**

The control-flow layer is fully covered. All five partials are in the LLM/prompt layer or in experimental-design assignment.

---

## 4. Worked encodings

### 4.1 The 14-step trial as a subgraph

The chat maps onto two `llm` nodes, each of which emits one bot message and captures the participant's reply:

```yaml
graph:
  id: mobi_trial
  version: 0.1.0
  state_schema:
    story_text:      { type: string, scope: session, mutable: true }
    option_a_text:   { type: string, scope: session, mutable: true }
    option_b_text:   { type: string, scope: session, mutable: true }
    choice_pre:      { type: enum, values: [option_a, option_b], scope: session, mutable: true }
    certainty_pre:   { type: integer, scope: session, mutable: true }
    abs_fit_pre:     { type: integer, scope: session, mutable: true }
    justification:   { type: string, scope: session, mutable: true }
    bot_reply_1:     { type: string, scope: session, mutable: true }
    reply_1:         { type: string, scope: session, mutable: true }
    bot_reply_2:     { type: string, scope: session, mutable: true }
    reply_2:         { type: string, scope: session, mutable: true }
    crisis_flag:     { type: boolean, scope: session, mutable: true }
  views:
    options:
      type: list<enum>
      source: [state.option_a_text, state.option_b_text]
  entry: start
  nodes:
    - { id: start, type: start }
    - { id: choice, type: ask, config: { prompt: "{{ state.story_text }}\n\nWhich option is closer to what you would do or think?", input_type: single_select, options_ref: views.options, slot: choice_pre } }
    - { id: certainty, type: ask, config: { prompt: "How certain are you?", input_type: scale, slot: certainty_pre, response_schema: { min: 1, max: 7 } } }
    - { id: abs_fit, type: ask, config: { prompt: "How close is that option to your own view?", input_type: scale, slot: abs_fit_pre, response_schema: { min: 1, max: 7 } } }
    - { id: justify, type: ask, config: { prompt: "Why?", input_type: text, slot: justification } }
    - id: bot_1
      type: llm
      config:
        model: sonnet-class
        mode: conversational
        prompt_template: "{{ context.base_system }}\n\n{{ state.stance_instruction }}\n\n{{ state.story_text }}"
        slot: reply_1
    - id: bot_2
      type: llm
      config:
        model: sonnet-class
        mode: conversational
        prompt_template: "{{ context.base_system }}\n\n{{ state.stance_instruction }}"
        slot: reply_2
    - { id: closing, type: say, config: { message: "{{ context.trial_close_text }}" } }
    - { id: choice_post, type: ask, config: { prompt: "Same question again: which option is closer?", input_type: single_select, options_ref: views.options, slot: choice_post } }
    - { id: certainty_post, type: ask, config: { prompt: "How certain are you now?", input_type: scale, slot: certainty_post, response_schema: { min: 1, max: 7 } } }
    - { id: abs_fit_post, type: ask, config: { prompt: "How close is that option to your own view now?", input_type: scale, slot: abs_fit_post, response_schema: { min: 1, max: 7 } } }
    - { id: activation, type: ask, config: { prompt: "How activated do you feel right now?", input_type: scale, slot: activation, response_schema: { min: 1, max: 7 } } }
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start, to: choice, type: next }
    - { from: choice, to: certainty, type: next }
    - { from: certainty, to: abs_fit, type: next }
    - { from: abs_fit, to: justify, type: next }
    - { from: justify, to: bot_1, type: next }
    - { from: bot_1, to: bot_2, type: next }
    - { from: bot_2, to: closing, type: next }
    - { from: closing, to: choice_post, type: next }
    - { from: choice_post, to: certainty_post, type: next }
    - { from: certainty_post, to: abs_fit_post, type: next }
    - { from: abs_fit_post, to: activation, type: next }
    - { from: activation, to: done, type: next }
  terminals: [done]
```

Steps 6–7 and 8–9 collapse into `bot_1` and `bot_2`: the node emits the bot message and its `slot` receives the participant's reply. Steps 2–5 and 11–14 are ordinary `ask` nodes. The "Passed to the bot?" column (L63–L78) becomes prompt composition: only the variables a template interpolates reach the model.

### 4.2 Trial loop with isolation

```yaml
- id: trials
  type: foreach
  config:
    over: { list_ref: state.assignment }
    body_entry: run_trial
    item_slot: current_cell
    max_iterations: 40
- id: run_trial
  type: call
  config:
    graph_ref: { id: mobi_trial, version: 0.1.0 }
    state_isolation: isolated
    input_map:
      story_text: state.current_story_text
      stance_instruction: state.current_stance_instruction
    output_map:
      trial_record: state.last_trial_record
```

`state_isolation: isolated` is the right fit: each trial is a self-contained conversation, and only the trial record crosses back (§6.2).

### 4.3 Crisis stop rule — current workaround

Because `llm` does not write the generated message to state, the reply must be generated silently, inspected, then shown:

```yaml
- { id: gen_1, type: llm, config: { model: sonnet-class, mode: silent, prompt_template: "{{ state.prompt_1 }}", slot: bot_reply_1 } }
- { id: check_1, type: classify, config: { backend: rules, model: crisis_rules, inputs: [state.bot_reply_1], labels: [safe, crisis], slot: crisis_flag_label } }
- { id: gate_1, type: branch }
- { id: say_1, type: say, config: { message: "{{ state.bot_reply_1 }}" } }
- { id: ask_1, type: ask, config: { prompt: "", input_type: text, slot: reply_1 } }
```

This works, but it replaces one conversational exchange with four nodes and requires the author to reconstruct history by hand (G3).

### 4.4 What does not encode declaratively today

The **assignment**. Nothing in v0.4.0 can express "take the 18 cells of this design, repeat them *n* times, shuffle within blocks, and draw stories without replacement from per-type pools, reproducibly from the session seed". `sample` draws one edge per execution; `set` and `views` have no permutation, no `range`, and no access to `session.seed`. The only faithful encoding pre-computes the assignment outside the graph and supplies it as `context`, which moves a design guarantee out of the inspectable graph (P1, P3).

---

## 5. Gaps and observations

### G1 — Experimental-design assignment · *Needs spec extension (or accept external pre-computation)*

**Affects** P02, P03. **Severity**: highest. Counterbalancing and without-replacement story sampling are the design's integrity properties; supplying them as `context` makes the graph depend on an unrecorded external computation.

Missing: a deterministic permutation, a without-replacement draw, a range constructor, and read access to `session.seed` from expressions.

### G2 — Prompt composition · *Needs spec extension*

**Affects** P11. **Severity**: high. `llm` has one `prompt_template` and no system-message field. The instrument's primary manipulation is a system-prompt instruction (L29–L35), and the base prompt is a large fixed document (L86–L88). Flattening both into one template changes the thing being studied.

Missing: a system message distinct from the user message, both templated.

### G3 — Conversation state · *Needs spec extension*

**Affects** P13, P15. **Severity**: medium. Two separate omissions:

1. **History.** No rule states whether a `conversational` `llm` receives prior turns. Multi-reply chat requires it (L86).
2. **Generated message addressability.** `llm.slot` receives the participant's reply; the bot's message is recorded for audit (§4.3) but is not a readable state value, so a content check cannot target it (L82).

### G4 — Subgraph termination contract · *Needs clarification*

**Affects** P23. **Severity**: low. §6.2 defines subgraph inputs/outputs and version pinning, but not what a subgraph's `end` does to the caller. A trial that stops early must return control to the loop; today that is expressed by writing a status slot before each `end` and mapping it out.

### Lower-priority observations

- **Trial count is TBD** (L57, L109). The graph must take it from `context` and bound the loop with `max_iterations`; there is no declared "design parameter" concept, and none is needed.
- **Likert ranges are unspecified** (L107). Assessment assumes `scale` with a declared range; the graph is unaffected.
- **Post-crisis continuation is unspecified** (L82). Whether the session proceeds to the next trial after a crisis-terminated one is an authoring decision; the graph can express either.
- **Safety rules are prompt text** (L37–L42). "Nothing false", "never invalidate feelings", the REF negative-appraisal rule, and the INF partial-evidence rule are not machine-checkable outcomes. No spec change is warranted; they belong in the system prompt.
- **Story display persistence** (L65) is rendering, which §1 places out of scope.

---

## 6. Verification

- **Spec citations** re-checked against `spec.md` v0.4.0: node types (`ask`, `say`, `llm`, `classify`, `branch`, `foreach`, `call`, `set`, `end`); `§`/`VR` references (§3.1, §3.2, §3.6, §3.7, §3.8, §4.1–§4.5, §5.3, §5.5, §6.2, §6.5.3, §7.1, VR-8, VR-21, VR-23) all confirmed present with the stated semantics.
- **No invented node types or fields.** The four gap sections name only constructs that do not exist, marked as such.
- **YAML blocks** in §4 parse (PyYAML).
- **Pattern coverage**: all 24 inventory patterns appear exactly once in §3.

---

## 7. Assumptions and limitations

- Assessed against `spec.md` **v0.4.0 as written**; §11 open issues are treated as unavailable.
- "Covered" means the *interaction* is encodable; it does not claim the resulting conversation is indistinguishable from the study design.
- The study's analysis model, covariates, and physiology pipeline are out of scope; only their *records* are considered (P21).
- Where the source is silent (trial count, Likert ranges, post-crisis continuation), the assessment states the reading used rather than resolving it.

---

## 8. Post-integration re-evaluation (v0.5.0)

The §5 gaps were addressed by [`proposal-v0.5.0-llm-interface-and-assignment.md`](./proposal-v0.5.0-llm-interface-and-assignment.md), applied in `spec.md` v0.5.0. Re-scored against the revised spec:

| Pattern | v0.4.0 | v0.5.0 | Basis |
|---|---|---|---|
| P02 Balanced randomized assignment | Partial | **Covered** | `session.seed` (§3.1, §5.3) + `range`/`shuffle`/`flatten`/`map` (§5.5); §10.9. |
| P03 Story sampling without replacement | Partial | **Covered** | `shuffle` + `take` (§5.5). |
| P11 Stance-conditioned reply | Partial | **Covered** | `llm.messages` with `system` roles (§4.3). |
| P13 Second reply with history | Partial | **Covered** | `llm.history: all` plus the conversation-scope rules (§4.3, §7.1). |
| P15 Crisis stop rule | Partial | **Covered** | `llm.message_slot` + `classify` + `branch` (§4.3, §10.9). |
| P23 Early trial termination | Partial | **Covered** | Subgraph termination contract and the reserved `outcome` output (§4.4, §6.2). |
| All other patterns | Covered | Covered | Additive changes; no construct changed meaning. |

**Covered: 24 · Partial: 0 · Gap: 0.** Verdict moves from *Sufficient with reservations* to **Sufficient**.

Two caveats carried forward, neither affecting the verdict:

- The assignment encoding is one dense expression (§10.9) rather than a sequence of readable steps. It is inspectable and its result is state, but an author will want the inventory's prose beside it. A `views` decomposition into named intermediate lists is the remedy if that proves painful.
- `shuffle` is property-based, so two conforming implementations may permute differently for the same seed (§5.5, `spec.md` §11 *Permutation algorithm*). MoBI does not require cross-implementation identity — a study runs on one implementation and the realized assignment is recorded — but the core tier's portability promise is weaker here than elsewhere.

Requirements traceability moved as follows: G1 → §3.1/§3.4/§5.3 (`session.*`), §5.5 (`range`/`shuffle`/`take`/`chunks`/`div`/`mod`, `get` overloads); G2 → §4.3 (`messages`); G3 → §4.3 (`history`, `message_slot`), §7.1 (conversation scope), §7.5 (prompt recording); G4 → §4.4/§6.2 (termination contract, `outcome`).
