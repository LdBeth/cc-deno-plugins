---
name: equational-reasoning
description: Use equational reasoning to simplify and optimize programs. Apply this skill when asked to simplify, refactor, optimize, or derive programs — especially when code involves list processing, recursive patterns, function composition, algebraic structures, try-catch chains, Promise.catch composition, or separating pure logic from effectful code (async, state, I/O). Also use when the user mentions "program calculation", "fusion", or "algebraic simplification".
type: skill
---

# Equational Reasoning for Program Simplification

Equational reasoning treats programs as mathematical expressions and transforms
them using substitution, algebraic laws, and calculation — the same way you
simplify algebraic equations. Instead of rewriting code by intuition, you derive
the simplified form step by step, each step justified by a named law.

## When to Use This

- Simplifying composed operations (nested maps, filters, folds)
- Eliminating intermediate data structures (deforestation / fusion)
- Refactoring recursive definitions into cleaner forms
- Simplifying try-catch / Promise.catch chains
- Separating pure logic from effectful shells (async, state, I/O)
- Verifying that a refactor preserves semantics
- Deriving an efficient implementation from a clear but naive specification

## Core Method: Calculate, Don't Guess

The central discipline: never jump to the answer. Show each step and name the
law that justifies it. This prevents errors and makes the reasoning auditable.

```
-- Goal: simplify `sum (map (+1) xs)`

  sum (map (+1) xs)
= { fold-map fusion: foldr f z . map g = foldr (f . g) z }
  foldr (\x acc -> (x + 1) + acc) 0 xs
= { arithmetic: (x+1)+acc = x + (acc+1) ... but simpler: }
  sum xs + length xs
= { recognizing: sum (map (+k) xs) = sum xs + k * length xs }
  sum xs + length xs
```

## Fundamental Laws

### Function Composition

```
(f . g) x  =  f (g x)                -- definition
f . id     =  f                      -- right identity
id . f     =  f                      -- left identity
(f . g) . h = f . (g . h)            -- associativity
```

### Map

```
map id          =  id                   -- identity
map f . map g   =  map (f . g)          -- fusion (eliminates intermediate list)
map f (xs ++ ys) = map f xs ++ map f ys -- distributivity
map f . concat  =  concat . map (map f) -- naturality
```

### Filter

```
filter p . filter q  =  filter (\x -> q x && p x)  -- fusion
filter p . map f     =  map f . filter (p . f)     -- when f is total
```

### Fold (foldr)

```
foldr f z []      =  z                              -- base
foldr f z (x:xs)  =  f x (foldr f z xs)             -- step
foldr f z . map g =  foldr (f . g) z                -- fold-map fusion
foldr (:) []      =  id                             -- fold-id
```

The **universal property of fold**: `h` satisfies `h [] = z` and
`h (x:xs) = f x (h xs)` if and only if `h = foldr f z`. This is the main tool
for recognizing when a recursive function is a fold and for fusing operations.

### Fold (foldl)

```
foldl f z []      =  z
foldl f z (x:xs)  =  foldl f (f z x) xs
-- When f is associative with identity z:
foldl f z xs      =  foldr (flip f) z (reverse xs)
```

### Unfold (anamorphism)

```
unfoldr f seed = case f seed of
  Nothing      -> []
  Just (a, s') -> a : unfoldr f s'
```

Dual to fold. Useful for recognizing when imperative loops that build lists can
be expressed as unfolds, then fused with consuming folds (hylomorphism).

### Scan

```
scanl f z  =  map (foldl f z) . inits   -- specification
-- Horner's rule emerges from scan fusion
```

### Monad Laws (for simplifying do-notation / bind chains)

```
return a >>= f   =  f a                 -- left identity
m >>= return     =  m                   -- right identity
(m >>= f) >>= g  =  m >>= (\x -> f x >>= g)  -- associativity
```

These justify flattening nested binds and eliminating redundant wrapping.

### Exception / Try-Catch

Try-catch is the operational encoding of `Either<Error, T>`. The isomorphism
`try { expr } catch(e) { h(e) }  ≅  either h id (toEither expr)` lets you lift
effectful code into pure Either space, reason equationally, then lower back.

```
try { throw x } catch(e) { f(e) }  =  f(x)           -- beta (throw-catch elimination)
try { expr } catch { h }           =  expr           -- when expr never throws
try { f() } catch(e) { throw e }   =  f()            -- catch-rethrow elimination
try { e } catch(_) { e }           =  e              -- idempotence
```

Nested try-catch flattening:

