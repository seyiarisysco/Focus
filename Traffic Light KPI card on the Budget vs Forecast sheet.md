👉 Do you also want me to add a “Traffic Light KPI card” on the Budget vs Forecast sheet (just a big Red/Yellow/Green indicator per OU), so leadership sees budget health in a single glance without charts?

Perfect ✅ — let’s add Traffic Light KPI cards to the Budget vs Forecast sheet. These will give executives a single-glance budget health indicator per OU, with Red / Yellow / Green status.

🔹 1. Traffic Light Logic

We’ll define status for each OU:

🟥 Red → Actual spend > Budget

🟨 Yellow → Forecast spend > Budget (risk of overspend)

🟩 Green → Both Actual & Forecast ≤ Budget

🔹 2. Athena View for KPI Cards

To make this easy in QuickSight, create a view in Athena:

CREATE OR REPLACE VIEW finops_db.ou_budget_health AS
WITH monthly AS (
  SELECT date_trunc('month', BillingDate) AS Month,
         OU,
         SUM(Cost) AS TotalCost
  FROM finops_db.finops_daily_summary
  GROUP BY 1, OU
),
latest_month AS (
  SELECT OU,
         MAX(Month) AS LatestMonth
  FROM monthly
  GROUP BY OU
),
latest_data AS (
  SELECT m.OU, m.Month, m.TotalCost
  FROM monthly m
  JOIN latest_month lm ON m.OU = lm.OU AND m.Month = lm.LatestMonth
),
forecast AS (
  SELECT OU,
         approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
         avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg,
         date_add('month', 1, max(Month)) AS NextMonth
  FROM monthly
  GROUP BY OU
),
joined AS (
  SELECT l.OU,
         l.Month,
         l.TotalCost,
         b.Budget,
         f.ForecastAvg,
         f.ForecastHigh
  FROM latest_data l
  JOIN ou_budgets b ON l.OU = b.OU
  JOIN forecast f ON l.OU = f.OU
)
SELECT OU,
       CASE
         WHEN TotalCost > Budget THEN 'RED'
         WHEN ForecastAvg > Budget THEN 'YELLOW'
         ELSE 'GREEN'
       END AS BudgetStatus
FROM joined;


This outputs OU + BudgetStatus.

🔹 3. QuickSight KPI Cards

Add a KPI visual per OU on the “Budget vs Forecast” sheet.

{
  "KPIVisual": {
    "Title": { "Text": "Budget Health - OU" },
    "ChartConfiguration": {
      "FieldWells": {
        "KPIAggregatedFieldWells": {
          "Values": [
            { "Column": { "DataSetIdentifier": "OU Budget Health", "ColumnName": "BudgetStatus" } }
          ],
          "TrendGroups": [
            { "Column": { "DataSetIdentifier": "OU Budget Health", "ColumnName": "OU" } }
          ]
        }
      },
      "ConditionalFormatting": {
        "ConditionalFormattingOptions": [
          {
            "KPI": {
              "FieldId": "BudgetStatus",
              "Formatting": {
                "TextColor": {
                  "Solid": {
                    "Color": "#FF0000",
                    "Expression": "BudgetStatus = 'RED'"
                  }
                }
              }
            }
          },
          {
            "KPI": {
              "FieldId": "BudgetStatus",
              "Formatting": {
                "TextColor": {
                  "Solid": {
                    "Color": "#FFD700",
                    "Expression": "BudgetStatus = 'YELLOW'"
                  }
                }
              }
            }
          },
          {
            "KPI": {
              "FieldId": "BudgetStatus",
              "Formatting": {
                "TextColor": {
                  "Solid": {
                    "Color": "#00CC66",
                    "Expression": "BudgetStatus = 'GREEN'"
                  }
                }
              }
            }
          }
        ]
      }
    }
  }
}

🔹 4. End Result in Dashboard

On the Budget vs Forecast sheet:

📊 Combo chart still shows Actual vs Budget vs Forecast

🟩🟨🟥 KPI Traffic Lights per OU:

Marketing → 🟥 (overspent)

Finance → 🟨 (at risk)

DevOps → 🟩 (safe)

Execs now see both detailed charts and instant traffic light indicators.

✅ Final FinOps Visibility

Your FOCUS Dashboard now gives leadership:

Trend views (daily, monthly, quarterly, yearly)

Forecasts (3–6 months)

Budget vs Forecast chart

Conditional formatting (red/yellow/green bars & lines)

Traffic Light KPI cards (single-glance OU health)

OU-specific forecast alerts via email