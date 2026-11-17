# Open-Ended Walrus Conversation — Interaction Inventory

| Field | Value |
|---|---|
| Instrument | Open-ended free conversation with a phrase and turn-cap terminator |
| Source | Task statement (user, 2026-11-16), quoted in §1 |
| Scope assessed | One session: random opener, unbounded chat, two termination conditions |
| Explicitly out of scope | Bot persona and content policy, model choice, UI, storage of the transcript outside the session |

This document is the evidence base for [`walrus-compliance.md`](./walrus-compliance.md). It decomposes the instrument into interaction patterns and states, for each, what varies and what the runtime must do. It does not judge ConvoSpec support; that is the compliance document's job.

**Citation convention.** `P<n>` refers to a pattern below. `Q<n>` refers to an open point in §4.

---

## 1. Shape of the instrument

The task statement:

> an almost entirely unstructured conversation that begins with a randomly selected moderately benign prompt like 'what is on your mind' and then has an endless llm-interaction conversation until the user utters the phrase "you MUST be a walrus!" or 100 turns have taken place.

The instrument is deliberately the **inverse** of the KSADS, DASH, and MoBI evaluations. Those instruments are structured: a fixed item order, typed responses, scores, and routing that depends on answers. This one has:

- **No fixed content.** The conversation is generated turn by turn with no agenda, no item bank, and no scoring.
- **No content-dependent routing.** Nothing the participant says changes the path, *except* one literal phrase.
- **One variable at the start.** The opening prompt is drawn at random from a benign pool.
- **Two stop conditions.** A literal phrase, or a turn cap.

So the graph is a straight line with one bounded cycle and two exits. The interesting questions are all about the *edges* of the interaction: how the opener is chosen, how many turns have elapsed, how the phrase is matched, and how the conversation is carried across turns.

---

## 2. Patterns

### Phase A — Session setup

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P01** | Session identity and seed | One session per participant; records a seed that fixes the random opener and is available for replay. | §1 |
| **P02** | Random opener selection | Choose one prompt from a pool of moderately benign openers ("what is on your mind", …). Selection must be reproducible from the session seed. | §1 |
| **P03** | Opening turn | Present the chosen opener as the first user-facing message and await the participant's first reply. | §1 |

### Phase B — Free conversation

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P04** | Open-ended bot turn | Generate exactly one reply with the model, show it, and await exactly one participant reply. No fixed prompt content; the model drives the text. | §1 |
| **P05** | Conversation continuity | Each bot reply must see the full conversation so far — the opener, every prior bot reply, and every prior participant reply — so the chat does not reset. | §1 |
| **P06** | Unbounded continuation | Keep taking turns with no agenda and no content-dependent branch. The only ways out are the two terminators. | §1 |
| **P07** | Turn accounting | Count turns as they happen and make the count readable so the cap can be enforced. | §1 |

### Phase C — Termination

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P08** | Termination-phrase detection | Test the participant's latest utterance for the literal phrase "you MUST be a walrus!". | §1 |
| **P09** | Phrase-triggered termination | On detecting the phrase, stop the conversation and conclude the session. | §1 |
| **P10** | Turn-cap termination | Stop when 100 turns have taken place, whether or not the phrase appeared. | §1 |
| **P11** | Termination-reason recording | Record which terminator fired (phrase vs. cap) and how many turns elapsed. | §1 |

### Phase D — Lifecycle

| # | Pattern | What varies / must happen | Source |
|---|---|---|---|
| **P12** | Participant abandonment | Handle a participant who stops responding, so the session does not hang indefinitely. | — |
| **P13** | Session close and transcript export | End the session and make the ordered transcript available for analysis. | — |
| **P14** | Safety and cost bounds | Bound wall-clock time and token spend independently of the turn cap. | — |

---

## 3. Cross-cutting semantics

**Determinism.** The only random decision is the opener (P02), which must be a pure function of the session seed. Everything else is deterministic given the participant's inputs, except model output, which is explicitly out of scope for reproducibility but must be recorded verbatim (spec §3.4, §7.5).

**Prompt composition.** The bot's reply is assembled from a system/persona block (if any) plus the conversation history. The opener is *not* model-generated — it is a fixed prompt chosen from a pool — so the conversation starts with a participant-visible message that the model did not produce. Whether and how that opener appears in later prompts is the one prompt-assembly subtlety here (P05).

**Phrase matching.** P08 is the whole content-dependence of the graph. "Utters the phrase" is stated literally, so the natural reading is an exact-string test on the participant's message. But a real participant may type `you must be a walrus`, add punctuation, or bury it in a sentence. Exactness, case sensitivity, and substring-vs-whole-message are unresolved (Q2) and are load-bearing: they decide whether the session ends or runs to the cap.

**Turn semantics.** "100 turns have taken place" needs a unit. The spec's own unit is the *interaction* — one user-facing prompt plus the reply that followed (§3.1, §7.1). The opener is one interaction; each bot turn is another. Whether the cap counts interactions, bot messages, or participant messages is unresolved (Q1) and shifts the effective length by one.

**No structure to encode.** There are no typed responses, no scores, no views, no assignment, and no subgraphs. This instrument exercises the *core loop, branching, and model-call* machinery and almost nothing else.

---

## 4. Open points in the source

| # | Point | Consequence for assessment |
|---|---|---|
| **Q1** | What counts as a "turn" for the 100-turn cap — interactions, bot messages, or participant messages? Does the opener count? | The assessment uses the spec's interaction unit and notes the off-by-one; the graph can encode either. |
| **Q2** | "Utters the phrase" — exact message, substring, case-insensitive, punctuation-stripped? | Determines whether the graph needs a string-normalization step before matching. |
| **Q3** | Is the opener pool fixed, and is selection independent per session or drawn without replacement across a batch? | Independent per session is assumed; without-replacement would add assignment machinery. |
| **Q4** | Is there a system prompt / persona for the bot, or is it fully open? | Assessed as an optional leading `system` message. |
| **Q5** | What happens if the participant stops responding (timeout vs. indefinite wait)? | Assessed as a timeout edge plus a graph-level wall-clock budget. |
| **Q6** | May the bot ever end the conversation, or only the participant and the cap? | The source names only the two terminators; the graph is built to match. |
