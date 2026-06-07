---
description: Spec-driven feature implementation. Plan, code, build, test and iterate.
argument-hint: "[feature-spec]"
allowed-tools: Bash(git:*), Bash(pwsh:*), Grep, Glob
model-recommended: Sonnet
---

Implement feature end-to-end: spec, analyze, plan, code 5 phases (TDD), build, test, iterate.

Feature spec: $ARGUMENTS

---

## Step 0 — Parse spec and clarify

```pwsh
$spec = '$ARGUMENTS'
if (-not $spec.Trim()) { throw 'Spec required. Example: "Add JSON validation to config loader"' }
Write-Host "Feature: $spec"
function vout($s) { Write-Host $s }
```

**Before planning, ask user if not mentioned:**
- Scope: new API, modify existing, or refactor?
- Acceptance criteria: what passes/fails?
- Affected modules: which files/components?
- Performance/security: constraints?
- Backwards compatible: must old callers work?

---

## Step 1 — Analyze current code

```pwsh
$keywords = $spec -split '\s+' | Select-Object -First 3
Write-Host "`nAnalyzing..."

foreach ($kw in $keywords) {
    $matches = git grep -l "$kw" 2>$null | Select-Object -First 5
    if ($matches) { Write-Host "  $kw: $($matches.Count) files"; $matches | % { Write-Host "    - $_" } }
}
```

**Report:**
- Complexity: low/medium/high
- Effort estimate: X-Y hours
- Architectural concerns?
- Can be improved vs baseline approach?

If unsatisfied, iterate Step 1. If approved, continue.

---

## Step 2 — Create test acceptance criteria

Define success (TDD — tests before code):

```
Feature: [Name]

Scenario 1: [happy path]
  Given: <setup>
  When: <action>
  Then: <expected result>

Scenario 2: [edge case]
  ...

Scenario 3: [error case]
  ...
```

**Show to user. Approve before coding.**

---

## Step 3 — Build implementation plan

Phased approach:

```
Phase 1: Scaffold
  [ ] Create/stub files → compile only

Phase 2: Core logic
  [ ] Main algorithm → unit tests pass

Phase 3: Error handling
  [ ] Validation, edge cases → all tests pass

Phase 4: Integration
  [ ] Wire into calling code → integration tests pass

Phase 5: Final
  [ ] Full suite, iterate, code review ready
```

**User approves plan + edits.**

---

## Step 4 — Phase 1: Scaffold

- Create/modify files as per plan
- Add function stubs
- Build: `cmake --build ...` (from docs/build_test_run.md)
- Commit: "Phase 1: Scaffold [feature]"

Iterate until clean build.

---

## Step 5 — Phase 2: Core logic

- Implement main algorithm
- Add unit tests (happy path)
- Build + test
- Commit: "Phase 2: Core [feature]"

Iterate until Phase 2 tests pass.

---

## Step 6 — Phase 3: Error handling

- Input validation
- Error cases, edge cases
- More tests
- Commit: "Phase 3: Errors [feature]"

Iterate until all tests pass.

---

## Step 7 — Phase 4: Integration

- Wire feature into calling code
- Update docs/comments
- Integration tests (if applicable)
- Commit: "Phase 4: Integration [feature]"

Build + full test suite passes.

---

## Step 8 — Phase 5: Final & review

Run full test suite from docs/build_test_run.md.

If failures:
  - Fix code
  - Re-test
  - Iterate

---

## Workflow

```
Spec → Clarify → Analyze → Test criteria → Plan
  ↓
Phase 1 (Scaffold) → Phase 2 (Core) → Phase 3 (Errors) → Phase 4 (Integration) → Phase 5 (Final)
  ↓
/mr-audit → /main-rebase (if behind) → /mr-create
```

**Key:**
- Tests define success (write first, code second)
- Commit after each phase (logical checkpoints)
- Iterate on failures immediately
- Ask user when uncertain
- Use docs/build_test_run.md (never improvise builds/tests)
