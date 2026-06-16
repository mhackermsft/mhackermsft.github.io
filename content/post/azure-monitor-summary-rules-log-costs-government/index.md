---
title: 'Taming High-Volume Log Costs: A Hands-On Guide to Azure Monitor Summary Rules'
date: 2026-06-16T18:07:20+00:00
author: Mike Hacker
tags:
- Data Platform
- How To
categories:
- Azure
- Monitoring
summary: 'A deep technical walkthrough of Azure Monitor Log Analytics summary rules: KQL aggregation, scheduled summarization, Bicep automation, and cost-aware retention patterns for government audit and compliance logging.'
draft: false
image_prompt: A bright government records archive where a wide river of paper log sheets pours onto a sorting table and is compressed into a small, neatly bundled stack of summary folders under warm golden light. The composition centers on the contrast between the overwhelming loose paperwork and the compact organized bundle, clearly suggesting high-volume audit logs being summarized to control storage costs. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Government IT teams manage a flood of logs. Container platforms, firewalls, identity systems, and resource diagnostics can generate millions of verbose records every day, and the bill for ingesting, querying, and retaining that data in a Log Analytics workspace can grow quickly. The instinct to keep everything for compliance collides with the reality of fixed annual budgets.

Azure Monitor summary rules offer a practical way to reduce that pressure. Instead of querying large raw tables every time you build a report or dashboard, you let Azure Monitor aggregate incoming data on a schedule, write the compact result into a dedicated Analytics table, and keep the raw logs in the table plan that fits your retention and query needs. This post walks through the portal experience, KQL patterns, Bicep automation, and retention trade-offs that matter for public sector workloads.

## What a summary rule actually does

A summary rule performs batch processing directly inside your Log Analytics workspace. According to the [Microsoft Learn summary rules documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/summary-rules), the rule aggregates chunks of incoming data based on a KQL query and a scheduled bin size, then re-ingests the summarized result into a custom table that uses the Analytics log plan.

Three characteristics make this useful for government workloads:

- **It reads from multiple table plans.** A summary rule can aggregate data from Analytics, Basic, or Auxiliary tables. That means raw, verbose logs can sit in a lower-cost Basic or Auxiliary table while the summarized output lands in an Analytics table for reporting.
- **It builds the schema from the query.** Azure Monitor creates the destination table schema from the query results. If the destination table already exists, Azure Monitor appends the columns needed for the updated query results.
- **It stamps provenance on every row.** Each summarized record includes standard fields such as `_RuleName`, `_RuleLastModifiedTime`, `_BinSize`, and `_BinStartTime`. Those columns help auditors understand which rule produced a summary and which interval it represents.

A workspace can have up to 100 active summary rules. Rules can write to separate destination tables or to the same destination table when the schemas are compatible.

## The cost math that justifies the effort

Microsoft Learn states that summary rules incur no extra rule charge. You pay for the query and for ingestion of the summarized results, and the query cost depends on the source table plan:

| Source table plan | Query cost | Summary ingestion cost |
|---|---|---|
| Analytics | No query charge | Analytics log ingestion |
| Basic or Auxiliary | Data scan | Analytics log ingestion |

The leverage comes from the ratio between raw volume and summarized volume. Microsoft guidance recommends aiming for a result volume of 0.01 percent or less of the source. If an hourly rule reduces a million verbose container records to a few hundred aggregated rows, the Analytics ingestion for the summary is small while the raw table remains available under the plan and retention settings you choose.

The [Azure Monitor Logs table plan model](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-platform-logs) is designed for this kind of tiering:

- **Analytics**: high-value data for continuous monitoring, real-time detection, alerting, dashboards, and full KQL.
- **Basic**: medium-touch data for troubleshooting and incident response, with full KQL on a single table and lookup to Analytics tables.
- **Auxiliary**: low-touch, verbose data for auditing and compliance, with lower-cost ingestion and query patterns suited to infrequent access.

All three plans support total retention of up to 12 years. Analytics tables can have interactive retention extended up to two years. Basic tables have 30 days of interactive query retention, while Auxiliary tables can be queried across their total retention period, with plan-specific considerations. For long-term retention beyond the interactive window, use search jobs as described in the [Log Analytics retention documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure).

The pattern is straightforward: keep raw evidence in the table plan that matches its access pattern, then use a summary rule to promote a compact, query-optimized slice into an Analytics table for dashboards, alerting, and recurring reporting.

