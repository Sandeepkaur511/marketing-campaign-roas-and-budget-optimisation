# Marketing Campaign ROAS & Budget Optimisation (Power BI)

A three-page Power BI report and the analysis behind it. The project has two parts: **how the report is built** (two fact tables on shared dimensions, 36 DAX measures, a what-if driven by disconnected tables) and **what it shows** (platform-reported ROAS versus the ROAS from deals actually won).

![Dashboard preview](screenshots/mkt-spend-roas.png)

> **Data note:** the dataset is **synthetic**. Figures such as 15–25× ROAS are far above typical real-world results, so the numbers demonstrate the *method*, not real marketing performance.

## Business question

A B2B advertiser spends across Google Ads, LinkedIn Ads and Meta Ads. Each platform reports its own conversions and ROAS, but do those match revenue from deals that were actually won, and where should the next budget dollar go?

## Questions the report answers

- What is true ROAS (won-deal revenue ÷ spend) compared with platform-reported ROAS?
- Which platform acquires customers most cheaply, and how does CAC compare with deal value?
- Where does the funnel leak between impression, click, lead and won deal?
- Which audience segments convert best?
- What happens to won deals and revenue if budget moves between platforms?

## Report pages

| Page | What it shows |
|---|---|
| Spend & ROAS Overview | Total spend, true revenue, true ROAS, CAC, spend vs revenue trend, CAC vs average deal value, true vs reported ROAS by platform |
| Funnel & Conversion | Impressions → clicks → leads → won, lead status mix, conversion by audience segment, trend over time |
| Efficiency + What-if | Spend vs won deals by campaign, platform efficiency (CAC and ROAS), budget-shift scenario |

Screenshots are in [`screenshots/`](screenshots/) and a PDF export is in [`report/`](report/).

## How the report is built

### Data model

A multi-fact star schema (also called a fact constellation). Two fact tables, `Campaigns_Daily` (spend and platform-reported results, 962 rows) and `Leads` (CRM outcomes, 1,280 rows), share three dimensions: `Dim_Campaign` (platform, campaign), `Dim_Date` and `Dim_Segment`. Every relationship is many-to-one, single direction. Four disconnected helper tables drive the funnel chart and the what-if.

| Table | Type | Role |
|---|---|---|
| Campaigns_Daily | Import | Fact: daily spend, impressions, clicks, platform-reported conversions and value (962 rows, 22 campaigns). |
| Leads | Import | Fact: CRM leads with status and deal value (1,280 rows). Also has an added column `Lead Segment`. |
| Dim_Campaign | Import | Campaign and platform. |
| Dim_Date | Import (date table) | Calendar. |
| Dim_Segment | Calculated | `DISTINCT(Campaigns_Daily[target_audience_segment])`. |
| FunnelStage | Calculated, disconnected | `DATATABLE` of four stages with a sort order. |
| Budget Shift % | Calculated, disconnected | `GENERATESERIES(-50, 50, 5)` ÷ 100. |
| Source Platform / Target Platform | Calculated, disconnected | `DISTINCT(Dim_Campaign[platform])`, one copy each. |

| From (many) | To (one) | Cardinality | Filter direction | State |
|---|---|---|---|---|
| Leads[campaign_id] | Dim_Campaign[campaign_id] | Many-to-one | Single | Active |
| Leads[lead_date] | Dim_Date[Date] | Many-to-one | Single | Active |
| Leads[Lead Segment] | Dim_Segment[target_audience_segment] | Many-to-one | Single | Active |
| Campaigns_Daily[campaign_id] | Dim_Campaign[campaign_id] | Many-to-one | Single | Active |
| Campaigns_Daily[date] | Dim_Date[Date] | Many-to-one | Single | Active |
| Campaigns_Daily[target_audience_segment] | Dim_Segment[target_audience_segment] | Many-to-one | Single | Active |

**Design choices and what they made easy**

- **Shared dimensions between two facts.** Because spend and leads hang off the same platform, date and segment dimensions, one slicer filters both. That is what lets `True ROAS` divide CRM revenue by ad spend without any lookup work.
- **Segment as its own dimension.** The same campaign appears with different audience segments on different days (for example `CMP-101` has rows for Enterprise IT and Tech Leaders), so segment is not a campaign attribute. `Dim_Segment` plus a `Lead Segment` column on `Leads` lets both facts be cut by segment.
- **Disconnected `FunnelStage`.** Impressions and clicks come from one table and leads and won deals from another, so a stage table with a sort order plus one `SWITCH` measure (`Funnel Value`) gives a single chart with four ordered bars.
- **Disconnected what-if tables.** `Budget Shift %` (−50% to +50% in 5-point steps), `Source Platform` and `Target Platform` turn a scenario into three slicer choices, with defaults set through `SELECTEDVALUE`.

