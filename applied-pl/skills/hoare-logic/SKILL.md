---
name: hoare-logic
description: Use Hoare logic to verify, derive, and reason about imperative programs. Apply this skill when asked to prove correctness, find loop invariants, verify pre/postconditions, calculate weakest preconditions, or reason about imperative code with assignments, loops, and conditionals. Also use when the user mentions "Hoare triple", "wp", "program verification".
type: skill
---

# Hoare Logic for Program Verification

Hoare logic treats imperative programs as transformers on logical predicates. A
Hoare triple `{P} C {Q}` asserts: if predicate P holds before executing command
C, then predicate Q holds afterward. Instead of testing or informal argument,
you derive correctness step by step, each step justified by a named inference
rule.

## When to Use This

- Verifying that a loop computes what it claims (finding and proving invariants)
- Deriving preconditions: "what must hold for this code to be correct?"
- Proving postconditions: "what does this code guarantee?"
- Checking whether a proposed fix actually addresses a bug
- Reasoning about array bounds, index safety, and numeric overflow
- Comparing two implementations for semantic equivalence
- Establishing termination (total correctness)

## Core Method: Derive, Don't Argue

The central discipline: never assert correctness by inspection. State the
triple, apply inference rules backward (weakest precondition) or forward
(strongest postcondition), and show that each step follows from a named rule.
This prevents hand-waving and makes the reasoning auditable.

```
-- Goal: verify {x = 5} x := x + 1 {x = 6}

  wp(x := x + 1, x = 6)
= { assignment axiom: wp(x := E, Q) = Q[x/E] }
  (x = 6)[x / x + 1]
= { substitution }
  x + 1 = 6
= { arithmetic }
  x = 5
-- This matches the precondition, so the triple holds. ∎
```

## Inference Rules

### Assignment Axiom

The foundation of Hoare logic. Works backward: to establish Q after `x := E`,
require Q with x replaced by E beforehand.

```
───────────────────────────
  {Q[x/E]}  x := E  {Q}
```

Equivalently via weakest precondition: `wp(x := E, Q) = Q[x/E]`.

This is an _axiom_ — it needs no premise. The substitution does all the work.

### Sequencing

```
  {P} C₁ {R}    {R} C₂ {Q}
─────────────────────────────
      {P}  C₁; C₂  {Q}
```

The intermediate assertion R (the "midcondition") connects the two halves. In
weakest-precondition style: `wp(C₁; C₂, Q) = wp(C₁, wp(C₂, Q))`.

### Conditional

```
  {P ∧ B} C₁ {Q}    {P ∧ ¬B} C₂ {Q}
───────────────────────────────────────
     {P}  if B then C₁ else C₂  {Q}
```

Prove both branches establish Q, using the branch condition as an extra fact.

### While Loop (Partial Correctness)

```
       {I ∧ B} C {I}
──────────────────────────────
  {I}  while B do C  {I ∧ ¬B}
```

I is the **loop invariant** — a predicate preserved by every iteration. After
the loop, the invariant still holds _and_ the guard is false.

Finding the right invariant is the hard part. See "Finding Loop Invariants"
below.

### Consequence (Strengthening / Weakening)

```
  P' → P    {P} C {Q}    Q → Q'
──────────────────────────────────
         {P'} C {Q'}
```

You may strengthen the precondition (assume more) or weaken the postcondition
(promise less). This rule glues derived triples to the conditions you actually
need.

### Skip

```
─────────────
  {P} skip {P}
```

The do-nothing command preserves any predicate.

## Weakest Precondition Calculus

Working backward from a desired postcondition is usually more mechanical than
working forward. The weakest precondition `wp(C, Q)` is the _least restrictive_
predicate P such that `{P} C {Q}` holds.

| Command                | wp(C, Q)                           |
| ---------------------- | ---------------------------------- |
| `skip`                 | Q                                  |
| `x := E`               | Q[x/E]                             |
| `C₁; C₂`               | wp(C₁, wp(C₂, Q))                  |
| `if B then C₁ else C₂` | (B → wp(C₁, Q)) ∧ (¬B → wp(C₂, Q)) |
| `while B do C`         | requires invariant (see below)     |

For everything except loops, wp is purely mechanical — substitute and simplify.

## Finding Loop Invariants

The invariant must satisfy three obligations:

1. **Initialization**: P → I (the invariant holds on entry)
2. **Maintenance**: {I ∧ B} C {I} (each iteration preserves it)
3. **Finalization**: (I ∧ ¬B) → Q (when the loop exits, the goal holds)

### Strategies

**From the postcondition**: Take Q and relax it by replacing the "done" part
with a variable tracking progress. For summing an array:

- Q: `s = Σ a[0..n)`
- Invariant: `s = Σ a[0..i) ∧ 0 ≤ i ≤ n`
- Guard: `i < n`
- On exit: `i = n`, so `s = Σ a[0..n)` = Q. ∎

**Constant + progress**: Identify what the loop preserves (the constant part)
and what changes (the progress part). The invariant captures both.

**Conjunctive strengthening**: Start with a candidate, attempt the maintenance
proof. If it fails, the failing proof obligation tells you what to conjoin.

## Total Correctness

