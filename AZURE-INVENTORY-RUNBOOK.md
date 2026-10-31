# Azure tenant-wide inventory collection runbook

Use this client-run process to inventory the intended Azure tenant and all **35
expected subscriptions**. Run PowerShell 7 (`pwsh`) on a client-controlled
workstation using the client's interactive Azure sign-in. Collection reads Azure
configuration; module installation writes locally. Creating Cost Management
exports is a separate, explicitly approved Azure change.

**No secrets should be shared.** Do not send passwords, client secrets, tokens,
certificates, storage keys, SAS URLs, or authentication caches to anyone or commit
them here. The inventory recipient does not need Azure credentials.

## 1. Prepare and verify all 35 subscriptions

Obtain an authoritative, client-approved list of all 35 subscription IDs from the
tenant/subscription owner, independently of the collecting identity's visibility.
Save it locally as `expected-subscriptions.csv` with one `SubscriptionId` column.
Replace every `<...>` placeholder below.

Use approved module versions/sources and an encrypted, client-managed output
directory outside Git checkouts and automatically shared folders. Confirm Reader
access across each subscription, not just selected resource groups. Confirm
Defender for Cloud (Security Center) and cost-data read permissions separately.
Resource Reader does not establish cost visibility for every billing agreement:
check Cost Management read access (for example, Cost Management Reader at the
appropriate scope) and applicable billing cost-visibility policies.

```powershell
# Run in PowerShell 7; install client-approved versions where required.
$ErrorActionPreference = 'Stop'
$tenantId = '<TENANT-GUID>'
$out = '<ABSOLUTE-CLIENT-SECURE-OUTPUT-DIRECTORY>'
$expectedFile = '<ABSOLUTE-PATH-TO-expected-subscriptions.csv>'
New-Item -ItemType Directory -Path $out -Force | Out-Null

Install-Module -Name Az.Accounts -Scope CurrentUser -Repository PSGallery
Install-Module -Name AzureResourceInventory -Scope CurrentUser -Repository PSGallery
Install-Module -Name Az.CostManagement -Scope CurrentUser -Repository PSGallery
Import-Module Az.Accounts
Import-Module AzureResourceInventory
Import-Module Az.CostManagement

Connect-AzAccount -Tenant $tenantId
if ((Get-AzContext).Tenant.Id -ne $tenantId) {
    throw 'Wrong tenant: stop and correct the sign-in.'
}
$subs = @(Get-AzSubscription -TenantId $tenantId)
$subs | Select-Object Name, Id, TenantId, State |
    Export-Csv (Join-Path $out 'powershell-subscriptions.csv') -NoTypeInformation

$expected = @(Import-Csv $expectedFile |
    ForEach-Object { ([guid]$_.SubscriptionId).ToString() } | Sort-Object -Unique)
$actual = @($subs |
    ForEach-Object { ([guid]$_.Id).ToString() } | Sort-Object -Unique)
if ($expected.Count -ne 35) {
    throw 'The approved manifest must contain exactly 35 unique subscription IDs.'
}
$difference = @(Compare-Object -ReferenceObject $expected -DifferenceObject $actual)
if ($difference.Count -gt 0) {
    $difference | Format-Table
    throw 'Resolve missing or unexpected subscription IDs before collection.'
}
if (@($subs | Where-Object { $_.State -ne 'Enabled' }).Count -gt 0) {
    $subs | Where-Object { $_.State -ne 'Enabled' } | Format-Table
    throw 'Review non-enabled subscriptions with the owner before collection.'
}
Get-Module Az.Accounts, AzureResourceInventory, Az.CostManagement |
    Select-Object Name, Version |
    Export-Csv (Join-Path $out 'module-versions.csv') -NoTypeInformation
```

Inspect the account, tenant, names, IDs, and states with the client. A visible
count of 35 is insufficient: IDs must match. After new access grants, sign out and
back in to refresh subscription visibility, then repeat. Record unavailable
subscriptions explicitly; do not silently reduce the target.

## 2. Run Microsoft Azure Resource Inventory (ARI)

`-IncludeCosts` requires **Az.CostManagement** and cost permissions; installing
the module does not grant access. If a switch is unavailable, inspect
`Get-Help Invoke-ARI -Full` and resolve version issues rather than omit data.

```powershell
$startedUtc = (Get-Date).ToUniversalTime().ToString('o')
Invoke-ARI -TenantID $tenantId -IncludeTags -SecurityCenter -IncludeCosts `
    -ReportDir $out -ReportName 'TenantInventory' -NoAutoUpdate
$finishedUtc = (Get-Date).ToUniversalTime().ToString('o')
"Started UTC: $startedUtc`nFinished UTC: $finishedUtc" |
    Set-Content (Join-Path $out 'collection-window.txt')
