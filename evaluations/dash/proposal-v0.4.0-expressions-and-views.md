# Proposal: ConvoSpec v0.4.0 — Expressions, Collections, and Derived Views

| Field | Value |
|---|---|
| Status | **Applied** — integrated into `spec.md` v0.4.0 (integration notes in [§13](#13-integration-notes)) |
| Target spec version | 0.4.0 |
| Baseline | `spec.md` v0.3.0 |
| Source evaluation | [`dash-compliance.md`](./dash-compliance.md) |
| Evidence base | [`dash-interaction-inventory.md`](./dash-interaction-inventory.md) |
| Review input | User review of DASH G1/G2/G3, 2026-11-16; decisions confirmed and extensibility policy added, 2026-11-16 |

This document proposed changes to `spec.md`. They are now **applied in `spec.md` v0.4.0**; [§13](#13-integration-notes) records integration notes and every deviation from the text below. The body of this document is preserved as the reviewed proposal.

**Citation convention.** `§X` and `VR-N` refer to `spec.md` **v0.3.0**. References to this document are written `this §X`. Sections marked *(new)* are proposed spec sections that do not exist yet.

---

## 1. Status and scope

The DASH compliance evaluation ([`dash-compliance.md`](./dash-compliance.md) §5) found three narrow gaps, none of which blocked a faithful interview:

| Gap | Evaluation's finding | Disposition here |
|---|---|---|
| **G1** Declarative numeric scoring | `set` does scalar arithmetic but has no aggregate over item sets and no enum→number mapping | **Accepted.** A typed standard-function library in the expression grammar: `sum`, `count`, `len`, `avg`, `min`, `max`, plus collection/object constructors (this §3) |
| **G2** Nested (2-D) iteration | Nested `foreach` unvalidated; workaround is outer `foreach` + inner `call` | **No patch.** Keep logic flat; the 2-D grid stays an authoring pattern, not a language feature (this §5) |
| **G3** Count-driven repetition + state assembly | No range constructor, no list append/collect, no way to assemble a section's responses into one value | **Accepted and expanded.** Declared derived `views` (live, read-only projections), `set` as a general state transformer, and a complete constructor set on lists/objects (this §4) |

Two constraints shape everything below:

1. **Additive only.** No existing node, field, or validation rule changes meaning. v0.3.0 graphs remain valid.
2. **No new node types.** The capability comes from the expression grammar, one new graph-level `views` block, and a widened `set`. The node taxonomy stays flat.
3. **Portable core, extensible edges.** The mandated library is small and spec-defined; further operations are declared implementation extensions, so a graph that uses none runs on every conforming implementation (this §3.7).

The goal, per review, is a **fully expressive conversational language**: state can be assembled from other state, aggregated, and read back as ordinary values, without special-casing instruments.

### 1.1 Summary of proposed changes

| Kind | Addition |
|---|---|
| Grammar (§5.3) | List and object literals; typed function calls; element-scoped arguments (`item`) for collection functions |
| Function library *(new §5.5)* | Numeric aggregates, collection constructors/operators, object operators, string ops, conversions, `if`/`coalesce`/`lookup`, `date_diff` — the **normative minimum** |
| Extensibility *(new §5.6)* | Core tier (MUST, spec-defined) vs. declared implementation extensions (MAY, manifest-typed); grammar is never extended |
| Graph field (§6.1) | `views` — declared, read-only, deterministically recomputed derived values |
| Reference namespace | `views.<name>`, usable wherever `state.*` is usable (expressions, guards, `decide`, `foreach`, templates) |
| `set` (§4.5) | May write `list<T>`/`object`; all assignments evaluated atomically against pre-node state |
| Canonical pattern *(new §6.5.3)* | Section collection: assemble a section's responses into a named view |
| Validation rules | VR-26 … VR-30 (provisional numbering) |
| Amended rules | VR-7, VR-8, VR-20, VR-23 (this §6.3) |
| Not changed | `decide`, `branch`, `foreach`, `loop`, `call`, `graph_registry`, session model |

---

## 2. Review decisions and their consequences

| # | Decision | Consequence for the spec |
|---|---|---|
| **D1** | Numeric aggregates are **expression-level functions**, not new `decide` tests. | `sum`/`count`/`len`/… live in §5.3 and are usable in `set`, `branch` guards, `loop.exit_when`, templates, and `views`. `decide` stays a boolean router. |
| **D2** | **G2 gets no patch.** Logic stays flat. | Nested `foreach` remains out of scope; the 2-D grid is outer `foreach` + inner `call`/unroll. §11 *Nested iteration* stays open. |
| **D3** | State management is **declared, not hidden**. | A graph-level `views` block names derived values and their sources explicitly. No `ask` node silently appends to a collection by default. |
| **D4** | Values are **immutable**; operations return new values. | No in-place mutation. Deterministic accumulation is an explicit `set` over a list slot inside `foreach`, with atomic-assignment semantics. |
| **D5** | Views are the general **state-projection** mechanism. | Views assemble and derive values from state. Any consumer that needs a derived value — a later node, a summary message, an external reader — reads `views.*` as an ordinary reference. Display is one incidental consumer, not the design driver: **no renderer, dashboard, or reporting node is proposed.** |
| **D6** | Operations lean on implementation engines, with a **mandated minimum**. | The core library is normative and MUST be supported with spec-defined semantics. Implementations MAY expose further pure host operations as *declared, typed* extensions (this §3.7). Grammar stays spec-owned. |

D1 and D3 both keep the spec inspectable: every aggregate and every derived collection is enumerable from the graph source. D4 keeps replay sound: no aliasing, no order-dependent mutation. D2 is a deliberate refusal to grow the control-flow surface.

---

## 3. G1 — Typed expression function library

### 3.1 Motivation

The DASH screener prints 25 calculated section scores (p56) and the FTND prints a total (line 25969). In v0.3.0, `set` can evaluate `a + b + c`, but it cannot:

- sum or count over a **collection** of responses;
- map an enum (`No`/`Yes`/`Unk`) to a number;
- conditionally weight an item.

The workaround — a `tool`/custom compute node, or changing every probe to a 0/1 integer — either adds an external dependency or distorts the interaction. The fix belongs in the expression language.

### 3.2 Grammar changes (§5.3)

Add list/object literals and generalize `predicate_call` to a typed `call`. The existing `local_ref` production is reused for element-scoped arguments.

```ebnf
primary        = comparison
               | call
               | list_literal
               | object_literal
               | "(" , guard , ")" ;

call           = identifier , "(" , [ expr , { "," , expr } ] , ")" ;

list_literal   = "[" , [ expr , { "," , expr } ] , "]" ;
object_literal = "{" , [ obj_entry , { "," , obj_entry } ] , "}" ;
obj_entry      = ( identifier | string ) , ":" , expr ;

local_ref      = "item" , { "." , identifier } ;
```

- `expr` is the existing arithmetic/boolean expression (`arith` / `guard`), extended by the literals and `call`.
- `call` replaces `predicate_call`. User predicates (the `predicates` registry) are a **subset** of functions: they are boolean-returning and declared in the graph. Standard-library functions are built in.
- `local_ref` (`item`) becomes legal in **element-scoped argument positions** of `map`, `filter`, `find`, `count`, `any`, `all`, `none`, and `sort.key`, in addition to the existing `decide.test.predicate` and `foreach.*` fields.
- Deep equality: `==`/`!=` are defined on lists and objects. Membership `in` already exists in the grammar.

**Reserved names.** Standard-library function names are reserved. A graph MUST NOT declare a predicate with a reserved name (VR-26). This keeps resolution unambiguous and inspectable.

### 3.3 Core function library — normative minimum *(new §5.5)*

Signatures use `T`/`U` for type parameters. Every function is **pure**, deterministic, and total (defined on `null`/empty inputs as noted). This table is the **conformance minimum**: every implementation MUST support all of it. Additional operations are permitted as declared extensions (this §3.7).

| Function | Signature | Semantics |
|---|---|---|
| `len` | `len(x) -> integer` | Size of a list, string, or object. `null` → `0`. |
| `count` | `count(list<T>) -> integer` | Number of **non-null** elements. |
| `count` | `count(list<T>, predicate) -> integer` | Number of elements for which the element-scoped predicate is true. |
| `sum` | `sum(list<number>) -> number` | Sum; empty list → `0`; `null` elements are skipped. |
| `avg` | `avg(list<number>) -> number` | Arithmetic mean of non-null elements; empty → `null`. |
| `min` / `max` | `min(list<T>) -> T \| null` | Natural-order minimum/maximum; empty → `null`. |
| `any` / `all` / `none` | `any(list<T>, predicate) -> boolean` | Existential / universal / empty-or-none-satisfy. |
| `map` | `map(list<T>, selector) -> list<U>` | Element-wise projection (`item` bound). |
| `filter` | `filter(list<T>, predicate) -> list<T>` | Keep elements satisfying the predicate, preserving order. |
| `find` | `find(list<T>, predicate) -> T \| null` | First matching element, or `null`. |
| `append` / `prepend` | `append(list<T>, T) -> list<T>` | New list with one element added at the end/start. |
| `concat` | `concat(list<T>, list<T>, ...) -> list<T>` | Concatenate lists; also `concat(string, ...) -> string`. |
| `flatten` | `flatten(list<list<T>>) -> list<T>` | One level of flattening. |
| `distinct` | `distinct(list<T>) -> list<T>` | Remove duplicates, keeping first-occurrence order. |
| `slice` | `slice(list<T>, start, end?) -> list<T>` | Half-open sub-list; `end` defaults to length. |
| `sort` | `sort(list<T>, key?, descending?) -> list<T>` | Stable total order; `key` is element-scoped. |
| `reverse` | `reverse(list<T>) -> list<T>` | Reverse order. |
| `merge` | `merge(object, object, ...) -> object` | Shallow merge; later keys win. |
| `keys` / `values` | `keys(object) -> list<string>` | Object keys / values in insertion order. |
| `get` | `get(object, key, default?) -> any` | Value for key, or `default` (`null` if omitted). |
| `put` | `put(object, key, value) -> object` | Immutable insert/overwrite; returns a new object. |
| `has` | `has(object, key) -> boolean` | Key presence. |
| `join` | `join(list<string>, sep) -> string` | Join with separator; `null` elements become empty strings. |
| `if` | `if(boolean, T, T) -> T` | Conditional selection (both branches type-compatible). |
| `coalesce` | `coalesce(T?, ...) -> T?` | First non-`null` argument. |
| `lookup` | `lookup(T, object, default?) -> U` | Map a value through a literal/object table; `default` (`null`) if absent. |
| `to_number` / `to_integer` / `to_string` / `to_boolean` | conversion | Explicit coercions; invalid conversion → `null`. |
| `abs` / `round` | `abs(number) -> number`, `round(number, digits?) -> number` | Numeric helpers. |
| `date_diff` | `date_diff(date, date, unit) -> integer` | Whole units between dates; `unit ∈ {days, weeks, months, years}`. |

**Beyond the minimum.** Operations not listed above — `split`, `trim`, `upper`, `lower`, `contains`, `replace`, `format`, `group_by`, `zip`, `reduce`, `median`, … — are **not** part of the core tier. Implementations expose whatever their host language provides as declared extensions (this §3.7). Nothing in the core list is a claim that these are unimportant; the claim is that they need not be identical everywhere to make a graph correct.

### 3.4 Semantics and typing

- **Type checking.** Every call MUST match a signature; arguments MUST be assignable to parameter types; the return type MUST match the target slot or enclosing parameter (VR-26).
- **Element scope.** `item` binds the current element in element-scoped parameters only. Its type is derived from the first list argument. `item` MUST NOT appear in ordinary arguments, edge guards, or `set` expressions outside a scoped call.
- **`null` propagation.** Aggregates skip `null` as documented; predicates treat `null` comparisons as `false` (consistent with §5.3); `if`/`coalesce`/`lookup` provide explicit null handling.
- **Total order.** `sort` and `min`/`max` require a defined natural order for the element type (`number`, `string`, `date`, `enum` declaration order). A `sort` with a non-total order MUST fail validation.
- **Contextual typing for empty literals.** `[]` and `{}` take their type from the assignment slot or parameter. An empty literal with no contextual type MUST fail validation.
- **Numeric semantics are spec-defined, not host-defined.** `+`, `-`, `*` are exact on integers. `/` is true division returning a number and never truncates. Integer division and remainder are `div`/`mod`, truncating toward zero, with the remainder taking the sign of the dividend. `round` is half-away-from-zero. Integer arithmetic MUST be exact or fail — never silently wrap. `to_integer` truncates toward zero; `to_number` parses decimal notation only. These rules exist because host languages disagree precisely here, and scoring correctness depends on them; an implementation MAY use an arbitrary-precision or checked-integer path to satisfy them.
- **Extensions are constrained.** An implementation-provided extension (this §3.7) MUST be pure, deterministic, and total (or documented as partial with a `null` result). It MUST NOT perform I/O, call models, read time or randomness, or depend on host iteration order. `sort`/`min`/`max` over an extension type require a declared total order.

### 3.5 Examples

DASH screener score (count of endorsed probes):

```yaml
views:
  mdd_responses:
    type: list<enum>
    source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]
  mdd_score:
    type: integer
    expr: "count(views.mdd_responses, item == 'yes')"
```

Weighted score with explicit enum→number mapping (FTND-style):

```yaml
views:
  ftnd_items:
    type: list<enum>
    source: [state.ftnd_q1, state.ftnd_q2, state.ftnd_q3]
  ftnd_score:
    type: integer
    expr: "sum(map(views.ftnd_items, lookup(item, {none: 0, low: 1, mid: 2, high: 3}, 0)))"
```

Duration threshold using `date_diff`:

```yaml
nodes:
  - id: b_two_years
    type: branch
edges:
  - { from: b_two_years, to: chronic_path, type: conditional, when: "date_diff(state.onset_date, state.offset_date, 'years') >= 2", priority: 1 }
  - { from: b_two_years, to: acute_path,  type: default }
```

### 3.6 Rejected alternatives

| Alternative | Why rejected |
|---|---|
| Aggregates only as `decide` tests | `decide` is a boolean router; a score is a value that must be stored, exported, displayed, and compared. Confining it to `decide` hides the value. |
| A new `aggregate`/`reduce` node | Adds a node type for a pure function. `set` + `views` already have the right await/effect semantics. |
| Enum→number coercion in `ask` (0/1 integers) | Distorts the respondent-facing interaction and loses `Unk`/`N/A` as distinct values. |
| Ternary syntax | `if(cond, a, b)` is a function, keeps the grammar small, and composes with `map`/`lookup`. |

### 3.7 Extensibility — conformance minimum vs. implementation operations *(new §5.6)*

The library in §3.3 is a **floor, not a ceiling**. Implementations should lean on their host language's type system and native operations rather than waiting for the spec to enumerate them — but a spec language's value is that the same graph produces the same answers everywhere, so the two concerns have to be separated.

Three tiers:

| Tier | Requirement | Semantics defined by |
|---|---|---|
| **Core** (§3.3) | **MUST** — supported by every conforming implementation | Spec |
| **Extension** | **MAY** — any further pure operation the host provides | Implementation, via a declared manifest |
| **Prohibited** | **MUST NOT** — non-deterministic, effectful, or order-dependent operations | — |

**Rules for extensions**

1. **Declared, not implicit.** An implementation advertises each extension as a manifest entry: qualified name, signature in ConvoSpec's type system, and whether the result may be `null`. A function absent from the manifest does not exist as far as validation is concerned — VR-26 type-checking is unchanged in strength.
2. **Namespaced.** Extension names MUST be qualified (`py.`, `js.`, `rs.`, or a declared vendor prefix) so they cannot shadow the core library or a graph predicate.
3. **Functions only, never grammar.** Extensions add *functions*. Operators, literals, and evaluation order stay spec-owned, and an implementation MUST NOT extend the expression grammar. This is what keeps every graph parseable by every conforming tool, including ones that cannot execute it.
4. **Host types are mapped, not leaked.** Implementations SHOULD expose native collection/string/number operations directly (`sorted`, `join`, `floor`, …) with signatures mapped into ConvoSpec types. Where the host type system is richer, the implementation maps at the boundary or rejects; host-specific types MUST NOT appear in state, views, or exported values.
5. **Determinism is a precondition.** Per this §3.4: no I/O, no models, no time or randomness, no dependence on host iteration order. Operations whose host semantics are order- or locale-dependent (map iteration order, collation, casing, float formatting) MUST be normalised before exposure or not exposed at all.
6. **Capability declaration.** A graph that uses an extension declares the requirement (VR-31). A graph that declares none is portable to every conforming implementation.

**Why not "all native operations are supported."** An unbounded surface cannot be validated or documented, and host engines diverge exactly where instruments are sensitive — integer division, modulo sign, float rounding, map ordering, collation. Declared, typed extensions give implementers the reach they want while keeping the reproducible core intact. Portability then becomes a checkable property of the (graph, manifest) pair rather than an assumption.

**Illustrative extensions** (implementation-defined, not required): `split` `trim` `upper` `lower` `contains` `starts_with` `ends_with` `replace` `format` `group_by` `zip` `reduce` `median` `stdev`. A future revision MAY promote any of these into the core tier once two or more implementations need it with identical semantics.

---

## 4. G3 — State management: derived views and general state transformation

### 4.1 Motivation

G3 is two related needs:

1. **Assembly.** The instrument repeatedly needs "the responses of this section" as one value — to count endorsed items, to score, to gate a later node, to build a summary string ("concat these 10 inputs, label it X").
2. **Repetition counts.** Count-driven blocks (attempts ≤6, fears ≤3, events =5) need a count or a range, and today neither a range constructor nor a list accumulator exists.

v0.3.0's `set` can only assign scalars from scalar expressions, and there is no way to name a derived collection. The fix is a **declared derived view** plus a **general `set`**.

### 4.2 New graph block: `views` *(new §3.8 / §6.1)*

A view is a named, **read-only**, deterministically recomputed value derived from state, context, or other views. It is not state: it cannot be written, and it is not part of `state_schema`.

```yaml
views:
  mdd_responses:
    type: list<enum>
    source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]

  mdd_score:
    type: integer
    expr: "count(views.mdd_responses, item == 'yes')"

  mdd_flagged:
    type: boolean
    expr: "views.mdd_score >= 3"

  mdd_summary:
    type: string
    expr: "join(map(views.mdd_responses, to_string(item)), ', ')"

  mdd_answered:
    type: integer
    expr: "count(views.mdd_responses)"
```

| Field | Required | Description |
|---|---|---|
| `type` | yes | The view's value type (`list<T>`, `object`, or a scalar type). |
| `source` | conditional | A list of references (`state.*` / `context.*` / `views.*`); sugar for a list literal. REQUIRED if `expr` is absent. |
| `expr` | conditional | An expression evaluating to `type`. REQUIRED if `source` is absent. |

Exactly one of `source` or `expr` MUST be present.

**Rules**

- Views are **read-only**. No `set`, `ask`, `fetch`, or `tool` may write a view. A view name MUST NOT collide with a `state_schema` variable or a `context` path.
- A view is recomputed deterministically from current state whenever read. The spec defines the *observable* semantics (a pure function of current state); caching/invalidation is a runtime detail.
- The view dependency graph MUST be acyclic. `source`/`expr` references MUST resolve. Cycles fail validation (VR-29).
- Views MUST NOT perform I/O, call models, or depend on time/randomness. `now()` is deliberately not a function; time comes from `context` or state.
- A node that reads a view MUST declare the view in `effects.read`. The validator expands view dependencies to underlying state for VR-7.
- Views are referenced as `views.<name>` wherever `state.*` is legal: expressions, edge guards, `loop.exit_when`, `decide.operands` (`refs` or `list_ref`), `foreach.over`, `ask.options_ref`, and templates.

**Why a separate namespace, not `state`.** State is mutable and written; views are derived and read-only. Keeping them in distinct namespaces makes VR-7-style validation and replay reasoning simpler, and prevents accidental writes.

**Not a reporting feature.** Views exist so that state can be *projected* — assembled, aggregated, and read as one value. That a user interface, an export, or a summary message can consume them follows from views being ordinary readable values; it is not a design goal. Nothing here defines a dashboard, a layout, or a presentation format.

**Heterogeneous sections.** When a section's items have different types, build a list of objects via `expr`:

```yaml
views:
  intake_section:
    type: list<object>
    expr: "[{key: 'name', value: state.q_name}, {key: 'reason', value: state.q_reason}, {key: 'goal', value: state.q_goal}]"
```

### 4.3 `set` as a general state transformer (§4.5)

Widen `set` so it is the single, general way to assemble state from state:

- `assignments[].expression` may evaluate to any declared type, including `list<T>` and `object`.
- All assignments in one `set` node are evaluated **simultaneously** against the state at node entry, then applied atomically. No assignment may observe another assignment's write in the same node (VR-28). This removes order dependence and makes replay exact.
- Self-reference is legal and reads the pre-node value: `append(state.acc, state.current_item)` appends to the old list.
- Values are immutable; `append`/`concat`/`put`/`merge` return new values.
- `set` remains `await: none` and pure with respect to the external world (no I/O, no model calls).

Example — assemble a labelled summary from several inputs:

```yaml
- id: build_intake_summary
  type: set
  config:
    assignments:
      - slot: intake_summary
        expression: "join([state.q_name, state.q_reason, state.q_goal], ' | ')"
      - slot: intake_complete
        expression: "state.q_name != null and state.q_reason != null"
```

### 4.4 Deterministic accumulation in `foreach` *(new §6.5.3)*

A dynamic collection (variable-length section, attempt log) is built by initialising a list slot and appending inside a `foreach` body:

```yaml
- id: init_attempt_log
  type: set
  config: { assignments: [ { slot: attempt_log, expression: "[]" } ] }

- id: attempts_loop
  type: foreach
  config:
    over: { list_ref: state.attempt_slots }
    body_entry: record_attempt
    item_slot: current_attempt
    max_iterations: 6

- id: record_attempt
  type: set
  config:
    assignments:
      - slot: attempt_log
        expression: "append(state.attempt_log, state.current_attempt)"
```

- `foreach` snapshots only the **iteration set** (§4.4), not the accumulator. Execution is single-threaded and ordered, so appends are deterministic.
- Each iteration is a separate `set` execution; it reads the previous iteration's write. This is the intended accumulation semantics and is replay-safe because every write is recorded.
- The same pattern works with a bounded `loop` + `counter_slot` when the iteration set is a count rather than a list.

### 4.5 Section-collection pattern (canonical)

The DASH "responses of a section" pattern becomes:

```yaml
views:
  mdd_responses:
    type: list<enum>
    source: [state.mdd_q1, state.mdd_q2, state.mdd_q3]
  mdd_score:
    type: integer
    expr: "count(views.mdd_responses, item == 'yes')"
  mdd_complete:
    type: boolean
    expr: "count(views.mdd_responses) == len(views.mdd_responses)"
```

- `source` is the explicit sync point: the view names exactly which slots it draws from.
- `mdd_score` is the G1/G3 aggregate.
- `mdd_complete` is partial-reporting/completion tracking.
- Any consumer that needs the section's state — a later node, a summary message, an external reader — reads `views.mdd_score` / `views.mdd_complete` as ordinary values. Display is one consumer among several, not a separate layer.

**Optional shorthand (open question this §11.4).** A section tag could let `source` be `{ section: mdd }`, collecting every `ask` node tagged `collect_as: mdd` in declaration order. It is still statically resolvable, but it couples `ask` nodes to views; the explicit `source` list is the default.

### 4.6 Rejected alternatives

| Alternative | Why rejected |
|---|---|
| `ask` nodes auto-append to a hidden collection | Violates P1 (declarative) and P4 (inspectable); ordering and missingness become implicit. |
| Mutable, in-place list append | Aliasing and order dependence make replay and static analysis harder. Immutable values + atomic `set` are simpler. |
| A new `collect` node | `set` + `views` already express it; a node adds taxonomy for no new capability. |
| Nested `foreach` for G2 | Deliberately declined (this §5). |
| Storing views in `state_schema` | Views are derived, not mutable; a separate namespace keeps validation and replay clean. |
| A dashboard/reporting layer | Views are a state-projection mechanism; display consumes them as ordinary reads (D5). |

---

## 5. G2 — Flat iteration, no patch

Per review, logic stays flat. The DASH criterion × drug grid (P12) is **not** given a nested-iteration feature.

**Authoring pattern (documented in §6.5, not enforced):**

```yaml
- id: per_drug
  type: foreach
  config:
    over: { list_ref: state.drugs_endorsed }
    body_graph: { registry: drug_modules, key: "item" }
    item_slot: current_drug
    max_iterations: 10
# inside dash_drug_module: a second foreach over the fixed criterion list, or unrolled criteria
```

- Outer `foreach` over drugs; inner `call`/`foreach` over the fixed criteria list.
- The criteria list is fixed per instrument, so unrolling is acceptable when a `call` is inconvenient.
- §11 *Nested iteration* remains open. No grammar, node, or validation change is proposed.

Rationale: nested iteration is the one place where a flat edge model starts to resemble a query engine. Keeping the authoring pattern explicit preserves the inspectable control-flow graph, and DASH does not need it as a language primitive.

---

## 6. Consolidated spec change list

### 6.1 Sections to amend

| Spec section | Change |
|---|---|
| §3.2 State | Note that `views` are derived, not state; reference `views.*` in the reference list. |
| §3.3 Turn | No change. |
| §3.8 (new) | Derived views: definition, fields, rules, namespace. |
| §4.5 `set` | Widen to any type; atomic simultaneous assignment; accumulator pattern. |
| §5.3 Grammar | Add `list_literal`, `object_literal`, `call`; extend `local_ref` legality to element-scoped arguments; define list/object equality. |
| §5.5 (new) | Core function library and typing rules — the conformance minimum. |
| §5.6 (new) | Extensibility: three tiers, extension manifest, namespacing, capability declaration. |
| §6.1 Graph anatomy | Add `views` (and `capabilities`, if adopted — this §11.2). |
| §6.5.3 (new) | Section collection + deterministic accumulation patterns. |
| §8 Validation | Add VR-26 … VR-31; amend VR-7/8/20/23. |
| §10 Examples | Add screener-score view, weighted score, section summary, accumulator. |
| §11 Open Issues | Update per this §11. |

### 6.2 Proposed validation rules (numbering provisional)

- **VR-26** — Every call MUST match a declared signature — core (§5.5) or, for a namespaced extension, the implementation manifest — argument types MUST be assignable to parameter types, and the return type MUST match the target slot or enclosing parameter. Core and extension names are reserved and MUST NOT be redeclared as predicates.
- **VR-27** — `local_ref` (`item`) MUST appear only in element-scoped positions (`decide.test.predicate`, `foreach.filter`/`order_by`/`tiebreak`/`body_graph`, and element-scoped arguments of `map`/`filter`/`find`/`count`/`any`/`all`/`none`/`sort`). It MUST type-check against the derived element type.
- **VR-28** — `set` assignments MUST be evaluated simultaneously against pre-node state and applied atomically; each expression MUST type-check against its slot; empty literals MUST be contextually typed; no assignment may depend on another assignment in the same node.
- **VR-29** — Every `views` entry MUST declare `type` and exactly one of `source`/`expr`; all references MUST resolve; `expr` MUST type-check to `type`; the view dependency graph MUST be acyclic; views MUST NOT be written by any node.
- **VR-30** — Every `views.*` reference in a template or guard MUST resolve to a declared view; a node reading a view MUST declare it in `effects.read`; view dependencies MUST resolve transitively to `state_schema`/`context_schema`.
- **VR-31** — A graph that calls an implementation-provided extension MUST declare the required capability. Validation MUST fail if the target implementation's manifest does not advertise a compatible name and signature. Extension names MUST be namespaced and MUST NOT shadow core functions or graph predicates.

### 6.3 Existing rules affected

| Rule | Effect |
|---|---|
| VR-7 | State refs resolve in `state_schema`; view refs resolve in `views`; `item` only in scoped positions. |
| VR-8 | `ask.options_ref` MAY reference a view whose element type matches the input type. |
| VR-20 | `decide.operands` MAY reference `views.*`; `list_ref` MAY be a view. |
| VR-23 | `foreach.over.list_ref` MAY be a view; element type is the view's element type. |
| §5.3 null rule | Unchanged; aggregates define their own null-skipping explicitly. |

### 6.4 Grammar diff (summary)

```diff
- primary      = comparison | predicate_call | "(" , guard , ")" ;
- predicate_call = identifier , "(" , [ operand , { "," , operand } ] , ")" ;
+ primary      = comparison | call | list_literal | object_literal | "(" , guard , ")" ;
+ call         = identifier , "(" , [ expr , { "," , expr } ] , ")" ;
+ list_literal = "[" , [ expr , { "," , expr } ] , "]" ;
+ object_literal = "{" , [ obj_entry , { "," , obj_entry } ] , "}" ;
+ obj_entry    = ( identifier | string ) , ":" , expr ;
```

`local_ref` production is unchanged; its **legal positions** expand (VR-27).

---

## 7. Determinism and inspectability impact

| Concern | Assessment |
|---|---|
| P1 Declarative | Views name their sources; functions are a fixed library; no runtime discovery. |
| P3 Deterministic control | All functions are pure and total; `sort` requires a total order; `set` is atomic; accumulation is ordered and single-threaded. |
| P4 Inspectable | Function calls type-check statically; view dependencies are enumerable and acyclic; `effects.read` covers views. |
| P5 Single-await | Untouched; no new awaiting node. |
| P6 Explicit failure | Invalid calls/views fail validation; `null` conversions return `null`; runtime type errors route to `error`. |
| Portability | The core library is identical everywhere with spec-defined semantics; extensions are declared, typed, namespaced, and pure, so a graph's requirements are enumerable from its source (this §3.7). |
| §3.4 / §7.5 Replay | Atomic `set` and ordered accumulation reproduce exactly; evaluated view values used in decisions are recorded. |

One residual requirement: because views are recomputed, the runtime MUST record the **evaluated view values** that fed any decision or user-facing message, so replay does not depend on re-deriving them from a possibly different state snapshot.

---

## 8. Traceability (DASH)

| Pattern | Before | After | Basis |
|---|---|---|---|
| P04 Derived screener scoring | Partial | **Covered** | `views` + `count`/`sum`/`lookup` (this §3–§4) |
| P09 Count-driven repetition | Covered | Covered | `len`/`count` + accumulator/`loop` (this §3.3, §4.4) |
| P12 2-D criteria × substance grid | Partial | Partial | Unchanged; flat outer `foreach` + inner `call` (this §5) |
| P18 Embedded FTND | Partial | **Covered** | `views` + `sum(map(..., lookup(...)))` (this §3.5) |
| All other patterns | Covered | Covered | No regression; additive changes |

DASH totals after the proposal: **Covered 23 · Partial 1 · Gap 0.** The one remaining partial (P12) is explicitly deferred by review, not blocked by capability.

---

## 9. Backwards compatibility and migration

- **Additive.** No v0.3.0 construct changes meaning. `predicate_call` becomes a special case of `call`; existing graphs remain valid.
- **Reserved names.** Graphs that declared a predicate named `sum`, `count`, `len`, `map`, etc. would now fail VR-26. This is a deliberate, documented breaking change; a migration note is warranted. (No such predicate exists in the current examples.)
- **`set` widening.** Scalar assignments behave exactly as before. Atomic simultaneous evaluation matches the intuitive reading of a multi-assignment `set`; any graph that relied on sequential order within one `set` was relying on undefined behavior.
- **Templates.** `{{ state.* }}` and `{{ context.* }}` are unchanged; `{{ views.* }}` is additive.
- **Extensions.** No effect on v0.3.0 graphs. A graph that declares no capabilities is portable to every conforming implementation; a graph that uses one is portable to any implementation advertising it.

---

## 10. Implementation plan (proposed order)

1. **Grammar + core library** (§5.3/§5.5): literals, `call`, element-scoped arguments, function signatures, numeric semantics, VR-26/27.
2. **Extensibility** (§5.6): three tiers, extension manifest, namespacing, capability declaration, VR-31.
3. **`set` transformer** (§4.5): any-type assignments, atomic semantics, accumulator, VR-28.
4. **`views`** (§3.8/§6.1): block, namespace, dependency/cycle rules, VR-29/30; amend VR-7/8/20/23.
5. **Canonical patterns** (§6.5.3) and **examples** (§10).
6. **§11** updates and traceability.

Steps 1–3 are independent of step 4; step 4 depends on 1 (views use the library).

---

## 11. Decisions and remaining questions

**Settled in review, 2026-11-16:**

| # | Decision |
|---|---|
| 1 | Namespace is `views:` + `views.<name>`. |
| 2 | Keep `avg`/`min`/`max`; `lookup` is the enum→number mechanism (no `case`/`match`). |
| 3 | Bare `item`, consistent with `foreach`; `index` binds alongside it. |
| 4 | Keep the split: `len` = size, `count(list)` = non-null, `count(list, pred)` = matching. |
| 5 | `set` assignments evaluate simultaneously against pre-node state and apply atomically. |
| 6 | A `set` inside `foreach` may read and append its own list slot. |
| 7 | Explicit `source: [state.q1, …]` only; no `collect_as` shorthand. |
| 8 | Templates restricted to `{{ views.<name> }}`. |
| 9 | Views are session-local read-only projections; no export. |
| 10 | No further core functions. Everything else lives in the extension tier (this §3.7). |
| 11 | Target 0.4.0, applied in the §10 order. |
| — | Views are not a reporting/dashboard feature; display is one incidental consumer (D5). |

**Remaining questions:**

1. **Extension manifest location.** An out-of-band published profile per implementation, or a machine-readable file the validator loads? The former is simpler to specify; the latter is checkable in CI.
2. **Capability declaration shape.** A graph-level `capabilities: [py.sorted, py.split]`, or inferred from use? Explicit is more inspectable; inferred is less authoring noise.
3. **Promotion bar.** Should §5.6 state a concrete rule for moving an extension into the core tier (e.g. "two independent implementations, identical semantics, used by ≥2 instruments"), or leave it to future revisions?
4. **Numeric edge semantics.** Confirm: true division for `/`; `div`/`mod` truncate toward zero with dividend-signed remainder; `round` half-away-from-zero; no silent integer wraparound. Host languages disagree on each of these, so they need to be explicit.
5. **`index` binding.** Confirm `index` is the position in the *source* list — i.e. pre-`filter` for a `filter` predicate.

---

## 12. Explicitly not proposed

| Item | Reason |
|---|---|
| Nested `foreach` / 2-D iteration (G2) | Declined by review; flat pattern documented (this §5). |
| Auto-collection by `ask` (hidden writes) | Violates P1/P4; explicit `views.source` is the default (this §4.6). |
| Mutable in-place list mutation | Aliasing/replay hazard; immutable values + atomic `set` (this §4.3). |
| A new `aggregate`/`collect` node | Functions + `views` + `set` cover it; no node needed (this §3.6, §4.6). |
| `now()` or other non-deterministic functions | Breaks replay; time comes from `context`/state. |
| Query-engine features (`group_by`, joins as first-class) | Out of scope; `group_by` is an illustrative extension, not a core function (this §3.7). |
| Implementation-defined semantics for core operations | Core ops are spec-defined so scoring is reproducible (this §3.4). Only extensions carry host semantics. |
| Extending the expression grammar via extensions | Extensions are functions only; grammar stays spec-owned so every tool parses every graph (this §3.7). |
| A dashboard/reporting layer | Views are a state-projection mechanism; display consumes them as ordinary reads (D5). |

---

## 13. Integration notes

Applied in `spec.md` v0.4.0, following the order in §10 (grammar + library → extensibility → `set` → `views` → patterns/examples → §11). Sections 3–12 landed as proposed, with the deviations below.

### 13.1 Deviations and additions

| # | Change | Reason |
|---|---|---|
| 1 | **New principle P10** (§2) | Not in the proposal. The three-tier model needs a stated principle; without it, "portable core, declared extensions" is an orphan rule in §5.6. |
| 2 | **New §6.5.4 and VR-32** — nested `foreach` is forbidden | §5 said "documented pattern, no validation change". Documenting flat iteration without a rule leaves the flatness unenforced. VR-32 makes the constraint checkable while still allowing the `call`-boundary pattern. |
| 3 | **`capabilities` adopted as an explicit graph field** (§6.1) | §11.2 left the shape open. Explicit was chosen over inferred: it is inspectable from the graph source and consistent with the D3/D4 bias. Inference remains open (§11). |
| 4 | **`call` grammar uses a `name` production** (§5.3) | The proposed `call` used `identifier`, which cannot express `py.split`. `name` allows qualified names and explicitly forbids the reserved roots `state`/`context`/`views`. |
| 5 | **`index` added to the `local_ref` production** (§5.3) | Decision 3 bound `index`; the proposed grammar only showed `item`. |
| 6 | **VR-30 scoped to node config** | `branch` declares `Effects: none`, and its guards live on edges. Requiring `effects.read` for every view reference would have contradicted the node contract. Config reads require a declaration; guard and `loop.exit_when` references resolve only — the same treatment VR-7 already gives `state.*`. |
| 7 | **Core numeric semantics made explicit** (§5.5) | The proposal said "true division". `div`/`mod` truncating toward zero with dividend-signed remainder, `round` half-away-from-zero, and no silent integer wraparound are now stated, because host languages disagree and scores depend on them. |
| 8 | **`ask.options_ref`, `decide.operands`, `foreach.over` node text amended** (§4.2, §4.4) | VR-8/20/23 were amended for views, but the node-level field descriptions still said "state variable". Left unamended, the two halves of the spec would disagree. |
| 9 | **`set` gains a SHOULD-level state/derived distinction** (§3.2) | Prevents views from being re-implemented as mutable caches, while leaving point-in-time snapshots legitimately state. |
| 10 | **Example 10.8 added with an explicit `effects.read`** (§10.8) | Demonstrates VR-30 in a complete graph; the earlier examples predate `effects` declarations and do not show it. |

### 13.2 Decisions and open questions

All eleven review decisions are settled and implemented. The five questions raised in §11 remain open; none blocked integration. Each is carried into `spec.md` §11.

| §11 question | `spec.md` §11 entry |
|---|---|
| 1 Extension manifest location | Expression extensions |
| 2 Capability declaration shape | Expression extensions |
| 3 Core-tier promotion bar | Core-tier promotion |
| 4 Numeric edge semantics | Numeric edge semantics |
| 5 `index` binding scope | `index` binding scope |
| — View export (deferred by decision 9) | View export |

### 13.3 Verification performed

- All 24 YAML blocks in `spec.md` parse (PyYAML).
- VR-1…VR-32 are defined exactly once each, in order, with no duplicates or gaps.
- Every `§`/`VR` cross-reference introduced by this proposal resolves in the revised spec.
- No tabs or trailing whitespace in `spec.md` or this document.
- Traceability in §8 above matches [§8 of the compliance evaluation](./dash-compliance.md#8-post-integration-re-evaluation-v040).
