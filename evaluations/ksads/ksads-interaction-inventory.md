# KSADS DSM-5 — Interaction Inventory (Evidence Base)

Source for the compliance evaluation in [`ksads-compliance.md`](./ksads-compliance.md).

## Source document

- **Title**: Schedule for Affective Disorders and Schizophrenia for School-Age Children — Present and Lifetime Version (K-SADS-PL), DSM-5, November 2016.
- **URL**: https://www.pediatricbipolar.pitt.edu/sites/default/files/assets/AH/AK/KSADS_DSM_5_SCREEN_Final.pdf
- **Length**: 69 pages (front matter i–xiv; Screen Interview pp. 1–52; Supplements; checklists).
- **Retrieval caveat**: fetched through a text proxy; the PDF's embedded fonts produced mojibake (e.g. `'60`, `3 & 6`). Claims below are **paraphrased** and cited by **section name**, not quoted verbatim. Numbers/scale anchors were cross-checked against readable fragments.

## Instrument phases

| # | Phase | Role |
|---|---|---|
| 1 | Unstructured Introductory Interview | ~15 min; rapport, demographics, presenting complaint, history, functioning. |
| 2 | Screen Interview | Surveys primary symptoms of every disorder; drives which supplements run. |
| 3 | Supplement Completion Checklist | Records which supplements are required + dates of possible episodes. |
| 4 | Diagnostic Supplements (×5) | Deep-dive per disorder; administered conditionally, in onset order. |
| 5 | Summary Lifetime Diagnostic Checklist | Final synthesis across informants; clinician resolves discrepancies. |

## Cross-cutting semantics

- **Scoring scale (majority of items)**: `0 = No information`, `1 = Not present`, `2 = Subthreshold`, `3 = Threshold`. A minority use a 0–2 scale (`0 = No information`, `1 = No`, `2 = Yes`).
- **Threshold determination**: severity + frequency + duration + impairment; explicit anchors are given per item (e.g. depressed mood "more days than not, most of the day, ≥50% of awake time").
- **Two timeframes per item**: Current Episode (CE) and Most Severe Past (MSP). Both rated for each item.
- **Three informant columns per item**: Parent, Child, Summary. Parent and child are interviewed **separately**; the Summary rating is a clinical synthesis.
- **Administration order**: sections need not be surveyed in order; the interviewer may start with presenting-complaint-relevant diagnoses, but **all sections must be completed**.
- **Anchoring**: the "current" window is defined a priori per study; "past" = most severe between the last interview and the current window; an episode is "resolved/past" after ≥2 months symptom-free.
- **Longitudinal use**: ratings are "since last interview"; a separate follow-up checklist captures course.

## Interaction patterns

| ID | Pattern | Purpose | Inputs | State produced | Control-flow character | Source section |
|---|---|---|---|---|---|---|
| P01 | Unstructured rapport interview | Build rapport; elicit narrative context | Free speech | Notes / narrative slots | Open, interviewer-driven dialogue | Introduction; Guidelines for the Introductory Unstructured Interview |
| P02 | Demographics / subject info capture | Record identity, living situation, informants | Typed fields, multi-select | Structured demographic slots | Linear form | Subject Information; Caregiver Information |
| P03 | Multi-informant item rating | Rate one item from Parent and Child independently, then synthesize | Two informant responses | `{parent, child, summary}` per item | Fan-out to 2 informants → merge | Every screen item (Parent/Child/Summary columns) |
| P04 | Dual-timeframe rating (CE/MSP) | Rate each item for current and most-severe-past | Same item, two windows | `{ce, msp}` per item | Paired ratings | Every screen item (CE / MSP rows) |
| P05 | Semi-structured probe → clarify → rate | Suggest probes, follow up adaptively, then assign a code | Symptom stem + candidate probes | Rating + rationale | Adaptive sub-dialogue → structured output | Screen Interview administration rules; each item's probes |
| P06 | Rubric-based scoring | Map severity/frequency/duration/impairment to 0–3 or 0–2 | Evidence gathered in P05 | Integer code | Deterministic mapping | Scoring rules; per-item anchors |
| P07 | Per-disorder skip-out | Decide whether a disorder's supplement is needed | Primary-symptom ratings | Skip-out flag | Branch on ratings | Each disorder's "Skip Out Criteria" |
| P08 | Score-gated supplement trigger | Trigger a supplement if any item crossed threshold | A **set** of item ratings | Trigger flag | Conditional on **aggregate over a set** | e.g. Depression, Alcohol Use, Substance Use, PTSD, Autism |
| P09 | Dynamic supplement dispatch + onset-order sequencing | Choose and order the supplements to run | Screen results + onset history | Supplement worklist/order | Runtime selection + ordering of subgraphs | Diagnostic Supplements; Supplement Completion Checklist |
| P10 | Any-order navigation with completion tracking | Interviewer picks next section; all must be done | Interviewer choice | Completion set | Non-linear navigation + worklist loop | Screen Interview introduction rules |
| P11 | Supplement Completion Checklist | Record required supplements + episode dates | Checklist selections, dates | Checklist state | Structured checklist | Supplement Completion Checklist |
| P12 | Summary Lifetime Diagnostic Checklist | Synthesize lifetime/current diagnoses | All collected ratings | Final diagnoses | Synthesis + clinician decision | Summary Lifetime Diagnostic Checklist |
| P13 | Clinician discrepancy resolution | Reconcile divergent parent/child reports | Divergent ratings | Resolved rating | Conditional on disagreement + clinician input | Introduction (discrepancy guidance) |
| P14 | Family-history grid | Score relatives × conditions | 2-D grid entries | Grid of 0–3 scores | Nested iteration over two dimensions | Family History for Biological Relatives |
| P15 | Trauma "ever" screening | Screen traumatic event types, then gate PTSD section | Per-event Ever ratings | Ever flags → PTSD gate | List iteration + gated follow-up | Post Traumatic Stress Disorder (Traumatic Events) |
| P16 | Substance-use list + frequency | Per-drug frequency CE/MSP; gate substance section | Per-drug ratings | Per-drug CE/MSP codes | List iteration + gated follow-up | Substance Use Disorders; Alcohol Use Disorders |
| P17 | Risk / safety escalation | Detect suicidality; escalate | Ideation/act/lethality/NSSI codes | Risk flag | Risk-gated escalation | Suicide section |
| P18 | Timeframe anchoring rules | Define CE window and MSP window | Study config | Anchor metadata | Cross-cutting config | Introduction; Longitudinal Studies |
| P19 | Informant-specific adaptation | Parent-first vs adolescent-first; adapt wording | Age / informant | Adapted prompts | Branch + prompt adaptation | Guidelines for Administration |
| P20 | Longitudinal follow-up | Re-rate "since last interview" | Prior session data | Longitudinal deltas | Cross-session synthesis | Longitudinal Studies; Follow-up checklist |

## Notes for the compliance evaluation

- P03, P08, P09, P10 are the patterns most likely to strain a general conversation-graph model.
- P08, P10 (and P16, P15 by extension) all require reasoning over a **collection** of item ratings, not a single scalar.
- P09 requires **runtime selection and ordering** of reusable subgraphs.
- P05 requires **multi-turn** dialogue inside a single logical "item".