## Building a rule in the portal

The portal experience lives under your Log Analytics workspace. In the left menu under **Settings**, choose **Summary rules**, then select **+ Create**. You provide a rule name, description, and destination table, then move to **Set rule logic**.

The rule logic step opens the Log Analytics query editor. Author and test your KQL there, then select **Apply** when the result shape is correct. Next, set **Run summary every**, which corresponds to the bin size, then review and create the rule.

One rule from the documentation is easy to miss: do not add a time filter to a summary rule query. The query already operates over the time range defined by the bin size. Adding a filter such as `where TimeGenerated > ago(1h)` combines that filter with the bin window and can leave only an overlapping subset of data.

## KQL patterns for summarization

The value of a summary rule depends on the query. These patterns are useful starting points.

**Collapse repetitive container logs.** The canonical example from Microsoft Learn turns thousands of similar container log lines into per-message counts:

```kusto
ContainerLogV2
| summarize Count = count() by
    Computer, ContainerName, PodName, PodNamespace,
    LogSource, LogLevel, Message = tostring(LogMessage.Message)
```

With an hourly bin, the destination table stores the count of each unique entry instead of every repeated raw line. The `ContainerLogV2` table supports the Basic plan, so raw container logs can use lower-cost retention while the summary table stays optimized for reporting.

**Build an audit roll-up for sign-in activity.** Compliance reviews often need counts, distinct users, and failure patterns by application rather than every authentication event:

```kusto
SigninLogs
| summarize
    SignIns = count(),
    DistinctUsers = dcount(UserPrincipalName),
    Failures = countif(ResultType != '0')
  by AppDisplayName, Location, ConditionalAccessStatus
```

In the current `SigninLogs` schema, `ResultType` is a string, and `0` indicates success. Using `'0'` avoids a type mismatch and keeps the failure count aligned with the documented table schema.

**Summarize firewall network rule activity.** The current `AZFWNetworkRule` table includes rule-hit fields such as action, protocol, destination port, rule, and rule collection. It does not expose a documented byte-count column, so use it for connection-count summaries rather than byte-volume rollups:

```kusto
AZFWNetworkRule
| summarize Connections = count()
  by Action, Protocol, DestinationPort, Rule, RuleCollection
```

If you need byte-level network reporting, choose a source table or diagnostic category that exposes documented byte fields, then verify the schema before putting the query into a scheduled rule.

A few constraints are important. Summary rules do not support cross-resource or cross-service queries such as `workspaces()`, `app()`, `resource()`, `ADX()`, or `ARG()`. They also do not support schema-reshaping plugins such as `bag_unpack`, `pivot`, or `narrow`, user-defined functions, or `union *` with `isfuzzy=true`. For Basic and Auxiliary source tables, KQL is limited to a single source table, although you can enrich with up to five Analytics tables by using the `lookup` operator.

