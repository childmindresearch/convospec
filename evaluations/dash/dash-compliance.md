# Compliance Evaluation: ConvoSpec vs. DASH Adult

**Subject**: `spec.md` — Conversation Specification (ConvoSpec), Status: Draft, Version: **0.3.0**.
**Reference instrument**: DASH Mental Health Interview — Adult, 28 April 2023 (see [source](./dash-interaction-inventory.md)).
**Question**: (a) What interactions/nodes does a DASH-style semi-structured diagnostic interview require, and (b) is ConvoSpec sufficient to encode it **today**?

**Baseline note**: unlike the [KSADS evaluation](../ksads/ksads-compliance.md), this assessment is made against **v0.3.0 directly**. The multi-informant, typed-decision, dynamic-dispatch, and probe-loop capabilities added for KSADS are already present, so no remediation layer is needed to read this document.

---

## Verdict

**Sufficient.** ConvoSpec v0.3.0 encodes a faithful DASH adult interview without new capabilities. DASH is **single-informant**, its routing is deterministic and enumerable, and its clinician ratings are ordinary structured capture. The patterns that carry the instrument — probe battery, module selection, skip-outs, reusable sub-modules, selection-driven repetition, and the episode-selection table — all map onto existing node types.

Three patterns are **Partial** — P04 (screener scoring), P12 (2-D grid), P18 (FTND total) — and all three are *ergonomic or computational*, not missing conversational capability. A fourth observation (G3, count-driven repetition) is **Covered** but verbose:

| # | Gap | Blocks | Severity |
|---|---|---|---|
| **G1** | Declarative numeric scoring over item sets | P04 (25 screener scores), P18 (FTND totals) | Partial — computable via a `tool`/custom node or pre-numeric inputs, but not declaratively in `set`. |
| **G2** | Nested (2-D) iteration | P12 (criterion × drug grid) | Partial — expressible by an outer `foreach` with an inner `call`/unroll, but nested `foreach` is unvalidated (§11). |
| **G3** | Count-driven repetition ergonomics | P09 (attempts/fears/events) | Covered, but there is no range/collection constructor; bounded `loop` or unrolled slots are required (§11). |