Partial correctness says "if the program terminates, Q holds." Total correctness
says "the program terminates _and_ Q holds." For total correctness of a while
loop, you additionally need a **variant** (also called a "measure" or "ranking
function"):

```
  {I ∧ B ∧ t = V} C {I ∧ t < V}    I ∧ B → t ≥ 0
──────────────────────────────────────────────────────
         [I]  while B do C  [I ∧ ¬B]
```

- t is an integer-valued expression (the variant)
- Each iteration strictly decreases t
- t is bounded below (≥ 0) when the guard holds

This guarantees the loop terminates.

Square brackets `[P] C [Q]` denote total correctness (as opposed to curly braces
for partial correctness).

## Array and Indexed Reasoning

For programs that manipulate arrays, extend substitution to handle indexed
updates:

```
wp(a[i] := E, Q) = Q[a / a⟨i ↦ E⟩]
```

where `a⟨i ↦ E⟩` is the array identical to a except at index i, where it has
value E. In predicate form:

```
a⟨i ↦ E⟩[j] = if j = i then E else a[j]
```

Common invariant patterns for array loops:

- **Initialization**: `∀ k. 0 ≤ k < i → a[k] = f(k)` (processed portion)
- **Partition**:
  `(∀ k. 0 ≤ k < p → a[k] < pivot) ∧ (∀ k. p ≤ k < i → a[k] ≥ pivot)`
- **Sortedness**: `∀ k. 0 ≤ k < i-1 → a[k] ≤ a[k+1]`

## Procedure

### Step 1: State the Triple

Write down `{P} C {Q}` explicitly. If the user gives informal requirements
("this function should return the maximum"), formalize P and Q as logical
predicates. Be precise about variable scoping and initial values — use
subscripts or ghost variables for "old" values (e.g., `x₀` for x's initial
value).

### Step 2: Decompose Structurally

Break C into its constituent statements and apply inference rules top-down
(forward) or bottom-up (backward via wp). For straight-line code, wp is
mechanical. For loops, identify candidate invariants.

### Step 3: Discharge Proof Obligations

Each application of the consequence rule produces an implication to verify. Each
loop produces three obligations (init, maintenance, finalization) plus
termination if total correctness is needed. Discharge each one explicitly — this
is where arithmetic, set theory, or domain-specific reasoning appears.

### Step 4: Conclude

If all obligations discharge, the triple holds. State the result. If an
obligation fails, the failure pinpoints the bug — report which obligation fails
and why.

## Worked Example: Linear Search

```python
# Find the index of value v in array a[0..n), or return n if absent.
i = 0
while i < n and a[i] != v:
    i = i + 1
return i
```

**Triple**: `{n ≥ 0} C {(i < n ∧ a[i] = v) ∨ (i = n ∧ v ∉ a[0..n))}`

**Invariant**: `I ≡ 0 ≤ i ≤ n ∧ v ∉ a[0..i)`

**Guard**: `B ≡ i < n ∧ a[i] ≠ v`

Proof obligations:

1. **Init**: `n ≥ 0 → (0 ≤ 0 ≤ n ∧ v ∉ a[0..0))`. Holds: empty range. ✓
2. **Maintenance**: Assume `I ∧ B`, i.e.,
   `0 ≤ i ≤ n ∧ v ∉ a[0..i) ∧ i < n ∧ a[i] ≠ v`. After `i := i + 1`:
   - `0 ≤ i+1 ≤ n`: from `0 ≤ i` and `i < n`. ✓
   - `v ∉ a[0..i+1)`: `v ∉ a[0..i)` and `a[i] ≠ v`. ✓
3. **Finalization**: `I ∧ ¬B → Q`. `¬B ≡ i ≥ n ∨ a[i] = v`. With `I`: either
   `i = n ∧ v ∉ a[0..n)`, or `i < n ∧ a[i] = v`. Both disjuncts match Q. ✓
4. **Termination**: Variant `t = n - i`. Decreases by 1 each iteration.
   `I ∧ B → t ≥ 0`: `i < n → n - i > 0`. ✓

The program is totally correct. ∎

## Translating to Practical Verification

| Hoare Logic Concept | Practical Equivalent                              |
| ------------------- | ------------------------------------------------- |
| Precondition {P}    | `assert` / argument validation at function entry  |
| Postcondition {Q}   | `assert` at function exit / return type contract  |
| Loop invariant I    | `assert` at loop head (or comment documenting it) |
| Variant t           | Checked by adding `assert t' < t` in loop body    |
| Consequence rule    | Type narrowing, refinement types                  |
| Ghost variables     | Variables used only in assertions, not in code    |

After proving a triple on paper, you can embed key predicates as runtime
assertions for defense in depth, or use them to guide test case generation (each
proof obligation suggests a boundary condition to test).

## Common Pitfalls

- **Wrong substitution direction**: The assignment axiom works _backward_ —
  substitute in the postcondition, not the precondition. `{Q[x/E]} x:=E {Q}`,
  not `{P} x:=E {P[x/E]}`.
- **Invariant too weak**: If finalization fails, the invariant doesn't capture
  enough about what the loop computes. Strengthen it.
- **Invariant too strong**: If initialization fails, the invariant assumes
  something not yet established. Weaken it or adjust what's conjoined.
- **Forgetting the guard in maintenance**: You get `I ∧ B` as your assumption,
  not just I. The guard is crucial information — use it.
- **Aliasing**: Standard Hoare logic assumes variables are independent. If `x`
  and `y` alias the same location, the assignment axiom for `x := E` silently
  invalidates predicates on `y`. Use separation logic or explicit anti-aliasing
  preconditions for pointer-heavy code.

## Presentation

When presenting a verification to the user:

1. State the triple {P} C {Q} formally
2. For loops, state the invariant and variant
3. List and discharge each proof obligation, naming the rule used
4. State the conclusion (correct, or which obligation fails and why)

This makes the reasoning transparent, mechanically checkable, and pinpoints
exactly where a bug lives if the proof breaks down.
