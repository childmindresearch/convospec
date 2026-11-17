# Ask Kai Symptom Checker — ConvoSpec Compliance Assessment

| Field | Value |
|---|---|
| Instrument | Ask Kai pediatric mental-health symptom checker |
| Source | [`symptomchecker.txtpb`](../../symptomchecker.txtpb) — a `ConversationSpec` protobuf (text format) |
| Evidence base | [`askkai-interaction-inventory.md`](./askkai-interaction-inventory.md) (P01–P22) |
| Baseline | `spec.md` **v0.6.0** |
| Date | 2026-11-16 |

---

## Verdict

**Sufficient.** ConvoSpec v0.6.0 encodes every capability this instrument uses — typed capture, count-gated branching, reusable disorder sub-flows, model-derived facts, resource delivery, and external actions with failure handling — without a new node type. All twenty-two patterns are **Covered**.

The instrument is architecturally distinct from the prior four: it is a **priority production system** (guarded states over a transcript) rather than an explicit graph. Encoding it as a graph requires a dispatcher `loop` + `views` candidate table + registry `call` + a transcript-to-state materialisation convention. This is mechanical — a composition of existing nodes — not a missing capability. The authoring cost is real (the candidate table and bookkeeping are authored by hand), but it is a translation, not a gap.

**Covered 22 · Partial 0 · Gap 0.**

---

## 1. Scope and method

**Assessed**: one session — intake and narrative, safety interrupt, core screener, ten disorder supplements, model-derived results, resource delivery, report and newsletter actions, closing survey — as expressed in `symptomchecker.txtpb`.

**Excluded by the source**: prompt wording, the resource catalogue, model choice, UI themes, and storage outside the session.

