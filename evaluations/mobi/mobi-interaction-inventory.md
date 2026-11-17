# MoBI×MSM Structured Chat Trial — Interaction Inventory

| Field | Value |
|---|---|
| Source | [`MoBIxLLM_task_summary_for_GK_20261116.md`](../../MoBIxLLM_task_summary_for_GK_20261116.md) |
| Author / date | @Dan Lurie, 2026-11-16 |
| Scope assessed | One structured chat trial plus the factors that vary across trials |
| Explicitly out of scope | Rest of the session (calibration, reference tasks, self-directed chat, questionnaires), story/item wording, TBD values |

This document is the evidence base for [`mobi-compliance.md`](./mobi-compliance.md). It decomposes the task summary into interaction patterns and states, for each, what varies and what the runtime must do. It does not judge ConvoSpec support; that is the compliance document's job.

**Citation convention.** `L<n>` refers to a line in the source summary. `P<n>` refers to a pattern below.

---

## 1. Shape of the instrument

The task is a **within-subject experiment**, not a single interview. Three factors vary across trials inside one participant:

| Factor | Levels | Set by | Scope |
|---|---|---|---|
| Position type | INF, DEC, REF (L15–L25) | Story pool | Per trial |
| Chatbot stance | Aligning, calibrated, counterbalancing (L27–L42) | System-prompt instruction | Per trial, both bot replies |
| Reply question | Question / no question (L44–L46) | Instruction to the chatbot | Per trial, both bot replies |

A **trial** is one story run through 14 fixed steps (L59–L82). The unit of repetition is therefore the trial, and the unit of variation is the (type × stance × question) cell. The number of trials is TBD but constrained to a multiple of 9, or 18 when question is fully crossed (L57).

Two design properties matter more than the conversation content:

1. **Assignment.** Each participant must receive a balanced set of cells, in a randomized order, with stories drawn without replacement from per-type pools (L48–L53).
2. **Repeated measurement.** Steps 2–4 are re-asked verbatim at steps 11–13 (L75–L77), so the trial is a pre/post design with a chat in the middle.

---

## 2. Patterns

### Phase A — Session setup

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P01** | Session identity and design parameters | One session per participant; records participant id, seed, trial count, and the realized assignment. | L5–L7, L57 |
| **P02** | Balanced randomized condition assignment | Build the set of (type × stance × question) cells, replicate to the planned trial count, shuffle within blocks, and fix an order. Must be reproducible from the session seed. | L27–L57 |
| **P03** | Story sampling without replacement | Each position type has a pool of 12+ stories (L50); each participant receives a subset, different participants different subsets, no story repeated within a participant. | L48–L51 |
| **P04** | Option order and label counterbalancing | Each story has exactly two options; display order and Option A/B labels are counterbalanced (L52). The order must stay fixed for the trial once chosen. | L52 |
| **P05** | Practice trial | One trial with the same structure and the calibrated stance, run before the measured trials (L57). | L57 |

### Phase B — Trial, pre-chat

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P06** | Story presentation | Story text plus two options, shown at step 1 and visible for the whole trial. | L65 |
| **P07** | Forced binary choice (relative fit) | Two-way forced choice: which option is closer to what the participant would do (DEC) or think (INF, REF). | L21–L23, L66 |
| **P08** | Choice certainty | Likert rating attached to the choice. | L67 |
| **P09** | Absolute fit | Likert rating of how close the chosen option is to the participant's own position. | L68 |
| **P10** | Justification | Free text explaining the choice; this is the first chat message. | L69 |

### Phase C — Trial, chat

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P11** | Stance-conditioned bot reply | Generate a reply from: fixed base system prompt + stance instruction + question instruction + story context + pre-chat responses (+ INF evidence summary, planned). Stance rules constrain content ("nothing false", never invalidate, REF negative self-appraisal, INF partial evidence). | L31–L42, L86–L96 |
| **P12** | Participant reply 1 | Free text, appended to conversation history. | L71 |
| **P13** | Second bot reply with conversation history | Generate a reply that sees the story, the pre-chat responses, and both prior chat messages. | L72, L86 |
| **P14** | Participant reply 2, terminal | Free text; no bot reply follows. | L73 |
| **P15** | Crisis stop rule | If a generated reply contains crisis resources: send it unchanged, end the trial, show a crisis-resources page. | L82 |

### Phase D — Trial, post-chat

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P16** | End-of-trial message | Fixed text, shown after reply 2 is sent. | L74 |
| **P17** | Post binary choice | Same item as step 2, re-asked. | L75 |
| **P18** | Post choice certainty | Same item as step 3, re-asked. | L76 |
| **P19** | Post absolute fit | Same item as step 4, re-asked. | L77 |
| **P20** | Activation rating | Likert rating of current arousal. | L78 |

### Phase E — Trial lifecycle and data

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P21** | Trial record assembly | One record per trial: factor values (type, stance, question), story id, option order, all 7 responses, any covariates, and whether the crisis rule fired. | L48–L53, L59–L82 |
| **P22** | Repetition over trials with isolation | Trials repeat; per-trial state must not leak between trials except through the trial record. | L7, L55–L57 |
| **P23** | Early trial termination | The crisis rule ends a trial before step 14; the remaining steps are skipped. | L82 |
| **P24** | Session close and export | Session ends; trial records and design metadata are available for analysis. | L5–L7, L53 |

---

## 3. Cross-cutting semantics

**Determinism.** Assignment, story selection, and option order must all be reproducible from the session seed; analysis depends on knowing exactly which cell each trial was (L48–L57). Model outputs are not reproducible and must be recorded verbatim (L86–L96).

**Prompt composition.** The bot receives a fixed base system prompt plus per-trial inserts (L86–L96). The inserts are the mechanism by which stance, question, story, and pre-chat responses reach the model. The table's "Passed to the bot?" column (L63–L78) is a per-variable disclosure decision: steps 4, 9, 10, 11–14 are deliberately withheld.

**Safety constraints are prompt-level.** "Nothing false", "never invalidate feelings", the REF negative-appraisal rule, and the INF partial-evidence rule (L37–L42) are instructions to the model, not routable outcomes. They constrain *how* a reply is generated, not what the graph does next.

**Measurement, not conversation.** Physiology, keystrokes, and screen are recorded throughout (L7) but are outside the conversation graph. Story covariates such as personal relevance and valence (L53) are measured, not manipulated.

**Out-of-scope boundary.** Persistence of the story display across the trial (L65) and the crisis-resources page (L82) are rendering concerns. The graph is responsible for the *values* and the *routing*, not the layout.

---

## 4. Open points in the source

| # | Point | Consequence for assessment |
|---|---|---|
| 1 | Trials per participant is TBD (multiple of 9 or 18) (L57, L109) | The graph must be parameterized, not hard-coded. |
| 2 | Likert scale points and labels are not given (L107) | Assessment treats ratings as `scale` with an undeclared range. |
| 3 | Whether the session continues after a crisis-terminated trial is not stated (L82) | Both readings are assessed; the graph can express either. |
| 4 | INF evidence summaries are "planned", not specified (L96, L110) | Assessed as an optional prompt insert. |
| 5 | Stance rules are content constraints with no machine-checkable definition (L37–L42) | Treated as prompt text, not validation. |
