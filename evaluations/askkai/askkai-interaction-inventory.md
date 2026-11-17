# Ask Kai Symptom Checker — Interaction Inventory (Evidence Base)

| Field | Value |
|---|---|
| Instrument | Ask Kai pediatric mental-health symptom checker |
| Source | [`symptomchecker.txtpb`](../../symptomchecker.txtpb) — a `ConversationSpec` protobuf (text format), `api/proto/config/conversation_spec.proto` |
| Size | 68 top-level states, 186 distinct `prompt_id`s, 12 disorder labels, 3 external actions |
| Scope assessed | One session: intake, safety interrupt, screener, disorder supplements, model-derived results, content delivery, report/newsletter actions, closing survey |
| Explicitly out of scope | Prompt wording and resource catalogue contents, model choice, UI themes, and storage outside the session |

This document is the evidence base for [`askkai-compliance.md`](./askkai-compliance.md). It decomposes the instrument into interaction patterns and states, for each, what varies and what the runtime must do. It does not judge ConvoSpec support; that is the compliance document's job.

**Citation convention.** `P<n>` refers to a pattern below. `Q<n>` refers to an open point in §4. Source line numbers refer to [`symptomchecker.txtpb`](../../symptomchecker.txtpb).

**Source caveat.** The `.proto` schema is not in this repository; field meanings below are inferred from the text-format data and from naming. Where a semantic is ambiguous, it is recorded as an open point (Q) rather than assumed.

---

## 1. Shape of the instrument

Ask Kai is **not a flow graph**. It is a **set of guarded states** plus a selection policy:

```text
selection { type: SELECTION_POLICY_TYPE_PRIORITY }
```

Every state declares `state_id`, an optional `priority`, optional `tags`, a list of `prompts`, a `constraint_logic` predicate over conversation history and derived facts, and optionally an `action_spec`. The runtime repeatedly picks the highest-priority state whose `constraint_logic` currently holds, runs it, and re-evaluates. `once {}` marks a state as non-repeatable; `STATE_TAG_TRANSIENT` marks states that do not count as the "last" ordinary state; `priority: 99` marks an urgent interrupt; `priority: -99` marks the final message.

The consequences for assessment:

- **Control is implicit.** There are no edges. Ordering is a function of priority plus guards over the *transcript* and over *derived facts*.
- **Guards read history.** `constraint_logic` is dominated by `history { match … position … response … reduce_by … }` (107 occurrences), not by named state variables. A state says "the last thing that happened was X and its response looked like Y."
- **Facts are derived, not captured.** Risk (`risk { assessment }`), predicted diagnosis (`diagnosis`, `tag { disorders { predicted: true } }`), and severity tags are produced from upstream inputs and then used as guards and as content-selection keys.
- **The tail is content delivery.** Result states attach `resources { queries … }` and `action_spec { … }` to serve resources, email a report, render a file, or sign up for a newsletter.

This is the **inverse authoring model** of KSADS, DASH, and MoBI (explicit graphs) and of Walrus (a straight line with two exits). It is closest to a **production system**: a working memory (the transcript and derived facts) and a priority-ordered rule set.

---

## 2. Patterns

### Phase A — Setup and narrative

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P01** | Priority-selected state dispatch | Pick the highest-priority state whose constraint holds; re-evaluate after each state; honour `once`, transient tags, and terminal priority. | 19–21, 68–70, 1756–1761 |
| **P02** | Structured intake | Age (slider), gender (single select), relation (single select) captured in sequence. | 24–55 |
| **P03** | Free-text narrative + length-gated follow-up | Open narrative prompt; if the reply is shorter than 50 characters, ask a follow-up before proceeding. | 57–66, 104–146 |

### Phase B — Safety

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P04** | Risk detection and safety interrupt | After a turn, assess risk by keyword and by model (`KeywordAssessment`, `LlamaGuardAssessment`); if risk, preempt the flow with a warning and a continue/stop choice; stop ends the conversation. | 68–102 |

