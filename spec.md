# Conversation Specification (ConvoSpec)

**Node Types and Graph Composition**

| Field | Value |
|---|---|
| Status | RC1 |
| Version | 0.6.0 |
| Audience | Spec authors, runtime implementers, conversation designers |
| Scope | Conceptual node taxonomy and graph semantics (format-agnostic) |

**Revision notes (0.6.0).** Revision driven by the walrus compliance evaluation ([`evaluations/walrus/walrus-compliance.md`](evaluations/walrus/walrus-compliance.md)) and the remediation proposal ([`evaluations/walrus/proposal-v0.6.0-string-matching-and-turn-rendering.md`](evaluations/walrus/proposal-v0.6.0-string-matching-and-turn-rendering.md)). Changes: `matches` is now defined as an ASCII-case-insensitive contiguous-substring test (§5.3) with validation rule VR-37; §7.1 gains a per-turn rendering table for `ask`, `say`, and `llm`; the `loop` declares its `error` port (§4.4); §7.4 states the relationship between `limits` and `loop.max_iterations`; new §10.10. No node type was added. One behaviour changes: history now renders a turn's emitted output before the participant's response, correcting v0.5.0's user-then-assistant order for bot-speaks-first turns (§7.1, §11).

v0.6.0 has been assessed against two additional instruments (Ask Kai, GuideManifold) with no capability gaps. A [node taxonomy consolidation review](evaluations/spec-consolidation-review.md) confirmed all 19 node types are distinct and irreducible. Status promoted to **RC1**.

**Revision notes (0.5.0).** Additive revision driven by the MoBI compliance evaluation ([`evaluations/mobi/mobi-compliance.md`](evaluations/mobi/mobi-compliance.md)) and the remediation proposal ([`evaluations/mobi/proposal-v0.5.0-llm-interface-and-assignment.md`](evaluations/mobi/proposal-v0.5.0-llm-interface-and-assignment.md)). Changes: the `session.*` expression root and `session.seed` access (§3.1, §3.4, §5.3); core functions `range`, `shuffle`, `take`, `chunks`, `div`, `mod` and refined `get` overloads (§5.5); `llm.messages`, `llm.history`, and `llm.message_slot` (§4.3); conversation scope and prompt-assembly rules (§7.1); the subgraph termination contract and reserved `outcome` output (§4.4, §6.2); validation rules VR-33–VR-36 and an amendment to VR-7. No node type was added. One behaviour is newly defined where v0.4.0 was silent: a `conversational` `llm` now sees prior turns in its conversation scope by default (§4.3, §7.1).

**Revision notes (0.4.0).** Additive revision driven by the DASH compliance evaluation ([`evaluations/dash/dash-compliance.md`](evaluations/dash/dash-compliance.md)) and the remediation proposal ([`evaluations/dash/proposal-v0.4.0-expressions-and-views.md`](evaluations/dash/proposal-v0.4.0-expressions-and-views.md)). Changes: typed expression library with collection/object constructors and aggregates (§5.5); expression extensibility tiers — a mandated core plus declared implementation extensions (§5.6); `set` widened to any type with atomic simultaneous assignment (§4.5); derived read-only `views` (§3.8) and the `views.*` namespace; canonical collection/accumulation patterns (§6.5.3); validation rules VR-26–VR-31 and amendments to VR-7/8/20/23. No node type was added and no v0.3.0 construct changed meaning.

**Revision notes (0.3.0).** Additive revision driven by the KSADS compliance evaluation ([`evaluations/ksads/ksads-compliance.md`](evaluations/ksads/ksads-compliance.md)) and the remediation proposal ([`evaluations/ksads/proposal-v0.3.0-ksads-remediation.md`](evaluations/ksads/proposal-v0.3.0-ksads-remediation.md)). Changes: new `decide` and `foreach` control nodes; `graph_registry` for bounded dynamic dispatch; session identity, links, and exports for multi-informant cases; the `session_store` data-source kind; `ask.options_ref`; principles P8 (single-exchange nodes) and P9 (session-scoped identity); validation rules VR-20–VR-25. No existing node type changed incompatibly.

## Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

A **node** is a typed unit of conversation behavior. An **edge** is a directed transition between nodes. A **graph** is a set of nodes and edges with a single entry and one or more terminals. A **turn** is a unit of user-facing exchange (see §3.3).

Pseudo-YAML is used for illustrative serialization only; no wire format is mandated by this version.

## Table of Contents

