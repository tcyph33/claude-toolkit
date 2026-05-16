---
name: code-review
description: >-
  Review a GitHub pull request for correctness, completeness, code quality, simplicity,
  test coverage, and documentation. Use when user says "review this PR", "code-review",
  "check this PR", "look at this pull request", or provides a GitHub PR URL and asks
  for feedback. Produces a structured local review — does not post comments on the PR.
metadata:
  author: Tyler Cypher
  keywords: "code-review, pull-request, pr-review, correctness, completeness, quality, simplicity, testing, security, performance"
  tags: "code-review, code-quality"
---

# Code Review Skill

Reviews a GitHub pull request locally and produces a structured assessment. Does NOT post comments on the PR — the review stays in your terminal for you to decide what feedback is worth leaving.

## When to Invoke

Use when a user:
- Provides a GitHub PR URL and asks for a review
- Says "review this PR", "check this PR", "look at this pull request"
- Runs `/code-review <url>`

## Invocation Format

```bash
/code-review <PR_URL> [--context "<description of intent>"] [--focus "<additional areas to emphasize>"]
```

- `PR_URL` (required): Full GitHub PR URL (e.g., `https://github.com/org/repo/pull/123`)
- `--context` (optional): Description of the PR's intent. Helps the reviewer understand *why* the changes are being made.
- `--focus` (optional): Additional review areas to emphasize on top of the standard checklist. Does NOT replace the standard checks.

## Workflow

### Step 1: Fetch the PR and Clone the Repository

Extract org, repo, and PR number from the URL.

<!-- Maintainer note: We use a shallow clone (--depth=50) because we only need the current
     state of files for review, not full git history. This keeps clone times fast and disk
     usage low for large repos while still providing all files at their current state. -->

Clone the repo at the PR's head branch for full file access during review:

```bash
# Get the PR branch name
PR_BRANCH=$(gh pr view <number> --repo <org>/<repo> --json headRefName --jq '.headRefName')

# Create the review directory if it doesn't exist
mkdir -p ~/PR-Reviews

# Shallow clone the PR branch using gh (handles SSL/auth correctly in corporate environments)
gh repo clone <org>/<repo> ~/PR-Reviews/<repo>-<number> -- --depth=50 --branch "$PR_BRANCH"
```

Fetch the diff to know what changed:

```bash
gh pr diff <number> --repo <org>/<repo>
```

Also fetch PR metadata for context:

```bash
gh pr view <number> --repo <org>/<repo> --json title,body,files
```

### Step 2: Analyze the Diff

Read the full diff carefully. If the diff is large (>500 lines), process it in sections but still review ALL of it.

Before evaluating individual checks, establish:
1. **What is this PR doing?** (use `--context` if provided, otherwise infer from the PR description and diff)
2. **What is the scope?** (which systems/files are touched)
3. **What is the risk?** (breaking changes, public API changes, data migrations, etc.)
4. **Requirements alignment**: Does this PR actually achieve what the ticket/task specifies?

### Step 3: Validate Against the Full Codebase

Using the cloned repo, go beyond the diff.

**Core principle: Reason about indirect effects of every change.** For each change in the diff, ask: "What else in this repo *depends on* the thing that changed?" Then verify those dependents still work — even if they aren't in the diff. The most dangerous breakages happen in files the PR *didn't* touch. Don't treat "not modified" as "not affected."

Specific checks:

- **Read full files** that the diff touches — not just the changed lines, but surrounding context
- **Verify field names, property names, function calls, and values** against their source of truth (schemas, types, interfaces, APIs, configs) defined elsewhere in the repo. **Trigger**: any time you see a name being used — whether in code, YAML, JSON, documentation, examples, or tests — confirm it actually exists and is valid in the definition it references. Do not assume a name is valid just because it was already there before the PR.
- **Grep the entire repo for leftover references** to removed/renamed concepts — not just the files in the diff. Run `grep -r "<removed-concept>"` across the full cloned repo to find stragglers the PR author missed.
- **Check that examples use real property names** by cross-referencing against the source of truth (schema files, type definitions, config specs)
- **Look for related code** that should have been updated but wasn't included in the diff
- **Verify import/dependency graphs** — when files are deleted or exports are removed, confirm nothing still imports or references them. In typed languages the compiler may catch this, but in dynamic languages, docs, configs, and scripts it won't.
- **Check generated/derived files** — if the repo uses code generation (schemas → types, OpenAPI → clients, Zod → JSON schemas, projen → configs, lock files), verify they've been regenerated to reflect the source changes. Stale generated files are a common miss.
- **Check CI/CD and build tool configs** — look for references to deleted paths in package.json scripts, Makefile targets, CI workflows, projen tasks, Dockerfiles, and similar configuration files.