```
try { try { a } catch(e) { b(e) } } catch(e) { c(e) }
= try { a } catch(e) { try { b(e) } catch(e2) { c(e2) } }
-- further, when b never throws:
= try { a } catch(e) { b(e) }
```

Finally distribution (finally is not a catch — it preserves the Either
structure):

```
try { a } finally { b }
= let r = toEither(a); b; fromEither(r)
```

### Promise.catch (async exception monad)

Promise.catch obeys the monad laws directly:

```
Promise.reject(e).catch(f)          =  Promise.resolve(f(e))  -- left identity
p.catch(e => Promise.reject(e))     =  p                      -- right identity
p.catch(f).catch(g)                 =  p.catch(x => Promise.resolve(f(x)).catch(g))
                                                              -- associativity
-- when f never throws:
p.catch(f).catch(g)                 =  p.catch(f)             -- dead catch elimination
```

Ordering matters: `.then(f).catch(g)` catches errors from both `p` and `f`,
while `.catch(g).then(f)` recovers first, then applies `f` to the recovery.
These are not interchangeable — `.then`/`.catch` composition is non-commutative.

## Reasoning About Effectful Code

Equational reasoning assumes referential transparency — you can substitute
equals for equals. Side effects break this. But most effectful code has a pure
core that can be extracted, simplified, and put back. The discipline is knowing
_where_ substitution is safe and _how_ to recover it when it isn't.

### Principle: Pure Core, Effectful Shell

Separate code into a pure computation (amenable to equational reasoning) and an
effectful boundary (left untouched). Simplify the core, then reassemble.

```
-- Before: effectful code with pure logic entangled
async function process(ids: string[]) {
  const items = [];
  for (const id of ids) {
    const raw = await fetch(`/api/${id}`);    // effect
    const data = await raw.json();             // effect
    const name = data.name.toUpperCase();      // pure
    if (name.length > 3) items.push(name);     // pure (accumulation pattern)
  }
  return items;
}

-- Separation: the pure core is  filter (len > 3) . map toUpper . map name
-- Apply map fusion:             filter (len > 3) . map (toUpper . name)
-- The effectful shell (fetch) is untouched.

-- After:
async function process(ids: string[]) {
  const items = await Promise.all(ids.map(id =>
    fetch(`/api/${id}`).then(r => r.json())
  ));
  return items.flatMap(data => {
    const s = data.name.toUpperCase();
    return s.length > 3 ? [s] : [];
  });
}
```

The equational step (map fusion + filter-map fold) applied only to the pure
fragment. The fetch calls were not rearranged.

### State-Passing Transformation

Mutable state breaks substitution. Recover it by making state explicit: rewrite
`let mut s; f(&mut s); g(&mut s)` as `let s1 = f(s0); let s2 = g(s1)`. Now `f`
and `g` are pure functions of state, and equational reasoning applies.

```
-- Mutable original:
let count = 0;
let sum = 0;
for (const x of xs) { count += 1; sum += x; }
return sum / count;

-- State-passing form (state = (count, sum)):
foldl (\(c, s) x -> (c + 1, s + x)) (0, 0) xs
-- This is a standard foldl with a pair accumulator.
-- Apply: result = let (c, s) = foldl ... in s / c
-- Recognize: s = sum xs, c = length xs
-- Result: sum xs / length xs  (i.e., the mean)
```

After simplification, translate back to idiomatic mutable code if desired — the
equational derivation justifies the result even if the final code uses mutation.

### Effect Commutation

Two effects commute when swapping their order does not change the observable
result. When effects commute, you can reorder them freely during reasoning.

| Effect pair              | Commutes? | Why                                |
| ------------------------ | --------- | ---------------------------------- |
| read x; read y           | Yes       | No state change                    |
| write x; write y (x ≠ y) | Yes       | Independent targets                |
| write x; read x          | No        | Read observes the write            |
| log a; log b             | Depends   | Order matters if log order matters |
| throw e; anything        | No        | Throw aborts — nothing after runs  |
| await f(); pure g()      | Yes       | Pure computation has no effects    |

Use commutation to rearrange effectful code into a form where pure fragments
cluster together, then apply equational laws to those clusters.

### Kleisli Composition

For monadic effects (Promise chains, Result chains, Option chains), Kleisli
composition lets you reason about effectful function pipelines:

```
-- Kleisli composition: (f >=> g) x  =  f x >>= g
-- Associativity:       (f >=> g) >=> h  =  f >=> (g >=> h)
-- Identity:            return >=> f  =  f  =  f >=> return
```

This is the effectful analog of function composition. In TypeScript/Promise
terms:

