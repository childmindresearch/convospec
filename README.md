# ConvoSpec

Specification that governs how we design and implement semi-structured conversational interactions in our systems. It defines a format-agnostic node taxonomy and graph semantics: **19 node types** across interaction, cognition, control, state, and external families, plus composition, execution, and validation rules.

The specification is developed **against real instruments**, not in the abstract. Each instrument is decomposed, assessed for conformance, and any gaps are closed through a reviewed proposal before `spec.md` changes. This README records that exercise and where the specification currently stands.

---

## Specification status

| Field | Value |
|---|---|
| Version | **0.6.0** (Draft) |
| Node types | 19 (`start`, `end`, `ask`, `say`, `llm`, `analyze`, `classify`, `branch`, `decide`, `sample`, `fork`, `join`, `loop`, `foreach`, `call`, `set`, `fetch`, `tool`, `handoff`) |
| Validation rules | VR-1 … VR-37 |
| Instruments assessed | 4 (KSADS, DASH, MoBI, Walrus) |
| Current verdict | All four instruments fully encodable at v0.6.0 and rated **Sufficient**. KSADS and DASH each carry one 2-D-grid partial retained by design (P14 / P12), not by missing capability |

Revisions are **additive by default**: no existing node or rule changes meaning unless a revision note says so. The one documented exception is v0.6.0's history-rendering order (see the version table). Open questions are tracked in [`spec.md` §11](spec.md#11-open-issues).

---

## The conformance exercise

Every instrument follows the same four-step loop:

1. **Interaction inventory** — decompose the instrument into interaction patterns (`P01`, `P02`, …) and state what each requires of a runtime.
2. **Compliance assessment** — map each pattern onto the current spec and assign a verdict. Findings are classified as **Covered** (encodable directly), **Partial** (encodable with a brittle or verbose workaround), or **Gap** (not encodable).
3. **Remediation proposal** — for every partial or gap, a proposal describing the minimal spec change, the alternatives considered, and the backwards-compatibility impact. Proposals are reviewed before anything is applied.
4. **Spec revision** — accepted changes are integrated, the version is bumped, and the compliance assessment is re-scored against the new version.

This keeps the spec grounded in instruments that must actually run, and makes every capability traceable to the pattern that demanded it.

---

## Instruments evaluated

Assessed in this order; each one drove the next spec version.

| # | Instrument | Kind | Assessed at | Revised to | Final result |
|---|---|---|---|---|---|
| 1 | K-SADS-PL DSM-5 | Clinician diagnostic interview | v0.2.0 | v0.3.0 | Covered 19 · Partial 1 · Gap 0 — **Partially sufficient → remediated → Sufficient** (v0.6.0 re-score) |
| 2 | DASH Adult | Single-informant structured interview | v0.3.0 | v0.4.0 | Covered 23 · Partial 1 · Gap 0 — **Sufficient** |
| 3 | MoBI×MSM chat trial | LLM-mediated structured experiment | v0.4.0 | v0.5.0 | Covered 24 · Partial 0 · Gap 0 — **Sufficient** |
| 4 | Walrus open conversation | Open-ended chat with phrase/turn terminators | v0.5.0 | v0.6.0 | Covered 14 · Partial 0 · Gap 0 — **Sufficient** |

### 1. K-SADS-PL DSM-5

The hardest instrument and the one that shaped the core. A multi-informant diagnostic interview (parent / child / summary), with per-disorder skip-outs, dynamic supplement dispatch, cross-item triggers, and adaptive probing. Drove the multi-session **case** model, typed `decide` routing, `graph_registry` dynamic dispatch, and bounded probe loops. Baseline was **partially sufficient**; remediated in v0.3.0 and re-scored to **Sufficient** at v0.6.0, with one 2-D-grid partial (P14) retained by design.

[inventory](evaluations/ksads/ksads-interaction-inventory.md) · [compliance](evaluations/ksads/ksads-compliance.md) · [proposal](evaluations/ksads/proposal-v0.3.0-ksads-remediation.md)

### 2. DASH Adult

A single-informant interview with deterministic routing and clinician ratings. It confirmed the KSADS additions and exposed a different need: **declarative computation** — screener scores, totals, and derived section values. Drove the typed expression library, derived read-only `views`, and the core/extension extensibility tiers in v0.4.0.

