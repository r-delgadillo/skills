---
name: pr-work-items
description: Create ADO bugs or tasks from Azure DevOps PR review comments. Fetches PR threads, lets the user pick which comments to track, and creates work items under a parent PBI. Use when the user wants to file bugs from PR feedback, create tasks from PR comments, or track PR review items in ADO.
---

Create ADO bugs or tasks from Azure DevOps pull request review comments.

## When to Use

- User wants to create work items (bugs/tasks) from PR comments
- User says "create bugs from PR", "file work items from PR", "track PR comments as bugs", etc.
- User has a PR URL and a parent PBI/Feature to file under
- **NOT** for reviewing code or fixing comments — use the `pr-review` skill for that

## Step-by-Step Process

### 1. Parse Inputs

**PR ID** — Extract from URL or number:
- `https://msazure.visualstudio.com/One/_git/Azure-IoT-Platform-DeviceRegistry/pullrequest/12345` → `12345`
- `https://dev.azure.com/msazure/One/_git/Azure-IoT-Platform-DeviceRegistry/pullrequest/12345` → `12345`
- Just the number: `12345`

**Parent work item ID** — Extract from sprint URL or number:
- `https://dev.azure.com/msazure/One/_sprints/...?workitem=37403622` → `37403622`
- `https://dev.azure.com/msazure/One/_workitems/edit/37403622` → `37403622`
- Just the number: `37403622`

Ask the user for either if not provided.

### 2. Authenticate and Fetch Data

```powershell
$token = az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv 2>$null
$headers = @{ "Authorization" = "Bearer $token"; "Content-Type" = "application/json" }

$org = "msazure"
$project = "One"
$repo = "Azure-IoT-Platform-DeviceRegistry"
$prId = <PR_ID>

# Fetch PR comment threads
$threadsUrl = "https://$org.visualstudio.com/$project/_apis/git/repositories/$repo/pullRequests/$prId/threads?api-version=7.0"
$threads = Invoke-RestMethod -Uri $threadsUrl -Headers $headers -Method Get

# Fetch parent work item for area/iteration
$headersGet = @{ "Authorization" = "Bearer $token"; "Content-Type" = "application/json" }
$parentUrl = "https://dev.azure.com/msazure/One/_apis/wit/workitems/<PARENT_ID>?api-version=7.0"
$parent = Invoke-RestMethod -Uri $parentUrl -Headers $headersGet -Method Get

$areaPath = $parent.fields.'System.AreaPath'
$iterationPath = $parent.fields.'System.IterationPath'
```

### 3. Categorize and Present Comments

**Filter comments into categories:**

**Human comments** — `commentType -ne 'system'` and author is NOT `"GitOps (Git LowPriv)"`
**Bot comments** — `commentType -eq 'system'` or author is `"GitOps (Git LowPriv)"`

**Thread statuses:** `active`, `pending`, `fixed`, `closed`, `byDesign`, `wontFix`, `null`

**Default view:** Show all threads with human comments regardless of status (the user may want to file bugs even for "fixed" comments that need follow-up). Report counts per status.

Strip HTML from bot comments: `-replace '<[^>]+>', ' '`

**Present a numbered table:**

```
PR #12345 — 8 comment threads found
  Human: 3 active, 2 fixed | Bot: 3

| # | File                        | Line | Reviewer | Comment                       | Status |
|---|-----------------------------|------|----------|-------------------------------|--------|
| 1 | /src/Middleware/Validator.cs | 42   | Alice    | Should this validate nulls... | active |
| 2 | /src/Models/Query.cs        | 18   | Bob      | Consider using Guid? instead  | active |
| 3 | /components/context/Ctx.cs  | 33   | Alice    | Case sensitivity issue        | fixed  |
```

Show the full comment text for each thread so the user can decide.

**Ask the user** which comments to create work items for. They may say:
- "All of them"
- "1 and 3"
- "Only the active ones"
- "Include bot comments too"