### Phase C — Core screener

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P05** | Bulk yes/no screener block | One state presents 19 yes/no toggle prompts spanning several disorders. | 148–178 |
| **P06** | Tagged response-set aggregation | Count responses that are tagged `relevant`, optionally scoped to disorders; count distinct disorders with a qualifying response. | 188–238 |
| **P07** | Count-gated follow-up branch | Choose a single-disorder or multi-disorder follow-up based on the aggregates from P06. | 180–240 |

### Phase D — Disorder supplements

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P08** | Reusable per-disorder supplemental survey | A short yes/no sub-survey per disorder, gated on a relevant core response for that disorder. | 242–280, 326–362, 408–444, 490–528, 575–612, 659–696, 742–778, 887–935, 981–1032 |
| **P09** | Situational multi-select + severity + opt-in | Multi-select context (home/school/friends/activities), a severity select, and a yes/no opt-in to extra details. | 282–306, 364–388, 446–470, 530–555, 614–639, 698–722, 780–805, 825–866, 937–961, 1035–1059 |
| **P10** | Conditional free-text elaboration | A free-text state, gated on the opt-in from P09. | 308–324, 390–406, 472–488, 557–573, 641–657, 724–740, 807–823, 868–885, 963–979, 1061–1077 |

### Phase E — Synthesis and content

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P11** | Model-derived predicted diagnosis | Predicted diagnoses and severity labels are produced from collected inputs and used downstream. | 1240–1314, 1128, 1270–1277, 1299–1311 |
| **P12** | Diagnosis/severity-conditioned result routing | Route to a diagnostic or non-diagnostic result, and to a distress / no-distress variant, from the predicted facts. | 1094–1234, 1240–1314 |
| **P13** | Age-banded content variants | Child (≤7) vs teen (>7) variants of the non-diagnostic result. | 1094–1234 |
| **P14** | Resource recommendation delivery | Attach resource queries (slug, recommendation count, diagnosis, history) to result states and serve them. | 1094–1314, 1459–1465, 1526–1532 |
| **P15** | Report delivery choice + device branch | Offer email or download, with a device-platform variant; then route to the chosen action. | 1319–1341, 1470–1535 |

### Phase F — Actions, messaging, close

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P16** | Email capture with pattern validation + confirm + retry | Capture an email, validate it against a pattern, confirm it, and loop on invalid or rejected input. | 1343–1445, 1571–1690 |
| **P17** | External action with failure handling | Send an email, render a file, or sign up for a newsletter; on failure, route to a failure state. | 1447–1468, 1497–1550, 1653–1696 |
| **P18** | Newsletter signup | A second instance of the P16/P17 shape: intro, signup, confirm, send, invalid, failure. | 1552–1696 |
| **P19** | Closing feedback survey | Services question and response, then satisfaction / recommendation / relevance ratings and optional free text. | 1698–1754 |
| **P20** | Terminal final message | A priority `-99` final message that runs after everything else and ends the conversation. | 1756–1761 |

### Cross-cutting

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P21** | History-relative guard semantics | Guards match the transcript by `prompt_id`, `state_id`, or `user_data_tag`; test last-vs-exists; inspect response shape; reduce to the last match; optionally include transient states. | throughout; e.g. 107–146, 185–238, 1348–1403 |
| **P22** | Prompt presentation modes | Each prompt declares a display mode (slider, toggle, multi-select, checkbox, select, radio, resources), a theme, and behavioural flags. | throughout; e.g. 30, 151–170, 286–292, 1099–1114 |

---

## 3. Cross-cutting semantics

**Selection and priority.** Default priority is 0; the header defines `-99` (final), `-10` (deprioritized), `10` (sequential within a section), `20` (sub-sections), `+10` per nesting level, and `99` (urgent interrupt). The runtime is expected to choose the runnable state with the highest priority. Tie-breaking and whether "highest" means numerically largest or smallest are not stated (Q1).