### DAX measures

36 measures in 5 groups. Full list with code: [`powerbi/dax_measures.md`](powerbi/dax_measures.md). Model notes: [`powerbi/data_model.md`](powerbi/data_model.md).

- **Funnel** (11): Total Leads, Total Won Deals, Click to Lead Rate, Lead to Won Rate, Total MQL, Total SQL, Total Lost, Win Rate, Total Impressions, Total Clicks, CTR
- **Spend & ROAS** (7): True Revenue (Won), Total Pipeline Value, True ROAS, CAC, Avg Deal Value (Won), Total Spend, CPC
- **Platform-Reported (Unverified)** (4): Platform-Reported Conversions, Platform-Reported Conversion Value, Platform-Reported ROAS, Platform-Reported Conv Rate
- **What-If** (12): Shift %, Source Platform (Selected), Target Platform (Selected), Source Spend (Current), Target Spend (Current), Shifted Budget Amount, Source Spend (Projected), Target Spend (Projected), Source Won Deals (Projected), Target Won Deals (Projected), Net Won Deals Impact, Net Revenue Impact
- **Other** (2): Overall Funnel Conversion Rate, Funnel Value

**Design choices and what they made easy**

- **True and platform-reported ROAS kept apart.** Platform figures sit in a display folder named "Platform-Reported (Unverified)" and their descriptions say why: the platform conversion count does not reconcile with CRM leads (the ratio varies from 18.6× to 66.5× by campaign). The gap between the two ROAS figures becomes the headline finding.
- **Assumptions live in measure descriptions.** `Avg Deal Value (Won)` records that it stands in for LTV, and the projected won-deal measures record that CAC is held constant.
- **What-if built from ordinary measures.** Projected spend = current spend ± shifted amount; projected won deals = projected spend ÷ current CAC; projected revenue = projected deals × platform average deal value. A negative shift moves budget out of the source platform.

### Visual choices

Each visual answers one question.

| Question | Visual | Measures | Why it makes the analysis easier |
|---|---|---|---|
| Is spend paying off, and how is it trending? | KPI cards + line chart of spend vs won revenue over time | `Total Spend`, `True Revenue (Won)`, `True ROAS`, `CAC` | Totals first, trend second; date and platform slicers apply to every visual on the page. |
| Which platform is cheapest, and what is a customer worth? | Clustered columns: CAC vs average deal value by platform | `CAC`, `Avg Deal Value (Won)` | Acquisition cost sits next to deal value (the LTV stand-in), so cheap versus valuable is one comparison. |
| Are platforms over-claiming? | Paired bars: True ROAS vs Platform-Reported ROAS by platform | `True ROAS`, `Platform-Reported ROAS` | Two bars per platform make the attribution gap visible without any calculation by the viewer. |
| Where does the funnel leak? | Stage bar chart, lead-status donut and three rate cards | `Funnel Value`, `Click to Lead Rate`, `Lead to Won Rate`, `Win Rate` | Counts show every stage, and the rate cards give the conversion at each step. |
| Which segments convert? | Clustered columns: leads vs won deals by segment | `Total Leads`, `Total Won Deals` | Works across both fact tables because segment is a shared dimension. |
| Which campaigns are efficient? | Bubble chart of spend vs won deals by campaign, coloured by platform, plus CAC/ROAS combo chart | `Total Spend`, `Total Won Deals`, `CAC`, `True ROAS` | Campaign-level outliers show up as isolated bubbles, and the combo chart gives the platform reference. |
| What if we move budget? | Source and target buttons, shift dropdown, four impact cards, detail table | `Shift %`, `Shifted Budget Amount`, `Net Won Deals Impact`, `Net Revenue Impact` | A scenario takes three clicks and every card and the table update together. |

### Key measure excerpts

```dax
-- True ROAS
DIVIDE([True Revenue (Won)], [Total Spend])

-- CAC
DIVIDE([Total Spend], [Total Won Deals])

-- Source Won Deals (Projected)
DIVIDE([Source Spend (Projected)], CALCULATE([CAC], FILTER(ALL(Dim_Campaign[platform]), Dim_Campaign[platform] = [Source Platform (Selected)])), 0)
```

## What the analysis found (synthetic data)

