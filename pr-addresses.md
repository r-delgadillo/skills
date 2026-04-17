---
name: pr-address
description: Address reviewer feedback on an Azure DevOps PR by fixing code. Fetches PR comment threads, presents them, and makes code changes to resolve reviewer concerns. Use when the user wants to fix PR comments, address PR feedback, or resolve reviewer requests on their own PR.
---

Address reviewer feedback on your Azure DevOps PR by making code fixes.

## When to Use

- User is the PR author and wants to fix reviewer comments
- User says "address PR comments", "fix PR feedback", "resolve PR comments", etc.
- **NOT** for reviewing someone else's code — use the `pr-review` skill for that
- **NOT** for creating ADO work items — use the `pr-work-items` skill for that

## Step-by-Step Process

### 1. Parse PR Input

Extract the PR ID from whatever the user provides:
- Full URL: `https://msazure.visualstudio.com/One/_git/Azure-IoT-Platform-DeviceRegistry/pullrequest/12345` → `12345`
- Short URL: `https://dev.azure.com/msazure/One/_git/Azure-IoT-Platform-DeviceRegistry/pullrequest/12345` → `12345`
- Just the number: `12345`

### 2. Authenticate and Fetch PR Comment Threads

```powershell
$token = az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv 2>$null
$headers = @{ "Authorization" = "Bearer $token"; "Content-Type" = "application/json" }

$org = "msazure"
$project = "One"
$repo = "Azure-IoT-Platform-DeviceRegistry"
$prId = <PR_ID>

# Fetch comment threads
$threadsUrl = "https://$org.visualstudio.com/$project/_apis/git/repositories/$repo/pullRequests/$prId/threads?api-version=7.0"
$threads = Invoke-RestMethod -Uri $threadsUrl -Headers $headers -Method Get
```

### 3. Categorize and Filter Comments

**Human comments** — `commentType -ne 'system'` and author is NOT `"GitOps (Git LowPriv)"`
**Bot comments** — `commentType -eq 'system'` or author is `"GitOps (Git LowPriv)"` (PR Assistant)

**Thread statuses:**
| Status | Meaning |
|--------|---------|
| `active` | Not yet addressed |
| `pending` | Waiting for response |
| `fixed` | Already resolved |
| `closed` | Closed by reviewer |
| `byDesign` / `wontFix` | Won't fix |
| `null` / empty | No status (common for bot threads) |

**Default filter:** Show threads that are `active` or `pending` with human comments. Report how many were filtered (e.g., "filtered 2 resolved, 4 bot").

```powershell
foreach ($thread in $threads.value) {
    $humanComments = $thread.comments | Where-Object {
        $_.commentType -ne 'system' -and
        $_.author.displayName -ne 'GitOps (Git LowPriv)'
    }
    $isHuman = $humanComments.Count -gt 0
    $isActive = $thread.status -in @('active', 'pending', $null)
}
```

Strip HTML from bot comments: `-replace '<[^>]+>', ' '`

### 4. Present Comments for Triage

Show actionable comments in a numbered table:

```
PR #12345: "Add query validation middleware"

Found 3 active reviewer comments (filtered 2 resolved, 4 bot):

| # | File                        | Line | Reviewer | Comment                        | Status |
|---|-----------------------------|------|----------|--------------------------------|--------|
| 1 | /src/Middleware/Validator.cs | 42   | Alice    | Should this validate nulls...  | active |
| 2 | /src/Models/Query.cs        | 18   | Bob      | Consider using Guid? instead.. | active |
| 3 | /tests/ValidatorTests.cs    | 95   | Alice    | Missing edge case for empty... | active |
```

Show the full comment text for each thread (including any replies).

**Ask the user** how to proceed:
- Walk through one by one
- Fix specific comments by number (e.g., "fix 1 and 3, skip 2")
- Fix all

For any comment that's too large or complex to fix now, suggest using the `pr-work-items` skill to file a bug.

### 5. Fix Comments with Code Changes

For each comment the user wants to fix:

1. **Read the file** at the referenced line and surrounding context
2. **Read the full thread** including all replies for additional context
3. **Investigate the codebase** — grep for usages, check related patterns, understand the impact of changes
4. **Make the code fix** — edit the file to address the reviewer's feedback
5. **Verify** — run relevant build/test:
   - Identify the correct project from the file path
   - Build: `dotnet build <project>.csproj`
   - Test: `dotnet test <test-project>.csproj --filter "FullyQualifiedName~<relevant>"`
6. **Report** what was changed and the test result

### 6. Final Summary

After addressing all selected comments:

```
## PR Feedback Summary — PR #12345

| # | Comment                         | Action  | Result                 |
|---|---------------------------------|---------|------------------------|
| 1 | Null validation in Validator.cs | Fixed   | Updated lines 42-45    |
| 2 | Use Guid? in Query.cs           | Skipped | Too large, file a bug  |
| 3 | Missing edge case test          | Fixed   | Added test case         |

Fixed: 2 | Skipped: 1
```

## Important Notes

- **This skill modifies code in the local workspace** — the user is responsible for committing and pushing changes
- **Investigate before fixing** — grep for usages, check patterns, understand blast radius before making changes
- **Always verify with build/test** — don't leave the code in a broken state
- **If a fix is too large or risky** — suggest the user file it as a bug via `pr-work-items` instead of making a potentially incomplete fix
- **Strip HTML from bot comments** — use `-replace '<[^>]+>', ' '` to get plain text
- **Respect the user's choices** — if they want to include bot comments, skip certain items, or change approach, adapt accordingly
