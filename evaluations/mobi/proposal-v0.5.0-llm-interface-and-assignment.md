# Proposal: ConvoSpec v0.5.0 — LLM Interface, Conversation History, and Design Assignment

| Field | Value |
|---|---|
| Status | **Applied** — integrated into `spec.md` v0.5.0 (integration notes in [§14](#14-integration-notes)) |
| Target spec version | 0.5.0 |
| Baseline | `spec.md` v0.4.0 |
| Source evaluation | [`mobi-compliance.md`](./mobi-compliance.md) |
| Evidence base | [`mobi-interaction-inventory.md`](./mobi-interaction-inventory.md) (P01–P24) |
| Review input | User review of MoBI G1–G4 and questions 1–6, 2026-11-16 |

This document proposed changes to `spec.md` v0.4.0. They are now **applied in `spec.md` v0.5.0**; [§14](#14-integration-notes) records integration notes and every deviation from the text below. The body of this document is preserved as the reviewed proposal.

**Citation convention.** `§X` and `VR-N` refer to `spec.md` **v0.4.0**. References to this document are written `this §X`. Sections marked *(new)* are proposed spec sections that do not exist yet. `L<n>` refers to a line in [`MoBIxLLM_task_summary_for_GK_20261116.md`](../../MoBIxLLM_task_summary_for_GK_20261116.md).

---

## 1. Status and scope

The MoBI compliance evaluation ([`mobi-compliance.md`](./mobi-compliance.md) §5) found **no gap** and five partials, all in the model interface or in experimental-design assignment. The control-flow layer was already fully covered.

| Gap | Evaluation's finding | Disposition here |
|---|---|---|
| **G1** Design assignment | No permutation, no without-replacement draw, no `range`, no expression access to `session.seed` | **Accepted.** A `session.*` reference root plus four core collection functions (this §3) |
| **G2** Prompt composition | `llm` has one `prompt_template`; the manipulation *is* a system prompt (L29–L35) | **Accepted.** `messages: [{role, template}]`, with `prompt_template` kept as sugar (this §4) |
| **G3** Conversation state | No history semantics; the generated message is audited but unreadable | **Accepted.** A `history` scope on `llm` and a `message_slot` for the emitted text (this §5) |
| **G4** Subgraph termination | §6.2 never says what a subgraph `end` does to the caller | **Accepted.** A termination contract with a reserved `outcome` output (this §6) |

Four constraints shape everything below:

1. **Additive only.** No existing node, field, or validation rule changes meaning. Every v0.4.0 graph remains valid and unchanged in behaviour. The single exception is reserved-name pressure (§10.2), which is already the established policy from v0.4.0.
2. **No new node types.** G1 is functions plus one grammar root; G2–G4 are config fields on `llm` and `call`. The taxonomy stays flat.
3. **Core stays spec-owned, extensions stay declared.** The four new functions are added to §5.5 with stated semantics. Everything else an implementation's host language offers stays in the extension tier (§5.6).
4. **The manipulation must be inspectable.** Assignment, stance, and history scope are all values the graph declares, so the design is readable from the source (P1, P4). Nothing is inferred or computed out of band.

### 1.1 Summary of proposed changes

| Kind | Addition |
|---|---|
| Grammar (§5.3) | `session.` reference root; `session` joins the reserved roots |
| Core library (§5.5) | `range`, `shuffle`, `take`, `chunks`; `div`/`mod` rows added (currently referenced but unlisted); `get` overloads refined |
| `llm` (§4.3) | `messages` (alternative to `prompt_template`), `history`, `message_slot` |
| `call` / subgraphs (§4.4, §6.2) | Termination contract; reserved `outcome` output |
| Execution (§7.1, §7.5) | Conversation-scope rules; history is recorded, not re-derived |
| Validation (§8) | VR-33 … VR-36; VR-7 amended for `session.*` |

### 1.2 What does not change

- `analyze` and `classify` keep explicit `inputs`; neither gains history. History is a property of a *conversational turn*, and these nodes are silent by definition (§4.3).
- `sample` (§4.4) is untouched. It draws one edge per execution; it is not a permutation primitive and G1 does not try to make it one.
- `views`, `set`, and the core library's existing rows are unchanged.

---

## 2. Review decisions and their consequences

| # | Question | Decision | Consequence |
|---|---|---|---|
| 1 | Seed access (G1) | **New `session.*` reference root** | Grammar change; `session.seed` becomes an ordinary operand (this §3.2) |
| 2 | Assignment as functions or a node (G1) | **Functions + `set`** | The realized assignment is an ordinary state value, replayed by the existing `seed` rule (this §3.5) |
| 3 | History scope (G3) | **Default all prior interactions; scope narrowed by count or by explicit variables; `state_isolation` bounds it** | One `history` field with four shapes (this §5.3) |
| 4 | `message_slot` vs. redefining `slot` (G3) | **Additive `message_slot`** | `slot` keeps its meaning in both modes (this §5.2) |
| 5 | G2 shape | **`messages: [{role, template}]`** | Anticipates instruments that need multi-message prompts (this §4.2) |
| 6 | G4 | **Explicit `output_map` binding** | Matches how every other subgraph output crosses the boundary (this §6.2) |

---

## 3. G1 — Experimental-design assignment

### 3.1 Motivation

The design's integrity properties are balance and without-replacement sampling, both reproducible from the session seed (L48–L57). v0.4.0 can express neither:

- `sample` (§4.4) draws one edge per execution. It cannot produce a permutation or a block-balanced order.
- `set` and `views` are pure functions of State, Context, and other views. They cannot read the seed, so they cannot randomize at all.
- There is no range constructor, so a list whose length is a runtime parameter cannot be generated.

The current workaround pre-computes the assignment outside the graph and supplies it as `context`. That moves a design guarantee out of the inspectable graph (P1) and leaves the realized assignment unrecorded.

### 3.2 `session.*` references (§5.3)

The grammar's `ref` production gains a third root:

```ebnf
ref            = "state." , identifier , { "." , identifier }
               | "context." , identifier , { "." , identifier }
               | "views." , identifier
               | "session." , identifier ;
```

**Rule.** `session.<field>` MUST name a scalar field of the session record (§3.1): `seed`, `id`, `case_id`, `role`. It MUST NOT name `state`, `context`, `links`, `export`, `data_sources`, `transcript`, or `status` — those are either non-scalar or already reachable by another root. Session references are read-only; no node MAY write one (VR-33).

`session` joins `state`, `context`, and `views` in the reserved-roots list of §5.3, so a predicate or extension cannot shadow it.

**Why a root and not a function.** `seed()` would be a function that reads runtime state rather than its arguments — the one thing §5.5 forbids of the core tier. A reference root keeps the rule intact: the *seed* is data, and every function that consumes it stays pure.

**Determinism.** `session.seed` is an input to the expression, not a source of hidden nondeterminism. Two runs with the same seed evaluate identically, which is exactly the guarantee §3.4 already gives `sample`.

### 3.3 Core function additions (§5.5)

| Function | Signature | Semantics |
|---|---|---|
| `range` | `range(end) -> list<integer>`; `range(start, end, step?) -> list<integer>` | Half-open integer range; `step` defaults to `1` and MUST be non-zero; negative steps count down. |
| `shuffle` | `shuffle(list<T>, seed) -> list<T>` | Deterministic permutation. Same list and seed MUST yield the same order within an implementation. |
| `take` | `take(list<T>\|string, n) -> same` | The first `n` elements; `n` clamped to `[0, len]`. |
| `chunks` | `chunks(list<T>, size) -> list<list<T>>` | Consecutive blocks of `size`; the last block MAY be short. `size` MUST be ≥ 1. |
| `div` / `mod` | `div(integer, integer) -> integer`; `mod(integer, integer) -> integer` | Truncating division and remainder, as already specified in the numeric-semantics paragraph. |

`div` and `mod` are not new behaviour: §5.5's numeric-semantics paragraph already fixes their semantics, but they were missing from the function table. This proposal adds the rows so the table matches the prose. The grammar has no `%` operator, so `mod` is the only way to express a remainder.

`shuffle`'s determinism is **property-based, not algorithm-based**: the specification requires stability and independence from the host RNG and from host iteration order, but does not pin a PRNG. This matches `sample` (§3.4), which also guarantees reproducibility from `seed` and node id without naming an algorithm. The consequence is deliberate and stated: two conforming implementations MAY produce different permutations for the same seed, so the realized assignment MUST be written to state and is the authoritative record. See this §12, question 1.

### 3.4 Supporting amendments

**`get` overloads.** The current signature returns `any`, which makes `get(pool_list, index)` unusable as an argument to `shuffle` under VR-26. Refine to:

| Function | Signature |
|---|---|
| `get` | `get(list<T>, integer, T?) -> T`; `get(object, string, any?) -> any` |

This is a typing refinement, not a behaviour change: every currently-valid call keeps its meaning, and more calls type-check.

### 3.5 Worked sketch — the MoBI assignment

Narrow sketch, not a full graph. State is declared per pool so the element types are known:

```yaml
state_schema:
  cells:          { type: list<object>, scope: session, mutable: true }
  assignment:     { type: list<object>, scope: session, mutable: true }
  pool_inf:       { type: list<string>, scope: session, mutable: true }
  pool_dec:       { type: list<string>, scope: session, mutable: true }
  pool_ref:       { type: list<string>, scope: session, mutable: true }
  story_plan:     { type: list<object>, scope: session, mutable: true }
  reps:           { type: integer, scope: session, mutable: true }
  trials_per_type: { type: integer, scope: session, mutable: true }
```

```yaml
- id: build_cells
  type: set
  effects: { write: [cells] }
  config:
    assignments:
      - slot: cells
        expression: >-
          map(range(18), {
            position: get(['INF', 'DEC', 'REF'], mod(index, 3)),
            stance: get(['aligning', 'calibrated', 'counterbalancing'], mod(div(index, 3), 3)),
            question: mod(index, 2) == 0
          })

- id: build_assignment
  type: set
  effects: { write: [assignment] }
  config:
    assignments:
      - slot: assignment
        expression: "flatten(map(range(state.reps), shuffle(state.cells, session.seed + index)))"

- id: build_stories
  type: set
  effects: { write: [story_plan] }
  config:
    assignments:
      - slot: story_plan
        expression: >-
          map(['INF', 'DEC', 'REF'], {
            type: item,
            stories: take(
              shuffle(get([state.pool_inf, state.pool_dec, state.pool_ref], index), session.seed + index),
              state.trials_per_type)
          })
```

What this buys:

- **Balance.** `cells` is the complete 18-cell design, built once from constants. `assignment` repeats it `reps` times and shuffles *within* each block, so every block is a full counterbalanced set (L57).
- **Without replacement.** `shuffle` + `take` gives each participant a distinct subset of each type's pool, and no story repeats within a participant (L50–L51).
- **Inspectability.** Both results are ordinary state values. `assignment` is the realized order; it is written to state, so a run can be audited without re-deriving it (P1, P4).
- **Reproducibility.** Same graph, same `session.seed`, same pools → same assignment, subject to this §3.3's property-based determinism.

### 3.6 Why functions and not a node

A `randomize` node would be a new node type, which the taxonomy has deliberately avoided since v0.1.0, and it would need ports, an error path, and its own validation rules to do what four pure functions do. More importantly, a node would put the assignment *in the control flow* rather than in state, and the design's properties would then be a side effect of node placement rather than a value an analyst can read.

The functions also compose with what already exists: `flatten(map(…))` is block shuffling, `take(shuffle(…))` is sampling without replacement, and neither needs new syntax.

### 3.7 Rejected alternatives

| Alternative | Why rejected |
|---|---|
| `seed()` function | A core function that reads runtime state rather than its arguments breaks the purity rule of §5.5. |
| Seed supplied only as `context` | Zero spec change, but the graph cannot generate its own design; the caller must pre-compute it, which is the status quo being fixed. |
| `%` and `//` operators | Operators are grammar; `div`/`mod` already exist in prose and need only table rows. Adding operators would extend the grammar for two functions' worth of value. |
| A `permute_by(list, keys)` primitive | Equivalent to `sort(list, key)` with a hash function, which needs `hash` — a larger addition for no more expressive power. |
| Pinning a PRNG in the spec | Would give cross-implementation identical permutations at the cost of freezing an algorithm in a draft spec. Deferred as this §12, question 1. |

---

## 4. G2 — Prompt composition

### 4.1 Motivation

Stance is *defined* as a system-prompt instruction (L29–L35), and the base prompt is a large fixed document (L86–L88). v0.4.0 gives `llm` a single `prompt_template`, so both must be flattened into one string. That changes the thing being studied: the model no longer sees a system message distinct from the user's story and turn.

### 4.2 `llm.messages` (§4.3)

```yaml
- id: bot_1
  type: llm
  effects: { write: [bot_reply_1, reply_1] }
  config:
    model: sonnet-class
    mode: conversational
    messages:
      - { role: system, template: "{{ context.base_system }}" }
      - { role: system, template: "{{ state.stance_instruction }}" }
      - { role: user, template: "{{ state.story_text }}" }
    message_slot: bot_reply_1
    slot: reply_1
```

| Field | Required | Description |
|---|---|---|
| `messages` | conditional | Ordered list of `{role, template}`. Exactly one of `messages` or `prompt_template` MUST be declared. |
| `prompt_template` | conditional | Retained as sugar for a single `user` message. |

`role` is one of `system`, `user`, `assistant`. Rules:

- At least one message MUST be declared, and at least one MUST be `user` or `assistant`.
- All `system` messages MUST precede all non-system messages. An implementation MAY reject a system message that appears after a user or assistant message; the validator MUST reject it (VR-34).
- `template` uses the same interpolation as `prompt_template` (§3.2): `{{ state.* }}`, `{{ context.* }}`, `{{ views.* }}`, and `{{ session.* }}`.
- `prompt_template: T` is exactly equivalent to `messages: [{role: user, template: T}]`. A graph MUST NOT declare both.

### 4.3 Ordering with history

When `history` is present (this §5), the assembled prompt is:

1. leading `system` messages, in declaration order;
2. the history block;
3. the remaining `messages`, in declaration order.

With `prompt_template`, step 1 is empty and step 3 is the single user message. This rule is why system messages must be leading: it gives history an unambiguous insertion point that does not depend on the author's ordering of user and assistant messages.

### 4.4 Backwards compatibility

Every v0.4.0 graph uses `prompt_template`, which is unchanged and still means one user message. The only new failure mode is declaring both fields, which no existing graph does.

---

## 5. G3 — Conversation history and message addressability

### 5.1 Motivation

Two separate omissions, both visible in the trial:

1. **History.** §4.3 requires a `conversational` `llm` to emit exactly one message and await exactly one reply, and says nothing about whether the model sees prior turns. The second bot reply must see the story, the pre-chat responses, and both prior chat messages (L72, L86).
2. **Addressability.** `llm.slot` receives the *participant's reply*. The generated message is recorded for audit (§4.3) but is not a readable state value, so `classify.inputs` cannot inspect the text that was sent — which is what the crisis rule requires (L82).

### 5.2 `llm.message_slot` (§4.3)

| Field | Required | Description |
|---|---|---|
| `message_slot` | no | `string` state variable receiving the exact text emitted to the user. Valid only when `mode: conversational`. |

- When declared, the node writes both `slot` (the participant's validated reply) and `message_slot` (the generated message). Both MUST appear in `effects.write`.
- `message_slot` MUST be a declared `string` state variable and MUST differ from `slot` (VR-35).
- `silent` mode is unchanged: `slot` receives the model output, and `message_slot` MUST NOT be declared.

**Why not redefine `slot`.** Making `slot` mean "the generated message" in conversational mode would be a breaking change to every existing `llm` node, and would remove the only place the participant's reply is stored. Two slots cost one field and preserve both meanings.

**Why not read the transcript instead.** §3.1 records a `transcript`, but it is a session record, not a state reference, and `classify.inputs` takes state references (§4.3). Adding a `transcript.*` root would create a second, parallel read path for values that belong in state. One slot is smaller.

### 5.3 `llm.history` (§4.3, §7.1)

```yaml
history: all                      # default
history: none                     # suppress entirely
history: { last: 4 }              # last 4 interactions
history:
  from:                           # explicit, ordered, template-based
    - { role: user, template: "Story: {{ state.story_text }}" }
    - { role: user, template: "Initial choice: {{ state.choice_pre }}" }
```

| Shape | Meaning |
|---|---|
| `all` (default) | Every prior interaction in the visible conversation scope, in order. |
| `none` | No history. |
| `{ last: N }` | The last `N` interactions in scope. `N` MUST be a non-negative integer; `0` is equivalent to `none`. |
| `{ from: [...] }` | The listed `{role, template}` entries, in the given order. |

`last` and `from` MAY be combined; when both are present the `from` entries are rendered first, then the selected interactions.

**What "interaction" means.** One interaction is one user-facing exchange: the participant's message and the reply that followed it. It is the unit the transcript already records (§3.1), so `last: 4` is "the four most recent exchanges", not "four messages".

**What the default renders.** Each recorded interaction becomes one `user` message (the participant's text) followed by one `assistant` message (the generated text). Nothing else is included: arbitrary state is not serialized into the prompt by default, because there is no canonical way to render a nested object as prose. An author who wants a specific value in the prompt says so with `from`, which reuses the template mechanism of §3.2 rather than inventing a serialization rule.

**Why `from` is template-based.** A plain `from: [state.story_text]` would need a rendering rule for every type — verbatim strings, `to_string` for scalars, and something undefined for lists and objects. Templates already exist, already handle every type the author can name, and let the author label the value ("Story: …") instead of relying on an inferred one.

### 5.4 Interaction with `state_isolation`

The **visible conversation scope** is what a node can see, and `call.state_isolation` (§4.4) is what sets it:

| Caller setting | Child's visible scope | Effect on `history: all` |
|---|---|---|
| `state_isolation: isolated` | The child's own state and its own transcript | Only the child's own interactions; parent turns are not in scope |
| `state_isolation: shared` | The parent's state and transcript | The whole session |

This is the "override" the review asked for, expressed as a consequence rather than a special case: isolation already means "the child MUST NOT read parent state" (§4.4), so a child cannot read the parent's conversation either. `history: none` overrides everything, and `from` references MUST resolve in the visible scope (VR-35).

For MoBI this is the right default: each trial is an isolated subgraph, so a trial's chat starts empty and cannot leak the previous trial's conversation, while `output_map` carries the trial record back out.

### 5.5 Worked sketch — the MoBI chat with a crisis check

Narrow sketch. The two bot replies and the stop rule become four nodes instead of eight, and the check can see the text that was sent:

```yaml
- id: bot_1
  type: llm
  effects: { write: [bot_reply_1, reply_1] }
  config:
    model: sonnet-class
    mode: conversational
    messages:
      - { role: system, template: "{{ context.base_system }}" }
      - { role: system, template: "{{ state.stance_instruction }}" }
      - { role: system, template: "{{ state.question_instruction }}" }
      - { role: user, template: "{{ state.story_text }}" }
    history:
      from:
        - { role: user, template: "Initial choice: {{ state.choice_pre }} (certainty {{ state.certainty_pre }})" }
        - { role: user, template: "My reason: {{ state.justification }}" }
    message_slot: bot_reply_1
    slot: reply_1

- id: crisis_check_1
  type: classify
  effects: { read: [bot_reply_1], write: [crisis_label] }
  config:
    backend: rules
    model: crisis_rules
    inputs: [state.bot_reply_1]
    labels: [safe, crisis]
    slot: crisis_label

- id: crisis_gate_1
  type: branch

- id: bot_2
  type: llm
  effects: { write: [bot_reply_2, reply_2] }
  config:
    model: sonnet-class
    mode: conversational
    messages:
      - { role: system, template: "{{ context.base_system }}" }
      - { role: system, template: "{{ state.stance_instruction }}" }
    history: all
    message_slot: bot_reply_2
    slot: reply_2
```

`bot_2` uses `history: all`, which renders the story exchange, both bot messages, and the participant's first reply in order — exactly the disclosure the trial table specifies (L72, L86). The crisis gate routes on `state.crisis_label` and, on `crisis`, takes a `say` of the unchanged message followed by an `end` whose outcome is bound by the caller (this §6).

### 5.6 Determinism and replay

History is **assembled from recorded values**, not re-derived. Both components — the transcript and any `from` template — are already recorded: the transcript by §3.1, the template inputs by the state snapshot. §7.5's existing rule that evaluated values used in a decision or message MUST be recorded therefore extends to the assembled prompt. This proposal adds one sentence to §7.5 saying so, so a replay never reconstructs a prompt from a different state snapshot.

`history` has no effect on which node runs next, so it cannot change control flow. It changes only what the model sees, which is already the case for `prompt_template`.

### 5.7 Rejected alternatives

| Alternative | Why rejected |
|---|---|
| Session-wide history always on | Silently leaks one trial's chat into the next; the MoBI design requires per-trial isolation (P22). |
| History scoped only to the current subgraph invocation | The review asked for the default to be broader: all prior interactions, bounded by isolation. |
| A separate `conversation` node or block | History is a property of a turn, not a structure. A block would need lifecycle and clearing rules that isolation already provides. |
| `transcript.*` reference root | A second read path for values that belong in state; `message_slot` is smaller and reuses `classify.inputs`. |
| Plain `from: [refs]` with implicit rendering | Needs a serialization rule for every type, including nested objects. Templates already exist and label the value. |

---

## 6. G4 — Subgraph termination contract

### 6.1 Motivation

§6.2 defines subgraph inputs, outputs, isolation, and version pinning, but never states what a subgraph's `end` does to its caller. MoBI needs it: the crisis rule ends a trial before step 14 (L82), and the parent loop must continue with the next trial and know why this one stopped.

### 6.2 The rule (§6.2)

Three sentences, added to §6.2:

1. A subgraph `end` concludes **the subgraph** and returns control to the caller through the `call` node's `out` port. It MUST NOT terminate the parent session.
2. Every subgraph exposes one **reserved output**, `outcome`, whose value is the `outcome` of the `end` that terminated it. A subgraph MUST NOT declare an output named `outcome`.
3. `output_map` MAY bind `outcome` like any other output. The target MUST be a declared `enum` or `string` state variable (VR-36).

Control flow on the error path is unchanged: a subgraph that ends via `on_error` takes the caller's `error` port, and `outcome` is not bound.

### 6.3 Worked sketch

```yaml
- id: run_trial
  type: call
  effects: { write: [trial_outcome, trial_record] }
  config:
    graph_ref: { id: mobi_trial, version: 0.1.0 }
    state_isolation: isolated
    input_map:
      story_text: state.current_story_text
      stance_instruction: state.current_stance_instruction
    output_map:
      outcome: state.trial_outcome
      trial_record: state.last_trial_record
```

The trial subgraph ends at one of two nodes, `done` (`completed`) or `crisis_end` (`escalated`). The parent gets the outcome as a value and can branch on it — for example, to pause the trial loop and show the crisis page once rather than per trial.

### 6.4 Rejected alternatives

| Alternative | Why rejected |
|---|---|
| Status slot written before each `end` | The current workaround: every terminal path must remember to write it, and forgetting is silent. |
| A `call.outcome_slot` field | A second binding mechanism for the same thing `output_map` already does; two ways to express one boundary is how specs rot. |
| A subgraph `end` terminating the session | Makes early termination unable to return control, which is the whole problem. |

---

## 7. Consolidated spec change list

### 7.1 Sections to amend

| Spec section | Change |
|---|---|
| §3.1 Session | Note that scalar session fields are readable as `session.<field>` in expressions. |
| §3.4 Seed | Cross-reference `session.seed` as the expression-level accessor for the same seed. |
| §4.3 `llm` | Add `messages`, `history`, `message_slot`; state the assembly order. |
| §4.4 `call` | Cross-reference the termination contract. |
| §5.3 Grammar | `session.` root in `ref`; `session` joins the reserved roots. |
| §5.5 Library | Add `range`, `shuffle`, `take`, `chunks`, `div`, `mod`; refine `get` overloads. |
| §6.2 Subgraphs | Termination contract; reserved `outcome` output. |
| §7.1 Turn model | Visible conversation scope; history assembly. |
| §7.5 Replay | The assembled prompt is recorded, not re-derived. |
| §8 Validation | Add VR-33 … VR-36; amend VR-7. |
| §10 Examples | One example combining `messages`, `history`, `message_slot`, and a bound `outcome`. |

### 7.2 Proposed validation rules (numbering provisional)

- **VR-33** — Every `session.*` reference MUST name a scalar field of the session record (§3.1); no node MAY write one; `session` MUST NOT be redeclared as a predicate or extension name.
- **VR-34** — An `llm` MUST declare exactly one of `prompt_template` or `messages`; `messages` MUST contain at least one entry, at least one of which is `user` or `assistant`; every `role` MUST be `system`, `user`, or `assistant`; every `system` message MUST precede every non-system message; every template MUST resolve.
- **VR-35** — `llm.history` MUST be `all`, `none`, `{last: N}` with `N` a non-negative integer, or `{from: [...]}` of `{role, template}` entries; every `from` reference MUST resolve in the node's visible conversation scope; `message_slot` MUST be a declared `string` state variable, MUST differ from `slot`, MUST be declared in `effects.write`, and MUST NOT appear when `mode: silent`.
- **VR-36** — Every subgraph exposes a reserved `outcome` output and MUST NOT declare an output of that name; a subgraph `end` MUST return control through the caller's `out`; an `output_map` binding for `outcome` MUST target a declared `enum` or `string` state variable.

### 7.3 Existing rules affected

| Rule | Change |
|---|---|
| VR-7 | Extended to say `session.*` references resolve against §3.1's session record rather than `state_schema`. |
| VR-26 | Unchanged in form; the new core function names join the reserved set automatically, since §5.5 names are already reserved. |
| VR-13 / VR-22 | Unchanged; the termination contract is orthogonal to version pinning. |

### 7.4 Grammar diff (summary)

```
ref            += | "session." , identifier
reserved roots  += session
```

No other production changes. `messages`, `history`, and `message_slot` are config fields, not syntax.

---

## 8. Determinism and inspectability impact

| Principle | Effect |
|---|---|
| **P1 Declarative** | Improved: assignment, stance, and history scope are all values in the graph source. Nothing is pre-computed outside it. |
| **P3 Deterministic control** | Unchanged: `history` cannot change routing; `session.seed` is an input, not hidden state. |
| **P4 Enumerable** | Unchanged: no dynamic dispatch is added. |
| **§3.4 / §7.5 Replay** | `shuffle` is reproducible within an implementation and its result is written to state; the assembled prompt is recorded. |
| **§5.6 Portability** | Weakened in one narrow, stated way: `shuffle` is property-based, so two implementations MAY permute differently for the same seed. See this §12, question 1. |

---

## 9. Traceability (MoBI)

| Pattern | v0.4.0 | After this proposal | Basis |
|---|---|---|---|
| P02 Balanced randomized assignment | Partial | **Covered** | `session.seed`, `range`, `shuffle`, `flatten`, `map` (§3) |
| P03 Story sampling without replacement | Partial | **Covered** | `shuffle` + `take` (§3.3) |
| P11 Stance-conditioned reply | Partial | **Covered** | `llm.messages` with `system` roles (§4) |
| P13 Second reply with history | Partial | **Covered** | `llm.history: all` (§5.3) |
| P15 Crisis stop rule | Partial | **Covered** | `message_slot` + `classify` + `branch` (§5.2) |
| P23 Early trial termination | Partial | **Covered** | Subgraph termination contract + `outcome` binding (§6) |
| All other patterns | Covered | Covered | Additive changes; no construct changed meaning |

**Projected: Covered 24 · Partial 0 · Gap 0**, and the instrument verdict moves from *Sufficient with reservations* to **Sufficient**.

Two honest caveats, neither of which affects the verdict:

- The assignment encoding (this §3.5) is one dense expression rather than a sequence of readable steps. It is inspectable, but an author will want the inventory's prose next to it. If that proves painful, a `views` decomposition into named intermediate lists is the remedy, not new syntax.
- `shuffle`'s cross-implementation portability is weaker than the rest of the core tier. MoBI does not need cross-implementation identity — a study runs on one implementation and records the realized assignment — but the core tier's promise should be stated accurately rather than glossed.

---

## 10. Backwards compatibility and migration

### 10.1 Source compatibility

Every v0.4.0 graph remains valid:

- `prompt_template` is unchanged and still means one `user` message.
- `history` defaults to `all`, but with no prior interactions in scope the assembled prompt is byte-identical to v0.4.0's. A graph whose `llm` follows an earlier `conversational` `llm` *will* now see that exchange, which is a behaviour change in the narrow sense that v0.4.0 left undefined. This is the one place where "additive" needs qualification, and it is stated in §7.1.
- `message_slot`, `messages`, `history`, and the `outcome` binding are all opt-in.

### 10.2 Reserved names

`session` joins the reserved roots, and `range`/`shuffle`/`take`/`chunks`/`div`/`mod` join the reserved core function names. A predicate or extension using any of these names now fails VR-26. No such name exists in this repository. This is the same pressure v0.4.0 introduced and should be called out in the revision notes.

### 10.3 Migration

None required. Authors who want the new behaviour add fields; authors who do not, change nothing.

---

## 11. Implementation plan (proposed order)

1. **Grammar + seed access** (§5.3, §3.1, §3.4): `session.` root, reserved roots, VR-33.
2. **Core functions** (§5.5): `range`, `shuffle`, `take`, `chunks`, `div`, `mod`, `get` overloads.
3. **`llm` config** (§4.3): `messages`, `message_slot`, `history`; assembly order; VR-34, VR-35.
4. **Conversation scope** (§7.1, §7.5): visible scope, history assembly, replay recording.
5. **Subgraph termination** (§4.4, §6.2): contract and `outcome`; VR-36.
6. **Examples and §11** (§10, §11): one combined example, open-issue updates, version bump.

Steps 1–2 are independent of 3–5. Step 4 depends on 3. Step 5 is independent of everything.

---

## 12. Decisions and remaining questions

**Settled in review, 2026-11-16:**

| # | Decision |
|---|---|
| 1 | `session.*` reference root, not a `seed()` function. |
| 2 | Assignment as pure functions plus `set`, not a new node. |
| 3 | History defaults to all prior interactions; scope narrowed by count (`last`) or by explicit variables (`from`); `state_isolation` bounds the scope. |
| 4 | Additive `message_slot`; `slot` keeps its meaning. |
| 5 | `messages: [{role, template}]` for G2, anticipating future instruments. |
| 6 | G4 via explicit `output_map` binding of a reserved `outcome` output. |

**Remaining questions:**

1. **`shuffle` portability.** Accept property-based determinism (same implementation, same inputs → same order; the realized assignment is recorded), or pin a PRNG and algorithm so two implementations agree? Pinning is stronger but freezes an algorithm into the spec; the rest of the core tier is algorithm-free.
2. **`history` rendering of non-transcript values.** This proposal uses `{role, template}` entries, reusing the template mechanism. The alternative is plain references with a specified serialization. Confirm templates are preferred.
3. **`session.*` field set.** Proposed: `seed`, `id`, `case_id`, `role`. Is anything else needed, or should the set stay minimal until an instrument asks?
4. **`outcome` on the error path.** Proposed: not bound when a subgraph ends via `on_error`. Alternatively bind `on_error`'s outcome too. Confirm.
5. **`last: N` counting.** Proposed: N counts *interactions* (participant message + reply), matching the transcript's unit. The alternative is counting messages, which makes `last: 1` mean half an exchange.
6. **Version and scope.** Target 0.5.0, applied in the §11 order after review? Any item to defer?

---

## 13. Explicitly not proposed

| Not proposed | Reason |
|---|---|
| A `randomize` or `assignment` node | This §3.6: it puts a design property in control flow instead of state, and needs ports and rules to do what four functions do. |
| A pinned PRNG in the core spec | This §12, question 1. Deferred, not rejected. |
| A `transcript.*` reference root | This §5.7: a second read path for values that belong in state. |
| History on `analyze`/`classify` | They are silent by definition and take explicit `inputs` (§4.3). |
| A conversation or memory block in the graph | Isolation already provides the lifecycle; a block would add clearing rules for no gain. |
| Multi-message *output* from `llm` | `conversational` still emits exactly one message (P8, §4.3). `messages` composes the input, not the output. |
| Serialization or rendering of arbitrary state into prompts | This §5.3: templates let the author name and label what goes in. |

---

## 14. Integration notes

Applied in `spec.md` v0.5.0, following the order in §11 (grammar + seed access → core functions → `llm` config → conversation scope → subgraph termination → examples and §11). Sections 3–13 landed as proposed, with the deviations below.

### 14.1 Deviations and additions

| # | Change | Reason |
|---|---|---|
| 1 | **§1 gains a goal bullet** for experimental designs | Not in the proposal. The revision's largest addition is design assignment; without a stated goal it reads as an expression-library feature rather than a capability. |
| 2 | **`div`/`mod` added as table rows** (§5.5) | The proposal flagged that the numeric-semantics paragraph already fixed their behaviour but the table omitted them. Integrated as rows so the table and prose agree. |
| 3 | **"Permutation determinism" paragraph formalized** (§5.5) | The proposal stated the property in §3.3 and left it to prose. It is now normative text beside the function table, because it is the one place the core tier's portability promise is weaker. |
| 4 | **§3.1 gains an explicit session-reference rule** listing which fields are readable | The proposal said only "note that scalar fields are readable". Enumerating the scalar set and excluding the non-scalar fields is what VR-33 checks, so the two must agree in one place. |
| 5 | **The `history` shape table lives in §7.1, not §4.3** | §4.3 is a node contract; §7.1 owns execution semantics. §4.3 carries the field and points at §7.1, which holds the assembly order, the four shapes, and the definition of an "interaction". |
| 6 | **`history` is not mode-restricted** (§4.3, VR-35) | The proposal restricted `message_slot` to `conversational` but said nothing about `history`. Integrated text restricts only `message_slot`: a `silent` `llm` may assemble history too, which avoids a validation rule for no safety gain. |
| 7 | **§10.9 also demonstrates the assignment expression** | The proposal's §7.1 asked for "one example combining `messages`, `history`, `message_slot`, and a bound `outcome`". Adding the `session.seed` assignment makes the example cover all four gaps and keeps §10 from needing a second example. |
| 8 | **The revision notes state the newly-defined behaviour** | The proposal §10.1 raised `history: all` as the one non-additive behaviour. The spec header now says so, so a reader comparing versions sees it without reading this document. |
| 9 | **§11 rewritten in three places** | "Randomization breadth" was amended, and "Permutation algorithm", "History rendering", and "Session reference set" were added, carrying the proposal's §12 questions forward as recorded issues rather than live blockers. |

### 14.2 Decisions and open questions

All six review questions are settled and implemented: `session.*` root, functions-not-a-node for assignment, interaction-counted history with `state_isolation` bounding the scope, additive `message_slot`, `messages: [{role, template}]`, and explicit `outcome` binding.

| §12 question | Decision | `spec.md` §11 entry |
|---|---|---|
| 1 `shuffle` portability | Property-based determinism | Permutation algorithm |
| 2 `from` rendering | `{role, template}` entries | History rendering |
| 3 `session.*` field set | `seed`, `id`, `case_id`, `role` | Session reference set |
| 4 `outcome` on the error path | Not bound | — (stated in §6.2) |
| 5 `last: N` counting | Interactions, not messages | — (stated in §7.1) |
| 6 Version and order | 0.5.0, §11 order | — |

### 14.3 Verification performed

- All 26 YAML blocks in `spec.md` parse (PyYAML), including the two added in §10.9.
- VR-1…VR-36 are defined exactly once each, in order, with no duplicates or gaps.
- Every `§` reference and internal anchor resolves in the revised spec; every relative link resolves.
- No tabs or trailing whitespace in `spec.md` or this document.
- Traceability in §9 matches [§8 of the compliance evaluation](./mobi-compliance.md#8-post-integration-re-evaluation-v050).
