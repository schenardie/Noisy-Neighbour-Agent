# Noisy Neighbour Agent

A **Microsoft Security Copilot** plugin that analyzes Microsoft Intune audit logs for the last 6 months to detect **noisy neighbour automation patterns** — accounts or automations that repeatedly update the same Intune settings with the same values at consistent intervals (hourly, daily, or weekly).

Microsoft treats these repetitive, idempotent changes as "noisy neighbours" in multi-tenant environments because they consume disproportionate platform resources and can degrade performance for all tenants.

---

## What This Agent Does

1. **Retrieves** Microsoft Intune audit events from the Microsoft Graph API for the last 6 months
2. **Analyzes** the audit logs to identify actors (users or service principals) making repeated identical changes at consistent intervals
3. **Reports** which actions are happening, which accounts are responsible, and provides actionable remediation steps

## Files

| File | Description |
|------|-------------|
| `manifest.yaml` | Security Copilot plugin manifest — defines the API and GPT skills |
| `openapi.yaml` | OpenAPI 3.0 specification for the Microsoft Graph Intune audit events API |
| `NoisyNeighbour_Promptbook.yaml` | Step-by-step Security Copilot promptbook for guided analysis |

---

## Prerequisites

- **Microsoft Security Copilot** license and access
- **Microsoft Intune** tenant with audit logging enabled
- An Azure AD account with one of the following delegated permissions:
  - `DeviceManagementApps.Read.All`
  - `DeviceManagementConfiguration.Read.All`
  - `DeviceManagementManagedDevices.Read.All`
  - `DeviceManagementRBAC.Read.All`

---

## Installation

### Option 1: Upload Plugin to Security Copilot

1. Navigate to [Microsoft Security Copilot](https://securitycopilot.microsoft.com)
2. Click **Sources** in the left navigation
3. Select **Custom** under the Plugin section
4. Click **Upload plugin**
5. Upload the `manifest.yaml` file
6. When prompted, authenticate with your Azure AD account that has the required Intune read permissions
7. The plugin will appear as **Noisy Neighbour Agent** in your plugin list

### Option 2: Use the Promptbook

1. After installing the plugin (Step 1 above), navigate to **Promptbooks** in Security Copilot
2. Click **New promptbook** → **Upload**
3. Upload the `NoisyNeighbour_Promptbook.yaml` file
4. Run the promptbook to perform a guided 8-step analysis

---

## Available Skills

### API Skills (from `openapi.yaml`)

#### `GetIntuneAuditEvents`
Retrieves Intune audit events from Microsoft Graph API.

**Key parameters:**
- `$filter` — OData filter (e.g., `activityDateTime ge 2024-10-28T00:00:00Z`)
- `$select` — Fields to retrieve
- `$orderby` — Sort order (use `activityDateTime asc` for interval analysis)
- `$top` — Events per page (max 999)
- `$skipToken` — Pagination token for multi-page results

### GPT Skills (from `manifest.yaml`)

#### `AnalyzeIntuneNoisyNeighbours`
Analyzes a batch of Intune audit log JSON for noisy neighbour patterns.

**Input:** `auditLogData` — JSON array of audit events from Graph API

**Output:** Structured analysis report with:
- Detected patterns grouped by actor + activity type
- Interval classification (hourly/daily/weekly)
- Confirmed idempotent value overwrites
- Impact assessment (Low/Medium/High)
- Remediation recommendations

#### `GetNoisyNeighbourReport`
End-to-end orchestrated report — retrieves and analyzes 6 months of audit logs in one command.

**Input:** `lookbackDays` (optional, default: 180)

**Example prompts:**
- *"Get the noisy neighbour report for Intune"*
- *"Find noisy neighbours in Intune audit logs"*
- *"Who is making repetitive changes in Intune?"*
- *"What automations are repeatedly changing Intune settings?"*

---

## Example Output

When a noisy neighbour pattern is detected, the agent reports findings like:

```
#### Pattern 1: SyncAutomation@contoso.com — Patch DeviceConfiguration

| Field          | Value                                        |
|----------------|----------------------------------------------|
| Actor          | SyncAutomation@contoso.com                   |
| Actor Type     | Service Principal                            |
| Application ID | a1b2c3d4-e5f6-7890-abcd-ef1234567890        |
| Activity       | Patch DeviceConfiguration — Update Policy    |
| Resource Type  | DeviceConfiguration                          |
| Pattern Interval | Hourly                                     |
| Average Interval | 61 minutes                                 |
| Occurrences    | 2,847 times                                  |
| Date Range     | 2024-10-28 → 2025-04-28                      |
| Impact Level   | 🔴 High                                      |

Repeated Property Changes:
- `passwordRequired`: always set to `true`
- `passwordMinimumLength`: always set to `8`
```

---

## Pattern Detection Logic

The agent identifies noisy neighbours by:

1. **Grouping** events by: `actor identity` + `activityType` + `resource type`
2. **Measuring intervals** between consecutive events in each group
3. **Classifying** as consistent if the coefficient of variation (std_dev/mean) < 25%
4. **Confirming idempotency** by checking that `modifiedProperties.newValue` is always the same
5. **Classifying interval** as:
   - **Hourly**: ~3,600 seconds average interval
   - **Daily**: ~86,400 seconds average interval
   - **Weekly**: ~604,800 seconds average interval
   - **Other**: any other consistent interval

---

## Required Microsoft Graph API Permissions

| Permission | Type | Description |
|------------|------|-------------|
| `DeviceManagementApps.Read.All` | Delegated | Read Intune app audit events |
| `DeviceManagementConfiguration.Read.All` | Delegated | Read Intune configuration audit events |
| `DeviceManagementManagedDevices.Read.All` | Delegated | Read Intune device audit events |
| `DeviceManagementRBAC.Read.All` | Delegated | Read Intune RBAC audit events |

> **Note:** Only one of the above permissions is required. `DeviceManagementConfiguration.Read.All` covers the majority of audit event types relevant to noisy neighbour detection.

---

## Why This Matters

In Microsoft multi-tenant environments, when an automation or account repeatedly calls the Intune Graph API to set the same values (even when no actual change is needed), it:

- Generates unnecessary audit log volume
- Consumes shared platform API quota
- Can trigger unwanted change management notifications
- May cause compliance drift detection false positives
- Degrades platform performance for neighboring tenants

The **Noisy Neighbour Agent** helps administrators identify and remediate this behavior before it becomes a support escalation.

---

## Support and Contribution

This is a community plugin for Microsoft Security Copilot. For issues, feature requests, or contributions, please visit the [GitHub repository](https://github.com/schenardie/Noisy-Neighbour-Agent).