Get-ChildItem -Path $out
```

Do not pass subscription, resource-group, management-group, or tag filters for
this tenant-wide run. Tenant selection still collects only accessible resources.
`-NoAutoUpdate` avoids ARI self-updates during collection; review approved updates
separately. Review warnings and failures, not just output-file existence. Mark
missing cost/security data as missing, not zero or compliant; resolve and rerun
affected collection as needed.

Keep the Excel workbook and generated Draw.io-compatible XML diagrams together.
Network diagrams are generated unless skipped; add `-DiagramFullEnvironment` if
the client needs the broader network-component view. Open diagrams in an approved
local Draw.io/diagrams.net application, not an unapproved website. Inspect workbook
subscription coverage and diagrams; neither proves every resource type was captured.

## 3. Export an independent Resource Graph inventory

In the Azure portal, verify the same directory/tenant and open **Resource Graph
Explorer**. Select **Directory** to set scope to the intended directory and all
35 subscriptions. Remove narrower scope selections and verify the selected
subscription IDs. Directory scope never bypasses RBAC.

Run each query, review **Messages** for errors/counts, and select **Download as
CSV**. Retain the query text, scope, UTC collection time, and CSVs in the secure
output directory. Do not add `take`, `limit`, or `top` to inventory queries.

### Subscription manifest: graph-subscriptions.csv

```kusto
ResourceContainers
| where type =~ 'microsoft.resources/subscriptions'
| project subscriptionId, subscriptionName = name,
          state = tostring(properties.state), tenantId
| order by subscriptionId asc
```

Compare these IDs with the approved manifest and `powershell-subscriptions.csv`.
Require the exact same 35 IDs, not just 35 rows. Investigate every discrepancy.

### Resource inventory: resources.csv

```kusto
Resources
| project id, subscriptionId, resourceGroup, name, type, location,
          kind, managedBy, tags = tostring(tags)
| order by subscriptionId asc, id asc
```

This covers the management-plane `Resources` table, not data-plane contents,
backups, Entra ID objects, or all other Resource Graph tables. Tags can be sensitive.

**Explorer CSV exports are limited to 55,000 records.** If the inventory exceeds
the limit, export each subscription separately. Insert the following immediately
after `Resources` and name files `resources-<SUBSCRIPTION-ID>.csv`:

```kusto
| where subscriptionId == '<SUBSCRIPTION-GUID>'
```

If one subscription still exceeds 55,000, split it into disjoint resource-type or
resource-group partitions, including empty resource-group values. Record all
filters and expected counts; keep each export below the limit and verify combined
IDs are unique. Never accept a truncated CSV as complete.

### Per-subscription reconciliation: graph-resource-counts.csv

Keep the same full directory/subscription scope:

```kusto
ResourceContainers
| where type =~ 'microsoft.resources/subscriptions'
| project subscriptionId, subscriptionName = name
| join kind=leftouter (
    Resources
    | summarize resourceCount = count() by subscriptionId
) on subscriptionId
| project subscriptionId, subscriptionName,
          resourceCount = coalesce(resourceCount, tolong(0))
| order by subscriptionId asc
```

The left join retains visible subscriptions with zero resources. Match these
35 rows to the manifest and compare each count with exported CSV rows grouped by
subscription. For an unsplit inventory:

```powershell
Import-Csv (Join-Path $out 'resources.csv') |
    Group-Object subscriptionId |
    Select-Object Name, Count |
    Sort-Object Name
```

For split exports, combine only recorded inventory partitions before grouping,
not manifest/count CSVs. Check duplicate `id` values; treat subscriptions without
inventory rows as zero. Require per-subscription counts and the total to match
the count query. Reconcile ARI subscription coverage too, but do not assume ARI
worksheet totals equal `Resources` counts: ARI transforms data and has
type-specific coverage.

## 4. Collect separate costs and close out

ARI cost summaries do **not** replace separate Azure Cost Management exports.
Have the client billing owner use **Cost Management > Exports** at supported
billing/subscription scopes for the agreed period and dataset (actual cost and,
where needed, amortized cost). Confirm all 35 subscriptions are covered, accounting
for separate billing scopes. Record currency, dates, dataset, scope, and export
completion; retain delivered files and manifests. Creating exports/storage
requires explicit approval and additional permissions, not just inventory Reader.

Resource Graph is eventually consistent, supports specific resource types and
properties, and returns only RBAC-visible data. Partial results may not warn
about inaccessible subscriptions/resources. Tenant membership or an Entra
administrator role alone does not grant Azure resource read access. Matching
counts proves export consistency for the selected visible scope, not universal
completeness. Recheck after recent changes/access grants and record timing,
unsupported types, inaccessible resources, throttling/errors, and unresolved
count differences as coverage limitations.

Retain approved/observed manifests, workbook, diagrams, inventory/count CSVs,
cost exports, queries, module versions, timestamps, and an exceptions record in
client-controlled encrypted storage with least-privilege access and approved
retention. Review/redact identifiers, tags, costs, IPs, and topology before
authorized transfer. Use an approved secure channel, not a public repository or
unapproved chat/email attachment. Keep secrets out of artifacts; never share
authentication caches or unreviewed debug logs. Use `Disconnect-AzAccount` when
finished on a shared workstation.

## Official references

- [Microsoft ARI README and parameters](https://github.com/microsoft/ARI/blob/main/README.md)
- [Microsoft ARI basic usage guide](https://github.com/microsoft/ARI/blob/main/docs/user-guide/basic-usage.md)
- [Resource Graph overview, permissions, and consistency](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)
- [Resource Graph Explorer quickstart, scope, and CSV limit](https://learn.microsoft.com/en-us/azure/governance/resource-graph/first-query-portal)
- [Create and manage Cost Management exports](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data)
