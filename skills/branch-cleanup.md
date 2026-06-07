---
description: Delete all local branches except default, current, and user-specified.
argument-hint: "[extra-branches-to-keep,comma-separated]"
allowed-tools: Bash(git:*), Bash(pwsh:*)
model-recommended: Haiku
---

Notes:
- CLI options may be provided via environment vars for non-interactive use:
  - QUIET=1 to suppress verbose output
  - PROTECTED_PATTERNS_FILE=path to a file with regex patterns (one per line) that must not be deleted remotely

Behavior:
- Respects hardcoded protections (main, master, develop) and detects remote default branch.
- If a PROTECTED_PATTERNS_FILE exists, branches matching any regex there are refused for deletion.
- Exit codes: 0=success, 1=error, 2=user aborted/no-op


Delete local and remote branches safely. Preserves: default branch, current branch, and any user-specified extras.

Keep branches (optional, comma-separated): $ARGUMENTS

---

## Step 1 — Preflight

```pwsh
if (git status --porcelain) { throw 'Dirty tree. Commit or stash first.' }
```

## Step 2 — Resolve protected list

```pwsh
$curr = git rev-parse --abbrev-ref HEAD
$def = git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null
if ($LASTEXITCODE -eq 0) { $def = $def -replace '^origin/', '' }
else { $def = git remote show origin | Select-String 'HEAD branch' | % { ($_ -split ':')[-1].Trim() } }
if (-not $def) { $def = 'main' }

$extra = if ('$ARGUMENTS'.Trim()) { '$ARGUMENTS'.Split(',') | % { $_.Trim() } } else { @() }
$keep = @($curr, $def, 'main', 'master', 'develop') + $extra | ? { $_ } | Sort-Object -Unique
Write-Host "Protected: $($keep -join ', ')"
```

## Step 3 — List and confirm

```pwsh
$cands = git for-each-ref --format='%(refname:short)' refs/heads/ | ? { $_ -notin $keep }
if (-not $cands) { Write-Host "Nothing to delete."; return }
Write-Host "Will delete:`n$($cands | % { "  - $_" } | Out-String)"
```

**Ask user to confirm.**

## Step 4 — Delete safely

```pwsh
$del = @(); $fail = @()
foreach ($b in $cands) {
    git branch -d $b 2>$null
    if ($?) { $del += $b } else { $fail += $b }
}

if ($fail) {
    Write-Host "`nUnmerged (would lose commits):`n$($fail | % { "  - $_" } | Out-String)"
}
```

**Ask user which unmerged branches to force-delete (per-branch, not blanket).**

```pwsh
foreach ($b in $approved) { git branch -D $b; $del += $b }
```

## Step 5 — Prune stale refs

```pwsh
git remote prune origin
```

## Step 6 — Offer remote cleanup

```pwsh
$remCands = @()
foreach ($b in $del) {
    git ls-remote --exit-code --heads origin $b *>$null
    if ($?) { $remCands += $b }
}

if ($remCands) {
    Write-Host "`nStill on origin:`n$($remCands | % { "  - $_" } | Out-String)"
}
```

**Ask user to confirm per-branch remote deletion. Never delete default branch remotely.**

```pwsh
# Load protected patterns (regex), fallback to safe defaults
if ($env:PROTECTED_PATTERNS_FILE -and (Test-Path $env:PROTECTED_PATTERNS_FILE)) {
    $patterns = Get-Content $env:PROTECTED_PATTERNS_FILE | % { $_.Trim() } | ? { $_ }
} else {
    $patterns = @('^main$','^master$','^develop$','^release/','^prod')
}

foreach ($b in $approved) {
    # Safety: never delete the detected default branch
    if ($b -eq $def) { Write-Host "Skipping default branch: $b"; continue }

    # Refuse to delete if matches any protected regex
    $isProtected = $false
    foreach ($p in $patterns) { if ($b -match $p) { $isProtected = $true; break } }
    if ($isProtected) {
        Write-Host "Refusing to delete protected branch on origin: $b"
        continue
    }

    # Perform remote deletion with error handling
    try {
        Write-Host "Deleting remote: origin/$b"
        git push origin --delete $b
        if ($LASTEXITCODE -ne 0) { Write-Error "Failed to delete origin/$b"; exit 1 }
    } catch {
        Write-Error "Error deleting origin/$b: $_"
        exit 1
    }
}
```

## Step 7 — Report

Report: local deleted N, remote deleted R, kept M, skipped K (unmerged).
