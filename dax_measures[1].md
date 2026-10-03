# DAX measures: Marketing Campaign ROAS & Budget Optimisation

Measures are organised into display folders. The `Platform-Reported (Unverified)` folder is kept apart from the CRM-verified measures on purpose, so the two kinds of ROAS cannot be mixed up on a visual.

**36 measures** in a dedicated measure table, grouped below.

## Funnel

### Total Leads

```dax
COUNTROWS(Leads)
```

### Total Won Deals

```dax
CALCULATE(COUNTROWS(Leads), Leads[lead_status]="Won")
```

### Click to Lead Rate

```dax
DIVIDE([Total Leads], [Total Clicks])
```

### Lead to Won Rate

```dax
DIVIDE([Total Won Deals], [Total Leads])
```

### Total MQL

```dax
CALCULATE(COUNTROWS(Leads), Leads[lead_status]="MQL")
```

### Total SQL

```dax
CALCULATE(COUNTROWS(Leads), Leads[lead_status]="SQL")
```

### Total Lost

```dax
CALCULATE(COUNTROWS(Leads), Leads[lead_status]="Lost")
```

### Win Rate

_Won deals as a share of decided (Won+Lost) leads. Excludes leads still active as MQL/SQL._

```dax
DIVIDE([Total Won Deals], [Total Won Deals] + [Total Lost], 0)
```

### Total Impressions

```dax
SUM(Campaigns_Daily[impressions])
```

### Total Clicks

```dax
SUM(Campaigns_Daily[clicks])
```

### CTR

```dax
DIVIDE([Total Clicks], [Total Impressions])
```

## Spend & ROAS

### True Revenue (Won)

```dax
CALCULATE(SUM(Leads[deal_value]), Leads[lead_status]="Won")
```

### Total Pipeline Value

```dax
SUM(Leads[deal_value])
```

### True ROAS

_CRM-verified ROAS: actual Won deal revenue / ad spend. Use this for platform/campaign performance decisions._

```dax
DIVIDE([True Revenue (Won)], [Total Spend])
```

### CAC

_Customer Acquisition Cost: ad spend per Won deal._

```dax
DIVIDE([Total Spend], [Total Won Deals])
```

### Avg Deal Value (Won)

_Stand-in for LTV: this is lead-gen data (one deal per lead, no repeat-purchase history), so true LTV isn't available. Average Won deal value is the closest honest proxy for comparing against CAC._

```dax
DIVIDE([True Revenue (Won)], [Total Won Deals])
```

### Total Spend

```dax
SUM(Campaigns_Daily[daily_spend])
```

### CPC

```dax
DIVIDE([Total Spend], [Total Clicks])
```

## Platform-Reported (Unverified)

### Platform-Reported Conversions

_Ad-platform self-reported conversion events. Does NOT reconcile with actual Leads table (ratio varies 18.6x-66.5x by campaign) - shown for reference only, not used in the true CRM funnel._

```dax
SUM(Campaigns_Daily[conversions])
```

### Platform-Reported Conversion Value

```dax
SUM(Campaigns_Daily[conversion_value])
```

### Platform-Reported ROAS

_Ad-platform self-attributed ROAS. Not CRM-verified - compare against True ROAS, don't use alone for decisions._

```dax
DIVIDE([Platform-Reported Conversion Value], [Total Spend])
```

### Platform-Reported Conv Rate

```dax
DIVIDE([Platform-Reported Conversions], [Total Clicks])
```

## Not in a display folder

### Overall Funnel Conversion Rate

```dax
DIVIDE([Total Won Deals], [Total Impressions],0)
```

### Funnel Value

```dax
SWITCH(
    SELECTEDVALUE(FunnelStage[Stage]),
    "1. Impressions", [Total Impressions],
    "2. Clicks", [Total Clicks],
    "3. Leads", [Total Leads],
    "4. Won Deals", [Total Won Deals]
)
```

## What-If

### Shift %

```dax
SELECTEDVALUE('Budget Shift %'[Budget Shift %], 0)
```

### Source Platform (Selected)

```dax
SELECTEDVALUE('Source Platform'[Select Source Platform], "Google Ads")
```

### Target Platform (Selected)

```dax
SELECTEDVALUE('Target Platform'[Select Target Platform], "Meta Ads")
```

### Source Spend (Current)

```dax
CALCULATE([Total Spend], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)]))
```

### Target Spend (Current)

```dax
CALCULATE([Total Spend], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Target Platform (Selected)]))
```

### Shifted Budget Amount

_Dollar amount moved from Source to Target platform, based on the Shift % what-if slider._

```dax
[Source Spend (Current)] * [Shift %]
```

### Source Spend (Projected)

```dax
[Source Spend (Current)] + [Shifted Budget Amount]
```

### Target Spend (Projected)

```dax
[Target Spend (Current)] - [Shifted Budget Amount]
```

### Source Won Deals (Projected)

_Assumes Source platform's current CAC (cost per won deal) holds constant at the new spend level._

```dax
DIVIDE([Source Spend (Projected)], CALCULATE([CAC], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)])), 0)
```

### Target Won Deals (Projected)

_Assumes Target platform's current CAC (cost per won deal) holds constant at the new spend level._

```dax
DIVIDE([Target Spend (Projected)], CALCULATE([CAC], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Target Platform (Selected)])), 0)
```

### Net Won Deals Impact

```dax
([Source Won Deals (Projected)] + [Target Won Deals (Projected)]) - (CALCULATE([Total Won Deals], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)])) + CALCULATE([Total Won Deals], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Target Platform (Selected)])))
```

### Net Revenue Impact

```dax
([Source Won Deals (Projected)] * CALCULATE([Avg Deal Value (Won)], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)]))) + ([Target Won Deals (Projected)] * CALCULATE([Avg Deal Value (Won)], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Target Platform (Selected)]))) - (CALCULATE([True Revenue (Won)], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)])) + CALCULATE([True Revenue (Won)], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Target Platform (Selected)])))
```

