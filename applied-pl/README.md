# Applied PL Theory Plugin

A Claude Code plugin providing formal methods for program reasoning and
optimization.

## Installation

This plugin is installed in your local plugins directory:

```
~/.claude/plugins/marketplaces/local-plugins/applied-pl/
```

## Skills

### 1. equational-reasoning

Simplify and optimize programs using algebraic laws and program calculation.

**Use when**:

- Simplifying functional pipelines (map/filter/fold chains)
- Eliminating intermediate data structures (deforestation / fusion)
- Refactoring recursive definitions into cleaner forms
- Simplifying try-catch or Promise.catch chains
- Separating pure logic from effectful shells (async, state, I/O)
- Deriving efficient implementations from naive specifications
- Verifying that a refactor preserves semantics

**Example**: Transform
`users.map(u => u.name).map(s => s.toUpperCase()).filter(s => s.length > 3)`
into a single-pass operation.

### 2. hoare-logic

Verify correctness of imperative programs using pre/postconditions and loop
invariants.

**Use when**:

- Proving loop correctness via Hoare triples `{P} C {Q}`
- Finding and verifying loop invariants
- Calculating weakest preconditions (`wp`)
- Checking array bounds and index safety
- Establishing termination via variants
- Comparing implementations for equivalence

**Example**: Prove that a binary search correctly finds elements or returns -1.

## Quick Start

Both skills emphasize **rigorous derivation** over informal reasoning:

- **Equational reasoning**: Show each transformation step with the law that
  justifies it
- **Hoare logic**: State triples {P} C {Q} and discharge proof obligations
  explicitly

## Files

- `.claude-plugin/plugin.json` - Plugin manifest
- `skills/equational-reasoning/SKILL.md` - Equational reasoning skill
- `skills/hoare-logic/SKILL.md` - Hoare logic skill
- `README.md` - This file

## Choosing a Skill

| Your Code                       | Use This Skill       |
| ------------------------------- | -------------------- |
| Pure functional transformations | equational-reasoning |
| Loops with state                | hoare-logic          |
| List/array processing           | equational-reasoning |
| Imperative algorithms           | hoare-logic          |
| Simplifying expressions         | equational-reasoning |
| Proving correctness             | hoare-logic          |

For mixed imperative/functional code, use both: simplify pure parts with
equational reasoning, verify stateful parts with Hoare logic.

## Version

1.1.0 (2026-04-17)
