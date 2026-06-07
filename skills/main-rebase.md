---
description: Rebase current branch onto base (auto-detected) with safe fallback.
argument-hint: "[base-branch]"
allowed-tools: Bash(git:*), Bash(pwsh:*)
model-recommended: Haiku
---


Rebase feature branch onto base branch, preserving commits and resolving conflicts interactively.

Override base: $ARGUMENTS | Auto-detect: upstream → origin HEAD → error

---

## Step 0 — Preflight

```pwsh
if (git status --porcelain) { throw 'Dirty tree. Stash or commit first.' }

$gitDir = git rev-parse --git-dir
foreach ($s in 'rebase-merge','rebase-apply','MERGE_HEAD','CHERRY_PICK_HEAD') {
    if (Test-Path (Join-Path $gitDir $s)) { throw "$s in progress. Resolve first." }
}
```

## Step 1 — Resolve base branch

```pwsh
if ('$ARGUMENTS'.Trim()) { $baseBranch = '$ARGUMENTS'.Trim() }
else {
    $baseBranch = git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null
    if ($LASTEXITCODE -eq 0) { $baseBranch = $baseBranch -replace '^origin/', '' }
    else {
        $baseBranch = git remote show origin | Select-String 'HEAD branch' | % { ($_ -split ':')[-1].Trim() }
        if (-not $baseBranch) { throw 'Cannot auto-detect base. Provide as argument.' }
    }
}
$orig = git rev-parse --abbrev-ref HEAD
Write-Host "Current: $orig → Base: $baseBranch"
```

**Confirm base branch before continuing.**

If same branch, fast-forward only:
```pwsh
git pull --ff-only origin $baseBranch
```

## Step 2 — Fetch, prune, and check if rebasing needed

```pwsh
git fetch --prune origin $baseBranch

$behind = [int](git rev-list --count "HEAD..origin/$baseBranch")
$ahead  = [int](git rev-list --count "origin/$baseBranch..HEAD")
Write-Host "Ahead: $ahead | Behind: $behind"

if ($behind -eq 0) {
    Write-Host "Up to date. Nothing to rebase."
    return
}
```

## Step 3 — Standard rebase

Try first. On conflicts: show files, ask user to resolve each, then `git add` + `git rebase --continue`. To abort: `git rebase --abort`.

```pwsh
git rebase --update-refs "origin/$baseBranch"
```

`--update-refs` auto-preserves stacked branches.

---

## Step 4 — Cherry-pick fallback (if Step 3 fails)

Use when rebase is unviable (history rewritten, entangled merges).

```pwsh
git rebase --abort 2>$null

$tmp = "tmp/$orig"
if (git show-ref --verify --quiet "refs/heads/$tmp") { throw "$tmp exists. Delete first." }
git branch -m $orig $tmp
git checkout -b $orig "origin/$baseBranch"

$commits = git log --reverse --pretty=format:'%H' "origin/$baseBranch..$tmp"
foreach ($c in $commits -split "`n" | ? { $_ }) {
    Write-Host "Cherry-pick: $($c.Substring(0,7))"
    git cherry-pick $c
    if ($LASTEXITCODE -ne 0) {
        Write-Host "CONFLICT: resolve then 'git cherry-pick --continue' or --skip or --abort"
        break
    }
}
```

On success: `git branch -D $tmp`

On abort: restore original state:
```pwsh
git checkout $tmp
git branch -D $orig
git branch -m $tmp $orig
```

## Step 5 — Validate build

Read `docs/build_test_run.md` and run exact build command. Never improvise.

## Step 6 — Push (user approval only)

Force-with-lease (safer than --force):
```pwsh
git push --force-with-lease origin $orig
```

**Never force-push without explicit user confirmation.**
