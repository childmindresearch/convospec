# Conversation Specification (ConvoSpec)

**Node Types and Graph Composition**

| Field | Value |
|---|---|
| Status | Draft |
| Version | 0.2.0 |
| Audience | Spec authors, runtime implementers, conversation designers |
| Scope | Conceptual node taxonomy and graph semantics (format-agnostic) |

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

**Non-Goals**

- A concrete serialization or wire format (deferred; see §11).
- A prompt-authoring language or model API.
- UI/rendering details beyond input-type semantics.
- Multi-agent orchestration outside a single conversation graph.
- A full plugin ABI for custom nodes (stub only; see §9).

---

## 2. Design Principles

- **P1 — Declarative.** A graph MUST fully describe its structure; runtime behavior MUST be derivable from the graph plus state, context, and inputs, without hidden state.
- **P2 — Composable.** Every node type MUST compose with every other node type through a common port and edge contract.
- **P3 — Deterministic control.** Branching, sampling, and iteration MUST be deterministic given the session seed and inputs. Model calls MAY be nondeterministic, but their effects MUST be confined to declared state writes.
- **P4 — Inspectable.** Every node MUST declare its state/context reads and state writes and its await semantics so a validator can reason about the graph without executing it.
- **P5 — Single-await.** At most one execution path MAY await end-user input at any time (§6.4).
- **P6 — Explicit failure.** Every node that can fail or block SHOULD declare error and timeout transitions (§5).
- **P7 — Extensible without churn.** New node types MUST be addable through a declared contract without altering existing graph structure (§9).

---

## 3. Core Model

### 3.1 Session

A **session** is one execution of a graph for one participant. A session MUST have:

| Field | Type | Description |
|---|---|---|
| `session_id` | string | Unique identifier for the execution. |
| `graph_ref` | string | Graph id and version being executed. |
| `seed` | integer | Seed governing all randomized decisions (§3.4). |
| `state` | State | Mutable scoped variables (§3.2). |
| `context` | Context | Read-only inline data attached to the conversation (§3.6). |
| `data_sources` | map | Resolved data-source bindings and credentials (§3.5). |
| `transcript` | list | Ordered record of user-facing turns. |
| `status` | enum | `active`, `completed`, `abandoned`, `escalated`, `limit_exceeded`. |

A session MAY be paused and resumed. On resume, `seed`, `state`, and `context` MUST be preserved.

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

Templates (for `prompt`/`message` fields) use `{{ state.<path> }}` interpolation. A runtime MUST treat missing references as an error unless a default is declared.

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
| `kind` | yes | Connector type (`postgres`, `http`, `vector`, or a namespaced custom kind). |
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

- **Effects**: writes `slot`.
- **Await**: `user`.
- **Requirements**:
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
| `prompt_template` | yes | Template with state/context interpolation. |
| `output_schema` | no | If set, output MUST validate against it. |
| `slot` | yes | State variable receiving the output. |
| `tools` | no | Tool references available to the model. |
| `temperature`, `max_tokens` | no | Generation parameters. |
| `timeout_s` | no | Timeout before firing the `timeout` edge. |

- **Effects**: writes `slot`.
- **Await**: `user` when `mode: conversational`; `none` when `mode: silent`.
- **Requirements**:
  - In `conversational` mode the node MUST emit a user-facing message and await the user's reply, writing the validated reply to `slot`.
  - In `silent` mode the node MUST NOT produce a user-facing turn and MUST NOT block.
  - When `output_schema` is present, output MUST validate before writing; on failure the `error` edge MUST be taken.
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
- **Ports**: `in`; `body` (enter loop body); `exit`.
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

#### `call`

- **Purpose**: Invoke a subgraph as a reusable unit (e.g., a shared consent flow).
- **Ports**: `in`; `out`; `error`.
- **Config**:
  - `graph_ref` (REQUIRED): subgraph id and version.
  - `state_isolation` (REQUIRED): `isolated` or `shared`.
  - `input_map` (OPTIONAL): parent state → child inputs.
  - `output_map` (OPTIONAL): child outputs → parent state.
- **Effects**: writes declared `output_map` targets.
- **Await**: inherited from the subgraph (MAY be `user`).
- **Requirements**:
  - The referenced subgraph MUST exist and be version-pinned (VR-13).
  - In `isolated` mode the child MUST NOT read or write parent state except through `input_map`/`output_map`.
  - Subgraph errors MUST propagate to the caller's `error` edge.
  - A `call` that can await `user` MUST NOT be placed in a forked silent stream.

### 4.5 State Family

#### `set`

