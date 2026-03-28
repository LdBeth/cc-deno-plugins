# Applied PL Theory Plugin

A Claude Code plugin providing formal methods for program reasoning and optimization.

## Installation

This plugin is already installed in your local plugins directory:
```
~/.claude/plugins/marketplaces/local-plugins/applied-pl-theory/
```

## Skills

### 1. equational-reasoning
Simplify and optimize programs using algebraic laws and program calculation.

**Use when**:
- Simplifying functional pipelines (map/filter/fold chains)
- Eliminating intermediate data structures (fusion)
- Refactoring recursive functions
- Optimizing Promise.catch or try-catch chains
- Deriving efficient implementations from specifications

**Example**: Transform `users.map(u => u.name).map(s => s.toUpperCase()).filter(s => s.length > 3)` into a single-pass operation.

### 2. hoare-logic
Verify correctness of imperative programs using pre/postconditions and loop invariants.

**Use when**:
- Proving loop correctness
- Finding and verifying loop invariants
- Checking array bounds and index safety
- Deriving preconditions for functions
- Establishing termination guarantees
- Comparing implementations for equivalence

**Example**: Prove that a binary search correctly finds elements or returns -1.

## Quick Start

Both skills emphasize **rigorous derivation** over informal reasoning:

- **Equational reasoning**: Show each transformation step with the law that justifies it
- **Hoare logic**: State triples {P} C {Q} and discharge proof obligations explicitly

## Files

- `PLUGIN.md` - Plugin manifest and overview
- `AUDIT.md` - Quality assessment and recommendations
- `skills/equational-reasoning.md` - Equational reasoning skill
- `skills/hoare-logic.md` - Hoare logic skill
- `README.md` - This file

## Choosing a Skill

| Your Code | Use This Skill |
|-----------|----------------|
| Pure functional transformations | equational-reasoning |
| Loops with state | hoare-logic |
| List/array processing | equational-reasoning |
| Imperative algorithms | hoare-logic |
| Simplifying expressions | equational-reasoning |
| Proving correctness | hoare-logic |

For mixed imperative/functional code, use both: simplify pure parts with equational reasoning, verify stateful parts with Hoare logic.

## Version

1.0.0 (2026-03-28)