[inventory](evaluations/dash/dash-interaction-inventory.md) · [compliance](evaluations/dash/dash-compliance.md) · [proposal](evaluations/dash/proposal-v0.4.0-expressions-and-views.md)

### 3. MoBI×MSM chat trial

A structured experiment in which the LLM **is** the manipulation: stance is a system prompt, replies must see the conversation so far, and a stop rule inspects generated text. Drove `session.seed` access and design-assignment functions, the `llm.messages` / `history` / `message_slot` interface, conversation scope, and the subgraph `outcome` contract in v0.5.0.

[inventory](evaluations/mobi/mobi-interaction-inventory.md) · [compliance](evaluations/mobi/mobi-compliance.md) · [proposal](evaluations/mobi/proposal-v0.5.0-llm-interface-and-assignment.md)

### 4. Walrus open conversation

Deliberately the inverse of the diagnostic interviews: almost no structure. A seeded random opener, an unbounded chat, and two terminators (a literal phrase or 100 turns). It surfaced the last two seams — literal string matching and history rendering across a non-model turn. Drove `matches` semantics and the per-turn rendering table in v0.6.0.

[inventory](evaluations/walrus/walrus-interaction-inventory.md) · [compliance](evaluations/walrus/walrus-compliance.md) · [proposal](evaluations/walrus/proposal-v0.6.0-string-matching-and-turn-rendering.md)

---

## Version history

| Version | Driver | Headline changes |
|---|---|---|
| 0.2.0 | — | Baseline assessed by the KSADS evaluation. |
| **0.3.0** | KSADS | Multi-session/case model; typed `decide`; `graph_registry` dynamic dispatch; bounded probe loops. |
| **0.4.0** | DASH | Typed expression library with aggregates and constructors; derived `views`; core/extension extensibility tiers; atomic `set`. |
| **0.5.0** | MoBI | `session.*` seed access; `range`/`shuffle`/`take`/`chunks`; `llm.messages`/`history`/`message_slot`; conversation scope; subgraph `outcome`; VR-33–36. |
| **0.6.0** | Walrus | `matches` as a case-insensitive substring test; per-turn history rendering for `ask`/`say`/`llm`; `loop` `error` port; `limits`/`max_iterations` relationship; VR-37. |

Each version's revision notes in [`spec.md`](spec.md) name the driving evaluation and proposal.

---

## Repository layout

```
spec.md                     # the specification
evaluations/
  ksads/                    # inventory, compliance, proposal
  dash/
  mobi/
  walrus/
symptomchecker.txtpb        # ConversationSpec protobuf example (symptom-checker conversation)
```

Every `evaluations/<instrument>/` directory contains:

- `<instrument>-interaction-inventory.md` — the patterns the instrument requires.
- `<instrument>-compliance.md` — the per-pattern verdicts, worked encodings, and gaps.
- `proposal-vX.Y.Z-*.md` — the reviewed remediation proposal (with integration notes once applied).

---

## Conventions

- **Verdict scale**: Covered · Partial · Gap, as defined in the exercise above.
- **No spec change without a reviewed proposal.** Compliance documents assess; proposals recommend; only then is `spec.md` edited and versioned.
- **Traceability.** Every capability in the spec traces to an inventory pattern; every proposal maps its gaps to the sections and rules it changes.
- **Extensibility.** Core functions are spec-owned with fixed semantics; implementations may add namespaced extensions under the declared/typed/pure rules; non-deterministic or effectful operations are prohibited (§5.6).

## Next steps

The evaluated instruments are all sufficient at v0.6.0. KSADS was re-scored from *partially sufficient* to *sufficient* after the v0.4.0–v0.6.0 revisions ([§9](evaluations/ksads/ksads-compliance.md#9-post-integration-re-evaluation-v060)); its one remaining partial (P14, family-history grid) is retained by design, as is DASH's P12. Outstanding design questions — serialization, custom-node ABI, multi-await forks, string folding, and the rest — are recorded in [`spec.md` §11](spec.md#11-open-issues). New instruments should follow the same inventory → compliance → proposal → revision loop.
