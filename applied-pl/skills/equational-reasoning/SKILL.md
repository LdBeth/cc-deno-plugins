---
name: equational-reasoning
description: Use equational reasoning to simplify and optimize programs. Apply this skill when asked to simplify, refactor, clean up, deduplicate, or make code more efficient — especially when code involves functional/effect programming patterns. Also use when the user mentions "program calculation", "fusion", or "algebraic simplification".
type: skill
---

# Equational Reasoning for Program Simplification

Equational reasoning treats programs as mathematical expressions and transform
them using substitution and algebraic laws. Each step must be justified by a
named law — never jump to the answer.

## When to Use This

- Simplifying composed operations (nested maps, filters, folds)
- Eliminating intermediate data structures (deforestation / fusion)
- Refactoring recursive definitions into cleaner forms
- Simplifying try-catch / Promise.catch chains
- Verifying that a refactor preserves semantics
- Deriving an efficient implementation from a naive specification

## Fundamental Laws

### Function Composition

```
(f . g) x   =  f (g x)
f . id      =  f                         -- right identity
id . f      =  f                         -- left identity
(f . g) . h =  f . (g . h)               -- associativity
```

### Map

```
map id           =  id
map f . map g    =  map (f . g)           -- fusion (eliminates intermediate list)
map f (xs ++ ys) =  map f xs ++ map f ys
map f . concat   =  concat . map (map f)  -- naturality
```

### Filter

```
filter p . filter q  =  filter (\x -> q x && p x)  -- fusion
filter p . map f     =  map f . filter (p . f)     -- when f is total
```

### Fold (foldr)

```
foldr f z []      =  z
foldr f z (x:xs)  =  f x (foldr f z xs)
foldr f z . map g =  foldr (f . g) z     -- fold-map fusion
foldr (:) []      =  id
```

**Universal property of fold**: `h = foldr f z` iff `h [] = z` and
`h (x:xs) = f x (h xs)`. Use this to recognize when a recursive function is a
fold and to fuse operations.

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

Dual to fold. When a loop builds a list via unfold and another consumes it via
fold, the two can be fused into a single pass (hylomorphism).

### Scan

```
scanl f z  =  map (foldl f z) . inits   -- specification
```

### Monad Laws

```
return a >>= f   =  f a                             -- left identity
m >>= return     =  m                               -- right identity
(m >>= f) >>= g  =  m >>= (\x -> f x >>= g)         -- associativity
```

### Exception / Try-Catch

Try-catch is the operational encoding of `Either<Error, T>`:
`try { expr } catch(e) { h(e) }  ≅  either h id (toEither expr)`

```
try { throw x } catch(e) { f(e) }  =  f(x)      -- throw-catch elimination
try { expr } catch { h }           =  expr      -- when expr never throws
try { f() } catch(e) { throw e }   =  f()       -- catch-rethrow elimination
try { e } catch(_) { e }           =  e         -- idempotence
```

Nested try-catch flattening:

```
try { try { a } catch(e) { b(e) } } catch(e) { c(e) }
= try { a } catch(e) { try { b(e) } catch(e2) { c(e2) } }
-- further, when b never throws:
= try { a } catch(e) { b(e) }
```

**Caveat**: assumes exceptions are the only effect. If the try block also
mutates state, catch branches cannot be freely reordered. Isolate the pure data
flow (the Either skeleton), simplify, then add effects back at the boundary.

### Promise.catch (async exception monad)

```
Promise.reject(e).catch(f)       =  Promise.resolve(f(e))       -- left identity
p.catch(e => Promise.reject(e))  =  p                           -- right identity
p.catch(f).catch(g)              =  p.catch(x => Promise.resolve(f(x)).catch(g))
-- when f never throws:
p.catch(f).catch(g)              =  p.catch(f)                  -- dead catch elimination
```

**Non-commutativity**: `.then(f).catch(g)` catches errors from both `p` and `f`;
`.catch(g).then(f)` recovers first then applies `f`. These are not
interchangeable.

## Procedure

1. **State the equation**: write the expression clearly; transcribe imperative
   code into functional style to expose structure.
2. **Unfold definitions**: replace calls with definitions to reveal patterns
   matching known laws.
3. **Apply laws**: prefer fusion laws (they eliminate intermediate structures);
   name every law.
4. **Fold back**: recognize the result using the universal property of fold.
5. **Translate back**: `foldr` → loop, `map . filter` → single-pass iteration,
   etc.

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
target language.

## Worked Example: Map Fusion + Filter

Original TypeScript:

```typescript
const names = users.map((u) => u.name);
const upper = names.map((s) => s.toUpperCase());
const long = upper.filter((s) => s.length > 3);
```

```
  filter (\s -> length s > 3) . map toUpper . map name
= { map fusion }
  filter (\s -> length s > 3) . map (toUpper . name)
= { fuse into single foldr traversal }
  foldr (\u acc -> let s = toUpper (name u)
                   in if length s > 3 then s : acc else acc) []
```

Result:

```typescript
const long = users.flatMap((u) => {
  const s = u.name.toUpperCase();
  return s.length > 3 ? [s] : [];
});
```

## Worked Example: Promise Chain

Original:

```typescript
fetchUser(id)
  .catch((e) => Promise.reject(e)) // right identity → eliminates
  .then((user) => enrichUser(user))
  .catch((e) => Promise.reject(e)) // right identity → eliminates
  .then((result) => result); // then-identity → eliminates
```

Result: `fetchUser(id).then((user) => enrichUser(user))`

## Common Pitfalls

- **Skipping steps**: each step must be small and justified; break "big jumps"
  into smaller steps.
- **Effectful code**: equational reasoning assumes referential transparency.
  Isolate pure fragments before applying laws.
- **Strictness**: in strict languages, `map f . map g = map (f . g)` only when
  both `f` and `g` are total. Note any partiality.
