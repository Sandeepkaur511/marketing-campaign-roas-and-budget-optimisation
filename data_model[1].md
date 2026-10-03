# Data model: Marketing Campaign ROAS & Budget Optimisation

## Shape

A multi-fact star schema: two fact tables at different grains share three conformed dimensions, so spend (daily, per campaign) and outcomes (per lead) can be sliced by the same platform, date and audience segment. All relationships are many-to-one and single-direction.

| Table | Type | Notes |
|---|---|---|
| `Campaigns_Daily` | Fact (Import, 962 rows) | Daily spend, impressions, clicks, platform-reported conversions and value |
| `Leads` | Fact (Import, 1,280 rows) | One row per lead: status (MQL / SQL / Won / Lost) and deal value |
| `Dim_Campaign` | Dimension (Import) | Campaign and platform |
| `Dim_Date` | Dimension (Import, date table) | Shared calendar |
| `Dim_Segment` | Dimension (Calculated) | `DISTINCT(Campaigns_Daily[target_audience_segment])` |
| `FunnelStage` | Disconnected helper | `DATATABLE` of four stages with a sort order (drives the funnel visual) |
| `Budget Shift %` | Disconnected helper | `GENERATESERIES(-50, 50, 5)` divided by 100 (what-if parameter) |
| `Source Platform`, `Target Platform` | Disconnected helpers | `DISTINCT(Dim_Campaign[platform])`, used as slicers in the what-if |

## Relationships

| From (many) | To (one) |
|---|---|
| `Campaigns_Daily[campaign_id]` | `Dim_Campaign[campaign_id]` |
| `Leads[campaign_id]` | `Dim_Campaign[campaign_id]` |
| `Campaigns_Daily[date]` | `Dim_Date[Date]` |
| `Leads[lead_date]` | `Dim_Date[Date]` |
| `Campaigns_Daily[target_audience_segment]` | `Dim_Segment[target_audience_segment]` |
| `Leads[Lead Segment]` | `Dim_Segment[target_audience_segment]` |

## Derived column

| Column | Definition |
|---|---|
| `Leads[Lead Segment]` | TODO: paste the expression. (Every lead matches exactly one `Campaigns_Daily` row on campaign and date, so the segment is presumably taken from that row; confirm.) |

## Design decisions worth explaining

- **Two facts, shared dimensions.** Spend lives at daily campaign grain and leads at lead grain. Joining them directly would multiply rows; slicing both through `Dim_Campaign`, `Dim_Date` and `Dim_Segment` keeps every total correct.
- **Cross-table measures.** `True ROAS` divides `Leads` revenue by `Campaigns_Daily` spend; `CAC` divides spend by `Leads` won deals. Both work per platform, date or segment because both facts filter through the same dimensions.
- **Reported vs true kept separate.** Platform-reported measures sit in their own `Platform-Reported (Unverified)` display folder and have descriptions warning that they do not reconcile with CRM leads.
- **Disconnected tables** drive the funnel (one `SWITCH` measure returns a different measure per stage) and the what-if (source, target, shift %).
