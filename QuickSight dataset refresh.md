👉 Do you want me to also wire up a QuickSight dataset refresh (triggered after the partition registration), so the dashboard always reflects the latest daily costs without manual refresh?

Perfect ✅ — let’s make sure your QuickSight dashboard always auto-refreshes after the daily pipeline finishes (CSV → Parquet archive → Athena partition registration).

🔹 1. How QuickSight Refresh Works

QuickSight datasets can be SPICE or Direct Query.

If SPICE: You must refresh after Athena partition updates.

If Direct Query: No refresh needed, but slower for dashboards.

👉 Best practice: Use SPICE for speed + daily scheduled refresh.
We’ll trigger refresh automatically from Lambda after adding the partition.

🔹 2. Add QuickSight Refresh to Lambda

Extend your digest Lambda with QuickSight refresh:

import boto3

quicksight = boto3.client("quicksight")

def refresh_quicksight_dataset(dataset_id, aws_account_id):
    response = quicksight.create_ingestion(
        AwsAccountId=aws_account_id,
        DataSetId=dataset_id,
        IngestionId=f"ingestion-{date.today()}",
    )
    print(f"Triggered QuickSight ingestion: {response['IngestionId']}")
    return response


🔹 3. IAM Permissions Needed

Add QuickSight permissions to the Lambda role:

resource "aws_iam_role_policy" "lambda_quicksight" {
  role = aws_iam_role.metric_lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "quicksight:CreateIngestion",
          "quicksight:DescribeIngestion"
        ],
        Effect   = "Allow",
        Resource = "*"
      }
    ]
  })
}

🔹 4. Workflow (Daily Run)

Lambda generates OU CSV for email digest.

Archives CSV → Parquet in S3.

Registers Athena partition for the new day.

Triggers QuickSight dataset refresh.

QuickSight dashboard updates automatically.

📊 Exec PDF snapshots and 📧 FinOps digests always show freshest data.

🔹 5. End State (Full FinOps Automation)

✅ Ops View

📧 Daily digest email (OU-level + CSV attachment)

📂 S3 archive (Parquet, versioned, partitioned)

🔎 Athena queries ready immediately

✅ Exec View

🟩🟨🟥 KPI traffic lights (Budget health per OU)

📊 Charts (Actual vs Budget vs Forecast)

🔎 Drill-through dashboards (OU → Service → Account)

📨 Daily PDF snapshots always in sync

✅ Automation

🚨 Anomaly detection + budget alerts

🔮 Forecasting (3–6 months)

⚡ QuickSight dataset auto-refresh → no manual refresh needed