| Platform | Spend | Reported ROAS | True ROAS | CAC | Avg won deal |
|---|---|---|---|---|---|
| Google Ads | $252.8K | 23.2× | 16.3× | $2,503 | ~$40.7K |
| LinkedIn Ads | $243.0K | 25.0× | 14.7× | $2,730 | ~$40.1K |
| Meta Ads | $147.8K | 24.9× | 24.7× | $1,643 | ~$40.6K |
| **Blended** | **$643.6K** | **24.3×** | **17.6×** | **$2,298** | **~$40.5K** |

- **Platform reporting flatters results.** Blended reported ROAS is 24.3× against 17.6× true. The gap is largest for LinkedIn (25.0× reported vs 14.7× true) and Google (23.2× vs 16.3×); Meta is close (24.9× vs 24.7×).
- **Meta is cheapest per won deal.** CAC is about $1.6K for Meta against $2.5K for Google and $2.7K for LinkedIn, with similar average deal values (roughly $40–41K).
- **The funnel narrows sharply.** 27.0M impressions lead to 720K clicks, 1,280 leads and 280 won deals: click-to-lead is 0.18% and lead-to-won is 21.9%.
- **Segments differ.** Enterprise IT converts best (25.7% lead-to-won). Tech Leaders bring the most leads (389) but convert at 18.5%, and DevOps Engineers are weakest at 10.6%.
- **Budget what-if:** moving 50% of Meta budget ($73.9K) to Google projects 15 fewer won deals and about $625K less revenue.

## Recommendations to test

Hypotheses, not proven interventions.

- Judge platforms on won-deal revenue rather than platform-reported conversions.
- Avoid cutting Meta to fund higher-CAC platforms; test small shifts toward Meta and watch marginal CAC.
- Review LinkedIn lead qualification, since its reported-to-true ROAS gap is the largest.
- Investigate lead quality for Tech Leaders and DevOps Engineers.

## Definitions

- **Platform-reported ROAS:** `Platform-Reported Conversion Value` ÷ `Total Spend`.
- **True ROAS:** deal value on leads with `lead_status = "Won"` ÷ `Total Spend`. MQL and SQL deal values are pipeline, not revenue, and are excluded.
- **CAC:** `Total Spend` ÷ Won deals. Blended CAC is $2,298; the report card shows "$2K" because of its display units. TODO: set the card's display units to None.
- **Win rate:** Won ÷ (Won + Lost) = 280 ÷ 566 = 49.5%. **Lead-to-won rate:** Won ÷ all leads = 21.9%.
- **What-if:** projected spend for each platform = current spend ± `Shifted Budget Amount` (= source spend × Shift %); projected won deals = projected spend ÷ that platform's current CAC; projected revenue = projected won deals × that platform's average won deal value.
- **Average deal value replaces LTV** because lead-gen data has no repeat-purchase history.

## Data

| File | Rows | Description |
|---|---|---|
| `Campaigns_Daily.csv` | 962 | Daily spend, impressions, clicks, platform-reported conversions and value for 22 campaigns (Jan 2025 – Jun 2026) |
| `Leads.csv` | 1,280 | Lead ID, campaign, date, source, status (MQL / SQL / Won / Lost), deal value |

The two tables join on `campaign_id` (100% of leads match a campaign).

**Where the data came from:** TODO: say how the data was produced and link the source if public.

**Preparation:** currency and number columns arrive as text (`"$495"`, `"19,800"`) and were converted to numbers. TODO: confirm steps in Power Query.

### Data observations

- Platform-reported "conversions" (51,991) are a different unit from CRM leads (1,280) and do not reconcile; the model keeps them in a separate "Unverified" folder for that reason.

## Limitations

- **Synthetic data.** ROAS of 15–25× is far above typical real-world figures, so treat the numbers as a demonstration of method.
- **The what-if is linear.** It assumes cost per won deal stays constant as spend changes, with no saturation or diminishing returns, so it is directional only.
- **Sign convention.** In the what-if, a negative Budget Shift % moves budget out of the source platform; a positive value would move it the other way.
- **Funnel scale.** Bars are linear, so leads and won deals look like slivers beside impressions; the data labels carry the values.
- Revenue is counted when a lead is Won; the delay between spend and close is not modelled, and average deal value stands in for LTV.

## Repository structure

```
marketing-roas-optimization-powerbi/
├── README.md
├── data/                  # synthetic CSVs + data_dictionary.md
├── powerbi/               # .pbix file + dax_measures.md
├── screenshots/
└── report/                # PDF export
```

## Tools

Power BI Desktop · DAX · Data modelling · Funnel analysis · What-if scenario

## Author

Sandeep Kaur · [Portfolio](TODO_your_portfolio_url) · [LinkedIn](https://www.linkedin.com/in/sandeep-kaur-4b960219a/)
