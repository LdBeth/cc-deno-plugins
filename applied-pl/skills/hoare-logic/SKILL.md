---
name: hoare-logic
description: Use Hoare logic to verify, derive, and reason about imperative programs. Apply this skill when asked to prove correctness, find loop invariants, verify pre/postconditions, calculate weakest preconditions, or reason about imperative code with assignments, loops, and conditionals. Also use when the user mentions "Hoare triple", "wp", "program verification".
type: skill
---

# Hoare Logic for Program Verification

`{P} C {Q}`: if P holds before C, then Q holds after. Derive correctness with
named inference rules — never by inspection.

## Inference Rules

**Assignment Axiom** (backward — substitute in postcondition):

```
──────────────────────────
  {Q[x/E]}  x := E  {Q}
```

`wp(x := E, Q) = Q[x/E]`

**Sequencing**: `wp(C₁; C₂, Q) = wp(C₁, wp(C₂, Q))`

**Conditional**:

```
  {P ∧ B} C₁ {Q}    {P ∧ ¬B} C₂ {Q}
───────────────────────────────────────
     {P}  if B then C₁ else C₂  {Q}
```

**While (partial correctness)**:

```
       {I ∧ B} C {I}
──────────────────────────────
  {I}  while B do C  {I ∧ ¬B}
```

**Consequence**:

```
  P' → P    {P} C {Q}    Q → Q'
──────────────────────────────────
         {P'} C {Q'}
```

## Weakest Precondition Table

| Command                | wp(C, Q)                           |
| ---------------------- | ---------------------------------- |
| `skip`                 | Q                                  |
| `x := E`               | Q[x/E]                             |
| `C₁; C₂`               | wp(C₁, wp(C₂, Q))                  |
| `if B then C₁ else C₂` | (B → wp(C₁, Q)) ∧ (¬B → wp(C₂, Q)) |
| `while B do C`         | requires invariant                 |

## Finding Loop Invariants

Three obligations must hold for invariant I, guard B, loop body C:

1. **Init**: P → I
2. **Maintenance**: {I ∧ B} C {I}
3. **Finalization**: (I ∧ ¬B) → Q

**Strategies:**

- **Relax the postcondition**: replace the "done" index with a progress
  variable.
  - Q: `s = Σ a[0..n)` → Invariant: `s = Σ a[0..i) ∧ 0 ≤ i ≤ n`
- **Conjunctive strengthening**: if maintenance fails, the failing obligation
  tells you what to conjoin.

## Total Correctness

Requires a **variant** t (integer, bounded below by 0 when guard holds, strictly
decreasing each iteration):

```
  {I ∧ B ∧ t = V} C {I ∧ t < V}    I ∧ B → t ≥ 0
──────────────────────────────────────────────────────
         [I]  while B do C  [I ∧ ¬B]
```

Square brackets `[P] C [Q]` denote total correctness.

## Array Assignments

```
wp(a[i] := E, Q) = Q[a / a⟨i ↦ E⟩]
```

where `a⟨i ↦ E⟩[j] = if j = i then E else a[j]`.

Common invariant patterns:

- **Prefix processed**: `∀ k. 0 ≤ k < i → a[k] = f(k)`
- **Partition**: `(∀ k < p → a[k] < pivot) ∧ (∀ p ≤ k < i → a[k] ≥ pivot)`
- **Sortedness**: `∀ k. 0 ≤ k < i-1 → a[k] ≤ a[k+1]`

## Worked Example: Linear Search

```python
i = 0
while i < n and a[i] != v:
    i = i + 1
return i
```

**Triple**: `{n ≥ 0} C {(i < n ∧ a[i] = v) ∨ (i = n ∧ v ∉ a[0..n))}`
**Invariant**: `I ≡ 0 ≤ i ≤ n ∧ v ∉ a[0..i)`

1. **Init**: `n ≥ 0 → (0 ≤ 0 ≤ n ∧ v ∉ a[0..0))` — empty range. ✓
2. **Maintenance**: Assume `I ∧ B`. After `i := i+1`: bounds hold from `i < n`;
   `v ∉ a[0..i+1)` from `v ∉ a[0..i)` and `a[i] ≠ v`. ✓
3. **Finalization**: `¬B ≡ i ≥ n ∨ a[i] = v`. With I: either
   `i = n ∧ v ∉ a[0..n)` or `i < n ∧ a[i] = v`. Both match Q. ✓
4. **Termination**: Variant `n - i`, decreases by 1; `i < n → n - i > 0`. ✓ ∎

## Common Pitfalls

- **Wrong substitution direction**: assignment axiom works backward — substitute
  in Q, not P.
- **Invariant too weak**: finalization fails → strengthen it.
- **Invariant too strong**: initialization fails → weaken it.
- **Forgetting guard in maintenance**: assumption is `I ∧ B`, not just I.
- **Aliasing**: standard Hoare logic assumes variables are independent. For
  pointer-heavy code, use separation logic or explicit anti-aliasing
  preconditions.

## Presentation Format

1. State `{P} C {Q}` formally
2. For loops: state invariant and variant
3. List and discharge each proof obligation with rule name
4. Conclude: triple holds, or identify which obligation fails and why