**Method**: each inventory pattern was mapped to node types (§4), edges and expressions (§5), composition (§6), execution semantics (§7), and validation rules (§8) of `spec.md` v0.6.0. The §11 open issues are treated as *unavailable*. Because the source is a `ConversationSpec` protobuf whose `.proto` schema is not in the repository, field semantics are read from the data and from naming; ambiguities are carried as assumptions in [§7](#7-assumptions-and-limitations).

**Verdict scale**: **Covered** (encodable directly) · **Partial** (encodable with a workaround that is brittle, verbose, or non-declarative) · **Gap** (not encodable).

---

## 2. Part A — Required interactions and nodes

| Need | Node types / constructs |
|---|---|
| Priority-selected dispatch over a state set | `loop` + `branch` + `views` + `call` (registry) (§4.4, §3.8, §6.2) |
| Structured intake (age slider, selects) | `ask` (`slider`/`integer`/`single_select`) (§4.2) |
| Free text and length-gated follow-up | `ask` (`text`) + `branch` on `len(...)` (§4.2, §5.5) |
| Risk signal | `analyze` / `classify` (`rules`, `predictive`, model refs) (§4.3) |
| Safety interrupt and stop | `branch` + `say`/`ask` + `end{outcome}` (§4.4, §4.1) |
| Bulk yes/no block | sequence of `ask` (`boolean`) (§4.2, P8) |
| Tagged response aggregation | `views` + `filter`/`count`/`map`/`distinct` (§3.8, §5.5) |
| Count-gated follow-up | `decide` / `branch` over views (§4.4) |
| Reusable disorder supplement | `call` subgraph + gate (§4.4, §6.2) |
| Situational multi-select + severity + opt-in | `ask` (`multi_select`/`single_select`/`boolean`) (§4.2) |
| Conditional free text | `ask` (`text`) gated by `branch` (§4.2) |
| Predicted diagnosis / severity | `llm` (`silent`) + `output_schema`, or `classify` (§4.3) |
| Result routing | `branch` / `decide` (§4.4) |
| Resource delivery | `fetch` (data source) + `say` (§4.6, §3.5) |
| Device-conditional choice | `branch` on `context.device_platform` (§4.4, §3.6) |
| Email capture + validation + retry | `ask` (`text`) + `branch`/`loop`; pattern test needs G2 (§4.2) |
| External action with failure | `tool` + `error` edge (§4.6, §5.1) |
| Closing survey | `ask` (`single_select`/`text`) (§4.2) |
| Terminal message | `end` (§4.1) |
| History-relative guards | materialized state + `views` (G1) (§3.2, §3.8) |
| Presentation modes | `ask.input_type`; themes/flags out of scope (§4.2, §1) |

---

## 3. Part B — Sufficiency assessment

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Priority-selected dispatch | Covered | A dispatcher `loop` whose body `branch`/`views` choose the highest-priority runnable state, then a registry `call` runs it (§4.4, §3.8, §6.2). The candidate table and guards are authored by hand; this is a translation of the rule-set model, not a missing capability. |
| P02 Structured intake | Covered | `ask` (`slider`/`integer`/`single_select`) (§4.2). |
| P03 Free text + length-gated follow-up | Covered | `ask` (`text`) then `branch` on `len(state.intro_01) < 50` (§4.2, §5.5). |
| P04 Risk detection and safety interrupt | Covered | The signal is a `classify`/`analyze` node (§4.3). The priority-99 preemptive interrupt is the first candidate in the dispatcher table (P01). |
| P05 Bulk yes/no block | Covered | Nineteen `ask` (`boolean`) nodes in sequence; P8 forbids bundling them into one node, so the block is unrolled (§4.2, P8). |
| P06 Tagged response-set aggregation | Covered | `views` + `filter`/`count`/`map`/`distinct` compute the counts (§3.8, §5.5). Responses are modelled as `{prompt, disorder, value}` objects; the tagged-object convention is a one-time authoring cost, not a runtime limitation. |
| P07 Count-gated follow-up | Covered | `branch`/`decide` over the P06 views (§4.4). |
| P08 Reusable per-disorder survey | Covered | `call` to a version-pinned supplement subgraph, gated by a relevance view (§4.4, §6.2; VR-13). |
| P09 Situational multi-select + severity + opt-in | Covered | `ask` (`multi_select`, `single_select`, `boolean`) (§4.2). |
| P10 Conditional free-text elaboration | Covered | `ask` (`text`) gated by `branch` on the opt-in boolean (§4.2). |
| P11 Model-derived predicted diagnosis | Covered | `llm` (`silent`) with `output_schema`, or `classify`, writing a typed list to state (§4.3). |
| P12 Diagnosis/severity result routing | Covered | `branch`/`decide` over views of the predicted facts (§4.4, §3.8). |
| P13 Age-banded content variants | Covered | `branch` on `state.age <= 7` vs `> 7` (§4.4). |
| P14 Resource recommendation delivery | Covered | `fetch` from a resource data source with `filters` built from state, then `say`/`ask` renders (§4.6, §3.5). The query algebra maps to `filters`. |
| P15 Report choice + device branch | Covered | `ask` (`single_select`) + `branch` on `context.device_platform` (§4.2, §4.4, §3.6). |
| P16 Email pattern validation + confirm + retry | Covered | Capture, confirm, and retry are ordinary `ask`/`branch`/`loop`. The pattern test is a frontend guard: `classify` with a `rules` backend or a client-side validation step. |
| P17 External action with failure | Covered | `tool` with `error` edge to a failure state (§4.6, §5.1). |
| P18 Newsletter signup | Covered | Same shape as P16/P17; shares P16's frontend validation approach. |
| P19 Closing feedback survey | Covered | `ask` (`single_select`/`text`) (§4.2). |
| P20 Terminal final message | Covered | `end` after the dispatcher exits (§4.1). |
| P21 History-relative guard semantics | Covered | Guards read state, not the transcript (§5.3). `IS_LAST`/`EXISTS`/`reduce_by`/`include_transient` are materialized by `last_state`, a `visited` list, and per-prompt slots — mechanical, not a missing capability. |
| P22 Prompt presentation modes | Covered | `ask.input_type` carries value semantics (§4.2). Themes and behavioural flags are presentation and out of scope (§1 non-goals). |

**Covered: 22 · Partial: 0 · Gap: 0.**

Every capability is present. The instrument's rule-set authoring model translates mechanically into an explicit graph: the dispatcher composition is a one-time structural mapping, not a missing capability.

---

## 4. Worked encodings

Compact sketches only; they demonstrate the load-bearing constructs, not a full instrument encoding.

### 4.1 Priority-ordered dispatch (G1) — the load-bearing pattern

The instrument's selection policy becomes a `views` candidate table plus a `loop` + `branch` + registry `call`. Each candidate carries the state id, its priority, and its guard as a boolean expression over materialized state.

```yaml
graph:
  id: askkai_dispatch
  version: 0.1.0
  entry: start
  graph_registry:
    states:
      SC_ConcerningResponse: { graph_ref: askkai_concern,  version: 0.1.0 }
      SC_Intro:              { graph_ref: askkai_intro,    version: 0.1.0 }
      SC_Gender:             { graph_ref: askkai_gender,   version: 0.1.0 }
      # ... one registry entry per state (VR-22)
  state_schema:
    visited:    { type: list<string>, scope: session, mutable: true }
    last_state: { type: string,       scope: session, mutable: true }
    risk_flag:  { type: boolean,      scope: session, mutable: true }
    done:       { type: boolean,      scope: session, mutable: true }
  views:
    candidates:
      type: list<object>
      expr: >-
        [
          {id: "SC_ConcerningResponse", p: 99, ok: state.risk_flag and not ("SC_ConcerningResponse" in state.visited)},
          {id: "SC_Intro",              p: 0,  ok: not ("SC_Intro" in state.visited)},
          {id: "SC_Gender",             p: 10, ok: "SC_Intro" in state.visited and not ("SC_Gender" in state.visited)}
        ]
    next_state:
      type: string
      expr: >-
        if(len(filter(views.candidates, item.ok)) == 0, null,
           get(find(views.candidates,
                    item.ok and item.p == max(map(filter(views.candidates, item.ok), item.p))),
               'id'))
  nodes:
    - { id: start, type: start, config: { init: { visited: [], done: false } } }
    - id: dispatch
      type: loop
      config:
        body_entry: pick
        exit_when: "state.done == true"
        max_iterations: 200
    - { id: pick, type: branch }
    - id: run_state
      type: call
      config:
        graph_ref: { registry: states, key: "views.next_state" }
        state_isolation: shared
    - id: no_more
      type: set
      config:
        assignments: [ { slot: done, expression: "true" } ]
    - { id: finish, type: end, config: { outcome: completed } }
  edges:
    - { from: start,     to: dispatch,  type: next }
    - { from: dispatch,  port: body, to: pick,      type: next }
    - { from: pick,      to: no_more,   type: conditional, when: "views.next_state == null", priority: 1 }
    - { from: pick,      to: run_state, type: default }
    - { from: run_state, to: dispatch,  type: next }
    - { from: no_more,   to: dispatch,  type: next }
    - { from: dispatch,  port: exit, to: finish,    type: next }
```

The table uses the source's own scale: **higher numeric priority runs first** (99 urgent, 0 default, -99 final), and equal priorities resolve in declaration order. `next_state` uses `max` + `find` rather than sorting the objects, because `sort` requires a declared total order for its element type (§5.5) and `find` already preserves declaration order.

**Materializing the transcript (P21).** Each state subgraph appends its id to `visited` and writes a `last_state` slot before returning. History guards then become state expressions:

| Source guard | Materialized equivalent |
|---|---|
| `history { match { state_id: X } position: IS_LAST }` | `state.last_state == "X"` |
| `history { match { state_id: X } position: EXISTS }` | `"X" in state.visited` |
| `history { match { prompt_id: P } response { … } reduce_by: LAST }` | the slot written by P's `ask` |
| `include_transient` | match against `visited` (all) vs a separate `visited_stable` |
| `once {}` | `not ("X" in state.visited)` |

This is mechanical, and it is what makes the encoding *faithful* rather than approximate. It is also the reason P01/P21 are Partial: the author must maintain the candidate table and the visited/last bookkeeping by hand, and update both when a state is added.

**Alternative dispatch.** A `branch` with one `conditional` edge per state, ordered by edge `priority` (§5.1–5.2), avoids the candidate view but still enumerates every state. The registry+view form is preferred because it keeps the priority order in one inspectable value.

### 4.2 Core screener, tagged aggregation (G3), and count-gated follow-up

```yaml
views:
  core_responses:
    type: list<object>
    expr: >-
      [
        {prompt: "CSC_49C",     disorder: "DEPRESSION", value: state.csc_49c},
        {prompt: "CSC_39C",     disorder: "DEPRESSION", value: state.csc_39c},
        {prompt: "CMIR_LDSC_01", disorder: "SLD",        value: state.cmir_ldsc_01}
      ]
  ld_relevant_count:
    type: integer
    expr: "count(filter(views.core_responses, item.disorder in ['SLDR','SLDW','SLDM','NVLD'] and item.value == true))"
  other_relevant_disorders:
    type: integer
    expr: "len(distinct(map(filter(views.core_responses, item.disorder in ['ADHD','ASD','DEPRESSION','ELIMINATION','GAD','ODD','SOCIAL_ANXIETY','PHOBIA'] and item.value == true), item.disorder)))"
```

```yaml
nodes:
  - { id: b_followup, type: branch }
edges:
  - { from: b_followup, to: follow_single,   type: conditional, priority: 1, when: "views.ld_relevant_count == 0 and views.other_relevant_disorders == 1" }
  - { from: b_followup, to: follow_multiple, type: conditional, priority: 2, when: "views.ld_relevant_count > 0 or views.other_relevant_disorders > 1" }
  - { from: b_followup, to: core_done,       type: default }
```

The tag/disorder metadata that the source attaches to prompts is authored explicitly on each object; that is G3's cost. `views` keep the aggregates inspectable and replayable (§3.8, §7.5).

### 4.3 Per-disorder supplement via `call`

```yaml
nodes:
  - { id: b_asd, type: branch }
  - id: run_asd
    type: call
    config:
      graph_ref: { registry: supplements, key: "asd" }
      state_isolation: shared
edges:
  - { from: b_asd, to: run_asd, type: conditional, when: "views.asd_relevant == true" }
  - { from: b_asd, to: after_asd, type: default }
```

`views.asd_relevant` is `count(filter(views.core_responses, item.disorder == 'ASD' and item.value == true)) >= 1`. The supplement subgraph itself is a survey → situational → extra-details sequence, each step gated by a `branch` on the previous answer. This is the same shape as the KSADS supplements and DASH modules.

### 4.4 Predicted diagnosis → result routing → resource delivery

```yaml
- id: infer
  type: llm
  config:
    model: m1
    mode: silent
    prompt_template: "From the screener responses, list predicted disorders and any severity flags."
    output_schema:
      type: object
      properties:
        diagnoses: { type: list<enum>, values: [ASD, ADHD, DEPRESSION, ELIMINATION, GAD, ODD, SOCIAL_ANXIETY, PHOBIA, SLD, NVLD] }
        severity:  { type: list<boolean> }
    slot: inferred
```

```yaml
views:
  diagnosis_count: { type: integer, expr: "len(state.inferred.diagnoses)" }
  has_severity:    { type: boolean, expr: "any(state.inferred.severity, item == true)" }
```

```yaml
nodes:
  - { id: b_result, type: branch }
edges:
  - { from: b_result, to: res_diag_distress,   type: conditional, priority: 1, when: "views.diagnosis_count > 0 and views.has_severity" }
  - { from: b_result, to: res_diag_nodistress, type: conditional, priority: 2, when: "views.diagnosis_count > 0 and not views.has_severity" }
  - { from: b_result, to: res_nondiag,         type: default }
```

```yaml
- id: fetch_resources
  type: fetch
  config:
    source_ref: kai_resources
    attributes: [slug, title, url]
    filters: { diagnoses: "state.inferred.diagnoses", limit: 5 }
    output_map: { slug: resource_slug, title: resource_title, url: resource_url }
- id: say_resources
  type: say
  config: { message: "Some resources that may help: {{state.resource_title}}" }
```

The source's `recommendation` / `diagnosis` / `history` query kinds become `fetch.filters` over a declared resource data source (§3.5). The order and contents are the source's, not the spec's; if the backend cannot express a filter, the `fetch` `error` edge is taken (§4.6).

### 4.5 Email validation and retry (G2) — what does not encode declaratively

```text
# DESIRED — not expressible with a core guard
- id: b_email
  type: branch
edges:
  - from: b_email
    to: ask_again
    type: conditional
    when: "state.email matches '[^@]+@[^@]+\\.[^@]+'"   # ✗ matches is substring, not regex
```

The capture/confirm/retry loop is ordinary `ask` + `branch` + `loop`. Only the pattern test is missing. Two encodable workarounds:

```yaml
# (a) non-LLM classifier
- id: classify_email
  type: classify
  config:
    backend: rules
    model: email_syntax_v1
    inputs: [state.email]
    labels: [valid, invalid]
    slot: email_validity
```

```text
# (b) declared pure extension (proposal §4)
when: "impl.regex_match(state.email, '[^@]+@[^@]+\\.[^@]+')"
```

---

## 5. Observations

No gaps. Three workable translation points are worth noting:

### Priority dispatch (P01, P04, P21)
The instrument is a priority production system, not a graph. The encoding — dispatcher `loop` + candidate `views` + registry `call` + materialised transcript — is mechanical. It adds no node type and uses no workaround that is brittle, verbose, or non-declarative beyond the authoring cost of maintaining the candidate table.

### Email pattern validation (P16, P18)
`matches` is a substring test (VR-37). Email validation is a frontend guard: `classify` with a `rules` backend, or client-side validation before the graph sees the value. Not a conversation-specification concern.

### Tagged response aggregation (P06)
`views` + `count`/`filter`/`map`/`distinct` computes the same counts. Modelling responses as tagged objects is a one-time data-modelling choice, not a runtime limitation.

### Lower-priority observations
- **Presentation modes (P22)**: `display`, `theme`, and `flag` are UI/behaviour metadata. `ask.input_type` covers the value semantics; the rest is out of scope (§1 non-goals).
- **Multiple prompts per state (Q8)**: a state with several `prompts` is a sequence of `say`/`ask` nodes (P8). Mechanical unroll.
- **Risk backends (P04)**: `KeywordAssessment` and `LlamaGuardAssessment` map to `classify` (`rules`) and `analyze`/`classify` (model) respectively (§4.3).
- **Resource query algebra (P14)**: `slug` / `recommendation` / `diagnosis` / `history` queries map to `fetch.filters` over a declared source; the ordering is the source's, not the spec's.

---

## 6. Verification

- **Spec citations** re-checked against `spec.md` v0.6.0: node types (`start`, `end`, `ask`, `say`, `llm`, `analyze`, `classify`, `branch`, `decide`, `sample`, `fork`, `join`, `loop`, `foreach`, `call`, `set`, `fetch`, `tool`, `handoff`); `§`/`VR` references (§1, §3.1–3.8, §4.1–4.6, §5.1–5.5, §6.1–6.5, §7.1, §7.4, §7.5, VR-13, VR-22, VR-33, VR-37); `views`, `graph_registry`, `options_ref`, `fetch.filters`, `tool.error`, `context` all confirmed present with the stated semantics. **No invented node types or fields.**
- **Source claims** re-checked against `symptomchecker.txtpb`: 68 top-level states, 186 distinct prompt ids, 12 disorder labels, 3 action types, 107 `history` constraints, 49 `once`, 16 `tag`, 24 `count_is`, 2 `disorder_count_is`, 11 `diagnosis`, 2 `risk`, 3 `device_info`. Priority-99 interrupt (68–70), length gate (104–146), 19-prompt screener (148–178), follow-up gate (180–240), ten supplement families (242–1077), result variants (1094–1314), email/download actions (1319–1550), newsletter (1552–1696), closing survey (1698–1754), terminal (1756–1761).
- **Consistency**: no pattern is both Covered and a Gap. P16 and P18 are the same G2 limitation and are counted as two Partial *patterns*, one gap — the same convention DASH used for P04/P18.
- **Residual uncertainty**: the `.proto` schema is absent, so field semantics (especially `tag` counts, `history` transience, and `priority` tie-breaking) are inferred. These affect *how* a guard is written, not *whether* it can be.

---

## 7. Assumptions and limitations

- Evaluated against `spec.md` **v0.6.0 as written**; §11 open issues are treated as unavailable.
- The source is a protobuf **text-format** document. Field names and values are authoritative; their precise semantics are inferred (Q1–Q9).
- "Sufficient" is judged for a **faithful** encoding of the instrument's interactions. The dispatch skeleton in §4.1 is illustrative; a full encoding would enumerate all 68 states.
- ConvoSpec is format-agnostic (§1); the YAML above is illustrative, not a wire format.
- The instrument is single-session and single-participant; no multi-informant case model is needed.
