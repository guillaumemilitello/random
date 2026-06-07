---
description: Code audit. Top 15 issues, GitLab pipeline/comments. Parallel sub-agents.
argument-hint: "[branch-name] [path-scope]"
allowed-tools: Bash(git:*), Bash(pwsh:*), Grep, Glob
model-recommended: Sonnet
---


Complete audit of current or specified branch. Produces top 15 issues with severity, GitLab pipeline status, and comments.

Replaces: mr-ready + mr-review (consolidated for token efficiency)

Branch override (optional): $ARGUMENTS | Scope filter (optional): follow branch arg

---

## Step 0 — Parse args and resolve branch

```pwsh
$args = "$env:ARGS" -split '\s+' | ? { $_ }
$branchArg = $args[0]
$scopeArg = $args[1..99] | Join-String -Separator ' '

$curr = git rev-parse --abbrev-ref HEAD 2>$null
if ($branchArg -and $branchArg -ne $curr) {
    git checkout $branchArg
    if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Cannot checkout $branchArg"; exit 1 }
}
$curr = git rev-parse --abbrev-ref HEAD 2>$null
Write-Host "Auditing: $curr"

# quiet helper
function vout($s) { Write-Host $s }
```

## Step 1 — Collect files and metadata

```pwsh
$base = git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null
if ($LASTEXITCODE -eq 0) { $base = $base -replace '^origin/', '' }
else { $base = git remote show origin | Select-String 'HEAD branch' | % { ($_ -split ':')[-1].Trim() } }

git fetch origin $base 2>&1 | Out-Null

$files = git diff --name-only "origin/$base...HEAD" | ? { $_ }
if ('$scopeArg'.Trim()) { $files = $files | ? { $_ -like "*$scopeArg*" } }

Write-Host "`nModified: $($files.Count) files"
$files | % { Write-Host "  - $_" }

$cpp = $files | ? { $_ -match '\.(cpp|h|hpp|cc)$' }
$py = $files | ? { $_ -match '\.py$' }
$proto = $files | ? { $_ -match '\.proto$' }
$other = $files | ? { $_ -notmatch '\.(cpp|h|hpp|cc|py|proto)$' }

Write-Host "`nBreakdown: C++: $($cpp.Count) | Python: $($py.Count) | Proto: $($proto.Count) | Other: $($other.Count)"
```

## Step 2 — Run existing linters

| Tool | Command |
|---|---|
| `.clang-format` | `cmake --build --preset conan-relwithdebinfo --target run_clang_format_check` |

Run only if present. Skip otherwise.

## Step 3 — Parallel sub-agent analysis

**Sub-Agent A**: Code Quality & Standards (naming, docs, patterns)
**Sub-Agent B**: Security & Bugs (null deref, hardcoded secrets, input validation)
**Sub-Agent C**: Test & Docs (coverage, docstrings, completeness)

Each analyzes modified files independently.

---

## Step 4 — Security & bug review (OWASP-aware)

- **C++**: null deref, use-after-free, buffer overflow, raw `new`/`delete`, thread-unsafe singletons, uncaught exceptions, integer overflow, dangling refs, uninitialized reads, missing `virtual` dtor, signed/unsigned compare, narrowing, format strings
- **Python**: untrusted → `subprocess`/`os.system`, `eval`/`exec`, `yaml.load` no SafeLoader, broad `except:`, mutable defaults
- **All**: hardcoded secrets, race conditions, missing input validation, SQL/command injection, unsafe deserialization

## Step 5 — Modernity check

**C++23/20**: `std::expected` over codes, `std::ranges`, structured bindings, `if constexpr`, concepts, `std::string_view`, `std::span`, `std::optional`, `std::format`, `constexpr`, smart pointers, `= default`/`= delete`

**Python**: type hints, `dataclasses`/`TypedDict`/`pydantic`, `pathlib.Path`, `match`/`case`, f-strings, `async`/`await`

## Step 6 — Coding standards

Read `docs/cpp_coding_standard.md` + `docs/python_coding_standard.md`. Validate:
- **C++**: `CamelCase` classes/methods, `lower_case` locals, `m_` members, `s_` statics, `g_` globals, `_ptr` pointers, `_opt` optionals
- **Python**: PEP 8
- **Docs**: Doxygen in headers only; `@brief` required; free functions only

## Step 7 — Typos

Scan comments, docstrings, strings, logs, identifiers.

## Step 8 — Test coverage

- New functions/branches → unit tests (GTest/pytest)?
- New public API → integration tests?
- Edge cases (empty, boundary, error)?

Look in matching `tests/` directory.

## Step 9 — C++ performance

For each `.cpp`/`.h`:
- Unnecessary copies: value params → `const&` or move; missing `std::move`; NRVO-ineligible returns
- Pass-by-value antipatterns: `std::string`/`std::vector` → `std::string_view`/`std::span`/`const&`
- Redundant heap: temp containers; use stack arrays / `std::array`
- Loop inefficiencies: repeated `.size()`/`.end()`; re-computed invariants; use `std::ranges`
- Missing `reserve`: `push_back` loops with known size
- String concat in loops: `+=` → `std::ostringstream` or `std::vector<std::string_view>` join
- Costly conversions: `std::string` from literals in hot paths
- `shared_ptr` overuse: use `unique_ptr` where it suffices
- Locking granularity: mutex across I/O; narrow critical sections

Static analysis only. Flag visible issues only.

---

## Step 10 — Compile findings

```pwsh
$sha = git rev-parse HEAD
Write-Host "`n📋 Pipeline for: $($sha.Substring(0, 8))"

