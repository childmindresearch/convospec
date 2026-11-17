# Open-Ended Walrus Conversation — ConvoSpec Compliance Assessment

| Field | Value |
|---|---|
| Instrument | Open-ended free conversation with a phrase and turn-cap terminator |
| Source | Task statement (user, 2026-11-16) |
| Evidence base | [`walrus-interaction-inventory.md`](./walrus-interaction-inventory.md) (P01–P14) |
| Baseline | `spec.md` **v0.5.0** |
| Date | 2026-11-16 |

---

## Verdict

**Sufficient with reservations.** ConvoSpec v0.5.0 encodes the whole shape of this instrument — a seeded random opener, a bounded loop of single-exchange model turns, and two terminators — without a new node type. Twelve of fourteen patterns are **Covered**.

The two partials are at the *seams* of the conversation, not in its control flow:

1. **Continuity across a non-model turn.** The opener is a fixed prompt, so it is naturally an `ask`. §7.1 defines history rendering only for `llm` exchanges, so whether — and how — the opener reaches later prompts is unspecified (G1).
2. **Literal phrase matching.** The grammar lists a `matches` operator but never defines it, and the core library has no `contains` or case-normalization. Exact whole-message `==` works; the robust reading of "utters the phrase" does not (G2).

**Covered 12 · Partial 2 · Gap 0.** Neither partial blocks delivery; both have workarounds that are brittle or verbose. Details in [§5](#5-gaps-and-observations).

*Re-scored against `spec.md` v0.6.0 after integration: both partials become Covered — **Covered 14 · Partial 0 · Gap 0**, verdict **Sufficient** ([§8](#8-post-integration-re-evaluation-v060)).*

---

## 1. Scope and method

**Assessed**: one session — seeded opener selection, unbounded chat, phrase and turn-cap termination — as stated in the source.

**Excluded by the source**: bot persona and content policy, model choice, UI, and any storage of the transcript outside the session.

**Method**: each inventory pattern was mapped to node types (§4), edges and expressions (§5), composition (§6), execution semantics (§7), and validation rules (§8) of `spec.md` v0.5.0. The §11 open issues are treated as *unavailable*.

**Verdict scale**: **Covered** (encodable directly) · **Partial** (encodable with a workaround that is brittle, verbose, or non-declarative) · **Gap** (not encodable).

---

## 2. Part A — Required interactions and nodes

| Need | Node types / constructs |
|---|---|
| One session, seeded | `session.seed` (§3.1, §5.3) |
| Random opener from a pool | `set` + `shuffle` + `get` (§4.5, §5.5) |
| Opening prompt and first reply | `ask` (`text`) (§4.2) |
| Free model turn, one exchange | `llm` (`conversational`) (§4.3) |
| Conversation continuity | `llm.history: all`; conversation scope (§4.3, §7.1) |
| Unbounded chat with two exits | `loop` with `exit_when` + `max_iterations` (§4.4) |
| Turn accounting | `loop.counter_slot` (§4.4) |
| Phrase test | edge guard / `branch` (§5.3, §4.4) |
| Termination-reason record | `branch` + `set` + `end` (§4.4, §4.5, §4.1) |
| Abandonment and cost bounds | `timeout` edges; graph `limits` (§4.2, §4.3, §7.4) |
| Session close and export | `end`; `session.export` (§4.1, §3.7) |

---

## 3. Part B — Sufficiency assessment

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Session identity and seed | Covered | §3.1 session record (`session_id`, `seed`); `session.seed` readable in expressions (§5.3, VR-33). |
| P02 Random opener selection | Covered | `get(shuffle(context.openers, session.seed), 0)` in a `set` node (§4.5, §5.5). Deterministic per seed and independent per session. |
| P03 Opening turn | Covered | `ask` (`text`) with the chosen prompt interpolated into `prompt` (§4.2, VR-8). |
| P04 Open-ended bot turn | Covered | `llm` (`conversational`): emits exactly one message and awaits exactly one reply, writing it to `slot` (§4.3, P8). |
| P05 Conversation continuity | **Partial** | `llm.history: all` carries prior `llm` exchanges (§4.3, §7.1), but §7.1 defines rendering only for model exchanges. The opener is an `ask`, whose rendering is unspecified, so the model may not see the opener it is continuing (G1). |
| P06 Unbounded continuation | Covered | `loop` with `exit_when` and `max_iterations`; one `llm` node as the body (§4.4, VR-24). |
| P07 Turn accounting | Covered | `loop.counter_slot` tracks iterations (§4.4). Counts loop turns; the opener is separate (Q1). |
| P08 Termination-phrase detection | **Partial** | Whole-message exactness via `==` is available, but the `matches` operator in the grammar has no semantics (§5.3) and the core library has no `contains` or case functions (§5.5). Substring or case-insensitive matching is not expressible (G2). |
| P09 Phrase-triggered termination | Covered | `branch` on the phrase guard → `end` (§4.4, §4.1). Conditional on the guard being expressible (G2). |
| P10 Turn-cap termination | Covered | `loop.max_iterations` takes the `exit` port at the cap (§4.4). |
| P11 Termination-reason recording | Covered | Post-exit `branch` on `state.last_reply` / `counter_slot`, then `set` a reason slot before `end` (§4.4, §4.5). Ordering caveat when both conditions hold (Q1). |
| P12 Participant abandonment | Covered | `ask`/`llm` `timeout` edges to an `abandoned` end; graph `limits.max_wall_clock_s` (§4.2, §4.3, §7.4). |
| P13 Session close and transcript export | Covered | `end` (§4.1); `session.export` allowlist (§3.7, VR-21). |
| P14 Safety and cost bounds | Covered | Graph `limits` (`max_turns`, `max_wall_clock_s`, `max_tokens`) (§6.1, §7.4). Note the competing terminator path (Q2). |

**Covered: 12 · Partial: 2 · Gap: 0.**

The control-flow layer is fully covered, and so is the model-turn layer. The partials are at the opener/history seam and in string matching.

---

## 4. Worked encodings

### 4.1 Opener selection and the opening turn

```yaml
context:
  persona: "You are a warm, curious conversation partner."
  openers:
    - "What's on your mind?"
    - "How has your day been?"
    - "What have you been thinking about lately?"
    - "Tell me something interesting."
nodes:
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
```

`shuffle` is a deterministic permutation of the pool keyed by the session seed, and `get(..., 0)` takes the first element (§5.5). The chosen prompt is written to state, so the realized opener is recorded rather than re-derived (§7.5). The opener is an `ask` because its text is fixed by the pool, not generated.

### 4.2 The conversation loop and its two exits

```yaml
- id: convo_loop
  type: loop
  effects: { write: [turn_count] }
  config:
    body_entry: chat_turn
    exit_when: "state.last_reply == 'you MUST be a walrus!'"
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
- id: classify_exit
  type: branch
```

The body is a **single** `llm` node, so one iteration is exactly one bot message and one participant reply. Exiting *after* the body therefore leaves no dangling bot message: the last thing the participant said is what the guard tests. `exit_when` is evaluated after each iteration (§4.4), so the phrase is checked on every turn.

### 4.3 Full graph

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
      - "Tell me something interesting."
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
    - id: open_check
      type: branch
    - id: convo_loop
      type: loop
      effects: { write: [turn_count] }
      config:
        body_entry: chat_turn
        exit_when: "state.last_reply == 'you MUST be a walrus!'"
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
    - id: classify_exit
      type: branch
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
    - { from: start,        to: choose_opener, type: next }
    - { from: choose_opener, to: open,          type: next }
    - { from: open,         port: timeout, to: abandoned, type: next }
    - { from: open,         to: open_check,     type: next }
    - { from: open_check,   to: mark_phrase,    type: conditional, when: "state.last_reply == 'you MUST be a walrus!'", priority: 1 }
    - { from: open_check,   to: convo_loop,     type: default }
    - { from: convo_loop,   port: body, to: chat_turn, type: next }
    - { from: chat_turn,    to: convo_loop,     type: next }
    - { from: chat_turn,    port: timeout, to: abandoned, type: next }
    - { from: convo_loop,   port: exit, to: classify_exit, type: next }
    - { from: classify_exit, to: mark_phrase,   type: conditional, when: "state.last_reply == 'you MUST be a walrus!'", priority: 1 }
    - { from: classify_exit, to: mark_cap,      type: default }
    - { from: mark_phrase,  to: done,           type: next }
    - { from: mark_cap,     to: done,           type: next }
  terminals: [done, abandoned]
```

The graph has two phrase checks: one after the opener (`open_check`) and one after the loop (`classify_exit`). The first is necessary because `loop.exit_when` is not evaluated until after a body iteration, so a phrase spoken at the opener would otherwise be missed. The loop handles every later turn; the cap is `max_iterations`.

### 4.4 What does not encode declaratively today

**Robust phrase matching.** `state.last_reply == 'you MUST be a walrus!'` is exact and whole-message. If the participant writes `you must be a walrus` (lowercase), adds punctuation, or says it inside a longer sentence, the guard is false. The grammar exposes `matches` (§5.3) but the specification never says whether it is regex, substring, or anchored, or how it treats case, so it cannot be relied on. The core library has no `contains`, `lower`, or `upper` (§5.5). The graph as written is therefore correct only for the strictest reading of "utters the phrase" (G2).

**Opener in history.** `chat_turn` uses `history: all`, which renders prior *interactions*. The opener is an `ask`, and §7.1 specifies rendering for `llm` exchanges only. On the first loop turn the model may see the participant's first reply without the opener that prompted it. The workaround is to inject the opener explicitly with `history: {from: [{role: assistant, template: "{{ state.opener_prompt }}"}]}`, but `from` cannot be combined with `all`, so the first turn would have to leave the loop and use `from` while later turns use `all` — a split that duplicates the `llm` node (G1).

---

## 5. Gaps and observations

### G1 — History rendering for non-`llm` turns · *Needs clarification*

**Affects** P05. **Severity**: medium. §7.1 defines an interaction as "the participant's message and the reply that followed" and renders it as a `user` message then an `assistant` message. That describes a model exchange. An `ask` prompt is user-facing but not model-generated, and a `say` has no reply at all; neither has a defined rendering in `history: all`. This instrument opens with an `ask`, so the first model turn may not see the opener it is continuing.

Missing: a rendering rule for `ask`/`say` turns in history, or a way to combine an explicit `from` block with `all`.

### G2 — Literal string matching · *Needs spec extension*

**Affects** P08, P09. **Severity**: medium. `matches` appears in the `comp_op` production (§5.3) and nowhere else: no semantics, no anchoring, no case rule. The core library has no `contains`, `lower`, or `upper` (§5.5), and `==` is exact structural equality. The instrument's terminator is a literal phrase that a real participant will not type identically every time.

Missing: either a defined `matches` (with stated regex/substring and case semantics) or core string predicates such as `contains`/`lower`.

### Lower-priority observations

- **Two competing terminators.** The `loop` bounds itself with `max_iterations`/`exit_when` and takes its `exit` port (§4.4); graph `limits` independently terminates at `on_error` or `limit_exceeded` (§7.4). For a 100-turn conversation the author must decide which mechanism owns the cap, and the two end differently. The spec does not say which wins when both are configured.
- **Exit ordering at the cap.** If the phrase arrives on the same iteration that reaches `max_iterations`, §4.4 says the loop takes `exit` "when `exit_when` holds or `max_iterations` is reached" without stating precedence. P11's post-exit branch reads `last_reply`, so the label is still correct if `exit_when` is checked first; the spec should say so.
- **`loop` `error` port is referenced but not declared.** §4.4 lists `in`; `body`; `exit`, yet the requirements say it "MUST take `exit` (or the `error` edge if declared)". No `error` port is declared in the port list. This affects P14 only if the author wants a distinct limit path; today the `on_error` fallback covers it.
- **VR-34 forces a non-system message.** A history-driven chat turn would naturally declare only a `system` persona, but VR-34 requires at least one `user` or `assistant` entry. The encoding adds a synthetic user instruction; it is harmless but is an artifact of the rule rather than the design.
- **Turn-count unit (Q1).** `counter_slot` counts loop iterations, and the opener is a separate turn. Whether "100 turns" includes the opener is an authoring decision; the graph can encode either by adjusting `max_iterations`.
- **No bot-initiated termination (Q6).** The source names only the phrase and the cap; the graph matches. A model-judged stop would need a `classify`/`branch`, which the instrument does not ask for.

---

## 6. Verification

- **Spec citations** re-checked against `spec.md` v0.5.0: node types (`start`, `ask`, `llm`, `set`, `branch`, `loop`, `end`); `§`/`VR` references (§3.1, §3.7, §4.1–§4.5, §5.3, §5.5, §6.1, §7.1, §7.4, §7.5, VR-8, VR-21, VR-24, VR-33, VR-34) all confirmed present with the stated semantics.
- **No invented node types or fields.** The two gap sections name only constructs that do not exist or are undefined, marked as such.
- **YAML blocks** in §4 parse (PyYAML).
- **Pattern coverage**: all 14 inventory patterns appear exactly once in §3.

---

## 7. Assumptions and limitations

- Assessed against `spec.md` **v0.5.0 as written**; §11 open issues are treated as unavailable.
- "Covered" means the *interaction* is encodable; it does not claim the resulting conversation matches any particular study design.
- The phrase terminator is read as an exact whole-message match for the Covered claim; the looser readings are what G2 records.
- The opener pool is assumed fixed and drawn independently per session (Q3); without-replacement across a batch would add assignment machinery.
- Where the source is silent (turn unit, abandonment, bot-initiated stop), the assessment states the reading used rather than resolving it.

---

## 8. Post-integration re-evaluation (v0.6.0)

The §5 gaps were addressed by [`proposal-v0.6.0-string-matching-and-turn-rendering.md`](./proposal-v0.6.0-string-matching-and-turn-rendering.md), applied in `spec.md` v0.6.0. Re-scored against the revised spec:

| Pattern | v0.5.0 | v0.6.0 | Basis |
|---|---|---|---|
| P05 Conversation continuity | Partial | **Covered** | §7.1 turn-rendering table; the `ask` opener now renders as an `assistant` message before the participant's reply. |
| P08 Termination-phrase detection | Partial | **Covered** | `matches` defined as a case-insensitive substring test (§5.3) with VR-37. |
| P09 Phrase-triggered termination | Covered | Covered | Guard is now fully specified; no workaround. |
| P14 Safety and cost bounds | Covered | Covered | §7.4 now states the `limits`/`loop.max_iterations` relationship. |
| All other patterns | Covered | Covered | Additive changes; no construct changed meaning. |

**Covered: 14 · Partial: 0 · Gap: 0.** Verdict moves from *Sufficient with reservations* to **Sufficient**.

Three caveats carried forward, none affecting the verdict:

- The rendering-order correction is the one non-additive change: history now renders emitted-then-response (§7.1). It fixes an ordering that v0.5.0 stated backwards for bot-speaks-first turns, and is recorded in `spec.md` §11.
- `matches` folds ASCII case only, so a non-Latin phrase compares exactly (§5.3, `spec.md` §11 *String folding*). The walrus phrase is Latin-script, so this instrument is unaffected.
- Punctuation is significant: `you must be a walrus` without `!` does not match. This is a stated consequence of the substring rule, not a gap.

Requirements traceability moved as follows: G1 → §7.1 (turn-rendering table); G2 → §5.3 (`matches`), §8 (VR-37); O1 → §4.4 (`loop` `error` port); O2 → §7.4 (`limits`/`max_iterations` relationship).
