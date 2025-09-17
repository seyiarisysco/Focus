👉 Do you also want me to add forecasting (QuickSight ML-powered) to the Monthly Trends sheet, so leadership sees a 3–6 month cost projection?

Awesome 🚀 — let’s add ML-powered forecasting to your Monthly Trends sheet in QuickSight, so leadership can see 3–6 month cost projections directly in the dashboard.

🔹 1. How QuickSight Forecasting Works

Available in Line Charts with a time-series field.

Uses built-in ML forecasting models (ARIMA-like).

You can configure:

Forecast horizon (e.g., 3 or 6 months)

Prediction interval (default 95%)

We’ll apply it to the Monthly Summary dataset, forecasting TotalCost by OU.

🔹 2. JSON Snippet (Forecast Configuration)

Extend the Monthly Trends LineChart:

{
  "LineChartVisual": {
    "Title": { "Text": "Monthly Spend by OU (with Forecast)" },
    "ChartConfiguration": {
      "FieldWells": {
        "LineChartAggregatedFieldWells": {
          "Category": [
            { "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "Month" } }
          ],
          "Values": [
            {
              "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "TotalCost" },
              "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
            }
          ],
          "Colors": [
            { "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "OU" } }
          ]
        }
      },
      "ForecastConfigurations": [
        {
          "ForecastProperties": {
            "ForecastHorizon": 6,
            "PredictionInterval": 0.95
          }
        }
      ]
    }
  }
}



🔹 3. Terraform (Dashboard Update)

No changes in dataset Terraform — just update the template JSON with the forecast config, then redeploy:


resource "aws_quicksight_template" "focus_ou" {
  template_id    = "focus-ou-template"
  name           = "FOCUS OU Template"
  aws_account_id = var.account_id
  source_entity {
    source_analysis {
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:analysis/focus-ou-analysis"
      data_set_references = [
        {
          data_set_arn         = aws_quicksight_data_set.monthly_summary.arn
          data_set_placeholder = "Monthly Summary"
        }
      ]
    }
  }
}


🔹 4. End Result

Now your Monthly Trends sheet shows:

📈 Past spend (actuals)

🔮 Forecasted spend (next 6 months) with shaded confidence interval

📊 Breakdown by OU

Executives can immediately see:

If costs are expected to rise/fall

Which OU is projected to drive spend

Uncertainty range (upper/lower bounds)

✅ Your FinOps Dashboard Now Supports

🚨 Anomaly detection & alerts

📧 Daily cost digest emails with Top 10 + CSV

📨 Daily dashboard PDF snapshots (execs)

📂 Historical archive in S3 + Athena views

📊 Monthly / Quarterly / Yearly trend sheets

🔮 ML-powered cost forecasting (3–6 months)