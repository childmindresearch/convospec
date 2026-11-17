# Proposal: ConvoSpec v0.3.0 — KSADS-Driven Remediation

| Field | Value |
|---|---|
| Status | **Applied** — integrated into `spec.md` v0.3.0 (integration notes in [§13](#13-integration-notes)) |
| Target spec version | 0.3.0 |
| Baseline | `spec.md` v0.2.0 |
| Source evaluation | [`ksads-compliance.md`](./ksads-compliance.md) |
| Evidence base | [`ksads-interaction-inventory.md`](./ksads-interaction-inventory.md) |
| Review input | Design review, 2026-11-16 |

This document proposed changes to `spec.md`. They are now **applied in `spec.md` v0.3.0**; [§13](#13-integration-notes) records integration notes and every deviation from the text below. The body of this document is preserved as the reviewed proposal.

**Citation convention.** `§X` and `VR-N` refer to `spec.md` v0.2.0. References to this document are written `this §X`. Sections marked *(new)* are proposed spec sections that do not exist yet.

---

## 1. Status and scope

The compliance evaluation ([`ksads-compliance.md`](./ksads-compliance.md) §5) identified six gaps in ConvoSpec v0.2.0:

| Gap | Evaluation's recommendation | Disposition here |
|---|---|---|
| G1 Multi-informant / multi-session | First-class `informant` dimension on Session and `ask`; informant-bound child sessions | **Replaced.** Separate sessions plus declared data links; no informant dimension inside a flow (this §3) |
| G2 Cross-item aggregation in guards | Aggregate functions in the guard grammar, or an `aggregate` node | **Replaced.** A type-driven `decide` node; guard grammar unchanged for collections (this §4) |
| G3 Dynamic subgraph dispatch & ordering | Dynamic `call.graph_ref` plus a `foreach`/`dispatch` construct | **Accepted, tightened.** Registry-bounded dispatch plus `foreach` (this §5) |
| G4 Multi-turn probing within a node | A `dialogue`/`probe` node with a turn budget | **Rejected.** Single-exchange nodes and a normative probe-loop pattern (this §6) |
| G5 Rater role / channel | `ask.audience: participant \| caregiver \| clinician` | **Replaced.** One responder question per conversation; no spec field (this §7) |
| G6 Data-driven iteration | Folded into G2/G3 | **Folded** into §5; no separate mechanism (this §8) |

Two constraints shape everything below:

1. **No concurrency across sessions.** G1 is solved by data linking, not by multi-informant orchestration inside one graph. The single-await invariant (§6.4) is untouched.
2. **No compound dialogue nodes.** G4 is solved by decomposing a probe into explicit single-exchange nodes. The node taxonomy stays flat.

### 1.1 Summary of proposed changes

| Kind | Addition |
|---|---|
| Design principles | P8 single-exchange nodes; P9 session identity and declared links |
| Session fields | `case_id`, `role`, `links`, `export` |
| Graph field | `graph_registry` |
| Data-source kind | `session_store` |
| Node types | `decide` (Control family); `foreach` (Control family) |
| Field additions | `fetch.session_ref`; `call.graph_ref` registry form; `foreach` `item.` binding in scoped expressions |
| Validation rules | VR-20 … VR-25 (provisional numbering) |
| Amended rules | VR-3, VR-4, VR-11, VR-13, VR-14 (see this §9.3) |

---

## 2. Review decisions and their consequences

| # | Decision | Consequence for the spec |
|---|---|---|
| **D1** | Multi-informant takes place in **separate sessions with external data linking**, not within one flow. | Session gains identity (`case_id`, `role`) and explicit links; cross-session reads are a declared data-source operation; case-level orchestration is out of scope. |
| **D2** | Decision logic is **type-driven**: operands are typed, and the operand type determines which tests are legal. | One `decide` node with a type-gated test matrix instead of a generic boolean combinator or grammar-level quantifiers. |
| **D3** | **Dynamic ordering** is a missing first-class element. | `graph_registry` bounds all dispatch targets; `foreach` iterates ordered, filtered collections; `call.graph_ref` may resolve through the registry. |
| **D4** | **Multi-turn probing is decomposed into multiple nodes.** | New principle P8; `llm`/`ask` are explicitly single-exchange; a normative loop pattern replaces any compound dialogue node. |
| **D5** | Rater identity is **not** a spec field; it is captured once per conversation. | A responder `ask` at the top of the graph; `session.role` for informant sessions. No `ask.audience`, no transcript schema change. |

D2 and D5 both push in the same direction: keep the spec's primitives general and let the graph author encode domain semantics. D4 does the same for dialogue: expressiveness comes from composition, not from a richer leaf node.

---

## 3. G1 — Multi-informant via linked sessions

### 3.1 Session identity (§3.1)

Add four fields to the Session record:

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | yes for sessions in a case | Groups sessions that concern the same participant/episode. |
| `role` | string | yes for sessions in a case | Informant or rater identity: `participant`, `caregiver`, `clinician`, or a namespaced custom value. |
| `links` | map<string, session_id> | no | Named references to other sessions in the same case. Immutable for the session lifetime. |
| `export` | object | no | Allowlist of state slots readable by linked sessions. |

```yaml
session:
  session_id: S-child-001
  case_id: CASE-1042
  role: child
  links:
    parent: S-parent-001
  export:
    attributes:
      dep_mood_summary_ce: { type: integer }
      dep_trigger: { type: boolean }
    access: link_read
```

**Rules**

- `links` MUST be declared at session start and MUST be immutable thereafter. Immutability keeps replay deterministic (§3.4) and prevents link retargeting mid-session.
- `links` MUST be acyclic. A link to a session that does not exist is a session-start error, not a runtime edge case.
- `export.attributes` is an **allowlist**, not a projection of all state. There is no wildcard. Anything not exported is invisible to linked sessions.
- `export` is versioned with the session's `graph_ref`. A linked reader validates against the exporting session's recorded export schema, not the current graph's.
- A session MUST NOT await, sequence, retry, or terminate another session. Sessions are independent executions; only their exported data crosses the boundary.
- `role` is an open string with reserved values `participant`, `caregiver`, and `clinician`. Domain labels such as `child` or `youth` are permitted; whether to close the enum is open question this §12.5.

Add to §1 **Non-Goals**: *"Case-level orchestration of multiple sessions (spawning, ordering, retry, concurrency) is outside the graph runtime."* This is the deliberate boundary that keeps G1 from turning into a multi-agent scheduler.

### 3.2 New data-source kind: `session_store` (§3.5)

```yaml
data_sources:
  linked_sessions:
    kind: session_store
    attributes:
      dep_mood_summary_ce: { type: integer }
      dep_trigger: { type: boolean }
    access: read_only
```

| Field | Required | Description |
|---|---|---|
| `kind` | yes | `session_store`. |
| `attributes` | yes | Union of attribute schemas this graph may read from linked sessions. |
| `access` | no | MUST be `read_only`; `read_write` is invalid for this kind. |
| `cache_ttl_s` | no | As §3.5. |

The runtime resolves `session_store` against the session's declared `links`. It is not a general session query API: no listing, no search, no unlinked access.

### 3.3 `fetch.session_ref` (§4.6)

| Field | Required | Description |
|---|---|---|
| `session_ref` | conditional | REQUIRED when `source_ref` is a `session_store`; MUST name a key in the current session's `links`. |

```yaml
- id: get_parent_ratings
  type: fetch
  config:
    source_ref: linked_sessions
    session_ref: parent
    attributes: [dep_mood_summary_ce, dep_trigger]
    output_map:
      dep_mood_summary_ce: parent_dep_mood_ce
      dep_trigger: parent_dep_trigger
```

**Rules**

- `session_ref` MUST name a declared link (VR-21). A raw session id MUST be rejected: links are the only addressing mechanism, which keeps the data-flow graph statically inspectable.
- Every requested attribute MUST exist in the `session_store` schema **and** in the linked session's `export.attributes`. A mismatch routes to `error`, never to a silent `null`.
- Cross-session reads MUST NOT write to the linked session, MUST NOT block, and MUST NOT create a turn boundary. `fetch` remains `await: none` (§4.6).
- A session still `active` MAY be read; the reader MUST treat absent exports as `null` and is responsible for its own `on_missing` policy (this §4.2 `decide`).

### 3.4 Resulting pattern: three sessions, one case

```yaml
# Parent session and child session: same graph, role differs.
# Synthesis session: no informant interaction of its own except clinician reconciliation.
graph:
  id: ksads_synthesis
  version: 0.1.0
  entry: start
  data_sources:
    linked_sessions:
      kind: session_store
      attributes:
        dep_mood_summary_ce: { type: integer }
        dep_trigger: { type: boolean }
      access: read_only
  state_schema:
    parent_dep_mood_ce:  { type: integer, scope: session, mutable: true }
    child_dep_mood_ce:   { type: integer, scope: session, mutable: true }
    dep_mood_summary_ce: { type: integer, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - id: get_parent
      type: fetch
      config:
        source_ref: linked_sessions
        session_ref: parent
        attributes: [dep_mood_summary_ce]
        output_map: { dep_mood_summary_ce: parent_dep_mood_ce }
    - id: get_child
      type: fetch
      config:
        source_ref: linked_sessions
        session_ref: child
        attributes: [dep_mood_summary_ce]
        output_map: { dep_mood_summary_ce: child_dep_mood_ce }
    - { id: b_disagree, type: branch }
    - { id: reconcile, type: ask, config: { prompt: "Parent rated {{ state.parent_dep_mood_ce }}, child rated {{ state.child_dep_mood_ce }}. Assign the summary rating.", input_type: scale, slot: dep_mood_summary_ce, response_schema: { min: 0, max: 3 } } }
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start,        to: get_parent,  type: next }
    - { from: get_parent,   to: get_child,   type: next }
    - { from: get_child,    to: b_disagree,  type: next }
    - { from: b_disagree,   to: reconcile,   type: conditional, when: "state.parent_dep_mood_ce != state.child_dep_mood_ce", priority: 1 }
    - { from: b_disagree,   to: done,        type: default }
    - { from: reconcile,    to: done,        type: next }
```

Notes on this pattern:

- P03 (dual informant) is satisfied **without** a new node dimension: identity lives on the session, ratings arrive through `fetch`.
- P13 (discrepancy resolution) is a pairwise guard on disagreement, then a clinician `ask` in the synthesis session. Pairwise comparison is a `branch` guard, not a `decide` test: `decide` compares operands to literals, never to each other (this §4.2).
- P20 (longitudinal) uses the same machinery with `links.prior` and a delta computed by `set` arithmetic.

---

## 4. G2 — `decide`: type-driven decision logic

### 4.1 Motivation

The evaluation's §4.2 shows the failure directly: KSADS triggers a supplement when *any* preceding item scored 3. v0.2.0 forces the author to enumerate every item in one guard string:

```yaml
when: "state.dep_mood_summary_ce >= 3 or state.dep_irritability_summary_ce >= 3 or ..."
```

That is brittle to instrument edits, invisible to operand-level analysis, and cannot express the dynamic cases ("any drug", "any traumatic event") at all.

### 4.2 Node definition (§4.4, Control family)

| Attribute | Value |
|---|---|
| Purpose | Evaluate a declared logical operation over typed state operands and route on the result. |
| Ports | `in`; `true`; `false`; `unknown`; `error` |
| Effects | MAY write `result_slot` |
| Await | `none` |

**Config**

| Field | Required | Description |
|---|---|---|
| `operands` | yes | `{ refs: [...] }` (explicit list) or `{ list_ref: state.x }` (list-typed variable). |
| `test` | conditional | REQUIRED for non-boolean operands; MUST be omitted for boolean operands. |
| `combine` | yes | `any`, `all`, `none`, `n_of_k`. |
| `k` | conditional | REQUIRED for `n_of_k`; MUST be `1 <= k`. |
| `on_missing` | no | `unknown` (default), `skip`, or `fail`. |
| `result_slot` | no | Receives the boolean result for audit or later reuse. |

`operands.refs` entries are either a string reference or `{ ref: state.x, negate: true }`. `negate` applies to that operand's test result.

**Type matrix** — the operand type gates the legal `test`:

| Operand type | Legal `test` | Required parameters | Combines over |
|---|---|---|---|
| `boolean` | none; use `negate` for inversion | — | the operands themselves |
| `integer`, `number` | `gt`, `gte`, `lt`, `lte`, `eq`, `neq` | `value` | per-operand test results |
| `integer`, `number` | `within` | `min`, `max` | per-operand test results |
| `date` | `before`, `after` | `value` | per-operand test results |
| `date` | `within` | `from`, `to` | per-operand test results |
| `enum` | `eq` | `value` | per-operand test results |
| `enum` | `in` | `values` | per-operand test results |
| `list<T>` (via `list_ref`) | `predicate` using the `item.` binding | — | per-element test results |

Any other combination MUST fail validation (VR-20). This is the whole point of the node: the operand type constrains what the author can say, so illegal comparisons are caught statically rather than silently evaluating false.

`test` compares each operand to a **literal** (or a literal set/range). It does not compare operands to each other: pairwise comparison such as `state.parent != state.child` belongs in a `branch` guard (§5.3), where both sides are references.

**`combine` semantics**

| `combine` | Result |
|---|---|
| `any` | true iff at least one operand test is true |
| `all` | true iff every operand test is true |
| `none` | true iff no operand test is true |
| `n_of_k` | true iff at least `k` operand tests are true |

Empty-set results: `any` → `false`, `all` → `true`, `none` → `true`, `n_of_k` → `false`.

**`on_missing` semantics** (applies to `null`/unset operands and to `null` elements in a `list_ref`):

| Value | Behavior |
|---|---|
| `unknown` | If any operand is `null`, take the `unknown` port; `combine` is not evaluated. |
| `skip` | Exclude `null` operands; evaluate `combine` over the remainder. |
| `fail` | Take the `error` port. |

Default is `unknown` deliberately. In KSADS, `0` means *"No information"*, not *"not present"*. A silent `false` for a missing rating would under-trigger a supplement. The `unknown` port makes missingness a first-class, routable outcome instead of a scoring error.

**Requirements**

- Exactly one of `true`, `false`, or `unknown` MUST be taken per evaluation (VR-20).
- A `decide` MUST be a pure function of state: no I/O, no model calls, no context writes.
- `decide` routes only on the boolean outcome (`true` / `false` / `unknown`). It is not a multi-way switch: routing to more than one functional target per outcome is a `branch` concern.
- `n_of_k` with `k` greater than the possible operand count MUST fail validation for `refs` operands; for `list_ref` it evaluates to `false`.

### 4.3 Sketches

OR over a handful of yes/no screens — the case called out in review:

```yaml
- id: si_any_positive
  type: decide
  config:
    operands:
      refs:
        - state.si_ideation
        - state.si_plan
        - { ref: state.si_intent, negate: true }
    combine: any
    result_slot: si_flagged
```

KSADS score-gated supplement trigger, fixed item set:

```yaml
- id: dep_trigger
  type: decide
  config:
    operands:
      refs: [state.dep_mood_summary_ce, state.dep_irritability_summary_ce, state.dep_anhedonia_summary_ce]
    test: { cmp: gte, value: 3 }
    combine: any
    on_missing: unknown
    result_slot: dep_supplement_needed
```

Dynamic case — "any drug at threshold":

```yaml
- id: any_drug_at_threshold
  type: decide
  config:
    operands:
      list_ref: state.drug_ratings
    test: { predicate: "item.score >= 3" }
    combine: any
    result_slot: sud_trigger
```

Completion gate for any-order navigation (P10):

```yaml
- id: all_sections_done
  type: decide
  config:
    operands:
      list_ref: state.section_status
    test: { predicate: "item == 'complete'" }
    combine: all
```

### 4.4 Rejected alternative: aggregates in the guard grammar

Adding `any(list, pred)`, `all`, `count`, `max` to §5.3 was the evaluation's first suggestion. Rejected because:

- It keeps the operand set inside a string, so no validator can enumerate which items fed a trigger — exactly the auditability a clinical instrument needs.
- It has no place to express the `unknown`/`on_missing` distinction, which is the clinically important case.
- It widens the expression grammar (§5.3) for every consumer of guards, including simple `loop.exit_when` predicates that never need collections.

`decide` keeps the grammar closed and confines the new semantics to one inspectable node.

### 4.5 Relationship to `branch`

`branch` remains the right node for heterogeneous routing (different targets per condition). `decide` handles homogeneous logic over a set of operands. They are complementary, not layered: a `decide` result may feed a `branch`, but a `decide` SHOULD NOT be inserted merely to pre-compute a guard that `branch` could evaluate inline.

---

## 5. G3 — Dynamic ordering and dispatch

### 5.1 The gap

`call.graph_ref` is REQUIRED and version-pinned (§4.4, VR-13). `loop.body_entry` is a static node id (§4.4). Neither can express "run the supplements, in onset order, for whichever supplements the screen triggered". That is P09, and P10 needs the same machinery for its worklist.

### 5.2 `graph_registry` (§6.1)

```yaml
graph_registry:
  supplements:
    dep:
      graph_ref: ksads_dep_supplement
      version: 0.1.0
      attributes: { required: true, onset_age: 9 }
    gad:
      graph_ref: ksads_gad_supplement
      version: 0.1.0
      attributes: { required: false, onset_age: 7 }
    ptsd:
      graph_ref: ksads_ptsd_supplement
      version: 0.1.0
      attributes: { required: false, onset_age: 11 }
```

- A registry is a named, version-pinned map from key to subgraph entry.
- An entry has `graph_ref`, `version`, and optional static `attributes` (e.g. `required`, `onset_age`). Attributes are graph metadata, not state, and are what `foreach.filter`/`order_by` read.
- A registry projection binds each element as `{ key, graph_ref, version, <attribute>... }`, so `item.key` and `item.onset_age` are both valid element-scoped references.
- Dynamic references resolve **only** through a registry. There is no runtime graph lookup by id.
- The validator can therefore enumerate every possible dispatch target from the graph source alone. P4 (inspectable) and VR-13's intent both survive.

`call.graph_ref` MAY take the registry form:

```yaml
- id: call_next_supplement
  type: call
  config:
    graph_ref: { registry: supplements, key: "state.current_supplement.key" }
    state_isolation: shared
    input_map:  { dep_mood_summary_ce: dep_mood_summary_ce }
    output_map: { dep_diagnosis: current_diagnosis }
```

Resolution failure (key not in registry) routes to `error`; it is not a validation failure, because the key is state-derived.

### 5.3 New node: `foreach` (§4.4, Control family)

| Attribute | Value |
|---|---|
| Purpose | Iterate an ordered, filtered collection, invoking a body once per element. |
| Ports | `in`; `body` (only when `body_entry` is used); `exit`; `error` |
| Effects | MAY write `item_slot`, `index_slot` |
| Await | inherited from the body |

**Config**

| Field | Required | Description |
|---|---|---|
| `over` | yes | `{ list_ref: state.x }` (list-typed variable) or `{ registry: name }` (registry projection). |
| `filter` | no | Scoped expression over `item.`; elements failing it are skipped. |
| `order_by` | no | Scoped expression over `item.`; ascending unless `descending: true`. |
| `descending` | no | Boolean; default `false`. |
| `tiebreak` | no | Scoped expression over `item.`; MUST yield a total order together with `order_by`. |
| `body_graph` | conditional | Registry form, resolved per element. REQUIRED if `body_entry` is absent. |
| `body_entry` | conditional | Static node id. REQUIRED if `body_graph` is absent. |
| `item_slot` | yes | State variable receiving the current element. |
| `index_slot` | no | State variable receiving the zero-based iteration index. |
| `max_iterations` | yes | Positive integer. |

**Requirements**

- The iteration set MUST be snapshotted at entry. Mutating the source collection inside the body MUST NOT change the set or the order of the current run (determinism, §3.4).
- Ordering MUST be total. If `order_by` can produce ties, `tiebreak` MUST be declared; otherwise the runtime MUST use stable input order.
- `max_iterations` MUST bound the loop even when the collection is larger; on reaching it, take `exit` (or `error` if declared).
- An empty iteration set MUST take `exit` immediately.
- `foreach` MUST NOT appear in a forked silent stream if its body can await `user` (VR-11, unchanged in spirit).
- `item.` and `index.` references MUST type-check against the element type derived from `over` (VR-23).
- When `body_graph` is used the `body` port is unavailable and MUST have no edges; when `body_entry` is used the invoked node MUST be reachable only from that port.

Sketch — onset-ordered supplement dispatch (P09):

```yaml
- id: run_supplements
  type: foreach
  config:
    over: { registry: supplements }
    filter: "item.required == true"
    order_by: "item.onset_age"
    tiebreak: "item.key"
    body_graph: { registry: supplements, key: "item.key" }
    item_slot: current_supplement
    index_slot: supplement_index
    max_iterations: 5
```

`state.supplement_worklist` is not required: the worklist is the registry projection filtered by per-supplement trigger flags, which keeps membership bounded and statically known.

### 5.4 Scoped `item.` binding (§5.3)

Add one production, legal **only** inside node-scoped expression fields (`decide.test.predicate`, `foreach.filter`, `foreach.order_by`, `foreach.tiebreak`):

```ebnf
local_ref = "item" , { "." , identifier } ;
```

`item` MUST NOT appear in edge guards, `set` expressions, or `loop.exit_when`. Those contexts have no iteration binding, and allowing it would make the grammar context-sensitive in ways a validator cannot check cheaply.

### 5.5 P10 — any-order navigation

No new node is needed; the pattern composes:

1. `ask(single_select)` presents the sections not yet complete, using `options_ref` bound to a list-typed state variable (this §9.1) rather than a static option list.
2. `call` invokes the chosen section.
3. `set` marks the section complete in `state.section_status`.
4. `loop.exit_when` uses `decide(combine: all)` over `state.section_status` (this §4.3) to terminate.

Recorded as a normative pattern, not a node, because the sequencing policy is authorial, not structural.

`options_ref` is a proposed `ask` field (this §9.1): a list-typed state reference that supplies the option set at turn time. The response validates against the resolved set, not a static list.

---

## 6. G4 — Single-exchange nodes and probe loops

### 6.1 New principle (§2)

> **P8 — Single-exchange nodes.** A node MUST perform at most one user exchange. Multi-exchange sub-dialogues MUST be expressed as an explicit loop of single-exchange nodes. No node type may encapsulate an unbounded internal dialogue.

Amend §4.3 `llm` (`mode: conversational`): *"The node MUST emit exactly one user-facing message and await exactly one reply. It MUST NOT conduct further exchanges internally, and MUST NOT hold an internal turn budget."*

Amend §4.2 `ask` similarly: one prompt, one validated response, `retries` counts re-prompts of the **same** prompt and does not constitute a new exchange.

### 6.2 Normative pattern: bounded probe loop (§6.5, new)

```yaml
- id: probe_loop
  type: loop
  config:
    body_entry: probe_turn
    exit_when: "state.probe_status == 'complete' or state.probe_status == 'declined'"
    max_iterations: 5
    counter_slot: probe_turn_count
- id: probe_turn
  type: llm
  config:
    model: m1
    mode: conversational
    prompt_template: "Ask one question about {{ state.current_item }}, using this probe bank: {{ state.probe_bank }}."
    slot: probe_reply
- id: probe_assess
  type: classify
  config:
    backend: llm
    model: m1
    inputs: [state.probe_reply]
    labels: [need_more, complete, declined]
    slot: probe_status
```

**Rules for the pattern**

- The exit flag MUST be written by a declared silent node (`classify`, `llm` in `silent` mode, or `analyze`) with recorded provenance (§4.3).
- A loop whose exit depends on a model-written flag MUST declare `max_iterations` (VR-24). The flag is a judgment, not a guarantee.
- Each probe turn is a separate turn (§3.3): separately validated, separately transcribed, separately subject to `limits`.
- Probe banks and item text come from state or `call` subgraphs; the loop body MUST NOT hard-code item content if the graph is reused across items.

### 6.3 Rejected alternative: `dialogue` / `probe` node

The evaluation's G4 recommendation — a node with `turn_budget` and an internal exit predicate — is rejected:

| Concern | Consequence of a compound node |
|---|---|
| Control | The author cannot branch between exchanges: no mid-probe safety check, no escalation, no switch of probe strategy. |
| Inspectability (P4) | State reads/writes inside the sub-dialogue become invisible to static analysis; `effects` would have to be over-approximated. |
| Turn semantics (§3.3) | A compound node needs its own turn model, duplicating validation, retry, timeout, and transcript rules. |
| Single-await (P5/§6.4) | An internal await loop is not expressible in the edge model; the node would become a second execution engine. |
| Testability | Per-node fixtures become per-node sub-fixtures; replay of a model-judged exit is opaque. |

Accepted cost: probe loops are more verbose to author. Mitigations already in the spec: `call` for reusable probe-bank subgraphs, `loop` for repetition, and `decide` for compound exit conditions. Verbosity is the price of control, and review chose control.

---

## 7. G5 — Responder identity without a new field

No spec change. The pattern is:

```yaml
- id: q_responder
  type: ask
  config:
    prompt: "Who is answering today?"
    input_type: single_select
    slot: responder_role
    response_schema: { options: [youth, caregiver, clinician, other] }
```

- Asked once, at the start of the conversation, session-scoped, and reused in templates (`{{ state.responder_role }}`) and in `decide`/`branch` guards.
- It is consistent for the whole conversation, which matches how the interview is actually administered.
- For informant sessions (this §3), `session.role` is set by the orchestrator; the responder question becomes a confirmation or is skipped by graph design.

Why not `ask.audience`: a clinical-only field over-fits the spec. Audience semantics are already expressible as state plus templates, and adding a field would require transcript schema changes, rendering semantics, and a validation rule for a single domain.

Open question (see this §12): whether turn records should carry `session.role` for audit. Deferred — no schema change proposed here.

---

## 8. G6 — Data-driven iteration, folded

The trauma-event list (P15) and drug list (P16) need no separate mechanism:

- Iteration: `foreach` with `over: { list_ref: state.event_types }` or `over: { registry: ... }`.
- Thresholding: `decide` with `list_ref` and a `predicate` (this §4.3).
- Gating: `branch` on the `decide` result.

No `aggregate`/`reduce` node is proposed. If list construction becomes a real authoring burden, that is a `set`-expression question (this §12), not a node-type question.

---

## 9. Consolidated spec change list

### 9.1 Sections to amend

| Spec section | Change |
|---|---|
| §1 Non-Goals | Add case-level session orchestration. |
| §2 Design Principles | Add P8 (single-exchange), P9 (session identity and declared links). |
| §3.1 Session | Add `case_id`, `role`, `links`, `export`; add link rules. |
| §3.5 Data Sources | Add `session_store` kind. |
| §3.7 (new) | Session links and cross-session data. |
| §4.2 `ask` | One-exchange wording; add `options_ref` for dynamic option sets. |
| §4.3 `llm` | One-exchange wording; no internal turn budget. |
| §4.4 Control | Add `decide`; add `foreach`; extend `call.graph_ref`. |
| §4.6 `fetch` | Add `session_ref`. |
| §5.3 Grammar | Add scoped `local_ref` production. |
| §6.1 Graph anatomy | Add `graph_registry`. |
| §6.2 Subgraphs | Registry-bounded dynamic composition. |
| §6.5 (new) | Canonical patterns: probe loop, any-order navigation. |
| §8 Validation | Add VR-20…VR-25; amend VR-3/4/11/13/14. |
| §10 Examples | Add linked-session synthesis, `decide` trigger, `foreach` dispatch. |
| §11 Open Issues | Update per this §12. |

### 9.2 Proposed validation rules (numbering provisional)

- **VR-20** — Every `decide` operand MUST resolve in `state_schema`; `test.cmp` MUST be legal for the operand type per the type matrix and MUST carry the parameters that `cmp` requires; `n_of_k` MUST declare `k`; exactly one of `true`/`false`/`unknown` MUST be taken per evaluation.
- **VR-21** — Every cross-session read MUST name a declared `session.links` entry; the source MUST be a `session_store` with `access: read_only`; every requested attribute MUST be exported by the linked session.
- **VR-22** — Every dynamic `graph_ref` MUST resolve through a declared `graph_registry`; every registry entry MUST be version-pinned; no other dynamic subgraph resolution is permitted.
- **VR-23** — Every `foreach` MUST iterate a declared list-typed variable or a registry projection; `filter`/`order_by`/`tiebreak`/`body_graph` expressions MUST type-check against the element type; `max_iterations` MUST be declared; ordering MUST be total.
- **VR-24** — No node may perform more than one user exchange; any `loop` whose `exit_when` references a model-written variable MUST declare `max_iterations`.
- **VR-25** — Sessions in a case MUST declare `case_id` and `role`; `links` MUST be acyclic; `export.attributes` MUST be typed.

### 9.3 Existing rules affected

| Rule | Effect |
|---|---|
| VR-3 | `decide` and `foreach` need outbound-transition coverage on their functional ports. |
| VR-4 | New ports (`true`/`false`/`unknown`, `body`/`exit`, `error`) must be registered for port validation. |
| VR-8 | `ask.options_ref`, where used, MUST resolve to a list-typed state variable whose element type matches the `ask` input type. |
| VR-11 | `foreach` inherits the fork-stream restriction when its body can await `user`. |
| VR-13 | Version-pinning extends to registry entries; intent unchanged. |
| VR-14 | Edge-type validity table extends to the new ports (`conditional`/`default` not legal from `decide`'s boolean ports). |
| VR-7 | `item_slot`/`index_slot`/`result_slot` writes must be declared like any other write. |

### 9.4 P9 — session identity and links

> **P9 — Session-scoped identity, declared links.** Participant and informant identity belong to the session, not to the graph. Data crosses session boundaries only through declared, immutable, acyclic links and declared exports.

This is the principle that makes G1 a data-model question rather than an orchestration question.

---

## 10. Determinism and inspectability impact

| Concern | Assessment |
|---|---|
| P1 Declarative | Registry bounds dispatch; links are declared; exports are allowlisted. No runtime discovery. |
| P3 Deterministic control | `decide` is pure. `foreach` snapshots the set, requires a total order, and bounds iterations. Session data is immutable once read. |
| P4 Inspectable | `decide` operand sets are enumerable; `foreach` element types are derived from the schema; registry targets are enumerable at validation time. |
| P5 Single-await | Unchanged. Sessions are independent; no cross-session await. `foreach` inherits the fork restriction. |
| P6 Explicit failure | `unknown` becomes a routable outcome; resolution failure routes to `error`. |
| §3.4 / §7.5 Replay | Same graph, seed, inputs, and exported linked data reproduce the same decisions. Linked data MUST be recorded in the session trace to make replay sound. |

One residual risk worth stating: a `session_store` read of a still-active linked session can observe partial data. Replay therefore requires the **observed snapshot** to be recorded, not merely the link. This is a runtime requirement, not a spec primitive.

---

## 11. Traceability

Verdicts after proposal. "Pattern" refers to the inventory IDs.

| Pattern | Before | After | Basis |
|---|---|---|---|
| P01 Unstructured intro | Partial | Covered | Probe-loop pattern (this §6.2) |
| P02 Demographics | Covered | Covered | `ask` input types |
| P03 Multi-informant item | Partial | Covered | Separate sessions, links, synthesis `fetch` (this §3) |
| P04 Dual timeframe | Covered | Covered | Unchanged; first-class pairing deferred |
| P05 Probe → clarify → rate | Partial | Covered | Probe-loop pattern (this §6.2) |
| P06 Rubric scoring | Covered | Covered | `llm` + `output_schema` / `classify` |
| P07 Skip-out | Covered | Covered | `branch` |
| P08 Cross-item trigger | Partial → Gap | Covered | `decide` `any`/`all`/`none`/`n_of_k` (this §4) |
| P09 Dynamic dispatch + order | Gap | Covered | `graph_registry` + `foreach` (this §5) |
| P10 Any-order navigation | Partial → Gap | Covered | Pattern: menu + `call` + `decide(all)` (this §5.5) |
| P11 Completion checklist | Covered | Covered | `ask` `multi_select` + `date` |
| P12 Summary checklist | Covered | Covered | `ask` / `llm` in synthesis session |
| P13 Discrepancy resolution | Partial | Covered | Synthesis `fetch` + `decide` + clinician `ask` (this §3.4) |
| P14 Family-history grid | Partial | Partial | Nested `foreach` plausible; 2-D ergonomics unvalidated, deferred |
| P15 Trauma "ever" screen | Partial | Covered | `foreach` + `decide` (this §8) |
| P16 Substance list | Partial | Covered | `foreach` + `decide` (this §8) |
| P17 Risk escalation | Covered | Covered | `branch` → `handoff` → `end{escalated}` |
| P18 Timeframe anchoring | Covered | Covered | `context` + graph config |
| P19 Informant adaptation | Covered | Covered | `branch` on age; `session.role` makes informant explicit |
| P20 Longitudinal | Partial | Covered | `links.prior` + `fetch` + `set` deltas (this §3.4) |

Totals: **Covered 19 · Partial 1 · Gap 0**. The one remaining partial is ergonomic, not a capability, gap. *(Corrected: this traceability table originally stated "Covered 18 · Partial 2" and counted P04 among the partials, though the table itself lists P04 as Covered. P14 is the sole remaining partial — see [`ksads-compliance.md` §9](./ksads-compliance.md#9-post-integration-re-evaluation-v060).)*

### 11.1 Explicitly not proposed

| Item | Reason |
|---|---|
| `ask.audience` (G5 as a field) | Over-fits; responder question is sufficient (this §7) |
| `dialogue` / `probe` node (G4) | Rejected; single-exchange nodes preferred (this §6.3) |
| Guard-grammar aggregates (G2) | Rejected; `decide` keeps operand sets inspectable (this §4.4) |
| Dual-timeframe paired-rating sugar (P04) | Deferred; two slots work today |
| First-class 2-D grid (P14) | Deferred by design (D2: logic stays flat). Nested `foreach` is now forbidden (VR-32, §6.5.4); the grid is encoded as outer `foreach` + inner `call`/unroll. |
| Multi-session orchestration | Out of scope by design (this §3.1) |
| Informant dimension on `ask`/Session flows | Replaced by session `role` (this §3.1) |

---

## 12. Open questions for the regroup session

1. **`decide` vs `branch`.** Should `decide` absorb `branch` (one decision primitive with an optional `operands` block), or stay complementary? Complementarity is proposed, but it does mean two routing nodes.
2. **Sentinel values.** Should state variables declare `missing_values: [0]` so `on_missing` can detect "No information" without repeating node-level policy? This is the single biggest clinical-correctness question in the proposal.
3. **List construction.** `set` has no list append/collect operators. Is registry projection (this §5.3) sufficient for worklists, or does the grammar need `append`/`collect`?
4. **`session_store` semantics.** Retention, redaction, snapshot-vs-live reads, and whether exports are frozen at session end. this §10 notes that replay requires recorded snapshots.
5. **`session.role` typing.** Closed enum or namespaced string? A closed enum is easier to validate; a string is easier to extend for non-clinical domains.
6. **Nested `foreach`.** Allow directly for P14, or require a `call` per dimension?
7. **`foreach` await.** Inherit from body (proposed, symmetric with `loop`) or force non-blocking bodies and require `call` for interactive iteration?
8. **Turn audit.** Should turn records carry `session.role` / responder? Proposed: no, until a second domain needs it.
9. **Integration order.** Proposed: G1 (§3.1/§3.7) → G2 (`decide`) → G3 (`foreach`/registry) → G4 (principles + patterns). The `foreach` sketches reference `decide`, so G2 should land first.

---

## 13. Integration notes

Applied in `spec.md` v0.3.0, following the order recommended in §12.9 (G1 → G2 → G3 → G4). Sections 3–11 landed as proposed, with the deviations below.

### 13.1 Deviations and additions

| # | Change | Reason |
|---|---|---|
| 1 | **Edge model gained a `port` field** (§5.1) | §5.1 already required multi-port nodes to "specify the source port" without naming a field. `decide` (`true`/`false`/`unknown`) and `foreach` (`body`/`exit`) made the omission blocking. `port` defaults to `out`. |
| 2 | **Port-scoped resolution rule** (§5.2) | Not in the proposal. Defines what happens when a `decide` outcome port has no edge: take `error`, else `on_error`; a port that can be active with neither is a validation failure. |
| 3 | **`ask.options_ref` specified in full** (§4.2, VR-8) | Flagged in §5.5 and §9.1 as a scope addition; required to justify P10 as Covered. |
| 4 | **`local_ref` also legal in `foreach.body_graph`** (§5.3) | The `body_graph` key is evaluated per element, so it needs the same `item.` binding as `filter`/`order_by`. |
| 5 | **§7.2 suspending-node list corrected** | `loop` was already missing from it in v0.2.0; `foreach` joins it. Both inherit await from their body. |
| 6 | **§10.3 YAML typo fixed** (`confidence:{` → `confidence: {`) | Pre-existing in v0.2.0; the example did not parse as YAML. |

### 13.2 Open questions status

Eight of the nine questions in §12 remain open; none blocked integration. Question 7 (`foreach` await) was settled during integration in favour of inheritance. Every open question is carried into `spec.md` §11.

| §12 question | `spec.md` §11 entry |
|---|---|
| 1 `decide` vs `branch` | Decision-node consolidation |
| 2 Sentinel values | Missing-value sentinels |
| 3 List construction | Collection construction |
| 4 `session_store` semantics | Session stores |
| 5 `session.role` typing | Session role typing |
| 6 Nested `foreach` | Nested iteration |
| 7 `foreach` await | Resolved in favour of inheritance (§4.4) |
| 8 Turn audit fields | Turn audit fields |
| 9 Integration order | Followed; recorded here |

### 13.3 Verification performed

- All 18 YAML blocks in `spec.md` parse (PyYAML).
- VR-1…VR-25 are defined exactly once, in order, with no duplicates.
- Every `§`/`VR` cross-reference introduced by this proposal resolves in the revised spec.
- Traceability in §11 above matches [§8.1 of the compliance evaluation](./ksads-compliance.md#81-per-pattern-verdicts-after-v030).

### 13.4 Post-integration corrections (v0.6.0)

- **Tally correction.** §11 and the compliance evaluation's §8.1 originally reported *Covered 18 · Partial 2* and named P04 as a remaining partial, although both tables mark P04 Covered. The correct v0.3.0 figure is **Covered 19 · Partial 1 · Gap 0**; P14 is the sole remaining partial. Corrected in place above.
- **P14 rationale superseded.** The deferral reason "needs nested `foreach` validation first" is obsolete. v0.4.0 added §6.5.4, which makes the outer-`foreach` + inner-`call`/unroll pattern normative and forbids nested `foreach` (VR-32). P14 is therefore fully encodable and remains Partial only by the explicit design decision D2, matching DASH P12. See [`ksads-compliance.md` §9](./ksads-compliance.md#9-post-integration-re-evaluation-v060).
