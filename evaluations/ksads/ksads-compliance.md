# Compliance Evaluation: ConvoSpec vs. KSADS DSM-5

**Subject**: `spec.md` — Conversation Specification (ConvoSpec), Status: Draft, Version: 0.2.0.
**Reference instrument**: K-SADS-PL DSM-5, November 2016 (see [source](./ksads-interaction-inventory.md)).
**Question**: (a) What interactions/nodes does a KSADS-style semi-structured diagnostic interview require, and (b) is ConvoSpec sufficient to encode it **today**?

**Remediation status**: the gaps identified here were addressed in `spec.md` v0.3.0 through [`proposal-v0.3.0-ksads-remediation.md`](./proposal-v0.3.0-ksads-remediation.md). Per-gap disposition is recorded in [§8](#8-remediation-status-v030). Everything above §8 remains the **v0.2.0 baseline assessment** and is preserved for traceability.

**Re-evaluation (v0.6.0)**: re-scored against `spec.md` v0.6.0 in [§9](#9-post-integration-re-evaluation-v060). Corrected tally **Covered 19 · Partial 1 · Gap 0** — the sole remaining partial (P14) is retained by design, on the same footing as DASH P12. Verdict: **Sufficient**.

---

## Verdict

**Partially sufficient.** ConvoSpec cleanly encodes the **deterministic skeleton** of the instrument — sequential and branching item delivery, per-disorder skip-outs, risk escalation, reusable supplement subgraphs, silent background analysis, and typed state — and its `handoff`/`escalated` machinery is a notably good fit for the suicide-risk flow.

It is **not sufficient, as written, to encode a faithful KSADS** without workarounds, because four capability classes are missing or only partially expressible:

| # | Gap | Blocks |
|---|---|---|
| **G1** | Multi-informant / multi-session model | P03, P13, P20 (Parent/Child/Summary) |
| **G2** | Cross-item aggregation in guards | P08, P10 (and P15, P16) |
| **G3** | Dynamic subgraph dispatch & ordering | P09, P10 |
| **G4** | Multi-turn probing within a node | P05, P01 |

G1 is primarily a **modeling decision** (expressible with existing nodes, awkwardly). G2–G4 require **spec extensions**. A fifth, weaker gap (G5, rater role) is a convenience issue. Details and sketches in [§5](#5-gaps-and-recommendations).

---

## 1. Scope and method

- **Scope**: full instrument — Unstructured Intro, Screen Interview, Supplement Completion Checklist, 5 Supplements, Summary Lifetime Checklist, and the family-history / trauma / substance sub-blocks.
- **Method**: the interaction patterns in [`ksads-interaction-inventory.md`](./ksads-interaction-inventory.md) (P01–P20) were each mapped to ConvoSpec node types and edge/validation semantics. "Sufficient today" = expressible with the node taxonomy (§4), edges (§5), composition (§6), and validation rules (§8) of spec.md v0.2.0, **without** proposed-but-uncommitted features (the `Open Issues` list in §11 is treated as *not yet available*).
- Spec citations use `§`/`VR` numbers from spec.md. KSADS citations use section names.

---

## 2. Part A — Required interactions and nodes

A KSADS-style instrument requires the following **capabilities**. (Node-type column = the ConvoSpec construct that would carry it.)

| Capability | Requirement | Candidate ConvoSpec construct |
|---|---|---|
| Open narrative dialogue | Unscripted rapport/context gathering | `llm` (`mode: conversational`) |
| Typed structured capture | Demographics, checklists, grids | `ask` (`text`/`integer`/`date`/`single_select`/`multi_select`/`scale`/`boolean`) |
| Dual-informant rating | Ask one item of Parent and Child, then synthesize | two `ask` nodes + a merge step (`ask` clinician / silent `llm` / `analyze`) |
| Dual-timeframe rating | CE and MSP per item | two state slots per item |
| Adaptive probing | Follow-up probes; not verbatim | `llm` conversational (+ `loop`) |
| Rubric scoring | Evidence → 0–3 / 0–2 code | `llm`+`output_schema`, or `classify` |
| Per-disorder skip-out | Gate a supplement on ratings | `branch` |
| Cross-item trigger | "score ≥3 on **any** previous item" | `branch` with an **aggregate** guard |
| Dynamic dispatch/ordering | Choose/order supplements by onset | `call` (needs runtime `graph_ref`) |
| Any-order completion | Interviewer-driven navigation; all sections done | `loop` + worklist |
| Risk escalation | Suicidality → escalate | `branch` → `handoff` → `end{escalated}` |
| Silent background inference | Risk/severity scoring | `analyze`, `classify` |
| Reusable subgraphs | The 5 supplements | `call` |
| Clinician synthesis | Resolve informant discrepancies | `ask` (clinician) / silent `llm` |

The instrument is **state-heavy and branch-heavy**, with two structural axes (informant × timeframe) applied to every item, and a **worklist** of supplements whose membership and order are computed from screening results.

---

## 3. Part B — Sufficiency assessment

Per-pattern verdict: **Covered** (encodable directly) · **Partial** (encodable with a workaround that is brittle or verbose) · **Gap** (not encodable today).

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Unstructured intro | Partial | `llm` conversational (§4.3) — but the node is **single-turn** (G4). |
| P02 Demographics | Covered | `ask` input types (§4.2); `multi_select` for race/living arrangements. |
| P03 Multi-informant item | Partial | Two `ask` nodes + merge works mechanically; Session is "one participant" (§3.1) so informant identity is unmodeled (G1). |
| P04 Dual timeframe | Covered | Two state slots per item (§3.2). Verbose, not first-class. |
| P05 Probe→clarify→rate | Partial | `llm` conversational + `loop` (§4.4); multi-turn within an item is not native (G4). |
| P06 Rubric scoring | Covered | `llm`+`output_schema` or `classify` (§4.3). |
| P07 Skip-out | Covered | `branch` (§4.4), VR-9. |
| P08 Cross-item trigger | Partial → Gap | Works for a **fixed** item set via enumerated `or` guards; **fails** for dynamic sets (any-drug, any-trauma) because the guard grammar (§5.3) has no aggregates (G2). |
| P09 Dynamic dispatch/order | Gap | `call.graph_ref` is static and version-pinned (§4.4); no runtime subgraph selection/ordering (G3). |
| P10 Any-order navigation | Partial → Gap | `loop` + `branch` menu possible; completion-set tracking needs aggregation (G2). |
| P11 Completion checklist | Covered | `ask` `multi_select` + `date` (§4.2). |
| P12 Summary checklist | Covered | `ask` (clinician) / silent `llm` (§4.3). |
| P13 Discrepancy resolution | Partial | `branch` (guard `parent != child`) → clinician `ask`; no rater role (G5). |
| P14 Family-history grid | Partial | Nested `loop`/`ask`; 2-D grid is awkward and not data-driven (G2/G6). |
| P15 Trauma "ever" screen | Partial | `loop` over a hard-coded list; gated `branch`. |
| P16 Substance list | Partial | Same as P15; per-drug dynamic list. |
| P17 Risk escalation | **Covered** | `branch` → `handoff` (§4.6) → `end{outcome: escalated}` (§4.1); VR-15. Strong fit. |
| P18 Timeframe anchoring | Covered | `context` (§3.6) + graph config; rules are documentary, not enforced. |
| P19 Informant adaptation | Covered | `branch` on age (§4.4) + `llm` prompt adaptation. |
| P20 Longitudinal | Partial | Session resume preserves state (§3.1); cross-session synthesis unmodeled (G1). |

**Covered: 9 · Partial: 8 · Gap: 3** (P08/P10 are *Partial* for fixed item sets and *Gap* for dynamic ones). The gaps and the partials that bottom out in G1–G4 are what make the answer to (b) "not today."

---

## 4. Worked encodings

### 4.1 What maps cleanly — Depression item, dual informant × dual timeframe

```yaml
graph:
  id: ksads_depression_screen_item
  version: 0.1.0
  entry: start
  state_schema:
    # one item × {parent,child,summary} × {ce,msp}
    dep_mood_parent_ce:   { type: integer, scope: session, mutable: true }  # 0-3
    dep_mood_parent_msp:  { type: integer, scope: session, mutable: true }
    dep_mood_child_ce:    { type: integer, scope: session, mutable: true }
    dep_mood_child_msp:   { type: integer, scope: session, mutable: true }
    dep_mood_summary_ce:  { type: integer, scope: session, mutable: true }
    dep_mood_summary_msp: { type: integer, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }

    # --- Parent ---
    - id: probe_parent
      type: llm
      config:
        model: m1
        mode: conversational          # single-turn probe (G4 caveat)
        prompt_template: "Ask the parent about depressed mood; use suggested probes adaptively."
        slot: probe_parent_text
    - { id: rate_parent_ce,  type: ask, config: { prompt: "Rate depressed mood (current).",  input_type: scale, slot: dep_mood_parent_ce,  response_schema: { min: 0, max: 3 } } }
    - { id: rate_parent_msp, type: ask, config: { prompt: "Rate depressed mood (worst past).", input_type: scale, slot: dep_mood_parent_msp, response_schema: { min: 0, max: 3 } } }

    # --- Child (separate informant) ---
    - id: probe_child
      type: llm
      config: { model: m1, mode: conversational, prompt_template: "Ask the child, in their words, about feeling sad.", slot: probe_child_text }
    - { id: rate_child_ce,  type: ask, config: { prompt: "Rate depressed mood (current).",  input_type: scale, slot: dep_mood_child_ce,  response_schema: { min: 0, max: 3 } } }
    - { id: rate_child_msp, type: ask, config: { prompt: "Rate depressed mood (worst past).", input_type: scale, slot: dep_mood_child_msp, response_schema: { min: 0, max: 3 } } }

    # --- Clinician summary (reconciliation) ---
    - { id: summarize_ce,  type: ask, config: { prompt: "Summary rating (current), all sources.",  input_type: scale, slot: dep_mood_summary_ce,  response_schema: { min: 0, max: 3 } } }
    - { id: summarize_msp, type: ask, config: { prompt: "Summary rating (worst past), all sources.", input_type: scale, slot: dep_mood_summary_msp, response_schema: { min: 0, max: 3 } } }

    - { id: b_skipout, type: branch }
    - { id: end_do_supplement,   type: end, config: { outcome: completed, reason: "administer depression supplement" } }
    - { id: end_no_supplement,   type: end, config: { outcome: completed, reason: "no depression supplement" } }
  edges:
    - { from: start,          to: probe_parent,   type: next }
    - { from: probe_parent,   to: rate_parent_ce, type: next }
    - { from: rate_parent_ce, to: rate_parent_msp,type: next }
    - { from: rate_parent_msp,to: probe_child,    type: next }
    - { from: probe_child,    to: rate_child_ce,  type: next }
    - { from: rate_child_ce,  to: rate_child_msp, type: next }
    - { from: rate_child_msp, to: summarize_ce,   type: next }
    - { from: summarize_ce,   to: summarize_msp,  type: next }
    - { from: summarize_msp,  to: b_skipout,      type: next }
    # skip-out on the SUMMARY rating (workaround: enumerated per-item guard)
    - { from: b_skipout, to: end_do_supplement, type: conditional, when: "state.dep_mood_summary_ce >= 3 or state.dep_mood_summary_msp >= 3", priority: 1 }
    - { from: b_skipout, to: end_no_supplement, type: default }
```

**Validation**: passes VR-1…VR-19. The dual-informant/dual-timeframe structure is expressible, but note the cost: this is **one item**. The depression screen has ~9 items, the full screen ~50+; hand-authoring `informant × timeframe × item` nodes does not scale and is error-prone.

### 4.2 What does not encode today — the score-gated supplement trigger

KSADS rule (paraphrased, appears repeatedly): *"If a score of 3 was received on the current rating of **any** of the preceding items, complete the … supplement."*

```text
# DESIRED — NOT expressible in ConvoSpec v0.2.0 (illustrative fragment, not valid YAML)
- id: b_dep_trigger
  type: branch
  # Intended guard: any depression-section item scored >= 3 for the current window.
edges:
  - from: b_dep_trigger
    to:   call_dep_supplement
    type: conditional
    when: "any(state.dep_section_summary_ce, >= 3)"   # ✗ no such operator in §5.3
```

The guard grammar (§5.3) supports only scalar `state.*`/`context.*` references, comparisons, arithmetic, and named predicates with **fixed positional args** (`predicates`, §5.3). There is no operator that quantifies over a `list<T>` (§3.2 defines the type but nothing consumes it in guards). The best available workaround is to **enumerate** the items:

```yaml
when: "state.dep_mood_summary_ce >= 3 or state.dep_irritability_summary_ce >= 3 or ... "
```

This is acceptable for a **fixed, known** item set but fails for the genuinely dynamic cases in KSADS — "any drug" (P16), "any traumatic event" (P15) — where membership is data-driven, and it is brittle to instrument edits.

The same limitation blocks P09/P10: ordering/iterating a computed worklist of supplements needs collection reasoning the spec does not provide.

**v0.3.0 resolution.** The trigger is now a single `decide` node (§4.4; VR-20), valid for both fixed and dynamic item sets:

```yaml
- id: b_dep_trigger
  type: decide
  config:
    operands: { list_ref: state.dep_section_summary_ce }
    test: { predicate: "item >= 3" }
    combine: any
    on_missing: unknown
    result_slot: dep_supplement_needed
```

Dynamic dispatch and ordering are covered by `graph_registry` plus `foreach` (§6.2, §4.4). See [§8](#8-remediation-status-v030).

---

## 5. Gaps and recommendations

Each gap is tagged **Fixable with existing node types** or **Needs spec extension**.

### G1 — Multi-informant / multi-session model · *Fixable with existing types (awkwardly) → recommend extension*
- **Evidence**: §3.1 defines a session as "one execution of a graph for **one participant**"; `context.participant` is singular (§3.6). KSADS rates every item from Parent, Child, and Summary (P03), interviews parent and child **separately** (P19), and synthesizes later (P13, P20).
- **Workaround**: run two sessions (parent, child) and a third synthesis session; or one graph that addresses two informants sequentially. Both lose informant identity and complicate the single-await model across sessions.
- **Recommendation**: introduce a first-class `informant`/`rater` dimension (on Session and on `ask`), and/or let `call` spawn child sessions bound to an informant with an explicit multi-source merge node. *(Extension.)*

### G2 — Cross-item aggregation in guards · *Needs spec extension*
- **Evidence**: §5.3 grammar has no aggregate/quantifier over `list<T>`; `predicates` take fixed args. Blocks P08 (dynamic), P10, and P14/P15/P16 iteration.
- **Recommendation**: extend the guard grammar with aggregate functions over `list<T>` state (e.g. `any(list, pred)`, `all`, `count`, `max`), or allow `predicates` to accept a collection argument, or add an `aggregate`/`reduce` node in the State family (§4.5) that materializes a flag. *(Extension.)*

### G3 — Dynamic subgraph dispatch & ordering · *Needs spec extension*
- **Evidence**: `call.graph_ref` is REQUIRED and version-pinned (§4.4, VR-13) — static. No construct selects a subgraph at runtime or iterates an ordered list of subgraphs. Blocks P09, P10.
- **Recommendation**: allow `call.graph_ref` to resolve from state (dynamic dispatch) and add a `foreach`/`dispatch` construct that invokes a subgraph per element of an ordered state list. *(Extension.)*

### G4 — Multi-turn probing within a node · *Needs spec extension (or formalized loop idiom)*
- **Evidence**: `llm` in `conversational` mode "emit[s] a user-facing message and await[s] the user's reply, writing the validated reply to `slot`" (§4.3); §7.1 confirms one turn per `await: user` node. A KSADS probe (P05) is a multi-exchange sub-dialogue whose termination is a **model** judgment, not a scalar guard.
- **Workaround**: `loop` (§4.4) around `llm` nodes, exiting on a model-written state flag — but the spec does not define "model decides the probe is complete" semantics, and `loop.exit_when` is a scalar guard (§5.3).
- **Recommendation**: add a `dialogue`/`probe` node (or `llm.turn_budget` + `until` predicate) that encapsulates a bounded multi-turn exchange with an explicit exit predicate and a single structured output. *(Extension.)*

### G5 — Rater role / channel · *Needs spec extension (minor)*
- **Evidence**: `ask` assumes an end-user respondent (§4.2); the clinician-as-rater is implicit. No `audience`/`channel` field beyond `say.channel`.
- **Recommendation**: add `ask.audience: participant | caregiver | clinician` and record it in the transcript (§3.3). *(Extension.)*

### G6 — Data-driven iteration · *Needs spec extension*
- **Evidence**: `loop.body_entry` is a node id, not a runtime list (§4.4). The trauma-event and drug lists (P15/P16) are data-driven.
- **Recommendation**: folded into G2/G3 — a `foreach` over a `list<T>` state variable.

### Lower-priority observations
- **Dual-timeframe (P04)** is covered but not first-class; a paired-rating pattern would reduce authoring errors.
- **Context size / transcript**: §3.6 and §11 already flag context-window and PII/retention concerns — directly relevant to a clinical instrument; no action beyond noting alignment.

---

## 6. Verification

- **Spec citations** re-checked against `spec.md`: node types (`ask`, `say`, `llm`, `analyze`, `classify`, `branch`, `sample`, `fork`, `join`, `loop`, `call`, `set`, `fetch`, `tool`, `handoff`), §/VR references (§3.1, §3.2, §3.3, §3.6, §4.1–4.6, §5.3, §6.4, §7.1, VR-9/13/15), and `predicates`/guard grammar all confirmed present with the stated semantics. **No invented node types.**
- **KSADS claims** re-checked against the source sections named in the inventory; scoring anchors, the CE/MSP dual rating, the Parent/Child/Summary columns, and the repeated "score of 3 on any previous item" trigger were confirmed across multiple sections.
- **Consistency**: no capability is marked Covered and also listed as a Gap. P08 is stated as *Partial for fixed sets, Gap for dynamic sets* and appears in G2 accordingly.
- **Residual uncertainty**: PDF text extraction was lossy; exact per-item probe wording and a few scale anchors were paraphrased rather than quoted. This does not affect the structural verdicts.

---

## 7. Assumptions and limitations

- Evaluated against spec.md **v0.2.0 as written**; items in §11 (Open Issues) are treated as unavailable.
- ConvoSpec is format-agnostic (§1); the pseudo-YAML is illustrative, not a wire format.
- "Sufficient" is judged for a **faithful** KSADS encoding. A reduced, structured-only approximation (skip adaptive probing and dynamic dispatch) is largely encodable today.

---

## 8. Remediation status (v0.3.0)

The four capability gaps (G1–G4) and the two weaker gaps (G5, G6) were addressed in `spec.md` v0.3.0. The remediation proposal is [`proposal-v0.3.0-ksads-remediation.md`](./proposal-v0.3.0-ksads-remediation.md); this table records the final disposition.

| Gap | Disposition | Where in `spec.md` v0.3.0 |
|---|---|---|
| **G1** Multi-informant / multi-session | **Resolved — replaced the evaluation's recommendation.** Informants run as separate sessions; identity and data crossing are session-level. No informant dimension is added to nodes. | §3.1 (`case_id`, `role`, `links`, `export`), §3.7, §3.5 (`session_store`), §4.6 (`session_ref`); VR-21, VR-25 |
| **G2** Cross-item aggregation | **Resolved — replaced the evaluation's recommendation.** A type-driven `decide` node replaces guard-string enumeration and grammar-level aggregates. Operand type gates legal tests; `unknown` routes missing ratings. | §4.4 `decide`; VR-20 |
| **G3** Dynamic dispatch & ordering | **Resolved — accepted with tightening.** All dispatch targets must come from a declared, version-pinned `graph_registry`; `foreach` iterates ordered, filtered collections. | §6.2, §4.4 (`foreach`, `call.graph_ref`), §6.1; VR-22, VR-23 |
| **G4** Multi-turn probing | **Resolved — the evaluation's `dialogue` node recommendation was rejected.** One user exchange per node (P8); probes are bounded loops of single-exchange nodes. | §2 P8, §6.5.1, §4.2/§4.3; VR-24 |
| **G5** Rater role / channel | **Closed without a spec field.** Responder identity is one `ask` at the start of the conversation; informant sessions carry `session.role`. No `ask.audience`. | §3.1 `role`; graph convention |
| **G6** Data-driven iteration | **Folded** into G2/G3. `foreach` over a list or registry projection plus `decide` with a list operand. | §4.4; VR-23 |

### 8.1 Per-pattern verdicts after v0.3.0

| Pattern | v0.2.0 | v0.3.0 | Basis |
|---|---|---|---|
| P01 Unstructured intro | Partial | Covered | Probe-loop pattern (§6.5.1) |
| P03 Multi-informant item | Partial | Covered | Separate sessions, links, synthesis `fetch` (§3.7) |
| P05 Probe → clarify → rate | Partial | Covered | Probe-loop pattern (§6.5.1) |
| P08 Cross-item trigger | Partial → Gap | Covered | `decide` (§4.4) |
| P09 Dynamic dispatch + order | Gap | Covered | `graph_registry` + `foreach` (§6.2) |
| P10 Any-order navigation | Partial → Gap | Covered | §6.5.2 |
| P13 Discrepancy resolution | Partial | Covered | Synthesis `fetch` + `branch` + clinician `ask` (§10.5) |
| P15 Trauma "ever" screen | Partial | Covered | `foreach` + `decide` |
| P16 Substance list | Partial | Covered | `foreach` + `decide` |
| P20 Longitudinal | Partial | Covered | `links.prior` + `fetch` + `set` deltas |

All other patterns (P02, P04, P06, P07, P11, P12, P14, P17, P18, P19) keep their v0.2.0 verdict. Totals: **Covered 19 · Partial 1 · Gap 0**. *(Corrected: an earlier revision of this section reported "Covered 18 · Partial 2" and named P04 as a remaining partial. P04 has been Covered since v0.2.0 — see [§9](#9-post-integration-re-evaluation-v060).)* The one remaining partial is ergonomic rather than a capability gap:

- **P14 Family-history grid** — nested `foreach` is plausible but unvalidated; 2-D grid ergonomics are deferred (spec §11, nested iteration). Superseded at v0.6.0 by §6.5.4 — see [§9](#9-post-integration-re-evaluation-v060).

### 8.2 Residual concerns carried into spec §11

- Sentinel handling: KSADS `0 = No information` is now routable via `decide.on_missing`, but per-variable sentinel declarations remain an open issue.
- Replay soundness of `session_store` reads of still-active sessions requires recording the observed snapshot.
- Whether `decide` should absorb `branch` into a single decision primitive.

---

## 9. Post-integration re-evaluation (v0.6.0)

The v0.3.0 remediation was re-checked against `spec.md` **v0.6.0** (which since v0.4.0 has included the typed expression library, derived `views`, and the flat-iteration pattern §6.5.4). Two findings: one bookkeeping error and one superseded rationale.

| Pattern | v0.3.0 | v0.6.0 | Basis |
|---|---|---|---|
| P04 Dual timeframe | Covered | Covered | Covered since v0.2.0 (two `ask` slots per item). v0.4.0 `views` + object constructors can project the paired `{ce, msp}` rating more cleanly (§3.8, §5.5), but the respondent-facing capture is unchanged. The §8.1 "two remaining partials" list was a miscount — P04 was never Partial. |
| P14 Family-history grid | Partial | Partial *(by design)* | §6.5.4 (v0.4.0) now states the normative 2-D pattern: an outer `foreach` over relatives plus an inner `call`/unroll over the fixed condition list; nested `foreach` is forbidden (VR-32). Fully encodable — retained as Partial by review decision **D2 ("logic stays flat")**, identical to DASH P12. |
| All other patterns | Covered | Covered | Additive v0.4.0–v0.6.0 changes; no construct changed meaning. |

**Covered: 19 · Partial: 1 · Gap: 0.** Verdict moves from *Partially sufficient* to **Sufficient** — the same footing as DASH, which carries the identical by-design 2-D-grid partial (P12). The capability gaps G1–G4 that produced the original "partially sufficient" verdict were closed in v0.3.0; nothing outstanding is a capability gap.

### 9.1 P14 encodes at v0.6.0 — family-history grid

The family-history grid is relatives × conditions. The inner condition list is fixed at authoring time, so the outer `foreach` plus an inner `call`/unroll is the normative shape (§6.5.4); no runtime iteration product arises, and VR-32 holds.

```yaml
graph:
  id: ksads_family_history
  version: 0.1.0
  entry: start
  graph_registry:                       # fixed, enumerable registry (VR-22)
    relatives:
      mother:  { graph_ref: relative_row, version: 0.1.0 }
      father:  { graph_ref: relative_row, version: 0.1.0 }
      sibling: { graph_ref: relative_row, version: 0.1.0 }
  state_schema:
    relative_options:   { type: list<string>, scope: session, mutable: true }
    relatives_to_score: { type: list<string>, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - { id: pick_relatives, type: ask, config: { prompt: "Which biological relatives should be scored?", input_type: multi_select, slot: relatives_to_score, options_ref: state.relative_options } }
    - id: per_relative
      type: foreach
      config:
        over: { list_ref: state.relatives_to_score }
        body_graph: { registry: relatives, key: "item" }
        item_slot: current_relative
        max_iterations: 12
```

`relative_row` is a subgraph that unrolls the fixed condition list — one `ask` (`scale`, 0–3) per condition, each writing `family_history[current_relative][condition]`. The inner dimension is a sequence of nodes, not a nested `foreach`, so the graph stays flat.

Caveats, none affecting the verdict:

- **Sentinel `0 = No information`** in family-history scores routes through `decide.on_missing` ([§8.2](#82-residual-concerns-carried-into-spec-11)); per-variable sentinel declarations remain a `spec.md` §11 open issue.
- **P14 is Partial by design, not by capability.** A first-class nested iterator is deliberately excluded (VR-32) to keep the graph flat and every iteration bound statically visible (§6.5.4). DASH P12 made the same call (D2). If that design decision is ever revisited, P14 becomes Covered with no other change.
- **P04's paired projection** benefits from v0.4.0 (`views` can expose `{ce, msp}` as one object for downstream scoring), but capture stays two `ask` nodes. Verdict unchanged.

Requirements traceability moved as follows: P14 → §4.4 (`foreach`, `call`), §6.5.4 (2-D pattern, no new primitive), VR-32; P04 → §3.8 `views`, §5.5 constructors (projection only).
