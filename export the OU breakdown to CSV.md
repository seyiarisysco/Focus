👉 Do you also want me to configure the drill-through sheet to export the OU breakdown to CSV (so FinOps can pull the exact numbers execs click into)?

Perfect ✅ — let’s extend the OU drill-through sheet so FinOps (or execs if you allow) can export the OU breakdown as CSV right from QuickSight.

🔹 1. How CSV Export Works in QuickSight

Export is enabled per visual (table/pivot table).

Users with Reader role can download data as CSV or Excel if export is enabled.

Export can be summary data (aggregated) or detailed data (row-level).

We’ll enable this on the OU Details table.

🔹 2. OU Detail Table With Export Enabled

Here’s the updated JSON for the OU Details Table:

{
  "TableVisual": {
    "Title": { "Text": "Cost Breakdown by Service" },
    "ChartConfiguration": {
      "FieldWells": {
        "TableAggregatedFieldWells": {
          "GroupBy": [
            { "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "Service" } },
            { "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "LinkedAccount" } }
          ],
          "Values": [
            {
              "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "Cost" },
              "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
            }
          ]
        }
      }
    },
    "Actions": [],
    "ExportOptions": {
      "CSV": { "AvailabilityStatus": "ENABLED" },
      "Excel": { "AvailabilityStatus": "ENABLED" }
    }
  }
}


🔹 3. User Workflow

Exec clicks OU KPI card → “View OU Details”.

Drill-through opens OU Details sheet.

Table lists Service, LinkedAccount, Cost for that OU.

Exec/FinOps clicks “Export → CSV”.

File downloads as:

OU_Detail_Marketing_2025-09-15.csv

Contains all visible data (Service, Account, Cost).

🔹 4. Example Exported CSV

Service,LinkedAccount,Cost
AmazonEC2,Account123,$7,200
AmazonS3,Account456,$2,800
AmazonSageMaker,Account789,$5,400
AWSLambda,Account234,$1,000
Total,,$16,400


🔹 5. End Result

Your FinOps dashboard now supports:

🚨 Alerts (anomaly + forecast budget)

📧 Daily digests (Ops view with CSV)

📨 Exec PDF snapshots (visual summary)

📊 Trends (daily, monthly, quarterly, yearly)

🔮 Forecasting (3–6 months)

⚠️ OU-specific budget alerts

🟩🟨🟥 Traffic Light KPIs (budget health per OU)

🔎 Drill-through per OU → Service/Account detail

📂 Export to CSV from OU drill-through tables