Keep each result record under 1 MB, and test the query against the [Azure Monitor service limits for log queries](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits#logs) before scheduling it. If the query approaches limits, reduce the bin size or return fewer high-volume fields.

Summary rules process incoming data from the recent past only. They do not provide historical backfill. The maximum bin size is 1,440 minutes, which corresponds to 24 hours.

## Automating with infrastructure as code

Government environments need repeatable, reviewable deployments. Summary rules are Azure resources, so you can express them in Bicep alongside the rest of your monitoring stack. The current resource type for summary rules is `Microsoft.OperationalInsights/workspaces/summaryLogs@2025-07-01`:

```bicep
param workspaceName string

resource workspace 'Microsoft.OperationalInsights/workspaces@2025-07-01' existing = {
  name: workspaceName
}

resource containerSummary 'Microsoft.OperationalInsights/workspaces/summaryLogs@2025-07-01' = {
  parent: workspace
  name: 'ContainerHourlyRollup'
  properties: {
    ruleType: 'User'
    description: 'Hourly aggregation of verbose container logs'
    ruleDefinition: {
      query: 'ContainerLogV2 | summarize Count = count() by Computer, ContainerName, PodNamespace, LogLevel'
      binSize: 60
      timeSelector: 'TimeGenerated'
      destinationTable: 'ContainerRollup_CL'
    }
  }
}
```

The documented `binSize` values are 20, 30, 60, 120, 180, 360, 720, and 1,440 minutes.

If you prefer imperative tooling, the same operation works through the Logs management API with Azure CLI. The CLI sample builds a management-plane URL and uses `az rest` to send a PUT request to the `summarylogs` resource:

```bash
resourceGroupName='rg-monitoring'
workspaceName='law-gov-central'
ruleName='ContainerHourlyRollup'
apiVersion='2025-07-01'

subscriptionId=$(az account show --query id --output tsv)
path=/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName
provider=Microsoft.OperationalInsights/workspaces/$workspaceName
url=$path/providers/$provider/summarylogs/$ruleName?api-version=$apiVersion

az rest --method put --url $url --body @body.json
```

Save this as `body.json` in the same directory:

```json
{
  "properties": {
    "ruleType": "User",
    "description": "Hourly container roll-up",
    "ruleDefinition": {
      "query": "ContainerLogV2 | summarize Count = count() by PodNamespace",
      "binSize": 60,
      "timeSelector": "TimeGenerated",
      "destinationTable": "ContainerRollup_CL"
    }
  }
}
```

Creating or updating a rule requires `Microsoft.OperationalInsights/workspaces/summarylogs/write` permission. Microsoft Learn lists this permission as included in the built-in **Log Analytics Contributor** role.

## Querying the summarized data for long-term audit

The payoff arrives at report time. Instead of scanning months of raw records, your compliance and trend queries hit the compact destination table and use the provenance columns:

```kusto
ContainerRollup_CL
| where _BinStartTime between (startofmonth(ago(365d)) .. now())
| summarize MonthlyEvents = sum(Count) by bin(_BinStartTime, 30d), PodNamespace
| order by _BinStartTime asc
```

Because `_BinStartTime` and `_BinSize` are stamped on every row, an auditor can see which interval each figure represents. `_RuleName` and `_RuleLastModifiedTime` help show which rule generated the row and when that rule last changed.

To move summarized data outside the workspace, define a Log Analytics data export rule on the destination table. Data export supports Analytics and Basic table plans, while Auxiliary tables are not supported for data export. Because summary results land in an Analytics table, the summarized destination table can be exported to Azure Storage or Event Hubs for immutable archive patterns or downstream systems.

## Why This Matters for Government

State and local agencies face a structural tension: audit and records-retention obligations require evidence to be kept for years, while modernization budgets do not expand automatically with log volume. Summary rules help reduce that tension without forcing teams to discard raw evidence. You keep the raw logs under a retention policy that matches the agency requirement, then spend Analytics-grade query and reporting effort on the smaller summaries that teams use every day.

There is also a governance benefit. Microsoft Learn calls out data privacy as a summary-rule use case: a rule can remove or obfuscate sensitive fields during aggregation, so the shareable summary table does not contain all of the personally identifiable detail in the raw table. You can restrict access to raw tables and grant broader read access to summaries, which supports least-privilege operations.

One caveat matters for public sector architects. Microsoft Learn lists summary rules as available only in the public cloud. Agencies operating in Azure US Government should validate feature availability for their specific tenant and region before designing a production dependency on summary rules. If your observability estate spans commercial Azure and Azure Government, document the difference and avoid assuming feature parity.

Start small. Pick one noisy table, draft a `summarize` query that reduces the result to a fraction of the source volume, validate it in the Log Analytics editor, and schedule it on an hourly bin. Move the raw table to the plan that matches your retention and access needs, point dashboards at the summary table, and measure the impact in Cost Management while keeping the audit trail intact.

### References

- [Aggregate data in a Log Analytics workspace by using summary rules - Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/summary-rules)
- [Azure Monitor Logs overview and table plans - Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-platform-logs)
- [Manage data retention in a Log Analytics workspace - Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure)
- [Microsoft.OperationalInsights workspaces/summaryLogs template reference - Microsoft Learn](https://learn.microsoft.com/en-us/azure/templates/microsoft.operationalinsights/workspaces/summarylogs)
- [Summary Logs create or update REST API - Microsoft Learn](https://learn.microsoft.com/en-us/rest/api/loganalytics/summary-logs/create-or-update?view=rest-loganalytics-2025-07-01)
- [Log Analytics workspace data export in Azure Monitor - Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-data-export)
- [Azure Monitor pricing - Microsoft Azure](https://azure.microsoft.com/en-us/pricing/details/monitor/)