**Once and transience.** `once {}` appears 49 times: a state that has run is not eligible again. `STATE_TAG_TRANSIENT` (and `include_transient` in history matches) distinguishes states that should not count as the "last" ordinary interaction. The exact bookkeeping — whether a re-asked prompt counts once or twice — is not stated (Q2).

**History predicates.** A `history` constraint carries a `match` (`prompt_id`, `state_id`, or `user_data_tag`), a `position` (`POSITION_PREDICATE_IS_LAST` or `POSITION_PREDICATE_EXISTS`), an optional `response` shape test (`text_like { length | match { query } }`, `bool_like`, `int_like`), an optional `reduce_by` (`SEQ_PREDICATE_LAST`), and an optional `include_transient`. This is a transcript query language, not a variable read (Q3).

**Response shapes.** `text_like` supports character-length comparisons and a `match { query: "…" }` field. The email flows use `[^@]+@[^@]+\.[^@]+`, which is a regular expression, not a literal substring (Q4).

**Tagged aggregation.** `tag { tags, disorders, prompts, responses, count_is, disorder_count_is }` counts responses carrying a prompt tag, optionally scoped to a disorder set. `disorder_count_is` counts distinct disorders with at least one qualifying response. The unit of counting and the tag source are not fully stated (Q5).

**Derived facts.** `risk { assessment: "KeywordAssessment" | "LlamaGuardAssessment" }` is a model/keyword signal. `diagnosis { count_is }` and `tag { disorders { predicted: true } … }` are predictions produced from upstream inputs. Whether these are re-evaluated every turn or computed once is not stated (Q6).

**Content delivery.** `resources { queries { slug { resource_slugs } | recommendation { count } | diagnosis { count, tag_filter } | history { recommendation_type, exempt_resource_ids } } }` is a resource query algebra attached to a prompt (Q7).

**Actions.** `action_spec { action { email_action | render_file_action | newsletter_action }, resources { … }, failure_state_id }` performs an external side effect and names a failure state.

**Presentation.** `display` (slider, toggle, start-multi, checkbox, select, radio, resources), `theme`, and `flag` (end conversation, diagnosis, report, action failure, newsletter, CMI services) are presentation and behavioural metadata, not value semantics (Q8).

---

## 4. Open points in the source

| # | Point | Consequence for assessment |
|---|---|---|
| **Q1** | How are equal-priority runnable states ordered, and is "highest priority" the numerically largest or the most negative? | The dispatch encoding must pin an explicit total order; the assessment uses declaration order as the tiebreak. |
| **Q2** | Does `once {}` mean the state never runs again, or only that it is not selected while a more specific state is eligible? How do transient states interact with `once`? | Determines whether the visited set is global or per-phase. |
| **Q3** | Is `history` evaluated over the raw transcript including silent/transient states, or only user-facing turns? | Determines what must be materialized into state for an equivalent guard. |
| **Q4** | Is `text_like { match { query } }` a full regex, and which dialect? | Email validation depends on regex; ConvoSpec `matches` is substring-only. |
| **Q5** | Does `count_is` count responses, distinct prompts, or turns? Does `disorder_count_is` count distinct disorders or disorder-tagged prompts? | The screener gate and follow-up choice depend on the unit. |
| **Q6** | Are predicted diagnoses and severity tags recomputed per turn, or fixed after the screener? | Affects where the `classify`/`analyze` nodes sit in the graph. |
| **Q7** | What do the `recommendation`, `diagnosis`, and `history` resource queries return, and in what order? | Determines whether a `fetch` filter can reproduce the selection. |
| **Q8** | Are a state's multiple `prompts` one turn or several? | A state with a narrative prompt and a question is two exchanges, not one. |
| **Q9** | Where does `device_info { device_platform }` come from, and is it fixed for the session? | Maps to read-only Context. |