### 4. Create Work Items

For each selected comment, create a Bug (default) or Task:

```powershell
$headers = @{ "Authorization" = "Bearer $token"; "Content-Type" = "application/json-patch+json" }
$type = "Bug"  # or "Task" — ask the user if they have a preference, default to Bug
$url = "https://dev.azure.com/msazure/One/_apis/wit/workitems/`$$($type)?api-version=7.0"

$body = @(
    @{ op = "add"; path = "/fields/System.Title"; value = $title }
    @{ op = "add"; path = "/fields/System.Description"; value = $description }
    @{ op = "add"; path = "/fields/System.AreaPath"; value = $areaPath }
    @{ op = "add"; path = "/fields/System.IterationPath"; value = $iterationPath }
    @{ op = "add"; path = "/relations/-"; value = @{
        rel = "System.LinkTypes.Hierarchy-Reverse"
        url = "https://dev.azure.com/msazure/One/_apis/wit/workitems/$parentId"
    }}
) | ConvertTo-Json -Depth 5

$result = Invoke-RestMethod -Uri $url -Headers $headers -Method Post -Body $body
```

### 5. Title and Description Format

**Title:** `[<Component>] <Concise problem statement>`

Derive `[Component]` from the file path:

| Path prefix | Component tag |
|-------------|---------------|
| `/src/` | `[DeviceRP]` |
| `/components/context/` | `[Context]` |
| `/components/event-hub/` | `[EventHub]` |
| `/components/query-library/` | `[QueryLibrary]` |
| `/components/resource-storage/` | `[ResourceStorage]` |
| `/services/control-plane/` | `[ControlPlane]` |
| `/services/data-plane/` | `[DataPlane]` |
| `/services/orchestration/` | `[Orchestration]` |
| `/services/namespaces-registry-rp/` | `[Namespaces]` |
| `/services/management-action/` | `[ManagementAction]` |
| `/service-asset-registry-rp/` | `[AssetRegistry]` |
| `/service-schema-registry-rp/` | `[SchemaRegistry]` |
| `/services/resource-operations/` | `[ResourceOperations]` |
| `/services/billing/` | `[Billing]` |
| `/services/mics/` | `[MICS]` |
| `/tools/` | `[Tools]` |
| `/.github/` | `[Repo]` |
| `/docs/` | `[Docs]` |

**Description** (HTML format for ADO rich text):

```html
<h3>Source</h3>
<p>PR #<PR_ID> review comment by <Reviewer Name></p>
<h3>File</h3>
<p><code>/path/to/file.cs</code> (line <N>)</p>
<h3>Issue</h3>
<p>Reviewer comment: <em>"exact comment text quoted here"</em></p>
<h3>Action Required</h3>
<ul>
<li>Specific action step derived from the comment</li>
<li>Additional investigation or fix steps</li>
<li>Add unit tests if applicable</li>
</ul>
```

**Cross-reference related bugs** — if thread replies mention existing work item URLs (e.g., `dev.azure.com/.../workitems/edit/12345`), include a "Related" section in the description.

### 6. Report Results

After creating all work items, present a summary:

```
## Work Items Created — PR #12345

| Bug ID  | Title                                      | File                        |
|---------|--------------------------------------------|------------------------------|
| 37551492| [EventHub] Verify underscore in error code | /components/event-hub/...    |
| 37551493| [Context] Align middleware naming           | /components/context/...      |

Created 2 bugs under PBI #37403622
```

## Important Notes

- **Always ask the user to confirm** which comments to create work items for — not every comment needs a work item
- **Default to Bug type** unless the user specifies Task
- **Inherit area path and iteration** from the parent work item automatically
- **Quote the reviewer's exact words** in the description
- **Generate actionable titles** — not just the comment text, but a clear problem statement
- **Generate specific action items** — analyze the comment to produce concrete steps, not generic "investigate" items
