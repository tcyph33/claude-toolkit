---
name: coding-standards
description: >-
  Enforces coding standards for software construction.
  Apply these standards to ALL code written or modified, regardless of language.
  These are mandatory principles governing design, variables, control flow,
  defensive programming, self-documenting code, refactoring, and testing.
metadata:
  author: Tyler Cypher
  keywords: "coding-standards, quality, clean-code, construction, best-practices"
  tags: "coding-standards, code-quality, construction"
---

# Coding Standards

These standards are **mandatory** and apply to all code you write or modify, in any language.

---

## Workflow

Follow this exact sequence every time you write or modify code. Do not skip or reorder steps.

---

### Step 1 — Load Standards (before writing anything)

Execute these Read calls before writing a single line of code. Do not proceed until all reads are complete.

**Always read (line-level rules — apply to every line you write):**
1. `Read standards/variables.md`
2. `Read standards/control-flow.md`
3. `Read standards/self-documenting-code.md`
4. `Read standards/formatting.md`

**Read based on what you are writing:**

| You are writing… | Also read |
|-----------------|-----------|
| Any function or method | `standards/design.md`, `standards/parameters.md` |
| Error handling or validation | `standards/defensive-programming.md` |
| Data structures or collections | `standards/data-structures.md`, `standards/defensive-data.md` |
| Ordered or sequential operations | `standards/sequential-dependencies.md` |
| Tests | `standards/testing.md` |
| Refactoring existing code | `standards/refactoring.md` |

---

### Step 2 — Design-Level Pass (before writing)

Before writing any code, verify the planned structure against the **[D] design-level rules** in the checklist below. These rules govern the shape of what you are about to create — they cannot be fixed by a line-by-line review after the fact.

Ask yourself for each:
- Does every planned routine do exactly one thing? (1.1)
- Is coupling minimized? Are dependencies on abstractions, not implementations? (1.2, 1.3)
- Is complexity kept under 7 decision points? Nesting under 3 levels? (1.4)
- Is each routine operating at one consistent abstraction level? (1.5)
- Are parameters grouped if there are more than 7? (9.1)
- Are order dependencies enforced by structure, not comments? (11.1–11.3)
- Is the right data structure chosen for the access pattern? (8.2)
- Are validation barricades in the right place? (4.4)

If the design fails any of these, restructure before writing.

---

### Step 3 — Write the Code

Write the code. As you write, apply the line-level rules from the loaded standards files — do not defer this to the review pass.

---

### Step 4 — Spawn a Fresh Review Agent

After writing, spawn a **fresh agent** (not a fork — zero shared context) with the following prompt, substituting the actual code and the list of category files you read in Step 1:

```
You are reviewing code against a set of mandatory coding standards. Your job is to find violations — not to be lenient.

## Code to review
[paste the complete code you wrote]

## Your task

**First**, Read each of these standards files:
[list each file you read in Step 1, e.g. standards/design.md, standards/variables.md, ...]

**Then run two passes:**

### Pass 1 — Design-level (whole-code structure)
Check the code as a whole against rules marked [D] in the standards:
- 1.1–1.5 (Design & Complexity)
- 4.4 (Barricades)
- 8.1–8.3 (Data Structures)
- 9.1–9.4 (Parameters)
- 11.1–11.3 (Sequential Dependencies)
- 12.1–12.3 (Defensive Data)

### Pass 2 — Line-level (every line and block)
Go line by line through the code. For each line, check:
- 2.1–2.9 (Variables): scope, naming, types, parentheses, no inline returns
- 3.1–3.6 (Control Flow): named booleans only, guard clauses, no inline conditions
- 5.1–5.7 (Self-Documenting Code): block intent comments, docstrings, naming conventions
- 10.1–10.4 (Formatting): whitespace, line length, parallel structure

Do not skip a line. Do not group lines to save effort. Each line gets checked individually.

## Output format

For **every rule ID** in each pass, output exactly one line — no exceptions, no skipping:

```
RULE_ID | PASS
RULE_ID | FAIL | file:line | what is wrong | what it should be
```

Example:
```
2.1 | PASS
2.2 | FAIL | src/order.ts:14 | variable declared at top of function, only used inside loop | declare inside the loop
2.3 | PASS
2.4 | FAIL | src/order.ts:22 | boolean named `isNotValid` embeds negation | rename to `isValid`, negate with `!` at use sites
```

A response with missing rule IDs is incomplete and must be rejected. Every rule ID must appear in the output — PASS or FAIL.
```