- **Purpose**: Deterministic state mutation (assignment or expression evaluation).
- **Ports**: `in`; `out`.
- **Config**:
  - `assignments` (REQUIRED): list of `{slot, expression}` entries.
- **Effects**: writes each `slot`.
- **Await**: `none`.
- **Requirements**:
  - A `set` node MUST be pure with respect to the external world and MUST NOT perform I/O.
  - Expressions MUST be evaluated using the guard/expression grammar (§5.3).
  - All written slots MUST be declared in `effects.write`.

### 4.6 External Family

#### `fetch`

- **Purpose**: Query a declared Data Source and materialize its typed attributes into State.
- **Ports**: `in`; `out`; `error`; `timeout`.
- **Config**:

| Field | Required | Description |
|---|---|---|
| `source_ref` | yes | Data source id (§3.5). |
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
- Multi-port control nodes (`loop`) MUST specify the source port (`body` or `exit`) on each edge.
- Edges MUST connect an existing outbound port to an existing node (VR-4, VR-14).
- Edges MUST NOT carry weight attributes; weights belong to `sample` config (§4.4).

### 5.2 Edge Resolution Order

When multiple outbound edges could fire, the runtime MUST resolve in this order:

1. `error` (on failure) — highest precedence.
2. `timeout` (on timeout).
3. `conditional` in ascending `priority` (lower number first); ties broken by edge declaration order.
4. `default`.

Exactly one edge MUST be taken. If none applies and no `default` exists, the graph MUST fail validation (VR-9).

### 5.3 Guard and Expression Grammar

Guards and `set` expressions use the same grammar:

```ebnf
guard        = or_expr ;
or_expr      = and_expr , { "or" , and_expr } ;
and_expr     = not_expr , { "and" , not_expr } ;
not_expr     = [ "not" ] , primary ;
primary      = comparison
             | predicate_call
             | "(" , guard , ")" ;
comparison   = operand , comp_op , operand ;
comp_op      = "==" | "!=" | "<" | "<=" | ">" | ">=" | "in" | "matches" ;
operand      = arith ;
arith        = term , { ( "+" | "-" ) , term } ;
term         = factor , { ( "*" | "/" ) , factor } ;
factor       = number | string | boolean | "null"
             | ref
             | "(" , arith , ")" ;
ref          = "state." , identifier , { "." , identifier }
             | "context." , identifier , { "." , identifier } ;
predicate_call = identifier , "(" , [ operand , { "," , operand } ] , ")" ;
```

**Named predicates** are resolved from a graph-level `predicates` registry. A predicate MUST be declared before use; undeclared predicates MUST fail validation.

```yaml
predicates:
  risk_exceeds:
    args: [number]
    returns: boolean
    expr: "state.risk_score > args[0]"
```

- Guards MUST be side-effect free.
- A guard referencing an unset state variable MUST evaluate to `null`, and comparisons with `null` MUST be `false` (except `== null` / `!= null`). A reference to an undeclared `context.*` path MUST fail validation.

### 5.4 Cycles and Loop Guards

- Unguarded cycles are forbidden (VR-10).
- A cycle MUST pass through either a `loop` node or a `conditional` edge whose guard is proven to eventually terminate (e.g., bounded counter).
- The runtime MUST enforce `loop.max_iterations` and the graph-level `limits`.

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
| `limits` | no | Turn/time/token budgets (§7.4). |
| `seed` | no | Default session seed (§3.4). |
| `entry` | yes | MUST reference the single `start` node. |
| `on_error` | no | Fallback node for unhandled errors. |
| `nodes`, `edges` | yes | Graph content. |
| `terminals` | yes | All `end` nodes. |

### 6.2 Subgraphs

A `call` node composes graphs. The subgraph MUST expose declared inputs/outputs, and the caller MUST map them via `input_map`/`output_map`. Subgraph state is isolated unless `state_isolation: shared`. Subgraphs MUST be version-pinned so graphs remain reproducible.

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

---

## 7. Execution Semantics

### 7.1 Turn Model

- Execution proceeds node by node following edges.
- `await: none` nodes execute immediately, in sequence (or concurrently under `fork`).
- `await: user` nodes emit their output, open a turn, and suspend until a validated response or timeout.
- `await: external` nodes suspend until the external event or timeout.
- Silent nodes executed between turns are attributed to the upcoming turn's record.

### 7.2 Awaiting and Silence

- A `say` or `llm` in `silent` mode MUST NOT suspend.
- An `analyze` MUST NOT suspend; it writes a result or `default`.
- Only `ask`, `llm` (`conversational`), `handoff`, and `call` (inherited) MAY suspend.