```typescript
// f >=> g  becomes:
const compose = (f, g) => (x) => f(x).then(g);

// Associativity means these are equivalent:
compose(compose(f, g), h) === compose(f, compose(g, h));

// So you can flatten .then chains:
p.then(f).then(g).then(h) === p.then((x) => f(x).then(g).then(h));
```

Use Kleisli associativity to reassociate `.then` chains into forms where pure
`.then` callbacks cluster (and can be fused via ordinary composition) while
effectful ones stay in sequence.

### When NOT to Reason Equationally

Do not apply equational transformations across effect boundaries when:

- **The effect is the point**: logging order, audit trails, user-visible
  sequencing. Reordering changes the program's meaning.
- **Effects interact**: write-then-read dependencies, lock acquisition order,
  resource cleanup sequences. Commutation analysis says no.
- **Concurrency introduces observation**: a pure-looking expression may be
  observed mid-evaluation by another thread. In shared-mutable-state
  concurrency, almost nothing commutes safely.

In these cases, leave the effectful structure alone and only simplify pure
subexpressions within each effectful step.

## Procedure

### Step 1: State the Equation

Write the expression to simplify as a clear equation. Name the input and output.
If working with imperative code, first transcribe the relevant fragment into a
functional/declarative style so equational laws apply. This is not about
changing the final language — it's a working notation.

### Step 2: Unfold Definitions

Replace function calls with their definitions to expose structure. This often
reveals patterns that match known laws.

### Step 3: Apply Laws

Apply the simplest applicable law at each step. Prefer fusion laws (they
eliminate intermediate structures). Name every law you apply.

### Step 4: Fold Back

Recognize the simplified expression as a known function or a simpler recursion
pattern. The universal property of fold is the main tool here.

### Step 5: Translate Back

If the target language is not Haskell, translate the derived expression back.
The equational structure guides you: a `foldr` becomes a loop, `map . filter`
becomes a single-pass iteration, etc.

## Translating to Imperative / Multi-Paradigm Languages

Equational reasoning works in any language where you can identify pure
subexpressions. The key translations:

| Functional Form     | Imperative Equivalent                     |
| ------------------- | ----------------------------------------- |
| `map f xs`          | `for x in xs: result.push(f(x))`          |
| `filter p xs`       | `for x in xs: if p(x): result.push(x)`    |
| `foldr f z xs`      | loop accumulating from right (or reverse) |
| `foldl f z xs`      | `acc = z; for x in xs: acc = f(acc, x)`   |
| `map f . filter p`  | single loop: `if p(x): push(f(x))`        |
| `foldr f z . map g` | single loop applying `g` inside `f`       |
| `concatMap f`       | `flatMap` / nested loop flattened         |

After deriving a simplification equationally, translate to idiomatic code in the
target language. The fusion laws tell you which loops to merge.

## Worked Example: Imperative Simplification

Original TypeScript:

```typescript
const names = users.map((u) => u.name);
const upper = names.map((s) => s.toUpperCase());
const long = upper.filter((s) => s.length > 3);
```

Equational calculation:

```
  filter (\s -> length s > 3) . map toUpper . map name
= { map fusion: map f . map g = map (f . g) }
  filter (\s -> length s > 3) . map (toUpper . name)
= { filter-map interchange is not directly applicable, }
  { but we can fuse into a single traversal via foldr: }
  foldr (\u acc -> let s = toUpper (name u)
                   in if length s > 3 then s : acc else acc) []
```

Result — single pass:

```typescript
const long = users.reduce((acc, u) => {
  const s = u.name.toUpperCase();
  return s.length > 3 ? [...acc, s] : acc;
}, []);
```

Or more idiomatically with `flatMap`:

```typescript
const long = users.flatMap((u) => {
  const s = u.name.toUpperCase();
  return s.length > 3 ? [s] : [];
});
```

## Common Pitfalls

- **Skipping steps**: The whole point is that each step is small and justified.
  If you find yourself making a "big jump", break it into smaller steps.
- **Applying laws across effect boundaries**: See "Reasoning About Effectful
  Code" above. Identify the pure core, check effect commutation, and only apply
  laws where substitution is safe.
- **Forgetting strictness**: In strict languages, fusion can change termination
  behavior. `map f . map g` and `map (f . g)` are semantically equivalent only
  when both `f` and `g` are total. Note any partiality.

## Presentation

When presenting a simplification to the user:

1. Show the original expression
2. Show the chain of equational steps, each labeled with the law used
3. Show the final simplified form
4. If translating back to imperative code, show the translation

This makes the reasoning transparent and verifiable.
