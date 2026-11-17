# GuideManifold (Adaptive Scoring) — Interaction Inventory (Evidence Base)

| Field | Value |
|---|---|
| Instrument | GuideManifold demo — adaptive conversational scoring of GAD-7 and PHQ-9 items |
| Source | [`adaptive.md`](../../adaptive.md) |
| Scope assessed | One session: free-text conversation, per-turn silent scoring, uncertainty-aware estimate, active turn selection over 22 candidates, transparency display, and dual termination |
| Explicitly out of scope | Training data and simulation, demo mode, model architecture (CNP), UI layout, and the scorer backend choice (Kev vs LLM) |

This document is the evidence base for [`adaptive-compliance.md`](./adaptive-compliance.md). It decomposes the instrument into interaction patterns and states. It does not judge ConvoSpec support; that is the compliance document's job.

**Citation convention.** `P<n>` refers to a pattern below. `Q<n>` refers to an open point in §4.

---

## 1. Shape of the instrument

GuideManifold sits between Walrus and Ask Kai: it *looks* like a free conversation but is silently driving a structured assessment.

After each user message:

1. **Scoring** — an LLM reads the message into 16 GAD-7/PHQ-9 items, each "not mentioned" or a 0-3 rating.
2. **Estimate** — a conditional neural process (CNP) takes all scored turns and outputs a probability distribution over every item value plus totals.
3. **Turn selection** — the model predicts expected information gain from each of 22 candidate turn types (16 probes + 6 conversational moves), subtracts a probe penalty λ, and picks the highest-scoring candidate.
4. **Reply** — an LLM generates the chosen turn.

The conversation terminates when a **maximum turn budget** is reached or when **sufficient evidence** is collected (confidence above a threshold). After every reply the user **sees** the current estimate with calibrated uncertainty.

---

## 2. Patterns

### Phase A — Conversation loop

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P01** | Free-text user exchange | The user writes freely; the bot replies with a generated turn. One exchange per turn. | "After each user message" |
| **P02** | Silent per-turn scoring | After the user replies, an LLM scores the message into 16 item ratings (0-3 or "not mentioned"). Silent — does not create a turn. | Step 1 |
| **P03** | Estimate update with uncertainty | A model takes all (turn, score) pairs and outputs probability distributions over 16 items plus GAD-7 and PHQ-9 totals. Silent. | Step 2 |
| **P04** | Active turn selection over candidate set | Predict expected information gain from 22 candidates; subtract probe penalty λ (doubled for consecutive probes); pick the winner. Silent. | Step 3 |
| **P05** | LLM turn generation | Write the chosen turn kind as a real message. | Step 4 |
| **P06** | Transparency display | After every reply, show the current estimates and their uncertainty to the user. | "shows what it believes after every reply" |

### Phase B — Turn candidates and conversational moves

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P07** | Candidate turn registry | 22 candidates: 16 targeted probes (one per item) + 6 conversational moves (open question, reflection, reassurance, advice, psychoeducation, small talk). | "22 candidate chatbot turns" |
| **P08** | Probe penalty and back-to-back doubling | A fixed λ subtracted from targeted questions; doubled when the previous turn was also a probe. | "probe penalty λ ... doubled for back-to-back probes" |
| **P09** | Non-probe conversational moves | The bot can choose open, reflective, reassuring, advisory, educational, or small-talk turns — not just direct questions. | 6 moves listed |

### Phase C — Termination

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P10** | Turn-budget termination | The loop ends when a maximum number of turns is reached. | "maximum number of turns" |
| **P11** | Confidence-threshold termination | The loop ends when sufficient evidence is collected to make a deliberation above a predefined confidence threshold. | "sufficient evidence ... confidence threshold" |

### Phase D — Lifecycle

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P12** | Session state accumulation | Each turn appends a (turn-type, scored-replies) pair to the running transcript used by the CNP. | "the conversation so far — every (chatbot turn, scored reply) pair" |

---

## 3. Cross-cutting semantics

**Silent pipeline.** Steps 1-3 (score, estimate, select) are all silent — they execute between turns and must not block the conversation or create turn boundaries. They are a **silent pipeline**: `score → estimate → select`, running after each `ask`/`llm` exchange.

**Scorer noise.** The LLM scorer is noisy: it rates revealed items, and sometimes rates items that were never mentioned ("often"). The CNP is trained to this noise profile. This is a model property, not a conversation property.

**Probe penalty.** λ is a fixed constant; the back-to-back doubling requires remembering whether the previous bot turn was a probe. This is state, not a new capability.

**Estimate transparency.** The user sees the estimate after every reply. This is a `say` of the current state values, not a special node.

**Item coverage.** 16 items × 4 values (0-3) plus two totals. The estimate is a distribution, not a point value. The spec does not need to carry distributions — the CNP's output is materialised as state.

**Self-harm avoidance.** "By default the chatbot never asks about self-harm (PHQ-9 item 9) directly; anything the person says about it is still scored." This is a candidate exclusion rule, not a new capability.

---

## 4. Open points in the source

| # | Point | Consequence for assessment |
|---|---|---|
| **Q1** | What is the exact confidence threshold and how is it computed? | The exit condition is a boolean over the CNP's output; the precise formula is a model detail, not a conversation primitive. |
| **Q2** | Is the CNP re-run every turn, or cached? | Irrelevant to the spec: a view is a pure function of state regardless of caching. |
| **Q3** | How are the 22 candidates ordered for tie-breaking? | Declaration order is the natural tie-break, as with Ask Kai. |
| **Q4** | Is the transparency display one message or several? | One or more `say` nodes after the exchange; mechanical. |
| **Q5** | Does the bot have an opening turn? | If so, a `say` or `ask` before the loop; not stated explicitly. |