### 7.3 Timeouts and Failure Resolution

- On timeout, the runtime MUST take the node's `timeout` edge if present; otherwise `error`; otherwise the graph's `on_error`; otherwise end with `limit_exceeded`.
- On model/tool/validation failure, the runtime MUST take `error` if present; otherwise `on_error`.
- Failure in a silent fork stream MUST NOT cancel sibling streams unless the graph declares `fail_fast`.

### 7.4 Budgets

When any `limits` value is exceeded, the runtime MUST terminate the current path at `on_error` or an `end` with outcome `limit_exceeded`. Budgets MUST be checked at turn boundaries.

### 7.5 Determinism and Replay

Given the same graph version, `seed`, and user inputs, all `sample` and `classify` (deterministic backends) decisions MUST be reproducible. Because `llm`/`analyze` outputs may vary, replays SHOULD pin recorded model outputs or accept divergence while preserving recorded `sample` choices.

---

## 8. Validation Rules

A graph MUST pass all rules below before execution. Rules are numbered for reference in tooling and examples.

- **VR-1** — Node ids MUST be unique within a graph.
- **VR-2** — Exactly one `start` node MUST exist; it MUST have no inbound edges.
- **VR-3** — Every non-`end` node MUST have at least one outbound transition (an edge, or a `sample` choice); every `end` node MUST have none.
- **VR-4** — Every edge endpoint MUST reference an existing node and an existing port on that node.
- **VR-5** — Every node MUST be reachable from `start`.
- **VR-6** — Every node MUST be able to reach at least one `end` node.
- **VR-7** — Every state reference MUST resolve in `state_schema`; every write MUST be declared in `effects.write`.
- **VR-8** — Every `ask` MUST declare a valid `input_type`, a `slot`, and a `response_schema` when `input_type: custom`.
- **VR-9** — Every `branch` MUST have a `default` edge or provably exhaustive guards.
- **VR-10** — Unguarded cycles are forbidden; every cycle MUST pass through a `loop` node or a terminating guarded edge.
- **VR-11** — `fork`/`join` MUST be balanced; forked streams MUST be entirely non-blocking; the graph MUST satisfy the single-await invariant (§6.4).
- **VR-12** — Every `sample` choice target MUST exist; weights MUST be positive; `record_slot` MUST be declared.
- **VR-13** — Every `call.graph_ref` MUST exist and be version-pinned.
- **VR-14** — Edge `type` MUST be valid for the source port (e.g., `conditional`/`default` only from control nodes).
- **VR-15** — Every `end` node MUST declare a valid `outcome`.
- **VR-16** — Every node that can fail or block SHOULD declare `error`/`timeout` edges; if absent, `on_error` MUST be defined.
- **VR-17** — Every `fetch.source_ref` MUST reference a declared data source, and every requested attribute MUST exist in that source's schema.
- **VR-18** — Context (`context.*`) is read-only: no node or `set` MAY write it, and every `context.*` reference MUST resolve in the supplied Context or `context_schema`.
- **VR-19** — Credentials MUST be referenced via `credentials_ref`; inline secrets in graph, config, or context are forbidden.

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

This section intentionally does not define a full plugin ABI, sandboxing model, or capability negotiation; those are deferred (§11).

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
    confidence:{ type: number, scope: session, mutable: true }
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

---

## 11. Open Issues

- **Serialization**: JSON, YAML, or a binary schema; canonical form and schema language.
- **Guard grammar finalization**: precedence edge cases, type coercion, and a standard predicate library.
- **Custom-node ABI**: full contract, sandboxing, capability negotiation, and whether to standardize on MCP as the transport adapter.
- **Multi-await forks**: whether to ever permit concurrent blocking user turns (currently forbidden by P5/VR-11).
- **Randomization breadth**: node-level `sample` is chosen for v1; revisit whether edge-level weights or deterministic assignment strategies (quota, A/B bucketing) are needed.
- **Streaming and partial responses**: token streaming, partial validation, and interruption semantics.
- **Localization and rendering**: message catalogs, RTL, and channel-specific rendering of `ask` input types.
- **Data sources**: connector taxonomy, schema drift, cache/consistency semantics, credential rotation, and row-level access control.
- **Context provenance**: versioning, size limits (context-window growth), redaction, and merge-depth rules for session-supplied Context.
- **Privacy and retention**: state/context encryption, PII handling, and transcript retention policy.
- **Versioning and migration**: graph migration, session compatibility across graph versions, and replay guarantees.
- **Testing harness**: conformance suite, deterministic replay tooling, and simulation of user inputs.
