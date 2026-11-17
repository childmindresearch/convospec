# Spec Consolidation Review — v0.6.0 RC1

| Field | Value |
|---|---|
| Spec version | `spec.md` **v0.6.0** (Draft → RC1) |
| Date | 2026-11-16 |
| Scope | Node taxonomy: 19 types across 6 families |
| Method | Pairwise comparison of node contracts (ports, `await`, config, effects, requirements) after six instrument evaluations |

This document evaluates whether the current node taxonomy can be simplified — merging or removing near-duplicates — without losing necessary capabilities. It was prompted by the completion of the sixth instrument evaluation (GuideManifold), which confirmed all 19 types are exercised and no capability gaps exist.

---

## Node Taxonomy

| Family | Nodes | Count |
|---|---|---|
| Boundary | `start`, `end` | 2 |
| Interaction | `ask`, `say` | 2 |
| Cognition | `llm`, `analyze`, `classify` | 3 |
| Control | `branch`, `decide`, `sample`, `fork`, `join`, `loop`, `foreach`, `call` | 8 |
| State | `set` | 1 |
| External | `fetch`, `tool`, `handoff` | 3 |
| **Total** | | **19** |

---

## Pairwise Consolidation Analysis

### 1. `analyze` → fold into `llm` (silent mode)?

**Overlap.** Both invoke a model backend, both are `await: none`, both write to `slot`, both have `error`. `llm` already has `mode: silent`.

**Deltas:**

| Feature | `analyze` | `llm` (silent) |
|---|---|---|
| Backend | any model or service | `model` |
| Inputs | `inputs: [state.x]` | `messages` / `prompt_template` |
| Failure semantics | `default` value on failure | no `default` — takes `error` |
| Fork-safe | explicit requirement | implicit (silent = `await: none`) |

The real delta is **`default`** — `analyze` writes a declared fallback value on failure rather than taking the `error` edge. This is a *fault-tolerance* contract: a risk detector that fails should degrade gracefully, not crash the conversation.

**Verdict: keep.** Adding `default` to `llm` would blur the distinction between graceful degradation and hard failure. Every `llm` node would need a guard against whether `default` is declared. The explicit marker ("this is fault-tolerant") is worth the second node.

---

### 2. `classify` → fold into `analyze` or `llm`?

**Overlap.** `classify` is a model call with a closed label set and optional confidence.

**Deltas:**

| Feature | `classify` | `analyze` | `llm` (silent) |
|---|---|---|---|
| Backend | `llm`, `bert`, `predictive`, `rules` | any | `model` |
| Labels | closed `labels` set | — | — |
| Confidence | `confidence_slot`, `threshold` | — | — |
| Error semantics | wrong label → `error` | failure → `default` | failure → `error` |
| Seed | stochastic backends honour seed | — | — |

`classify` adds **closed-label validation** (output *must* be exactly one value from `labels`), **confidence threshold routing**, and **non-LLM backends** (`bert`, `predictive`, `rules`).

**Verdict: keep.** The closed-label contract is not just an `output_schema` — it's a runtime-enforced validation rule. Dropping `classify` means every graph reimplements label validation with `set` + `branch`. The node pays for itself.

---

### 3. `decide` → fold into `branch`?

**Overlap.** Both route on a boolean outcome.

**What `decide` adds:**
- Multi-operand aggregation (`any`, `all`, `none`, `n_of_k`)
- `on_missing` policy (`unknown` / `skip` / `fail`) for null operands
- `negate` on individual operands
- Four-port routing including explicit `unknown`

**Can `branch` express it?** Yes — `any(state.a, state.b)` is a guard expression. But the `unknown` semantics are buried in guards rather than declared.

**Verdict: keep.** Driven by KSADS: multi-informant scoring where a missing rating is a *routable outcome*, not a silent false. `on_missing: unknown` is a clinical requirement, not a convenience.

---

### 4. `fork` / `join` → fold into one `parallel` node?

**Overlap.** Always appear as a pair.

**Verdict: keep.** The graph model benefits from explicit split and join points: it makes the control flow statically inspectable and lets the validator check that every stream is silent (§6.4, VR-11). Folding them hides the stream topology inside one node's config.

---

### 5. `loop` / `foreach` → fold into one?

**Overlap.** Both iterate with `max_iterations` and `exit`.

**Deltas:**

| Feature | `loop` | `foreach` |
|---|---|---|
| Source | guard expression | list/registry snapshot |
| Element binding | — | `item_slot`, `index_slot` |
| Ordering | — | `order_by`, `tiebreak`, `filter` |
| Body dispatch | static node id | per-element `body_graph` |

`foreach` adds element binding, snapshot semantics, ordering, and per-element dispatch.

**Verdict: keep.** A unified node would be a `foreach` with an optional empty `over` — more complex, not less.

---

### 6. `fetch` / `tool` → fold into `external`?

**Overlap.** Both call external services, both have `error`/`timeout`, both write to state.

**Deltas:**

| Feature | `fetch` | `tool` |
|---|---|---|
| Semantics | **Read** (safe, idempotent) | **Side effect** (write, update) |
| Credentials | runtime secret store (VR-19) | — |
| Caching | `cache_ttl_s` | — |
| Idempotency | — | `idempotency_key` |
| Source | declared `data_sources` | `tool_ref` (no declaration) |

Opposite ends of the effect spectrum: safe read vs arbitrary side effect.

**Verdict: keep.** Merging would either strip `fetch`'s safety guarantees or inflate `tool`'s surface with credential/caching semantics. The read/write distinction is important for auditing and replay.

---

### 7. `handoff` → fold into `tool`?

**Overlap.** Both interact with external actors.

**Deltas:**

| Feature | `handoff` | `tool` |
|---|---|---|
| `await` | `external` (pauses graph) | `none` (fire-and-forget) |
| Resume | `continue` or `terminate` | — |
| Event | `target` + `reason` emitted | — |

`handoff` is the only node with `await: external` — it pauses the conversation. `tool` cannot pause.

**Verdict: keep.** The `await` contract is the defining feature.

---

## Summary

| Candidate | Verdict | Reason |
|---|---|---|
| `analyze` → `llm` | **Keep** | Fault-tolerance (`default`) vs hard-error semantics |
| `classify` → `analyze`/`llm` | **Keep** | Closed-label contract + confidence + non-LLM backends |
| `decide` → `branch` | **Keep** | `on_missing` unknown routing is a clinical requirement |
| `fork`/`join` → one | **Keep** | Split/join topology is statically inspectable |
| `loop`/`foreach` → one | **Keep** | Element binding + snapshot + ordering are core |
| `fetch`/`tool` → one | **Keep** | Read vs side-effect semantics are opposite |
| `handoff` → `tool` | **Keep** | `await: external` pause semantics |

**Conclusion: the taxonomy is tight.** Every node type carries a distinct contract — in ports, `await`, effects, or error semantics. No two nodes are redundant. The six-instrument evaluation suite (KSADS, DASH, MoBI, Walrus, Ask Kai, GuideManifold) exercises all 19 types and finds no capability gaps, which is itself evidence that the types are right-sized: too few would leave gaps, too many would create dead weight.

The only genuinely close call is `analyze`/`llm` (silent mode overlap), and that survives on fault-tolerance semantics alone. Consolidation is not warranted.