$ciStatus = git log -1 --pretty=format:'%b' HEAD | Select-String -Pattern "Pipeline|CI|Status" -ErrorAction SilentlyContinue
if ($ciStatus) { Write-Host "$ciStatus" }
else { Write-Host "⚠️  Check GitLab CI at: https://gitlab.com/your-group/your-project/-/pipelines" }

Write-Host "`n💬 GitLab comments in MR (GitLab UI)"
```

```pwsh
if (Test-Path "CODEOWNERS") {
    Write-Host "`n👥 CODEOWNERS Impact:"
    $files | % {
        $owner = Select-String "^$_\s+" CODEOWNERS | % { $_.Line }
            if ($owner) { Write-Host "  $_ → $owner" }
    }
}
```

---

## Step 11 — Report findings

```
═══════════════════════════════════════════════════════════════════════════
                            MR AUDIT REPORT
═══════════════════════════════════════════════════════════════════════════

TOP 15 ISSUES (priority order)
───────────────────────────────────────────────────────────────────────────

1. [SECURITY] src/api.cpp:42 — Hardcoded auth
   → description
   → Fix: use env var

2. [BUG] src/handler.cpp:103 — Null pointer deref
   → description
   → Fix: add nil check

3. [TEST] src/new_func.cpp — No unit test
   → description
   → Fix: add GTest

... (up to 15 total)

───────────────────────────────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────────────────────────────

🔴 Blockers (fix before merge):
  - [SECURITY] xxx
  - [BUG] yyy

🟡 Should fix (before approval):
  - [TEST] zzz
  - [STYLE] aaa

🟢 Nice to have (follow-ups):
  - [MODERN] bbb
  - [DOCS] ccc

Modified files:      {count}
Issues found:        {top 15 count}
Est. fix time:       X-Y minutes

📋 Next: Run /main-rebase if behind, fix blockers, submit to /mr-create, request review
═══════════════════════════════════════════════════════════════════════════
```

**Issue types**: SECURITY, BUG, LINT, TEST, STYLE, MODERN, PERF, DOCS, TYPO
**Severities**: CRITICAL, HIGH, MEDIUM, LOW

---

## Configuration

**Sub-Agent A** (Code Quality & Standards):
- Naming: C++ CamelCase/snake_case, Python PEP 8
- Doxygen: C++ headers only, `@brief` required
- Test naming: `Method_Scenario_ExpectedBehavior`
- Outdated patterns

**Sub-Agent B** (Security & Bugs):
- Null deref, use-after-free, buffer overflow
- Hardcoded secrets
- Input sanitization at API boundaries
- Exception handling
- Python: subprocess/eval abuse, yaml.load, broad except

**Sub-Agent C** (Test & Docs):
- New functions/branches → unit tests?
- New public API → integration tests?
- Edge cases (empty, boundary, error)?
- Docstring completeness + accuracy

---

## Notes

- Consolidates `mr-ready` (scoring) + `mr-review` (detailed review) into one efficient skill
- No builds/tests run — pipeline checked via GitLab
- Parallel analysis for speed
- Top 15 issues with severity levels
- GitLab integration — comments, pipeline, CODEOWNERS aware
- Monorepo ready — respects path scoping