### Step 4: Run the Review Checklist

Evaluate every category below. For each finding, record the severity and specific file:line reference.

---

#### 3.1 Functionality & Correctness

Does the code actually do what it claims to do?

| Check | What to look for |
|-------|-----------------|
| CR-1 | **Logic errors**: off-by-one, wrong comparisons, using `and` instead of `or` |
| CR-2 | **Type safety**: incorrect type usage, unsafe casts, implicit `any` |
| CR-3 | **Edge cases**: boundary conditions, null inputs, empty collections, maximum limits |
| CR-4 | **Unexpected scenarios**: what happens if an API returns null, a network call fails, or the user triggers an action twice? |
| CR-5 | **Race conditions**: concurrency issues, thread-safety concerns, shared mutable state |
| CR-6 | **Data integrity**: database calls using parameters/sanitization to prevent injection |
| CR-7 | **Breaking changes**: modifications to public APIs or exported interfaces |
| CR-8 | **Downstream consumer impact**: if this is a library/package, will consumers break? Are there deprecation warnings, migration guides, or appropriate semver version bumps? |
| CR-9 | **Temporary workarounds**: hacks must be clearly commented with a plan for removal (link to ticket) |

---

#### 3.2 Design & Architecture

Is the code well-structured and in the right place?

| Check | What to look for |
|-------|-----------------|
| DA-1 | **Single Responsibility**: does each class/function do one thing, or is it trying to do too much? |
| DA-2 | **Dumping ground prevention**: avoid generic files like `utils.ts`, `helpers.js`, or `CommonManager`. If a file handles multiple unrelated domains, it must be split. |
| DA-3 | **Cohesion**: do the functions in a file actually relate to each other, or are they unrelated code sharing a room? |
| DA-4 | **Code location**: is logic placed as close as possible to where it's used? Is it in the correct module/directory consistent with existing architecture? |
| DA-5 | **Over-engineering (YAGNI)**: did the developer solve the current problem, or are they speculating on future needs? |
| DA-6 | **DRY principle**: is there duplicated logic that should be extracted into a shared utility or service? |
| DA-7 | **Tight coupling**: are there dependencies that make future changes risky or expensive? |
| DA-8 | **Reusability**: could this code be used elsewhere, or does it already exist elsewhere? |
| DA-9 | **Interface ergonomics**: for any new user-facing surface (CLI flags, API parameters, config options, function signatures), evaluate from the consumer's perspective. Would a user find the interface intuitive, or would they need to read the source to understand what to pass and why? |

---

#### 3.3 Readability & Maintainability

Can someone else understand this quickly?

| Check | What to look for |
|-------|-----------------|
| RM-1 | **Naming clarity**: do variable, function, and class names clearly communicate intent? |
| RM-2 | **Complexity**: deep nesting (3+ levels), long functions (>40 lines), complex conditionals — suggest early returns, guard clauses, extraction, or named booleans |
| RM-3 | **Comment quality**: comments explain *why* something exists, not just *what* the code does |
| RM-4 | **Dead code**: unused variables, functions, or commented-out code that should be removed |
| RM-5 | **Magic numbers/strings**: are constants defined, or are there hardcoded values that should be named variables? |
| RM-6 | **Debug artifacts**: leftover `console.log`, `print`, `debugger`, or debug statements |
| RM-7 | **Overly clever code**: prefer straightforward alternatives over clever one-liners |
| RM-8 | **Control flow**: can the logic be followed without mental stack tracking? |

---

#### 3.4 Completeness

