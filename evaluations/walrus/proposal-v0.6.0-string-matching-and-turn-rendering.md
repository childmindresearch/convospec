# Proposal: ConvoSpec v0.6.0 — Literal String Matching and Turn Rendering

| Field | Value |
|---|---|
| Status | **Applied** — integrated into `spec.md` v0.6.0 (integration notes in [§13](#13-integration-notes)) |
| Target spec version | 0.6.0 |
| Baseline | `spec.md` v0.5.0 |
| Source evaluation | [`walrus-compliance.md`](./walrus-compliance.md) |
| Evidence base | [`walrus-interaction-inventory.md`](./walrus-interaction-inventory.md) (P01–P14) |
| Review input | User review of walrus G1/G2 and questions 1–3, 2026-11-16 |

This document proposed changes to `spec.md` v0.5.0. They are now **applied in `spec.md` v0.6.0**; [§13](#13-integration-notes) records integration notes and every deviation from the text below. The body of this document is preserved as the reviewed proposal.

**Citation convention.** `§X` and `VR-N` refer to `spec.md` **v0.5.0**. References to this document are written `this §X`. Sections marked *(new)* are proposed spec text that does not exist yet.

---

## 1. Status and scope

The walrus compliance evaluation ([`walrus-compliance.md`](./walrus-compliance.md) §5) found **no gap** and two partials, both at the seams of an otherwise fully-covered interaction: string matching and history rendering across a non-model turn.

| Gap | Evaluation's finding | Disposition here |
|---|---|---|
| **G2** Literal string matching | `matches` is in the grammar but has no semantics; no `contains`/`lower`/`upper` in core | **Accepted.** Define `matches` as a case-insensitive substring test (this §3) |
| **G1** Non-model turn rendering | §7.1 renders only `llm` exchanges; an `ask` opener has no defined place in `history` | **Accepted.** A per-turn rendering table for `ask`/`say`/`llm` (this §4) |
| **O1** `loop` `error` port | Requirements reference the `error` edge; the port list omits it | **Accepted.** Declare the port (this §5.1) |
| **O2** Competing terminators | `loop.max_iterations` and graph `limits` both end the path, differently | **Accepted.** State the relationship (this §5.2) |

Three constraints shape everything below:

1. **No new node types.** Both gaps are prose and one already-present grammar operator. The taxonomy stays flat.
2. **`matches` was never usable, so defining it is additive in practice.** No valid v0.5.0 graph could depend on it, because it had no meaning. The only reserved-name pressure is that an extension named `matches` becomes a core name (this §9).
3. **One behaviour changes.** The turn-rendering rule in this §4 corrects an ordering that v0.5.0 stated backwards for bot-speaks-first turns. This is called out in this §1.2, this §4.2, and this §9 rather than buried.

### 1.1 Summary of proposed changes

| Kind | Addition |
|---|---|
| Grammar (§5.3) | No production change; `matches` already appears in `comp_op`. Semantics added in prose |
| Core semantics (§5.3) | `matches` is a case-insensitive substring test; both operands MUST be `string` |
| Execution (§7.1) | A per-turn rendering table for `ask`, `say`, and `llm`; history is the chronological concatenation of rendered turns |
| `loop` (§4.4) | `error` added to the declared port list |
| Budgets (§7.4) | `limits` and `loop.max_iterations` relationship stated |
| Validation (§8) | VR-37 (`matches` operand types) |
| Worked example (§10.10, new) | A bounded free-conversation loop with a phrase terminator |

### 1.2 What does not change

- `llm`, `ask`, `say`, `loop`, and `end` keep their ports, config, and `await` behaviour.
- `history` keeps its four shapes (`all`, `none`, `{last: N}`, `{from: [...]}`). This proposal changes how a *turn* renders into messages, not which turns are selected.
- `sample`, `views`, `set`, `call`, and the core library are untouched. No `contains`/`lower`/`upper` is added (this §3.5).
- Graphs whose every turn is `llm` and whose bot speaks first keep the same *set* of messages; this proposal fixes their *order* to match the transcript (this §4.2).

---

## 2. Review decisions and their consequences

| # | Question | Decision | Consequence |
|---|---|---|---|
| 1 | Phrase matching (G2) | **Define `matches` as substring, case-insensitive, no regex** | One grammar operator gains semantics; no core function added (this §3) |
| 2 | Non-model rendering (G1) | **Add `ask`/`say` rendering rule** | A per-turn table; the opener becomes visible in history (this §4) |
| 3 | Proposal scope | **Both G1 and G2**, plus the two observations | This document covers all four |

---

## 3. G2 — Literal string matching

### 3.1 Motivation

The walrus terminator is the instrument's entire content-dependence: the conversation runs until the participant utters a fixed phrase (P08). The grammar already lists `matches`:

```
comp_op = "==" | "!=" | "<" | "<=" | ">" | ">=" | "in" | "matches" ;
```

but the specification never says what it does. A graph that uses it today is relying on undefined behaviour, which VR-26 cannot validate. The core library has no `contains`, `lower`, or `upper`, and `==` is exact structural equality, so the only *defined* way to test for the phrase is a whole-message exact match.

### 3.2 `matches` semantics (§5.3)

Add to the "Equality and membership" paragraph:

> **String matching.** `a matches b` is true when `b` occurs as a contiguous substring of `a`, comparing characters with ASCII case folding only (`A`–`Z` ↔ `a`–`z`); all other characters compare exactly. It is not a regular expression, it does not anchor, and it treats no character specially. Both operands MUST be `string`; either operand `null` makes the result `false`, consistent with the null-comparison rule above. Case folding is ASCII-only so the result does not depend on the host locale.

Worked consequences, all with pattern `"you MUST be a walrus!"`:

| Subject | Result | Why |
|---|---|---|
| `you MUST be a walrus!` | true | exact |
| `you must be a walrus!` | true | ASCII case fold |
| `well, you must be a walrus! anyway` | true | contiguous substring |
| `you must be a walrus` | false | pattern's `!` absent |
| `you must be a walrus?` | false | `?` ≠ `!` |

Whitespace and punctuation inside the pattern are significant. An author who wants punctuation- or whitespace-insensitive matching composes an extension (§5.6) or uses an explicit normalized value in state; the core rule stays simple and total.

### 3.3 Why `matches` and not a `contains` function

`matches` is already in the grammar, so defining it adds no surface area and no reserved core name beyond one that already exists. A `contains(string, string)` core function would express the same test but would leave `matches` dangling and undefined — a worse outcome. The two would also overlap confusingly. One defined operator is preferable to one defined function plus one undefined operator.

### 3.4 Worked sketch — the walrus terminator

```yaml
- id: convo_loop
  type: loop
  config:
    body_entry: chat_turn
    exit_when: "state.last_reply matches 'you MUST be a walrus!'"
    max_iterations: 100
    counter_slot: turn_count
```

The guard is now fully specified. No `classify` node, no normalization step, and no exactness caveat is needed.

### 3.5 Rejected alternatives

- **Core `contains`/`lower`/`upper`.** More general, but leaves `matches` undefined and adds three functions where one operator suffices. If a future instrument needs normalization, it can be added then.
- **Regex semantics for `matches`.** Powerful, but introduces a regex dialect, catastrophic-backtracking risk, and a portability problem across implementations. The instrument needs a literal phrase, not a pattern language.
- **Full Unicode case folding.** Correct in principle, but locale- and version-dependent across runtimes. ASCII folding is deterministic and covers the source's Latin-script phrase. Non-ASCII folding can be a declared extension.

---

## 4. G1 — Turn rendering for non-`llm` nodes

### 4.1 Motivation

The walrus conversation opens with a fixed prompt chosen from a pool, so it is an `ask` (P03). Every later turn is an `llm` (P04). For the chat to continue coherently, each `llm` must see the opener and the participant's reply to it (P05). v0.5.0 §7.1 defines an interaction as "the participant's message and the reply that followed" and renders it as "one `user` message … followed by one `assistant` message". That describes a model exchange and says nothing about an `ask` prompt or a `say`.

There is also an ordering defect to fix at the same time. A `conversational` `llm` **emits its message and then awaits the reply** (§4.3). So within one turn the emitted message precedes the participant's response. §7.1 renders the pair user-then-assistant, which reverses the order the messages actually occurred in. For a bot-speaks-first conversation — the walrus loop, and MoBI's two-reply trial — this produces an out-of-order transcript.

### 4.2 Rendering rule (§7.1)

Replace the single-sentence rendering rule with a per-turn table. Proposed text:

> **Turn rendering.** History is assembled from the turns in the visible scope, in chronological order. Each turn renders by node type:
>
> | Node | Emitted output | Participant response |
> |---|---|---|
> | `ask` | one `assistant` message (the prompt) | one `user` message (the validated reply) |
> | `llm` (`conversational`) | one `assistant` message (the emitted text) | one `user` message (the validated reply) |
> | `say` | one `assistant` message (the text) | — |
> | `llm` (`silent`), and all other `await: none` nodes | no message | — |
>
> Within a turn the emitted output renders before the participant's response, because that is the order in which they occurred. Turns render in the order they executed, so a bot-speaks-first conversation renders `assistant`, `user`, `assistant`, `user`, … An `ask` prompt and a `say` message are authored text, not model output, but they are user-facing and render with the `assistant` role so the model sees what it is continuing.

This is a **correction**, not an addition: it changes the order of the `assistant`/`user` pair for existing bot-speaks-first graphs. It is called out in this §1.2 and this §9. The alternative — keeping user-then-assistant and only adding `ask`/`say` — would either orphan the opener or drop the participant's most recent message from the prompt (this §4.5).

### 4.3 Consequences for the four `history` shapes

The four shapes keep their meaning: they select *turns*, and the table above renders each selected turn. In particular:

- `all` — every prior turn in scope, rendered chronologically. The opener is now included as an `assistant` message.
- `{last: N}` — the last `N` turns, each rendered as above. `N` still counts turns, not messages, so a turn may contribute two messages.
- `{from: [...]}` — explicit entries, unchanged.
- `none` — no history, unchanged.

### 4.4 Worked sketch — the opener visible to the first model turn

With the opener an `ask` and the loop body an `llm`, the first model turn sees `assistant: "What's on your mind?"`, then `user: <the participant's reply>`. No `history.from` injection and no split node are needed:

```yaml
- id: open
  type: ask
  config:
    prompt: "{{ state.opener_prompt }}"
    input_type: text
    slot: last_reply
- id: chat_turn
  type: llm
  config:
    model: m1
    mode: conversational
    messages:
      - { role: system, template: "{{ context.persona }}" }
      - { role: user,   template: "Reply to my last message and keep the conversation going." }
    history: all
    slot: last_reply
```

### 4.5 Rejected alternatives

- **Keep user-then-assistant; add only `ask`/`say`.** Pairing each user message with the *next* turn's assistant message is the only reading that preserves user-then-assistant, and it drops the participant's latest message from the prompt (the one the model is replying to). Pairing within a turn keeps the latest message but reverses chronology. Neither is coherent for a bot-speaks-first loop.
- **Allow `from` to combine with `all`.** Lets an author inject the opener ahead of prior turns, but requires every author to remember to do it, duplicates the opener if it is also rendered, and does not fix the ordering defect for `llm` turns. The rendering rule fixes all cases at once.
- **Add a `rendering` field per node.** More configurable than any instrument needs; the node type already determines the role.

---

## 5. Minor clarifications

### 5.1 `loop` `error` port (§4.4)

The `loop` port list is `in`; `body`; `exit`, but its requirements say it "MUST take `exit` (or the `error` edge if declared)". Declare the port:

> **Ports**: `in`; `body` (enter loop body); `exit`; `error` (optional).

No behaviour changes; this makes the requirements consistent with the port list.

### 5.2 `limits` and `loop.max_iterations` (§7.4, §4.4)

Both can end a path, and they end differently: the `loop` takes `exit`; graph `limits` terminate at `on_error` or an `end` with `limit_exceeded` (§7.4). Add:

> A `loop.max_iterations` is the local bound for that loop; graph `limits` are session-wide backstops checked at turn boundaries. When both are configured, the loop's bound SHOULD be set at or below the corresponding session limit so the loop's `exit` is reached first. If a session limit fires before the loop exits, the path terminates at `limit_exceeded` and the loop's `exit` is not taken.

This turns a silent interaction into a stated rule. It changes no behaviour for a graph whose loop bound is already below its session limit.

---

## 6. Consolidated spec change list

### 6.1 Sections to amend

| Section | Change |
|---|---|
| §5.3 | Add the "String matching" paragraph defining `matches` |
| §7.1 | Replace the single rendering sentence with the per-turn table and ordering rule |
| §4.4 (`loop`) | Add `error` to the port list |
| §7.4 | Add the `limits`/`max_iterations` relationship |
| §8 | Add VR-37 |
| §10.10 *(new)* | Worked example: bounded free-conversation loop with a phrase terminator |
| §11 | Record the ASCII-only folding decision and the rendering-order correction |
| Header | Revision notes for 0.6.0, stating the one behaviour change |

### 6.2 Proposed validation rules

- **VR-37** — Both operands of `matches` MUST be `string`; an implementation MUST evaluate it as a case-insensitive (ASCII folding) contiguous-substring test with no regular-expression or anchoring semantics.

No validation rule is needed for rendering; it is execution semantics, not a graph constraint.

### 6.3 Existing rules affected

- **VR-26** already requires every call to match a declared signature; `matches` is an operator, not a call, so VR-37 is its typing rule.
- No other rule changes. `history` selection (VR-35) is unchanged; only rendering is defined.

### 6.4 Grammar diff

None. `matches` is already in `comp_op`; this proposal supplies the missing semantics. No new token, production, or reserved root is introduced.

---

## 7. Determinism and inspectability impact

- `matches` is a pure function of its two string operands; it introduces no randomness, I/O, or clock read, and it is locale-independent by construction. Replay is unaffected.
- Turn rendering is a pure function of the recorded transcript. Because §7.5 already requires the runtime to record the assembled prompt (or its hash and inputs), the corrected ordering is captured automatically.
- The realized opener is written to state (P02), so the random choice stays auditable under the existing `seed` rule (§3.4).

---

## 8. Traceability (walrus)

| Requirement | Before | After |
|---|---|---|
| G2 — phrase terminator | P08 Partial: `matches` undefined, `==` exact only | **Covered**: §5.3 `matches`, VR-37 |
| G1 — opener in history | P05 Partial: `ask` rendering undefined | **Covered**: §7.1 turn-rendering table |
| P09 — phrase termination | Covered, conditional on G2 | Covered unconditionally |
| P14 — cost bounds | Covered, with competing terminator | Covered; §7.4 states the relationship |
| O1 — `loop` error port | Port referenced, not declared | Declared (§4.4) |

With this proposal applied, the walrus re-score is **Covered 14 · Partial 0 · Gap 0**.

---

## 9. Backwards compatibility and migration

### 9.1 Source compatibility

- Every v0.5.0 graph still validates. No node, field, or rule is removed or retyped.
- **One behaviour change:** history message order for turns whose emitted message precedes the participant's response (every `conversational` `llm`, and now every `ask`). v0.5.0 rendered user-then-assistant; v0.6.0 renders emitted-then-response. Existing graphs get the corrected, chronological order.
- Defining `matches` cannot break a valid graph, because the operator previously had no meaning and could not be validated.

### 9.2 Reserved names

`matches` becomes a defined core operator. An implementation extension or predicate named `matches` is now a collision under VR-26. This is the same reserved-name policy already applied to core functions in v0.4.0.

### 9.3 Migration

No graph rewrite is required. An author who wants the old user-then-assistant order can reconstruct it with `history.from`, but the corrected order is the intended one.

---

## 10. Implementation plan (proposed order)

1. **§5.3 prose** — define `matches` (no grammar change).
2. **§8 VR-37** — the typing and semantics rule.
3. **§7.1** — the turn-rendering table and ordering rule.
4. **§4.4 / §7.4** — the two clarifications.
5. **§10.10** — the worked example.
6. **Header and §11** — revision notes and recorded decisions.

Steps 1–2 are independent of 3–4 and can land first.

---

## 11. Decisions and remaining questions

| # | Question | Status |
|---|---|---|
| 1 | `matches` semantics | **Settled**: substring, ASCII case-insensitive, no regex (this §3) |
| 2 | Non-model turn rendering | **Settled**: per-turn table with emitted-then-response order (this §4) |
| 3 | Proposal scope | **Settled**: G1 + G2 + O1 + O2 (this §1) |

Open for review:

1. **Punctuation insensitivity.** The rule keeps `!` significant, so `you must be a walrus` (no `!`) does not match. Is that acceptable for the instrument, or should a `normalize`-style core function be considered later?
2. **Unicode folding.** ASCII-only is deterministic but narrow. Record it as an §11 open issue, or leave it implicit?
3. **Rendering-order correction.** This is the one non-additive change. Confirm the version should bump to 0.6.0 with the behaviour change stated in the revision notes, rather than shipping it as an erratum to 0.5.x.

---

## 12. Explicitly not proposed

- A `contains`/`lower`/`upper` core library addition.
- Regex, glob, or pattern matching in `matches`.
- A per-node `rendering` or `role` field.
- Any change to `history` selection shapes, `state_isolation`, or conversation scope.
- A dedicated turn-counter node or a `max_turns`-as-expression; `loop.counter_slot` and `limits` suffice.
- Punctuation or whitespace normalization of participant input.

---

## 13. Integration notes

Applied in `spec.md` v0.6.0, following the order in §10 (matching semantics → validation rule → rendering → clarifications → example → header/§11). Sections 3–5 landed as proposed, with the deviations below.

### 13.1 Deviations and additions

| # | Change | Reason |
|---|---|---|
| 1 | **§10.10 authored in full** | The proposal described the example but did not include its YAML. It was written during integration and validated against VR-1…VR-37. |
| 2 | **Two §11 entries added** — *String folding* and *Turn rendering order* | The proposal's §11 questions 1–2 were answered as "keep and record"; recording them as §11 issues is the concrete form of that answer. |
| 3 | **Revision notes state the one behaviour change** | The proposal §1.2/§9 required it; the spec header now says the history order changed, so a reader comparing versions sees it without this document. |
| 4 | **`matches` paragraph names VR-37 inline** | Keeps the prose and the validation rule in one place, as §5.3 does for other operators. |

### 13.2 Decisions

All three review questions are settled and implemented:

| Question | Decision | Where it lives |
|---|---|---|
| 1 Punctuation significance | Keep `!` and punctuation significant | §5.3 (examples in §10.10) |
| 2 Unicode folding | ASCII-only; record as open issue | §5.3, §11 *String folding* |
| 3 Apply as | v0.6.0 with the rendering correction stated | Header, §7.1, §11 *Turn rendering order* |

### 13.3 Verification performed

- All 27 YAML blocks in `spec.md` parse (PyYAML), including the new §10.10 graph.
- VR-1…VR-37 are defined exactly once each, in order, with no duplicates or gaps.
- Every `§` reference and internal anchor resolves in the revised spec; every relative link resolves.
- No tabs or trailing whitespace in `spec.md` or this document.
- Traceability in §8 matches [§8 of the compliance evaluation](./walrus-compliance.md#8-post-integration-re-evaluation-v060).