---

### Step 5 — Apply Fixes

Apply every violation the review agent reported. Then confirm the fix resolves the violation without introducing new ones.

---

## Priority Order

When writing or modifying code, apply these standards in this priority order:

1. **Correctness** — Code must do what it claims (testing, boundary validation)
2. **Clarity** — Code must be understandable without author context (naming, self-documentation)
3. **Simplicity** — Prefer the simplest solution that works (complexity management, single responsibility)
4. **Robustness** — Code must handle failure gracefully (defensive programming, fail fast)
5. **Maintainability** — Code must be easy to change (loose coupling, information hiding, refactoring)

---

## Rule Checklist

`[D]` = design-level (Pass 1 — check against whole-code structure)
`[L]` = line-level (Pass 2 — check on every line/block)
`[D+L]` = both

### Design & Complexity (`standards/design.md`)
- **1.1** `[D]` Every routine does one thing; its name completely describes it
- **1.2** `[D]` Modules depend on as few others as possible; connections are narrow and explicit
- **1.3** `[D]` Internal data structures and implementation are hidden behind simple interfaces
- **1.4** `[D]` Cyclomatic complexity < 7; nesting ≤ 3 levels
- **1.5** `[D]` Each routine operates at a single, consistent level of abstraction

### Variables & Data (`standards/variables.md`)
- **2.1** `[L]` Variables declared and initialized as close as possible to first use; in the smallest scope that contains all uses
- **2.2** `[L]` Each variable given the smallest scope possible; local over instance, instance over class, class over global
- **2.3** `[L]` Names fully and accurately describe what the variable represents; no abbreviations except universally understood ones
- **2.4** `[L]` Boolean names always in positive/affirmative form (`isEmpty`, not `isNotEmpty`); negate with `!`
- **2.5** `[L]` Every meaningful literal number or string replaced with a named constant
- **2.6** `[L]` Each variable has exactly one purpose; never reused for a different meaning
- **2.7** `[L]` Types always declared explicitly; no reliance on type inference
- **2.8** `[L]` All arithmetic expressions fully parenthesized to make evaluation order unambiguous
- **2.9** `[L]` Never return an expression directly; always assign to a named variable first

### Control Flow (`standards/control-flow.md`)
- **3.1** `[L]` Normal/common case in the `if` clause; use guard clauses when one branch exits early
- **3.2** `[L]` Every `if`/`while` condition evaluates a named boolean variable, never an inline expression; each comparison parenthesized
- **3.3** `[L]` Complex boolean expressions decomposed into named intermediate variables; 3+ terms broken to one per line
- **3.4** `[L]` Loop has a single advancement mechanism; `for` header and body do not both mutate the index
- **3.5** `[L]` Guard clauses handle edge cases and errors at the top; main logic stays at base nesting level
- **3.6** `[L]` Only standard control structures used; no goto, no exceptions as control flow

### Defensive Programming (`standards/defensive-programming.md`)
- **4.1** `[D]` All data crossing a trust boundary validated; internal code trusts validated data without re-validating
- **4.2** `[L]` Every error path explicitly handled with a deliberate strategy; no empty catch blocks
- **4.3** `[L]` On detecting an inconsistency that makes further processing meaningless, stop immediately with a clear message
- **4.4** `[D]` Clear barricade in the architecture; validation outside, trusted processing inside

