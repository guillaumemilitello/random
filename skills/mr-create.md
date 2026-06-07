---
description: Open a GitLab MR for current branch (defaults + optional overrides).
argument-hint: "[target-branch]"
allowed-tools: Bash(git:*), Bash(pwsh:*)
model-recommended: Haiku
---

Options:
- GITLAB_REVIEWERS_FILE=path (one username per line) to override hardcoded reviewers


Open GitLab MR via API. Token required (terminal only, never ask chat). Hardcoded assignee + reviewers.

Override target branch (optional, defaults to auto-detect): $ARGUMENTS

---

## Step 1 — Preflight

```pwsh
if (-not $env:GITLAB_TOKEN) {
    Write-Host "Set GITLAB_TOKEN in this terminal: `$env:GITLAB_TOKEN = '<your-PAT-with-api-scope>'"
    throw 'Missing GITLAB_TOKEN.'
}
if (git status --porcelain) { throw 'Dirty tree. Commit or stash first.' }
```

## Step 2 — Branch and project info

```pwsh
$branch = git rev-parse --abbrev-ref HEAD
if ($branch -in @('main','master','develop')) { throw "Refusing to open MR from $branch." }

if ('$ARGUMENTS'.Trim()) { $target = '$ARGUMENTS'.Trim() }
else {
    $upstream = git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null
    if ($LASTEXITCODE -eq 0) { $target = $upstream -replace '^origin/', '' }
    else { $target = git remote show origin | Select-String 'HEAD branch' | % { ($_ -split ':')[-1].Trim() } }
}

$url = git remote get-url origin
if     ($url -match '^https?://([^/:]+)/(.+?)(?:\.git)?/?$') { $host = $Matches[1]; $proj = $Matches[2] }
elseif ($url -match '^ssh://[^@]+@([^/:]+)/(.+?)(?:\.git)?/?$') { $host = $Matches[1]; $proj = $Matches[2] }
elseif ($url -match '^[^@]+@([^:]+):(.+?)(?:\.git)?/?$') { $host = $Matches[1]; $proj = $Matches[2] }
else { throw "Cannot parse remote: $url" }

$projEnc = [uri]::EscapeDataString($proj)
$api = "https://$host/api/v4"
$hdr = @{ 'PRIVATE-TOKEN' = $env:GITLAB_TOKEN }
Write-Host "Branch: $branch → $target | Project: $host / $proj"
```

## Step 3 — Push branch

```pwsh
git fetch origin $target
$ahead = [int](git rev-list --count "origin/$target..HEAD")
if ($ahead -eq 0) { throw "No commits ahead of $target." }
git push --set-upstream origin $branch
```

## Step 4 — Skip if MR exists

```pwsh
$existing = Invoke-RestMethod -Method Get -Headers $hdr `
    -Uri "$api/projects/$projEnc/merge_requests?state=opened&source_branch=$([uri]::EscapeDataString($branch))" -ErrorAction SilentlyContinue
if ($existing -and $existing.Count -gt 0) {
    Write-Host "MR already open: $($existing[0].web_url)"
    return
}
```

## Step 5 — Summarize changes

```pwsh
Write-Host "`nCommits:"
git log --pretty=format:'- %s' "origin/$target..HEAD"
Write-Host "`nStats:"
git diff --stat "origin/$target...HEAD"
```

Generate:
- **Title**: imperative, ≤90 chars. Prefix `Draft: ` if WIP. Include ticket key prefix (e.g. `[MERRY-1234] `) from branch name.
- **Description** (markdown):
  - what + why + key changes, ≤10 lines, structured with bullet points

**Show to user. Wait for approval/edits.**

## Step 6 — Resolve user IDs

```pwsh
# Defaults (kept for speed)
$assigneeUsernames = @('guillaume.militello')
$defaultReviewers = @('francoisxavier.derue','sebastien.cadieux','florian.tami')

# Optional override via env file (one username per line)
if ($env:GITLAB_REVIEWERS_FILE -and (Test-Path $env:GITLAB_REVIEWERS_FILE)) {
    Write-Host "Using reviewers file: $env:GITLAB_REVIEWERS_FILE"
    $reviewerUsernames = Get-Content $env:GITLAB_REVIEWERS_FILE | % { $_.Trim() } | ? { $_ }
    if (-not $reviewerUsernames) {
        Write-Warning "Reviewers file empty — falling back to defaults."
        $reviewerUsernames = $defaultReviewers
    }
} else {
    $reviewerUsernames = $defaultReviewers
}

function Get-GitLabUserId($username) {
    try {
        $r = Invoke-RestMethod -Method Get -Headers $hdr -Uri "$api/users?username=$([uri]::EscapeDataString($username))" -ErrorAction Stop
    } catch {
        Write-Warning "Failed to look up user: $username"
        return $null
    }
    if (-not $r -or $r.Count -eq 0) { Write-Warning "User not found: $username"; return $null }
    return $r[0].id
}

# Resolve IDs, skip not-found users but warn
$assignee = $assigneeUsernames | ForEach-Object { Get-GitLabUserId $_ } | ? { $_ }
$reviewers = $reviewerUsernames | ForEach-Object { Get-GitLabUserId $_ } | ? { $_ }

if (-not $assignee) { Write-Warning "No assignee resolved; MR will be created without assignee." }
if (-not $reviewers) { Write-Warning "No reviewers resolved; MR will be created without reviewers." }
```

## Step 7 — Create MR

```pwsh
$body = @{
    source_branch = $branch
    target_branch = $target
    title = '<approved title>'
    description = '<approved description>'
    assignee_ids = @($assignee)
    reviewer_ids = $reviewers
    remove_source_branch = $true
    squash = $false
} | ConvertTo-Json -Depth 5

try {
    $resp = Invoke-RestMethod -Method Post -Headers $hdr -ContentType 'application/json' `
        -Uri "$api/projects/$projEnc/merge_requests" -Body $body
} catch {
    if ($_.Exception.Response.StatusCode.value__ -eq 409) {
        Write-Host "Conflict: MR already exists for $branch."
        return
    }
    throw
}
```

## Step 8 — Report

```
✅ MR created: $($resp.web_url)
```