There is **no capability-class gap** (contrast KSADS G1–G4). Notably, the KSADS multi-informant gap **does not apply**: DASH records one `Informant ID` and has no Parent/Child/Summary synthesis. Details in [§5](#5-gaps-and-observations).

*Re-scored against `spec.md` v0.4.0 after integration: P04 and P18 become Covered, leaving P12 Partial by design — **Covered 23 · Partial 1 · Gap 0** ([§8](#8-post-integration-re-evaluation-v040)).*

---

## 1. Scope and method

- **Scope**: full instrument — Register, Timeline, Screener (25 sections), Calculated Scores, Screener Checklist, 23 disorder modules (including four substance modules), GAF, and Interviewer Observation.
- **Method**: the interaction patterns in [`dash-interaction-inventory.md`](./dash-interaction-inventory.md) (P01–P24) were each mapped to ConvoSpec node types and edge/validation semantics. "Sufficient today" = expressible with the node taxonomy (§4), edges (§5), composition (§6), execution semantics (§7), and validation rules (§8) of spec.md **v0.3.0**, treating the §11 open issues as *not yet available*.
- Spec citations use `§`/`VR` numbers from spec.md; DASH citations use section names or extracted-text line numbers.

---

## 2. Part A — Required interactions and nodes

A DASH-style instrument requires the following **capabilities** (node-type column = the construct that carries it).

| Capability | Requirement | Candidate ConvoSpec construct |
|---|---|---|
| Identity / administration metadata | Typed IDs, date, interviewer | `ask` (`text`/`integer`/`date`/`single_select`) |
| Structured probe battery | Long sequences of `No/Yes/Unk` and frequency items | `ask` (`boolean`/`single_select`/`scale`) |
| Derived scores | One score per screener section; FTND totals | `set` arithmetic (scalars) / `tool` / custom compute node |
| Interviewer-driven module selection | Choose and track modules; all applicable completed | `ask` (`single_select`, `options_ref`) + `call` + `set` + `loop` (§6.5.2) |
| Per-module skip-out / early end | Gate or terminate a module | `branch` |
| Clinician diagnostic ratings | Diagnostic / severity / certainty per diagnosis | `ask` (`single_select`/`scale`) in a reusable `call` |
| Episode-selection table | Map 2 booleans to one of 4 episodes | `branch` (multi-way) |
| Count-driven repetition | Repeat a block N times | `loop` (counter + `exit_when`) or `foreach` over a supplied list |
| Selection-driven repetition | Run a block per endorsed product/drug | `ask` (`multi_select`) → `foreach` + registry `body_graph` |
| Fixed subtype enumeration | Iterate a static subtype list, each with a sub-block | `foreach` over a list / registry projection |
| 2-D grid | Criterion × substance frequency matrix | nested `foreach` (Partial) / outer `foreach` + `call` |
| Reusable sub-modules | Help-seeking (25×), distress/impairment (56×), associated conditions | `call` |
| Mixed-unit duration | N + unit → normalized duration | `ask` + `branch` + `set` arithmetic |
| Computed anthropometric + lookup | BMI → WHO thinness category | `set` arithmetic + `branch`/`decide` (`within`) |
| Embedded scored sub-instrument | FTND gated on monthly use | `decide`/`branch` gate + `ask` items + scoring |
| Sequential sub-assessments + tail | Social vs performance → shared impairment | sequential edges (no `fork` needed) |
| Mode-conditional termination | End observation when virtual | `branch` → `end` |
| Risk escalation (advisory) | Report current risk immediately | `branch` → `handoff` → `end{escalated}` |
| Free-text elaboration | `Specify (Other)`, narrative probes | `ask` (`text`) |
| Semi-structured probing | Adaptive probe → clarify → rate | `loop` of single-exchange nodes (§6.5.1) |
| Global functioning / observation | GAF 0–100; quality ratings | `ask` (`scale`/`text`) with `session.role: clinician` |

DASH is **form-heavy, branch-heavy, and collection-heavy**, but it is **single-threaded**: only one informant is ever awaiting, so the single-await invariant (§6.4) is never strained.

---

## 3. Part B — Sufficiency assessment

Per-pattern verdict: **Covered** (encodable directly) · **Partial** (encodable with a workaround that is brittle, verbose, or non-declarative) · **Gap** (not encodable today).

| Pattern | Verdict | ConvoSpec constructs / reason |
|---|---|---|
| P01 Registration / identity | Covered | `ask` input types (§4.2); `set` computes `Study ID`. |
| P02 Developmental timeline | Covered | `ask` (`text`/`single_select`/`integer`) + interviewer prompts. |
| P03 Probe battery | Covered | `ask` (`boolean`/`single_select`/`scale`) (§4.2), sequential edges. |
| P04 Derived screener scoring | **Partial** | `set` does scalar arithmetic (§4.5, §5.3) but has **no aggregate/`sum` over item sets** and no conditional enum→number mapping (G1). |
| P05 Module selection + completion | Covered | §6.5.2: `ask` (`single_select`, `options_ref`), `call`, `set` completion list, `loop` exit via `decide`. |
| P06 Per-module skip-out | Covered | `branch` (§4.4), VR-9. |
| P07 Clinician rating block | Covered | `ask` (`single_select`/`scale`); reusable `call`; `session.role: clinician`. |
| P08 Episode-selection table | Covered | `branch` with ordered `conditional` edges + `default` (§4.4). Multi-way is explicitly a `branch` concern, not `decide`. |
| P09 Count-driven repetition | Covered | Bounded `loop` (counter + `exit_when`) or `foreach` over a supplied list (§4.4); ergonomic caveat G3. |
| P10 Selection-driven repetition | Covered | `multi_select` → `foreach` with registry `body_graph` (§4.4, §6.2); VR-22/VR-23. |
| P11 Fixed subtype enumeration | Covered | `foreach` over a static list / registry projection (§4.4). |
| P12 2-D criteria × substance grid | **Partial** | Nested `foreach` is unvalidated (§11, nested iteration); workaround is outer `foreach` + inner `call`/unroll (G2). |
| P13 Help-seeking sub-module | Covered | `call` (§4.4, §6.2). |
| P14 Distress/impairment grid | Covered | Four `ask` (`scale`, 0–10) in a reusable `call`. |
| P15 Associated-conditions block | Covered | Reusable `call` subgraph. |
| P16 Mixed-unit duration | Covered | `branch` on unit → `set` multiplier → `set` total (§4.5, §5.3). Verbose, not a gap. |
| P17 BMI → WHO category | Covered | Scalar `set` arithmetic; range lookup via `branch` or `decide` (`within`, §4.4). |
| P18 Embedded FTND | **Partial** | Gate and items are Covered (`decide`/`branch` + `ask`); the **total** shares G1's scoring limitation. |
| P19 Sequential sub-assessments + tail | Covered | Sequential edges; a shared tail is ordinary convergence. No `fork` (and none is legal for awaiting streams). |
| P20 Mode-conditional termination | Covered | `branch` on `In-person`/`Virtual` → `end` (§4.4). |
| P21 Manual risk escalation | Covered | `branch` → `handoff` (§4.6) → `end{outcome: escalated}`; VR-15. DASH only advises it; encoding it is authorial. |
| P22 Free-text elaboration | Covered | `ask` (`text`, `multiline`) (§4.2). |
| P23 Probe → clarify → rate | Covered | §6.5.1 bounded probe loop (`loop` + `classify` + `max_iterations`; VR-24). |
| P24 GAF + observation | Covered | `ask` (`scale` 0–100, `text`, `single_select`) (§4.2). |

**Covered: 21 · Partial: 3 · Gap: 0.**

The three partials are narrow and non-conversational. P04 and P18 are the **same** limitation (numeric aggregation over item sets); P12 is a **grid-iteration** ergonomics issue. None of them prevents delivery of the interview or changes the turn structure.

---

## 4. Worked encodings

Compact sketches only; they demonstrate the load-bearing constructs, not a full instrument encoding.

### 4.1 What maps cleanly — interviewer-driven module selection and completion (P05)

```yaml
graph:
  id: dash_module_navigation
  version: 0.1.0
  entry: start
  graph_registry:
    modules:
      mdd:    { graph_ref: dash_mdd,    version: 0.1.0 }
      mania:  { graph_ref: dash_mania,  version: 0.1.0 }
      ptsd:   { graph_ref: dash_ptsd,   version: 0.1.0 }
  state_schema:
    outstanding:  { type: list<string>, scope: session, mutable: true }
    completed:    { type: list<string>, scope: session, mutable: true }
    chosen:       { type: string, scope: session, mutable: true }
    nav_done:     { type: boolean, scope: session, mutable: true }
  nodes:
    - { id: start, type: start }
    - id: nav_loop
      type: loop
      config:
        body_entry: pick_module
        exit_when: "state.nav_done == true"   # written by the completion check
        max_iterations: 25
    - { id: pick_module, type: ask, config: { prompt: "Which section next?", input_type: single_select, slot: chosen, options_ref: state.outstanding } }
    - id: run_module
      type: call
      config:
        graph_ref: { registry: modules, key: "state.chosen" }
        state_isolation: isolated
    - { id: done, type: end, config: { outcome: completed } }
  edges:
    - { from: start,       to: nav_loop,   type: next }
    - { from: nav_loop,    port: body, to: pick_module, type: next }
    - { from: pick_module, to: run_module, type: next }
    - { from: run_module,  to: nav_loop,   type: next }
    - { from: nav_loop,    port: exit, to: done, type: next }
```

Completion bookkeeping is abbreviated here: the full form (§6.5.2) records the chosen module as complete and sets `nav_done` with a `decide` (`combine: all`) over the completion list. List append is the spec §11 *Collection construction* open issue, so the sketch omits it rather than showing invalid `set` syntax. Registry dispatch is enumerable (VR-22) and version-pinned (VR-13).

### 4.2 Selection-driven repetition — per-drug modules (P10)

```yaml
- { id: q_drugs, type: ask, config: { prompt: "Select drugs endorsed for ever use.", input_type: multi_select, slot: drugs_endorsed, options_ref: state.drug_options } }
- id: per_drug
  type: foreach
  config:
    over: { list_ref: state.drugs_endorsed }
    body_graph: { registry: drug_modules, key: "item" }
    item_slot: current_drug
    max_iterations: 10
```

The endorsed set is produced by `multi_select` (§4.2); `foreach` snapshots it at entry and dispatches a version-pinned module per element (§4.4, §6.2; VR-23). This is the same shape as the tobacco `Check all that apply` → per-product blocks.

### 4.3 Episode-selection table (P08)

```yaml
nodes:
  - { id: b_episode, type: branch }
edges:
  - { from: b_episode, to: code_worse,   type: conditional, when: "state.worse == true", priority: 1 }              # Groups B, D
  - { from: b_episode, to: code_current, type: conditional, when: "state.current == true", priority: 2 }            # Group C
  - { from: b_episode, to: code_last,    type: default }                                                           # Group A
```

The printed A–D table collapses to three outcomes: if a worse episode exists it wins (B, D); otherwise current wins (C); otherwise last (A). This is multi-way routing, so it is a `branch`, **not** a `decide` (which routes only `true`/`false`/`unknown`, §4.4).

### 4.4 What does not encode declaratively today

**Numeric screener score (G1).** The instrument prints 25 calculated section scores (p56) but not their formulas.

```text
# DESIRED — not expressible with core nodes
- id: score_mdd
  type: set
  config:
    assignments:
      - slot: mdd_score
        expression: "sum(state.mdd_probes)"   # ✗ no aggregate over list<T>; probes are enum No/Yes/Unk
```

The grammar (§5.3) has no `sum`/`count` over a `list<T>`, and no conditional/ternary or enum→number coercion, so a faithful `No/Yes/Unk → 1/0` sum cannot be written in `set`. Workarounds: compute with a `tool`/custom compute node (§4.6/§9), or change the probes to `ask` `integer` 0/1 (which alters the interaction). `decide` (§4.4) can produce *boolean* outcomes (`any`/`all`/`n_of_k`) but not the integer score.

**2-D criterion × drug grid (G2).**

```text
# DESIRED — nested iteration
foreach over drugs:            # outer
  foreach over criteria:       # inner — nested foreach unvalidated (spec §11)
    ask frequency: No | Once | Twice | Three or more times
```

The spec supports `foreach` but not nested `foreach` (§11, nested iteration). Workaround: outer `foreach` over drugs with an inner `call` to a criterion-grid subgraph, or unroll the (fixed) criteria list.

---

## 5. Gaps and observations

Each gap is tagged **Fixable with existing node types** or **Needs spec extension**.

### G1 — Declarative numeric scoring over item sets · *Needs spec extension (or accept `tool`/custom node)*
- **Evidence**: 25 calculated screener scores (p56); FTND totals (line 25969); the same limitation would hit any weighted symptom count. BMI (P17) is *not* affected — it is scalar arithmetic and encodes cleanly.
- **Workaround**: a `tool` or custom compute node (§4.6, §9) computes the score; or store probe answers as `integer` 0/1 instead of `No/Yes/Unk`, which changes the respondent-facing interaction.
- **Recommendation (gated on review)**: add a pure compute/aggregate facility in the State family (`sum`, `count`, `avg` over `list<T>`; conditional mapping), or extend the §5.3 grammar. This aligns with the existing §11 items *Collection construction* and *Guard grammar finalization*. *(Extension.)*

### G2 — Nested / 2-D iteration · *Needs spec extension (ergonomic)*
- **Evidence**: the Drug Use `With which drug(s) has that happened? How many times?` grid rates each DSM criterion × each drug (line 33527). Spec §11 flags nested iteration as open.
- **Workaround**: outer `foreach` over drugs, inner `call` to a reusable criterion-grid subgraph; or unroll the fixed criterion list.
- **Recommendation (gated)**: define whether nested `foreach` is legal, or standardize the "outer `foreach` + inner `call`" idiom in §6.5. *(Extension; ergonomic.)*

### G3 — Count-driven repetition without a range constructor · *Covered; ergonomic note*
- **Evidence**: suicide `Attempts (Up to Six Entries)` (line 7409), Specific Phobia `How many fears` 1–3 (line 12013), PTSD `Event 1`…`Event 5` (line 24332).
- **Status**: Covered by a bounded `loop` with `counter_slot` + `exit_when`, or by `foreach` over a list supplied in `context`; per-entry slots are unrolled for the fixed maxima. There is no "range 1..N" or list-append (§11 *Collection construction*), so authoring is more verbose than ideal.
- **Recommendation (gated)**: a `repeat`/range form would reduce authoring cost but is not required. *(Ergonomic.)*

### Lower-priority observations
- **Mixed-unit duration (P16)** is Covered but verbose: `branch` on the unit enum into per-unit `set` conversions. A unit-aware duration type would help; not a gap.
- **Risk escalation (P21)** is only advisory in DASH. `handoff` → `end{escalated}` is a strong fit if an escalation flow is authored, but DASH itself prescribes no graph.
- **Single-informant advantage**: the KSADS multi-informant gap is absent. `case_id`/`role`/`links` (§3.7) remain available if a study wants to link sessions, but DASH needs no synthesis session.
- **Clinician-as-respondent** is handled by `session.role: clinician`; no `ask.audience` field is needed (consistent with the v0.3.0 position).

---

## 6. Verification

- **Spec citations** re-checked against `spec.md` v0.3.0: node types (`ask`, `say`, `llm`, `analyze`, `classify`, `branch`, `decide`, `sample`, `fork`, `join`, `loop`, `foreach`, `call`, `set`, `fetch`, `tool`, `handoff`); `§`/`VR` references (§3.1, §3.2, §3.5, §3.7, §4.2–4.6, §5.3, §6.1–6.5, §7, VR-8/9/10/11/13/15/20/21/22/23/24/25); `options_ref`, `graph_registry`, `body_graph`, `session_store`, and `session_ref` all confirmed present with the stated semantics. **No invented node types or fields.**
- **DASH claims** re-checked against the extracted text at the cited lines: 25 calculated scores (p56), Screener Checklist (p57), episode A–D table (lines 3539–3572), risk instruction (line 1440), attempts/fears/events (lines 7409, 12013, 24332), tobacco and drug selections (lines 25148, 30186), FTND gate (line 25969), BMI/WHO table (lines 23579–23597), mode-conditional end (line 37112).
- **Consistency**: no pattern is both Covered and a Gap. P04/P18 are the same G1 limitation and are counted as two Partial *patterns*, one gap. P12 is G2. P09 is Covered with an ergonomic note (G3), not a Partial.
- **Residual uncertainty**: the PDF does not print scoring formulas (only `[d…_screenscore]` placeholders), so G1 concerns *expressiveness*, not formula recovery. RedCAP renders age/option lists as long vertical enumerations; these were read as single-selects, not as repeated items.

---

## 7. Assumptions and limitations

- Evaluated against spec.md **v0.3.0 as written**; §11 open issues are treated as unavailable.
- DASH is a **single-informant** adult interview; one session per administration. Multi-session `case` modelling is optional.
- "Sufficient" is judged for a **faithful** encoding of the instrument's interactions. Exact RedCAP scoring formulas are out of scope; they are assumed to be derived numeric aggregates over each section's probes.
- ConvoSpec is format-agnostic (§1); the YAML above is illustrative, not a wire format.

---

## 8. Post-integration re-evaluation (v0.4.0)

The §5 gaps were addressed by [`proposal-v0.4.0-expressions-and-views.md`](./proposal-v0.4.0-expressions-and-views.md), applied in `spec.md` v0.4.0. Re-scored against the revised spec:

| Pattern | v0.3.0 | v0.4.0 | Basis |
|---|---|---|---|
| P04 Derived screener scoring | Partial | **Covered** | `views` (§3.8) + `count`/`sum`/`lookup` (§5.5); §6.5.3. |
| P09 Count-driven repetition | Covered | Covered | Unchanged; `len`/`count` and the accumulator pattern (§6.5.3) make it less verbose. |
| P12 2-D criteria × substance grid | Partial | Partial | Unchanged by design. Nested `foreach` is forbidden (VR-32); the grid is an outer `foreach` plus a `call` boundary (§6.5.4). |
| P18 Embedded FTND | Partial | **Covered** | `views` + `sum(map(…, lookup(…)))` (§5.5, §10.8). |
| All other patterns | Covered | Covered | Additive changes; no construct changed meaning. |

**Covered: 23 · Partial: 1 · Gap: 0.**

P12 remains Partial by an explicit review decision (D2: logic stays flat), not by lack of capability — the instrument is fully encodable as `foreach` + `call`, and §6.5.4 now states that pattern normatively. The verdict for the instrument overall is unchanged: **Sufficient**, now with no workaround needed for scoring.

Requirements traceability moved as follows: G1 → §5.5; G2 → §6.5.4 (pattern, no new primitive); G3 → §3.8 `views`, §4.5 atomic `set`, §5.5 constructors, §6.5.3 patterns.
