👉 Do you also want me to add a QuickSight Budget vs Forecast sheet (visual variance chart per OU), so leadership sees how each OU is tracking against its budget visually?

Perfect ✅ — let’s add a QuickSight “Budget vs Forecast” sheet so leadership can see visually how each OU is tracking against its budget. This will complement the OU-specific forecast alerts with an executive-friendly view.

🔹 1. Data Setup

We already have:

monthly_summary (Athena view with actual monthly costs)

ou_budgets (Athena table with OU budgets)

forecast (Athena query for forecasted spend)

We’ll create a combined dataset:

WITH monthly AS (
  SELECT date_trunc('month', BillingDate) AS Month,
         OU,
         SUM(Cost) AS TotalCost
  FROM finops_db.finops_daily_summary
  GROUP BY 1, OU
),
latest_forecast AS (
  SELECT OU,
         date_add('month', 1, max(Month)) AS NextMonth,
         approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
         avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg
  FROM monthly
),
budgets AS (
  SELECT * FROM ou_budgets
)
SELECT m.OU,
       m.Month,
       m.TotalCost,
       b.Budget,
       f.ForecastAvg,
       f.ForecastHigh,
       f.NextMonth
FROM monthly m
LEFT JOIN latest_forecast f ON m.OU = f.OU
LEFT JOIN budgets b ON m.OU = b.OU;


This produces:

Actual cost (TotalCost)

Budget (Budget)

Forecasted spend (ForecastAvg / ForecastHigh)

resource "aws_quicksight_data_set" "budget_forecast" {
  data_set_id   = "budget-forecast-dataset"
  name          = "Budget vs Forecast"
  aws_account_id = var.account_id
  import_mode   = "DIRECT_QUERY"

  physical_table_map = {
    "budget_forecast" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "Budget vs Forecast SQL"
        sql_query       = file("${path.module}/sql/budget_forecast.sql")
      }
    }
  }
}


🔹 3. New Dashboard Sheet (JSON Snippet)

{
  "Name": "Budget vs Forecast",
  "Visuals": [
    {
      "ComboChartVisual": {
        "Title": { "Text": "Actual vs Budget vs Forecast by OU" },
        "ChartConfiguration": {
          "FieldWells": {
            "ComboChartAggregatedFieldWells": {
              "Category": [
                { "Column": { "DataSetIdentifier": "Budget vs Forecast", "ColumnName": "Month" } }
              ],
              "BarValues": [
                {
                  "Column": { "DataSetIdentifier": "Budget vs Forecast", "ColumnName": "TotalCost" },
                  "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                }
              ],
              "LineValues": [
                {
                  "Column": { "DataSetIdentifier": "Budget vs Forecast", "ColumnName": "Budget" }
                },
                {
                  "Column": { "DataSetIdentifier": "Budget vs Forecast", "ColumnName": "ForecastAvg" }
                }
              ],
              "Colors": [
                { "Column": { "DataSetIdentifier": "Budget vs Forecast", "ColumnName": "OU" } }
              ]
            }
          }
        }
      }
    }
  ]
}


🔹 4. End Result in Dashboard

On the new “Budget vs Forecast” sheet:

📊 Bars = Actual monthly spend (TotalCost)

📈 Line 1 = Budget (Budget)

🔮 Line 2 = Forecast (ForecastAvg)

⚠️ If actuals or forecast exceed the budget line, it’s immediately visible

✅ Final FinOps Dashboard Capabilities

🚨 Anomaly detection & alerts (real-time spikes)

📧 Daily cost digest (Top 10 + CSV)

📨 Exec PDF snapshot (QuickSight subscription)

📂 Historical archive + Athena roll-ups (daily, monthly, quarterly, yearly)

📈 Monthly/Quarterly/Yearly trend sheets (QuickSight)

🔮 Forecasting (3–6 months) with ML in QuickSight

⚠️ OU-specific forecast budget alerts (Athena + Lambda + SNS)

📊 Budget vs Forecast sheet (visual actual vs budget vs forecast)