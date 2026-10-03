# Governance & Report Lifecycle

How this multi-tenant report is run in Power BI Service.

## Workspaces & deployment pipeline

| Stage | Workspace | Data source | Who has access |
|---|---|---|---|
| Dev | `MerchantAnalytics [Dev]` | `DataFolder` = dev extract | BI developers |
| Test | `MerchantAnalytics [Test]` | test Azure SQL | BI + 3 pilot account managers |
| Prod | `MerchantAnalytics` | prod Azure SQL | App audience only |

- Deployment rules swap the `DataFolder` / server parameter per stage, so the same .pbix promotes without edits.
- Changes go Dev → Test → Prod through the deployment pipeline; nobody publishes straight to Prod.
- The .pbix is saved as a **Power BI Project (.pbip)** in Git so measures and RLS roles are diffable in pull requests.

## Security model

- Users never get workspace roles in Prod; they consume the **Power BI App**.
- Dataset role `Dynamic Tenant` is assigned to one AAD group per audience (Execs, Account Managers, Merchant Owners).
- The mapping lives in `security_user_access`, owned by the data team, loaded with the dataset. Onboarding a merchant = one row, no report change.
- Every release is tested with **View as** for one user per role (exec / region AM / merchant owner / unknown user → sees nothing).

## Refresh

- Scheduled refresh 4× daily via on-premises gateway (or cloud connection for Azure SQL).
- `fact_orders` uses **incremental refresh**: keep 30 months, refresh last 3 days, detect data changes on `DateKey`.
- Refresh failure notifications go to the BI on-call mailbox.

## Performance tuning checklist

- Star schema only; no bi-directional relationships (RLS + bi-di = ambiguous paths and slow queries).
- Integer surrogate keys; high-cardinality text (`CustomerID`) kept only where a visual needs it.
- RLS expression filters the small dimension (`dim_merchant`, 180 rows), never the fact table.
- Checked with Performance Analyzer and DAX Studio (Server Timings) — target < 1 s per visual.