Are all related files updated consistently?

| Check | What to look for |
|-------|-----------------|
| CM-1 | Documentation updated to reflect code changes (README, API docs, wiki) |
| CM-2 | Related configuration files updated (package.json, tsconfig, CI configs) |
| CM-3 | All references to renamed/removed things are updated (no dangling imports, broken links) |
| CM-4 | Migration path documented if this is a breaking change |
| CM-5 | Examples and sample code still work as standalone, realistic configurations after changes |

---

#### 3.5 Semantic Validity

After mechanical changes (find-and-replace, renames, type swaps, deletions), does everything still make logical sense? This catches a class of bugs where code is technically "updated" but no longer correct in context.

| Check | What to look for |
|-------|-----------------|
| SV-1 | **Examples are realistic**: after changes, do examples still represent valid, working configurations? A label like "Scheduled job" with no schedule attached is a semantic lie. |
| SV-2 | **Comments match reality**: do descriptions, docstrings, and inline comments still accurately describe what the code does after the change? |
| SV-3 | **Names reflect purpose**: do variable/function names still reflect their actual purpose after refactoring? |
| SV-4 | **Tables and enums are coherent**: do table rows, enum values, and structured documentation still make logical sense? (e.g., a generic component shouldn't have a column that assumes a specific trigger type) |
| SV-5 | **No ghost references**: after removing a concept, are there any leftover references that assume it still exists? |
| SV-6 | **Pre-existing invalid code surfaced by the PR**: when a PR modifies a line or its surrounding context, check whether the code *already* contained invalid values, outdated field names, or incorrect assumptions that were not introduced by this PR but are now visible/relevant. If the PR touches the area, flag these as "pre-existing issue surfaced by this change" — they should be fixed as part of the PR or explicitly noted as out of scope. |

---

#### 3.6 Testing & Quality

Do tests verify behavior correctly and completely?

| Check | What to look for |
|-------|-----------------|
| TC-1 | **Behavior verification**: tests verify actual outcomes, not just that code runs without throwing |
| TC-2 | **Edge cases**: empty inputs, nulls, boundary values, error paths all covered |
| TC-3 | **No duplicates**: no redundant tests verifying the same behavior differently (AI bloat) |
| TC-4 | **No test leakage into production**: no exports that exist only for testing, no test-only conditionals, no test helpers in `src/` |
| TC-5 | **Production code independence**: no `export` on internals solely for test access, no `if (process.env.NODE_ENV === 'test')` patterns |
| TC-6 | **Specific assertions**: not just `toBeDefined()` or `toHaveBeenCalled()` without verifying args/values |
| TC-7 | **Coverage of new paths**: new code paths introduced by the PR have corresponding test coverage |
| TC-8 | **Test independence**: tests run in any order without relying on shared mutable state |
| TC-9 | **Test reliability**: are tests non-flaky? No timing dependencies, no reliance on external services without mocking |
| TC-10 | **Tautological assertions**: trace the data flow from mock → code under test → assertion. If an assertion only confirms that mocked data passes through unchanged (no branching, transformation, or error handling exercised), it is testing the harness, not the code. Flag these as low-value assertions that give false confidence in coverage. |

---

#### 3.7 Security & Privacy

Is the code safe from common vulnerabilities?

| Check | What to look for |
|-------|-----------------|
| SP-1 | **Input validation**: all user/external input sanitized to prevent XSS, SQL injection, command injection |
| SP-2 | **Secret handling**: credentials, API keys, or tokens excluded from source control |
| SP-3 | **Sensitive data**: PII encrypted at rest and in transit where applicable |
| SP-4 | **Error messages**: do error messages or logs reveal too much information to potential attackers? |
| SP-5 | **Unsafe deserialization**: untrusted data not deserialized without validation |

---

#### 3.8 Performance & Efficiency

Is the code efficient with resources?

| Check | What to look for |
|-------|-----------------|
| PE-1 | **Database calls**: unnecessary queries (e.g., querying inside a loop, N+1 problems) |
| PE-2 | **Resource usage**: not loading huge datasets into memory unnecessarily |
| PE-3 | **Async opportunities**: synchronous calls that could be asynchronous to prevent blocking |
| PE-4 | **Caching**: is caching used where applicable to improve performance? |
| PE-5 | **Algorithmic complexity**: O(n²) or worse where a better approach exists |

---

#### 3.9 Documentation & Intent

Is non-obvious logic explained?

| Check | What to look for |
|-------|-----------------|
| DI-1 | Tricky algorithms or non-obvious logic has comments explaining *why* (not *what*) |
| DI-2 | Assumptions are explicitly documented (e.g., "assumes input is sorted", "caller guarantees non-null") |
| DI-3 | Workarounds reference the issue they work around (ticket, bug, upstream limitation) |
| DI-4 | Business logic that isn't self-evident from the code has intent documentation |
| DI-5 | Do NOT flag missing comments on straightforward, self-documenting code |

---

### Step 5: Pre-Report Self-Check

Before finalizing findings, pause and verify:

1. **Did I actually verify my claims?** For every issue or "looks good" statement, confirm you read the relevant code — don't assume correctness because the pattern looks right.
2. **Did I grep the full repo?** If the PR removes or renames something, did I search beyond just the diff files?
3. **Did I check the result, not just the delta?** The diff may be mechanically correct but produce an invalid end state. Read the final version of modified files, not just the changed lines.
4. **Am I missing generated files?** If schemas, types, or configs changed, are the derived artifacts up to date?
5. **Would a downstream consumer break?** If this is a library, think about what happens when someone upgrades.

---

### Step 6: Determine Verdict

Based on findings, assign an overall verdict:

| Verdict | Criteria |
|---------|----------|
| **APPROVE** | No issues found, or only minor nits that don't affect correctness. Safe to merge as-is. |
| **APPROVE WITH NITS** | Minor issues worth noting but not blocking. Could be addressed in a follow-up. |
| **NEEDS CHANGES** | Issues that should be fixed before merging. |
| **NEEDS DISCUSSION** | Architectural or design questions that require human judgment. |

### Step 7: Generate the Review Report

```markdown
# PR Review: <PR title>
**PR**: <URL>
**Verdict**: APPROVE | APPROVE WITH NITS | NEEDS CHANGES | NEEDS DISCUSSION

## Summary
[2-3 sentences: what the PR does, overall assessment, key concern if any]

## Issues

### [SEVERITY] <Check ID>: <Short title>
**File**: `path/to/file:line`
**Issue**: What is wrong
**Suggestion**: How to fix it

(Repeat for each issue found. Order by severity: blockers first, then warnings, then nits.)

## What Looks Good
- [Specific positive observations about the PR — acknowledge good solutions or elegant code]

## Verdict Rationale
[1-2 sentences explaining why you chose this verdict]
```

**Severity levels:**
- **BLOCKER**: Must fix before merge (correctness bugs, security issues, breaking changes, missing tests for new behavior)
- **WARNING**: Should fix before merge (incomplete docs, semantic validity issues, test quality problems, design concerns)
- **NIT**: Optional improvement (style, minor simplification, documentation polish)

### Step 8: Offer Next Steps

After presenting the report:
- "Would you like me to look deeper at any specific area?"
- "Want me to suggest fixes for any of these issues?"
- "The cloned repo is at `~/PR-Reviews/<repo>-<number>`. Want me to clean it up, or would you like to keep it for further investigation?"

**Do NOT delete the cloned repo until the user explicitly confirms.** Always ask first.

## Important Notes

- This skill is **read-only** — it does NOT post comments or approve PRs on GitHub
- Always cite specific file paths and line numbers
- When the `--context` is provided, verify that the PR actually achieves the stated intent
- When `--focus` is provided, add those areas as additional emphasis but still run the full checklist
- For large PRs, do not skip sections — review everything even if it takes multiple passes
- Be concise for clean PRs — don't pad the report with "PASS" lines for things that are fine
- Only report actual findings, not a checklist of everything you checked
- If the PR is trivially correct (typo fix, version bump), say so briefly and give APPROVE
- Focus on the code, not the person — be constructive
- Don't rush — thorough reviews catch critical bugs