### Self-Documenting Code (`standards/self-documenting-code.md`)
- **5.1** `[L]` Code communicates intent through names and structure; no comments that restate what the code does
- **5.2** `[L]` Comments explain *why* a decision was made, never *what* the code does
- **5.3** `[L]` Non-obvious side effects, performance implications, and unintuitive behaviors documented
- **5.4** `[L]` Every logical block (blank-line-bounded) preceded by a one-line intent comment naming its purpose
- **5.5** `[L]` Routine name is a verb phrase for procedures, noun/adjective for value-returning functions; describes everything it does
- **5.6** `[L]` Every function has a documentation comment: summary, each parameter, return value (if any), exceptions
- **5.7** `[L]` Naming conventions consistent throughout; fixed constants in UPPER_SNAKE_CASE, runtime constants in regular variable casing

### Refactoring & Quality (`standards/refactoring.md`)
- **6.1** `[D+L]` Logic appearing 3+ times extracted into a shared routine
- **6.2** `[D+L]` Routines exceeding ~50-100 lines broken into focused, named helpers
- **6.3** `[D]` Difficulty naming a routine or variable treated as a design smell; restructure until a clear name emerges
- **6.4** `[L]` Small, safe, unambiguous fixes made immediately when encountered in the working area
- **6.5** `[L]` Immediate surroundings improved incrementally when modifying a file; no large unrelated changes

### Testing (`standards/testing.md`)
- **7.1** `[L]` Tests assert observable behavior, not implementation details; tests must not break on internal refactors
- **7.2** `[L]` Boundary conditions explicitly covered: empty, single element, maximum, null, duplicates, type edges
- **7.3** `[D]` Each test runs in isolation in any order; no shared mutable state between tests
- **7.4** `[D]` Each test verifies a single behavior or scenario
- **7.5** `[L]` Test names read as specifications: what scenario is tested and what the expected outcome is

### Data Structures & Table-Driven Methods (`standards/data-structures.md`)
- **8.1** `[D]` Complex conditional mappings (long if/else or switch chains) replaced with data-driven table lookups
- **8.2** `[D]` Data structure chosen based on actual access patterns: insertion frequency, lookup frequency, ordering requirements
- **8.3** `[D]` Complex data structures with non-trivial access rules wrapped in a class or module that enforces them

### Routine Parameters (`standards/parameters.md`)
- **9.1** `[D]` No more than 7 parameters; group related parameters into a cohesive object when exceeded
- **9.2** `[D]` Parameter ordering convention consistent across all related routines in a module
- **9.3** `[L]` Parameter constraints not obvious from the type explicitly documented or enforced via assertions
- **9.4** `[D]` No boolean flag parameters; use separate named routines or an enum instead

### Code Layout & Formatting (`standards/formatting.md`)
- **10.1** `[L]` Blank lines separate logical sections; visual shape of code reflects its logical shape
- **10.2** `[L]` Lines limited to 80-120 characters; broken at logical seams (after comma, before operator)
- **10.3** `[L]` Code operating on the same data or contributing to the same logical step is physically adjacent
- **10.4** `[L]` Parallel structures formatted in parallel; one space before and after operators; no column-aligned assignments

### Sequential Dependencies & Temporal Coupling (`standards/sequential-dependencies.md`)
- **11.1** `[D]` Order dependencies enforced by code structure (return values, type state), not by comments or convention
- **11.2** `[D+L]` Data flow between statements makes ordering dependencies explicit; independent statements grouped by affinity
- **11.3** `[D]` Multi-step sequences that must always happen together wrapped in a single routine

### Defensive Data Handling (`standards/defensive-data.md`)
- **12.1** `[D]` Data treated as immutable by default; mutations are explicit and contained
- **12.2** `[D+L]` Defensive copies made when receiving mutable data from a caller or returning internal mutable data
- **12.3** `[D+L]` Multiple references to the same mutable structure avoided unless intentional and documented