1. [Purpose and Non-Goals](#1-purpose-and-non-goals)
2. [Design Principles](#2-design-principles)
3. [Core Model](#3-core-model)
4. [Node Taxonomy](#4-node-taxonomy)
5. [Edges and Control Flow](#5-edges-and-control-flow)
6. [Graph Composition](#6-graph-composition)
7. [Execution Semantics](#7-execution-semantics)
8. [Validation Rules](#8-validation-rules)
9. [Extensibility](#9-extensibility)
10. [Worked Examples](#10-worked-examples)
11. [Open Issues](#11-open-issues)

---

## 1. Purpose and Non-Goals

This specification defines the primitive **node types** from which arbitrary conversations can be composed, and the **graph semantics** that govern how those nodes connect and execute. It targets conversations ranging from fully structured survey delivery to semi-structured, LLM-driven interviews with branching, context management, triage, and silent background model calls.

**Goals**

- Provide a small, orthogonal set of node types that compose into arbitrary conversation structures.
- Keep control flow deterministic and statically inspectable.
- Quarantine model nondeterminism inside well-defined node types.
- Support silent background inference (e.g., risk detection) alongside user-facing turns.
- Support reproducible randomization of branches and response variants.
- Distinguish mutable session State, read-only inline Context, and external Data Sources, with declared, auditable access.
- Support multi-session cases in which informant-specific sessions exchange declared, read-only data.
- Support declarative decision logic over typed state operands and runtime-ordered dispatch of reusable subgraphs.
- Support declarative computation over collections — aggregates, constructors, and named derived views — so scores, counts, and summaries are inspectable values rather than out-of-band code.
- Support experimental designs as graphs: balanced assignment, without-replacement sampling, and reproducible ordering are declared values derived from the session seed, not pre-computed outside the graph.

**Non-Goals**

- A concrete serialization or wire format (deferred; see §11).
- A prompt-authoring language or model API.
- UI/rendering details beyond input-type semantics.
- Multi-agent orchestration, or case-level orchestration of multiple sessions (spawning, ordering, retry, concurrency), outside a single conversation graph.
- A full plugin ABI for custom nodes (stub only; see §9).
- A reporting, dashboard, or presentation layer. Derived views (§3.8) are readable values; how they are displayed is out of scope.
- An exhaustive standard library. §5.5 mandates a minimum; implementations MAY expose further pure operations from their host language as declared extensions (§5.6).

---

## 2. Design Principles

- **P1 — Declarative.** A graph MUST fully describe its structure; runtime behavior MUST be derivable from the graph plus state, context, and inputs, without hidden state.
- **P2 — Composable.** Every node type MUST compose with every other node type through a common port and edge contract.
- **P3 — Deterministic control.** Branching, sampling, and iteration MUST be deterministic given the session seed and inputs. Model calls MAY be nondeterministic, but their effects MUST be confined to declared state writes.
- **P4 — Inspectable.** Every node MUST declare its state/context reads and state writes and its await semantics so a validator can reason about the graph without executing it.
- **P5 — Single-await.** At most one execution path MAY await end-user input at any time (§6.4).
- **P6 — Explicit failure.** Every node that can fail or block SHOULD declare error and timeout transitions (§5).
- **P7 — Extensible without churn.** New node types MUST be addable through a declared contract without altering existing graph structure (§9).
- **P8 — Single-exchange nodes.** A node MUST perform at most one user exchange. Multi-exchange sub-dialogues MUST be expressed as an explicit loop of single-exchange nodes; no node type may encapsulate an unbounded internal dialogue (§6.5).
- **P9 — Session-scoped identity, declared links.** Participant and informant identity belong to the session, not to the graph. Data crosses session boundaries only through declared, immutable, acyclic links and declared exports (§3.7).
- **P10 — Portable core, declared extensions.** The mandated expression library MUST behave identically in every conforming implementation. Operations beyond it MAY be supplied by an implementation, but only when declared, typed, pure, and deterministic (§5.6).

---

## 3. Core Model

### 3.1 Session

A **session** is one execution of a graph for one participant. A session MUST have:

| Field | Type | Description |
|---|---|---|
| `session_id` | string | Unique identifier for the execution. |
| `graph_ref` | string | Graph id and version being executed. |
| `case_id` | string | Groups sessions that concern the same participant or episode (§3.7). |
| `role` | string | Informant or rater identity: `participant`, `caregiver`, `clinician`, or a domain label such as `child` (§3.7). |
| `links` | map | Named, immutable references to other sessions in the same case (§3.7). |
| `export` | object | Allowlist of state slots readable by linked sessions (§3.7). |
| `seed` | integer | Seed governing all randomized decisions (§3.4). |
| `state` | State | Mutable scoped variables (§3.2). |
| `context` | Context | Read-only inline data attached to the conversation (§3.6). |
| `data_sources` | map | Resolved data-source bindings and credentials (§3.5). |
| `transcript` | list | Ordered record of user-facing turns. |
| `status` | enum | `active`, `completed`, `abandoned`, `escalated`, `limit_exceeded`. |

A session MAY be paused and resumed. On resume, `seed`, `state`, and `context` MUST be preserved.

`case_id`, `role`, `links`, and `export` are REQUIRED only for sessions participating in a multi-session case. A standalone session MAY omit them. Sessions are independent executions: one session MUST NOT await, sequence, retry, or terminate another. Case-level orchestration is out of scope (§1).

**Session references.** The scalar session fields `seed`, `id`, `case_id`, and `role` are readable in expressions as `session.<field>` (§5.3). They are read-only: no node MAY write one (VR-33). The non-scalar fields — `state`, `context`, `links`, `export`, `data_sources`, `transcript`, `status` — MUST NOT be referenced this way; each is reachable through its own mechanism.

### 3.2 State

**State** is the mutable, typed variable store shared across nodes. Variables MUST be declared in the graph's `state_schema`. Each variable declares:

```yaml
state_schema:
  age:
    type: integer
    scope: session
    mutable: true
  consent:
    type: boolean
    scope: session
    default: false
  turn_intent:
    type: enum
    values: [support, challenge, probe]
    scope: turn
    mutable: true
```

**Scopes**

| Scope | Lifetime | Typical use |
|---|---|---|
| `global` | Across all sessions (tenant/config level) | Locale, feature flags, configuration. |
| `session` | Entire session | Answers, scores, flags, profile. |
| `turn` | A single turn; cleared at turn boundary | Intermediate intent, per-turn scratch. |

**Types**: `string`, `integer`, `number`, `boolean`, `date`, `enum`, `list<T>`, `object`, `any`.

**Rules**

- A node MUST NOT read or write a state variable absent from `state_schema` (VR-7).
- Every write MUST be declared in the node's `effects.write`.
- Writes to `global` scope MUST be explicitly permitted by graph policy; otherwise they are rejected.
- `turn`-scoped variables MUST be reset when a turn ends.
- State is mutable and written; it is not a cache for derived values. A value that is a pure function of other state SHOULD be declared as a view (§3.8) rather than maintained by `set`. A deliberate point-in-time snapshot is legitimately state and MUST NOT be confused with a view.

Templates (for `prompt`/`message` fields) use `{{ state.<path> }}` or `{{ views.<name> }}` interpolation. A runtime MUST treat missing references as an error unless a default is declared.

### 3.3 Turn

A **turn** is a unit of user-facing exchange. A turn:

- **Begins** when execution reaches a node with `await: user` and emits its user-facing output.
- **Ends** when the user response is accepted (validated) or the node's timeout fires.
- **Records** the prompt, the raw response, the validated value, and any silent node effects that occurred between the previous and current turn.

Silent nodes (`await: none`) between turns MUST NOT create new turns. Multiple silent nodes MAY execute within a single turn's boundary.

### 3.4 Seed and Determinism

All randomized decisions are governed by `session.seed`. Given the same graph, seed, and inputs, the sequence of `sample` outcomes MUST be identical.

- A `sample` node MUST derive its randomness deterministically from `seed` and its own node id.
- The realized choice MUST be written to state so a run can be replayed or audited.
- To **fix** a random sample, a caller MUST either pin `seed` or pre-populate the sample node's recorded-choice slot.
- The seed is readable in expressions as `session.seed` (§3.1, §5.3), so a graph can derive its own randomized assignment rather than receiving one as Context. Functions that consume it (`shuffle`, §5.5) remain pure functions of their arguments.
- Model outputs are explicitly **out of scope** for determinism guarantees, but their state effects MUST still be declared.

### 3.5 Data Sources

A **Data Source** is a declared, read-only connection to an external system (database, API, warehouse, vector store) that exposes a set of typed **attributes**. Data sources are not data themselves; they are handles queried by `fetch` nodes (§4.6). Credentials MUST be referenced, never embedded.

```yaml
data_sources:
  crm:
    kind: postgres
    endpoint: ${env.CRM_DSN}
    credentials_ref: secret://crm/readonly
    attributes:
      account_tier: { type: enum, values: [free, pro, enterprise] }
      tenure_days:  { type: integer }
    access: read_only
    cache_ttl_s: 300
```

| Field | Required | Description |
|---|---|---|
| `id` | yes | Namespaced identifier referenced by `fetch.source_ref`. |
| `kind` | yes | Connector type (`postgres`, `http`, `vector`, `session_store`, or a namespaced custom kind). |
| `endpoint` | yes | Connection target (DSN/URL); MAY itself be a secret reference. |
| `credentials_ref` | conditional | Reference to a secret in the runtime secret store; REQUIRED for authenticated sources. |
| `attributes` | yes | Typed attribute schema exposed by the source. |
| `access` | no | `read_only` (default) or `read_write`. |
| `cache_ttl_s` | no | Default cache lifetime for fetched results. |

**Rules**

- Credentials MUST NOT appear inline; they MUST be referenced via `credentials_ref` and resolved by the runtime secret store (VR-19).
- Every exposed attribute MUST be typed; `fetch` results MUST conform to these types (VR-17).
- Data sources MUST be declared at graph or environment level; a session binds to them at start.
- `read_write` sources MUST be explicitly permitted by graph policy; the default is `read_only`.
- Data sources MUST NOT be used to write to State implicitly; only `fetch` nodes materialize attributes into State.
- A `session_store` source resolves against the current session's declared `links` (§3.7). It MUST be `access: read_only`; `read_write` is invalid for this kind. It exposes only attributes that the linked session has exported.

### 3.6 Context (Inline Data)

**Context** is read-only data attached directly to a conversation rather than fetched from an external system. It is a set of named attribute sets supplied either as graph defaults or at session start. Nodes reference it as `context.<path>`. Context is **immutable** for the lifetime of a session.

```yaml
context_schema:
  participant: { type: object, scope: session, mutable: false }
context:
  participant:
    id: "P-1042"
    locale: "en-GB"
    cohort: "2026-q1"
  documents:
    - { id: doc1, text: "..." }
```

**Rules**

- Context MUST be serializable and is versioned with the graph or session that supplies it.
- Session-supplied Context MAY override graph-default Context; the merge is shallow by top-level key unless a deeper merge is declared.
- Context is read-only: no node or `set` MAY write to `context.*` (VR-18).
- Every `context.*` reference MUST resolve in the supplied Context or its optional `context_schema`.
- Context is distinct from State: State is mutable and scoped; Context is immutable and supplied.
- Context size SHOULD be bounded by graph `limits` to avoid unbounded context-window growth.

### 3.7 Session Links and Cross-Session Data

A multi-session **case** groups sessions that concern the same participant or episode, typically one session per informant plus a synthesis session. Sessions exchange data only through declared links and declared exports. A graph MUST NOT address two informants within a single flow.

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
- Cross-session reads MUST use a `session_store` data source (§3.5) and a `fetch` node with `session_ref` (§4.6). They MUST NOT write to the linked session, block the conversation, or create a turn boundary.
- A session that is still `active` MAY be read. The reader MUST treat absent exports as `null` and MUST apply an explicit missing-value policy (see `decide.on_missing`, §4.4).
- A session MUST NOT await, sequence, retry, or terminate another session (P9). Only exported data crosses the boundary.

### 3.8 Derived Views

A **view** is a named, read-only, deterministically recomputed value derived from State, Context, or other views. A view is **not** state: it cannot be written, it is not declared in `state_schema`, and it is not subject to scope or mutation.

```yaml
views:
  mdd_responses:
    type: list<enum>
    source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]

  mdd_score:
    type: integer
    expr: "count(views.mdd_responses, item == 'yes')"

  mdd_flagged:
    type: boolean
    expr: "views.mdd_score >= 3"

  intake_summary:
    type: string
    expr: "join([state.q_name, state.q_reason, state.q_goal], ' | ')"
```

| Field | Required | Description |
|---|---|---|
| `type` | yes | The view's value type: `list<T>`, `object`, or a scalar type (§3.2). |
| `source` | conditional | A list of references (`state.*`, `context.*`, or `views.*`); sugar for a list literal. REQUIRED if `expr` is absent. |
| `expr` | conditional | An expression (§5.3) evaluating to `type`. REQUIRED if `source` is absent. |

Exactly one of `source` or `expr` MUST be present.

**Rules**

- Views are **read-only**. No node MAY write a view; `set`, `ask`, `fetch`, and `tool` MUST NOT target one. A view name MUST NOT collide with a `state_schema` variable or a `context` path.
- A view's value is a **pure function of current State, Context, and other views**. A runtime MAY cache or recompute at will; the observable value MUST be identical. Caching and invalidation are not specified.
- The view dependency graph MUST be acyclic, and every reference MUST resolve (VR-29).
- Views MUST NOT perform I/O, call models, read wall-clock time, or consume randomness. Time and locale come from `context` or State, never from the runtime clock.
- A node that reads a view in its own config MUST declare it in `effects.read` by its bare name, e.g. `read: [mdd_score]` (VR-30). The validator expands view dependencies to the underlying state variables for VR-7. Guard references resolve without a separate declaration, as with `state.*`.
- Views are referenced as `views.<name>` wherever `state.*` is legal: expressions, edge guards, `loop.exit_when`, `decide` operands, `foreach.over`, `ask.options_ref`, and templates.
- A graph-level `views` block is OPTIONAL; a graph with none behaves exactly as in v0.3.0.

**Why a separate namespace.** State is mutable and written; views are derived and read-only. Keeping them apart makes write-validation and replay reasoning direct, and prevents a derived value from being silently overwritten.

**Scope of the feature.** Views exist so that state can be *projected*: assembled from many slots, aggregated, and read back as one value. That a user interface, an export, or a message can consume a view follows from its being an ordinary readable value; it is not a design goal. This specification defines no dashboard, layout, or presentation format.

**Heterogeneous sections.** When a section's items differ in type, build a list of objects with `expr`:

```yaml
views:
  intake_section:
    type: list<object>
    expr: "[{key: 'name', value: state.q_name}, {key: 'reason', value: state.q_reason}]"
```

---

## 4. Node Taxonomy

Nodes are grouped into six families. Every node declares the following common attributes:

| Attribute | Required | Description |
|---|---|---|
| `id` | yes | Unique within the graph. |
| `type` | yes | One of the node types defined below. |
| `label` | no | Human-readable name. |
| `config` | per type | Type-specific configuration. |
| `effects.read` | no | State/context references read. |
| `effects.write` | no | State variables written (Context is read-only). |
| `await` | derived | `user`, `external`, or `none`. |
| `ports` | derived | Named attachment points for edges. |

**Await semantics**

| Value | Meaning |
|---|---|
| `user` | Blocks for end-user input; defines a turn boundary. |
| `external` | Blocks for an external event (human handoff, long-running tool). |
| `none` | Silent; does not block the conversation. |

Only one path MAY be in `user` await at a time (P5, VR-11).

### 4.1 Boundary Family

#### `start`

- **Purpose**: Single entry point of a graph.
- **Ports**: `out` (exactly one).
- **Config**: optional `init` map of state assignments applied before the first node.
- **Effects**: writes declared `init` targets.
- **Await**: `none`.
- **Requirements**: Exactly one `start` MUST exist per graph; it MUST have no inbound edges (VR-2).

#### `end`

- **Purpose**: Terminal node; concludes a path.
- **Ports**: `in` (one or more); no outbound.
- **Config**:
  - `outcome` (REQUIRED): `completed`, `abandoned`, `escalated`, `limit_exceeded`, or a namespaced custom value.
  - `reason` (OPTIONAL): template string.
- **Effects**: none.
- **Await**: `none`.
- **Requirements**: Every terminal path MUST end at an `end` node with an `outcome` (VR-15).

### 4.2 Interaction Family

#### `ask`

- **Purpose**: Emit a prompt, await a user response, validate it, and store it.
- **Ports**: `in`; `out` (success); `error` (validation failure); `timeout` (optional).
- **Config**:

| Field | Required | Description |
|---|---|---|
| `prompt` | yes | Template shown to the user. |
| `slot` | yes | State variable receiving the validated value. |
| `input_type` | yes | One of the enum below. |
| `response_schema` | conditional | Validation schema; REQUIRED for `custom`. |
| `options_ref` | no | List-typed state or view reference supplying the option set at turn time; alternative to static `options`. |
| `required` | no | Whether an empty response is acceptable (default `true`). |
| `retries` | no | Max re-prompts on invalid input (default `0`). |
| `timeout_s` | no | Time to wait before firing the `timeout` edge. |

- **`input_type` enum**:

| `input_type` | Value produced | Config |
|---|---|---|
| `text` | string | `multiline: bool`, `max_length` |
| `integer` | integer | `min`, `max` |
| `number` | number | `min`, `max`, `precision` |
| `slider` | number | `min`, `max`, `step`, `labels` |
| `single_select` | enum value | `options` |
| `multi_select` | list | `options`, `min_selections`, `max_selections` |
| `boolean` | boolean | `true_label`, `false_label` |
| `scale` | integer | `min`, `max`, `step`, `labels` (Likert) |
| `date` | date | `min`, `max` |
| `custom` | any | `response_schema` |

For `single_select` and `multi_select`, the option set MUST be supplied by exactly one of `options` (static) or `options_ref` (resolved from state or a view). When `options_ref` is used, the response MUST validate against the resolved set.

- **Effects**: writes `slot`.
- **Await**: `user`.
- **Requirements**:
  - The node MUST perform exactly one user exchange (P8): one prompt and one accepted response. `retries` re-prompts the **same** prompt and does not constitute a new exchange.
  - The response MUST be validated against `input_type` and `response_schema` before writing `slot`.
  - On validation failure, the runtime MUST either re-prompt (up to `retries`) or follow the `error` edge.
  - The node MUST define a turn boundary and MUST record the raw and validated response in the transcript.
  - `multi_select` MUST enforce `min_selections`/`max_selections` when declared.

#### `say`

- **Purpose**: Emit a one-way message; no response expected.
- **Ports**: `in`; `out`.
- **Config**: `message` (template), `channel` (optional).
- **Effects**: none (reads allowed).
- **Await**: `none`.
- **Requirements**: A `say` node MUST NOT create a turn boundary.

### 4.3 Cognition Family

Model-backed nodes MUST declare `effects.write` and MUST record enough provenance (model id, prompt hash, output) for auditing.

#### `llm`

- **Purpose**: Invoke a language model, producing a user-facing turn or a silent output.
- **Ports**: `in`; `out`; `error`; `timeout`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `model` | yes | Model identifier. |
| `mode` | yes | `conversational` or `silent`. |
| `messages` | conditional | Ordered list of `{role, template}`; `role` ∈ `system`, `user`, `assistant`. Exactly one of `messages` or `prompt_template` MUST be declared. |
| `prompt_template` | conditional | Sugar for a single `user` message: `prompt_template: T` is exactly `messages: [{role: user, template: T}]`. |
| `history` | no | Conversation scope for the assembled prompt: `all` (default), `none`, `{last: N}`, or `{from: [...]}` (§7.1). |
| `output_schema` | no | If set, output MUST validate against it. |
| `slot` | yes | State variable receiving the validated reply (`conversational`) or the model output (`silent`). |
| `message_slot` | no | `string` state variable receiving the message emitted to the user. `conversational` only; MUST differ from `slot`. |
| `tools` | no | Tool references available to the model. |
| `temperature`, `max_tokens` | no | Generation parameters. |
| `timeout_s` | no | Timeout before firing the `timeout` edge. |

- **Effects**: writes `slot` and, when declared, `message_slot`.
- **Await**: `user` when `mode: conversational`; `none` when `mode: silent`.
- **Requirements**:
  - In `conversational` mode the node MUST emit exactly one user-facing message and await exactly one reply (P8), writing the validated reply to `slot`. It MUST NOT conduct further exchanges internally and MUST NOT hold an internal turn budget.
  - In `silent` mode the node MUST NOT produce a user-facing turn and MUST NOT block.
  - When `output_schema` is present, output MUST validate before writing; on failure the `error` edge MUST be taken.
  - `messages` MUST contain at least one entry, and at least one entry MUST have role `user` or `assistant`. Every `system` message MUST precede every non-system message (VR-34).
  - The prompt is assembled in a fixed order: leading `system` messages, then the `history` block (§7.1), then the remaining messages in declaration order. This is why `system` messages MUST be leading.
  - When `message_slot` is declared, the node MUST write the exact text it emitted, so a later node can inspect it. `slot` continues to receive the participant's reply.
  - The runtime MUST record model id, prompt hash, and output for every invocation.

#### `analyze`

- **Purpose**: Silent background inference (e.g., risk detection) that writes scores/flags without interrupting the conversation.
- **Ports**: `in`; `out`; `error`.
- **Config**:
  - `backend` (REQUIRED): model or service reference.
  - `inputs` (REQUIRED): state/context references to analyze.
  - `output_schema` / `slot` (REQUIRED): where results are written.
  - `default` (OPTIONAL): value used if inference fails or times out.
- **Effects**: writes `slot` (and any declared derived flags).
- **Await**: `none`.
- **Requirements**:
  - An `analyze` node MUST NOT block or create a turn boundary.
  - It MUST write either a result or a declared `default` to `slot`.
  - It MUST be usable inside a `fork` silent stream (§6.4).
  - Failures MUST NOT terminate the conversation unless the `error` edge is taken.

#### `classify`

- **Purpose**: Assign one label from a fixed taxonomy, optionally with confidence. Distinct from `llm` because it MAY use non-LLM backends.
- **Ports**: `in`; `out`; `error`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `backend` | yes | `llm`, `bert`, `predictive`, `rules`, or a namespaced model ref. |
| `model` | yes | Model/service identifier. |
| `inputs` | yes | State/context references to classify (e.g., an `ask` response). |
| `labels` | yes | Closed label set. |
| `slot` | yes | State variable receiving the label. |
| `confidence_slot` | no | Receives a confidence score. |
| `threshold` | no | Below this confidence, take the `error` edge. |

- **Effects**: writes `slot` and optional `confidence_slot`.
- **Await**: `none`.
- **Requirements**:
  - Output MUST be exactly one value from `labels`; otherwise the `error` edge MUST be taken.
  - Non-LLM backends (`bert`, `predictive`, `rules`) MUST be supported.
  - Stochastic backends MUST honor the session seed where the backend allows it.
  - A `classify` node SHOULD be paired with a `branch` node for routing.

### 4.4 Control Family

#### `branch`

- **Purpose**: Deterministic routing based on state.
- **Ports**: `in`; multiple `out` via `conditional` edges plus an optional `default` edge.
- **Config**: optional `strategy` (default `first_match`).
- **Effects**: none.
- **Await**: `none`.
- **Requirements**:
  - Outgoing edges MUST be `conditional` and/or `default`; at least one MUST exist.
  - A `branch` MUST have a `default` edge or guards that are provably exhaustive (VR-9).
  - Exactly one outgoing edge MUST be taken per evaluation.

#### `decide`

- **Purpose**: Evaluate a declared logical operation over typed state operands and route on the boolean result.
- **Ports**: `in`; `true`; `false`; `unknown`; `error`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `operands` | yes | `{ refs: [...] }` (explicit list) or `{ list_ref: state.x }` (list-typed variable or view). |
| `test` | conditional | REQUIRED for non-boolean operands; MUST be omitted for boolean operands. |
| `combine` | yes | `any`, `all`, `none`, or `n_of_k`. |
| `k` | conditional | REQUIRED for `n_of_k`; MUST be a positive integer. |
| `on_missing` | no | `unknown` (default), `skip`, or `fail`. |
| `result_slot` | no | Receives the boolean result for audit or later reuse. |

- **Operands**: `refs` entries are a state or view reference, or an object `{ ref: state.x, negate: true }`; `negate` inverts that operand's test result. All operands MUST share a type, and `list_ref` MUST resolve to a `list<T>`.
- **`test`** compares each operand to a literal, set, or range. It MUST NOT compare operands to each other; pairwise comparison belongs in a `branch` guard (§5.3).

| Operand type | Legal `test` | Required parameters |
|---|---|---|
| `boolean` | none; use `negate` for inversion | — |
| `integer`, `number` | `gt`, `gte`, `lt`, `lte`, `eq`, `neq` | `value` |
| `integer`, `number` | `within` | `min`, `max` |
| `date` | `before`, `after` | `value` |
| `date` | `within` | `from`, `to` |
| `enum` | `eq` | `value` |
| `enum` | `in` | `values` |
| `list<T>` | `predicate` using the `item.` binding (§5.3) | — |

- **`combine`**: `any` is true iff at least one operand test is true; `all` iff every test is true; `none` iff no test is true; `n_of_k` iff at least `k` tests are true. For an empty operand set, `any` is `false`, `all` and `none` are `true`, and `n_of_k` is `false`.
- **`on_missing`**: `unknown` takes the `unknown` port if any operand or element is `null`; `skip` excludes `null` operands and evaluates `combine` over the remainder; `fail` takes the `error` port. The default `unknown` exists so a missing rating is a routable outcome rather than a silent `false` (§5.3).
- **Effects**: MAY write `result_slot`.
- **Await**: `none`.
- **Requirements**:
  - Exactly one of `true`, `false`, or `unknown` MUST be taken per evaluation (VR-20).
  - A `decide` MUST be a pure function of state: no I/O, no model calls, no Context writes.
  - `decide` routes only on the boolean outcome. It is not a multi-way switch; routing to more than one functional target per outcome is a `branch` concern.
  - `n_of_k` with `k` greater than the possible operand count MUST fail validation for `refs` operands; for `list_ref` it evaluates to `false`.

#### `sample`

- **Purpose**: Randomly select one outgoing path, e.g., to deliver a supportive vs. challenging variant.
- **Ports**: `in`; multiple `out`.
- **Config**:

```yaml
type: sample
config:
  distribution: weighted   # weighted | uniform
  choices:
    - to: llm_supportive
      weight: 1
    - to: llm_challenging
      weight: 1
  record_slot: variant_chosen   # REQUIRED
```

- **Effects**: writes `record_slot` with the realized choice (target node id).
- **Await**: `none`.
- **Requirements**:
  - Weights MUST be positive; for `uniform`, weights MUST be omitted or equal (VR-12).
  - Selection MUST be deterministic given `seed` and node id (§3.4).
  - The realized choice MUST be written to `record_slot`.
  - If `record_slot` is pre-populated, that value MUST be used instead of sampling (replay/pinning).
  - Outbound transitions MUST be declared by `config.choices`; a `sample` node MUST NOT declare separate `next` edges.
  - Weights live in the `sample` node's config; edges MUST NOT carry weight attributes.

#### `fork`

- **Purpose**: Start parallel execution streams.
- **Ports**: `in`; multiple `out` (one per stream).
- **Config**: `streams` (REQUIRED): list of head node ids.
- **Effects**: none.
- **Await**: `none`.
- **Requirements**:
  - All forked streams MUST be **entirely non-blocking** (only `await: none` nodes) until they reach the matching `join` (VR-11).
  - A `fork` MUST NOT create more than one concurrently awaiting path (P5).
  - A `fork` MUST have a matching `join` reachable from every stream.
  - Stream heads MUST be distinct and MUST NOT be `join` or `end` nodes.

#### `join`

- **Purpose**: Synchronize forked streams.
- **Ports**: multiple `in`; `out`; `timeout` (optional).
- **Config**:
  - `mode` (REQUIRED): `all`, `any`, or `n_of_k`.
  - `k` (REQUIRED for `n_of_k`).
  - `timeout_s` (OPTIONAL).
- **Effects**: MAY merge stream outputs via declared writes.
- **Await**: `none`.
- **Requirements**:
  - The `join` MUST wait according to `mode` before proceeding.
  - On timeout it MUST take the `timeout` edge if present; otherwise it MUST take the `error` edge.
  - A `join` MUST be paired with exactly one `fork` (VR-11).

#### `loop`

- **Purpose**: Bounded iteration (e.g., repeated interview follow-ups).
- **Ports**: `in`; `body` (enter loop body); `exit`; `error` (optional).
- **Config**:
  - `body_entry` (REQUIRED): node id entering the body.
  - `exit_when` (REQUIRED): guard expression evaluated after each iteration.
  - `max_iterations` (REQUIRED): positive integer.
  - `counter_slot` (OPTIONAL): state variable tracking iterations.
- **Effects**: MAY write `counter_slot`.
- **Await**: inherited from the body (may include `user` nodes).
- **Requirements**:
  - The node MUST increment its iteration count each pass and MUST take `exit` when `exit_when` holds or `max_iterations` is reached.
  - On reaching `max_iterations` without `exit_when`, it MUST take `exit` (or the `error` edge if declared).
  - Every cycle in the graph MUST pass through a `loop` node or a guarded edge with a counter (VR-10).

#### `foreach`

- **Purpose**: Iterate an ordered, filtered collection, invoking a body once per element (e.g., runtime-ordered dispatch of reusable subgraphs).
- **Ports**: `in`; `body` (only when `body_entry` is used); `exit`; `error`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `over` | yes | `{ list_ref: state.x }` (list-typed variable or view) or `{ registry: name }` (registry projection, §6.1). |
| `filter` | no | Scoped expression over `item.`; elements failing it are skipped. |
| `order_by` | no | Scoped expression over `item.`; ascending unless `descending: true`. |
| `descending` | no | Boolean; default `false`. |
| `tiebreak` | no | Scoped expression over `item.`; together with `order_by` MUST yield a total order. |
| `body_graph` | conditional | Registry form resolved per element; REQUIRED if `body_entry` is absent. |
| `body_entry` | conditional | Static node id; REQUIRED if `body_graph` is absent. |
| `item_slot` | yes | State variable receiving the current element. |
| `index_slot` | no | State variable receiving the zero-based iteration index. |
| `max_iterations` | yes | Positive integer. |

- **Effects**: writes `item_slot`; MAY write `index_slot`.
- **Await**: inherited from the body (MAY be `user`).
- **Requirements**:
  - The iteration set MUST be snapshotted at entry; mutating the source collection inside the body MUST NOT change the set or the order of the current run (§3.4).
  - Ordering MUST be total. If `order_by` can produce ties, `tiebreak` MUST be declared; otherwise stable input order MUST be used.
  - `max_iterations` MUST bound the loop; on reaching it the `exit` edge MUST be taken (or `error` if declared).
  - An empty iteration set MUST take `exit` immediately.
  - When `body_graph` is used the `body` port MUST have no edges; when `body_entry` is used the invoked node MUST be reachable only from the `body` port.
  - A `foreach` whose body can await `user` MUST NOT appear in a forked silent stream (VR-11).

#### `call`

- **Purpose**: Invoke a subgraph as a reusable unit (e.g., a shared consent flow).
- **Ports**: `in`; `out`; `error`.
- **Config**:
  - `graph_ref` (REQUIRED): subgraph id and version, or a registry reference `{ registry: name, key: <scoped expression> }` (§6.1).
  - `state_isolation` (REQUIRED): `isolated` or `shared`.
  - `input_map` (OPTIONAL): parent state → child inputs.
  - `output_map` (OPTIONAL): child outputs → parent state.
- **Effects**: writes declared `output_map` targets.
- **Await**: inherited from the subgraph (MAY be `user`).
- **Requirements**:
  - The referenced subgraph MUST exist and be version-pinned (VR-13).
  - A registry-based `graph_ref` MUST resolve to a key in the named registry; resolution failure MUST take the `error` edge.
  - In `isolated` mode the child MUST NOT read or write parent state except through `input_map`/`output_map`.
  - Subgraph errors MUST propagate to the caller's `error` edge.
  - A subgraph `end` MUST conclude the subgraph and return control through this node's `out` port; it MUST NOT terminate the parent session. The terminal `outcome` is exposed as the reserved output `outcome` and MAY be bound through `output_map` (§6.2, VR-36).
  - A `call` that can await `user` MUST NOT be placed in a forked silent stream.

### 4.5 State Family

#### `set`

- **Purpose**: Deterministic state mutation (assignment or expression evaluation), including assembly of collections and objects from existing state.
- **Ports**: `in`; `out`.
- **Config**:
  - `assignments` (REQUIRED): list of `{slot, expression}` entries.
- **Effects**: writes each `slot`.
- **Await**: `none`.
- **Requirements**:
  - A `set` node MUST be pure with respect to the external world and MUST NOT perform I/O.
  - Expressions MUST be evaluated using the guard/expression grammar (§5.3).
  - An expression MAY evaluate to any declared type, including `list<T>` and `object`. Values are immutable; constructors (`append`, `concat`, `put`, `merge`, …) return new values (§5.5).
  - All assignments in one `set` node MUST be evaluated **simultaneously** against the state at node entry and then applied atomically. No assignment MAY observe another assignment's write in the same node (VR-28).
  - Self-reference is legal and reads the pre-node value: `append(state.log, state.current_item)` appends to the previous list.
  - All written slots MUST be declared in `effects.write`.
  - A `set` MUST NOT target `context.*` (VR-18) or a view (§3.8).

A `set` inside a `foreach` body that reads and appends its own list slot is the canonical accumulator pattern (§6.5.3). It is legal because each iteration is a separate `set` execution and therefore sees the previous iteration's write.

### 4.6 External Family

#### `fetch`

- **Purpose**: Query a declared Data Source and materialize its typed attributes into State.
- **Ports**: `in`; `out`; `error`; `timeout`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `source_ref` | yes | Data source id (§3.5). |
| `session_ref` | conditional | REQUIRED when `source_ref` is a `session_store`; MUST name a key in the current session's `links` (§3.7). |
| `attributes` | yes | Attributes to retrieve; MUST exist in the source schema. |
| `filters` | no | Parameterized filters; MAY reference `state.*`/`context.*`. |
| `output_map` | yes | Map of source attribute → state slot. |
| `cache_ttl_s` | no | Override the source's default cache TTL. |
| `timeout_s` | no | Timeout before firing the `timeout` edge. |

- **Effects**: writes each `output_map` target.
- **Await**: `none`.
- **Requirements**:
  - `source_ref` MUST reference a declared data source (VR-17).
  - Returned values MUST conform to the source attribute types; on mismatch the `error` edge MUST be taken.
  - Credentials MUST be resolved by the runtime secret store and MUST NOT appear in config or traces (VR-19).
  - A `fetch` MUST NOT write to Context, which is read-only (VR-18).
  - Failures and timeouts MUST route to `error`/`timeout`, not terminate the session.
  - A `fetch` against a `session_store` MUST name a declared link and MUST request only attributes exported by that linked session (VR-21). It MUST NOT write to the linked session.

#### `tool`

- **Purpose**: Perform a non-LLM external side effect (lookup, scheduling, record update).
- **Ports**: `in`; `out`; `error`; `timeout`.
- **Config**:
  - `tool_ref` (REQUIRED): tool/service identifier.
  - `input_map` (REQUIRED).
  - `output_slot` (OPTIONAL): where the result is stored.
  - `idempotency_key` (OPTIONAL).
  - `timeout_s` (OPTIONAL), `retries` (OPTIONAL).
- **Effects**: writes `output_slot`; MUST declare external side effects.
- **Await**: `none` (the conversation is not blocked), though the node MAY run asynchronously.
- **Requirements**:
  - Failures MUST be routed to `error`/`timeout`, not terminate the session.
  - Retries MUST respect `idempotency_key` when declared.
  - A `tool` node MUST NOT be treated as a turn boundary.

#### `handoff`

- **Purpose**: Pause the graph and escalate to a human or external queue.
- **Ports**: `in`; `out` (resumed); `timeout` (optional).
- **Config**:
  - `target` (REQUIRED): queue or agent identifier.
  - `reason` (REQUIRED): template string.
  - `timeout_s` (OPTIONAL).
  - `resume` (REQUIRED): `continue` (return control to graph) or `terminate`.
- **Effects**: MAY write handoff metadata to state.
- **Await**: `external`.
- **Requirements**:
  - A `handoff` MUST emit a handoff event with `target` and `reason`.
  - On `resume: terminate`, the graph MUST end with outcome `escalated`.
  - On timeout, the `timeout` edge MUST be taken if present; otherwise the graph MUST end with outcome `escalated`.

---

## 5. Edges and Control Flow

### 5.1 Edge Model

An edge is a directed transition:

```yaml
- id: e_q2_adult
  from: q2
  to: q3
  type: conditional      # next | conditional | default | error | timeout
  port: out              # source port; default `out` when the node has one
  when: "state.age >= 18"
  priority: 10           # lower number = higher precedence
```

| `type` | Meaning |
|---|---|
| `next` | Unconditional successor; used where a node has a single outbound. |
| `conditional` | Taken when `when` evaluates truthy. |
| `default` | Fallback when no `conditional` matches. |
| `error` | Taken on validation/model/tool failure. |
| `timeout` | Taken when the node's timeout fires. |

**Rules**

- A node with a single outbound port MUST use `next` (or a single `conditional` on control nodes).
- `sample` nodes declare outbound transitions via `config.choices`; no edge objects are required for them.
- Multi-port control nodes (`loop`, `foreach`, `decide`) MUST specify the source `port` on each edge. `decide` ports are `true`, `false`, and `unknown`.
- For `loop`, the `body` edge target MUST equal `body_entry`.
- Edges MUST connect an existing outbound port to an existing node (VR-4, VR-14).
- Edges MUST NOT carry weight attributes; weights belong to `sample` config (§4.4).

### 5.2 Edge Resolution Order

When multiple outbound edges could fire, the runtime MUST resolve in this order:

1. `error` (on failure) — highest precedence.
2. `timeout` (on timeout).
3. `conditional` in ascending `priority` (lower number first); ties broken by edge declaration order.
4. `default`.

Exactly one edge MUST be taken. If none applies and no `default` exists, the graph MUST fail validation (VR-9).

**Port-scoped resolution.** A multi-port node first selects its active port — `decide` by outcome, `loop`/`foreach` by `body` or `exit` — then resolves edges within that port using the order above. If the active port has no edge, the runtime MUST take `error` if present, otherwise the graph's `on_error`; a port that can be active with no edge and no fallback MUST fail validation.

### 5.3 Guard and Expression Grammar

Guards and `set` expressions use the same grammar:

```ebnf
guard          = or_expr ;
or_expr        = and_expr , { "or" , and_expr } ;
and_expr       = not_expr , { "and" , not_expr } ;
not_expr       = [ "not" ] , primary ;
primary        = comparison | expr | "(" , guard , ")" ;
comparison     = expr , comp_op , expr ;
comp_op        = "==" | "!=" | "<" | "<=" | ">" | ">=" | "in" | "matches" ;
expr           = arith ;
arith          = term , { ( "+" | "-" ) , term } ;
term           = factor , { ( "*" | "/" ) , factor } ;
factor         = number | string | boolean | "null"
               | ref
               | local_ref
               | call
               | list_literal
               | object_literal
               | "(" , expr , ")" ;
ref            = "state." , identifier , { "." , identifier }
               | "context." , identifier , { "." , identifier }
               | "views." , identifier
               | "session." , identifier ;
local_ref      = ( "item" | "index" ) , { "." , identifier } ;
call           = name , "(" , [ expr , { "," , expr } ] , ")" ;
name           = identifier , { "." , identifier } ;
list_literal   = "[" , [ expr , { "," , expr } ] , "]" ;
object_literal = "{" , [ obj_entry , { "," , obj_entry } ] , "}" ;
obj_entry      = ( identifier | string ) , ":" , expr ;
```

`call` resolves to one of three things, in this order: a core function (§5.5), a namespaced implementation extension (§5.6), or a named predicate declared in the graph's `predicates` registry. A call name MUST NOT begin with the reserved roots `state`, `context`, `views`, or `session`. Core and extension names are reserved; a predicate MUST NOT shadow them (VR-26).

`session.<field>` names a scalar session field (§3.1): `seed`, `id`, `case_id`, or `role`. It is a read-only reference, resolvable wherever a `state.*` reference is legal, and it MUST NOT be written (VR-33). Because the seed is an ordinary operand, functions that consume it stay pure (§5.5).

`local_ref` (`item`, `index`) is legal **only** in element-scoped positions: `decide.test.predicate`, `foreach.filter`/`order_by`/`tiebreak`/`body_graph`, and the element-scoped argument of `map`/`filter`/`find`/`count`/`any`/`all`/`none`/`sort`. In those positions `item` binds the current element and `index` its zero-based position in the source list. Their types are derived from the list argument. `item` and `index` MUST NOT appear in edge guards, `set` expressions, or `loop.exit_when`, which have no iteration binding (VR-27).

**Equality and membership.** `==` and `!=` are structural on lists and objects: element-wise, order-sensitive for lists, key-set-and-value for objects. `<`, `<=`, `>`, `>=` are defined only for `number`, `string`, `date`, and `enum` (declaration order); applying them to lists, objects, or booleans MUST fail validation. `in` tests membership for a list and key presence for an object.

**String matching.** `a matches b` is true when `b` occurs as a contiguous substring of `a`, comparing characters with ASCII case folding only (`A`–`Z` ↔ `a`–`z`); all other characters compare exactly. It is not a regular expression, it does not anchor, and it treats no character specially. Both operands MUST be `string` (VR-37); either operand `null` makes the result `false`, consistent with the null-comparison rule above. Folding is ASCII-only so the result does not depend on the host locale.

**Named predicates** are resolved from a graph-level `predicates` registry. A predicate MUST be declared before use; undeclared predicates MUST fail validation. A predicate is a typed function: it declares `args` and `returns`, and its body is an expression over `args[i]` and, where declared, `item`.

```yaml
predicates:
  risk_exceeds:
    args: [number]
    returns: boolean
    expr: "args[0] > state.risk_threshold"
```

- Guards and expressions MUST be side-effect free.
- A reference to an unset state variable MUST evaluate to `null`, and comparisons with `null` MUST be `false` (except `== null` / `!= null`). A reference to an undeclared `context.*` path MUST fail validation.
- An empty list or object literal takes its type from its assignment slot or enclosing parameter; with no contextual type it MUST fail validation (VR-28).

### 5.4 Cycles and Loop Guards

- Unguarded cycles are forbidden (VR-10).
- A cycle MUST pass through either a `loop` node or a `conditional` edge whose guard is proven to eventually terminate (e.g., bounded counter).
- The runtime MUST enforce `loop.max_iterations` and the graph-level `limits`.

### 5.5 Standard Function Library

These functions are the **conformance minimum**: every implementation MUST provide all of them, with the semantics given here. They are pure, deterministic, and total; behaviour on `null` and empty inputs is defined per row. Additional operations are permitted as declared extensions (§5.6), but MUST NOT alter these.

| Function | Signature | Semantics |
|---|---|---|
| `len` | `len(list\|string\|object) -> integer` | Number of elements, characters, or keys. `null` → `0`. |
| `count` | `count(list<T>) -> integer` | Number of **non-null** elements. |
| `count` | `count(list<T>, predicate) -> integer` | Number of elements satisfying the element-scoped predicate. |
| `sum` | `sum(list<number>) -> number` | Sum; empty → `0`; `null` elements skipped. |
| `avg` | `avg(list<number>) -> number \| null` | Mean of non-null elements; empty → `null`. |
| `min` / `max` | `min(list<T>) -> T \| null` | Minimum/maximum under the type's natural order; empty → `null`. |
| `any` / `all` / `none` | `any(list<T>, predicate) -> boolean` | Existential; universal; empty-or-none-satisfy. |
| `map` | `map(list<T>, selector) -> list<U>` | Element-wise projection; `item`/`index` bound. |
| `filter` | `filter(list<T>, predicate) -> list<T>` | Keep elements satisfying the predicate, preserving order. |
| `find` | `find(list<T>, predicate) -> T \| null` | First matching element, else `null`. |
| `append` / `prepend` | `append(list<T>, T) -> list<T>` | New list with one element at the end / start. |
| `concat` | `concat(list<T>, …) -> list<T>`; `concat(string, …) -> string` | Concatenate lists, or join strings. |
| `flatten` | `flatten(list<list<T>>) -> list<T>` | One level of flattening. |
| `distinct` | `distinct(list<T>) -> list<T>` | Deduplicate, keeping first-occurrence order. |
| `slice` | `slice(list<T>\|string, start, end?) -> same` | Half-open range; `end` defaults to length. |
| `range` | `range(end) -> list<integer>`; `range(start, end, step?) -> list<integer>` | Half-open integer range; `step` defaults to `1` and MUST be non-zero; negative steps count down. |
| `take` | `take(list<T>\|string, n) -> same` | The first `n` elements; `n` is clamped to `[0, len]`. |
| `chunks` | `chunks(list<T>, size) -> list<list<T>>` | Consecutive blocks of `size`; the last block MAY be short; `size` MUST be ≥ 1. |
| `shuffle` | `shuffle(list<T>, seed) -> list<T>` | Deterministic permutation. Same list and seed MUST yield the same order within an implementation. |
| `sort` | `sort(list<T>, key?, descending?) -> list<T>` | Stable sort under a total order; `key` is element-scoped. |
| `reverse` | `reverse(list<T>\|string) -> same` | Reverse order. |
| `merge` | `merge(object, …) -> object` | Shallow merge; later keys win. |
| `keys` / `values` | `keys(object) -> list<string>`; `values(object) -> list<any>` | Keys / values in insertion order. |
| `get` | `get(list<T>, integer, T?) -> T`; `get(object, string, any?) -> any` | Value at index/key, else `default` (`null`). |
| `put` | `put(object, key, value) -> object` | Immutable insert or overwrite. |
| `has` | `has(object, key) -> boolean` | Key presence. |
| `join` | `join(list<string>, separator) -> string` | Join; `null` elements become empty strings. |
| `if` | `if(boolean, T, T) -> T` | Conditional selection; both branches MUST be type-compatible. |
| `coalesce` | `coalesce(T?, …) -> T?` | First non-`null` argument. |
| `lookup` | `lookup(T, object, default?) -> U` | Map a value through a table; `default` (`null`) when absent. |
| `to_number` / `to_integer` / `to_string` / `to_boolean` | conversion | Explicit coercion; an invalid conversion yields `null`. |
| `abs` / `round` | `abs(number) -> number`; `round(number, digits?) -> number` | Numeric helpers. |
| `div` / `mod` | `div(integer, integer) -> integer`; `mod(integer, integer) -> integer` | Truncating division and remainder, with the semantics below. |
| `date_diff` | `date_diff(date, date, unit) -> integer` | Whole units between dates; `unit` ∈ `days`, `weeks`, `months`, `years`. |

**Permutation determinism.** `shuffle` is deterministic within an implementation: the same list and seed MUST yield the same order, and the result MUST NOT depend on the host RNG, host hash ordering, locale, or wall-clock time. This specification does not pin an algorithm, so two conforming implementations MAY produce different permutations for the same seed — the same treatment `sample` receives (§3.4). A graph that depends on a specific permutation MUST write the result to state, which is then the authoritative record for replay and analysis (§7.5).

**Typing rules**

- Every call MUST match exactly one signature. Arguments MUST be assignable to parameter types, and the result MUST be assignable to its target slot or enclosing parameter (VR-26).
- Element-scoped parameters (`map`'s selector, the predicate of `filter`/`find`/`count`/`any`/`all`/`none`, and `sort`'s key) bind `item` and `index`; their types are derived from the list argument. `item`/`index` MUST NOT appear in ordinary argument positions (VR-27).
- `sort`, `min`, and `max` require a declared total order for the element type: `number`, `string`, `date`, or `enum` declaration order. Any other element type MUST fail validation.

**Numeric semantics.** These are specified here rather than inherited from the host language, because host languages disagree and scoring correctness depends on them.

- `+`, `-`, `*` are exact on integers. Integer arithmetic MUST be exact or raise a runtime error; it MUST NOT wrap silently.
- `/` is true division returning `number`; it never truncates. Integer division and remainder are `div`/`mod`: truncating toward zero, with the remainder taking the sign of the dividend.
- `round` is half-away-from-zero. `to_integer` truncates toward zero. `to_number` parses decimal notation only.
- An implementation MAY satisfy exactness with arbitrary-precision or checked integers.

### 5.6 Expression Extensibility

The library in §5.5 is a **floor, not a ceiling**. Implementations SHOULD expose the operations their host language provides rather than waiting for this specification to enumerate them. Three tiers keep that from eroding reproducibility:

| Tier | Requirement | Semantics defined by |
|---|---|---|
| **Core** (§5.5) | **MUST** — every conforming implementation | This specification |
| **Extension** | **MAY** — any further pure operation the host provides | The implementation, via a declared manifest |
| **Prohibited** | **MUST NOT** — non-deterministic, effectful, or order-dependent | — |

**Rules for extensions**

- **Declared, not implicit.** An implementation advertises each extension as a manifest entry: qualified name, signature in the type system of §3.2, and whether the result may be `null`. A function absent from the manifest does not exist for validation purposes, so VR-26 is unchanged in strength.
- **Namespaced.** Extension names MUST be qualified (`py.`, `js.`, `rs.`, or a declared vendor prefix) and MUST NOT shadow a core function, another extension, or a graph predicate.
- **Functions only, never grammar.** Extensions add functions. Operators, literals, precedence, and evaluation order are owned by this specification, and an implementation MUST NOT extend the grammar of §5.3. Every graph MUST remain parseable by every conforming tool, including tools that cannot execute it.
- **Host types are mapped, not leaked.** Native operations SHOULD be exposed directly (`sorted`, `join`, `floor`, …) with signatures mapped into the types of §3.2. Where the host type system is richer, the implementation maps at the boundary or rejects; host-specific types MUST NOT appear in State, views, or exported values.
- **Determinism is a precondition.** Extensions MUST be pure, deterministic, and total (or documented as partial with a `null` result). They MUST NOT perform I/O, call models, read wall-clock time or randomness, or depend on host iteration order. Operations whose host semantics are order- or locale-dependent — map iteration order, collation, casing, float formatting — MUST be normalised before exposure, or not exposed.
- **Capability declaration.** A graph that calls an extension MUST declare it in the graph-level `capabilities` list (§6.1). Validation MUST fail when the target implementation's manifest does not advertise a compatible name and signature (VR-31). A graph that declares none is portable to every conforming implementation.

**Why not simply allow every native operation.** An unbounded surface cannot be validated or documented, and host engines diverge exactly where instruments are sensitive: integer division, remainder sign, float rounding, map ordering, collation. Declared, typed extensions give implementers the reach they need while leaving a reproducible core intact. Portability becomes a checkable property of the graph/manifest pair rather than an assumption.

Illustrative extensions — implementation-defined, not required: `py.split`, `py.trim`, `js.upper`, `py.contains`, `py.replace`, `py.format`, `py.group_by`, `py.zip`, `py.reduce`, `py.median`. A future revision MAY promote any of these into the core tier once two or more implementations need it with identical semantics.

---

## 6. Graph Composition

### 6.1 Graph Anatomy

```yaml
graph:
  id: onboarding_survey
  version: 0.1.0
  state_schema: { ... }
  data_sources: { ... }
  context_schema: { ... }
  context: { ... }
  predicates: { ... }
  graph_registry: { ... }
  views: { ... }
  capabilities: [ ... ]
  limits:
    max_turns: 50
    max_wall_clock_s: 1800
    max_tokens: 20000
  seed: 42
  entry: start
  on_error: end_abandoned
  nodes: [ ... ]
  edges: [ ... ]
  terminals: [ end_completed, end_abandoned ]
```

| Field | Required | Description |
|---|---|---|
| `id`, `version` | yes | Identity and version pin. |
| `state_schema` | yes | Declared mutable variables (§3.2). |
| `data_sources` | no | External data-source declarations (§3.5). |
| `context_schema` | no | Optional typing for inline Context (§3.6). |
| `context` | no | Graph-default inline Context data (§3.6). |
| `predicates` | no | Named predicate registry (§5.3). |
| `graph_registry` | no | Named, version-pinned subgraph registries for dynamic dispatch (§6.2). |
| `views` | no | Derived read-only values over State, Context, and other views (§3.8). |
| `capabilities` | conditional | Namespaced expression extensions the graph requires (§5.6). REQUIRED if any extension is called (VR-31). |
| `limits` | no | Turn/time/token budgets (§7.4). |
| `seed` | no | Default session seed (§3.4). |
| `entry` | yes | MUST reference the single `start` node. |
| `on_error` | no | Fallback node for unhandled errors. |
| `nodes`, `edges` | yes | Graph content. |
| `terminals` | yes | All `end` nodes. |

### 6.2 Subgraphs

A `call` node composes graphs. The subgraph MUST expose declared inputs/outputs, and the caller MUST map them via `input_map`/`output_map`. Subgraph state is isolated unless `state_isolation: shared`. Subgraphs MUST be version-pinned so graphs remain reproducible.

**Termination contract.** A subgraph `end` concludes **the subgraph** and returns control to the caller through the `call` node's `out` port; it MUST NOT terminate the parent session. Every subgraph therefore exposes one reserved output, `outcome`, whose value is the `outcome` of the `end` that terminated it, and a subgraph MUST NOT declare an output of that name. `output_map` MAY bind `outcome` like any other output, to a declared `enum` or `string` state variable (VR-36). A subgraph that ends via `on_error` takes the caller's `error` port instead, and `outcome` is not bound.

**Dynamic dispatch.** A `call.graph_ref` MAY be selected at runtime, but only through a declared `graph_registry`. Dynamic references resolve **only** through a registry; there is no runtime graph lookup by id. This keeps every possible dispatch target enumerable from the graph source (P4) and version-pinned (VR-13, VR-22).

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
```

A registry entry has `graph_ref`, `version`, and optional static `attributes`. Attributes are graph metadata, not state; `foreach` uses them for `filter`/`order_by`. A registry projection binds each element as `{ key, graph_ref, version, <attribute>... }`, so `item.key` and `item.onset_age` are valid element-scoped references. Resolution failure at runtime (key not present) takes the `error` edge; it is not a validation failure because the key is state-derived.

### 6.3 Branching vs. Forking

- **Branch** (`branch`/`sample`): exactly **one** path is taken; the others are never executed. Any of those paths MAY contain blocking user turns because they run sequentially.
- **Fork**: **all** streams execute in parallel; streams MUST be entirely non-blocking until `join`.

### 6.4 Single-Await Invariant

> A graph MUST NOT have more than one execution path awaiting end-user input at the same time.

Consequences:

- A `fork` MUST contain only silent (`await: none`) streams until `join`.
- Blocking user turns MAY appear on alternative branches (only one is taken), but not on concurrent fork streams.
- A `call` that can await `user` MUST NOT appear inside a forked stream.
- The runtime MUST reject any graph violating this invariant (VR-11).

### 6.5 Canonical Patterns

These patterns are normative compositions, not node types. They exist because the node taxonomy deliberately provides no compound dialogue node (P8).

#### 6.5.1 Bounded probe loop

A semi-structured probe is a sequence of single-exchange turns whose termination is a judgment. It MUST be authored as a `loop` of single-exchange nodes:

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

**Rules**

- The exit flag MUST be written by a declared silent node (`classify`, `llm` in `silent` mode, or `analyze`) with recorded provenance (§4.3).
- A loop whose exit depends on a model-written flag MUST declare `max_iterations` (VR-24). The flag is a judgment, not a guarantee.
- Each probe turn is a separate turn (§3.3): separately validated, separately transcribed, and separately subject to `limits`.
- Probe banks and item text MUST come from state or `call` subgraphs, not hard-coded in the loop body, when the graph is reused across items.

#### 6.5.2 Any-order navigation

Interviewer-driven navigation with completion tracking composes from existing nodes:

1. `ask` (`single_select`) presents the outstanding sections, using `options_ref` bound to a list-typed state variable (§4.2).
2. `call` invokes the chosen section.
3. `set` records the section as complete in a list-typed state variable.
4. `loop.exit_when` uses a `decide` (`combine: all`) over that variable to terminate (§4.4).

No worklist node is required: the sequencing policy is authorial, not structural.

#### 6.5.3 Section collection and accumulation

Two patterns cover "assemble a section's responses" and "build a collection whose length is discovered at runtime".

**Section collection.** A section's responses are projected into a view, and derived values are expressed over that view rather than over scattered slots:

```yaml
views:
  mdd_responses:
    type: list<enum>
    source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]
  mdd_score:
    type: integer
    expr: "count(views.mdd_responses, item == 'yes')"
  mdd_complete:
    type: boolean
    expr: "count(views.mdd_responses) == len(views.mdd_responses)"
```

- `source` is the explicit sync point: the view names exactly the slots it draws from, so nothing is collected implicitly.
- A section with heterogeneous item types uses a list of objects built with `expr` (§3.8).
- Completion, partial reporting, and any score are views over the same list; they MUST NOT be recomputed independently in several `set` nodes.

**Accumulation.** A collection whose length is not known at authoring time is built by initialising a list slot and appending inside a `foreach` body:

```yaml
- id: init_log
  type: set
  config: { assignments: [ { slot: attempt_log, expression: "[]" } ] }
- id: attempts_loop
  type: foreach
  config:
    over: { list_ref: state.attempt_slots }
    body_entry: record_attempt
    item_slot: current_attempt
    max_iterations: 6
- id: record_attempt
  type: set
  config:
    assignments:
      - slot: attempt_log
        expression: "append(state.attempt_log, state.current_attempt)"
```

- `foreach` snapshots the **iteration set** at entry (§4.4, `foreach`) but not the accumulator; execution is single-threaded and ordered, so appends are deterministic.
- Each iteration is a separate `set` execution and therefore observes the previous iteration's write. This is the only sanctioned read-modify-write pattern and it is replay-safe because every write is recorded.
- When the iteration set is a count rather than a list, the same pattern uses a bounded `loop` with `counter_slot`.
- A pre-populated list in state is never reordered or mutated in place; `append` returns a new value (§5.5).

#### 6.5.4 Flat iteration over multiple dimensions

Iteration is one-dimensional. A grid — e.g. criteria × substances — MUST be expressed as an outer `foreach` over one dimension and a `call` (or a fixed sequence of nodes) for the inner dimension:

```yaml
- id: per_drug
  type: foreach
  config:
    over: { list_ref: state.drugs_endorsed }
    body_graph: { registry: drug_modules, key: "item.key" }
    item_slot: current_drug
    max_iterations: 10
```

- Nested `foreach` is not part of the language (VR-32). The inner dimension is either a subgraph invoked per element or an unrolled sequence, which is acceptable because the inner set is fixed at authoring time.
- This keeps the control-flow graph flat and every iteration bound statically visible; a nested iterator would turn the effective iteration count into a runtime product.
- A `loop` inside a `foreach` body, or vice versa, is permitted; only `foreach` inside `foreach` is forbidden, because that is the case where the bound becomes a runtime product.

---

## 7. Execution Semantics

### 7.1 Turn Model

- Execution proceeds node by node following edges.
- `await: none` nodes execute immediately, in sequence (or concurrently under `fork`).
- `await: user` nodes emit their output, open a turn, and suspend until a validated response or timeout.
- `await: external` nodes suspend until the external event or timeout.
- Silent nodes executed between turns are attributed to the upcoming turn's record.

**Conversation scope.** The **visible conversation scope** of a node is the set of prior interactions it can see, and `call.state_isolation` (§4.4) sets it. In `shared` mode the scope is the whole session; in `isolated` mode it is the subgraph's own execution, so a child cannot see parent turns any more than it can read parent state. This is what keeps one trial's chat out of the next.

**History assembly.** An `llm` assembles its prompt as: leading `system` messages, then the block selected by `history`, then the remaining messages in declaration order (§4.3). An **interaction** is one turn (§3.3) — a node's user-facing output and the participant's response — and each is rendered by the turn-rendering rule below. `history` takes one of four shapes:

| Shape | Meaning |
|---|---|
| `all` (default) | Every prior interaction in the visible scope, in order. |
| `none` | No history. |
| `{last: N}` | The last `N` interactions in scope. `N` MUST be a non-negative integer; `0` is equivalent to `none`. |
| `{from: [...]}` | The listed `{role, template}` entries, in the given order. |

`last` and `from` MAY be combined; `from` entries render first, then the selected interactions.

**Turn rendering.** History is assembled from the turns in the visible scope, in chronological order. Each turn renders by node type:

| Node | Emitted output | Participant response |
|---|---|---|
| `ask` | one `assistant` message (the prompt) | one `user` message (the validated reply) |
| `llm` (`conversational`) | one `assistant` message (the emitted text) | one `user` message (the validated reply) |
| `say` | one `assistant` message (the text) | — |
| `llm` (`silent`), and all other `await: none` nodes | no message | — |

Within a turn the emitted output renders before the participant's response, because that is the order in which they occurred. Turns render in the order they executed, so a bot-speaks-first conversation renders `assistant`, `user`, `assistant`, `user`, … An `ask` prompt and a `say` message are authored text, not model output, but they are user-facing and render with the `assistant` role so the model sees what it is continuing. A `history` shape selects **turns**, not messages, so one turn may contribute two messages.

Arbitrary state is not serialized into the prompt by default, because there is no canonical rendering for nested values; an author who wants a specific value in the prompt declares it with `from`, which reuses the template mechanism (§3.2).

### 7.2 Awaiting and Silence

- A `say` or `llm` in `silent` mode MUST NOT suspend.
- An `analyze` MUST NOT suspend; it writes a result or `default`.
- Only `ask`, `llm` (`conversational`), `handoff`, and nodes that inherit await from a body or subgraph (`loop`, `foreach`, `call`) MAY suspend.

### 7.3 Timeouts and Failure Resolution

- On timeout, the runtime MUST take the node's `timeout` edge if present; otherwise `error`; otherwise the graph's `on_error`; otherwise end with `limit_exceeded`.
- On model/tool/validation failure, the runtime MUST take `error` if present; otherwise `on_error`.
- Failure in a silent fork stream MUST NOT cancel sibling streams unless the graph declares `fail_fast`.

### 7.4 Budgets

When any `limits` value is exceeded, the runtime MUST terminate the current path at `on_error` or an `end` with outcome `limit_exceeded`. Budgets MUST be checked at turn boundaries.

A `loop.max_iterations` is the local bound for that loop; graph `limits` are session-wide backstops checked at turn boundaries. When both are configured, the loop's bound SHOULD be set at or below the corresponding session limit so the loop's `exit` is reached first. If a session limit fires before the loop exits, the path terminates at `limit_exceeded` and the loop's `exit` is not taken.

### 7.5 Determinism and Replay

Given the same graph version, `seed`, and user inputs, all `sample` and `classify` (deterministic backends) decisions MUST be reproducible. Because `llm`/`analyze` outputs may vary, replays SHOULD pin recorded model outputs or accept divergence while preserving recorded `sample` choices.

Expressions, guards, `decide` tests, and views are pure functions of State, Context, and inputs, so they replay exactly. Because views are recomputed rather than stored, a runtime MUST record the **evaluated view values** that fed any routing decision or user-facing message, so replay never depends on re-deriving them from a different state snapshot.

The same rule covers assembled prompts: because `history` is assembled from recorded interactions and recorded state, a runtime MUST record the **assembled prompt** (or its hash plus the inputs it was built from) for every `llm` invocation. A replay MUST NOT reconstruct a prompt from a different state snapshot than the one the model actually saw.

---

## 8. Validation Rules

A graph MUST pass all rules below before execution. Rules are numbered for reference in tooling and examples.

- **VR-1** — Node ids MUST be unique within a graph.
- **VR-2** — Exactly one `start` node MUST exist; it MUST have no inbound edges.
- **VR-3** — Every non-`end` node MUST have at least one outbound transition (an edge, a `sample` choice, or a `decide` outcome); every `end` node MUST have none. Every `foreach` MUST declare an `exit` transition.
- **VR-4** — Every edge endpoint MUST reference an existing node and an existing port on that node.
- **VR-5** — Every node MUST be reachable from `start`.
- **VR-6** — Every node MUST be able to reach at least one `end` node.
- **VR-7** — Every state reference MUST resolve in `state_schema`; every `views.*` reference MUST resolve to a declared view (§3.8); every `session.*` reference MUST name a scalar session field (§3.1) and MUST NOT be written (VR-33); every write MUST be declared in `effects.write`. `item` and `index` MUST appear only in element-scoped positions (§5.3).
- **VR-8** — Every `ask` MUST declare a valid `input_type`, a `slot`, and a `response_schema` when `input_type: custom`. For `single_select` and `multi_select`, exactly one of `options` or `options_ref` MUST be declared, and `options_ref` MUST resolve to a list-typed state variable or view whose element type matches the input type.
- **VR-9** — Every `branch` MUST have a `default` edge or provably exhaustive guards.
- **VR-10** — Unguarded cycles are forbidden; every cycle MUST pass through a `loop` node or a terminating guarded edge.
- **VR-11** — `fork`/`join` MUST be balanced; forked streams MUST be entirely non-blocking; the graph MUST satisfy the single-await invariant (§6.4). A `foreach` whose body can await `user` MUST NOT appear in a forked stream.
- **VR-12** — Every `sample` choice target MUST exist; weights MUST be positive; `record_slot` MUST be declared.
- **VR-13** — Every `call.graph_ref` MUST exist and be version-pinned. A registry-based `graph_ref` MUST name a declared registry whose every entry is version-pinned.
- **VR-14** — Edge `type` MUST be valid for the source port (e.g., `conditional`/`default` only from control nodes; `conditional`/`default` MUST NOT originate from `decide` outcome ports or a `sample` node).
- **VR-15** — Every `end` node MUST declare a valid `outcome`.
- **VR-16** — Every node that can fail or block SHOULD declare `error`/`timeout` edges; if absent, `on_error` MUST be defined.
- **VR-17** — Every `fetch.source_ref` MUST reference a declared data source, and every requested attribute MUST exist in that source's schema.
- **VR-18** — Context (`context.*`) is read-only: no node or `set` MAY write it, and every `context.*` reference MUST resolve in the supplied Context or `context_schema`.
- **VR-19** — Credentials MUST be referenced via `credentials_ref`; inline secrets in graph, config, or context are forbidden.
- **VR-20** — Every `decide` operand MUST resolve in `state_schema` or `views`; all operands MUST share a type; `test.cmp` MUST be legal for the operand type and MUST carry the parameters that `cmp` requires; `n_of_k` MUST declare a positive `k`; exactly one of `true`/`false`/`unknown` MUST be taken per evaluation.
- **VR-21** — Every cross-session read MUST name a declared `session.links` entry; the source MUST be a `session_store` with `access: read_only`; every requested attribute MUST be exported by the linked session.
- **VR-22** — Every dynamic `graph_ref` MUST resolve through a declared `graph_registry`; every registry entry MUST be version-pinned; no other dynamic subgraph resolution is permitted.
- **VR-23** — Every `foreach` MUST iterate a declared list-typed variable, a declared view, or a registry projection; `filter`/`order_by`/`tiebreak`/`body_graph` expressions MUST type-check against the element type; `max_iterations` MUST be declared; ordering MUST be total.
- **VR-24** — No node may perform more than one user exchange (P8); any `loop` whose `exit_when` references a model-written variable MUST declare `max_iterations`.
- **VR-25** — Sessions in a case MUST declare `case_id` and `role`; `links` MUST be acyclic; `export.attributes` MUST be typed.
- **VR-26** — Every call MUST match a declared signature — a core function (§5.5) or, for a namespaced extension, an entry in the implementation manifest (§5.6) — with arguments assignable to parameter types and a result assignable to its target slot or enclosing parameter. Core and extension names are reserved and MUST NOT be redeclared as predicates.
- **VR-27** — `item` and `index` MUST appear only in element-scoped positions (`decide.test.predicate`, `foreach.filter`/`order_by`/`tiebreak`/`body_graph`, and the element-scoped argument of `map`/`filter`/`find`/`count`/`any`/`all`/`none`/`sort`), and MUST type-check against the element type derived from the list argument.
- **VR-28** — `set` assignments MUST be evaluated simultaneously against pre-node state and applied atomically; each expression MUST type-check against its slot; empty list/object literals MUST be contextually typed; no assignment MAY depend on another assignment in the same node.
- **VR-29** — Every view MUST declare `type` and exactly one of `source`/`expr`; all references MUST resolve; `expr` MUST type-check to `type`; the view dependency graph MUST be acyclic; no node MAY write a view.
- **VR-30** — Every `views.*` reference MUST resolve to a declared view; view dependencies MUST resolve transitively to `state_schema` or `context_schema`. A node that reads a view in its own config MUST declare it in `effects.read`. Edge guards and `loop.exit_when` MUST resolve but need not repeat their references in `effects.read`, exactly as for `state.*` under VR-7.
- **VR-31** — A graph calling an extension MUST declare it in `capabilities`; validation MUST fail when the target implementation's manifest does not advertise a compatible name and signature. Extension names MUST be namespaced and MUST NOT shadow core functions, other extensions, or graph predicates.
- **VR-32** — A `foreach` MUST NOT appear in the body of another `foreach` within the same graph. Multi-dimensional iteration MUST cross a `call` boundary (§6.5.4).
- **VR-33** — Every `session.*` reference MUST name a scalar field of the session record (§3.1); no node, `set`, or view MAY write one; `session` MUST NOT be redeclared as a predicate or extension name.
- **VR-34** — An `llm` MUST declare exactly one of `prompt_template` or `messages`; `messages` MUST contain at least one entry, at least one of which has role `user` or `assistant`; every `role` MUST be `system`, `user`, or `assistant`; every `system` message MUST precede every non-system message; every template MUST resolve.
- **VR-35** — `llm.history` MUST be `all`, `none`, `{last: N}` with `N` a non-negative integer, or `{from: [...]}` of `{role, template}` entries; every `from` reference MUST resolve in the node's visible conversation scope (§7.1); `message_slot` MUST be a declared `string` state variable, MUST differ from `slot`, MUST be declared in `effects.write`, and MUST NOT appear when `mode: silent`.
- **VR-36** — Every subgraph exposes a reserved `outcome` output and MUST NOT declare an output of that name; a subgraph `end` MUST return control through the caller's `out`; an `output_map` binding for `outcome` MUST target a declared `enum` or `string` state variable (§6.2).
- **VR-37** — Both operands of `matches` MUST be `string`; an implementation MUST evaluate it as a case-insensitive (ASCII folding) contiguous-substring test with no regular-expression or anchoring semantics (§5.3).

---

## 9. Extensibility

Custom node types are a **stub** in v1. A custom node MUST declare the same common contract as built-in nodes:

```yaml
type: x-vendor.sentiment_shift
ports: { in: 1, out: 1, error: 1 }
config_schema: { ... }
effects: { read: [last_reply], write: [sentiment_delta] }
await: none
```

**Contract**

- **Identity**: namespaced `type` id and version.
- **Ports**: declared inbound/outbound ports usable by edges.
- **Config schema**: validates `config`.
- **Effects**: declared state/context reads and state writes.
- **Await semantics**: one of `user`, `external`, `none`.
- **Failure behavior**: which edge is taken on error/timeout.

**Access and payload handling**

- A custom node MUST document **how the runtime obtains and executes it** (e.g., in-process handler, HTTP endpoint, or a standard protocol such as MCP).
- A custom node MUST define **what to do with the returned payload**: which slot receives it, whether it is validated against a schema, and what happens on invalid output.
- Adapters MAY normalize external protocols (including MCP) to this contract, but this specification does not mandate a transport.

This section intentionally does not define a full plugin ABI, sandboxing model, or node-level capability negotiation; those are deferred (§11). Expression-level capabilities are separate and are defined in §5.6: an implementation declares the pure functions it provides, a graph declares the ones it requires, and validation checks the two against each other.

---

## 10. Worked Examples

All examples use only node and edge types defined in §4–§5.

### 10.1 Structured Survey

Demonstrates varied `ask` input types, a `branch`, and a `loop`-free linear flow.

```yaml
graph:
  id: wellbeing_survey
  version: 0.1.0
  seed: 7
  entry: start
  state_schema:
    age:        { type: integer, scope: session, mutable: true }
    consent:    { type: boolean, scope: session, mutable: true }
    mood:       { type: integer, scope: session, mutable: true }
    topics:     { type: list,    scope: session, mutable: true }
    comments:   { type: string,  scope: session, mutable: true }
  nodes:
    - { id: start,  type: start }
    - { id: q_age,  type: ask, config: { prompt: "How old are you?", input_type: integer, slot: age, response_schema: { min: 0, max: 120 } } }
    - { id: b_adult, type: branch }
    - { id: q_consent, type: ask, config: { prompt: "Do you consent to this survey?", input_type: boolean, slot: consent } }
    - { id: q_mood, type: ask, config: { prompt: "Rate your mood today.", input_type: slider, slot: mood, response_schema: { min: 1, max: 10, step: 1 } } }
    - { id: q_topics, type: ask, config: { prompt: "Pick topics to discuss.", input_type: multi_select, slot: topics, response_schema: { options: [sleep, work, social], max_selections: 2 } } }
    - { id: q_comments, type: ask, config: { prompt: "Anything else?", input_type: text, slot: comments, required: false } }
    - { id: done,   type: end, config: { outcome: completed } }
    - { id: decline, type: end, config: { outcome: abandoned, reason: "minor without consent" } }
  edges:
    - { from: start,     to: q_age,     type: next }
    - { from: q_age,     to: b_adult,   type: next }
    - { from: b_adult,   to: q_consent, type: conditional, when: "state.age >= 18", priority: 1 }
    - { from: b_adult,   to: decline,   type: default }
    - { from: q_consent, to: q_mood,    type: conditional, when: "state.consent == true", priority: 1 }
    - { from: q_consent, to: decline,   type: default }
    - { from: q_mood,    to: q_topics,  type: next }
    - { from: q_topics,  to: q_comments,type: next }
    - { from: q_comments,to: done,      type: next }
```

Validation: passes VR-1…VR-19. `q_comments` is `required: false`, so an empty response is valid.

### 10.2 Semi-Structured Interview with Silent Risk Analysis and Randomized Variant

Demonstrates a `fork` running a silent `analyze` in parallel, a seeded `sample` choosing a supportive vs. challenging follow-up, a conversational `llm`, and a risk-based `handoff`.

```yaml
graph:
  id: interview_v1
  version: 0.1.0
  seed: 1234
  entry: start
  state_schema:
    user_reply:  { type: string,  scope: session, mutable: true }
    risk_score:  { type: number,  scope: session, mutable: true }
    variant:     { type: string,  scope: session, mutable: true }
    topic:       { type: string,  scope: session, mutable: true }
  limits: { max_turns: 20, max_wall_clock_s: 900 }
  nodes:
    - { id: start, type: start }
    - { id: q0, type: ask, config: { prompt: "Tell me about your day.", input_type: text, slot: user_reply } }
    - { id: f1, type: fork, config: { streams: [risk, prep] } }
    - { id: risk, type: analyze, config: { backend: risk_model_v2, inputs: [state.user_reply], slot: risk_score, default: 0.0 } }
    - { id: prep, type: classify, config: { backend: bert, model: topic_clf_v1, inputs: [state.user_reply], labels: [work, health, social], slot: topic } }
    - { id: j1, type: join, config: { mode: all } }
    - { id: pick, type: sample, config: { distribution: weighted, choices: [ { to: llm_support, weight: 1 }, { to: llm_challenge, weight: 1 } ], record_slot: variant } }
    - { id: llm_support,   type: llm, config: { model: m1, mode: conversational, prompt_template: "Respond supportively about {{ state.topic }}.", slot: user_reply } }
    - { id: llm_challenge, type: llm, config: { model: m1, mode: conversational, prompt_template: "Respectfully challenge the user about {{ state.topic }}.", slot: user_reply } }
    - { id: b_risk, type: branch }
    - { id: escalate, type: handoff, config: { target: crisis_queue, reason: "risk_score={{ state.risk_score }}", resume: continue } }
    - { id: end_escalated, type: end, config: { outcome: escalated } }
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start,       to: q0,           type: next }
    - { from: q0,          to: f1,           type: next }
    - { from: f1,          to: risk,         type: next }
    - { from: f1,          to: prep,         type: next }
    - { from: risk,        to: j1,           type: next }
    - { from: prep,        to: j1,           type: next }
    - { from: j1,          to: pick,         type: next }
    - { from: llm_support, to: b_risk,       type: next }
    - { from: llm_challenge, to: b_risk,     type: next }
    - { from: b_risk,      to: escalate,     type: conditional, when: "state.risk_score > 0.7", priority: 1 }
    - { from: b_risk,      to: done,         type: default }
    - { from: escalate,    to: end_escalated,type: next }
```

Validation: passes VR-1…VR-19. The forked streams `risk`/`prep` are silent (`await: none`), satisfying VR-11. `sample` records `variant`; pinning `variant` or `seed` fixes the chosen variant.

### 10.3 Triage with Non-LLM Classification

Demonstrates `classify` with a non-LLM backend driving a `branch`.

```yaml
graph:
  id: support_triage
  version: 0.1.0
  seed: 1
  entry: start
  state_schema:
    complaint: { type: string, scope: session, mutable: true }
    priority:  { type: string, scope: session, mutable: true }
    confidence: { type: number, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - { id: q, type: ask, config: { prompt: "Describe your issue.", input_type: text, slot: complaint, response_schema: { max_length: 2000 } } }
    - { id: clf, type: classify, config: { backend: predictive, model: urgency_v3, inputs: [state.complaint], labels: [urgent, routine, self_serve], slot: priority, confidence_slot: confidence, threshold: 0.5 } }
    - { id: b, type: branch }
    - { id: human, type: handoff, config: { target: urgent_queue, reason: "urgent: {{ state.complaint }}", resume: continue } }
    - { id: kb, type: llm, config: { model: m1, mode: conversational, prompt_template: "Answer using the knowledge base.", slot: complaint } }
    - { id: end_ok, type: end, config: { outcome: completed } }
    - { id: end_human, type: end, config: { outcome: escalated, reason: "handled by human" } }
    - { id: end_low, type: end, config: { outcome: escalated, reason: "low confidence" } }
  edges:
    - { from: start, to: q,        type: next }
    - { from: q,     to: clf,      type: next }
    - { from: clf,   to: b,        type: next }
    - { from: clf,   to: end_low,  type: error }
    - { from: b,     to: human,    type: conditional, when: "state.priority == 'urgent'", priority: 1 }
    - { from: b,     to: kb,       type: conditional, when: "state.priority == 'self_serve'", priority: 2 }
    - { from: b,     to: end_low,  type: default }
    - { from: human, to: end_human,type: next }
    - { from: kb,    to: end_ok,   type: next }
```

Validation: passes VR-1…VR-19. `clf` uses `backend: predictive` (non-LLM); low confidence routes to `end_low` via the `error` edge.

### 10.4 Data Source Fetch and Inline Context

Demonstrates a `fetch` node reading a typed attribute from an external Data Source into State, and read-only inline Context used in a message template.

```yaml
graph:
  id: personalized_greeting
  version: 0.1.0
  entry: start
  data_sources:
    crm:
      kind: postgres
      endpoint: ${env.CRM_DSN}
      credentials_ref: secret://crm/readonly
      attributes:
        account_tier: { type: enum, values: [free, pro, enterprise] }
      access: read_only
  state_schema:
    account_tier: { type: enum, values: [free, pro, enterprise], scope: session, mutable: true }
  context_schema:
    participant: { type: object }
  context:
    participant: { id: "P-1042", locale: "en-GB" }
  nodes:
    - { id: start, type: start }
    - { id: get_tier, type: fetch, config: { source_ref: crm, attributes: [account_tier], output_map: { account_tier: account_tier } } }
    - { id: greet, type: say, config: { message: "Hi {{ context.participant.id }}, your {{ state.account_tier }} plan is active ({{ context.participant.locale }})." } }
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start,    to: get_tier, type: next }
    - { from: get_tier, to: greet,    type: next }
    - { from: greet,    to: done,     type: next }
```

Validation: passes VR-1…VR-19. `fetch` reads `crm` into `state.account_tier`; `greet` reads both immutable `context.*` and mutable `state.*`. Credentials are a `credentials_ref`, never inline (VR-19).

### 10.5 Multi-Informant Synthesis Across Linked Sessions

Demonstrates a synthesis session reading exported ratings from two informant sessions through a `session_store`, then resolving disagreement with a clinician `ask`. The parent and child sessions are separate executions of their own graph; no single flow addresses both informants.

```yaml
graph:
  id: ksads_synthesis
  version: 0.1.0
  entry: start
  data_sources:
    linked_sessions:
      kind: session_store
      attributes:
        dep_mood_summary_ce: { type: integer }
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
    - { from: start,      to: get_parent, type: next }
    - { from: get_parent, to: get_child,  type: next }
    - { from: get_child,  to: b_disagree, type: next }
    - { from: b_disagree, to: reconcile,  type: conditional, when: "state.parent_dep_mood_ce != state.child_dep_mood_ce", priority: 1 }
    - { from: b_disagree, to: done,       type: default }
    - { from: reconcile,  to: done,       type: next }
```

Validation: passes VR-1…VR-25. The session declares `links: {parent: …, child: …}` and both linked sessions export `dep_mood_summary_ce` (VR-21). Pairwise disagreement is a `branch` guard; `decide` compares operands to literals, never to each other.

### 10.6 Typed Decision and Onset-Ordered Dispatch

Demonstrates a `decide` score-gated trigger replacing an enumerated guard, followed by a `foreach` that dispatches reusable supplement subgraphs in onset order.

```yaml
graph:
  id: ksads_supplement_dispatch
  version: 0.1.0
  seed: 11
  entry: start
  graph_registry:
    supplements:
      dep:
        graph_ref: ksads_dep_supplement
        version: 0.1.0
        attributes: { required: true, onset_age: 9 }
      ptsd:
        graph_ref: ksads_ptsd_supplement
        version: 0.1.0
        attributes: { required: false, onset_age: 11 }
  state_schema:
    dep_mood_ce:        { type: integer, scope: session, mutable: true }
    dep_irrit_ce:       { type: integer, scope: session, mutable: true }
    dep_trigger:        { type: boolean, scope: session, mutable: true }
    current_supplement: { type: object,  scope: session, mutable: true }
    supplement_index:   { type: integer, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - { id: q_mood,  type: ask, config: { prompt: "Depressed mood (current).", input_type: scale, slot: dep_mood_ce,  response_schema: { min: 0, max: 3 } } }
    - { id: q_irrit, type: ask, config: { prompt: "Irritability (current).",  input_type: scale, slot: dep_irrit_ce, response_schema: { min: 0, max: 3 } } }
    - id: dep_trigger_check
      type: decide
      config:
        operands: { refs: [state.dep_mood_ce, state.dep_irrit_ce] }
        test: { cmp: gte, value: 3 }
        combine: any
        on_missing: unknown
        result_slot: dep_trigger
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
    - { id: done,       type: end, config: { outcome: completed } }
    - { id: incomplete, type: end, config: { outcome: completed, reason: "trigger unknown; flagged for review" } }
  edges:
    - { from: start,             to: q_mood,           type: next }
    - { from: q_mood,            to: q_irrit,          type: next }
    - { from: q_irrit,           to: dep_trigger_check, type: next }
    - { from: dep_trigger_check, port: true,    to: run_supplements, type: next }
    - { from: dep_trigger_check, port: false,   to: done,            type: next }
    - { from: dep_trigger_check, port: unknown, to: incomplete,      type: next }
    - { from: run_supplements,   port: exit,    to: done,            type: next }
```

Validation: passes VR-1…VR-25. `dep_trigger_check` is a pure `decide` (VR-20) whose `unknown` port routes missing ratings rather than scoring them as absent. `run_supplements` iterates a registry projection with a total order (VR-23); its `body` port is unused because `body_graph` supplies the body. Dispatch targets are enumerable from the registry (VR-22).

### 10.7 Bounded Probe Loop

Demonstrates P8: a multi-exchange probe authored as single-exchange nodes inside a bounded `loop`, with a model-judged exit flag.

```yaml
graph:
  id: ksads_probe_item
  version: 0.1.0
  entry: start
  state_schema:
    current_item:     { type: string,  scope: session, mutable: true }
    probe_bank:       { type: string,  scope: session, mutable: true }
    probe_reply:      { type: string,  scope: session, mutable: true }
    probe_status:     { type: enum, values: [need_more, complete, declined], scope: session, mutable: true }
    probe_turn_count: { type: integer, scope: session, mutable: true }
    item_rating:      { type: integer, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
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
    - { id: rate_item, type: ask, config: { prompt: "Assign the item rating.", input_type: scale, slot: item_rating, response_schema: { min: 0, max: 3 } } }
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start,      to: probe_loop,  type: next }
    - { from: probe_loop, port: body, to: probe_turn, type: next }
    - { from: probe_turn, to: probe_assess, type: next }
    - { from: probe_assess, to: probe_loop, type: next }
    - { from: probe_loop, port: exit, to: rate_item, type: next }
    - { from: rate_item,  to: done,        type: next }
```

Validation: passes VR-1…VR-25. The cycle passes through a `loop` (VR-10); the exit depends on a model-written flag, so `max_iterations` is REQUIRED (VR-24). Each probe turn is a distinct turn (§3.3), separately transcribed and budgeted.

### 10.8 Derived Views, Section Scoring, and Accumulation

Demonstrates `views` (§3.8), aggregates and constructors (§5.5), atomic `set` (§4.5), and the accumulation pattern (§6.5.3). A three-item mood section is scored, gated, summarised, and supplemented by a runtime-sized attempt log.

```yaml
graph:
  id: mood_screen
  version: 0.1.0
  state_schema:
    mdd_q1: { type: enum, values: [no, yes, unk], scope: session, mutable: true }
    mdd_q2: { type: enum, values: [no, yes, unk], scope: session, mutable: true }
    mdd_q3: { type: enum, values: [no, yes, unk], scope: session, mutable: true }
    severity: { type: integer, scope: session, mutable: true }
    attempt_slots: { type: list<integer>, scope: session, mutable: true }
    current_attempt: { type: integer, scope: session, mutable: true }
    attempt_log: { type: list<integer>, scope: session, mutable: true }
    summary: { type: string, scope: session, mutable: true }
  views:
    mdd_responses:
      type: list<enum>
      source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]
    mdd_score:
      type: integer
      expr: "count(views.mdd_responses, item == 'yes')"
    mdd_flagged:
      type: boolean
      expr: "views.mdd_score >= 2"
    mdd_complete:
      type: boolean
      expr: "count(views.mdd_responses) == len(views.mdd_responses)"
    weighted_severity:
      type: integer
      expr: "sum(map(views.mdd_responses, lookup(item, {no: 0, unk: 1, yes: 2}, 0)))"
  entry: start
  on_error: end_abandoned
  nodes:
    - { id: start, type: start }
    - { id: q1, type: ask, effects: { write: [mdd_q1] }, config: { prompt: "Little interest or pleasure?", input_type: single_select, options: [no, yes, unk], slot: mdd_q1 } }
    - { id: q2, type: ask, effects: { write: [mdd_q2] }, config: { prompt: "Feeling down or hopeless?", input_type: single_select, options: [no, yes, unk], slot: mdd_q2 } }
    - { id: q3, type: ask, effects: { write: [mdd_q3] }, config: { prompt: "Trouble sleeping?", input_type: single_select, options: [no, yes, unk], slot: mdd_q3 } }
    - id: score
      type: set
      effects: { read: [weighted_severity, mdd_responses], write: [severity, summary] }
      config:
        assignments:
          - { slot: severity, expression: "views.weighted_severity" }
          - { slot: summary, expression: "join(map(views.mdd_responses, to_string(item)), ', ')" }
    - id: gate
      type: branch
    - { id: init_log, type: set, effects: { write: [attempt_log] }, config: { assignments: [ { slot: attempt_log, expression: "[]" } ] } }
    - id: log_loop
      type: foreach
      effects: { write: [current_attempt] }
      config:
        over: { list_ref: state.attempt_slots }
        body_entry: record_attempt
        item_slot: current_attempt
        max_iterations: 6
    - id: record_attempt
      type: set
      effects: { write: [attempt_log] }
      config:
        assignments:
          - { slot: attempt_log, expression: "append(state.attempt_log, state.current_attempt)" }
    - { id: done, type: end, config: { outcome: completed } }
    - { id: end_abandoned, type: end, config: { outcome: abandoned } }
  edges:
    - { from: start,  to: q1, type: next }
    - { from: q1,     to: q2, type: next }
    - { from: q2,     to: q3, type: next }
    - { from: q3,     to: score, type: next }
    - { from: score,  to: gate, type: next }
    - { from: gate, to: init_log, type: conditional, when: "views.mdd_flagged", priority: 1 }
    - { from: gate, to: done, type: default }
    - { from: init_log,    to: log_loop, type: next }
    - { from: log_loop,    port: body, to: record_attempt, type: next }
    - { from: record_attempt, to: log_loop, type: next }
    - { from: log_loop,    port: exit, to: done, type: next }
  terminals: [done, end_abandoned]
```

Validation: passes VR-1…VR-32. `mdd_responses` names its sources explicitly, so nothing is collected implicitly. `mdd_score` and `mdd_complete` are pure functions of that one view; `weighted_severity` shows `lookup` as an enum→number table composed inside `map`. The `score` node reads two views in its config and declares both in `effects.read` (VR-30). Its two assignments are evaluated against pre-node state and applied atomically (VR-28). `record_attempt` reads and appends its own slot; this is legal because each iteration is a separate `set` execution (§6.5.3). `views.mdd_flagged` is read only by an edge guard, so it must resolve but needs no `effects.read` entry.

### 10.9 Design Assignment, Composed Prompts, and Subgraph Outcome

Demonstrates `session.seed` with `range`/`shuffle` (§5.5), `llm.messages` and `llm.history` (§4.3), `message_slot` for inspecting generated text, and the subgraph termination contract (§6.2).

The trial subgraph emits two bot replies, inspects the first for a crisis rule, and can end early with a different `outcome`:

```yaml
graph:
  id: mobi_trial
  version: 0.1.0
  entry: start
  on_error: abandoned
  state_schema:
    story_text:           { type: string,  scope: session, mutable: true }
    stance_instruction:   { type: string,  scope: session, mutable: true }
    question_instruction: { type: string,  scope: session, mutable: true }
    choice_pre:           { type: enum, values: [option_a, option_b], scope: session, mutable: true }
    certainty_pre:        { type: integer, scope: session, mutable: true }
    justification:        { type: string,  scope: session, mutable: true }
    bot_reply_1:          { type: string,  scope: session, mutable: true }
    reply_1:              { type: string,  scope: session, mutable: true }
    crisis_label:         { type: enum, values: [safe, crisis], scope: session, mutable: true }
    bot_reply_2:          { type: string,  scope: session, mutable: true }
    reply_2:              { type: string,  scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - id: bot_1
      type: llm
      effects: { write: [bot_reply_1, reply_1] }
      config:
        model: m1
        mode: conversational
        messages:
          - { role: system, template: "{{ context.base_system }}" }
          - { role: system, template: "{{ state.stance_instruction }}" }
          - { role: system, template: "{{ state.question_instruction }}" }
          - { role: user,   template: "{{ state.story_text }}" }
        history:
          from:
            - { role: user, template: "Initial choice: {{ state.choice_pre }} (certainty {{ state.certainty_pre }})" }
            - { role: user, template: "My reason: {{ state.justification }}" }
        message_slot: bot_reply_1
        slot: reply_1
    - id: crisis_check
      type: classify
      effects: { read: [bot_reply_1], write: [crisis_label] }
      config:
        backend: rules
        model: crisis_rules
        inputs: [state.bot_reply_1]
        labels: [safe, crisis]
        slot: crisis_label
    - id: crisis_gate
      type: branch
    - id: bot_2
      type: llm
      effects: { write: [bot_reply_2, reply_2] }
      config:
        model: m1
        mode: conversational
        messages:
          - { role: system, template: "{{ context.base_system }}" }
          - { role: system, template: "{{ state.stance_instruction }}" }
        history: all
        message_slot: bot_reply_2
        slot: reply_2
    - { id: closing,    type: say, config: { message: "{{ context.trial_close_text }}" } }
    - { id: done,       type: end, config: { outcome: completed } }
    - { id: crisis_end, type: end, config: { outcome: escalated } }
    - { id: abandoned,  type: end, config: { outcome: abandoned } }
  edges:
    - { from: start,       to: bot_1,       type: next }
    - { from: bot_1,       to: crisis_check, type: next }
    - { from: crisis_check, to: crisis_gate, type: next }
    - { from: crisis_gate, to: crisis_end,  type: conditional, when: "state.crisis_label == 'crisis'", priority: 1 }
    - { from: crisis_gate, to: bot_2,       type: default }
    - { from: bot_2,       to: closing,     type: next }
    - { from: closing,     to: done,        type: next }
  terminals: [done, crisis_end, abandoned]
```

The caller fragment derives a block-balanced assignment from the session seed and binds the trial's terminal outcome:

```yaml
- id: build_assignment
  type: set
  effects: { write: [assignment] }
  config:
    assignments:
      - slot: assignment
        expression: "flatten(map(range(state.reps), shuffle(state.cells, session.seed + index)))"
- id: trials
  type: foreach
  effects: { write: [current_cell] }
  config:
    over: { list_ref: state.assignment }
    body_entry: run_trial
    item_slot: current_cell
    max_iterations: 40
- id: run_trial
  type: call
  effects: { write: [trial_outcome] }
  config:
    graph_ref: { id: mobi_trial, version: 0.1.0 }
    state_isolation: isolated
    input_map:
      story_text: state.current_cell.story_text
      stance_instruction: state.current_cell.stance_instruction
    output_map:
      outcome: state.trial_outcome
```

Validation: the trial graph passes VR-1…VR-36; the caller fragment is shown without its enclosing state declarations. `bot_1` declares leading `system` messages and a `from` history block, so the assembled prompt is system instructions, then the two recorded values, then the story (VR-34). `bot_2` uses `history: all`, which renders the first exchange and both participant messages in order (§7.1). `message_slot` lets `crisis_check` inspect the text that was actually sent, which `slot` alone cannot provide (VR-35). The trial's three `end` nodes differ only in `outcome`, and the caller binds it through `output_map` (VR-36); `state_isolation: isolated` means the trial's conversation scope does not extend to earlier trials. `build_assignment` is a pure function of `state.cells`, `state.reps`, and `session.seed`, and its result is written to state, so the realized order is auditable rather than re-derived (VR-33, §7.5).

### 10.10 Bounded Free Conversation with a Phrase Terminator

Demonstrates a seeded random opener (P02), an `ask` opener rendered into history (§7.1), a bounded loop of single-exchange `llm` turns, and a literal phrase test with `matches` (§5.3, VR-37). The conversation runs until the participant utters the phrase or 100 loop turns elapse.

```yaml
graph:
  id: walrus_conversation
  version: 0.1.0
  entry: start
  on_error: abandoned
  context:
    persona: "You are a warm, curious conversation partner."
    openers:
      - "What's on your mind?"
      - "How has your day been?"
      - "What have you been thinking about lately?"
  limits:
    max_turns: 101
    max_wall_clock_s: 7200
    max_tokens: 200000
  state_schema:
    opener_prompt:      { type: string,  scope: session, mutable: true }
    last_reply:         { type: string,  scope: session, mutable: true }
    turn_count:         { type: integer, scope: session, mutable: true }
    termination_reason: { type: string,  scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - id: choose_opener
      type: set
      effects: { write: [opener_prompt] }
      config:
        assignments:
          - slot: opener_prompt
            expression: "get(shuffle(context.openers, session.seed), 0)"
    - id: open
      type: ask
      config:
        prompt: "{{ state.opener_prompt }}"
        input_type: text
        slot: last_reply
        timeout_s: 1800
    - { id: open_check, type: branch }
    - id: convo_loop
      type: loop
      effects: { write: [turn_count] }
      config:
        body_entry: chat_turn
        exit_when: "state.last_reply matches 'you MUST be a walrus!'"
        max_iterations: 100
        counter_slot: turn_count
    - id: chat_turn
      type: llm
      effects: { write: [last_reply] }
      config:
        model: m1
        mode: conversational
        messages:
          - { role: system, template: "{{ context.persona }}" }
          - { role: user,   template: "Reply to my last message and keep the conversation going." }
        history: all
        slot: last_reply
        timeout_s: 1800
    - { id: classify_exit, type: branch }
    - id: mark_phrase
      type: set
      effects: { write: [termination_reason] }
      config:
        assignments:
          - { slot: termination_reason, expression: "'phrase'" }
    - id: mark_cap
      type: set
      effects: { write: [termination_reason] }
      config:
        assignments:
          - { slot: termination_reason, expression: "'turn_cap'" }
    - { id: done, type: end, config: { outcome: completed } }
    - { id: abandoned, type: end, config: { outcome: abandoned } }
  edges:
    - { from: start,         to: choose_opener, type: next }
    - { from: choose_opener, to: open,          type: next }
    - { from: open,          port: timeout, to: abandoned, type: next }
    - { from: open,          to: open_check,    type: next }
    - { from: open_check,    to: mark_phrase,   type: conditional, when: "state.last_reply matches 'you MUST be a walrus!'", priority: 1 }
    - { from: open_check,    to: convo_loop,    type: default }
    - { from: convo_loop,    port: body, to: chat_turn, type: next }
    - { from: chat_turn,     to: convo_loop,    type: next }
    - { from: chat_turn,     port: timeout, to: abandoned, type: next }
    - { from: convo_loop,    port: exit, to: classify_exit, type: next }
    - { from: classify_exit, to: mark_phrase,   type: conditional, when: "state.last_reply matches 'you MUST be a walrus!'", priority: 1 }
    - { from: classify_exit, to: mark_cap,      type: default }
    - { from: mark_phrase,   to: done,          type: next }
    - { from: mark_cap,      to: done,          type: next }
  terminals: [done, abandoned]
```

Validation: passes VR-1…VR-37. `choose_opener` is a pure function of `context.openers` and `session.seed`, and writes the realized prompt to state (§3.4, §7.5). The opener is an `ask`, so the first `chat_turn` sees it rendered as an `assistant` message followed by the participant's `user` reply (§7.1). The loop body is a single `llm`, so one iteration is one bot message and one reply; exiting after the body leaves no dangling message, and `exit_when` is checked every turn (§4.4, VR-24). `matches` is a case-insensitive substring test, so `you must be a walrus!` and `well, you must be a walrus! anyway` both terminate while `you must be a walrus` does not (§5.3, VR-37). The two phrase checks — after the opener and after the loop — cover a participant who says the phrase on the very first turn. `mark_phrase` and `mark_cap` record which terminator fired, and `limits` is a backstop above the loop's own bound (§7.4).

---

## 11. Open Issues

- **Serialization**: JSON, YAML, or a binary schema; canonical form and schema language.
- **Guard grammar finalization**: precedence edge cases and type-coercion corner cases. The standard function library is now specified (§5.5); this item is narrowed accordingly.
- **Custom-node ABI**: full contract, sandboxing, capability negotiation, and whether to standardize on MCP as the transport adapter.
- **Multi-await forks**: whether to ever permit concurrent blocking user turns (currently forbidden by P5/VR-11).
- **Randomization breadth**: node-level `sample` is chosen for v1; revisit whether edge-level weights or deterministic assignment strategies (quota, A/B bucketing) are needed. Design assignment is now expressible with `session.seed` plus `shuffle`/`take`/`range` (§5.5, §10.9); whether a dedicated assignment primitive is warranted remains open.
- **Permutation algorithm**: `shuffle` is property-based, so two conforming implementations MAY permute differently for the same seed (§5.5). Pinning a PRNG and algorithm would make assignments portable across implementations at the cost of freezing an algorithm in the spec. The realized assignment is written to state either way.
- **History rendering**: `history.from` uses `{role, template}` entries rather than plain references, because plain references would need a serialization rule for every type (§7.1). Revisit if a canonical rendering is ever specified.
- **String folding**: `matches` folds ASCII case only, so non-Latin scripts compare exactly (§5.3). Full Unicode case folding is locale- and version-dependent across runtimes; revisit if an instrument needs it.
- **Turn rendering order**: v0.6.0 renders a turn's emitted output before the participant's response (§7.1), correcting v0.5.0's user-then-assistant order for bot-speaks-first turns. The alternative pairing drops the participant's latest message from the prompt; revisit only if a canonical chat ordering is needed.
- **Session reference set**: `session.*` exposes `seed`, `id`, `case_id`, and `role` (§3.1). Extend only when an instrument needs another scalar field.
- **Streaming and partial responses**: token streaming, partial validation, and interruption semantics.
- **Localization and rendering**: message catalogs, RTL, and channel-specific rendering of `ask` input types.
- **Data sources**: connector taxonomy, schema drift, cache/consistency semantics, credential rotation, and row-level access control.
- **Context provenance**: versioning, size limits (context-window growth), redaction, and merge-depth rules for session-supplied Context.
- **Privacy and retention**: state/context encryption, PII handling, and transcript retention policy.
- **Versioning and migration**: graph migration, session compatibility across graph versions, and replay guarantees.
- **Testing harness**: conformance suite, deterministic replay tooling, and simulation of user inputs.
- **Decision-node consolidation**: whether `decide` should absorb `branch` into a single decision primitive with an optional `operands` block, or remain complementary.
- **Missing-value sentinels**: whether state variables should declare sentinel values (e.g. `0` meaning "no information") so `on_missing` can detect them without repeating node-level policy.
- **Expression extensions**: where an implementation publishes its manifest of extension signatures (an out-of-band conformance profile, or a machine-readable file the validator loads), and whether a graph's `capabilities` list should stay explicit or be inferred from use.
- **Session stores**: retention, redaction, snapshot-vs-live reads of an active linked session, and whether exports are frozen at session end. Replay requires the observed snapshot to be recorded.
- **Nested iteration**: 2-D grids are expressed as an outer `foreach` plus a `call` per dimension, and nested `foreach` is forbidden (VR-32, §6.5.4). Revisit only if an instrument cannot be expressed that way.
- **Core-tier promotion**: the bar for moving an extension into §5.5 — proposed: two independent implementations with identical semantics, used by at least two instruments.
- **Numeric edge semantics**: `/`, `div`, `mod`, `round`, and overflow behaviour are fixed in §5.5 so scores are reproducible. Confirm these choices before independent implementations diverge.
- **`index` binding scope**: `index` is the position in the source list, i.e. pre-`filter` for a `filter` predicate. Confirm no other interpretation is intended.
- **View export**: whether derived views may be exported to linked sessions (§3.7). Deferred, because exporting a recomputed value requires a snapshot rule that does not yet exist.
- **Session role typing**: whether `session.role` should be a closed enum or remain a namespaced string.
- **Turn audit fields**: whether turn records should carry session role or responder identity.
